# Mac Studio LLM Appliance Setup (llama.cpp)

> **Hardware:** Mac Studio M2 Max, 32 GB unified memory. See [Models.md](Models.md) for what has been tested, or the [insiderllm guide](https://insiderllm.com/guides/best-local-llms-mac-2026/) for current recommendations.

This guide sets up a dedicated Mac Studio as a local coding assistant using local models via llama.cpp, running under a separate service account (`bender`).

---

## 1. Create Service Account

Create a standard (non-admin) user:

```bash
sudo sysadminctl -addUser bender
```

Notes:
- Do NOT grant admin privileges
- Do NOT use interactively

---

## 2. Install Runtime (Main Account)

### llama.cpp
Install llama.cpp via Homebrew:

```bash
brew install llama.cpp
```

Verify install:

```bash
which llama-server
# /opt/homebrew/bin/llama-server
```

### mlx-openai-server

Curently a dependency of mlx-openai-server requires `Python 3.13` (see issue) and brew is installing `Python 3.14` by default. `Rust` is also required.

```bash
brew install python@3.13 rust
```

Then install mlx-openai-server via `uv`:

```bash
uv tool install mlx-openai-server --python 3.13
```

Verify install:
```bash
mlx-openai-server --version

✨ mlx-openai-server - OpenAI Compatible API Server for MLX models ✨
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚀 Version: 1.8.1
```

For Qwen 3.6 tool use, launch `mlx-openai-server` with `--reasoning-parser qwen3 --tool-call-parser qwen3_coder --enable-auto-tool-choice` so the server returns structured `tool_calls` instead of raw `<tool_call>` text.

---

## 3. Directory Structure

From your main admin account, create the service account's working directories:

```bash
sudo mkdir -p /Users/bender/{models,logs}
sudo chown -R bender:staff /Users/bender/{models,logs}
```

Final layout:

```
/Users/bender/
  models/
  logs/
```

---

## 4. Add Model

From your main account, download a single GGUF file with `bin/fetch`, or an MLX model repo with `bin/fetch-mlx`:

```bash
sudo bin/fetch unsloth/Qwen3.6-27B-GGUF Qwen3.6-27B-UD-Q4_K_XL.gguf
sudo bin/fetch-mlx mlx-community/Qwen3.6-27B-4bit
```

If you need a non-default branch or tag:

```bash
sudo bin/fetch-mlx mlx-community/Qwen3.6-27B-4bit main
```

This downloads the file into `/Users/bender/models/`, or the MLX repo into `/Users/bender/models/<repo>/`, then sets ownership to `bender:staff` and locks permissions to read-only.

> **Note:** `llama-cli --hf-repo` also downloads from Hugging Face but always writes to the HF cache (`~/.cache/huggingface/hub/`) regardless of any path flag — it cannot download directly to a target directory. Use `bin/fetch` or `bin/fetch-mlx` when the destination matters.

---

## 5. System Tuning

Two one-time settings before running the server. Full details and LaunchDaemons for persistence in [Tuning.md](Tuning.md).

**Wired memory limit** — raise the GPU memory allocation ceiling to 70% of unified memory, or large model allocations may be silently rejected and fall back to CPU:

```bash
sudo sysctl iogpu.wired_limit_mb=22938
```

**Flags and context** — per-model flags are pre-configured in each plist in [`daemons/`](daemons/). See [Tuning.md](Tuning.md) for the rationale behind each flag and the recommended context window per model.

---

## 6. Create LaunchDaemon

A LaunchDaemon starts automatically at boot without requiring a user login — the right choice for an appliance-style server.

The [`daemons/`](daemons/) folder in this repo contains a ready-made plist for each tested model, named after the model file. To deploy one, copy it into place and rename it:

```bash
sudo cp daemons/<model>.plist /Library/LaunchDaemons/local.llama.server.plist
```

The plist content is shown below for reference. If creating manually:

```bash
sudo nano /Library/LaunchDaemons/local.llama.server.plist
```

Paste the following for a `llama.cpp` hosted model:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
 "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>local.llama.server</string>

    <key>UserName</key>
    <string>bender</string>

    <key>ProgramArguments</key>
    <array>
      <string>/opt/homebrew/bin/llama-server</string>
      <string>-m</string>
      <string>/Users/bender/models/<model>.gguf</string>
      <string>--host</string>
      <string>0.0.0.0</string>
      <string>--port</string>
      <string>8080</string>
      <string>--ctx-size</string>
      <string>65536</string>
      <string>--batch-size</string>
      <string>512</string>
      <string>--ubatch-size</string>
      <string>512</string>
      <string>--threads</string>
      <string>8</string>
      <string>--n-gpu-layers</string>
      <string>-1</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/bender/logs/llm.out</string>

    <key>StandardErrorPath</key>
    <string>/Users/bender/logs/llm.err</string>
  </dict>
</plist>
```

---

## 7. Set Permissions

LaunchDaemons require specific ownership and permissions or launchd will refuse to load them:

```bash
sudo chown root:wheel /Library/LaunchDaemons/local.llama.server.plist
sudo chmod 644 /Library/LaunchDaemons/local.llama.server.plist
```

---

## 8. Load and Start Service

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/local.llama.server.plist
sudo launchctl kickstart system/local.llama.server
```

---

## 9. Verify

Check process:

```bash
ps aux | grep llama-server
```

Check health endpoint:

```bash
curl http://localhost:8080/health
```

Check logs:

```bash
tail -f /Users/bender/logs/llm.out
```

---

## 10. Restart After Changes

After editing the plist, fully reload it:

```bash
sudo launchctl bootout system/local.llama.server
sudo launchctl bootstrap system /Library/LaunchDaemons/local.llama.server.plist
```

To restart only the process without reloading the plist (e.g. after swapping a model file):

```bash
sudo launchctl kickstart -k system/local.llama.server
```

---

## 11. Reboot Test

Reboot the Mac Studio:

```bash
sudo reboot
```

After reboot, verify the service came up automatically:

```bash
curl http://localhost:8080/health
```

If successful, the model is running without user login — appliance behaviour confirmed.

---

## 12. Connect from Editor

Use the OpenAI-compatible endpoint:

```
http://<mac-studio>:8080/v1
```

Example config:

```json
{
  "api_base": "http://mac-studio:8080/v1",
  "api_key": "dummy"
}
```

### Zed example

In Zed's `settings.json`, under `language_models`:

```json
"openai_compatible": {
  "Freyr": {
    "api_url": "http://freyr.local:8080/v1",
    "available_models": [
      {
        "name": "gemma-4-26B-A4B-it-UD-Q4_K_M.gguf",
        "max_tokens": 32768,
        "max_output_tokens": 8192,
        "max_completion_tokens": 8192,
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

For Qwen 3.6 tool use, `mlx-openai-server` must be launched with `--reasoning-parser qwen3 --tool-call-parser qwen3_coder --enable-auto-tool-choice`; otherwise the model may emit raw `<tool_call>` text and Zed will not execute tools.

The model name must match the `id` returned by `GET /v1/models`. To enable vision, a separate mmproj file is required — see llama.cpp docs for `--mmproj`.

---

## 13. Updating

From your main account:

```bash
brew upgrade llama.cpp
```

Restart the service:

```bash
sudo launchctl kickstart -k system/local.llama.server
```

---

## 14. Switching Models

Each model in [`daemons/`](daemons/) has its own plist pre-configured with the right model path and context settings.

**1. Download the model file or MLX repo to the Mac Studio:**

Use `bin/fetch` for a single GGUF file, or `bin/fetch-mlx` for a Hugging Face MLX repo tree:

```bash
sudo bin/fetch <repo> <filename>
sudo bin/fetch-mlx <repo> [revision]
```

**2. Deploy its plist:**

The convience script `bin/deploy` provides this functionality, call it with `sudo bin/deploy <model>`:

```bash
sudo cp daemons/<model>.plist /Library/LaunchDaemons/local.llama.server.plist
sudo chown root:wheel /Library/LaunchDaemons/local.llama.server.plist
sudo chmod 644 /Library/LaunchDaemons/local.llama.server.plist
```

**3. Reload the service:**

The convience script `bin/reload` provides this functionality, call it with `sudo bin/reload`:

```bash
sudo launchctl bootout system/local.llama.server
sudo launchctl bootstrap system /Library/LaunchDaemons/local.llama.server.plist
```

**4. Verify:**

```bash
curl http://localhost:8080/health
tail -f /Users/bender/logs/llm.out
```

The old model file can be left in `/Users/bender/models/` for quick switching, or removed to free up disk space.

---

## 15. Optional Enhancements

- Run a second model on port 8081 for a different use case
- Add firewall rules to restrict access to trusted LAN hosts
- Reverse proxy with auth if exposing beyond LAN

---

## Summary

- Main account: installs and manages tools
- `bender`: runs model only
- llama.cpp: fast, native Metal inference
- launchd: reliable service management

This setup gives high performance, clean isolation, and minimal operational overhead.
