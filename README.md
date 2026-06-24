# Local LLM

Personal setup and model log for running LLMs locally on Apple Silicon via [oMLX](https://github.com/jundot/omlx).

## My Requirements - Context Window

1. Ideally 128k or more context window for agentic workloads.
2. 48-64k is a managable amount of context for single tasks.
3. 32k is the *bare minimium* amount of context in my experience.

## Repo Contents

| File | Description |
|:------:|-------------|
| [`config/`](config/) | oMLX configuration files |
| [`daemons/`](daemons/) | GPU RAM limit configuration |
| [`Models.md`](Models.md) | Personal log of models tested |
| [`Setup.md`](Setup.md) | oMLX setup on a dedicated Mac host |
| [`Tuning.md`](Tuning.md) | Performance tuning notes |

## My Hardware

| | |
|:---:|---|
| **Machine** | Mac Studio |
| **CPU** | M2 Max |
| **RAM** | 32 GB (Unified) |
| **Memory bandwidth** | ~400 GB/s |
| **Storage** | 1 TB SSD |

## TLDR - My Current Setup

I've increased the MacOS GPU RAM limit to 28 GB to be able to run this configuration!

### Daily Driver - Qwen 3.5 9B (4 bit)

- Using 4-bit turboquant allows this model to have 128k context on the host machine.
- Thinking mode disabled for more direct, efficient responses.
- Speculative Prefill using Qwen 3.5 0.8B when context sent is > 64k, helps reduce waiting.
- Set as default model for daily use.

### Deep Thinker - Qwen 3.6 27B (4 bit)

- A more capable model but due to its large size it can only have a third of the context of 9B.
- Thinking mode enabled for increased capability.
- Not pinned - use when deep thinking/reasoning is needed.

## Reference Links

| Article | What it covers |
|---------|----------------|
| [Best Local LLMs for Mac in 2026](https://insiderllm.com/guides/best-local-llms-mac-2026/) | Model picks by memory tier, real performance numbers, hardware buying advice |
| [Qwen 3.6 Complete Guide](https://insiderllm.com/guides/qwen-3-6-local-ai-guide/) | 27B dense vs 35B-A3B MoE trade-offs, VRAM fit, backend compatibility, sampling params |
