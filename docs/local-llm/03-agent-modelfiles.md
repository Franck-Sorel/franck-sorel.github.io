---
layout: default
title: Pin Context with Modelfiles
parent: Local LLM Guide
nav_order: 3
description: "Build pinned-context agent models with Ollama Modelfiles — why num_ctx must be explicit, with both Modelfiles inline."
---

# Pin Context with Modelfiles

**Goal of this page:** agent models with an explicit, hardware-appropriate context window — so the agent never silently "forgets".

## Why pin `num_ctx` at all

Ollama's default context window is small (historically 2k–4k tokens) and — the dangerous part — it **silently truncates** older context instead of erroring. For an agent, that looks like the model "forgot" an instruction, when really the window was too small.

{: .note }
Ollama's own guidance for coding-tool and agent workloads recommends at least 64,000 tokens of context — a number sized for cards with far more VRAM than 6 GB. Pin a hardware-appropriate ceiling instead of the recommendation.

Using a Modelfile (rather than an environment variable) means the ceiling **travels with the model tag** — every agent config pointing at `qwen-coder-agent` gets the same guaranteed window, regardless of how the daemon was started.

## Build the two agent models

```bash
ollama create qwen-coder-agent -f Modelfile.qwen-coder-agent   # 8192 ctx
ollama create qwen3-agent       -f Modelfile.qwen3-agent       # 6144 ctx
```

## Modelfile.qwen-coder-agent

```dockerfile
# Build with: ollama create qwen-coder-agent -f Modelfile.qwen-coder-agent
#
# hhao/qwen2.5-coder-tools is a community build of Qwen2.5-Coder with a
# working Ollama tool-call TEMPLATE — vanilla qwen2.5-coder frequently
# returns finish_reason "stop" with an empty tool_calls array when handed
# an OpenAI-style tools array, even though the underlying Qwen2.5 family
# supports tool calling in principle. This is a known, documented gap, not
# a one-off bug — do not assume plain qwen2.5-coder will drive an agent.
#
# num_ctx is set explicitly rather than trusted to the daemon default.
# Ollama defaults to a small window (historically 2k, now commonly 4k)
# unless told otherwise, and silently truncates older context instead of
# erroring — the failure mode is a confused agent, not a visible crash.
# 8192 is chosen as a 6 GB VRAM-safe ceiling; see the guide's VRAM budget
# math before raising it.

FROM hhao/qwen2.5-coder-tools

PARAMETER num_ctx 8192
PARAMETER temperature 0.2
```

## Modelfile.qwen3-agent

```dockerfile
# Build with: ollama create qwen3-agent -f Modelfile.qwen3-agent
#
# Qwen3 is trained natively for tool calling (unlike vanilla Qwen2.5-Coder),
# and community testing consistently ranks it with the lowest dropped/invalid
# tool-call rate among small local models. It runs closer to the 6 GB ceiling
# than the 7B coder model (~5.2 GB at Q4), so num_ctx is kept tighter here —
# raise only after confirming headroom with nvidia-smi during a live run.

FROM qwen3:8b

PARAMETER num_ctx 6144
PARAMETER temperature 0.2
```

{: .tip }
Qwen3 runs closer to the 6 GB ceiling than the 7B coder model (~5.2 GB at Q4). Raise its `num_ctx` only after confirming headroom with `nvidia-smi` during a live run — not before.
