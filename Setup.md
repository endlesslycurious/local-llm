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

Switch to the `bender` user and create directories:

```bash
su - bender
mkdir -p ~/models ~/logs ~/run
```

Final layout:

```
/Users/bender/
  models/
  logs/
  run/
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

Optional hardening:

```bash
chmod -R 555 /Users/bender/models
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

For coding assistant usage:
- Use 8192 for interactive/editor use
- Use a larger model or larger context only when doing deeper architecture or large refactors

---

## 6. Create launchd Service

As `bender`:

```bash
mkdir -p ~/Library/LaunchAgents
nano ~/Library/LaunchAgents/local.llama.server.plist
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

    <key>ProgramArguments</key>
    <array>
      <string>/opt/homebrew/bin/llama-server</string>
      <string>-m</string>
      <string>/Users/bender/models/gemma-4b-q4.gguf</string>
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

## 7. Start Service

As `bender`:

```bash
launchctl load ~/Library/LaunchAgents/local.llama.server.plist
launchctl start local.llama.server
```

Check logs:

```bash
tail -f ~/logs/llm.out
```

---

## 8. LaunchDaemon Setup (Recommended)

For appliance-style behaviour, use a LaunchDaemon instead of a LaunchAgent.

Benefits:
- Starts automatically at boot
- No desktop login required
- Runs fully in background
- Behaves like a proper server/service

Startup flow:

```text
boot
→ launchd
→ llama-server starts automatically
```

---

### 8.1 Remove Previous LaunchAgent (if present)

As `bender`:

```bash
launchctl unload ~/Library/LaunchAgents/local.llama.server.plist
rm ~/Library/LaunchAgents/local.llama.server.plist
```

---

### 8.2 Create LaunchDaemon

As your main admin account:

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

### 8.3 Set Correct Ownership and Permissions

LaunchDaemons are strict about permissions.

Set ownership:

```bash
sudo chown root:wheel /Library/LaunchDaemons/local.llama.server.plist
```

Set permissions:

```bash
sudo chmod 644 /Library/LaunchDaemons/local.llama.server.plist
```

---

### 8.4 Load Service

Load the daemon:

```bash
sudo launchctl load /Library/LaunchDaemons/local.llama.server.plist
```

Start immediately:

```bash
sudo launchctl start local.llama.server
```

---

### 8.5 Verify

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

### 8.6 Restart After Changes

After editing the plist:

```bash
sudo launchctl unload /Library/LaunchDaemons/local.llama.server.plist
sudo launchctl load /Library/LaunchDaemons/local.llama.server.plist
```

Or restart only the process:

```bash
sudo launchctl kickstart -k system/local.llama.server
```

---

### 8.7 Reboot Test

Reboot the Mac Studio:

```bash
sudo reboot
```

After reboot:

```bash
curl http://localhost:8080/health
```

If successful, the model is now behaving like a proper appliance-style service.

---

## 9. Connect from Editor

Use OpenAI-compatible endpoint:

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

## 10. Updating

From your main account:

```bash
brew upgrade llama.cpp
```

Restart service:

```bash
sudo launchctl kickstart -k system/local.llama.server
```

---

## 11. Optional Enhancements

- Run second model on port 8081 (e.g. 26B)
- Add firewall rules for LAN restriction
- Reverse proxy with auth if exposing beyond LAN

---

## 12. Verify Auto-start

Reboot the Mac Studio and verify the service is running:

```bash
curl http://localhost:8080/health
```

Or:

```bash
ps aux | grep llama-server
```

Check logs:

```bash
tail -f /Users/bender/logs/llm.out
```

If using a LaunchDaemon, the service should start automatically at boot without user login.

---

## Summary

- Main account: installs and manages tools
- `bender`: runs model only
- llama.cpp: fast, native Metal inference
- launchd: reliable service management

This setup gives high performance, clean isolation, and minimal operational overhead.
