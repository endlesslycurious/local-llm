# Models Tried

Personal log of every model pulled and tested on the Mac Studio M2 Max (32 GB). See [README.md](README.md) for the tier cheat sheet, or the [insiderllm guide](https://insiderllm.com/guides/best-local-llms-mac-2026/) for current recommendations.

Add a row each time you pull and test a new model. Link directly to the HuggingFace repo or Ollama library page so the model can be re-pulled later.

---

## Log

| Model | Quant | File size | Source | Runtime | Speed (tok/s) | Notes | Date |
|-------|-------|-----------|--------|---------|---------------|-------|------|
| [Gemma 4 26B-A4B](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Q4_K_M (UD) | ~15 GB | [unsloth/gemma-4-26B-A4B-it-GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | llama.cpp | — | Initial appliance model; 4B active params, 256K context | 2026-05 |
| [Qwen 3.6-35B-A3B](https://ollama.com/library/qwen3.6) | Q4_K_M (UD) | ~20 GB | [unsloth GGUF](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF) / [ollama](https://ollama.com/library/qwen3.6) | llama.cpp | — | 2026 default MoE — 3B active params, 262K context | 2026-05 |

---

## Config notes

All tested models run at `--ctx-size 65536` (64K) with `-ctk q8_0 -ctv q8_0` KV cache quantization. Measured memory pressure with llama.cpp on 32 GB:

**Baseline (llama.cpp not running):** ~91% free, ~1.5 GB wired. Activity Monitor reports ~13 GB "used" at idle, but the majority is reclaimable file cache — not real pressure.

| Model | ctx | KV quant | Truly free (peak) | Compressor | Swap | Verdict |
|-------|-----|----------|-------------------|------------|------|---------|
| Gemma 4 26B Q4 | 64K | none | ~67 MB | ~1.5 GB | 0 | Tight |
| Gemma 4 26B Q4 | 64K | q8_0 | ~875 MB | ~1.4 GB | 0 | Comfortable |
| Qwen 3.6-35B Q4 | 64K | none | ~65 MB | ~3.4 GB | 0 | Marginal |
| Qwen 3.6-35B Q4 | 64K | q8_0 | ~2.3 GB | ~1.6 GB | 0 | Comfortable |

KV cache quantization (`q8_0`) halves KV cache memory with negligible quality impact — use it at 64K on 32 GB.

---

## Future Models

Models from the insiderllm guide worth pulling next, suited to the 32–48 GB tier:

| Model | Quant | File size | Source | Why |
|-------|-------|-----------|--------|-----|
| Qwen 3.6-27B | Q4_K_M | ~16.8 GB | [unsloth GGUF](https://huggingface.co/unsloth/Qwen3.6-27B-GGUF) | Best local coding model; 25.57 tok/s per Simon Willison |
| Gemma 4 26B-A4B | Q8 | ~28 GB | [unsloth GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Higher-quality quant of the current appliance model |
