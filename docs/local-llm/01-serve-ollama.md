---
layout: default
title: Serve Ollama in Docker
parent: Local LLM Guide
nav_order: 1
description: "Run the Ollama server in Docker with NVIDIA GPU passthrough."
---

# Serve Ollama in Docker

**Goal of this page:** a running Ollama server on `http://localhost:11434`, with the container seeing your GPU. Docker keeps the serving stack reproducible and isolated — no CUDA toolkit soup on the host.

## Check the GPU gate first

Before anything else, verify the container can see the GPU. If it can't, nothing downstream will work — it's like checking the oven heats before mixing batter.

{: .warning }
A container that cannot see the GPU will still start and serve models — on the CPU, 5–10× slower. Don't discover this two benchmarks in: gate every node before touching Compose.

```bash
./scripts/verify-node.sh   # gate: don't proceed if this fails
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv,noheader
```

## Start the server

```bash
cd docker
docker compose -f docker-compose.ollama.yml up -d
```

Ollama now listens on `http://localhost:11434` and exposes an OpenAI-compatible API.

## Pull a first model inside the container

```bash
docker exec -it $(docker compose -f docker-compose.ollama.yml ps -q ollama) \
  ollama pull qwen2.5-coder:7b
```

## Confirm what actually loaded — and how

```bash
ollama list
ollama ps    # PROCESSOR column must read "100% GPU"
```

{: .danger}
If `ollama ps` shows a **CPU/GPU split**, the model didn't fully fit and speed drops sharply. Drop to a smaller model or lower `num_ctx` — don't try to troubleshoot the split. A partially-offloaded model makes every later benchmark and reliability number meaningless.

## Build agent models in the same container

The pinned-context agent models (next pages) should be built inside the same container so they land in the same Ollama data volume:

```bash
docker cp ollama/Modelfile.qwen-coder-agent <container>:/tmp/
docker cp ollama/Modelfile.qwen3-agent      <container>:/tmp/
docker exec -it <container> ollama create qwen-coder-agent -f /tmp/Modelfile.qwen-coder-agent
docker exec -it <container> ollama create qwen3-agent       -f /tmp/Modelfile.qwen3-agent
```
