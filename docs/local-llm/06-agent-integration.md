---
layout: default
title: Integrate Coding Agents
parent: Local LLM Guide
nav_order: 6
description: "Two-tier agent test: Aider (soft, diff-based) then Cline (hard, native tool calling)."
---

# Integrate Coding Agents

**Goal of this page:** connect the models to real agents, in two tiers deliberately chosen to stress different things — a forgiving one first, a strict one second.

## Tier A — Aider (soft test)

Aider is a terminal-first, Git-centric coding agent. It's the easier first integration because much of its edit workflow doesn't hinge on perfectly-formed JSON tool calls — it can work from structured text diffs in the prompt/response itself.

```bash
pip install --break-system-packages aider-chat

export OLLAMA_API_BASE=http://127.0.0.1:11434
export OLLAMA_CONTEXT_LENGTH=8192

cd /path/to/a/test/repo
aider --model ollama_chat/qwen-coder-agent
```

{: .warning }
Use the `ollama_chat/` prefix, **not** `ollama/` — Aider's own documentation recommends it, and the wrong prefix has historically caused chat-template mismatches.

**Test task (run this exact prompt inside Aider):**

> Add a `add_numbers(a, b)` function to `utils.py` with a docstring and a unit test in `test_utils.py`. Run the tests after.

Record: did it produce a correct diff, did it apply cleanly, did it actually invoke the test runner (a tool-use step), and how many turns it took versus a correct single-shot response.

## Tier B — Cline (hard test)

Cline is a VS Code extension (with a standalone CLI) that does real, permissioned tool calls — file edits, terminal commands, MCP tools — through the model's **native function-calling interface**, with a Plan/Act approval step before actions run. Unlike Aider, it depends directly on the tool-call reliability measured on the previous page.

1. Install the Cline extension in VS Code.
2. Point its provider at Ollama: `http://localhost:11434`, model `qwen-coder-agent` — then repeat with `qwen3-agent`.
3. Keep Cline's compact-prompt / reduced-context setting **on** — VRAM-constrained runs are exactly what that setting exists for.
4. Run the same test task, in Plan/Act mode, so every proposed tool call is visible before it executes. You're watching the raw tool-call format the model produces, not just the end result.

Record: number of proposed tool calls valid on first try, any loops (the model retrying a malformed call), and whether Plan/Act caught anything Aider's softer format would have silently gotten away with — or silently botched.
