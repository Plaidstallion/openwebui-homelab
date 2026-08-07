# Backup & Restore

This covers full disaster recovery for the Debian server — everything needed to rebuild it from scratch if the OS drive dies, short of the actual bulk media library (replaceable, not configuration).

## Why Duplicati, not custom scripts

Duplicati was already installed in the stack but not actually running. Rather than build custom cron + tar/rsync scripts, it was configured properly instead — purpose-built for this, less to maintain, and its native Google Drive backend with built-in AES-256 encryption covers off-site backup without needing a separate rclone-crypt setup.

## What's backed up, and where

Two Duplicati jobs, each with two destinations (a local SnapRAID/MergerFS pool, and Google Drive):

> Note: this backup covers the full `~/docker` directory on the real server, which includes other self-hosted services beyond what's in this repo's trimmed `docker-compose.yml` (that file only includes what's relevant to the Open-WebUI stack this writeup is about). The backup strategy itself is general-purpose and works the same regardless of which services live in your own compose file.

### 1. Docker Data
Source: the whole `~/docker` directory, minus:
- One-off manual backup snapshots (`*.bak` folders) that shouldn't be backed up themselves
- Fully unused/decommissioned service data directories
- Disposable cache directories (see note below)

> **Note on cache directories:** the first backup run threw non-fatal `UnauthorizedAccessException` warnings on a couple of cache folders, caused by a PUID/PGID ownership mismatch between the host and Duplicati's container user. Rather than debug container permissions for disposable, auto-regenerating cache data, those specific cache paths were simply added to the exclusion filter.

### 2. System Config
Source: OS-level configuration outside the Docker directory —
- `/etc/fstab`
- `/etc/environment`
- `/etc/snapraid.conf`
- Custom systemd units

> **Note on root-owned files:** `/etc/ssh/*` and NetworkManager connection profiles were initially included but throw the same permission error as above (correctly — they're root-owned 600 files, and Duplicati's non-root user shouldn't be able to read them). Rather than grant broader access, these were removed from the backup job and handled by manual documentation instead — see "Things Duplicati can't back up" below.

Both jobs use a smart retention policy (e.g. 7 daily / 4 weekly / 12 monthly) rather than unbounded retention, and both are AES-256 encrypted with the same encryption key.

## What's deliberately NOT backed up here

- **Bulk media** on the MergerFS pool itself — replaceable, not configuration, and would balloon backup size for no benefit
- **The Open-WebUI database / vector embeddings / Knowledge collections** — this is a database, not text/config, so it doesn't belong in Duplicati's file-based backup the same way. (If you're also using this alongside a git repo for the text/config side of your setup, keep the two separate — don't try to put a live database in git either.)

## Setting it up yourself

1. Confirm Duplicati is actually running (`docker ps` — it's easy to have it defined in your compose file but never actually started)
2. Create a job per source above, with two destinations each (local + cloud)
3. Set exclusion filters for `.bak` folders, unused services, and cache directories before your first run
4. **Save your Duplicati encryption key somewhere off this server** — a password manager, printed and stored safely, anything except "only on this machine." If the OS drive dies and this key only ever lived on it, every encrypted backup — local and cloud — becomes permanently unrecoverable. This is the single most important step in this whole setup.
5. Run each job manually once and check for warnings before trusting the schedule
6. **Actually test a restore.** A successful backup run is not the same as a working restore. Restore at least one file to an alternate location and diff it against the live copy to confirm byte-for-byte accuracy before you consider this done.

## Things Duplicati can't back up (documented manually instead)

**Network config (`eno1` static IP)** — no secrets in this file, safe to just document the values and recreate manually on a fresh install (adjust to your own actual values — these are placeholders):
- IP: `192.168.1.X/24`
- Gateway: `192.168.1.1`
- DNS: `8.8.8.8`, `8.8.4.4`
- Recreate with the equivalent `nmcli` commands for your distro/interface name

**SSH host keys** — no manual restoration needed at all. `ssh-keygen -A` regenerates them automatically on a fresh Debian install. The only consequence is a one-time `known_hosts` mismatch warning on machines that previously connected to the old keys, which clears with `ssh-keygen -R <hostname-or-ip>` on the client side.

## Real numbers from this setup, for reference

| Job | Source size | Stored (compressed+encrypted) |
|---|---|---|
| Docker Data (Local) | 12.9 GB | 2.4 GB |
| Docker Data (Google Drive) | 12.0 GB | 1.6 GB |
| System Config (Local) | ~576 KB | ~51 KB |
| System Config (Google Drive) | ~576 KB | ~51 KB |

All four ran clean with zero warnings once the exclusion filters and permission issues above were resolved.
