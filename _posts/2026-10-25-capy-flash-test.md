---
layout: post
title: "capy-flash on two RTX 5060 Ti: 1.7 times faster than llama.cpp, with slower follow-ups"
description: "capy-flash ran my Qwen3.8-27B suite 1.7 times faster than llama.cpp on two RTX 5060 Ti at nearly the same quality, but its follow-up replies start three times slower."
date: 2026-10-25
image: /assets/img/capy-flash-test/chart.png
tags: [Local LLM, LLM evaluation, Inference engines]
---

This weekend an engine called capy-flash showed up on r/LocalLLaMA: a two-GPU tensor-parallel server for Qwen3.8-27B on RTX 50 cards, built for NVFP4, the 4-bit format Blackwell cards run natively. The author tunes it on two RTX 5070 Ti. I have two RTX 5060 Ti, so I ran it through my own acceptance suite and compared it with my production llama.cpp and with vLLM. The suite, the grading and the overnight agent that runs it are described in [Automated nightly LLM testing](/articles/llm-testing-overnight/); this article only covers what capy-flash did. Code, suite and every result are in [local-llm-eval](https://github.com/siimtolk/local-llm-eval).

![Six panels comparing llama.cpp, capy-flash at low and default reasoning effort, and vLLM: graded runs passed, Estonian cases passed, prose speed, two requests at once, 30k prompt speed and follow-up latency](/assets/img/capy-flash-test/chart.png)

*Figure 1. capy-flash against llama.cpp and vLLM on the same two cards and the same suite. The two quality panels start at 240 and 30, not at zero.*

## What capy-flash is

capy-flash is not a general serving engine like vLLM or llama.cpp. It is a purpose-built server for two models, Qwen3.8-27B (dense) and Qwen3.8-Flash-Next (a 125B mixture of experts, with the experts that do not fit in VRAM computed on the CPU). It runs them on one or two 16 GB RTX 50-series (Blackwell) cards under Linux, and it exposes an OpenAI- and Anthropic-compatible API, so tools such as Claude Code and Codex CLI can talk to it directly. The code is Apache-2.0.

Its lineage explains the differences. It began as ninfer-tp2, a fork of [NInfer](https://github.com/Neroued/ninfer), a from-scratch C++/CUDA engine written for a single RTX 5090 that explicitly leaves multi-GPU out of scope. The fork added tensor parallelism over two cards, and it works without peer-to-peer copies: my 5060 Ti pair has no P2P link, and the engine stages the transfers through host memory. It was renamed capy-flash at v0.5.0, and the author says very little of NInfer is left. The binaries still carry the old `ninfer` prefix.

What that means in practice, compared with vLLM:

- **Weights come in its own format.** vLLM loads ordinary safetensors builds (NVFP4, AWQ, GPTQ) of many models. capy-flash runs `.ninfer` artifacts that someone has converted for it, such as Feyd89's QUASAR-QAT build below, so the list of usable models is short.
- **It is tuned for one box.** The author builds and measures everything on two RTX 5070 Ti, and the README says the engine is validated there without P2P. My 5060 Ti numbers are a different card, and the author's announcement says other people have posted results from other cards in the repository's issues.
- **Concurrency is a setting, and the stock profile starts at one.** The documented profiles run with `--max-concurrency 1`, so a second request waits, but the server accepts up to eight lanes sharing one KV pool, and two lanes ran fine here (see the results below). vLLM's scheduler was designed around many simultaneous requests; here the lane count is something you set and size the cache for.
- **It keeps conversations in memory.** A prefix cache works across turns, and idle conversations move to host RAM and optionally to disk. This is what the follow-up numbers below depend on.

The project does not compare itself with vLLM or llama.cpp; its own benchmarks are against Strata, another engine for the same cards. The comparison with vLLM in this article is mine, from the same suite and the same two cards. Sources: the [capy-flash repository](https://github.com/ValerioDolci/capy-flash) and its [announcement on r/LocalLLaMA](https://www.reddit.com/r/LocalLLaMA/comments/1xe9yyg/capyflash_v074_where_my_twogpu_qwen38_engine_is/).

## What I tested

The suite is the one from the first article: 97 cases from what my assistant really does, each run three times and graded by code. [What the test measures](/articles/llm-testing-overnight/#what-the-test-measures) lists the areas and what a pass means, and [How much the models may think](/articles/llm-testing-overnight/#how-much-the-models-may-think) explains the reasoning-effort settings, which matter a lot for this engine. Production's numbers are from the same suite, run on production's current llama.cpp build.

The weights are Feyd89's QUASAR-QAT NVFP4 build: stock Qwen3.8-27B with quantization-aware 4-bit projections, the MTP draft head and vision included. It is not uncensored.

Versions and setup:

- capy-flash v0.7.4 (commit 3193faf, prebuilt Linux x86_64 binaries), profile `27b` through `serve-tp2.sh`, MTP with 3 draft tokens, INT8 KV cache. The repository has since moved on to v0.7.5.
- Weights: Feyd89/Qwen3.8-27B-QUASAR-QAT-nvfp4-NInfer (`.ninfer` v3, 17.4 GB).
- Production baseline: llama.cpp build 11707 (commit 1268edac5), RVN Qwen3.8-27B in IQ4_XS with MTP, two 128k slots, low reasoning effort.
- vLLM column: `vllm/vllm-openai:v0.32.0` with Heretic ARA NVFP4, tested on 2026-10-21 with the same suite.
- Machine: two RTX 5060 Ti 16 GB (the GTX 1660 SUPER in the box is left out), NVIDIA driver 615.71.09, Fedora 44, kernel 7.2.7. The capy-flash binaries are built for Ubuntu 26.04 and ran on Fedora without trouble.
- Dates: default-thinking run on 2026-10-24 from 19:22, low-effort and speed runs on 2026-10-25 from 04:01. The suite is [local-llm-eval](https://github.com/siimtolk/local-llm-eval).

It came up on the first try at full defaults: 68 seconds to ready (vLLM took 4.5 to 5.5 minutes), one 196,608-token context with an INT8 KV cache, 25.9 GB of VRAM across both cards. The stock profile serves one request at a time; `--max-concurrency 2` makes two requests share the same pool.

The catch is reasoning. Production runs at low reasoning effort, but capy-flash has no server default for it, so out of the box the model thinks in full. A semver prompt took 5,955 tokens at the default and 1,607 with `reasoning_effort: low` in the request. I therefore ran the suite twice, and only the low-effort run is comparable with production. For the thinking-off passes the engine rejects an effort setting together with `enable_thinking: false`, so those requests send no effort.

## The numbers

|  | llama.cpp (production) | capy-flash, effort low | capy-flash, effort low, 2 lanes, 196k | capy-flash, effort low, 2 lanes, 256k | capy-flash, default thinking | vLLM, Heretic ARA NVFP4 |
| --- | --- | --- | --- | --- | --- | --- |
| Context | 2 x 128k | 1 lane, 196k | 2 lanes, 196k pool | 2 lanes, 256k pool | 1 lane, 196k | 2 x 128k |
| Graded / Estonian | 283/291, 41/45 | 280/291, 40/45 | not run | not run | 272/291, 41/45 | 278/291, 37/45 |
| Uncensored / thinking off | 11/11, 113/141 | 10/11, 117 and 120/141 | not run | not run | 6/11, not run | 11/11, 115/141 |
| Graded suite (min) | 54.5 | 34.5 | not run | not run | 52.4 | not run |
| Prose / tool JSON (tok/s) | 46.0 / 65.5 | 77.0 / 121.5 | 82.2 / not run | 78.3 / 121.3 | 78.3 / 120.0 | 59.1 / 89.1 |
| 30k prompt processing (tok/s) | 1213 | 2855 | not run | 2851 | 2833 | 1161 |
| Decode after 30k / 90k context (tok/s) | 39.0 / 26.5 | 78.3 / 68.8 | not run | 76.0 / 68.2 | 72.2 / 76.3 | not run |
| Two requests at once (tok/s) | 69 | 88 (the second waits) | 147.8 | 145.4 | 88 (the second waits) | 116.6 |
| Follow-up time to first token (s) | 1.3 | 4.2 | 2.2 | 4.2 | 4.2 | 2.9 |
| VRAM (GB) | 25.5 | 25.9 | 25.9 | 28.3 | 25.9 | 30.2 |

*Table 1. capy-flash vs llama.cpp and vLLM on the same two cards. The 2-lane columns use `--max-concurrency 2` and ran the speed part only, so their quality rows are empty; the 256k column has the same shape as production's two 128k slots. The stock profile has one lane, so a second request starts after 6.1 s.*

Per section of the graded suite (passed of runs):

| Section | llama.cpp (production) | capy-flash, effort low | capy-flash, default thinking |
| --- | --- | --- | --- |
| Tools | 42/42 | 42/42 | 42/42 |
| Agent | 47/48 | 46/48 | 48/48 |
| Reasoning | 30/30 | 30/30 | 30/30 |
| Extraction | 15/15 | 14/15 | 13/15 |
| Code | 21/24 | 21/24 | 12/24 |
| Hard | 48/48 | 48/48 | 47/48 |
| Instruction following | 21/21 | 21/21 | 21/21 |
| Judge | 18/18 | 18/18 | 18/18 |
| Estonian | 41/45 | 40/45 | 41/45 |

*Table 2. Graded sections. Out-of-tokens failures: 2 for production, 3 at effort low, 14 at default thinking.*

## Production's setup: two lanes in a 256k pool

Production runs two independent 128k slots, so I repeated the speed part with the same shape: `--max-context 262144 --max-concurrency 2`, a 256k pool for two lanes. It started on the first try, with 1.63 GiB free per card (2.81 GiB at 196k) and 28.3 GB of VRAM in total instead of 25.9.

In the two 2-lane columns of Table 1 the speed carries over: 145 tok/s with two requests at once, 2.1 times production, and the long-context decoding is unchanged. So the engine can serve the same two-conversation shape as production. The follow-up time did not improve. With the 196k pool the follow-ups averaged 2.2 s, and with 256k they took 4.2 s, the same as with one lane. I ran this once, so I cannot say whether the difference is noise or an effect of the larger pool; the first-follow-up cache miss explained above is the likely cause either way.

## What it showed

**Quality is close to production at low effort.** 280 graded runs against 283, 40 Estonian cases against 41. The misses were two remembered preferences in the agent section, one release name in extraction, two code cases (three runs out of tokens) and three Estonian cases. With thinking off the suite scored 117 and 120 of 141 in two runs, against production's 113, and Estonian was better too (33 against 29).

**The speed is the best I have measured on these cards.** Prose generates 1.7 times faster than production (77 against 46 tok/s) and tool-call JSON 1.9 times (121 against 65). A 30k-token prompt is read 2.4 times faster. Decoding barely slows with context: 78 tok/s after 30,000 tokens and 69 after 90,000, against production's 39 and 26.5. With two requests at once it reaches 148 tok/s, about 2.1 times production. It beats vLLM on every speed row with 4 GB less VRAM, and the graded suite finished in 34.5 minutes instead of 54.5.

**At the default reasoning it would be rejected.** The model thinks until the 6,000-token cap: the code section falls from 21 to 12 of 24 runs, and five of the eleven uncensored questions run out of tokens. Those are token-cap failures, not refusals. A live trial needs low effort on every request, or a thinking budget on the server, which I have not tested.

**The weak spot is chat latency.** In alternating 13k-token conversations, the first follow-up took 4.2 seconds to the first token against production's 1.3. The engine's own cache dump shows why: the follow-up matches the cached prompt for all 12,906 tokens up to the `<think>` tag and then diverges, because the chat template drops the earlier turn's thinking while the cache keeps it. The first follow-up of each conversation therefore reads the whole history again. From the second one on, the cache hits at 99.5% and the first token arrives in 0.27 seconds. `--preserve-thinking` changed nothing, because my benchmark does not send the reasoning back.

**It is not uncensored.** The weights are stock Qwen3.8. At low effort one of eleven questions (torrents) ran out of tokens, and I saw no outright refusal, but that is not guaranteed.

## Verdict

Tested, and worth a live trial. At low effort it meets every point of the bar I set for it: at least as many graded runs as the best vLLM build (280 against 278), Estonian at least 38 (40), clearly faster, and a context of at least 128k (196,608). The price is slower follow-ups, a lane count you have to set yourself (the stock profile queues a second request), and a model that is not uncensored. Where answers are long, the speed pays off; where short follow-ups dominate, production's 1.3 seconds still wins. I did not try DFlash2 speculative decoding, because this artifact ships no drafter, and the larger Flash-Next model needs more host RAM than my machine has.

This is the same kind of personal acceptance test as in the first article, not a general benchmark: one suite, one pair of GPUs, one request at a time unless noted. Results and the full journal entry are in [local-llm-eval](https://github.com/siimtolk/local-llm-eval).
