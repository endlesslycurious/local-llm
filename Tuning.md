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

## oMLX Settings

oMLX configuration settings that control runtime behavior, memory management, and caching.

### Memory Settings

Memory management is critical for oMLX on Apple Silicon. The memory_guard settings control how oMLX interacts with the system's memory pressure.

| Setting | Value | Description |
|:--------:|:------:|:------------|
| `prefill_memory_guard` | `true` | Apply memory guard during prefill phase |
| `memory_guard_tier` | `aggressive` | Use aggressive memory guard tier |
| `soft_threshold` | `0.85` | Memory pressure threshold for soft reclaim |
| `hard_threshold` | `0.95` | Memory pressure threshold for hard reclaim |
| `prefill_safe_zone_ratio` | `0.8` | Fraction of soft threshold as safe zone |
| `prefill_min_chunk_tokens` | `32` | Minimum tokens per chunk for batching |
### Scheduler Settings (Concurrency)

Scheduler settings control how oMLX handles concurrent requests and batching.

| Setting | Value | Description |
|:--------:|:------:|:------------|
| `max_concurrent_requests` | `1` | Maximum concurrent requests |
| `embedding_batch_size` | `16` | Batch size for embedding requests |
| `chunked_prefill` | `true` | Enable chunked prefill for long contexts |

### Cache Settings

KV cache settings control how oMLX stores intermediate states.

| Setting | Value | Description |
|:--------:|:------:|:------------|
| `enabled` | `true` | Enable KV cache |
| `hot_cache_only` | `false` | Keep both hot and cold cache |
| `ssd_cache_dir` | `/Users/bender/cache` | Directory for SSD cache |
| `ssd_cache_max_size` | `64GB` | Maximum SSD cache size |
| `hot_cache_max_size` | `2GB` | Maximum hot cache size |
| `initial_cache_blocks` | `256` | Number of blocks to pre-allocate |

### Sampling Settings

**Global defaults** (can be overridden per-model in model_settings.json):
| Setting | Default Value |
|:--------|:-------------|
| `max_context_window` | 128k |
| `max_tokens` | 16k |
| `temperature` | 0.7 |
| `top_p` | 0.8 |
| `top_k` | 20 |
| `repetition_penalty` | 1.0 |
| `min_p` | 0.0 |
| `presence_penalty` | 1.5 |

See the [Model Log](#model-log) for configured sampling parameters for each model.

### Speculative Decoding (SpecPrefill)

When enabled, the main model generates its first N tokens using a smaller draft model (speculative decoding). This can significantly speed up generation while maintaining quality.

**Currently configured models:**
- **Qwen3.5-9B**: Uses Qwen3.5-0.8B as drafter
  - Threshold: 64k tokens (triggers specprefill when context > 64k)
  - Draft keep ratio: 20% (keep 20% of draft tokens)
- **Other models**: Speculative decoding not enabled

## Memory Management

Memory pressure builds naturally over long sessions as the KV cache fills. Two habits that keep the system healthy:

**Start a new thread between tasks** — each new conversation resets the KV cache, recovering ~400 MB of wired memory and giving the model a clean context.

Check resource usage with oMLX dashboard, btop or `memory_pressure` command:

For memory pressure, a healthy idle state (model loaded, no active prompt) shows ~25% free, compressor under 500 MB, and zero swap.

## Troubleshooting

### Model runs slowly:
- Check GPU wired limit: `sysctl iogpu.wired_limit_mb`
- Monitor with: `btop` or admin dashboard
- Reduce context window in model settings

### Out of memory:
- Lower context window
- Enable TurboQuant KV cache (4-bit)
- Try smaller model
