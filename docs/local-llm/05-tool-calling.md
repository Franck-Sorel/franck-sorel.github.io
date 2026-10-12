---
layout: default
title: Test Tool-Calling Reliability
parent: Local LLM Guide
nav_order: 5
description: "Measure how reliably a model returns a valid tool call instead of prose — the number that predicts agent behavior."
---

# Test Tool-Calling Reliability

**Goal of this page:** the quantitative gate. A model that "answers well in chat" is **not the same claim** as a model that reliably drives an agent. The difference: whether it returns a correctly-formed function call instead of prose, *consistently*, under a fixed test.

If a model returns a valid tool call N% of the time on a simple, unambiguous single-tool task, expect roughly that ceiling — or worse, compounded across steps — on real multi-step agent runs.

## Run the test

```bash
# control — expect a low score, run it deliberately:
python3 scripts/test-tool-calling.py --model qwen2.5-coder:7b --runs 20
python3 scripts/test-tool-calling.py --model qwen-coder-agent --runs 20
python3 scripts/test-tool-calling.py --model qwen3-agent --runs 20
```

{: .warning }
Below **~70% valid** on this single-tool control task, expect an agent harness (multi-step, multi-tool) to feel unreliable in practice on this hardware tier.

## Read the control run correctly

The control case (plain `qwen2.5-coder:7b`) should score **visibly worse** than the two tool-tuned candidates. If it doesn't, something else is misconfigured:

- check the Modelfile actually built (`ollama list`),
- check `ollama show <model>` lists `tools` under Capabilities.

That's a misconfiguration signal — not a reason to trust plain `qwen2.5-coder` with an agent.

## The script: `test-tool-calling.py`

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
