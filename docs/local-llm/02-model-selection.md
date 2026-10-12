---
layout: default
title: Choose the Right Models
parent: Local LLM Guide
nav_order: 2
description: "Model selection for a 6 GB card — autocomplete vs tool-calling, the VRAM budget, and the vanilla qwen2.5-coder tool-call gap."
---

# Choose the Right Models

**Goal of this page:** pick the right model for each job within the 6 GB VRAM budget — and know which models *cannot* drive an agent, no matter how well they chat.

## The VRAM budget for a 6 GB card

| Component | Budget |
|---|---|
| OS / driver / compositor overhead | ~0.3–0.5 GB |
| Safety margin (stable inference under load) | ~1–2 GB |
| **Usable for model + KV cache** | **~4–4.5 GB** |

A 7B model at Q4_K_M quantization is already roughly 4–5 GB in weights *before any context is loaded*. That's the constraint every decision on this page respects.

## Two jobs, two (or three) models

| Job | Model | Approx. size (Q4) | Why |
|---|---|---|---|
| Autocomplete (Continue.dev) | `qwen2.5-coder:7b` | ~4.7 GB | Completes text, doesn't call tools — reliability not required here |
| Agent work, patched template | `hhao/qwen2.5-coder-tools` | ~4.7 GB | Community build of Qwen2.5-Coder with a working Ollama tool-call template |
| Agent work, natively trained | `qwen3:8b` | ~5.2 GB | Trained natively for tool calling; lowest dropped/invalid tool-call rate among small local models |
| Third comparison point | `llama3-groq-tool-use:8b` | ~4.9 GB | Fine-tune built for function calling; scored competitively on the Berkeley Function Calling Leaderboard. Comparison point, not the default pick |
| VRAM-pressure fallback | `qwen2.5-coder:3b` | — | Drop here when multi-turn agent context overflows the card |

```bash
ollama pull qwen2.5-coder:7b      # autocomplete
ollama pull qwen3:8b              # agent candidate
```

## The tool-calling gap — verified, not assumed

{: .warning }
Vanilla `qwen2.5-coder:7b` frequently returns `finish_reason: "stop"` with an **empty `tool_calls` array** when handed a standard OpenAI-style tools array. The model answers as if no tool existed — no error, no refusal. This is a known, documented chat-template gap in Ollama, not a one-off bug.

Why: Ollama implements its own template layer on top of the model, separate from how frameworks like Hugging Face Transformers handle it. If the template doesn't produce the right format, tool calls silently fail.

**Don't discover this mid-agent-run.** The [tool-calling test](/docs/local-llm/05-tool-calling/) verifies it in 20 runs before you trust any model with an agent.
