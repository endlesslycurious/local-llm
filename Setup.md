# Mac Studio LLM Appliance Setup

This guide sets up a *dedicated* Mac Studio (`<host>`) as a local coding assistant serving local models via [oMLX](https://github.com/jundot/omlx) only. 

- **Dedicated machine:** We are aiming to have `oMLX` utilise as much of the Mac Studio as possible to increase LLM performance, this machine will be used for *nothing else*!
- **Target Hardware:** Mac Studio M2 Max, 32 GB unified memory. 
- **Workstation:** The assumption is the user will be using another mac as their workstation, it will be refered to as `MacBook` in this doc.

---

## 1. Fresh MacOS Install

This guide assumes a fresh MacOS install is being configured.

1. Reinstall MacOS:
- Unencrypted hard drive, e.g. FileVault disabled e.g. `APFS` not `APFS(Encrypted)`.
- User account `<user>` in the admin group.
- Do not sign into an Apple account, this is to avoid syncing media, photos, etc and activating their related services e.g. `photosd`.

2. Post Installation:
- Auto-login enabled for the user account: System Settings → Users & Groups → Automatic login.
- Enable screen sharing: Systems Settings → Sharing → Screen Sharing.
- Enable firewall: System Settings → Network → Firewall.

---

### 2 SSH Key Authentication

Enable remote administration without entering password to connect:

1. Enable Remote Login on the Mac Studio:
   - System Settings → General → Sharing → Remote Login → On
   - Allow access for: your account (or All Users)

2. Copy your public key from the MacBook:
   ```bash
   ssh-copy-id <user>@<host>.local
   ```
   If you don't have a key yet, generate one first with `ssh-keygen -t ed25519`.

3. Disable password authentication (optional, hardens the appliance):
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   Set:
   ```
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   ```
   Then restart sshd:
   ```bash
   sudo launchctl kickstart -k system/com.openssh.sshd
   ```

4. Verify from the MacBook:
   ```bash
   ssh <user>@<host>.local
   ```
   Should connect without prompting for a password.

---

## 3. Install Runtime & Dependencies

1. HomeBrew - package manager.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

2. oMLX - LLM server.

Install the oMLX LLM server next:

```bash
brew tap jundot/omlx https://github.com/jundot/omlx
brew trust jundot/omlx
brew install omlx
```

3. Hugging Face CLI - Model downloader.

```bash
brew install hf
```

4. btop - Resource monitor.

Highly recommend [btop](https://github.com/aristocratos/btop) for monitoring the system over ssh.

```bash
brew install btop
```

---

## 4. Directory Structure

Create these working directories for oMLX to hold the models, disk cache and logs:

```bash
mkdir /Users/<user>/models
mkdir /Users/<user>/cache
mkdir /Users/<user>/logs
```

Then create the settings dir:

```bash
mkdir ~/.omlx
```

---

## 5. Settings files linked

We will keep the settings files in this repo for change tracking and link them into the `.omlx` directory for `oMLX` to use.

```bash
ln -sf ./config/settings.json ~/.omlx/settings.json
ln -sf ./config/model_settings.json ~/.omlx/model_settings.json
```

### Note:
- These settings files contain my oMLX config and model configurations, this is why there isn't more oMLX setting steps!
- There are API & secret keys defined in `settings.json` but as `skip_api_key_verification` is enabled they're not used and therefore not a concern. If you were to enable the API key you should rotate the key in the admin panel!
- The settings file do reference my current username and machine name, if using you'd need to update these!

---

## 6. Configure Firewall

Enable `oMLX` to accept incoming connections throught the firewall:

```bash
security set-appfirewall-rule -t incoming --app-firewall allow --process "omlx-server"
```

Verify firewall settings:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --listapps
```

---

## 7. Start oMLX Service

`oMLX` when installed via `brew` can use `brew service` to manage app lifespan, see [details](https://github.com/jundot/omlx#homebrew-service).

```bash
brew services start omlx
```

> Note: `oMLX` will use the settings files we linked earlier for its configuration.

---

## 8. Verify

Check process exists:

```bash
ps aux | grep omlx-server
```

Check health endpoint:

```bash
curl http://localhost:8080/health
```

Check logs:

```bash
tail -f /Users/<user>/logs/
```

---

## 9. Reboot Test

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

## 10. Connect from Editor

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
  "<host>": {
    "api_url": "http://<host>.local:8080/v1",
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
---

## Done!

In theory everything should be working and you can admistrate `oMLX` via its web admin e.g. `http://<host>.local:8080/admin/dashboard?tab=status`.
- See [Models](./Models.md) for the list of models I tried and how to download them.
- See [Tuning](./Tuning.md) for model settings and how to increase the GPU RAM limit on MacOS.
-
