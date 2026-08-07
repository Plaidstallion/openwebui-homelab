# Image Generation Setup (Automatic1111 + Open-WebUI)

This piece runs on a separate Windows machine (a gaming PC with an RTX 3090), not the main Debian server, since Stable Diffusion needs the GPU. It's not part of the Docker Compose stack — Automatic1111 and its companion image server run as persistent native Windows processes.

## Stack

- **Automatic1111** (Stable Diffusion WebUI) — image generation backend
- **A standalone Flask image server** — hosts generated images and returns short URLs, so Open-WebUI's tool-call payload stays small
- **A custom Open-WebUI Tool** (`Image_Tool.py`) — calls Automatic1111's API and returns the image link to the model

## 1. Installing Automatic1111

Installed to `C:\StableDiffusion\stable-diffusion-webui`, using SDXL as the base checkpoint (chosen over ComfyUI for the first pass — simpler to get running).

A few install-time issues you may hit, and how they were resolved here:

- **Python version conflict** — the installer picked up Python 3.14 instead of 3.10, which Automatic1111 needs. Fixed by pinning the `PYTHON=` path in `webui-user.bat` directly to a 3.10 install.
- **CLIP failing to build** — a pip build-isolation error. Fixed with `setuptools<81` plus a manual `--no-build-isolation` install of the CLIP package.
- **Upstream `Stability-AI/stablediffusion` repo unavailable** — that repo was removed/made private. Worked around by pointing at the community fork `w-e-w/stablediffusion` instead.

### Launch config

In `webui-user.bat`:
```
set COMMANDLINE_ARGS=--medvram-sdxl --xformers --no-half-vae --api --listen --api-auth
```

- `--api` — required so Open-WebUI's Tool can call it
- `--listen` — exposes it on the LAN, not just localhost
- `--api-auth` — LAN-exposed with basic auth enforced (don't skip this — it's the only thing stopping anyone else on your network from generating images or hitting the API)
- `--medvram-sdxl` — needed to fit SDXL comfortably alongside an LLM also resident in VRAM

At 1024x1024 (SDXL's native resolution — don't use the old SD1.5-era 512 default), generation takes roughly 6-7 seconds alone, ~14 seconds with a 31B LLM also loaded on the same GPU. No OOM issues, no need for a model load/unload strategy — both fit.

## 2. The image server (why it exists)

The first version of the Tool tried embedding generated images as raw base64 directly in the tool's return payload. This blew the model's context window (hundreds of thousands of characters) and caused ~1 minute delays with a broken/hallucinated final response.

Fix: a small standalone Flask server (`serve_images.py`, port 8001) that Automatic1111's output gets saved to, which then hands back a short URL instead of the raw image data. The Tool's payload is now just a link, not the image itself.

## 3. Running as persistent services

Both were originally run as manual PowerShell windows during development, then converted to persistent processes so they survive reboots and don't need a console window kept open. Along the way, the two ended up on **different mechanisms** — worth knowing why.

### `ImageServer` — NSSM (Windows Service)

```
nssm.exe install ImageServer "C:\Users\<you>\AppData\Local\Programs\Python\Python310\python.exe" "C:\StableDiffusion\image_server\serve_images.py"
nssm.exe set ImageServer AppDirectory "C:\StableDiffusion\image_server"
nssm.exe set ImageServer AppStdout "C:\StableDiffusion\image_server\service.log"
nssm.exe set ImageServer AppStderr "C:\StableDiffusion\image_server\service.log"
nssm.exe set ImageServer Start SERVICE_AUTO_START
```

`ImageServer` is a plain Python/Flask process with no GPU dependency — NSSM works fine for it. `SERVICE_AUTO_START`, stdout/stderr redirected to a `service.log` file. Verify with `Get-Service ImageServer` — should show `Running`.

### `AutomaticA1111` — Task Scheduler, **not** NSSM

This one has a real story behind it, worth documenting since the failure mode was genuinely confusing.

**Original setup** (worked fine for months):
```
nssm.exe install AutomaticA1111 "C:\StableDiffusion\stable-diffusion-webui\webui-user.bat"
nssm.exe set AutomaticA1111 AppDirectory "C:\StableDiffusion\stable-diffusion-webui"
nssm.exe set AutomaticA1111 AppStdout "C:\StableDiffusion\stable-diffusion-webui\service.log"
nssm.exe set AutomaticA1111 AppStderr "C:\StableDiffusion\stable-diffusion-webui\service.log"
nssm.exe set AutomaticA1111 Start SERVICE_AUTO_START
```

**What broke it:** after installing the NVIDIA Container Toolkit and migrating the voice stack from Docker Desktop to native Docker Engine in WSL2 (see [`wsl2-docker-migration.md`](wsl2-docker-migration.md)), `AutomaticA1111` started hanging indefinitely at `Creating model from config` on every startup — no crash, no error, just stuck. This happened consistently, including across a full Windows reboot.

**What it wasn't** (all individually tested and ruled out):
- `xformers` — removed it entirely, same hang
- VRAM contention — freed the LLM's ~20GB first, still hung, and CPU time on the process had genuinely flatlined, not just slowed
- Windows Defender scanning the checkpoint file — added an exclusion, no change
- GPU/CUDA/driver state — a standalone `python -c "import torch; torch.zeros(10).cuda()"` in the same venv worked instantly, ruling out driver corruption
- Git ownership issues (a real, separate bug hit along the way — see note below) — fixed with `git config --system --add safe.directory "*"`, got further, but still hung at the same later step
- The NSSM "log on as" account — tried both `LocalSystem` (original) and a real user account, same result either way
- `SERVICE_INTERACTIVE_PROCESS` flag — no effect (this flag has been unreliable on modern Windows since Vista/7; true services still get Session 0 isolation regardless)

**The actual cause:** running `webui.bat` directly in an interactive PowerShell window worked instantly every single time (5.2 second startup) — versus hanging indefinitely as any kind of Windows Service, under any account, with any flag. This isolates the problem to **Session 0 isolation** itself — the separate, non-interactive session all true Windows Services run in, regardless of which account they're configured to run as. Something in Automatic1111/PyTorch's GPU initialization at that step needs a real desktop session context that Session 0 doesn't provide, and this specific hang only started manifesting after enough GPU-driver-adjacent churn (the WSL2/NVIDIA Container Toolkit work) disturbed whatever had been working by coincidence before.

**The fix:** replace the NSSM service with a Task Scheduler task instead. Unlike a true Windows Service, a Scheduled Task gets a genuine session context:

- **General tab:** Name it (e.g. `Start Automatic1111`). Security options: **"Run whether user is logged on or not"** + **"Run with highest privileges"**
- **Triggers tab:** New → **"At startup"**
- **Actions tab:** New → Start a program:
  - Program/script: `C:\StableDiffusion\stable-diffusion-webui\webui-user.bat` (not `webui.bat` directly — `webui-user.bat` is what actually applies `COMMANDLINE_ARGS`, including `--listen` and `--api-auth`)
  - **Start in:** `C:\StableDiffusion\stable-diffusion-webui`

This works immediately, and has been confirmed to survive a genuine unattended reboot with no login required.

The old `AutomaticA1111` NSSM service was left in place but disabled (`Set-Service -Name AutomaticA1111 -StartupType Disabled`) rather than deleted, in case it's ever useful for comparison.

**Side note — the git ownership bug:** separately from the Session 0 issue above, a corrupted `webui-user.bat` (a stray leftover line from an earlier edit) combined with a genuine git "dubious ownership" error (a security check that trips when a repo's file owner doesn't match the account running git — happens whenever a service runs as a different account than whoever originally cloned the repo) caused an unrelated crash earlier in this same debugging session. If you ever see `fatal: detected dubious ownership in repository`, the fix is:
```
git config --system --add safe.directory "*"
```
(machine-wide, so it applies regardless of which account runs git — appropriate for a single-user home server, not a shared machine)

## 4. Exposing the image server externally (Traefik)

The image server isn't a Docker container, so it's added to Traefik via the file provider rather than Docker service-discovery labels. A router was added for `images.yourdomain.com`, restricted to `PathPrefix /images` only.

The `/save` endpoint (used internally by Automatic1111/the Tool to upload a freshly generated image) is deliberately **never** routed externally — it stays reachable only from the LAN, with no additional auth layer needed since it's structurally unreachable from outside.

Already covered by an existing wildcard cert and wildcard DNS entry for `*.yourdomain.com`, so no new DNS or certificate work was required — just the new Traefik router.

Verify: images should load correctly from outside the LAN via `https://images.yourdomain.com/images/...`, and hitting `/save` from outside should 404.

## 5. The Open-WebUI Tool config

`Image_Tool.py` has two separate Valves rather than one combined URL setting:

- **`IMAGE_UPLOAD_URL`** — the internal LAN address, used for the `/save` POST (must stay internal, since `/save` isn't routed externally)
- **`IMAGE_DISPLAY_URL`** — set to `https://images.yourdomain.com`, used to build the link shown to the user

These need to be split, not combined — pointing a single URL at the external domain breaks uploads, since `/save` intentionally isn't reachable that way.

## Known limitation (unresolved, living with it)

Images generate and upload correctly every time, and the correct link is always retrievable by expanding the tool-result panel in Open-WebUI. However, the model's **final visible chat response** is currently malformed — instead of relaying the image link normally, it outputs a fake second tool-call attempt as raw text.

This is 100% reproducible, not a sampling fluke. Root cause traced to Ollama's native tool-call parser not correctly handling a second, spontaneous tool-call-shaped output from the model after it sees the image result — this appears to be an upstream Open-WebUI/Ollama interaction bug affecting responses using the newer structured-output format, not something fixable from the Tool or system prompt side.

Things that were tried and ruled out:
- System prompt instructions — no effect
- An outlet Filter to clean up the malformed text after the fact — dead end; outlet() Filter modifications are silently discarded before the final response is persisted for structured-output responses (a currently open Open-WebUI bug)

Things considered and explicitly **not** done, as not worth the trade-off:
- Pinning to an older Open-WebUI version — the bug appeared sometime after v0.9.6, and downgrading that far risks real regressions elsewhere
- Switching this model's function-calling mode from native to legacy — native mode was a deliberate choice for reliable multi-tool-call behavior on other tools (weather, search); reverting it to fix this one issue risks breaking those

**Current accepted state:** the image is always there and correctly linked — you just have to expand the tool-result panel to see it, since it won't render inline in the final message. Living with this pending an upstream fix.
