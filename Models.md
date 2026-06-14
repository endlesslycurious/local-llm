# Models Tried

Personal log of every model pulled and tested on the Mac Studio M2 Max (32 GB). See [README.md](README.md) for the tier cheat sheet, or the [insiderllm guide](https://insiderllm.com/guides/best-local-llms-mac-2026/) for current recommendations.

Add a row each time you pull and test a new model. Link directly to the HuggingFace repo or Ollama library page so the model can be re-pulled later.

---

## Log

| Model | Quant | File size | Source | Runtime | Speed (tok/s) | Notes | Date |
|-------|-------|-----------|--------|---------|---------------|-------|------|
| [Gemma 4 26B-A4B](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Q4_K_M (UD) | ~15 GB | [unsloth/gemma-4-26B-A4B-it-GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | llama.cpp | — | Initial appliance model; 4B active params, 256K context | 2026-05 |
| [Qwen 3.6-35B-A3B](https://ollama.com/library/qwen3.6) | Q4_K_M (UD) | ~20 GB | [unsloth GGUF](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF) / [ollama](https://ollama.com/library/qwen3.6) | llama.cpp | — | 2026 default MoE — 3B active params, 262K context | 2026-05 |
| [Qwen 3.6-27B](https://huggingface.co/unsloth/Qwen3.6-27B-GGUF) | Q4_K_XL (UD) | ~17.6 GB | [unsloth/Qwen3.6-27B-GGUF](https://huggingface.co/unsloth/Qwen3.6-27B-GGUF) | llama.cpp | — | Best local coding model; SWE-bench 77.2; more headroom than 35B-A3B; ~25 tok/s (Simon Willison measured 25.57 on non-UD Q4_K_M — UD-Q4_K_XL will be marginally slower at 17.6 GB) | 2026-06 |
| [Qwen 3.6-27B (MLX)](https://huggingface.co/mlx-community/Qwen3.6-27B-4bit) | 4bit | — | [mlx-community/Qwen3.6-27B-4bit](https://huggingface.co/mlx-community/Qwen3.6-27B-4bit) | mlx-openai-server | — | MLX variant for the appliance-style OpenAI-compatible server | 2026-06 |

---

## Config notes

See [Tuning.md](Tuning.md) for flags, KV cache quantisation, sampling parameters, and per-model settings.

**Baseline (llama.cpp not running):** ~91% free, ~1.5 GB wired. Activity Monitor reports ~13 GB "used" at idle, but the majority is reclaimable file cache — not real pressure.

---

## Future Models

Models from the insiderllm guide worth pulling next, suited to the 32–48 GB tier:

| Model | Quant | File size | Source | Why |
|-------|-------|-----------|--------|-----|
| Gemma 4 26B-A4B | Q8 | ~28 GB | [unsloth GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Higher-quality quant of the current appliance model |
