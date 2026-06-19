# Tuning

Performance configuration for this Mac Studio (M2 Max, 32 GB).

---

## Wired Memory Limit

The macOS GPU driver allocates from a pool of wired (non-pageable) memory. The default limit is conservative — on a 32 GB Mac it is typically ~25 GB, and large model allocations can be rejected even when physical memory is available, causing a silent fallback to CPU inference. Raising the MacOS GPU RAM limit to give oMLX more room, see this [dev-note](https://github.com/ivanopcode/devnote-override-macos-metal-vram-cap) for more details.

1. First copy the supplied plist which sets the limit to `28` GB.
```bash
sudo cp ./daemons/llm.iogpu.wired-limit.plist /Library/LaunchDaemons/llm.iogpu.wired-limit.plist
```

2. Load the plist:

```bash
sudo chown root:wheel /Library/LaunchDaemons/llm.iogpu.wired-limit.plist
sudo chmod 644 /Library/LaunchDaemons/llm.iogpu.wired-limit.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/llm.iogpu.wired-limit.plist
```

3. Verify it took effect:

```bash
sysctl iogpu.wired_limit_mb
# iogpu.wired_limit_mb: 22938
```

---

## Context Window

The KV cache grows linearly with context length. On 32 GB with models in the 15–20 GB range:

| Model | Recommended ctx | Reason |
|-------|-----------------|--------|
| Gemma 4 26B Q4 (~15 GB) | 64K | Comfortable headroom |
| Qwen 3.6-27B Q4_K_XL (~17.6 GB) | 48K | More room than 32K; 64K pushes total wired too close to the ceiling on 32 GB |
| Qwen 3.6-35B-A3B Q4 (~20 GB) | 32K | 64K leaves only ~2.3 GB free |

64K is the practical ceiling on 32 GB for models in this weight range — do not go higher.

---

## KV Cache Quantisation (TurboQuant)

Quantising the KV cache (TurboQuant) halves its memory footprint with negligible quality loss (perplexity increase < 0.1 at `q8_0`). On oMLX 4 bit is currently [bugged](https://github.com/jundot/omlx/issues/705).

**Measured memory at peak inference:**

| Model | ctx | KV quant | Truly free (peak) | Compressor | Swap | Verdict |
|-------|-----|----------|-------------------|------------|------|---------|
| Gemma 4 26B Q4 | 64K | none | ~67 MB | ~1.5 GB | 0 | Tight |
| Gemma 4 26B Q4 | 64K | q8_0 | ~875 MB | ~1.4 GB | 0 | Comfortable |
| Qwen 3.6-35B Q4 | 64K | none | ~65 MB | ~3.4 GB | 0 | Marginal |
| Qwen 3.6-35B Q4 | 64K | q8_0 | ~2.3 GB | ~1.6 GB | 0 | Comfortable |
| Qwen 3.6-35B Q4 | 24K | q8_0/q8_0 | ~3.4 GB (est.) | — | 0 | Comfortable |
| Qwen 3.6-27B Q4_K_XL | 48K | q4_0/q4_0 | ~6 GB (est.) | — | 0 | More headroom, still aggressive |

---

## Model-Specific Configuration

See [model settings](./config/model_settings.json) for the per model configuration in use.
