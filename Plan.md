# Plan: oMLX Migration

Migrate from llama.cpp / mlx-openai-server to oMLX as the sole inference server, targeting **128K context** with Qwen3-Coder-30B-A3B for agentic coding sessions.

---

## Phase 1 — Experiment (Current Machine)

Validate that 128K context is viable on 32 GB before committing to a reinstall. Run everything over SSH on the current Mac Studio config — no plists, no service changes.

### 1.1 Install oMLX

```bash
brew tap jundot/omlx https://github.com/jundot/omlx
brew install omlx
```

### 1.2 Download the Model

```bash
sudo bin/fetch-mlx mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit
```

Model is ~17.2 GB. Lands in `/Users/bender/models/mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit/`.

### 1.3 Create SSD Cache Directory

```bash
sudo mkdir -p /Users/bender/cache
sudo chown bender:staff /Users/bender/cache
```

### 1.4 Run oMLX (Foreground)

SSH into the Mac Studio and run directly — no plist needed:

```bash
sudo -u bender omlx serve \
  --model-dir /Users/bender/models \
  --host 0.0.0.0 \
  --port 8080 \
  --memory-guard aggressive \
  --memory-guard-gb 24 \
  --hot-cache-max-size 4GB \
  --paged-ssd-cache-dir /Users/bender/cache \
  --no-hf-cache \
  --max-concurrent-requests 1
```

### 1.5 Configure the Model (Admin Panel)

Open `http://freyr.local:8080/admin` and set per-model settings for Qwen3-Coder-30B-A3B-Instruct-4bit:

| Setting | Value |
|---|---|
| Context length | 131072 |
| KV bits | 4 |
| Temperature | 0.7 |
| Top-p | 0.8 |
| Top-k | 20 |
| Model alias | `qwen3-coder` |
| Pin model | Yes |

### 1.6 Benchmark

Use the admin panel's built-in benchmark tool to measure:

- Prefill tokens/sec (PP) — especially at long context where SSD spill kicks in
- Text generation tokens/sec (TG)
- Partial prefix cache hit performance (simulates multi-turn conversation)

### 1.7 Test from Zed

Update Zed `settings.json` to point at oMLX:

```json
"openai_compatible": {
  "Freyr": {
    "api_url": "http://freyr.local:8080/v1",
    "available_models": [
      {
        "name": "qwen3-coder",
        "max_tokens": 131072,
        "max_output_tokens": 16384,
        "max_completion_tokens": 16384,
        "capabilities": {
          "tools": true,
          "images": false,
          "parallel_tool_calls": false,
          "prompt_cache_key": false,
          "chat_completions": true,
          "interleaved_reasoning": false
        }
      }
    ]
  }
}
```

Run an actual agentic coding session. Pay attention to:

- Does tool calling work out of the box? (oMLX auto-detects Qwen tool format)
- How does latency feel when context grows past what fits in hot cache?
- Does `memory_pressure` on the Mac Studio stay healthy?

### 1.8 Decide

| Outcome | Next step |
|---|---|
| 128K works well | Proceed to Phase 2 |
| 128K is too slow at long context | Try 96K or 64K with same setup, then proceed to Phase 2 |
| oMLX itself has issues | Debug or wait for fixes before Phase 2 |

---

## Phase 2 — Clean Reinstall

Contingent on Phase 1 results. Rebuild the Mac Studio as a dedicated oMLX appliance.

### 2.1 Fresh macOS Install

- Wipe & reinstall macOS
- Create single account: `bender` (admin, not signed into Apple/iCloud)
- Skip FileVault
- Enable auto-login: System Settings → Users & Groups → Automatic login → bender

### 2.2 Install Tooling

```bash
# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# oMLX
brew tap jundot/omlx https://github.com/jundot/omlx
brew install omlx

# Hugging Face CLI (for bin/fetch)
brew install huggingface-cli
```

### 2.3 Directory Layout

```
/Users/bender/
├── models/        ← model weights (--model-dir)
├── cache/         ← KV cold tier (--paged-ssd-cache-dir)
└── .omlx/         ← oMLX config & logs (managed by oMLX)
```

### 2.4 Configure oMLX

Run once to persist settings:

```bash
omlx serve \
  --model-dir /Users/bender/models \
  --host 0.0.0.0 \
  --port 8080 \
  --memory-guard aggressive \
  --memory-guard-gb 24 \
  --hot-cache-max-size 4GB \
  --paged-ssd-cache-dir /Users/bender/cache \
  --no-hf-cache \
  --max-concurrent-requests 1
```

Settings persist to `~/.omlx/settings.json`. Stop the foreground server, then start as a service:

```bash
omlx start
```

Or equivalently:

```bash
brew services start omlx
```

Service auto-restarts on crash and starts on login (which happens automatically via auto-login).

### 2.5 Download Models

```bash
bin/fetch mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit
```

Configure per-model settings via admin panel at `http://freyr.local:8080/admin`.

### 2.6 Verify Appliance Behaviour

Reboot the Mac Studio and confirm:

```bash
# From MacBook
curl http://freyr.local:8080/v1/models
```

If the model list comes back, the appliance boots to a working inference server with no human intervention.

### 2.7 Update Repo

Once the new setup is validated:

- [ ] Rewrite `Setup.md` — clean install, single account, brew services
- [ ] Rewrite `Tuning.md` — oMLX memory guard, KV tiers, per-model admin panel settings
- [ ] Update `README.md` — reflect oMLX as sole runtime, update cheat sheet
- [ ] Update `Models.md` — add MLX Qwen3-Coder entry, note GGUF models are archived
- [ ] Replace `bin/fetch` contents with current `bin/fetch-mlx` logic
- [ ] Remove `bin/fetch-mlx`, `bin/deploy`, `bin/reload`
- [ ] Archive llama.cpp plists (move `daemons/*.plist` to `archive/daemons/`, keep `daemons/omlx.plist` as reference)
- [ ] Track oMLX config in git — `config/settings.json` and `config/model_settings.json` symlinked into `~/.omlx/`

---

## Open Questions

- **Hot cache size**: 4GB is a conservative starting point (~12.5% of 32 GB), leaving ~6.8 GB headroom for MLX overhead. If benchmarks show SSD spill hurting prefill latency, try 6GB.
- **SpecPrefill**: Attention-based sparse prefill for MoE models ([paper](https://arxiv.org/abs/2502.02789)). Qwen3-Coder-30B-A3B is MoE (30B total, 3B active) so this applies. Toggle in admin panel — could meaningfully speed up prefill at 128K context. Test after baseline is working.
- **oQ quantization**: Could re-quantize from bf16 using oMLX's built-in oQ4 for better quality at same size. Worth trying after Phase 1 if baseline quality feels lacking.
- **Context scaling**: oMLX has a Claude Code optimization that scales reported token counts so auto-compact triggers correctly. Worth investigating if Zed has similar behaviour.
