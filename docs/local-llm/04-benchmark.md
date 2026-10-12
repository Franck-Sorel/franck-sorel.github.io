---
layout: default
title: Benchmark Throughput
parent: Local LLM Guide
nav_order: 4
description: "Measure real tokens/sec on your node with benchmark-tokens.sh — never trust published GPU numbers."
---

# Benchmark Throughput

**Goal of this page:** measure raw tokens/sec on **your** node *before* layering an agent on top, so any slowness observed later can be attributed correctly — model vs. harness overhead vs. hardware.

## Why your own numbers

The RTX 1000 Ada Laptop has no independently published Ollama benchmarks. Adjacent-hardware reasoning says to expect **well below** the 40–80 tokens/sec a desktop RTX 3070 reaches on a 7B Q4 model — plausibly 10–25 tokens/sec — because the laptop part has under half the memory bandwidth (~192 GB/s) and a tighter thermal envelope. Working estimate, not a spec. This script replaces it with a real number.

## Run it

```bash
chmod +x scripts/benchmark-tokens.sh
./scripts/benchmark-tokens.sh qwen2.5-coder:7b
./scripts/benchmark-tokens.sh qwen-coder-agent
./scripts/benchmark-tokens.sh qwen3-agent
```

Record tokens/sec, load time, and `nvidia-smi` memory readings for each — this becomes row 1 of the capability scorecard.

{: .danger}
If any run shows `ollama ps` reporting a CPU/GPU split, stop and fix the model choice or context size **before** moving on. Agent testing on a partially-offloaded model produces meaningless reliability numbers — slow generation compounds with, and can be mistaken for, tool-calling failures.

## The script: `benchmark-tokens.sh`

```bash
#!/usr/bin/env bash
# benchmark-tokens.sh — measure real tokens/sec on THIS node for a given
# Ollama model. Run this before trusting any published GPU benchmark:
# the RTX 1000 Ada Laptop has no widely-published Ollama numbers, and
# laptop-class GPUs consistently underperform desktop parts with the same
# nominal spec due to lower CUDA/tensor core counts and thermal limits.
#
# Usage: ./benchmark-tokens.sh <model-name> ["prompt text"]

set -euo pipefail

MODEL="${1:?Usage: $0 <model-name> [prompt]}"
PROMPT="${2:-Write a Python function that returns the nth Fibonacci number, with a short docstring.}"
HOST="${OLLAMA_HOST:-http://localhost:11434}"

command -v jq >/dev/null 2>&1 || { echo "jq is required: sudo apt install -y jq"; exit 1; }

echo "== Node GPU state before load =="
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv,noheader

echo
echo "== Running $MODEL =="
RESPONSE=$(curl -s "$HOST/api/generate" -d "$(jq -n \
  --arg model "$MODEL" --arg prompt "$PROMPT" \
  '{model: $model, prompt: $prompt, stream: false}')")

echo "$RESPONSE" | jq -r '.response' | head -c 400
echo
echo "..."
echo

EVAL_COUNT=$(echo "$RESPONSE" | jq -r '.eval_count // 0')
EVAL_NS=$(echo "$RESPONSE" | jq -r '.eval_duration // 0')
PROMPT_EVAL_COUNT=$(echo "$RESPONSE" | jq -r '.prompt_eval_count // 0')
PROMPT_EVAL_NS=$(echo "$RESPONSE" | jq -r '.prompt_eval_duration // 0')
LOAD_NS=$(echo "$RESPONSE" | jq -r '.load_duration // 0')

if [ "$EVAL_NS" -gt 0 ]; then
  TOK_S=$(echo "scale=2; $EVAL_COUNT / ($EVAL_NS / 1000000000)" | bc)
else
  TOK_S="n/a"
fi

echo "== Results =="
echo "Model load time:      $(echo "scale=2; $LOAD_NS/1000000000" | bc) s"
echo "Prompt tokens:        $PROMPT_EVAL_COUNT"
echo "Generated tokens:     $EVAL_COUNT"
echo "Generation speed:     $TOK_S tokens/sec"

echo
echo "== Node GPU state after load =="
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv,noheader

echo
echo "Also check: ollama ps   (confirm PROCESSOR column says 100% GPU, not CPU/GPU split)"
```
