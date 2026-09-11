# arr-pi

Documentation and operational notes for **pi-arr** — a self-hosted media server running on a single Raspberry Pi 5.

This repository is the off-box record of how the box is built and why. It holds documentation only; **no secrets** (VPN keys, app passwords, API keys) are stored here by design — those live in the gluetun environment and each app's config volume on the box itself.

## The box

- **Hardware:** Raspberry Pi 5 (8 GB), aarch64, single NVMe SSD (QLC NAND), official M.2 HAT+ with active cooling.
- **OS:** Raspberry Pi OS Lite (**trixie** / Debian 13, 64-bit).
- **Orchestration:** Docker, managed via Portainer, in three stacks (`media`, `monitoring`, `dashboard`) plus a standalone `upsnap` stack.
- **Access:** Tailscale (location-independent); all services reached via the Tailscale host IP, never the LAN IP.

## Architecture at a glance

- **Download / indexer stack** — qBittorrent, Prowlarr, Sonarr, Radarr, Bazarr, FlareSolverr — all share the network namespace of **gluetun** (AirVPN over WireGuard, with a kill switch), so nothing touches the internet outside the tunnel.
- **User-facing** — Jellyfin (playback) and Seerr (requests, formerly Jellyseerr — migrated 2026-07-08, container name kept as `jellyseerr`) run off the VPN on the bridge network. These are the only intended endpoints; the *arr internals stay opaque after setup.
- **Monitoring** — Glances (live metrics), Scrutiny (NVMe SMART health), Uptime Kuma v2 (service up/down, VPN liveness via a Docker Container health monitor, disk-space alerts), WUD/What's Up Docker (image-update notifications, notify-only by design).
- **Dashboard** — Homepage, a single landing page with service tiles for the whole box; compose + config authoring source lives in this repo under `dashboard/`.
- **Wake-on-LAN** — UpSnap (standalone `upsnap` stack, added 2026-09-10), used to remote-wake `bliss`, a separate desktop on the same network — not part of this box's own hardware.
- **Backups** — nightly encrypted restic snapshots of all config → rclone → Google Drive. Restore verified.

## Content intent

1080p, direct-play / H.264 bias (no 4K or remux). Greek + English subtitles on every item (Bazarr). Delete-after-watching by default.

## The documents

| File | What it covers |
|------|----------------|
| [`INVENTORY.md`](INVENTORY.md) | The concrete current state — every container, setting, path, port, and credential location. The source of truth for *what is*. |
| [`UPTIME_RELIABILITY_ROADMAP.md`](UPTIME_RELIABILITY_ROADMAP.md) | The hardening backlog against this box's failure modes (single disk, silent VPN, disk-fill, unsafe shutdown). Tracks what's done and what's next. |
| [`ISSUES.md`](ISSUES.md) | Anomalies noticed in passing — what was seen, why it's odd, where to start. Investigations recorded with original hypotheses and evidence-based corrections side by side. |
| [`INSTRUCTIONS.md`](INSTRUCTIONS.md) | Operating notes for working on the box — habits, reflexes, and the things that are easy to get wrong here. Approach, not state. |

## Status

Core stack operational. Roadmap items 1–3, 5, 7, and 8 complete (off-box backups, alerting + uptime monitoring, log rotation + disk alerts, image-update notifications, qBittorrent hygiene, dedicated VPN monitor). Open: #4 (UPS + NUT graceful shutdown), #6 (auto-recovery).
