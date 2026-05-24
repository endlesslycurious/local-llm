# Mac Studio LLM Appliance Setup (Gemma + llama.cpp)

> **Hardware:** Mac Studio M2 Max, 32 GB unified memory. See [Models.md](Models.md) for what has been tested, or the [insiderllm guide](https://insiderllm.com/guides/best-local-llms-mac-2026/) for current recommendations.

This guide sets up a dedicated Mac Studio as a local coding assistant using Gemma models via llama.cpp, running under a separate service account (`bender`).

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

Install llama.cpp via Homebrew:

```bash
brew install llama.cpp
```

Verify install:

```bash
which llama-server
# /opt/homebrew/bin/llama-server
```

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

From your main account:

1. Download a GGUF model (see [Models.md](Models.md) for tested options)
2. Copy to the service account:

```bash
sudo cp <model>.gguf /Users/bender/models/
sudo chown -R bender:staff /Users/bender/models
```

Optional hardening (prevents `bender` from modifying model files):

```bash
sudo chmod -R 555 /Users/bender/models
```

---

## 5. Configure Context Window (Mac Studio M2 Max, 32 GB)

For a 32 GB Mac Studio M2 Max running a 26B Q4 model, a practical balance is:

```text
Context Window: 8192–12288
Batch Size: 512
GPU Layers: all (-1)
```

Recommended starting point:

```bash
--ctx-size 8192
--batch-size 512
--ubatch-size 512
--n-gpu-layers -1
```

Notes:
- 8192 provides a good balance of responsiveness and memory usage
- 12288 is possible if memory pressure is acceptable
- 16384+ will usually start hurting responsiveness on 32 GB systems
- Larger context windows increase KV cache memory significantly
- `--threads` controls CPU threads for non-GPU work; with all layers on GPU this rarely bottlenecks, but 8–10 is a reasonable value on M2 Max (12 performance cores)

For coding assistant usage:
- Use 8192 for interactive/editor use
- Use a larger context only for deep architecture review or large refactors

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

Paste:

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
      <string>/Users/bender/models/gemma-4-26B-A4B-it-UD-Q4_K_M.gguf</string>
      <string>--host</string>
      <string>0.0.0.0</string>
      <string>--port</string>
      <string>8080</string>
      <string>--ctx-size</string>
      <string>8192</string>
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

**1. Copy the new model file onto the Mac Studio:**

```bash
sudo cp <model>.gguf /Users/bender/models/
sudo chown bender:staff /Users/bender/models/<model>.gguf
```

**2. Deploy its plist:**

```bash
sudo cp daemons/<model>.plist /Library/LaunchDaemons/local.llama.server.plist
sudo chown root:wheel /Library/LaunchDaemons/local.llama.server.plist
sudo chmod 644 /Library/LaunchDaemons/local.llama.server.plist
```

**3. Reload the service:**

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
