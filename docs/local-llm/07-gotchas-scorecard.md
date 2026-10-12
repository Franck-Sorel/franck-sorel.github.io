---
layout: default
title: Gotchas, Scorecard, and Scaling
parent: Local LLM Guide
nav_order: 7
description: "Known issues, the capability scorecard to fill in, and what this does — and doesn't — tell you about scaling."
---

# Gotchas, Scorecard, and Scaling

**Goal of this page:** the checklist of things that bite, the deliverable table of the whole exercise, and honest limits of what the results mean.

## Known issues — checklist for every new node

| Issue | Symptom | Fix |
|---|---|---|
| Default context window too small | Agent "forgets" earlier instructions, no error shown | Pin `num_ctx` via Modelfile — don't rely on env vars alone |
| Vanilla `qwen2.5-coder` lacks a working tool-call template | `finish_reason: "stop"`, empty `tool_calls`, model just talks | Use `hhao/qwen2.5-coder-tools` or `qwen3:8b` for agents; vanilla coder for autocomplete only |
| 6 GB VRAM ceiling under agent load | `ollama ps` shows a CPU/GPU split; generation 5–10× slower | Drop to `qwen2.5-coder:3b`; reduce `num_ctx`; don't push past the VRAM budget |
| Aider `ollama/` vs `ollama_chat/` prefix | Garbled or template-mismatched output | Always `ollama_chat/<model>` |
| Docker GPU passthrough not configured | Container can't see GPU at all | Gate every new node on `verify-node.sh` before touching Compose |
| No published benchmarks for your laptop GPU | Tempting to trust a desktop-GPU number | Always run `benchmark-tokens.sh` on the actual node |
| CUDA architecture not pinned when *building* inference software | Slow first run / bloated build | Not applicable to Ollama (ships prebuilt kernels); applies if you ever build llama.cpp (`CMAKE_CUDA_ARCHITECTURES=89`) |

## Capability scorecard — fill this in

| Metric | qwen2.5-coder:7b (control) | qwen-coder-agent | qwen3-agent |
|---|---|---|---|
| Tokens/sec | | | |
| `ollama ps` processor split | | | |
| Valid tool-call rate, 20 runs | | | |
| Aider task: correct on first try? | n/a | | |
| Cline task: valid tool calls / attempts | n/a | | |

{: .note }
This table is the actual deliverable of the whole guide. It — not intuition about which model "seems smarter" in casual chat — is what decides the scaling question below.

## What this does — and doesn't — tell you about scaling

**Replicates directly to other nodes, no changes needed:** the completion stack (`docker-compose.ollama.yml`, `qwen2.5-coder:7b`) is per-node and stateless — deploy it identically everywhere.

**Does not get fixed by adding more nodes:** if even the tool-tuned 7–8B candidates show a weak tool-call reliability rate, reaching for multi-node pipeline parallelism doesn't solve the actual problem. Pipeline parallelism changes how much VRAM+RAM is available to hold a *bigger* model — it doesn't make a given model's tool-calling more reliable. A flaky 7B tool-caller across four nodes is still a flaky 7B tool-caller, just slower per token from the network hops.

**The one legitimate reason to go multi-node:** if the scorecard shows the 6–8B tier is unreliable for agent work *and* a larger, reliably-tool-trained model (14B+ class) would plausibly fix it — that's the actual argument for testing a pipeline-split 14B model. Benchmarked, not assumed. Don't skip the single-node measurements and jump straight there.
