---
layout: post
title: "One engine, two cards: testing capy-flash against llama.cpp and vLLM"
description: "capy-flash ran my Qwen3.8-27B suite 1.7 times faster than llama.cpp on two RTX 5060 Ti at nearly the same quality, but its follow-up replies start three times slower."
date: 2026-10-25
image: /assets/img/capy-flash-test/chart.png
tags: [Local LLM, LLM evaluation, Inference engines]
---

This weekend an engine called capy-flash showed up on r/LocalLLaMA: a two-GPU tensor-parallel server for Qwen3.8-27B on RTX 50 cards, built for NVFP4, the 4-bit format Blackwell cards run natively. The author tunes it on two RTX 5070 Ti. I have two RTX 5060 Ti, so I ran it through my own acceptance suite and compared it with my production llama.cpp and with vLLM. The suite, the grading and the overnight agent that runs it are described in [Automated nightly LLM testing](/articles/llm-testing-overnight/); this article only covers what capy-flash did. Code, suite and every result are in [local-llm-eval](https://github.com/siimtolk/local-llm-eval).

![Six panels comparing llama.cpp, capy-flash at low and default reasoning effort, and vLLM: graded runs passed, Estonian cases passed, prose speed, two requests at once, 30k prompt speed and follow-up latency](/assets/img/capy-flash-test/chart.png)

*Figure 1. capy-flash against llama.cpp and vLLM on the same two cards and the same suite. The two quality panels start at 240 and 30, not at zero.*

## What I tested

The suite is the one from the first article: 97 cases from what my assistant really does, each run three times and graded by code. [What the test measures](/articles/llm-testing-overnight/#what-the-test-measures) lists the areas and what a pass means, and [How much the models may think](/articles/llm-testing-overnight/#how-much-the-models-may-think) explains the reasoning-effort settings, which matter a lot for this engine. Production's numbers are from the same suite, run on production's current llama.cpp build.

capy-flash is an engine, not a model, so it needs weights in its own `.ninfer` format. I used Feyd89's QUASAR-QAT NVFP4 artifact: 17.4 GB, stock Qwen3.8-27B with quantization-aware 4-bit projections, the MTP draft head and vision included. It is not uncensored.

It came up on the first try at full defaults: 68 seconds to ready (vLLM took 4.5 to 5.5 minutes), one 196,608-token context with an INT8 KV cache, 25.9 GB of VRAM across both cards. The stock profile serves one request at a time; `--max-concurrency 2` makes two requests share the same pool.

The catch is reasoning. Production runs at low reasoning effort, but capy-flash has no server default for it, so out of the box the model thinks in full. A semver prompt took 5,955 tokens at the default and 1,607 with `reasoning_effort: low` in the request. I therefore ran the suite twice, and only the low-effort run is comparable with production. For the thinking-off passes the engine rejects an effort setting together with `enable_thinking: false`, so those requests send no effort.

## The numbers

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

*Table 1. capy-flash vs llama.cpp and vLLM on the same two cards. The 148 tok/s and the 2.2 s follow-up (the figure in brackets) come from `--max-concurrency 2`; the default profile queues the second request, which starts after 6.1 s.*

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

## What it showed

**Quality is close to production at low effort.** 280 graded runs against 283, 40 Estonian cases against 41. The misses were two remembered preferences in the agent section, one release name in extraction, two code cases (three runs out of tokens) and three Estonian cases. With thinking off the suite scored 117 and 120 of 141 in two runs, against production's 113, and Estonian was better too (33 against 29).

**The speed is the best I have measured on these cards.** Prose generates 1.7 times faster than production (77 against 46 tok/s) and tool-call JSON 1.9 times (121 against 65). A 30k-token prompt is read 2.4 times faster. Decoding barely slows with context: 78 tok/s after 30,000 tokens and 69 after 90,000, against production's 39 and 26.5. With two requests at once it reaches 148 tok/s, about 2.1 times production. It beats vLLM on every speed row with 4 GB less VRAM, and the graded suite finished in 34.5 minutes instead of 54.5.

**At the default reasoning it would be rejected.** The model thinks until the 6,000-token cap: the code section falls from 21 to 12 of 24 runs, and five of the eleven uncensored questions run out of tokens. Those are token-cap failures, not refusals. A live trial needs low effort on every request, or a thinking budget on the server, which I have not tested.

**The weak spot is chat latency.** In alternating 13k-token conversations, the first follow-up took 4.2 seconds to the first token against production's 1.3. The engine's own cache dump shows why: the follow-up matches the cached prompt for all 12,906 tokens up to the `<think>` tag and then diverges, because the chat template drops the earlier turn's thinking while the cache keeps it. The first follow-up of each conversation therefore reads the whole history again. From the second one on, the cache hits at 99.5% and the first token arrives in 0.27 seconds. `--preserve-thinking` changed nothing, because my benchmark does not send the reasoning back.

**It is not uncensored.** The weights are stock Qwen3.8. At low effort one of eleven questions (torrents) ran out of tokens, and I saw no outright refusal, but that is not guaranteed.

## Verdict

Tested, and worth a live trial. At low effort it meets every point of the bar I set for it: at least as many graded runs as the best vLLM build (280 against 278), Estonian at least 38 (40), clearly faster, and a context of at least 128k (196,608). The price is slower follow-ups, one request at a time unless concurrency is raised, and a model that is not uncensored. Where answers are long, the speed pays off; where short follow-ups dominate, production's 1.3 seconds still wins. I did not try DFlash2 speculative decoding, because this artifact ships no drafter, and the larger Flash-Next model needs more host RAM than my machine has.

This is the same kind of personal acceptance test as in the first article, not a general benchmark: one suite, one pair of GPUs, one request at a time unless noted. Results and the full journal entry are in [local-llm-eval](https://github.com/siimtolk/local-llm-eval).
