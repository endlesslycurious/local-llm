# Models Tried

Personal log of every model pulled and tested on the Mac Studio M2 Max (32 GB). See [README.md](README.md) for the tier cheat sheet, or the [insiderllm guide](https://insiderllm.com/guides/best-local-llms-mac-2026/) for current recommendations.

Add a row each time you pull and test a new model. Link directly to the HuggingFace repo so the model can be re-pulled later.

---

## Model installation

`oMLX` can search for and download models itself from [Hugging Face](https://huggingface.co) via the admin dashboard, all that is needed is either the repo ID or the name.

---

## Log

| Model | Quant | File size | Context | Speed (tok/s) | Notes | Date |
|-------|-------|-----------|---------|---------------|-------|------|
| [Gemma 4 26B-A4B](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | 4bit | ~15 GB | - | — | Initial appliance model; 4B active params | 2026-05 |
| [Qwen 3.6-35B-A3B](https://ollama.com/library/qwen3.6) | 4bit | ~20 GB | - | — | 2026 default MoE — 3B active params | 2026-05 |
| [Qwen 3.6-27B](https://huggingface.co/unsloth/Qwen3.6-27B-GGUF) | 4bit | ~17.6 GB | 49K | — | Best local coding model; SWE-bench 77.2; more headroom than 35B-A3B; ~25 tok/s (Simon Willison) | 2026-06 |
| [Qwen 3.6-27B (MLX)](https://huggingface.co/mlx-community/Qwen3.6-27B-4bit) | 4bit | — | 49K | — | MLX variant; structured tool calls support | 2026-06 |
| [Qwen3-Coder-30B-A3B-Instruct](https://huggingface.co/unsloth/Qwen3-Coder-30B-A3B-Instruct-GGUF) | 4bit | ~17.7 GB | 131K | — | Coding-specialised model; requires specific flags for tool calls | 2026-06 |
| [Qwen3.5-9B](https://huggingface.co/mlx-community/Qwen3.5-9B-4bit-mlx) | 4bit | ~5.3 GB | 131K | — | Current default/pinned model; excellent speed-to-capability ratio | 2026-06 |

---

## Config notes

See [Tuning.md](Tuning.md) for detailed configuration information.

---

## Future Models

Models from the insiderllm guide worth pulling next, suited to the 32–48 GB tier:

| Model | Quant | File size | Source | Why |
|-------|-------|-----------|--------|-----|
| Gemma 4 26B-A4B | Q8 | ~28 GB | [unsloth GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Higher-quality quant of the current appliance model |
