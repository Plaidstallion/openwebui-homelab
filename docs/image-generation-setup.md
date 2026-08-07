# Image Generation Setup (Automatic1111 + Open-WebUI)

This piece runs on a separate Windows machine (a gaming PC with an RTX 3090), not the main Debian server, since Stable Diffusion needs the GPU. It's not part of the Docker Compose stack — Automatic1111 and its companion image server run as persistent native Windows services.

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

## 3. Running both as real services (NSSM)

Both were originally run as manual PowerShell windows during development, then converted to persistent Windows services using [NSSM](https://nssm.cc/) so they survive reboots and don't need a console window kept open.

Two services:
- `AutomaticA1111`
- `ImageServer`

**`ImageServer`:**
```
nssm.exe install ImageServer "C:\Users\<you>\AppData\Local\Programs\Python\Python310\python.exe" "C:\StableDiffusion\image_server\serve_images.py"
nssm.exe set ImageServer AppDirectory "C:\StableDiffusion\image_server"
nssm.exe set ImageServer AppStdout "C:\StableDiffusion\image_server\service.log"
nssm.exe set ImageServer AppStderr "C:\StableDiffusion\image_server\service.log"
nssm.exe set ImageServer Start SERVICE_AUTO_START
```

**`AutomaticA1111`:**
```
nssm.exe install AutomaticA1111 "C:\StableDiffusion\stable-diffusion-webui\webui-user.bat"
nssm.exe set AutomaticA1111 AppDirectory "C:\StableDiffusion\stable-diffusion-webui"
nssm.exe set AutomaticA1111 AppStdout "C:\StableDiffusion\stable-diffusion-webui\service.log"
nssm.exe set AutomaticA1111 AppStderr "C:\StableDiffusion\stable-diffusion-webui\service.log"
nssm.exe set AutomaticA1111 Start SERVICE_AUTO_START
```

Both are set to `SERVICE_AUTO_START` with stdout/stderr redirected to a `service.log` file in their respective folders, since there's no interactive console to watch once they're running as services. Verify with `Get-Service` — both should show `Running`.

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
