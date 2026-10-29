---
layout: post
title: "My production model is now Qwen3.8-27B QUASAR on capy-flash: 1.7 times faster, better on the extended tests"
description: "My assistant's production model moved from llama.cpp to capy-flash with Qwen3.8-27B QUASAR: 1.7 times faster at the same quality class, plus how I converted weights for capy-flash and extended the test cases."
date: 2026-10-29
tags: [Local LLM, LLM evaluation, Inference engines]
---

**In short: the new setup generates text 1.7 times faster (2.6 times at 90k context) and scores slightly higher on the extended test suite: 82.0% against 80.0% of all graded runs ([live results page](/results/)).**

Since 29 October 2026 my assistant's model runs on capy-flash v0.7.7 instead of llama.cpp. The [first capy-flash tests](/articles/capy-flash-test/) were run on the [core test suite](/articles/llm-testing-overnight/). Although capy-flash was much faster, its quality scores on the core suite gave no clear reason to switch immediately. To check the quality, I extended the suite with cases closer to my real use. This article only covers what changed since.

The extra sections are robust (prompt injection, tool errors, huge tool output, long tool lists, destructive commands), longagent (shell tasks of up to 30 turns, graded by the end state), estonian_free (free Estonian text) and vision (generated images). The own-traffic sections replay my assistant's transcripts: replay (its normal requests), cron_replay (its scheduled jobs) and ctx (long context).

Figure 1 compares the new (QUASAR) and old (RVN) production models in speed and in the share of passed runs per test-case section. On the core suite the old setup keeps a slight edge. Most sections end equal, and in agent, extract, Estonian and uncensored QUASAR is one run behind. The picture turns in the sections I added. QUASAR copes a bit better with prompt injection and messy tool output, finishes slightly more of the long shell tasks and writes better free Estonian. The clearest gain, and what makes the final case for switching, is on my own traffic: it passes more of my assistant's real requests. The margins are small, so I read it as: the new stack is at least as good at agent work, not clearly better. These runs are part of each setup's row on the [live results page](/results/).

And then, of course, it is clearly faster. The old production model, [RVN Qwen3.8-27B](https://huggingface.co/0bserverx/Qwen3.8-27B-Heretic-Abliterated-Uncensored-GGUF) in IQ4_XS on llama.cpp, writes 46 tokens per second. QUASAR on capy-flash writes 77 at the same quality. The graded suite takes 34.5 minutes instead of 54.5.

![Three speed numbers (writing speed, writing at 90k context, follow-up first token) and share of runs passed per test-case section for RVN on llama.cpp and QUASAR on capy-flash at low effort, grouped into core suite, extra sections and own traffic](/assets/img/new-leader-capy-quasar/test-cases.png)

*Figure 1. Top: speed, old → new, medians of three runs (follow-up: lower is better). Bottom: passed runs per section, with the counts; core suite has three runs per case.*

## Would my old RVN model be even better on capy-flash?

Before switching, I also tested my old production weights (RVN, uncensored) on capy-flash. capy-flash needs weights in its own `.ninfer` format. QUASAR comes prebuilt in it, RVN does not, so I converted it myself.

I used capy-flash's own converter, the `tools.convert` package in its source tree. It only needs CPU and takes about 5.5 minutes and 17 GB for the 27B. I took the BF16 weights, checked them tensor by tensor against their source, and quantized them to NVFP4. A recipe decides which layers keep a higher precision. I tried three:

- **A**, every projection NVFP4. The fastest (1.78 times production), but 94.3% of graded core runs (without Estonian) against production's 98.4%. It lost most on the hard and code sections.
- **B**, first and last two layers in FP8. Almost as fast (1.74 times) and 97.6%. It lost most on Estonian (38 of 45 against 41) and one uncensored case.
- **C**, Qwen's official mix (only the MLP of layers below 56 in NVFP4, attention and GDN in FP8). 96.7% at 1.57 times, but the best once Estonian and uncensored count: Estonian equal to production, uncensored 11/11. The downside is size: 20.9 GiB, so no 2 × 128k, and follow-ups are slower.

All three are on the [live results page](/results/). QUASAR still won. It is the only one ahead of production on my own traffic (replay and ctx, where C is behind). It runs at 1.7 times, close to A and B, and 2 × 128k fits. But it is stock weights, not uncensored.

Pitfalls if you want to repeat the conversion:

- Use the official Qwen `tokenizer.json` **and** `tokenizer_config.json`. The tokenizer from a quantized repo (lyf's Heretic ARA) died on `pre_tokenizer.Split`. With only one of the two files, the server died on a missing `added_tokens_decoder`.
- The converter refuses to overwrite an existing build. Before a rerun, move both the `.ninfer` file and its `.conversion.json` aside.
- Run the converter with `PYTHONPATH` set to the capy-flash source tree, not with `python -I`. Isolated mode hides the working directory, so Python does not find `tools.convert`.
- `/health` alone is not a start check. Check that the server process is still alive and send one real chat request.

## Running it in production: effort and thinking cap

The switch was small. The systemd unit that started llama.cpp now starts capy-flash on the same port with the same API key, so the clients did not change. The old unit stays next to it, and one script swaps them back.

The weights are Feyd89's [QUASAR-QAT NVFP4 build of Qwen3.8-27B](https://huggingface.co/Feyd89/Qwen3.8-27B-QUASAR-QAT-nvfp4-NInfer) in `.ninfer` format, converted from the [QUASAR-QAT checkpoint](https://huggingface.co/QUASAR-QAT/Qwen3.8-27B-QUASAR-NVFP4). They run as two parallel 128k-token lanes.

**QUASAR does best at low reasoning effort. In the capy tests it passed 280 of 291 core runs at `low` and 272 with full thinking, where code answers ran out of tokens. So production runs it at `low`. Getting there took two things that were not obvious.**

- **Reasoning effort.** capy-flash has no server-wide effort setting. The model thinks at `low` only when a request asks for it. Otherwise it thinks in full: in the first capy test, the code section dropped from 21 to 12 of 24. My assistant (Hermes) now sends `reasoning_effort: low` with every request. The server log shows `thinking low` for its requests, so a custom endpoint does receive it.
- **Clients that do not set an effort.** Open WebUI, the quiz and my scripts send none. For them the server has a cap, `--default-thinking-budget 1500`, the same budget as in the capy tests. It is a token limit, not the level `low`. A request without effort uses the default thinking template and is stopped at 1500 tokens. A request with `low` is capped too, but thinks shorter. In a quick check with the same question, the request without effort reasoned about twice as long. A small proxy that forces `low` on everything would be stricter. I did not need it.

One thing is missing: capy-flash has no `/metrics` endpoint, so the Grafana panel for llama.cpp's metrics is empty now. A small process reads the server log and writes the numbers for Prometheus instead. The dashboard still needs to be adapted.

## Trade-offs and open points

- After a long prompt, the first follow-up starts in about 4 seconds instead of 1.3. Later follow-ups are fast.
- QUASAR is not uncensored, but it still passed 10 of 11 uncensored cases, one fewer than production. RVN variant C is the fallback for that, with the context limit above.
- Production runs capy-flash v0.7.7, which fixes two engine-wide failures after a reasoning-close retry. The numbers here come from v0.7.4 and v0.7.5. v0.7.7 has only had a smoke test so far, not the full suite.

Code, suite and every result are in [local-llm-eval](https://github.com/siimtolk/local-llm-eval).
