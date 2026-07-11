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

- I've increased the MacOS GPU RAM limit to 28 GB to be able to run this configuration!
- An editor or agent that has context pruning *helps allot*; I use [OpenCode](https://opencode.ai) with pruning enabled, mostly via [Zed](https://zed.dev).
- oMLX caching helps *allot* and Speculative Prefill saves time once context grows large.
  - Without Spec-Prefill response times are dominated by prompt processing once context grows.
  - Spec-Prefill does bypass caching though, so a balence is required to benefit from both.

### Daily Driver - Qwen 3.5 9B (4 bit) - 38 tok/s

- Using 4-bit turboquant allows this model to have 128k context on the host machine.
- Thinking mode disabled for more direct, efficient responses.
- Speculative Prefill using `Qwen 3.5 0.8B` when context sent is > 48k, helps reduce waiting.
- Set as default model for daily use, although it does struggle with complexity at times.

### Deep Thinker - Qwen 3.6 27B (4 bit) - 17 tok/s

- A more capable model but due to its large size it can only have 32k of the context, even with 4-bit turboquant!
- Thinking mode enabled for increased capability.
- No Speculative Prefill as RAM is very limited due to model + context size.

## Reference Links

| Article | What it covers |
|---------|----------------|
| [Best Local LLMs for Mac in 2026](https://insiderllm.com/guides/best-local-llms-mac-2026/) | Model picks by memory tier, real performance numbers, hardware buying advice |
| [Qwen 3.6 Complete Guide](https://insiderllm.com/guides/qwen-3-6-local-ai-guide/) | 27B dense vs 35B-A3B MoE trade-offs, VRAM fit, backend compatibility, sampling params |
