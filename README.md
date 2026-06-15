# Local LLM

Personal setup and model log for running LLMs locally on Apple Silicon.

## Requirements

| | |
|---|---|
| **Context window** | 64K (65536 tokens) — tested on both current models |
| **Runtime** | llama.cpp via Homebrew (`brew install llama.cpp`) |

> **Note:** Models range from 15–20 GB. 64K context fits on 32 GB only with KV cache quantization (`-ctk q8_0 -ctv q8_0`) — see [`Models.md`](Models.md) for measured memory pressure data.

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
| [`bin/`](bin/) | Convienance scripts for the llm host machine |
| [`daemons/`](daemons/) | Per-model LaunchDaemon plists — copy and rename to deploy a model |
| [`Models.md`](Models.md) | Personal log of models tested — quant, speed, source URL, notes |
| [`Setup.md`](Setup.md) | llama.cpp appliance setup with a dedicated `bender` service account |
| [`Tuning.md`](Tuning.md) | Performance flags, wired memory limit, KV cache quantisation, sampling parameters |

## Quick start

```bash
# Ollama — simplest setup, works as an API server
curl -fsSL https://ollama.com/install.sh | sh
ollama run qwen3.6:35b-a3b-q4_K_M

# MLX-LM — fastest on Apple Silicon
uv tool install mlx-lm
mlx_lm.generate --model mlx-community/Qwen3.6-35B-A3B-4bit --prompt "Hello"
```

For the full appliance setup (launchd service, service account, llama.cpp), see [`Setup.md`](Setup.md).

## Memory tier cheat sheet

| RAM | Recommended model | Notes |
|-----|-------------------|-------|
| 8 GB | Gemma 4 E2B or Qwen 3.5 4B | ~5 GB available; platform is outgrowing this tier |
| 16 GB | Qwen 3.5 9B Q4 (~6.6 GB) | Best all-rounder; `/think` mode for chain-of-thought |
| 24 GB | Qwen 3.6-27B Q4_K_XL (tight) or Qwen 3 14B (safe) | 17.6 GB model leaves ~5 GB for OS + context |
| **32–48 GB** | **Qwen 3.6-35B-A3B Q4** (~20 GB) or **Qwen3-Coder-30B-A3B-Instruct UD-Q4_K_XL** | **2026 default — this machine's tier** — MoE, only 3B active params per token; coder variant is the higher-quality code-focused pick |
| 48–64 GB | Qwen 3.6-35B-A3B Q8 + Qwen 3.6-27B Q6/Q8 | Higher quant; best local coding setup |
| 96–128 GB | DeepSeek V4-Flash Q4 (~140 GB, aspirational) | No independent Mac benchmarks yet as of Apr 2026 |
| 192 GB+ | DeepSeek V4-Flash Q4–Q6 | Research-grade; 1M context window |

**Rule of thumb:** model file ≤ 60–70% of total RAM to leave room for macOS, KV cache, and framework overhead.

## Memory Management

Memory pressure builds naturally over long sessions as the KV cache fills. Two habits that keep the system healthy:

**Start a new thread between tasks** — each new conversation resets the KV cache, recovering ~400 MB of wired memory and giving the model a clean context.

**Restart llama-server if pressure builds** — stopping and restarting the service fully recovers memory without a reboot. Compressor and free pages return to clean-start levels within seconds:

```bash
sudo launchctl bootout system/local.llama.server
sudo launchctl bootstrap system /Library/LaunchDaemons/local.llama.server.plist
```

Check recovery with:

```bash
memory_pressure
```

A healthy idle state (model loaded, no active prompt) shows ~25% free, compressor under 500 MB, and zero swap.

## Reference Links

| Article | What it covers |
|---------|----------------|
| [Best Local LLMs for Mac in 2026](https://insiderllm.com/guides/best-local-llms-mac-2026/) | Model picks by memory tier, real performance numbers, hardware buying advice |
| [Qwen 3.6 Complete Guide](https://insiderllm.com/guides/qwen-3-6-local-ai-guide/) | 27B dense vs 35B-A3B MoE trade-offs, VRAM fit, backend compatibility, sampling params |
| [Tuning llama.cpp on Apple Silicon](https://medium.com/@michael.hannecke/tuning-llama-cpp-on-apple-silicon-843f37a6c3dc) | The seven flags that matter on M-series, wired memory limit, what to ignore from x86 guides |
