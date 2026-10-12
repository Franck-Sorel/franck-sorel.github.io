---
layout: default
title: Running LLMs Locally — Ollama on a 6 GB Laptop GPU
---

This guide explains how to run large language models (LLMs) **locally** — on your own machine, no cloud, no API keys — using [Ollama](https://ollama.com), based on a real setup tested on a ThinkPad P16v Gen2 with an NVIDIA RTX 1000 Ada (6 GB VRAM).

Think of it like this: a cloud LLM is eating at a restaurant (convenient, but you pay per plate and the chef sees your order). Running locally is cooking at home — you buy the ingredients once (the model weights), your kitchen is your GPU, and nobody sees what's on your plate.

**The mental model in one diagram:**

```
 ┌─────────────────────────── your laptop ───────────────────────────┐
 │                                                                    │
 │  Agent (Aider / Cline / opencode)   ← the "driver"                 │
 │        │  OpenAI-style HTTP (tools array)                          │
 │        ▼                                                           │
 │  Ollama server (localhost:11434)    ← the "engine"                 │
 │        │  loads weights + chat template                            │
 │        ▼                                                           │
 │  Model (qwen3-agent, qwen-coder-agent) ← the "brain", in VRAM      │
 │                                                                    │
 └────────────────────────────────────────────────────────────────────┘
```

- **Two different jobs, two different models.** Autocomplete (finishing your code as you type) and agentic tool-calling (the model deciding "I need to run this command") are different skills. Conflating them is the #1 way to be disappointed by a "coding model" that chats fine but can't drive an agent.
- **VRAM is the hard wall.** A 6 GB card has roughly 4–4.5 GB usable for model + context after OS overhead and a safety margin. Everything below is sized for that.
- **Measure, don't trust spec sheets.** Laptop GPUs consistently underperform desktop parts with similar names — the included benchmark script replaces guesses with real numbers.

## Contents

1. [Prerequisites](#prerequisites)
2. [Step 1 — Serve Ollama (Docker)](#step-1--serve-ollama-docker)
3. [Step 2 — Pull the models](#step-2--pull-the-models)
4. [Step 3 — Build pinned-context agent models](#step-3--build-pinned-context-agent-models)
5. [Step 4 — Benchmark throughput](#step-4--benchmark-throughput)
6. [Step 5 — Test tool-calling reliability](#step-5--test-tool-calling-reliability)
7. [Step 6 — Hook up a coding agent](#step-6--hook-up-a-coding-agent)
8. [Known issues & gotchas](#known-issues--gotchas)
9. [Appendix A — Modelfile.qwen-coder-agent](#appendix-a--modelfileqwen-coder-agent)
10. [Appendix B — Modelfile.qwen3-agent](#appendix-b--modelfileqwen3-agent)
11. [Appendix C — benchmark-tokens.sh](#appendix-c--benchmark-tokenssh)
12. [Appendix D — test-tool-calling.py](#appendix-d--test-tool-callingpy)
13. [Appendix E — Single-node capability assessment (full doc)](#appendix-e--single-node-capability-assessment-full-doc)

## Prerequisites

- Linux host with:
  - `docker` (+ NVIDIA Container Toolkit so containers see the GPU)
  - `curl`, `jq`, `bc`, `nvidia-smi`
  - Python 3 (the tool-calling test uses only the standard library)
  - `git`
- An NVIDIA GPU. This guide is calibrated for **6 GB VRAM**; the same steps work elsewhere with the VRAM budget math adjusted.

**Gate first:** run a node verification script (`verify-node.sh` in the toolchain) before anything else. If the container can't see the GPU, nothing downstream will work — it's like checking the oven heats before mixing batter.

## Step 1 — Serve Ollama (Docker)

Docker keeps the serving stack reproducible and isolated — no CUDA toolkit soup on the host.

```bash
docker compose -f docker-compose.ollama.yml up -d
docker exec -it $(docker compose -f docker-compose.ollama.yml ps -q ollama) ollama pull qwen2.5-coder:7b
```

Ollama now listens on `http://localhost:11434` and exposes an OpenAI-compatible API.

## Step 2 — Pull the models

Two jobs, two (or three) models:

| Job | Model | Approx. size (Q4) |
|---|---|---|
| Autocomplete (Continue.dev) — no tool calling needed | `qwen2.5-coder:7b` | ~4.7 GB |
| Agent work (tool calling), patched template | `hhao/qwen2.5-coder-tools` | ~4.7 GB |
| Agent work (tool calling), natively trained | `qwen3:8b` | ~5.2 GB |
| VRAM-pressure fallback during agent runs | `qwen2.5-coder:3b` | — |

```bash
ollama pull qwen2.5-coder:7b      # autocomplete
ollama pull qwen3:8b              # agent candidate
```

**Why a patched/community build?** Vanilla `qwen2.5-coder` frequently returns `finish_reason: "stop"` with an *empty* `tool_calls` array when handed a standard OpenAI-style tools array — the model just talks instead of calling the tool, with no error. It's a known template gap, not a one-off bug. Qwen3, by contrast, is trained natively for tool calling.

## Step 3 — Build pinned-context agent models

Ollama's default context window is small (historically 2k–4k tokens) and — the dangerous part — it **silently truncates** older context instead of erroring. For an agent that looks like the model "forgot" an instruction, when really the window was too small. The fix: pin `num_ctx` in a Modelfile so the ceiling travels with the model tag, instead of depending on how the daemon was started.

```bash
ollama create qwen-coder-agent -f Modelfile.qwen-coder-agent   # 8192 ctx
ollama create qwen3-agent       -f Modelfile.qwen3-agent       # 6144 ctx
```

The Modelfiles themselves are in [Appendix A](#appendix-a--modelfileqwen-coder-agent) and [Appendix B](#appendix-b--modelfileqwen3-agent).

Verify what actually loaded, and how:

```bash
ollama list
ollama ps    # PROCESSOR column must read "100% GPU"
```

If `ollama ps` shows a CPU/GPU **split**, the model didn't fully fit and speed drops sharply — drop to the 3B fallback or lower `num_ctx` rather than troubleshooting the split.

## Step 4 — Benchmark throughput

Before layering an agent on top, measure raw tokens/sec on **your** node — laptop parts have no trustworthy published numbers, and any slowness observed later can then be attributed correctly (model vs. harness overhead vs. hardware).

```bash
chmod +x benchmark-tokens.sh
./benchmark-tokens.sh qwen2.5-coder:7b
./benchmark-tokens.sh qwen-coder-agent
./benchmark-tokens.sh qwen3-agent
```

Record tokens/sec, load time, and `nvidia-smi` readings for each — that's row 1 of your capability scorecard. Full script in [Appendix C](#appendix-c--benchmark-tokenssh).

## Step 5 — Test tool-calling reliability

This is the number that actually predicts agent behavior. A model that "answers well in chat" is not the same claim as one that returns a correctly-formed function call *consistently*. If it returns a valid call N% of the time on a simple single-tool task, expect roughly that ceiling (or worse, compounded) on real multi-step runs.

```bash
# control — expect a low score, run it deliberately:
python3 test-tool-calling.py --model qwen2.5-coder:7b --runs 20
python3 test-tool-calling.py --model qwen-coder-agent --runs 20
python3 test-tool-calling.py --model qwen3-agent --runs 20
```

Rule of thumb: **below ~70% valid on this control task, an agent harness will feel unreliable in practice.** The control run should score visibly lower than the tool-tuned candidates — if it doesn't, something is misconfigured (check `ollama show <model>` lists `tools` under Capabilities). Full script in [Appendix D](#appendix-d--test-tool-callingpy).

## Step 6 — Hook up a coding agent

### Aider (soft test — tolerant of imperfect tool calling)

Aider can work from structured text diffs rather than hard-requiring native function calling for every step, making it the forgiving first integration:

```bash
pip install --break-system-packages aider-chat

export OLLAMA_API_BASE=http://127.0.0.1:11434
export OLLAMA_CONTEXT_LENGTH=8192

cd /path/to/a/test/repo
aider --model ollama_chat/qwen-coder-agent
```

Use the `ollama_chat/` prefix, not `ollama/` — the wrong one historically causes chat-template mismatches.

**Test task:** *"Add a `add_numbers(a, b)` function to `utils.py` with a docstring and a unit test in `test_utils.py`. Run the tests after."*

### Cline (hard test — native tool-calling via Ollama's tools API)

Cline does real, permissioned tool calls through the model's native function-calling interface, with a Plan/Act approval step — so you watch the raw tool-call format itself:

1. Install the Cline VS Code extension.
2. Point its provider at Ollama: `http://localhost:11434`, model `qwen-coder-agent` (then repeat with `qwen3-agent`).
3. Keep Cline's compact-prompt / reduced-context setting **on** — VRAM-constrained runs are exactly what it exists for.
4. Run the same test task in Plan/Act mode.

Record: valid-on-first-try tool calls, any retry loops on malformed calls, and whether Plan/Act caught anything Aider's softer format silently botched.

## Known issues & gotchas

| Issue | Symptom | Fix |
|---|---|---|
| Default context window too small | Agent "forgets" earlier instructions, no error | Pin `num_ctx` via Modelfile — don't rely on env vars alone |
| Vanilla `qwen2.5-coder` lacks a working tool-call template | `finish_reason: "stop"`, empty `tool_calls`, model just talks | Use `hhao/qwen2.5-coder-tools` or `qwen3:8b` for agents; vanilla coder for autocomplete only |
| 6 GB VRAM ceiling under agent load | `ollama ps` shows CPU/GPU split; generation 5–10× slower | Drop to `qwen2.5-coder:3b`; reduce `num_ctx` |
| Aider `ollama/` vs `ollama_chat/` prefix | Garbled / template-mismatched output | Always `ollama_chat/<model>` |
| Docker GPU passthrough not configured | Container can't see GPU | Gate every node on `verify-node.sh` before Compose |
| No published benchmarks for your laptop GPU | Tempting to trust desktop-GPU numbers | Always run `benchmark-tokens.sh` on the actual node |

## Appendix A — Modelfile.qwen-coder-agent

````markdown
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
````

## Appendix B — Modelfile.qwen3-agent

````markdown
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
````

## Appendix C — benchmark-tokens.sh

````markdown
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
````

## Appendix D — test-tool-calling.py

````markdown
```python
#!/usr/bin/env python3
"""
test-tool-calling.py — measure how reliably a local Ollama model returns
a valid, well-formed tool call instead of prose.

This is the check that a model "answers well in chat" does not substitute
for. An agent harness (Aider, Cline, etc.) is only as reliable as this
number: if the model returns a valid tool_calls block N% of the time on a
simple, unambiguous single-tool task, expect roughly that ceiling (or
worse, compounded across steps) on real multi-step agent runs.

Usage:
    python3 test-tool-calling.py --model qwen-coder-agent --runs 20
    python3 test-tool-calling.py --model qwen3-agent --runs 20
    python3 test-tool-calling.py --model qwen2.5-coder:7b --runs 20   # expect a low score — control case

No third-party dependencies: uses only the Python standard library.
"""

import argparse
import json
import time
import urllib.request

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read the contents of a file at the given path.",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Relative or absolute file path"}
                },
                "required": ["path"],
            },
        },
    }
]

PROMPT = "Read the contents of config.yaml so you can check its settings."


def call_ollama(host, model, tools):
    payload = {
        "model": model,
        "messages": [{"role": "user", "content": PROMPT}],
        "tools": tools,
        "stream": False,
    }
    req = urllib.request.Request(
        f"{host}/api/chat",
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    start = time.monotonic()
    with urllib.request.urlopen(req, timeout=120) as resp:
        body = json.loads(resp.read().decode("utf-8"))
    elapsed = time.monotonic() - start
    return body, elapsed


def is_valid_call(body):
    msg = body.get("message", {})
    calls = msg.get("tool_calls") or []
    if not calls:
        return False, "no tool_calls returned"
    call = calls[0]
    fn = call.get("function", {})
    if fn.get("name") != "read_file":
        return False, f"wrong/missing function name: {fn.get('name')!r}"
    args = fn.get("arguments")
    if isinstance(args, str):
        try:
            args = json.loads(args)
        except json.JSONDecodeError:
            return False, "arguments not valid JSON"
    if not isinstance(args, dict) or "path" not in args:
        return False, "missing required 'path' argument"
    return True, "ok"


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--model", required=True, help="Ollama model tag to test")
    ap.add_argument("--runs", type=int, default=20)
    ap.add_argument("--host", default="http://localhost:11434")
    args = ap.parse_args()

    valid = 0
    latencies = []
    failures = []

    print(f"Testing {args.model!r} — {args.runs} runs, single-tool task\n")

    for i in range(1, args.runs + 1):
        try:
            body, elapsed = call_ollama(args.host, args.model, TOOLS)
        except Exception as exc:  # network / timeout / server error
            failures.append((i, f"request failed: {exc}"))
            print(f"  run {i:2d}: FAIL  request error ({exc})")
            continue

        latencies.append(elapsed)
        ok, reason = is_valid_call(body)
        status = "OK  " if ok else "FAIL"
        print(f"  run {i:2d}: {status}  ({elapsed:5.1f}s)  {reason}")
        if ok:
            valid += 1
        else:
            failures.append((i, reason))

    print("\n== Summary ==")
    print(f"Model:              {args.model}")
    print(f"Valid tool calls:   {valid}/{args.runs}  ({100 * valid / args.runs:.0f}%)")
    if latencies:
        print(f"Avg latency:        {sum(latencies) / len(latencies):.1f}s")
    if failures:
        print("\nFailure reasons:")
        for i, reason in failures:
            print(f"  run {i}: {reason}")

    print(
        "\nRule of thumb for this hardware tier: below ~70% valid on this "
        "single-tool control task, expect an agent harness (multi-step, "
        "multi-tool) to feel unreliable in practice. Compare against the "
        "control run of plain qwen2.5-coder:7b — it should score visibly "
        "lower than the tool-tuned candidates."
    )


if __name__ == "__main__":
    main()
```
````

## Appendix E — Single-node capability assessment (full doc)

The complete assessment document the steps above are derived from:

````markdown
```markdown
# Single-Node Capability Assessment — ThinkPad P16v Gen2 (Node 1)

## Where this fits

Three docs precede this one: the architecture design review (production =
local Ollama, lab = llama.cpp pipeline-parallel, EXO experimental-only),
the corrected implementation toolchain (Docker-first, no host CUDA, modern
NVIDIA Container Toolkit), and the serving-engine correction (Ollama stays
the only production path — vLLM/SGLang rejected for this VRAM tier).

This doc is the missing step between "the toolchain is installed" and
"scale to four nodes": prove out **one** node end-to-end — model selection,
clean serving, and integration with a real open-source coding agent that
does tool calling — before assuming the other three will behave the same
way. Everything here runs on Laptop 1 only. Nothing here touches the
llama.cpp RPC lab stack.

Companion files (same directory as this doc):
- `ollama/Modelfile.qwen-coder-agent`
- `ollama/Modelfile.qwen3-agent`
- `scripts/benchmark-tokens.sh`
- `scripts/test-tool-calling.py`

---

## 1. Capability Assessment — What This Card Can Actually Hold

No independently published Ollama benchmark exists yet for the RTX 1000 Ada
Laptop specifically, so treat every number below as a starting estimate to
be replaced by your own measurement in §4, not a promise.

What we do know, from adjacent hardware:

- Token generation speed scales more closely with memory bandwidth than raw
  core count once a model fits in VRAM. The RTX 1000 Ada's ~192 GB/s puts it
  well below desktop cards in this comparison class.
- Laptop-class GPUs consistently underperform desktop parts with similar
  nominal specs — one benchmark found a laptop RTX 4070 (8 GB) ran
  roughly 70% slower than a desktop RTX 4090 on the same workload, driven
  by fewer CUDA/tensor cores and tighter thermal limits, not just the VRAM
  difference.
- As a rough anchor: a desktop RTX 3070 (8 GB, ~448 GB/s) reaches roughly
  40–80 tokens/sec on a 7B Q4 model. The RTX 1000 Ada has under half that
  bandwidth and a laptop power/thermal envelope, so expect a generation
  speed well below that range — plausibly 10–25 tokens/sec on a 7B model
  that fully fits in VRAM. This is a working estimate, not a spec — §4
  replaces it with a real number for this exact node.
- General guidance is to leave roughly 1–2 GB of VRAM margin for stable
  inference under load rather than filling the card to the last megabyte.

### VRAM budget math for this card (6 GB)

| Component | Budget |
|---|---|
| OS/driver/compositor overhead | ~0.3–0.5 GB |
| Safety margin (per general Ollama sizing guidance) | ~1–2 GB |
| **Usable for model + KV cache** | **~4–4.5 GB** |

A 7B model at Q4_K_M quantization is already roughly 4–5 GB in weights
alone before any context is loaded — this is the same constraint flagged
in the toolchain doc's §1.2, now made concrete for the sizing decisions
below.

---

## 2. Model Selection — Two Different Jobs, Two Different Models

Mode 1 (autocomplete, per the design review) and agentic tool-calling are
different capabilities, and conflating them is the single biggest way
teams end up disappointed by a "coding model" that tests fine in chat.

### 2.1 Completion model — unchanged from the toolchain doc

```bash
ollama pull qwen2.5-coder:7b
```

Still correct for Continue.dev tab-autocomplete: that workflow completes
text, it doesn't call tools, so this limitation doesn't apply to it.

### 2.2 The tool-calling gap — verified, not assumed

Qwen2.5's tool-calling support in Ollama depends on the model shipping a
working tool-call chat template — Ollama implements its own template
layer on top of the model, separate from how frameworks like Hugging Face
Transformers handle it, and if the template doesn't produce the right
format, tool calls silently fail. This isn't theoretical: a documented
case shows plain `qwen2.5-coder:7b-instruct` returning `finish_reason:
"stop"` with an empty tool-calls array when handed a standard OpenAI-style
tools array — the model answers as if no tool existed, rather than erroring
or refusing. **Don't discover this mid-agent-run; verify it in §6 before
trusting the model with an agent.**

### 2.3 Agent-capable candidates to test

Three candidates, each fitting the 6 GB budget from §1:

| Model | Ollama pull | Approx. size (Q4) | Why it's here |
|---|---|---|---|
| `hhao/qwen2.5-coder-tools` | community build | ~4.7 GB | Same Qwen2.5-Coder weights, patched with a working Ollama tool-call template; ships pre-wired for several agent harnesses including Claude Code-style and OpenCode launchers. |
| `qwen3:8b` | official library | ~5.2 GB | Qwen3 is trained natively for tool calling rather than patched after the fact, and independent local-model testing consistently ranks it with the lowest dropped/invalid tool-call rate among small local models. |
| `llama3-groq-tool-use:8b` (if available in your Ollama version's library, else import via GGUF) | — | ~4.9 GB | A fine-tune built specifically for function calling; the 8B variant scored competitively on the Berkeley Function Calling Leaderboard among open models at this size. Treat as a third comparison point, not the default pick. |

Fallback tier for VRAM pressure during agent runs (multi-turn context grows
fast):

```bash
ollama pull qwen2.5-coder:3b
```

### 2.4 Build the pinned-context agent models

Ollama's own default context window is small unless configured — the
daemon has shipped with defaults as low as 2k–4k tokens historically —
and it silently discards older context past the limit rather than erroring,
which is a dangerous failure mode for an agent: it looks like the model
"forgot" an instruction rather than reporting an error. Ollama's own
current guidance for coding-tool and agent workloads recommends at least
64,000 tokens of context — a number sized for cards with far more VRAM
than this one has. Pin an explicit, hardware-appropriate ceiling instead:

```bash
cd ollama
ollama create qwen-coder-agent -f Modelfile.qwen-coder-agent   # 8192 ctx
ollama create qwen3-agent       -f Modelfile.qwen3-agent        # 6144 ctx
```

Using a Modelfile instead of an environment variable means the context
ceiling travels with the model tag — every agent config that points at
`qwen-coder-agent` gets the same guaranteed window, instead of depending on
however the daemon happened to be started.

---

## 3. Clean Serving — Reusing the Established Stack

No new serving layer. This reuses `docker/docker-compose.ollama.yml` and
`scripts/verify-node.sh` from the toolchain doc, unchanged:

```bash
./scripts/verify-node.sh                 # gate: don't proceed if this fails
cd docker
docker compose -f docker-compose.ollama.yml up -d
docker exec -it $(docker compose -f docker-compose.ollama.yml ps -q ollama) \
  ollama pull qwen2.5-coder:7b
```

Build the two agent Modelfiles inside the same container so they end up in
the same Ollama data volume:

```bash
docker cp ollama/Modelfile.qwen-coder-agent <container>:/tmp/
docker cp ollama/Modelfile.qwen3-agent      <container>:/tmp/
docker exec -it <container> ollama create qwen-coder-agent -f /tmp/Modelfile.qwen-coder-agent
docker exec -it <container> ollama create qwen3-agent       -f /tmp/Modelfile.qwen3-agent
```

Confirm what's actually loaded and how:

```bash
ollama list
ollama ps          # PROCESSOR column must read "100% GPU" — if it shows a
                    # CPU/GPU split, the model didn't fully fit and speed
                    # will drop sharply; drop to the 3B fallback instead
                    # of troubleshooting the split.
```

---

## 4. Baseline Capability Test — Before Any Agent Touches It

Measure raw throughput on this exact node before layering an agent on top,
so any slowness observed later can be attributed correctly (model vs.
agent-harness overhead vs. hardware):

```bash
chmod +x scripts/benchmark-tokens.sh
./scripts/benchmark-tokens.sh qwen2.5-coder:7b
./scripts/benchmark-tokens.sh qwen-coder-agent
./scripts/benchmark-tokens.sh qwen3-agent
```

Record the tokens/sec, load time, and `nvidia-smi` memory readings for each
— this becomes row 1 of the capability scorecard in §8. If any run shows
`ollama ps` reporting a CPU/GPU split, stop and fix the model choice or
context size before moving to §5; agent testing on a partially-offloaded
model produces meaningless reliability numbers, since slow generation
compounds with — and can be mistaken for — tool-calling failures.

---

## 5. Tool-Calling Reliability — The Number That Actually Predicts Agent Behavior

This is the quantitative gate. A model that chats well is not the same
claim as a model that reliably drives an agent — the difference is whether
it returns a correctly-formed function call instead of prose, consistently,
under a fixed test.

```bash
python3 scripts/test-tool-calling.py --model qwen2.5-coder:7b --runs 20   # control — expect a low score
python3 scripts/test-tool-calling.py --model qwen-coder-agent --runs 20
python3 scripts/test-tool-calling.py --model qwen3-agent --runs 20
```

Run the control case (plain `qwen2.5-coder:7b`) deliberately — it should
score visibly worse than the two tool-tuned candidates, and if it doesn't,
that's a signal something else is misconfigured (check the Modelfile
actually built, check `ollama show <model>` lists `tools` under
Capabilities) rather than a reason to trust plain qwen2.5-coder with an
agent.

---

## 6. Agent Integration — Two-Tier Test

Two different open-source agents, deliberately chosen to stress different
things.

### 6.1 Tier A — Aider (soft test: tolerant of imperfect tool calling)

Aider is a terminal-first, Git-centric coding agent, and it's the easier
first integration test because much of its edit workflow doesn't hinge
entirely on the model returning perfectly-formed JSON tool calls — it can
work from structured text diffs in the prompt/response itself, which is
more forgiving of a shakier local model than a framework that hard-requires
native function calling for every step.

```bash
pip install --break-system-packages aider-chat

export OLLAMA_API_BASE=http://127.0.0.1:11434
export OLLAMA_CONTEXT_LENGTH=8192

cd /path/to/a/test/repo
aider --model ollama_chat/qwen-coder-agent
```

Use the `ollama_chat/` prefix, not `ollama/` — Aider's own documentation
recommends it, and the wrong prefix has historically caused chat-template
mismatches.

**Test task (run this exact prompt inside Aider):**
> "Add a `add_numbers(a, b)` function to `utils.py` with a docstring and a
> unit test in `test_utils.py`. Run the tests after."

Record: did it produce a correct diff, did it apply cleanly, did it
actually invoke the test runner (a tool-use step), and how many turns it
took versus what a correct single-shot response would need.

### 6.2 Tier B — Cline (hard test: native tool-calling via Ollama's tools API)

Cline is a VS Code extension (with a standalone CLI) that does real,
permissioned tool calls — file edits, terminal commands, and MCP tools —
through the model's native function-calling interface, with a Plan/Act
approval step before actions run. This is a stricter test: unlike Aider, it
depends directly on the JSON tool-call reliability measured in §5.

1. Install the Cline extension in VS Code.
2. Point its provider at Ollama: `http://localhost:11434`, model
   `qwen-coder-agent` (then repeat with `qwen3-agent`).
3. Keep Cline's compact-prompt / reduced-context setting on — local, VRAM-
   constrained runs are exactly the case that setting exists for.
4. Run the same test task as §6.1, but this time in Plan/Act mode, so every
   proposed tool call is visible before it executes — you're watching the
   raw tool-call format the model produces, not just the end result.

Record: number of proposed tool calls that were valid on first try, any
loops (the model retrying a malformed call), and whether Plan/Act caught
anything Aider's softer format would have silently gotten away with (or
silently botched).

---

## 7. Known Issues — Checklist for Every New Node

Carried forward from the design review and toolchain docs, plus what
surfaced in this phase:

| Issue | Symptom | Fix |
|---|---|---|
| Ollama default context window too small | Agent "forgets" earlier instructions mid-task, no error shown | Pin `num_ctx` via Modelfile (§2.4), don't rely on env vars alone |
| Vanilla `qwen2.5-coder` lacks a working tool-call template | `finish_reason: "stop"`, empty `tool_calls`, model just talks instead of calling | Use `hhao/qwen2.5-coder-tools` or `qwen3:8b` for agent work; keep vanilla coder for autocomplete only |
| 6 GB VRAM ceiling under agent load | `ollama ps` shows a CPU/GPU split; generation slows 5–10× | Drop to `qwen2.5-coder:3b` fallback; reduce `num_ctx`; don't try to push past the budget in §1 |
| Aider `ollama/` vs `ollama_chat/` prefix | Garbled or template-mismatched output | Always use `ollama_chat/<model>` (§6.1) |
| Docker GPU passthrough not configured | Container can't see GPU at all | Gate every new node on `verify-node.sh` before touching Compose (per toolchain doc §0.4) |
| RTX 1000 Ada has no published Ollama benchmarks | Tempting to trust a desktop-GPU number | Always run `benchmark-tokens.sh` on the actual node (§4) — laptop parts underperform desktop parts with similar specs |
| CUDA architecture not pinned when *building* inference software | Slow first run / bloated build | Not applicable to Ollama (ships prebuilt CUDA kernels); still applies if this node ever builds llama.cpp for Mode 2 (toolchain doc §2.1, `CMAKE_CUDA_ARCHITECTURES=89`) |
| llama.cpp RPC backend | N/A at this stage | Out of scope for this doc — Mode 1 only. Still proof-of-concept and network-isolation-only per the toolchain doc §2.2 if you get to Mode 2 later |

---

## 8. Capability Scorecard — Fill This In From §4–§6

| Metric | qwen2.5-coder:7b (control) | qwen-coder-agent | qwen3-agent |
|---|---|---|---|
| Tokens/sec (§4) | | | |
| `ollama ps` processor split | | | |
| Valid tool-call rate, 20 runs (§5) | | | |
| Aider task: correct on first try? (§6.1) | n/a | | |
| Cline task: valid tool calls / attempts (§6.2) | n/a | | |

This table is the actual deliverable of this phase — it's what decides §9,
not intuition about which model "seems smarter" in casual chat.

---

## 9. What This Does — and Doesn't — Tell You About Scaling

**Replicates directly to the other three nodes, no changes needed:**
Mode 1's completion stack (`docker-compose.ollama.yml`, `qwen2.5-coder:7b`)
is per-node and stateless — deploy it identically on Laptop 2–4 exactly as
specified in the toolchain doc. Nothing in this phase changes that.

**Does not get fixed by adding more nodes:**
If §5–§6 show that even the tool-tuned 7–8B candidates have a weak
tool-call reliability rate, the instinct to reach for the four-node
llama.cpp pipeline-parallel lab (Mode 2) doesn't solve the actual problem.
Pipeline parallelism changes how much VRAM+RAM is available to hold a
*bigger* model — it doesn't make a given model's tool-calling behavior more
reliable. A flaky 7B tool-caller run across four nodes is still a flaky
7B tool-caller, just slower per token because of the added network hops.

**The one legitimate reason this phase would justify Mode 2 involvement:**
if the scorecard shows the 6–8B tier is unreliable for agent work *and* a
larger, more reliably-tool-trained model (14B+ class) would plausibly fix
it, that's the actual argument for testing a pipeline-split 14B tool-tuned
model across Laptop 1–4 under the design review's already-approved
Mode 2 rules — benchmarked, not assumed, per the design review's own
"production only after benchmark validation" decision on pipeline
parallelism. Don't skip §4–§6 on a single node and jump straight there.
```
````
