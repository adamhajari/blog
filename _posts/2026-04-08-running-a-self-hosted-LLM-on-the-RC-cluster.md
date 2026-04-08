---
layout: post
title: "Running a Self-Hosted LLM on the RC Cluster"
---

The Recurse Center has a small computing cluster called the Heap — four bare metal Linux machines donated by a former partner company. Three of them have GPUs. We wanted to see if we could turn one of these machines into a self-hosted LLM server that RC community members could use for privacy-sensitive tasks, like summarizing the daily check-in posts on our internal Zulip chat.

## Choosing a Model

The GPU we worked with was an Nvidia GTX 1080 Ti — a solid card when it came out in 2017, with 11GB of VRAM. That 11GB ceiling dictated everything.

Large language models need to fit in VRAM to run at reasonable speeds. At Q4_K_M quantization (a compression scheme that preserves roughly 85% of a model's original quality), the biggest models we could run were in the 12-14 billion parameter range. We used benchmarks from llmrun.dev to compare options and landed on Google's Gemma 3 12B: it uses about 7.9GB of VRAM and generates around 40 tokens per second on the 1080 Ti. Plenty fast for conversation.

## Setup: The Easy Part

Getting the model running was surprisingly straightforward. Ollama is a tool that manages and serves LLM models. On the cluster machine (mercer), I downloaded the Ollama binary, pulled the model, and started the server.

For a web interface, we ran Open WebUI — an open-source ChatGPT-style frontend — using podman (a rootless Docker alternative that was already installed on the cluster).

Within an hour we had a working chat interface talking to Gemma 3 on a GPU. The hard part was making it accessible to anyone else.

## Serving It: The Hard Part

The cluster sits behind a firewall. Port 8080, where Open WebUI was running, wasn't reachable from the outside. We needed a way to get traffic in.

**ngrok** was the obvious first attempt — it tunnels traffic through their servers and gives you a public URL. It worked immediately, but the whole point of self-hosting was privacy. Routing every prompt and response through a third party defeated the purpose.

**The solution that stuck** was an nginx reverse proxy running on a Raspberry Pi at RC's office. RC has a community deployment tool called disco that lets Recursers deploy Docker apps to this Pi. We deployed a minimal nginx config that forwards all traffic to the Heap. The result was an HTTPS endpoint that proxies through the Pi to Open WebUI on the heap. All traffic stays on RC's infrastructure.

## The Limitations

### Model quality

To test each model we tried, we gave it a real task: summarize 24 hours of check-in posts from our Zulip chat. These are short personal updates — what people are working on, what they're stuck on, what they're excited about. About 20-30 people post each day.

Anthropic's Sonnet model handles this very well. It produces a concise summary that covers every person's update. We tested four models against the same task: Qwen 2.5 14B and Gemma 3 12B on the cluster, and Gemma 4 31B and Gemma 4 e4b locally on an M3 Max MacBook Pro.

Qwen 2.5 14B was the worst — it covered only 6-8 people and produced generic "reflections and recommendations" rather than actual per-person summaries. Gemma 3 12B was slightly better. Both Gemma 4 models were a significant step up, covering ~22-23 of the ~30 people with accurate, concise summaries. But they still dropped roughly the same set of people — mostly clustered in the middle of the data, consistent with a well-known problem called "lost in the middle". These models attend well to the beginning and end of the context but lose track of information in between. Interestingly, both Gemma 4 models skipped the same people despite being very different sizes, suggesting this is an architectural limitation of the transformer attention mechanism rather than a capacity issue.

Gemma 4 e4b is a Mixture of Experts (MoE) model — it has many more total parameters than its name suggests, but only activates ~4 billion of them per token. This makes it significantly faster than the dense 31B model while producing similar quality output. For tasks like this, the speed-quality tradeoff favored e4b by a long shot.

This is fundamentally a hardware constraint on the cluster. The Gemma 4 models that performed best didn't fit on the 1080 Ti's 11GB of VRAM — we had to run them locally on a laptop with more memory.

### A laptop might be better

Here's the irony: in 2026, a base model MacBook Air has 16GB of unified memory available for GPU inference — roughly 25% more than the 1080 Ti's 11GB, with faster memory bandwidth. My 2024 MacBook Pro has 64GB of available memory opening up even more options. Running Ollama locally on a laptop means no network latency, no proxy chain, no firewall issues, and access to slightly larger models. For individual use, a modern laptop is likely the better option for running open-source models.

The cluster approach still makes sense if you want to share a single model across multiple users, or if you have access to beefier GPUs. But for the hardware we had, the complexity of serving it over the network wasn't clearly worth it compared to just running the model locally.

## Was It Worth It?

As a practical tool, the self-hosted LLM fell short. The model wasn't strong enough for the summarization task we cared about, and the infrastructure to serve it was fragile.

As a learning exercise at RC, it was exactly the right project. We got hands-on experience with GPU inference, model quantization, reverse proxies, OAuth flows, systemd services, podman containers, and the particular pain of making things work on shared infrastructure you don't have root access to. That's a lot of surface area for a day of work.
