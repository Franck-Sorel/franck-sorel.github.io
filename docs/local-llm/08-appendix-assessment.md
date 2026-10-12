---
layout: default
title: Full Capability Assessment
parent: Local LLM Guide
nav_order: 8
description: "The complete single-node capability assessment document this guide is derived from."
---

# Full Capability Assessment

**This is the complete reference document** the rest of the guide is derived from — kept verbatim for those who want the full reasoning chain. It is the missing step between "the toolchain is installed" and "scale to four nodes": prove out **one** node end-to-end before assuming the other three will behave the same way.

*Where this fits:* it sits downstream of an architecture design review (production = local Ollama, lab = llama.cpp pipeline-parallel, EXO experimental-only), a Docker-first toolchain doc (no host CUDA, modern NVIDIA Container Toolkit), and a serving-engine correction (Ollama stays the only production path — vLLM/SGLang rejected for this VRAM tier).

Companion files: `Modelfile.qwen-coder-agent`, `Modelfile.qwen3-agent`, `benchmark-tokens.sh`, `test-tool-calling.py` — all embedded earlier in this guide.

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
alone before any context is loaded — this is the constraint now made
concrete for the sizing decisions below.

## 2. Model Selection — Two Different Jobs, Two Different Models

Mode 1 (autocomplete) and agentic tool-calling are different capabilities,
and conflating them is the single biggest way teams end up disappointed
by a "coding model" that tests fine in chat.

### 2.1 Completion model

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
or refusing. **Don't discover this mid-agent-run; verify it before
trusting the model with an agent.**

### 2.3 Agent-capable candidates to test

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

Ollama's own default context window is small unless configured, and it
silently discards older context past the limit rather than erroring —
a dangerous failure mode for an agent: it looks like the model "forgot"
an instruction rather than reporting an error. Ollama's current guidance
for coding-tool and agent workloads recommends at least 64,000 tokens of
context — a number sized for cards with far more VRAM than this one has.
Pin an explicit, hardware-appropriate ceiling instead:

```bash
cd ollama
ollama create qwen-coder-agent -f Modelfile.qwen-coder-agent   # 8192 ctx
ollama create qwen3-agent       -f Modelfile.qwen3-agent        # 6144 ctx
```

Using a Modelfile instead of an environment variable means the context
ceiling travels with the model tag — every agent config that points at
`qwen-coder-agent` gets the same guaranteed window, instead of depending
on however the daemon happened to be started.

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

## 7. Known Issues — Checklist for Every New Node

Carried forward from the design review and toolchain docs, plus what
surfaced in this phase:

| Issue | Symptom | Fix |
|---|---|---|
| Ollama default context window too small | Agent "forgets" earlier instructions mid-task, no error shown | Pin `num_ctx` via Modelfile (§2.4), don't rely on env vars alone |
| Vanilla `qwen2.5-coder` lacks a working tool-call template | `finish_reason: "stop"`, empty `tool_calls`, model just talks instead of calling | Use `hhao/qwen2.5-coder-tools` or `qwen3:8b` for agent work; keep vanilla coder for autocomplete only |
| 6 GB VRAM ceiling under agent load | `ollama ps` shows a CPU/GPU split; generation slows 5–10× | Drop to `qwen2.5-coder:3b` fallback; reduce `num_ctx`; don't try to push past the budget in §1 |
| Aider `ollama/` vs `ollama_chat/` prefix | Garbled or template-mismatched output | Always use `ollama_chat/<model>` (§6.1) |
| Docker GPU passthrough not configured | Container can't see GPU at all | Gate every new node on `verify-node.sh` before touching Compose |
| RTX 1000 Ada has no published Ollama benchmarks | Tempting to trust a desktop-GPU number | Always run `benchmark-tokens.sh` on the actual node (§4) — laptop parts underperform desktop parts with similar specs |
| CUDA architecture not pinned when *building* inference software | Slow first run / bloated build | Not applicable to Ollama (ships prebuilt CUDA kernels); still applies if this node ever builds llama.cpp (toolchain doc §2.1, `CMAKE_CUDA_ARCHITECTURES=89`) |
| llama.cpp RPC backend | N/A at this stage | Out of scope for this doc — Mode 1 only. Still proof-of-concept and network-isolation-only per the toolchain doc §2.2 if you get to Mode 2 later |

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
