# arr-pi

Documentation and operational notes for **pi-arr** — a self-hosted media server running on a single Raspberry Pi 5.

This repository is the off-box record of how the box is built and why. It holds documentation only; **no secrets** (VPN keys, app passwords, API keys) are stored here by design — those live in the gluetun environment and each app's config volume on the box itself.

## The box

- **Hardware:** Raspberry Pi 5 (8 GB), aarch64, single NVMe SSD (QLC NAND), official M.2 HAT+ with active cooling.
- **OS:** Raspberry Pi OS Lite (Bookworm, 64-bit).
- **Orchestration:** Docker, managed via Portainer, in two stacks (`media` + `monitoring`).
- **Access:** Tailscale (location-independent); SD card retained as a permanent escape hatch.

## Architecture at a glance

- **Download / indexer stack** — qBittorrent, Prowlarr, Sonarr, Radarr, Bazarr, FlareSolverr — all share the network namespace of **gluetun** (AirVPN over WireGuard, with a kill switch), so nothing touches the internet outside the tunnel.
- **User-facing** — Jellyfin (playback) and Jellyseerr (requests) run off the VPN on the bridge network. These are the only intended endpoints; the *arr internals stay opaque after setup.
- **Monitoring** — Glances (live metrics), Scrutiny (NVMe SMART health), Uptime Kuma v2 (service up/down, VPN liveness, disk-space alerts).
- **Backups** — nightly encrypted restic snapshots of all config → rclone → Google Drive. Restore verified.

## Content intent

1080p, direct-play / H.264 bias (no 4K or remux). Greek + English subtitles on every item (Bazarr). Delete-after-watching by default.

## The documents

| File | What it covers |
|------|----------------|
| [`INVENTORY.md`](INVENTORY.md) | The concrete current state — every container, setting, path, port, and credential location. The source of truth for *what is*. |
| [`UPTIME_RELIABILITY_ROADMAP.md`](UPTIME_RELIABILITY_ROADMAP.md) | The hardening backlog against this box's failure modes (single disk, silent VPN, disk-fill, unsafe shutdown). Tracks what's done and what's next. |
| [`ISSUES.md`](ISSUES.md) | Anomalies noticed in passing — what was seen, why it's odd, where to start. Investigations recorded with original hypotheses and evidence-based corrections side by side. |
| [`PROJECT_NOTES.md`](PROJECT_NOTES.md) | Operating notes for working on the box — habits, reflexes, and the things that are easy to get wrong here. Approach, not state. |

## Status

Core stack operational. Roadmap items 1–3, 7, and 8 complete (off-box backups, alerting + uptime monitoring, log rotation + disk alerts, qBittorrent hygiene, dedicated VPN monitor). Open: #4 (UPS + NUT graceful shutdown), #5 (image-update notifications), #6 (auto-recovery).
