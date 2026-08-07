# Migrating the Voice Stack off Docker Desktop to Native Docker Engine in WSL2

## The problem

The voice stack (Whisper STT, Piper TTS via openedai-speech, a custom TTS router, and Jupyter for code execution) ran under Docker Desktop on the gaming PC. Docker Desktop only starts its containers on an **interactive Windows login** — not on a headless/unattended boot. After a Windows Update reboot with nobody physically at the machine, the entire voice stack stayed down until someone logged in, unlike the [image generation stack](./image-generation-setup.md), which runs as a persistent process and survives unattended reboots without issue.

This was confirmed as the root cause of an intermittent Jupyter connection-timeout bug (code interpreter failing) — the container simply wasn't running.

## The fix, at a glance

Move off Docker Desktop entirely. Run Docker Engine natively inside a WSL2 distro (no GUI wrapper), and use a background process manager to keep that distro alive across reboots without requiring a login.

This took several attempts and surfaced three separate, non-obvious bugs. Documenting all of them exactly as encountered, since each is the kind of thing that's very hard to search for if you hit it cold.

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

**Gotcha:** `newgrp docker` only applies to the current shell. New terminal windows opened before a full `wsl --shutdown` may still hit `permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`. Either run `newgrp docker` in every new shell, or do a full `wsl --shutdown` once to make the group membership stick everywhere.

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

`nvidia-ctk runtime configure` writes `/etc/docker/daemon.json`. Worth noting for later: this file becomes important again in Step 6.

## Step 4: Migrate the compose stack

Copy the voice-stack folder into WSL2's **native** filesystem (`~/voice-stack`), not `/mnt/c/...`. Docker's file I/O against the Windows-mounted path is noticeably slower than against native ext4 — worth doing right from the start rather than discovering it later.

The `docker-compose.yml` itself needed no changes — no Docker-Desktop-specific settings were in it. Standard `deploy.resources.reservations.devices` GPU syntax works fine under the `docker compose` plugin outside Swarm mode.

```bash
cd ~/voice-stack
docker compose up -d --build
```

**Gotcha:** if a port is already in use (e.g. `failed to bind host port 0.0.0.0:8888/tcp: address already in use`), check whether Docker Desktop is still running in the background holding the old container open on that port. Quit Docker Desktop entirely, then retry.

## Step 5: The LAN reachability gap

Docker Desktop ran its own networking layer that automatically exposed container ports on **all** of Windows' network interfaces — so other machines on the LAN could reach a container port via the Windows host's real IP. Plain WSL2's default NAT networking mode does **not** do this by default; it only forwards `127.0.0.1` traffic into WSL2, not LAN-facing traffic.

Fix: enable WSL2's **mirrored networking mode**, which shares the host's network stack directly with WSL2 (Windows 11 22H2+):

`%UserProfile%\.wslconfig`:
```
[wsl2]
networkingMode=mirrored
vmIdleTimeout=-1
```

(`vmIdleTimeout=-1` is unrelated to networking — see Step 6 for why it's here, and an important caveat about when it does and doesn't help.)

`wsl --shutdown`, then relaunch. Confirm with `ip addr` inside WSL2 — you should see the distro's interface showing the **same IP as the Windows host** (e.g. `eth1: inet 192.168.1.x/24`), not a separate `172.x.x.x` NAT range.

Even with mirrored mode confirmed active, LAN access still failed — the second layer: **Windows Firewall**. Docker Desktop silently created inbound allow-rules for container ports; native WSL2 doesn't. A manual inbound rule closed the gap:

```powershell
New-NetFirewallRule -DisplayName "WSL2 Voice Stack" -Direction Inbound -Protocol TCP -LocalPort 8000,8001,8002,8888 -Action Allow -Profile Any
```

(Using `-Profile Any` rather than scoping to a specific network profile avoids this breaking again if the network profile ever changes — see the same lesson in [`image-generation-setup.md`](./image-generation-setup.md) re: A1111's own firewall rule.)

## Step 6: The unattended-boot problem

This is the part that took the most iteration.

**First attempt — Task Scheduler**, trigger "At system startup," "Run whether user is logged on or not," running `wsl.exe -d Ubuntu-24.04 -- echo "started"`. The task itself reported success (`0x0`) every time. But checking container uptime after a real reboot showed something wrong: containers showed a `CREATED` timestamp from boot time, but `STATUS: Up 5 seconds` — meaning they'd been *recreated/restarted* recently, not running continuously since boot. Confirmed directly with `wsl -l -v`, which showed the distro `Stopped` well after boot despite the task reporting success.

Root cause: WSL2 has a built-in idle-timeout that shuts the whole VM down once nothing stays actively attached to it. The scheduled task's command (`echo` and exit) started the VM, ran one command, and returned — nothing kept it alive afterward, so WSL2 tore it down again shortly after boot. `vmIdleTimeout=-1` in `.wslconfig` (meant to disable this) did **not** fix it in this specific non-interactive Task Scheduler context — the distro still showed `Stopped`. (The setting is harmless to leave in place, and does matter for other things, but it was not sufficient on its own here.)

**The actual fix — NSSM**, running a *persistent* process as a real Windows service rather than a fire-and-forget scheduled task:

```powershell
C:\nssm\nssm-2.24\win64\nssm.exe install WSL2VoiceStack "C:\Windows\System32\wsl.exe" "-d Ubuntu-24.04 -- sleep infinity"
Set-Service -Name WSL2VoiceStack -StartupType Automatic
```

`sleep infinity` keeps `wsl.exe` itself running indefinitely — an actual live process attached to the VM for the full uptime of Windows, not a brief flicker at boot.

**Gotcha #1:** the service failed to start and immediately went `Paused` (NSSM's protection against a rapidly-crash-looping process). Cause: NSSM services run as `Local System` by default, and WSL2 distros are registered per-user — `Local System` doesn't have the distro registered to it, so every launch attempt failed instantly. Fixed via:

```powershell
C:\nssm\nssm-2.24\win64\nssm.exe edit WSL2VoiceStack
```
→ Log on tab → "This account" → the actual Windows user account + password → Edit service.

**Gotcha #2:** if you hit `Error creating service! CreateService(): The specified service already exists.` on a re-attempt, remove the stale registration first:
```powershell
C:\nssm\nssm-2.24\win64\nssm.exe remove WSL2VoiceStack confirm
```
then re-run the install command.

After the account fix, `Start-Service WSL2VoiceStack` succeeded, `wsl -l -v` showed the distro `Running`, and — confirmed via a genuine full restart with no login — the voice stack came back up and was reachable from the LAN with zero manual intervention.

The old Task Scheduler task was deleted once the NSSM service was confirmed working, to avoid the two ever fighting over the same VM.

Docker Desktop was uninstalled once this was fully confirmed working end-to-end.

## Step 7: Whisper's intermittent DNS failures (a second, separate surprise)

Sometime after all of the above was working, the `whisper-stt` container started failing to (re)start with:

```
error: Failed to prepare distributions
  Caused by: Failed to fetch wheel: faster-whisper-server @ file:///root/faster-whisper-server
  Caused by: Failed to install requirements from `build-system.requires` (resolve)
  Caused by: No solution found when resolving: hatchling
  Caused by: Could not connect, are you offline?
  Caused by: Request failed after 3 retries
  Caused by: error sending request for url (https://pypi.org/simple/hatchling/)
  Caused by: client error (Connect)
  Caused by: dns error: failed to lookup address information: Try again
```

This particular image does a `pip`/`uv` install step at container startup (not just at image-build time), so it needs working DNS every time it starts, not just once.

**What made this confusing:** it was genuinely intermittent. A manual `docker run --rm alpine nslookup pypi.org` would sometimes succeed cleanly (all records resolved fine), and the very next `docker restart whisper-stt` would hit the exact same DNS failure again. This pointed at WSL2 mirrored networking mode's known DNS flakiness (a real, acknowledged issue — mirrored mode changes how the network namespace and DNS resolution path work compared to default NAT mode).

**First attempted fix — `dnsTunneling`:** Microsoft added a specific `.wslconfig` setting for exactly this class of problem:

```
[wsl2]
networkingMode=mirrored
vmIdleTimeout=-1
dnsTunneling=true
```

`wsl --shutdown`, relaunch. This did **not** fully fix it — the same intermittent DNS failure recurred on a later container restart even with this setting active.

**The actual fix — override Docker's DNS directly**, bypassing WSL2's internal DNS relay (`10.255.255.254`) entirely in favor of public resolvers:

```bash
cat /etc/docker/daemon.json
```
(check existing content first — ours already had the NVIDIA runtime block from Step 3, so we needed to merge, not overwrite)

```bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    },
    "dns": ["8.8.8.8", "1.1.1.1"]
}
EOF
sudo systemctl restart docker
```

**Gotcha — the one that actually cost the most time:** after adding this and restarting Docker, `docker restart whisper-stt` *still* hit the same DNS error, repeatedly, seemingly proving the fix didn't work. It wasn't that the fix was wrong — **`docker restart` does not regenerate a container's `/etc/resolv.conf`.** Docker only writes a container's DNS config at *creation* time, not on every restart. The `whisper-stt` container had already been created (long before `daemon.json` was edited), so every "restart" kept reusing its original, stale DNS config regardless of what the daemon-level settings said.

The fix for the fix: force the container to be recreated, not just restarted, so it picks up the new daemon-level DNS settings:

```bash
cd ~/voice-stack
docker compose up -d --force-recreate whisper
```

(Note: the compose *service* name is `whisper`, even though the container itself is named `whisper-stt` via `container_name:` in the compose file — `docker compose` commands need the service name, not the container name.)

After this, `docker logs whisper-stt --tail 20` showed a clean install and `Uvicorn running on http://0.0.0.0:8000` — no further DNS failures on subsequent restarts, since the container now has the correct `resolv.conf` baked in from creation.

**Takeaway if you hit a similar "fixed the config but the container still misbehaves" situation:** several categories of container configuration (DNS being one, but this generalizes) are only applied at container *creation*, not on every restart. If a daemon-level or compose-level config change doesn't seem to take effect, try `--force-recreate` before assuming the fix itself is wrong.

## Summary of the final working setup

- Ubuntu-24.04 WSL2 distro, systemd enabled
- Docker Engine installed natively (not Docker Desktop)
- NVIDIA Container Toolkit for GPU passthrough
- `/etc/docker/daemon.json`: NVIDIA runtime config plus `"dns": ["8.8.8.8", "1.1.1.1"]` to route around WSL2's flaky internal DNS relay
- `.wslconfig`: `networkingMode=mirrored`, `vmIdleTimeout=-1`, `dnsTunneling=true`
- A Windows Firewall inbound rule for the container ports, scoped to `Any` profile
- An NSSM service (`WSL2VoiceStack`) running `wsl.exe -d Ubuntu-24.04 -- sleep infinity`, logged in as the real user account, set to auto-start
- Containers recreated (not just restarted) after any daemon-level Docker config change

No Docker Desktop, no login dependency, survives a genuine unattended reboot.
