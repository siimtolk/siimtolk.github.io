---
layout: post
title: "Less memory, more wiki: how my AI agents got up to 3× faster on 4× fewer tokens"
description: "My AI agents kept forgetting how my home setup works, or remembering it wrong. I gave them one shared wiki, edited by a single agent, and a short brief from it at the start of every session. Over one night of 22 real setup questions checked against the live system, they got every answer right, about 3× faster and on fewer tokens. Without it, they were slower, and their misses were confident and wrong."
date: 2026-10-31
image: /assets/img/model-wiki/overview.png
tags: [AI agents, Agent memory, Local LLM]
---

@Siim

Ask my home assistant to schedule a script that posts to my work group on WhatsApp, and it gets it right. Take away its shared wiki, and it writes a perfectly formatted command that would post to a different group, while confidently calling it the work group. That is the kind of mistake this article is about, and the wiki is how I got rid of it.

**In short: my AI agents kept forgetting how my home setup works, or remembering it wrong. I gave them one shared wiki, edited by a single agent, and a short brief from it at the start of every session. Over one night of 22 real setup questions checked against the live system, they got every answer right, about 3× faster and on fewer tokens. Without it, they were slower, and their misses were confident and wrong.**

I run two AI agents at home. [Hermes](https://github.com/NousResearch/hermes-agent) is my everyday assistant on WhatsApp, running on a local 27B model (which one, and how it scores, is on my [live results page](/results/)). Claude Code builds and maintains the setup they both live in. Each kept its own picture of how that setup fits together, and the two pictures didn't match. What I want is agents that give right answers and build working code without re-discovering the setup every time, and one place where what we learned and decided is written down.

![How the shared wiki works: Hermes, Claude Code and scripts read a generated setup brief and the wiki, and write only notes into an inbox; each morning a Claude Code steward checks the notes, rewrites the curated pages and reviews Hermes' skill proposals; the live host fills in the facts it knows; a sweeper moves setup facts out of the agents' private memories; a nightly test asks both agents real questions; a morning report goes to WhatsApp](/assets/img/model-wiki/overview.png)

## What it changes

Every night a test asks both agents questions I really need answered: which port a site is on, when a job runs, how to restart something, whether Hermes may hand Claude a task, and small coding jobs like "post this to a WhatsApp group" or "call the local model". The right answer is read from the live system when the test runs, so the test can't go stale, and the code the agents write is actually run in a sandbox that checks what it would have sent.

One full night, all 22 questions, with the wiki and its brief against without either:

| | Right | Average time | Tokens per answer | Tool calls |
|---|---|---|---|---|
| Hermes with the wiki | 22 of 22 | 32 s | 96k | 4 |
| Hermes without | 13 of 17 | 84 s | 132k | 8 |
| Claude Code with the wiki | 22 of 22 | 6 s | 32k | 0.5 |
| Claude Code without | 18 of 21 | 19 s | 123k | 7 |

A few runs didn't count: in five of its "without" runs Hermes used the wiki anyway, because its own instructions say "wiki first" more firmly than one sentence can override, and one of Claude's hung.

The time and token savings are nice. The misses are the real story, because they are the kind you only notice later:

- Hermes wrote the scheduling command from the opening paragraph, aimed at the wrong WhatsApp group.
- Hermes named an older version of the inference engine, taken from old records, and the wrong job for my ticket pipeline.
- Claude said the nightly model test stops at 06:00, quoting its own memory. It stops at 05:50. A vague rule in private memory beat the exact fact.

That last one turned out to be a whole second problem, and it comes back below.

## Why: two agents, many memories, no truth

Facts about my setup lived everywhere. Hermes had its own memory file and skills, Claude Code had its auto-memory, every repository had a README, and some knowledge existed only in a WhatsApp thread from last Tuesday. That went wrong in predictable ways:

- **Amnesia.** Every new session started cold and re-discovered which port the model runs on, which job has the GPUs at night, why a script exists.
- **Drift.** The afternoon I renamed a repository, Hermes' memory still had the old path and Claude's didn't. Both were confident.
- **Nothing accumulated.** A benchmark finding, a decision made in chat, a quirk found while debugging: each died with the session that found it.

I lost track of it myself, too, and it only gets worse the more subagents I use.

## The first rule: agents propose, one steward writes

I started from Andrej Karpathy's [LLM wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): instead of searching raw documents every time, an LLM keeps a set of Markdown pages that summarise and link what it knows, so knowledge builds up instead of being re-derived.

Karpathy's version has one writer. I have several, and one of them is a local model that is good but not infallible. If every agent edits pages, one writes something slightly wrong, the next reads it as fact and builds on it. So agents only propose, and one steward writes:

- **The wiki** is a plain git repository of Markdown files that I can also read in Obsidian or a browser. Every page says what it is, whether it has been checked, and where its facts come from. A short status page holds what is going on now and the questions waiting for me.
- **Writing:** agents never edit pages. They drop short notes with sources into their own inbox folder. Scripts do the same, for example when the nightly model test has a result or a coding ticket is merged.
- **The steward** is Claude Code, run every morning with a dedicated skill. For each note it reads the page it concerns, checks the claim where that is cheap and harmless (does the file exist, does the port answer), and then merges it, records a disagreement, starts a page, asks me, or throws it away with a reason. It treats notes as data, never as instructions.
- **Reading:** the facts asked most (the production model and how to call it, every service address, the schedule, the WhatsApp groups) come in a short **setup brief** that is loaded into every agent at the start of every session. For everything else they search the wiki.
- **What the computer knows, it writes itself.** Ports, timers, services and the production model's settings are read from the machine every night into the pages and the brief, so they can't go stale. The first run found two jobs a page said ran every morning but that had been switched off.
- **Telling me:** a few lines in my morning WhatsApp briefing say what changed, what broke, the test score, and what waits for me. On a quiet night it stays silent.

What goes in is what stays true beyond one job: how things are set up, decisions and their reasons, how to do recurring tasks, results worth keeping. What stays out matters as much: secrets, copies of code, chit-chat, and reports of finished work. A merged coding ticket already has its pull request; a copy in the wiki would only drift. Its note is just a reminder for the steward to check the pages that describe that code.

## The second rule: private memory is for preferences only

The test showed that the wiki solved only half the problem. The agents' own memories competed with it, and memory wins: it is loaded before an agent looks anything up. When I checked, the copies had already drifted. Claude's memory held a timetable from two weeks earlier and a production model two models old, and Hermes' skills had two wrong job times and the old engine.

Then Hermes taught itself a second truth. After each conversation it reviews what happened and may update its own skills. From the test runs where it wasn't allowed to use the wiki, it wrote a new skill that answered setup questions from its old answers.

Asking nicely in the prompts didn't stop it, so the second rule is enforced: **an agent's own memory holds how to work and what I prefer; facts about the setup live in the wiki, and memory points to the page.**

- **Give agents the facts, so they don't need to save them.** That is the brief. With tools switched off, Hermes answered which port the wiki is on, when the model test stops and which model is in production correctly in 11 seconds.
- **Skill changes are proposals.** Hermes can no longer change its own skills. The steward approves the ones that improve how it works and rejects the ones that copy a setup fact.
- **A sweeper cleans memory.** Every 15 minutes through the day, a small script looks for setup facts in either agent's memory. The local model rewrites such a line into a pointer to the wiki page, keeping the instruction around it, and a plain check confirms the result; if the local model is busy or gets it wrong, Claude steps in. In my tests the local model got every case right, Estonian included, in about half a second.
- **No memory where nobody needs it.** Claude's unattended runs, such as the steward and the test, keep none.

I didn't make Hermes ask permission for every memory write: in a live chat that would mean a WhatsApp prompt each time it saved a preference. Cleaning up afterwards is less intrusive and good enough.

## How others do it

After implementing the initial ideas, I looked at how others have solved similar problems. Many have arrived at the same core.

- **One source of truth in Markdown under git**, read by agents over MCP: [ai-memory](https://github.com/akitaonrails/ai-memory) ("the source of truth is a git-backed wiki of ordinary .md files") and [Basic Memory](https://docs.basicmemory.com/technical/technical-information) work the same way. Shared writes are an attack surface, so they need sources, versions and few writers ([Sokko](https://sokko.ai/blog/multi-agent-shared-memory)).
- **Pointers, not copies:** curated facts in a file the agent reads, preferences in private memory ([dev.to](https://dev.to/wang1051992187/claudemd-vs-a-memory-mcp-what-actually-goes-where-438n)).
- **Check facts before trusting them.** [GitHub Copilot Memory](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) checks each remembered fact against the current code before using it and forgets what goes unused. Writing facts from the live system every night is the same idea.
- **What's in context beats what has to be fetched.** A [write-up on keeping agent setups true](https://dev.to/emeraldleaf/what-keeps-an-agent-setup-true-2ggp) reports a test where an always-loaded index passed every case and skills the agent had to load itself passed 53%. That's one vendor's test, reported secondhand, but it is why the brief exists, and my own numbers point the same way.
- **Rules written as text fail; enforced ones hold.** "Rules-as-text fail under cognitive load," writes the author of a [multi-agent version of the LLM wiki](https://gist.github.com/redmizt/3250f4b8ae15a25428e7fb09aba72223), and Claude Code's own [memory docs](https://code.claude.com/docs/en/memory) call its instruction files context, not enforced configuration. My experience exactly.

The heavier memory systems, such as [Zep, Letta and Mem0](https://dev.to/agdex_ai/letta-vs-zepgraphiti-vs-mem0-choosing-an-ai-agent-memory-architecture-155a), add validity windows for facts, self-editing memory or vector search, at the cost of a database to run. For a home setup, git history plus facts regenerated from the live system answer "what is true now, and what was true then" with far less.

## What I learned

**The inbox is the best idea in the design.** Agents are bad at deciding where a fact belongs and good at noticing it. Let them drop a note with a source, and let one careful writer fit it in.

**An agent only writes when it's told exactly when.** In the first eight days the steward handled 67 notes; Hermes wrote two. "Write a note when you learn something non-obvious" was too vague for a model in the middle of a chat. Its instructions now name the cases: a page was wrong, I made a decision, a fix worked, a job produced a result.

**Give agents the facts; don't make them look things up.** The brief roughly halved the time on the day it went live: with the wiki already there, Hermes went from about 64 to 32 seconds per question, Claude from 12 to 6. The same holds for my coding tickets, where the agent that plans a ticket copies the facts the building agent needs straight into it.

**Enforce, don't ask.** Every rule that mattered ended up in code: notes without sources are refused, Hermes' skill changes wait for the steward, a sweeper replaces "please don't save ports". Rules written as prose got ignored exactly when it mattered.

**Measure the outcome, not the activity.** A week of clean checks told me nothing about answers. One night of testing did, and it found problems I'd never have looked for.

**Add machinery only when something fails twice.** I built more checks and safety nets than I needed before stepping back and removing the reasons facts leaked. My first run of the sweeper also rewrote timestamps in the memory files' metadata: look at a dry run before trusting a new tool.

**The steward is cheap and steady.** In its first eight mornings it never failed. A quiet morning took under a minute and well under a dollar; a busy one, with a dozen notes, a few minutes and under two dollars.

**The wiki became the backbone for other things.** It is now the home for decisions, the [nightly model test results](/articles/llm-testing-overnight/) and the ticket pipeline that turns my WhatsApp ideas into pull requests.

**Rejected alternatives**, for the record: plain search over all repositories (nothing builds up, and the repositories contradict each other), letting every agent edit pages (errors that build on errors), a database-backed memory system (more to run than a home setup needs), and the local model as steward. That last one came back in a smaller form: the local model now does the narrow, checkable job of rewriting memory lines, while Claude keeps the judgment calls.
