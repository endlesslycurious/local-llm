# Local LLM

Personal setup and model log for running LLMs locally on Apple Silicon.

> **Reference guide:** [Best Local LLMs for Mac in 2026](https://insiderllm.com/guides/best-local-llms-mac-2026/) — model picks by memory tier, real performance numbers, hardware buying advice.

## Hardware

| | |
|---|---|
| **Machine** | Mac Studio M2 Max |
| **RAM** | 32 GB unified memory |
| **Memory bandwidth** | ~400 GB/s |
| **Tier** | 32–48 GB — runs Qwen 3.6-35B-A3B Q4 comfortably |

## Contents

| File | Description |
|------|-------------|
| [`Setup.md`](Setup.md) | llama.cpp appliance setup with a dedicated `bender` service account |
| [`Models.md`](Models.md) | Personal log of models tested — quant, speed, source URL, notes |
| [`daemons/`](daemons/) | Per-model LaunchDaemon plists — copy and rename to deploy a model |

## Quick start

```bash
# Ollama — simplest setup, works as an API server
curl -fsSL https://ollama.com/install.sh | sh
ollama run qwen3.6:35b-a3b-q4_K_M

# MLX-LM — fastest on Apple Silicon
pip install mlx-lm
mlx_lm.generate --model mlx-community/Qwen3.6-35B-A3B-4bit --prompt "Hello"
```

For the full appliance setup (launchd service, service account, llama.cpp), see [`Setup.md`](Setup.md).

## Memory tier cheat sheet

| RAM | Recommended model | Notes |
|-----|-------------------|-------|
| 8 GB | Gemma 4 E2B or Qwen 3.5 4B | ~5 GB available; platform is outgrowing this tier |
| 16 GB | Qwen 3.5 9B Q4 (~6.6 GB) | Best all-rounder; `/think` mode for chain-of-thought |
| 24 GB | Qwen 3.6-27B Q4_K_M (tight) or Qwen 3 14B (safe) | 16.8 GB model leaves ~5 GB for OS + context |
| **32–48 GB** | **Qwen 3.6-35B-A3B Q4** (~20 GB) | **2026 default — this machine's tier** — MoE, only 3B active params per token |
| 48–64 GB | Qwen 3.6-35B-A3B Q8 + Qwen 3.6-27B Q6/Q8 | Higher quant; best local coding setup |
| 96–128 GB | DeepSeek V4-Flash Q4 (~140 GB, aspirational) | No independent Mac benchmarks yet as of Apr 2026 |
| 192 GB+ | DeepSeek V4-Flash Q4–Q6 | Research-grade; 1M context window |

**Rule of thumb:** model file ≤ 60–70% of total RAM to leave room for macOS, KV cache, and framework overhead.
