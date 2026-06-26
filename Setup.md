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

## 3. Power / Sleep Settings

Prevent the system from sleeping as its a server, from the terminal run:

```bash
sudo pmset -a sleep 0 disksleep 0 womp 1 powernap 1
```

Then verify with:

```bash
pmset -g
```

---

## 4. Install Runtime & Dependencies

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

> Note: If your using models that use structured generation like Qwen3 then its recommended to install `oMLX` with grammer using `brew install omlx --with-grammar` to get [xgrammar](https://github.com/mlc-ai/xgrammar) installed!

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

## 5. Directory Structure

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

## 6. Settings files linked

> There is currently an issue wiht oMLX replacing `model_settings.json` and not respecting the symlink - [github](https://github.com/jundot/omlx/issues/1958).

We will keep the settings files in this repo for change tracking and link them into the `.omlx` directory for `oMLX` to use.

```bash
ln -f ./config/settings.json ~/.omlx/settings.json
ln -f ./config/model_settings.json ~/.omlx/model_settings.json
```

Verify by comparing inode numbers (first column) which should be identical on linked files:
- `467193` for `model_settings.json` 
- `467194` for `settings.json`

```bash
ls -li ~/.omlx ./config

./config:
total 24
467193 -rw-r--r--  2 bender  staff  4883 Jun 21 15:58 model_settings.json
467194 -rw-r--r--@ 2 bender  staff  2518 Jun 21 15:57 settings.json

/Users/bender/.omlx:
total 32
467193 -rw-r--r--  2 bender  staff  4883 Jun 21 15:58 model_settings.json
467194 -rw-r--r--@ 2 bender  staff  2518 Jun 21 15:57 settings.json
511639 -rw-r--r--  1 bender  staff   201 Jun 20 19:13 stats.json
```

### Note:
- These settings files contain my oMLX config and model configurations, this is why there isn't more oMLX setting steps!
- There are API & secret keys defined in `settings.json` with dummy values, but as `skip_api_key_verification` is enabled they're not used and therefore not a concern. If you were to enable the API key you should rotate the key in the admin panel!
- The settings file do reference my current username and machine name, if using you'd need to update these!

---

## 7. Configure Firewall

> Note: `oMLX` is actually a python process so the following isn't the correct path, skip this step for now!

Enable `oMLX` to accept incoming connections throught the firewall:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /opt/homebrew/bin/omlx
```

Verify firewall settings:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --list
```

---

## 8. Start oMLX Service

`oMLX` when installed via `brew` can use `brew service` to manage app lifespan, see [details](https://github.com/jundot/omlx#homebrew-service).

```bash
brew services start omlx
```

> Note: `oMLX` will use the settings files we linked earlier for its configuration.

---

## 9. Verify

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

## 10. Reboot Test

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

## 11. Connect from Editor

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
- See [Tuning](./Tuning.md) for model settings and how to increase the GPU RAM limit on MacOS (highly recommended).
-
