# Local LLM

Personal setup and model log for running LLMs locally on Apple Silicon via [oMLX](https://github.com/jundot/omlx).

## Requirements

### Context Window

1. Ideally 128k or more context window for agentic workloads, this will only be possible with oMLX's offloading to SSD (disc) feature.
2. 48-64k is a very managable amount of context for single tasks.
3. 32k is the bare minimium in my experience.

## Hardware

| | |
|---|---|
| **Machine** | Mac Studio M2 Max |
| **RAM** | 32 GB unified memory |
| **Memory bandwidth** | ~400 GB/s |
| **Tier** | 32–48 GB — runs Qwen 3.6-35B-A3B Q4 comfortably |

## Repo Contents

| File | Description |
|------|-------------|
| [`daemons/`](daemons/) | GPU RAM limit configuration |
| [`Models.md`](Models.md) | Personal log of models tested — quant, speed, source URL, notes |
| [`Setup.md`](Setup.md) | oMLX setup on a dedicated Mac host |
| [`Tuning.md`](Tuning.md) | Performance flags, wired memory limit, KV cache quantisation, sampling parameters |

## Memory tier cheat sheet

| RAM | Recommended model | Notes |
|-----|-------------------|-------|
| 8 GB | Gemma 4 E2B or Qwen 3.5 4B | ~5 GB available; platform is outgrowing this tier |
| 16 GB | Qwen 3.5 9B Q4 (~6.6 GB) | Best all-rounder; `/think` mode for chain-of-thought |
| 24 GB | Qwen 3.6-27B Q4_K_XL (tight) or Qwen 3 14B (safe) | 17.6 GB model leaves ~5 GB for OS + context |
| **32–48 GB** | Qwen 3.6-35B-A3B Q4 (~20 GB) or Qwen3-Coder-30B-A3B-Instruct UD-Q4_K_XL | 2026 default — this machine's tier — MoE, only 3B active params per token; coder variant is the higher-quality code-focused pick |
| 48–64 GB | Qwen 3.6-35B-A3B Q8 + Qwen 3.6-27B Q6/Q8 | Higher quant; best local coding setup |
| 96–128 GB | DeepSeek V4-Flash Q4 (~140 GB, aspirational) | No independent Mac benchmarks yet as of Apr 2026 |
| 192 GB+ | DeepSeek V4-Flash Q4–Q6 | Research-grade; 1M context window |

**Rule of thumb:** model file ≤ 60–70% of total RAM to leave room for macOS, KV cache, and framework overhead.

## Memory Management

Memory pressure builds naturally over long sessions as the KV cache fills. Two habits that keep the system healthy:

**Start a new thread between tasks** — each new conversation resets the KV cache, recovering ~400 MB of wired memory and giving the model a clean context.

Check resource usage with oMLX dashboard, btop or `memory_pressure` command:

For memory pressure, a healthy idle state (model loaded, no active prompt) shows ~25% free, compressor under 500 MB, and zero swap.

## Reference Links

| Article | What it covers |
|---------|----------------|
| [Best Local LLMs for Mac in 2026](https://insiderllm.com/guides/best-local-llms-mac-2026/) | Model picks by memory tier, real performance numbers, hardware buying advice |
| [Qwen 3.6 Complete Guide](https://insiderllm.com/guides/qwen-3-6-local-ai-guide/) | 27B dense vs 35B-A3B MoE trade-offs, VRAM fit, backend compatibility, sampling params |
