---
layout: post
title: "Deploying and Optimizing AI Inference at Scale"
description: "I spent the first day of GTC Berlin in NVIDIA's hands-on inference workshop, each of us with our own Kubernetes cluster and 4x A100. What I took home: the Gateway API instead of Ingress, rate limits by tokens, retries and canaries for model updates, autoscaling on queue depth instead of GPU use, and when disaggregated serving with Dynamo pays off."
date: 2026-11-04
tags: [Conferences, Inference engines, Kubernetes]
---

GTC Berlin started for me a day before the keynote, with the full-day workshop Deploying and Optimizing AI Inference at
Scale ([NVIDIA training](https://www.nvidia.com/en-eu/gtc/training/)). I had already moved GPU servers to
[Dynamo](https://github.com/ai-dynamo/dynamo) on Kubernetes and set up several inference models in production, so I
expected a recap. It was more than that: the most useful day of the conference for me. The rest of the conference is in my [GTC Berlin 2026 overview](/articles/gtc-berlin-2026/).

## How the day ran

Two NVIDIA solutions architects, Amit Kushwaha and Yesika Ramirez Cardenas, ran the workshop on 20 October (session
BW61086), 09:00-16:00. There were three blocks, each a short lecture followed by a lab, so most of the day was
hands-on. Every participant got their own Kubernetes cluster: a GPU node with 4x A100 80GB, with the NIM and Dynamo
operators, Envoy Gateway, Prometheus, Grafana and the DCGM exporter already installed. The labs were 14 Jupyter
notebooks with about 270 code cells, and they had us deploy real models, put load on them and measure the results
ourselves.

## The labs

**Block 1: one model, then several.** We deployed gpt-oss-20b, a mixture-of-experts model, as a NIM service and timed
its cold start. Then we tried the API settings that change speed and cost: reasoning effort, streaming and time to
first token, temperature, max tokens and JSON mode. Next came Qwen3-32B in BF16 and FP8 side by side, benchmarked with
AIPerf. In my run of 240 requests each, FP8 cut the average time between tokens from 61 to 42 ms, raised the speed per
user from 16.4 to 24.5 tokens per second, and raised total throughput from 639 to 719 tokens per second. The block
closed with Prometheus queries and Grafana dashboards for the vLLM metrics.

**Block 2: a gateway in front, then autoscaling.** We put all three models behind one Envoy AI Gateway endpoint, routed by model
name. On top of that we added API keys, token-based rate limits, token usage tracking, retries, circuit breakers and a
canary rollout. Then we freed the GPUs and let KEDA scale a NIM service on its request metrics, with a load test to
watch it scale up.

**Block 3: Dynamo.** We served Qwen2.5-32B-Instruct (GPTQ Int4) with vLLM under Dynamo, first aggregated, then with the
KV-cache-aware router, then disaggregated, with prefill and decode on separate GPU pairs linked by NVLink. We
benchmarked each setup. An optional notebook showed how Grove gang-schedules the prefill and decode pods.

**Assessment.** The certificate needed a model we had not used, Qwen3-1.7B. We had to write the Dynamo deployment
config, deploy it aggregated and disaggregated, benchmark both with AIPerf, try two prefill/decode replica ratios and
recommend one. There were 7 exercises worth 10 points, and we had to complete all of them to pass.

## What I took home

### Gateway, not Ingress

Ingress is legacy: Kubernetes has frozen the Ingress API and recommends the Gateway API
instead ([Kubernetes docs](https://kubernetes.io/docs/concepts/services-networking/ingress/)). Ingress is one resource
with host and path rules; anything more, like matching on headers or splitting traffic by weight, needs
controller-specific annotations. The [Gateway API](https://gateway-api.sigs.k8s.io/docs/) builds those into the spec
and splits the setup by role: a GatewayClass picks the implementation, a Gateway is the entry point the cluster
operator owns, and HTTPRoutes are the routing rules each team writes for its own services. So without any annotations
you can split traffic by weight for a canary, route on headers or query parameters, redirect and rewrite headers,
route gRPC as well as HTTP, and let teams in different namespaces attach their own routes to one shared Gateway
([migrating from Ingress](https://gateway-api.sigs.k8s.io/guides/getting-started/migrating-from-ingress/)).

The API is only a spec; [Envoy Gateway](https://gateway.envoyproxy.io/) is one implementation of it. It runs Envoy Proxy
for you, so you get Envoy as the cluster's gateway without learning Envoy's own config, and it extends the API with
Envoy features such as rate limiting. The Envoy AI Gateway sits on top and adds what LLM serving needs: API keys,
token limits and routing by model name behind one endpoint. Rate-limit by tokens, not requests: one 10k-token request
costs more than ten small ones. The AI Gateway counts the tokens, but Envoy Gateway enforces the limit with its global rate limit, so it needs rate limiting turned on
and a Redis behind it
([Envoy AI Gateway docs](https://github.com/envoyproxy/ai-gateway/blob/main/site/docs/capabilities/traffic/usage-based-ratelimiting.md)).

### Retries and rollouts

A 5xx error or a dropped connection means the server failed, so the same request may work
on a second try. Retry it, and wait longer after each try (exponential backoff) so the retries don't pile onto a
server that is already struggling. A 4xx error means the request itself is wrong, like a bad key or bad input, so a
retry gets the same answer: never retry those. A retry also needs somewhere to go, so run 2+ replicas, and a circuit
breaker stops sending traffic to a replica that keeps failing, so it gets time to recover. In my setup, when a model
container dies, the new one comes up but the user's request is lost; an auto-retry would spare them that.

Two tools make model updates safer. A canary release sends a small share of traffic to the new model version first.
Request mirroring sends a copy of real traffic to the new version and throws its answers away. Either way you see how
a new model behaves before users depend on it.

### Autoscale on the right metric

Out of the box, HPA scales on CPU or memory; KEDA can scale on any metric, such as a Prometheus query. Scale on
`vllm:num_requests_running` or queue depth, not DCGM GPU utilisation, which can map GPUs to the wrong pods on multi-GPU
nodes and show 0% under load. Scale down slowly (about 10 minutes): NIM pods take minutes to load a model.

### Aggregated or disaggregated

A request is served in two phases. Prefill reads the whole prompt at once and builds
the KV cache, the model's working memory for that prompt; it is heavy on compute. Decode then writes the answer one
token at a time from that cache; it is limited by memory bandwidth. Aggregated serving runs both on the same GPUs.
That is simpler and fine for dev and moderate traffic. Disaggregated serving puts them on separate GPU pools and
hands the KV cache from one to the other (in the lab, over NVLink), so each pool can be sized for its own phase. A
KV-cache-aware router sends a request to the worker that already holds the cache for the start of its prompt, so that
part of the prefill is skipped ([Dynamo architecture](https://docs.nvidia.com/dynamo/knowledge-base/concepts/architecture.md)).

Splitting plus the router pays off at high traffic with repeated prompts: chat, where every turn sends the whole
conversation again, and RAG, where many requests share the same system prompt. Dynamo's Planner watches the load and
changes how many prefill and decode workers run. ModelExpress makes new workers start faster: instead of loading the
weights from storage, a new worker copies them from one that is already running
([ModelExpress docs](https://docs.nvidia.com/dynamo/v1.3.0/kubernetes-deployment/model-loading/model-express.md)).

### Smaller points

Logprobs give per-token confidence. With them on, the API returns the log probability of each
token the model picked, so you can see how sure it was. If an answer has many low-probability tokens, the model was
guessing, and that answer can go to a bigger model or a person instead of straight to the user.

A fixed seed gives repeatable output for tests and CI.

[FlashAttention](https://arxiv.org/abs/2205.14135) computes exact attention in tiles, so it reads and writes GPU
memory far less often. That makes attention faster and lighter on memory.

### To try

- [AIPerf](https://github.com/ai-dynamo/aiperf), NVIDIA's benchmark tool for served models. It measures time to
  first token, time between tokens, and throughput per user and in total. We used it in the labs.
- [AIConfigurator](https://docs.nvidia.com/dynamo/user-guides/disaggregated-serving) compares aggregated and
  disaggregated layouts and estimates how many prefill and decode workers you need, before you deploy anything.
- [AISimulate](https://docs.nvidia.com/dynamo/latest/knowledge-base/concepts/simulation/ai-simulate/overview.md)
  simulates the inference engine and a whole Dynamo deployment offline and searches for good settings. It is still
  experimental, for evaluation and not for capacity planning.

## Sources

- [NVIDIA: GTC Berlin 2026 training](https://www.nvidia.com/en-eu/gtc/training/)
- [NVIDIA Dynamo on GitHub](https://github.com/ai-dynamo/dynamo)
- [AIPerf on GitHub](https://github.com/ai-dynamo/aiperf)
- [NVIDIA Dynamo: architecture (Planner, KV-aware routing, prefill and decode)](https://docs.nvidia.com/dynamo/knowledge-base/concepts/architecture.md)
- [NVIDIA Dynamo: ModelExpress](https://docs.nvidia.com/dynamo/v1.3.0/kubernetes-deployment/model-loading/model-express.md)
- [NVIDIA Dynamo: Disaggregated Serving guide (AIConfigurator)](https://docs.nvidia.com/dynamo/user-guides/disaggregated-serving)
- [NVIDIA Dynamo: AISimulate overview](https://docs.nvidia.com/dynamo/latest/knowledge-base/concepts/simulation/ai-simulate/overview.md)
- [Dao et al., FlashAttention (arXiv 2205.14135)](https://arxiv.org/abs/2205.14135)
- [Kubernetes: Ingress (frozen, use Gateway)](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes Gateway API overview](https://gateway-api.sigs.k8s.io/docs/)
- [Gateway API: Migrating from Ingress](https://gateway-api.sigs.k8s.io/guides/getting-started/migrating-from-ingress/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [Envoy AI Gateway: usage-based rate limiting](https://github.com/envoyproxy/ai-gateway/blob/main/site/docs/capabilities/traffic/usage-based-ratelimiting.md)
