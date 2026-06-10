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

All tested models run at `--ctx-size 32768` (32K). 64K was tested and works but runs the system closer to the memory ceiling — 32K gives ~1 GB more headroom and is sufficient for coding assistant use.

**Baseline (llama.cpp not running):** ~91% free, ~1.5 GB wired. Activity Monitor reports ~13 GB "used" at idle, but the majority is reclaimable file cache — not real pressure.

### KV cache quantization

| Model | ctx | KV quant | Truly free (peak) | Compressor | Swap | Verdict |
|-------|-----|----------|-------------------|------------|------|---------|
| Gemma 4 26B Q4 | 64K | none | ~67 MB | ~1.5 GB | 0 | Tight |
| Gemma 4 26B Q4 | 64K | q8_0 | ~875 MB | ~1.4 GB | 0 | Comfortable |
| Qwen 3.6-35B Q4 | 64K | none | ~65 MB | ~3.4 GB | 0 | Marginal |
| Qwen 3.6-35B Q4 | 64K | q8_0 | ~2.3 GB | ~1.6 GB | 0 | Comfortable |
| Qwen 3.6-35B Q4 | 32K | q8_0 | ~3.4 GB | ~430 MB | 0 | Comfortable |

Qwen uses asymmetric KV quantization (`-ctk q8_0 -ctv q4_0`) to reduce memory further. Gemma uses `-ctk q8_0 -ctv q8_0`.

### Sampling parameters

Both models use the same sampling settings:

| Parameter | Value | Reasoning |
|---|---|---|
| `--temp` | 0.6 | Balanced for agentic coding use — low enough for precise code, high enough for natural reasoning. Values below 0.3 risk hobbling thinking and increasing loop likelihood. |
| `--top-p` | 0.9 | Slightly tighter token selection than 0.95; minimal effect at temp 0.6 but consistent with coding-focused defaults. |
| `--repeat-penalty` | 1.03 | Very mild — enough to nudge out of loops without degrading code generation. Aggressive values hurt repetitive-by-nature constructs like variable names and brackets. |
| `--repeat-last-n` | 128 | Moderate lookback window — catches short phrase repetition and reasoning loops. 64 was too short to catch multi-sentence loops; 256+ risks degrading code with naturally repetitive constructs. |
| `--flash-attn` | — | Optimised attention computation — lower peak memory during inference, required for quantised V cache. Well supported on Apple Silicon via Metal. |
| `--threads` | 8 | Matches M2 Max performance core count. |

### Per-model differences

| | Gemma 4 26B | Qwen 3.6-35B |
|---|---|---|
| `-ctv` | q8_0 | q4_0 (extra memory saving) |
| Model size | ~15 GB | ~20 GB |

---

## Future Models

Models from the insiderllm guide worth pulling next, suited to the 32–48 GB tier:

| Model | Quant | File size | Source | Why |
|-------|-------|-----------|--------|-----|
| Qwen 3.6-27B | Q4_K_M | ~16.8 GB | [unsloth GGUF](https://huggingface.co/unsloth/Qwen3.6-27B-GGUF) | Best local coding model; 25.57 tok/s per Simon Willison |
| Gemma 4 26B-A4B | Q8 | ~28 GB | [unsloth GGUF](https://huggingface.co/unsloth/gemma-4-26B-A4B-it-GGUF) | Higher-quality quant of the current appliance model |
