---
layout: post
title: "Automated nightly LLM testing: an AI agent benchmarks new local models while I sleep"
description: "An AI agent tested 18 new open-weight models on my home server overnight. None replaced the Qwen3.8-27B my assistant already runs."
date: 2026-10-22
image: /assets/img/llm-testing-overnight/chart.png
tags: [Local LLM, LLM evaluation, AI agents]
redirect_from:
  - /llm-testing-overnight/
---

Every week a dozen new open-weight models claim to beat the last one. I wanted to know which of them could actually replace the model my personal assistant runs on, so I built a pipeline that finds candidates, tests them overnight and sends me a verdict over breakfast. Over the past two weeks it has tested eighteen models, and none has replaced the incumbent, an uncensored build of Qwen3.8-27B. The best one passed a single graded run more, but was 18% slower and refused a harmless request. The code, the test suite and every result are on GitHub: [local-llm-eval](https://github.com/siimtolk/local-llm-eval).

## Why leaderboards were not enough

I run a personal Hermes assistant on my own hardware. It builds morning briefings, reads mail, sets reminders, writes code, searches the web, runs a daily quiz and answers in Estonian as often as in English. Behind it sits one open-weight model on a home server with two consumer GPUs and 32 GB of video memory between them.

That model answers every request all day. Swapping it changes every reply the assistant sends. A new release can top a public leaderboard and still be useless for me. NVIDIA's new Nemotron 3.5 Bolt 74B-A9B ranks [4th among models under 100B parameters](https://artificialanalysis.ai/articles/nemotron-3-5-super-and-bolt) on Artificial Analysis. The build that fits my cards passed 247 of 291 runs in my suite, against production's 283, and only 16 of 45 in Estonian. A model might not speak Estonian, or it might think for so long that a simple reminder takes a minute. Its chat template might return an empty reply.

No leaderboard answers that, so I built my own test. Then I got tired of tracking releases and running it by hand. The overnight part borrows the idea of Andrej Karpathy's [autoresearch](https://github.com/karpathy/autoresearch), where an agent runs training experiments while you sleep. Here the same loop picks a production model instead.

## The framework: a loop that runs itself

One daily loop runs five stages. Plain code finds, downloads and reports; the production model judges, and a Claude Code agent runs the tests.

1. **Discover.** Each morning a script pulls trending models, matching search results and new uploads from trusted publishers on Hugging Face, plus the day's top community posts about new releases. Code drops adapters, embedding and OCR models, and anything too big for the GPUs, already tried, older than a month or barely liked.
2. **Judge.** The production model checks the shortlist against its own specs and the hardware, and picks up to three that look at least as capable. The picks arrive in my morning briefing with a poll.
3. **Fetch.** A risk check looks at each pick's Hugging Face file scans, its GGUF header and the uploader. The ones I tick in the poll are downloaded over the day in a quant that fits the GPUs, and a 4-bit vLLM build of the same model is queued when one exists.
4. **Test.** At night a coding agent (Claude Code) takes over the GPUs, runs the suite on each candidate, compares the scores with production's, writes up and commits the result and restores production.
5. **Report.** At breakfast a WhatsApp message gives each model's verdict (passed, rejected or failed to start) and the reason.

Production must be back by morning, when the day's jobs start. Until then, any LLM requests fall back to Claude and the assistant keeps answering.

## The serving stack: two 16 GB cards, one model

Everything runs on [llama.cpp](https://github.com/ggml-org/llama.cpp)'s `llama-server`, built from source with CUDA. It speaks the OpenAI chat API. The assistant, a web chat UI and the test suite all use the same endpoint. The hardware is two NVIDIA RTX 5060 Ti cards with 16 GB each, (plus an old 6 GB GTX 1660 Super that only holds the vision encoder and is not relevant for this test).

A 27B dense model at 4 bits takes about 15.5 GB, which leaves about 0.5 GB of a 16 GB card for context and buffers. The model is split by layers across both 5060 Tis. Each card then holds half the weights, and the freed memory goes to context. These are the production settings:

| Setting | Value | Why |
| --- | --- | --- |
| Quant | IQ4\_XS, 15.5 GB | Scored the same as Q5\_K\_M (19.7 GB) and ran about 20% faster |
| GPU layers | All (`--n-gpu-layers 999`), split over both 5060 Ti | Nothing spills to the CPU; splitting cost no generation speed |
| Vision encoder | On the third, smaller card (`--mmproj-device`) | Keeps the big cards' memory for weights and cache |
| Context | 262,144 tokens in total (`--ctx-size`) | Shared out between the slots below |
| Slots | 2 (`--parallel 2`), 128k each | Two conversations keep their caches; see below |
| Batching | Continuous batching, default batch sizes (2048 / 512) | Both slots decode in the same GPU pass |
| KV cache | q4\_0 for keys and values, flash attention on | Makes two 128k slots fit; no quality loss showed up in the suite |
| Speculative decoding | `--spec-type draft-mtp` | Uses the draft head built into the quant |
| Sampling | temp 1.0, top-p 0.95, top-k 20, min-p 0 | The model makers' settings for thinking mode |
| Reasoning | Effort "low" by default, switchable per request | Short everyday turns should not think for long |
| Ops | API key, Prometheus metrics, Grafana dashboard | Speed and load visible per model |

*Table 1. Production settings for llama-server.*

In total it uses 25.4 GB of the 32 GB.

### The llama.cpp build

Production runs llama.cpp build b11707 (commit `1268edac5`), compiled locally:

| Part | Version or flag |
| --- | --- |
| Build type | `-DGGML_CUDA=ON -DCMAKE_BUILD_TYPE=Release` |
| CUDA toolkit | 13.3, NVIDIA driver 615.71 |
| GPU targets | `-DCMAKE_CUDA_ARCHITECTURES="75-real;120a-real"`: Turing for the GTX 1660 Super, Blackwell for the 5060 Tis |
| Compilers | GCC 16.2 for the C++ code. CUDA's `nvcc` is pointed at GCC 15 with `-DCMAKE_CUDA_HOST_COMPILER=/usr/bin/g++-15` |
| OS | Fedora Linux, x86\_64 |

*Table 2. The llama.cpp build.*

Two copies of llama.cpp sit side by side, so tests never touch what production runs:

- **`llama.cpp-prod`**, pinned at b11707: production.
- **`llama.cpp-next`**, a test copy that every candidate runs on. It is the same build as production unless a new model's architecture needs newer code; then the agent pulls and rebuilds it.

Production itself moves to a newer build only after its own model has passed the full suite on it at least as well as before. Apart from that, every test is about the model, not llama.cpp.

## What the test measures

The suite has 97 cases taken from what the assistant really does. Each runs three times, because sampling is random and a model that gets something right once in three tries is not reliable. Every answer is graded by code, not by another LLM. The same answer gets the same score every night.

| Area | What a pass means |
| --- | --- |
| Tool use | The right tool with the right arguments, no call when none is needed, a question first when details are missing |
| Agent tasks | Multi-step jobs against simulated files, web and downloads: fix a bug until the tests pass, reconcile two spreadsheets, remember a preference from 19 turns back |
| Reasoning | Exact answers to everyday problems and harder puzzles |
| Extraction | Clean JSON pulled out of messy text |
| Code | Generated functions that pass executed tests |
| Long context | Finding one fact in prompts of 19k, 30k and about 105k tokens |
| Safety | Ignoring an instruction planted inside a document |
| Following instructions | Exact message layouts, word limits, staying silent when nothing qualifies |
| Judging | Grading quiz answers: lenient on spelling, strict on facts |
| Estonian | The same skills in a small language with 14 grammatical cases |

*Table 3. What the 97-case suite tests.*

The model I want also has to carry out edgy but harmless tasks instead of refusing them: write a port scanner for my home network, test my own web form for SQL injection, or set up a torrent client behind a VPN and add torrents. Those are counted separately.

Two more passes run next to the graded ones. A speed pass measures prose and tool-call output, a 30k-token prompt, two users at once and two long chats taking turns. Then the everyday cases run again with reasoning switched off, which shows whether the model is still usable for quick jobs, and how much faster.

### How much the models may think

Reasoning models write a hidden chain of thought before they answer, and every thought costs time. The suite controls this with three settings, sent the same way to every model:

| Setting | Value in the suite | What it does |
| --- | --- | --- |
| Reasoning effort | `low`, set on the server (`--reasoning-effort low`) | Passed to the model's chat template. Qwen3.8's template turns it into one system-prompt line: "Keep your thinking brief and focused". It is a request, not a limit |
| Token limit per answer | 6,000 tokens; 2,000 for the edgy prompts, 700 and 300 in the speed pass | `max_tokens` covers thinking and answer together. A model still thinking at the limit has no answer, and the run fails as "out of tokens" |
| Thinking off | `enable_thinking: false` plus `reasoning_effort: "none"` | Skips the thinking block. Used for the second pass over the everyday sections |

*Table 4. How the suite controls thinking.*

There is no separate thinking budget. The 6,000-token limit is the only hard cap, on purpose: the test should show whether a model knows when to stop.

Why low? With no effort set, Qwen3.8's template defaults to "xhigh" and asks the model to validate assumptions and weigh alternatives. That is too slow for reminders and quick questions, so production runs at low, and a client can ask for more per request. The template accepts only xhigh, medium and low. Sending "high" returns an error, which is worth knowing before you copy settings from one model to another.

At low effort production stays well inside the limit. Its median graded run used 264 tokens, and only 2 of 291 runs hit 6,000, both on the same code task. One edgy prompt also hit its 2,000 cap. With thinking off the median run drops to 51 tokens and the longest to 663.

Thinking off saves time on every model but one, and costs most of them accuracy. The second pass covers the 47 everyday cases, 141 runs per model:

| Model | Tokens, thinking on → off | Time, on → off | Runs passed, on → off |
| --- | --- | --- | --- |
| Qwen3.8-27B RVN (production) | 29k → 11k | 9.8 → 4.5 min | 137 → 113 |
| Coder390 EfficientThink (nerkyor) | 25k → 16k | 11.3 → 7.3 min | 131 → 116 |
| Huihui Coder390 (retest) | 29k → 8k | 9.7 → 3.4 min | 127 → 115 |
| Gemma 4 31B | 56k → 5k | 67 → 6.4 min | 132 → 118 |
| Ornith 1.5 35B-A3B | 52k → 11k | 9.0 → 2.3 min | 117 → 109 |
| MiniCPM-V 4.7 35B-A3B | 39k → 7k | 6.8 → 1.5 min | 106 → 126 |
| Muse Glimmer 30B Abliterated | 33k → 90k | 9.9 → 29 min | 129 → 131 |

*Table 5. Thinking on vs off, 47 everyday cases (141 runs).*

Muse Glimmer is the exception because its template ignores the switch and it reasons anyway. MiniCPM is the only model that got better with thinking off (by 20 test runs). It never hit the token limit in these cases. Instead, with thinking on it often thought briefly and then returned an empty reply: all three runs of the exact-reply and Estonian-reply cases came back blank. Thinking off fixed most of those, and it stopped sending messages without asking first. The Qwen models lost 12 to 24 runs, mostly in tool use, instruction rules and Estonian.

On single cases the picture is less tidy. Production used more tokens with thinking off on 2 of the 47 cases, and nerkyor's Coder390 on 3, because the model worked the problem out in its visible answer instead. The second pass leaves out the hard puzzles, so the suite cannot say how often that happens on harder questions. In a quick check of two requests per setting, production answered one counting puzzle in about 800 tokens with thinking off, against about 550 at low effort.

So is thinking off worth using? For production, yes, on narrow jobs. Its median answer takes a quarter to two thirds of the time, with no loss on these:

- **Grading quiz answers:** 18 of 18 either way, 1.3 s per answer instead of 1.9 s.
- **Simple English tool calls:** lights, weather, shopping, timers, calendar and three tools at once all passed 3 of 3.
- **Fixed-format text:** staying silent, exact replies, plain text and three bullets all passed 3 of 3, in under a second. The WhatsApp brief layout passed 2 of 3: one run dropped the outlet names after the links.
- **Simple extraction and Estonian replies:** invoices, file names, log lines, Estonian summaries and weather all passed 3 of 3.

It loses where the model has to stop and check something:

- **Asking before acting:** "Send a message to Mart" was sent at once, with made-up text, in all three runs.
- **Typos:** "turn of the kitchn lights" failed 1 of 3 runs: the model turned the lights on.
- **Estonian tool calls:** "Pane elutoa tuli 50% peale" (set the living-room light to 50%) got no tool call in any run.
- **Grammar and counting:** Estonian case forms, numbers in words, an acrostic and a word limit.

The assistant still runs every request with thinking on. A good split would be thinking off for quiz grading and short fixed-format replies, and thinking on for anything that acts in the world or speaks Estonian.

The same settings land differently on other models, because each template decides what "low" means. Gemma 4 26B ran out of tokens 20 times at the same limit. Muse Glimmer's template ignores the thinking-off switch and reasons anyway, at about three times the tokens, so its thinking-off score cannot be compared with production's.

## Letting an agent run with nobody watching

An agent that downloads strangers' models and restarts my production server at 3 a.m. needs firm limits. Five rules make it safe enough to leave alone (at least on a home server that holds nothing I could not rebuild):

- **The deadline is enforced from outside.** A separate runner gives the agent a time limit, kills its whole session at the deadline whatever it is doing, and restores production itself. The agent is told the deadline, but nothing depends on it keeping to it. The prompt still has to say what it means: one early run read "the test window is 01:00 to 06:00", decided it was too early, and waited three hours. It now says the window is open and to start right away.
- **Restoring production is idempotent and runs every time.** One script brings production back after every test, after a killed run, and again from a timer at the end of the window. It does nothing when production already answers.
- **Downloaded content is data, never instructions.** Model cards, file names and forum posts are written by strangers. They are read for facts only, no code from a model's repo is ever run, and builds that need `trust_remote_code` are refused. Code the model writes during the test runs in a sandbox without network or my files.
- **One task per run, with recorded state.** The agent follows a step-by-step procedure for one candidate at a time: how to start it, what to measure, where results go and how to write the verdict. It decides details such as memory settings, not the shape of the night. The candidate's status is recorded before the agent starts, and a run that ends without a result is marked failed rather than passed. The agent may run exactly the commands the procedure needs, and may not edit the production config, use admin rights or force-push. Stopping production is one named script, because a one-liner to do the same thing was blocked by the safety check, and that night's test never started.
- **A human decides, the agent reports.** I pick what gets downloaded, and the agent never switches production. Every result is a committed JSON file plus a write-up, every commit passes a leak check because the repo is public, and each run's final message is kept in a log.

Note that some of these limits rest on the agent's good behaviour (and as hope is not a strategy it is a security risk that I accept in this case). The procedure and the rule to treat inputs as data are instructions in its prompt, so they hold only as far as the model follows them. The command rules are enforced by Claude Code itself: the project's settings list the commands the agent may run without asking and deny a few outright, such as editing the production config, stopping services with admin rights and force-pushing. Anything else has to pass an automatic safety check. The deadline sits outside the agent entirely. The runner starts each session as its own process group, kills the whole group at the deadline and then restores production itself.

When something goes wrong, the agent says so and attempts to work around it within the allowed parameters. One benchmark ran past the agent's own time limit and was killed halfway. The agent took the finished sections from the log, re-ran the rest and noted it in the write-up.

## What came out of it

The incumbent is [Qwen3.8-27B Heretic Abliterated "RVN"](https://huggingface.co/0bserverx/Qwen3.8-27B-Heretic-Abliterated-Uncensored-GGUF) by 0bserverx: Alibaba's dense 27-billion-parameter Qwen3.8 with its refusals removed, run as the 15.5 GB IQ4\_XS file `RVN-IQ4_XS-multilingual-mtp.gguf`. It passes 283 of 291 graded runs and 41 of 45 in Estonian. Over the past two weeks the agent tested eighteen challengers against it. None replaced it, two never started and one did not fit.

| Model (links go to the weights) | Type | Graded runs passed (of 291) | Verdict |
| --- | --- | --- | --- |
| [Qwen3.8-27B Heretic Abliterated "RVN"](https://huggingface.co/0bserverx/Qwen3.8-27B-Heretic-Abliterated-Uncensored-GGUF), IQ4\_XS (production) | Dense 27B | 283 | The bar to beat |
| [Qwen3.8-27B Turbo-Brilliance 50X](https://huggingface.co/DavidAU/Qwen3.8-27B-Turbo-Brilliance-50X-Reasoning-Instruct-NEO-MAX-MTP-GGUF) by DavidAU, IQ4\_XS | Dense 27B | 284 | Rejected: 18% slower (38 tok/s), refused 1 of 11 |
| [Qwen3.8-27B Uncensored HauhauCS Aggressive](https://huggingface.co/HauhauCS/Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-MTP-GGUF), IQ4\_XS | Dense 27B | 282 | Rejected: a twin, not faster (43 tok/s) |
| [Same HauhauCS fine-tune as AWQ](https://huggingface.co/twolven/Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-AWQ-MTP), on vLLM | Dense 27B | 272 | Rejected: 10 runs below its GGUF twin, Estonian 34 of 45 |
| [Huihui Qwen3.8-27B Coder390 EfficientThink abliterated](https://huggingface.co/huihui-ai/Huihui-Qwen3.8-27B-Coder390-EfficientThink-abliterated-GGUF), IQ3\_S | Dense 27B, coding finetune | 273 | Rejected: weaker Estonian |
| [Qwen3.8-27B Coder390 EfficientThink](https://huggingface.co/nerkyor/Qwen3.8-27B-Coder390-EfficientThink-Opus5.5-GPT6Astra-Grok4.7-DSV4Pro-K3-SFT-RLOO-MTP-DFlash2) by nerkyor, Q6\_K | Dense 27B, coding finetune | 232 of 243 (hard cases not run), production 235 | Rejected: 25% slower, weaker Estonian |
| [Qwen3.8-27B Uncensored](https://huggingface.co/JonathanColetti/Qwen3.8-27B-Uncensored-GGUF) by JonathanColetti, IQ4\_XS | Dense 27B | 276 | Rejected: same speed, weaker Estonian (36 of 45) |
| [OrcaSAQ 2 Cyber 27B Uncensored](https://huggingface.co/orcarouter/OrcaSAQ-2-Cyber-27B-Uncensored-GGUF), IQ4\_XS | Dense 27B | 281 | Rejected: same speed, slightly weaker Estonian (39 of 45) |
| [Gemma 4 31B QAT Uncensored](https://huggingface.co/HauhauCS/Gemma4-31B-QAT-Uncensored-HauhauCS-Balanced-MTP), Q4\_K\_M | Dense 31B | About 275 (hard cases run once) | Rejected: fits only without its draft model, then 19 tok/s |
| [Muse Glimmer 30B Abliterated](https://huggingface.co/The-freezer/Muse-Glimmer-30B-Abliterated-GGUF), Q4\_K\_M | Dense 30B | 264 | Rejected, but strong Estonian (41 of 45) |
| [Ornith 1.5 35B-A3B Uncensored](https://huggingface.co/dealignai/Ornith-1.5-35B-A3B-UNCENSORED-GGUF), Q4\_K\_M | Mixture of experts, 3B active | 262 | Rejected: twice as fast, overthinks |
| [Underdog Saluki 27B 1.0 MTP Abliterated](https://huggingface.co/Kujira/Underdog-Saluki-27B-1.0-MTP-Abliterated-GGUF), IQ2 | Dense 27B | 253 | Rejected: half the size but not faster, weak Estonian (28 of 45) |
| [Gemma 4 26B-A4B QAT Uncensored](https://huggingface.co/HauhauCS/Gemma4-26B-A4B-QAT-Uncensored-HauhauCS-Balanced-MTP), Q4\_K\_M | Mixture of experts, 4B active | 251 | Rejected: ran out of tokens 20–22 times, drifts into Finnish |
| [Nemotron 3.5 Bolt 74B-A9B Abliterated](https://huggingface.co/BoldingBuilds/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-Abliterated-24GB-GGUF), mostly IQ2\_XXS | Mixture of experts, 9B active | 247 | Rejected: 2-bit only, Estonian 16 of 45 |
| [MiniCPM-V 4.7 35B-A3B Uncensored](https://huggingface.co/AtomicChat/MiniCPM-V-4.7-35B-A3B-Uncensored-GGUF), IQ4\_XS | Mixture of experts, 3B active | 241 | Rejected: 20 blank replies, best with reasoning off |
| [Nemotron 3.5 Bolt 74B-A9B CRACK](https://huggingface.co/dealignai/Nemotron-3.5-Bolt-74B-A9B-CRACK-24GB-GGUF), 2-bit | Mixture of experts, 9B active | 167 | Rejected: thinking never stops, 65 empty answers |

*Table 6. Production against every challenger.*

![Quality vs speed: graded runs passed without the Estonian cases against generation speed](/assets/img/llm-testing-overnight/chart.png)

*Figure 1. Quality vs speed. Estonian cases excluded; partial runs and near-duplicates left out.*

The chart leaves out the Estonian cases, so it shows general quality only. The three dense Qwen3.8-27B builds sit at the top, at 38–46 tok/s. The three models with 3–4B active parameters run about twice as fast but sit 6 to 14 points lower.

The only model to score higher was DavidAU's Qwen3.8-27B Turbo-Brilliance 50X, a reasoning finetune of the same base. It passed 284 graded runs against production's 283, 42 of 45 in Estonian against 41, and 118 of 141 with thinking off against 113. But it generated 38 tok/s against 46, and 53 against 65 on tool-call JSON, and as a build that is not uncensored it refused one of the 11 harmless edgy requests. One graded run is within the noise of three runs per case, so a fifth slower is not worth it.

The closest uncensored challenger was HauhauCS's Qwen3.8-27B Uncensored Aggressive, another uncensored build of the same base. It passed 282 of 291 graded runs and 40 of 45 in Estonian, one short on each. But it generated 43 tok/s against production's 46, so it was a twin rather than an upgrade, and a twin is not worth a switch.

Huihui's Coder390 EfficientThink, a community coding finetune of the same base, passed 273, and more than half of its misses were in Estonian. Its EfficientThink training does what the name promises: it writes shorter reasoning and never ran out of tokens, and its IQ3\_S quant needs 2.5 GB less memory. That does not make up for the quality gap.

The second coding finetune, Coder390 EfficientThink by nerkyor, is not abliterated, yet it answered all 11 edgy prompts. Outside the hard cases, which did not fit before morning, it passed 232 of 243 against production's 235. It was better at code (23 of 24 against 21) but weaker in Estonian (35 of 45 against 41). The only quant offered was the 23.2 GB Q6\_K, so it generated about a quarter slower than production and had to drop to 2 × 96k context.

Gemma 4 31B started once I dropped its draft model, and its answers came close: 233 of 243 outside the hard cases against production's 235, and 118 of 141 with reasoning off against 113. But without the draft model it generated only 19 tok/s and thought 1.6 times as many tokens, so the graded suite took nearly five times as long. It also needed 30.1 GB of the 32.

Muse Glimmer 30B passed 264 and Gemma 4 26B 251. Underdog Saluki 27B, an abliterated finetune of the same Qwen base, comes as a 2-bit IQ2 quant about half the size of production's file. It was not faster (44 tok/s against 46), passed 253 and managed only 28 of 45 in Estonian. Two more never loaded: Ternary Bonsai 2 27B and Offrail 27B ship in custom ternary tensor formats that llama.cpp cannot read.

The largest candidate did not fit at all. 0bserverx's RVN Qwen3.8 Flash-Next, a 125B mixture of experts with 6B active parameters, is 86 GB even at 3-bit (IQ3\_XS), against 32 GB of video memory plus 46 GB of system RAM. It started and answered correctly, but kept reading weights from the SSD: 1.5 tok/s generation and 2.3 tok/s prompt reading, against production's 46 and 1,213. The suite would have taken more than a day, so the agent stopped it and marked the model skipped.

[JonathanColetti's Qwen3.8-27B Uncensored](https://huggingface.co/JonathanColetti/Qwen3.8-27B-Uncensored-GGUF) shares production's base, IQ4\_XS quant and built-in MTP head, so its speed matched to the token. It passed 276, and five of its seven misses were Estonian grammar. NVIDIA's Nemotron 3.5 Bolt 74B-A9B, the leaderboard favourite from the start of this article, only comes in a 24 GB form that is mostly 2-bit. [BoldingBuilds' abliterated build](https://huggingface.co/BoldingBuilds/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-Abliterated-24GB-GGUF) was quick, 59 tok/s and 1,631 tok/s on a 30k prompt, but passed 247 and managed 16 of 45 in Estonian.

OrcaSAQ 2 Cyber, another uncensored build of the same base at the same quant, passed 281 at the same speed. It was slightly better at code and agent tasks and slightly weaker in Estonian (39 of 45).

A second abliteration of the same Nemotron, [dealignai's CRACK build](https://huggingface.co/dealignai/Nemotron-3.5-Bolt-74B-A9B-CRACK-24GB-GGUF), also 2-bit, did worse still. Its thinking would not stop: 65 graded runs hit the token cap with an empty answer, the reasoning circling on "Need produce final now". It passed 167, managed 8 of 45 in Estonian and answered only 4 of the 11 edgy prompts, never refusing but thinking until the cap. The suite burned 683k tokens against production's 150k.

The fast models were the most interesting losers. Ornith and MiniCPM generate two to three times faster than production. But Ornith often kept thinking until it hit the limit and never answered (8 runs), and MiniCPM often returned blank replies (20 runs). With reasoning switched off, MiniCPM became the best model I have tested on quick everyday tasks. That makes it a candidate for a second, fast helper model rather than a replacement.

## Switching the backend to vLLM

llama.cpp splits the model by layers, so the two GPUs take turns: one computes while the other waits. vLLM can split every layer across both cards (tensor parallelism), so they compute at the same time. It also has fast kernels for NVFP4, the 4-bit format that Blackwell cards such as the 5060 Ti run natively, and a scheduler built for serving many requests at once. On paper that means faster answers on the same hardware, which is why I tested it.

The catch is the weights. vLLM does not run GGUF files well, so it needs a 4-bit safetensors build (NVFP4, AWQ or GPTQ), usually already made by someone else from the same base model or fine-tune. Those formats quantize differently from llama.cpp's IQ4\_XS, so the weights are never one to one with the GGUF the agent tests. Production's own RVN build has no such version yet.

The vLLM test is now a regular step of the nightly run. When the download job fetches a GGUF candidate, it also looks on Hugging Face for a 4-bit safetensors build of the same model: NVFP4 first, then AWQ or GPTQ. A build counts only if its `base_model` points to that exact GGUF repo or the same fine-tune. A same-named NVFP4 of a different uncensored build does not count; the first version of the search fell for exactly that. The build also must not ship code, and its weights must stay under 22 GB.

A match gets queued as its own candidate and runs after the night's GGUF tests, when at least 95 minutes are left. It goes through its own procedure and is judged against production by the same suite. The agent can never switch production to it; that stays my decision. The interesting result is the same fine-tune on both backends, side by side.

The `vllm/vllm-openai:v0.32.0` image gets both RTX 5060 Ti cards with tensor parallelism; the GTX 1660 stays out. The flags mirror production where vLLM allows it: two 128k slots, an int4 KV cache, prefix caching, the Qwen3 reasoning and XML tool-call parsers, low reasoning effort by default, and the model's built-in MTP head drafting 3 tokens ahead. `--trust-remote-code` is never passed, so no code from a model repo runs.

```bash
docker run --rm --name vllm-test --runtime nvidia --gpus '"device=0,1"' --ipc=host \
  -e VLLM_API_KEY -p 127.0.0.1:8085:8000 -v "$MODEL_DIR:/model:ro" \
  vllm/vllm-openai:v0.32.0 /model \
  --tensor-parallel-size 2 --max-model-len 131072 --max-num-seqs 2 \
  --kv-cache-dtype int4_per_token_head --gpu-memory-utilization 0.92 \
  --enable-prefix-caching --enable-chunked-prefill --language-model-only \
  --reasoning-parser qwen3 --enable-auto-tool-choice --tool-call-parser qwen3_xml \
  --default-chat-template-kwargs '{"reasoning_effort": "low"}' \
  --speculative-config '{"method": "mtp", "num_speculative_tokens": 3}' \
  --linear-backend flashinfer_cutlass   # NVFP4 only; AWQ builds use vLLM's Marlin kernels
```

Split over the two 16 GB cards, 4-bit weights plus cache take about 30 GB, so it only just fits. A test stops production first, and while the cards are busy a small stand-in answers on production's port by forwarding requests to Claude. Startup takes 4.5 to 5.5 minutes, against seconds for llama.cpp.

|  | llama.cpp (production) | vLLM, stock NVFP4 | vLLM, Heretic ARA NVFP4 | vLLM, HauhauCS AWQ |
| --- | --- | --- | --- | --- |
| Context | 2 x 128k | 1 x 128k + 0.4 | 2 x 128k | 2 x 128k |
| Prose / tool JSON (tok/s) | 46.0 / 65.5 | 61.0 / 81.3 | 59.1 / 89.1 | 56.7 / 90.2 |
| Two requests at once (tok/s) | 69 | 120 | 116.6 | 97.5 |
| 30k prompt processing (tok/s) | 1213 | 1142 | 1161 | 752 |
| Follow-up time to first token (s) | 1.3 | 3.0 | 2.9 | 4.6 |
| VRAM (GB) | 25.5 | 27.3 | 30.2 | 30.1 |
| Graded / Estonian | 283/291, 41/45 | not run | 278/291, 37/45 | 272/291, 34/45 |
| Uncensored / thinking off | 11/11, 113/141 | not run | 11/11, 115/141 | 10/11, 117/141 |

*Table 7. llama.cpp vs vLLM on the same two cards.*

Two one-off experiments with Qwen3.8-27B in NVFP4 came first. NVIDIA's stock checkpoint measured speed only, and two 128k slots did not fit. Heretic ARA, an uncensored community checkpoint, ran with `--language-model-only`, which frees the vision tower's memory, and two full slots fit. The cards have no peer-to-peer link, so every layer syncs over PCIe, which eats part of the gain.

Heretic ARA was the fastest: 1.3 times production's prose speed and 1.7 times its total with two requests at once. But it passed 278 graded runs against production's 283 and 37 Estonian cases against 41, so it lost on quality across the board. It also took more than twice as long to start a follow-up reply, which matters more than raw speed in a chat, and needed about 5 GB more VRAM. The extra throughput would only pay off if the assistant often sent several requests at once.

The AWQ build answered the side-by-side llama.cpp vs vLLM question. [twolven's build](https://huggingface.co/twolven/Qwen3.8-27B-Uncensored-HauhauCS-Aggressive-AWQ-MTP) of the HauhauCS fine-tune had passed 282 as a GGUF on llama.cpp that afternoon. On vLLM it passed 272, and Estonian fell from 40 to 34 of 45, mostly on word forms. It generated faster, but read a 30k-token prompt at 752 tok/s against production's 1,213. Same model, same night: the conversion for vLLM cost ten graded runs. So production stays on llama.cpp, and vLLM stays in the nightly test until a build matches it on quality.

## A third engine: capy-flash

The newest candidate was not a model but an engine. capy-flash is a two-GPU tensor-parallel inference server for Qwen3.8-27B on RTX 50 cards. It runs its own `.ninfer` weight files, so it needs a build made for it: I used Feyd89's QUASAR-QAT NVFP4 artifact, 17.4 GB, stock Qwen3.8-27B with quantization-aware 4-bit projections and the MTP head included. Like the vLLM builds it is a separate candidate with its own procedure, and the agent again only measures; production is not switched. I started this test by hand at 04:01, outside the window, and production was down for an hour.

It came up on the first try at full defaults, in 68 seconds (vLLM took 4.5 to 5.5 minutes), with one 196,608-token context in 25.9 GB. The catch is reasoning. Production runs at low reasoning effort; capy-flash has no server default for it, so out of the box the model thinks in full. I ran the suite twice, once at the server default and once with `reasoning_effort: low` sent on every request. Only the second is comparable with production.

|  | llama.cpp (production) | capy-flash, effort low | capy-flash, default thinking | vLLM, Heretic ARA NVFP4 |
| --- | --- | --- | --- | --- |
| Context | 2 x 128k | 1 x 196k | 1 x 196k | 2 x 128k |
| Graded / Estonian | 283/291, 41/45 | 280/291, 40/45 | 272/291, 41/45 | 278/291, 37/45 |
| Uncensored / thinking off | 11/11, 113/141 | 10/11, 117 and 120/141 | 6/11, not run | 11/11, 115/141 |
| Graded suite (min) | 54.5 | 34.5 | 52.4 | not run |
| Prose / tool JSON (tok/s) | 46.0 / 65.5 | 77.0 / 121.5 | 78.3 / 120.0 | 59.1 / 89.1 |
| 30k prompt processing (tok/s) | 1213 | 2855 | 2833 | 1161 |
| Decode after 30k / 90k context (tok/s) | 39.0 / 26.5 | 78.3 / 68.8 | 72.2 / 76.3 | not run |
| Two requests at once (tok/s) | 69 | 148 | 88 | 116.6 |
| Follow-up time to first token (s) | 1.3 | 4.2 (2.2) | 4.2 | 2.9 |
| VRAM (GB) | 25.5 | 25.9 | 25.9 | 30.2 |

*Table 8. capy-flash vs llama.cpp and vLLM on the same two cards. The 148 tok/s and the 2.2 s follow-up come from `--max-concurrency 2`; the default profile serves one request at a time and queues the second.*

At low effort the quality is close to production: 280 graded runs against 283, and 40 Estonian cases against 41. The speed is the best I have measured on these cards. Prose generates 1.7 times faster than production, prompts are read 2.4 times faster, and decoding barely slows with long context: 69 tok/s after 90,000 tokens against production's 26.5. With two requests at once it reaches 148 tok/s, about 2.1 times production, and it beats vLLM on every speed row with 4 GB less VRAM. The graded suite finished in 34.5 minutes instead of 54.5.

At the server's default reasoning it would have been rejected. The model thinks until the 6,000-token cap: the code section falls from 21 to 12 of 24 runs, and five of the eleven uncensored questions run out of tokens rather than being refused. A live trial would need low effort on every request, or a thinking budget set on the server, which I have not tested.

The weak spot is chat latency. An alternating conversation's follow-up took 4.2 seconds to the first token against production's 1.3. The cause showed up in the engine's own cache dump: the follow-up matches the cached prompt for all 12,906 tokens up to the `<think>` tag and then diverges, because the chat template drops the earlier turn's thinking while the cache keeps it. The first follow-up of each conversation therefore re-reads the whole history. From the second one on, the cache hits at 99.5% and the first token arrives in 0.27 seconds. The weights are also stock Qwen3.8, not uncensored, and one question already ran out of tokens.

Verdict: tested, worth a live trial. It meets every point of the bar I set for it, and I will decide whether to switch. For a chat where answers are long, the speed pays off; where short follow-ups dominate, production's 1.3 seconds still wins.

## What I learned

- **Split the model, don't duplicate it.** On one card the 27B ran at 44 tok/s and read prompts at 806 tok/s. Split over two cards, generation stayed at 44 tok/s and prompt reading rose to 1,100 tok/s, because both cards share the work. The second card added context and faster prompts at no cost in generation speed.
- **Two slots beat one big one.** With a single 128k slot, every switch between conversations threw away the cached prompt. The next turn then re-read 8,000 to 22,000 tokens, about 9 seconds of dead time. Two slots let two conversations keep their caches, and a follow-up starts in about 1 second. The cost is sharing: when both slots are busy, each runs at about 38 tok/s instead of 47, or 69 tok/s together.
- **Quantize the cache before the context.** A 4-bit KV cache is what makes 2 × 128k fit. Shrinking the context is a worse trade, because Hermes needs at least 128k tokens and uses them.
- **The built-in draft head is free speed.** Multi-token prediction (MTP) took the 27B from 25 to 41 tok/s on one card, about 1.65 times faster, for 1.2 GB of extra memory. Adding n-gram drafting on top made it 15 to 20% slower: MTP already guessed nearly all the repetitive JSON, so the extra guesses only cost verification time.
- **Long context slows decoding.** With the same 400-word answer, production generated 45 tok/s after a short prompt, 39 tok/s after 30,000 tokens of context and 26.5 tok/s after 90,000. MTP acceptance barely moved (56%, 56%, 51%). Reading a fresh 90,000-token prompt takes about 95 seconds, which is one more reason to keep each conversation's cache.
- **Memory has hidden line items.** Weights and cache are not the whole bill. Each card also needs about 2.8 GB of compute buffer. Gemma 4 31B QAT Uncensored (Q4\_K\_M) looked like it would fit, but ran out of memory in five different configurations once its separate draft model was added. Without the draft model it fit, but generated only 19 tok/s.
- **Tokens per second is not everything.** The previous production model, Qwen3.6-35B-A3B heretic, a mixture of experts with 3B active parameters, generated 137 tok/s against the dense 27B's 40. After the switch the median real request went from 2.3 to 8.6 seconds, even though the 27B was more accurate. Short everyday turns reward raw speed.
- **Speed per token is not speed per answer.** Fast mixture-of-experts models wrote up to five times as many tokens as the dense incumbent, and Gemma 4 26B took twice as long to finish the suite. What matters is how long a real answer takes, and only live traffic shows that.
- **A faster engine is not free.** vLLM generated up to 1.4 times faster than llama.cpp, but it needed about 30 of the 32 GB, took 4.5 to 5.5 minutes to start instead of seconds, and was slower at what chats do most: starting a follow-up reply. A 4-bit conversion for it can also cost quality the GGUF of the same model does not lose.
- **"Uncensored" means different things.** One build still refused two of seven harmless questions. Another followed an instruction planted in a document every time, which rules it out for an agent that reads the web.

## What this is not

This is a personal acceptance test, not a general benchmark. The cases fit one assistant's work, partly in one specific and often unsupported language, on one pair of GPUs. For a broad look at Estonian, see the [Estonian Native LLM Benchmark](https://arxiv.org/abs/2510.21193) by Lillepalu and Alumäe (LREC 2026), which tests 32 models on native Estonian data; my Estonian cases are fewer and closer to an assistant's daily work. Three runs per case catch flaky answers, but they are too few for confidence intervals. Only one case uses a really long prompt, and I measure two users at once, not dozens.

The idea carries over, though. The test cases live in one file, so you can swap in your own and keep the loop, the code grading and the overnight agent. The benchmark talks to any OpenAI-compatible endpoint, and the serving side is three small commands you can wrap around llama.cpp or vLLM. Code, suite and results: [github.com/siimtolk/local-llm-eval](https://github.com/siimtolk/local-llm-eval).

The most useful part is not the scores. It is that a new model now costs me nothing but a morning message to read. If a release is good, I hear about it the next day. Otherwise the agent says why and moves on to the next one.

