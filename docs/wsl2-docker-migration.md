# Migrating the Voice Stack off Docker Desktop to Native Docker Engine in WSL2

## The problem

The voice stack (Whisper STT, Piper TTS via openedai-speech, a custom TTS router, and Jupyter for code execution) ran under Docker Desktop on the gaming PC. Docker Desktop only starts its containers on an **interactive Windows login** — not on a headless/unattended boot. After a Windows Update reboot with nobody physically at the machine, the entire voice stack stayed down until someone logged in, unlike the [image generation stack](./image-generation-setup.md), which runs as real NSSM Windows services and survives unattended reboots without issue.

This was confirmed as the root cause of an intermittent Jupyter connection-timeout bug (code interpreter failing) — the container simply wasn't running.

## The fix, at a glance

Move off Docker Desktop entirely. Run Docker Engine natively inside a WSL2 distro (no GUI wrapper), and use NSSM — the same tool already proven for the image generation stack — to keep that distro alive across reboots without requiring a login.

This took more attempts than expected. Documenting the dead ends along with the fix, since they're the non-obvious part.

## Step 1: A real WSL2 distro with systemd

Docker Desktop manages its own internal `docker-desktop` utility distro — it's not meant to be used directly. A genuine Ubuntu distro was installed separately:

```powershell
wsl --install -d Ubuntu-24.04
```

`wsl --install` sets up `/etc/wsl.conf` with systemd enabled by default now:

```
[boot]
systemd=true
[user]
default=<your-username>
```

If it's not already there, add it, then apply with `wsl --shutdown` followed by relaunching the distro. Confirm with `systemctl status` — you should see a real PID 1 `systemd` process tree, not an error.

## Step 2: Docker Engine, installed directly (no Docker Desktop)

Standard Docker apt-repo install inside the WSL2 distro:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add yourself to the `docker` group and enable the service:

```bash
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable docker
```

The installer creates the systemd symlinks automatically — `docker.service` starts with the distro.

## Step 3: GPU passthrough (NVIDIA Container Toolkit)

WSL2 already bridges the NVIDIA driver from Windows; native Docker Engine just needs the container toolkit to use it:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Test:

```bash
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi
```

Should show your GPU inside the container, same as running `nvidia-smi` natively on Windows.

## Step 4: Migrate the compose stack

Copy the voice-stack folder into WSL2's **native** filesystem (`~/voice-stack`), not `/mnt/c/...`. Docker's file I/O against the Windows-mounted path is noticeably slower than against native ext4 — worth doing right from the start rather than discovering it later.

The `docker-compose.yml` itself needed no changes — no Docker-Desktop-specific settings were in it. Standard `deploy.resources.reservations.devices` GPU syntax works fine under the `docker compose` plugin outside Swarm mode.

```bash
cd ~/voice-stack
docker compose up -d --build
```

## Step 5: The LAN reachability gap (the first real surprise)

Docker Desktop ran its own networking layer that automatically exposed container ports on **all** of Windows' network interfaces — so other machines on the LAN could reach a container port via the Windows host's real IP. Plain WSL2's default NAT networking mode does **not** do this by default; it only forwards `127.0.0.1` traffic into WSL2, not LAN-facing traffic.

Fix: enable WSL2's **mirrored networking mode**, which shares the host's network stack directly with WSL2 (Windows 11 22H2+):

`%UserProfile%\.wslconfig`:
```
[wsl2]
networkingMode=mirrored
```

`wsl --shutdown`, then relaunch. Confirm with `ip addr` inside WSL2 — you should see the distro's interface showing the **same IP as the Windows host**, not a separate `172.x.x.x` NAT range.

Even with mirrored mode confirmed active, LAN access still failed — the second layer: **Windows Firewall**. Docker Desktop silently created inbound allow-rules for container ports; native WSL2 doesn't. A manual inbound rule (Windows Defender Firewall with Advanced Security → Inbound Rules → New Rule → Port → TCP → the specific container ports → Allow → Private profile) closed the gap.

## Step 6: The unattended-boot problem (the real dead end)

This is the part that took the most iteration.

**First attempt — Task Scheduler**, trigger "At system startup," "Run whether user is logged on or not," running `wsl.exe -d Ubuntu-24.04 -- echo "started"`. The task itself reported success (`0x0`) every time. But checking container uptime after a real reboot showed something wrong: containers showed a `CREATED` timestamp from boot time, but `STATUS: Up 5 seconds` — meaning they'd been *recreated/restarted* recently, not running continuously since boot.

Root cause: WSL2 has a built-in idle-timeout that shuts the whole VM down once nothing stays actively attached to it. The scheduled task's command (`echo` and exit) started the VM, ran one command, and returned — nothing kept it alive afterward, so WSL2 tore it down again shortly after boot. `vmIdleTimeout=-1` in `.wslconfig` (meant to disable this) did **not** fix it in this specific non-interactive Task Scheduler context — `wsl -l -v` still showed the distro `Stopped` well after boot.

**The actual fix — NSSM**, the same tool already used for the image generation services. Instead of a fire-and-forget scheduled task, run a *persistent* process as a real Windows service:

```powershell
C:\nssm\nssm-2.24\win64\nssm.exe install WSL2VoiceStack "C:\Windows\System32\wsl.exe" "-d Ubuntu-24.04 -- sleep infinity"
Set-Service -Name WSL2VoiceStack -StartupType Automatic
```

`sleep infinity` keeps `wsl.exe` itself running indefinitely — an actual live process attached to the VM for the full uptime of Windows, not a brief flicker at boot.

One more gotcha here: the service failed to start and immediately went `Paused` (NSSM's protection against a rapidly-crash-looping process). Cause: NSSM services run as `Local System` by default, and WSL2 distros are registered per-user — `Local System` doesn't have the distro registered to it, so every launch attempt failed instantly. Fixed via `nssm edit WSL2VoiceStack` → Log on tab → "This account" → the actual Windows user account.

After that fix, `Start-Service WSL2VoiceStack` succeeded, `wsl -l -v` showed the distro `Running`, and — confirmed via a genuine full restart with no login — the voice stack came back up and was reachable from the LAN with zero manual intervention.

Docker Desktop was uninstalled once this was fully confirmed working end-to-end.

## Summary of the final working setup

- Ubuntu-24.04 WSL2 distro, systemd enabled
- Docker Engine installed natively (not Docker Desktop)
- NVIDIA Container Toolkit for GPU passthrough
- `.wslconfig`: `networkingMode=mirrored`
- A Windows Firewall inbound rule for the container ports
- An NSSM service (`WSL2VoiceStack`) running `wsl.exe -d Ubuntu-24.04 -- sleep infinity`, logged in as the real user account, set to auto-start

No Docker Desktop, no login dependency, survives a genuine unattended reboot.
