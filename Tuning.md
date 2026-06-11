# Tuning

Performance configuration for this Mac Studio (M2 Max, 32 GB). For broader Apple Silicon depth, see [Tuning llama.cpp on Apple Silicon](https://medium.com/@michael.hannecke/tuning-llama-cpp-on-apple-silicon-843f37a6c3dc).

---

## Wired Memory Limit

The macOS GPU driver allocates from a pool of wired (non-pageable) memory. The default limit is conservative — on a 32 GB Mac it is typically ~24 GB, and large model allocations can be rejected even when physical memory is available, causing a silent fallback to CPU inference.

Raise the limit to 70% of unified memory before starting the server:

```bash
sudo sysctl iogpu.wired_limit_mb=22938
```

> **Why 22938?** 70% of 32 GB (32768 MB). Apple's guidance is to stay at or below 70% so macOS retains enough non-pageable memory for system processes.

This setting resets on reboot. To apply it automatically at boot, create a LaunchDaemon:

```bash
sudo nano /Library/LaunchDaemons/local.iogpu.wired-limit.plist
```

Paste:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>local.iogpu.wired-limit</string>
    <key>ProgramArguments</key>
    <array>
      <string>/usr/sbin/sysctl</string>
      <string>iogpu.wired_limit_mb=22938</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
```

Load it:

```bash
sudo chown root:wheel /Library/LaunchDaemons/local.iogpu.wired-limit.plist
sudo chmod 644 /Library/LaunchDaemons/local.iogpu.wired-limit.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/local.iogpu.wired-limit.plist
```

Verify:

```bash
sysctl iogpu.wired_limit_mb
# iogpu.wired_limit_mb: 22938
```

---

## Core llama.cpp Flags

Seven flags do meaningful work on Apple Silicon. Common x86 flags (`--numa`, `--tensor-split`, `--cpu-mask`) are no-ops on this hardware.

| Flag | Value | Effect |
|------|-------|--------|
| `--n-gpu-layers` | `-1` | Offloads all layers to Metal; without it inference falls back to CPU |
| `--flash-attn` | `on` | Metal Flash Attention kernel — faster prefill, smaller memory footprint per token, required for KV cache quantisation |
| `--batch-size` / `--ubatch-size` | `2048` | Faster prefill vs default 512; Apple's own benchmarks recommend this on M-series |
| `--cache-type-k` / `--cache-type-v` | `q8_0` | Halves KV cache memory footprint; requires `--flash-attn on` |
| `--mlock` | — | Pins model and KV cache in RAM; prevents latency spikes under memory pressure |
| `--prio` | `2` | Reduces scheduler interruption on a dedicated inference host |
| `--threads` | `8` | Matches M2 Max performance core count |

`--mlock` is safe on this machine because 16–20 GB model weights + KV cache at 32K sits comfortably under the 70% wired memory ceiling (~22 GB) on a dedicated 32 GB host. At 64K context the KV cache grows to ~5.7 GB, pushing total wired past 23 GB — avoid 64K on any model with `--mlock` on this hardware.

> **Note:** The Gemma and Qwen 3.6-35B-A3B plists currently use `-b 512 -ub 512`. Updating both to `2048` would improve prefill speed and is worth doing on the next model swap.

---

## Context Window

The KV cache grows linearly with context length. On 32 GB with models in the 15–20 GB range:

| Model | Recommended ctx | Reason |
|-------|-----------------|--------|
| Gemma 4 26B Q4 (~15 GB) | 64K | Comfortable headroom |
| Qwen 3.6-27B Q4_K_XL (~17.6 GB) | 32K | Same ctx as 35B-A3B — --mlock wires KV cache; 64K pushes total wired past 23 GB |
| Qwen 3.6-35B-A3B Q4 (~20 GB) | 32K | 64K leaves only ~2.3 GB free |

64K is the practical ceiling on 32 GB for models in this weight range — do not go higher.

**Baseline (llama.cpp not running):** ~91% free, ~1.5 GB wired. Activity Monitor reports ~13 GB "used" at idle, but the majority is reclaimable file cache — not real pressure.

---

## KV Cache Quantisation

Quantising the KV cache halves its memory footprint with negligible quality loss (perplexity increase < 0.1 at `q8_0`). Requires `--flash-attn on`.

**Measured memory at peak inference:**

| Model | ctx | KV quant | Truly free (peak) | Compressor | Swap | Verdict |
|-------|-----|----------|-------------------|------------|------|---------|
| Gemma 4 26B Q4 | 64K | none | ~67 MB | ~1.5 GB | 0 | Tight |
| Gemma 4 26B Q4 | 64K | q8_0 | ~875 MB | ~1.4 GB | 0 | Comfortable |
| Qwen 3.6-35B Q4 | 64K | none | ~65 MB | ~3.4 GB | 0 | Marginal |
| Qwen 3.6-35B Q4 | 64K | q8_0 | ~2.3 GB | ~1.6 GB | 0 | Comfortable |
| Qwen 3.6-35B Q4 | 32K | q8_0 | ~3.4 GB | ~430 MB | 0 | Comfortable |
| Qwen 3.6-27B Q4_K_XL | 32K | q8_0/q8_0 | ~4.7 GB (est.) | — | 0 | Comfortable |

**`-ctv` choice:**

| Setting | Use when |
|---------|----------|
| `-ctv q8_0` | Headroom is comfortable — default for Gemma 4 26B and Qwen 3.6-27B |
| `-ctv q4_0` | Model is large enough to need extra KV headroom — e.g. Qwen 3.6-35B at 32K |

Value cache (`-ctv`) is more sensitive to quantisation than key cache (`-ctk`). Drop to `q4_0` only when needed.

---

## Sampling Parameters

All current models use the same sampling settings:

| Parameter | Value | Reasoning |
|-----------|-------|-----------|
| `--temp` | 0.6 | Balanced for agentic coding — low enough for precise code, high enough for natural reasoning. Values below 0.3 risk hobbling thinking and increasing loop likelihood. |
| `--top-p` | 0.9 | Slightly tighter than 0.95; minimal effect at temp 0.6 but consistent with coding-focused defaults. |
| `--repeat-penalty` | 1.03 | Very mild — nudges out of loops without degrading repetitive-by-nature constructs like variable names and brackets. |
| `--repeat-last-n` | 128 | Moderate lookback — catches short phrase and reasoning loops. 64 was too short; 256+ risks degrading code with naturally repetitive constructs. |

---

## Per-Model Settings

| | Gemma 4 26B | Qwen 3.6-35B-A3B | Qwen 3.6-27B |
|---|---|---|---|
| `-ctv` | `q8_0` | `q4_0` (extra memory saving) | `q8_0` (headroom allows symmetric quant) |
| `-b` / `-ub` | `512` | `512` | `2048` |
| Model size | ~15 GB | ~20 GB | ~17.6 GB |
| ctx target | 64K | 32K | 32K |
