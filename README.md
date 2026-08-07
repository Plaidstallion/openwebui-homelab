# Homelab: Open-WebUI + Self-Hosted LLM Stack

A self-hosted setup built around [Open-WebUI](https://github.com/open-webui/open-webui), running across two machines:

- **A Debian server** — Traefik reverse proxy, Open-WebUI (uses its own built-in username/password login), Google OAuth gating for other exposed services like SearXNG, Watchtower for auto-updates, Cloudflare DDNS, and Duplicati-based backup with off-site (Google Drive) replication
- **A Windows gaming PC** (RTX 3090) — handles the GPU-heavy pieces: local voice (Whisper STT + Piper TTS via a custom router) and local image generation (Automatic1111 + a companion image server), both exposed back to Open-WebUI

This repo shares the setup, config, and custom code — not because it's finished or perfect, but so anyone doing something similar can skip a few of the dead ends already hit here. Feedback and questions welcome.

> Note: `docker-compose.yml` here is trimmed to just the services relevant to this stack. The actual server also runs a broader set of unrelated self-hosted apps (media server, download tools, a git host, etc.) that aren't part of this writeup.

## What's in this repo

```
docker-compose.yml          # Main server stack (Traefik, Open-WebUI, SearXNG, OAuth, etc.)
.env.example                 # Template for the main stack's secrets
traefik/
  configurations/              # Traefik dynamic file-provider config (routers, middleware chains)
voice-stack/
  docker-compose.yml          # Gaming PC's Docker services (Whisper, TTS, Jupyter)
  .env.example                 # Template for the voice stack's secrets
open-webui/
  functions/                  # Exported custom Filters/Functions
  tools/                       # Exported custom Tools (Native Function Calling)
  models/                       # Exported custom Model presets
docs/
  backup-and-restore.md        # Full Duplicati backup/restore setup, including things it can't cover
  image-generation-setup.md    # Automatic1111 + image server setup (NSSM Windows services)
  knowledge-rag-setup.md       # Knowledge/RAG setup and a disambiguation bug + fix
```

A couple of the `traefik/configurations/` files (`app-plex.yml.bak`, `app-embyl.yml.bak`) are disabled/backup copies, included as-is for reference. Note `app-embyl.yml.bak` has a router/service naming collision with `app-pihole.yml` (both use `-pihole-` in their resource names) — worth renaming before actually using it, this wasn't cleaned up in the original setup.

## Getting started

1. **Copy the env templates and fill in your own values:**
   ```
   cp .env.example .env
   cp voice-stack/.env.example voice-stack/.env
   ```
   See the comments in each file for what every variable is for. A few (Cloudflare, Google OAuth) require setting things up on those platforms first — this repo assumes you already have accounts/API access there.

2. **Bring up the main stack:**
   ```
   docker compose up -d
   ```

3. **Bring up the voice stack** (on the machine with your GPU):
   ```
   cd voice-stack
   docker compose up -d
   ```

4. **Set up image generation** — this one isn't Docker; see [`docs/image-generation-setup.md`](docs/image-generation-setup.md) for the full Automatic1111 + Windows service setup.

5. **Import the Open-WebUI Tools/Functions/Models** from `open-webui/` via the admin panel (Workspace → Tools / Functions / Models → Import).

   > **After importing:** custom Tools default to private to whoever imports them. If a Tool works for you but not for other users on your instance, open the Tool (Workspace → Tools → **Edit**, not the `⋯` menu's "Share" option — that publishes to Open-WebUI's public community hub instead) and look for an access-control setting near the top of the editor. Set it to Public, or grant read access to the appropriate users/group. This is easy to miss and can cause the model to silently fall back to unrelated built-in capabilities (like a native code interpreter) instead of erroring, which is a confusing failure mode to debug.

6. **Set up Knowledge/RAG** (optional) — see [`docs/knowledge-rag-setup.md`](docs/knowledge-rag-setup.md) for the embedding config and a real disambiguation bug this setup hit (with the fix).

7. **Set up backups** — see [`docs/backup-and-restore.md`](docs/backup-and-restore.md). Don't skip this, and don't skip actually testing a restore once it's running.

## A few things worth knowing before you dig in

- **Two machines, not one.** If something in the main compose file references the gaming PC's IP (Ollama, the image server, etc.), you'll need to update that to match your own network.
- **Tracks Open-WebUI's `:main` tag**, kept current via Watchtower — not a pinned release. Tested against v0.11.0 at the time of writing, but expect drift over time.
- **One known unresolved bug**, documented honestly in `docs/image-generation-setup.md`: generated images don't render inline in the model's final chat response due to what looks like an upstream Open-WebUI/Ollama tool-call-parsing interaction. The image is always correctly generated and retrievable, just not auto-rendered. Living with it pending an upstream fix.

## Not included

- The actual Open-WebUI database, vector embeddings, and Knowledge-collection documents — these are data, not config, and don't belong in git. See `docs/backup-and-restore.md` for how that's handled instead (Duplicati, not git).
- Any of the media library itself.
