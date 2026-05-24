# Models Tried

Personal log of every model pulled and tested on the Mac Studio M2 Max (32 GB). See [README.md](README.md) for the tier cheat sheet, or the [insiderllm guide](https://insiderllm.com/guides/best-local-llms-mac-2026/) for current recommendations.

Add a row each time you pull and test a new model. Link directly to the HuggingFace repo or Ollama library page so the model can be re-pulled later.

---

## Log

| Model | Quant | File size | Source | Runtime | Speed (tok/s) | Notes | Date |
|-------|-------|-----------|--------|---------|---------------|-------|------|
| [Gemma 4 26B-A4B](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Q4_K_M (UD) | ~15 GB | [unsloth/gemma-4-26B-A4B-it-GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | llama.cpp | — | Initial appliance model; 4B active params, 256K context | 2026-05 |

---

## On the radar

Models from the insiderllm guide worth pulling next, suited to the 32–48 GB tier:

| Model | Quant | File size | Source | Why |
|-------|-------|-----------|--------|-----|
| Qwen 3.6-35B-A3B | Q4_K_M | ~20 GB | [ollama](https://ollama.com/library/qwen3.6) / [mlx-community](https://huggingface.co/mlx-community/Qwen3.6-35B-A3B-4bit) | 2026 default MoE — 3B active params, 35–55 tok/s, 262K context |
| Qwen 3.6-27B | Q4_K_M | ~16.8 GB | [unsloth GGUF](https://huggingface.co/unsloth/Qwen3.6-27B-GGUF) | Best local coding model; 25.57 tok/s per Simon Willison |
| Gemma 4 26B-A4B | Q8 | ~28 GB | [unsloth GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Higher-quality quant of the current appliance model |
