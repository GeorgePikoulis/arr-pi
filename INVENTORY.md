# pi-arr — system inventory
 
Reference snapshot of the media server as built. Last updated: 2026-07-07.
Secrets (VPN keys, app passwords/API keys) are intentionally **not** stored here — they
live in the gluetun environment and each app's config volume.
 
---
 
## Host
 
| Item | Value |
|------|-------|
| Device | Raspberry Pi 5, 8 GB RAM, active cooling + official M.2 HAT+ |
| Storage | Single NVMe — Micron MTFDKCD512QGN-1BN1AABLA, ~477 GiB. Holds OS **and** all data. No second disk. **QLC NAND, DRAM-less (HMB)** (Micron 2500-family, 232-layer QLC) — value drive: write speed/endurance degrade as it fills (pSLC cache shrinks), so keep extra free headroom. |
| OS | Raspberry Pi OS Lite (Bookworm, 64-bit / aarch64), kernel 6.12.75+rpt-rpi-2712 |
| Hostname | `pi-arr` |
| Primary user | `george` (UID 1000, GID 1000) |
| Boot | NVMe-first (`BOOT_ORDER=0xf461`), EEPROM updated |
| Timezone | Europe/Athens (host clock was found on `Europe/London` 2026-06-19 and corrected with `timedatectl set-timezone Europe/Athens`; UTC instant was always correct, only display/stamping was off) |
 
### Networking
| Item | Value |
|------|-------|
| Office LAN (current) | 10.0.1.0/24 — Pi at **10.0.1.59** |
| Home LAN (after move) | 10.0.69.0/24 |
| Tailscale (host-level) | **100.115.36.108** — location-independent; use this for all access |
 
---
 
## Host-level installs (apt / scripts)
 
- **Docker Engine** — installed via the get.docker.com convenience script. `george`
  added to the `docker` group.
- **Tailscale** — installed on the host (not containerized).
- **restic + rclone** — host binaries for the off-box config backup (see **Backups** below).
  Deliberately not containerized, so the backup survives a wedged Docker daemon.
- **msmtp** (1.8.28) — host SMTP sender for the low-disk-space CRITICAL alert (see
  **Low-disk-space alert** below). Host-level on purpose, so it can mail even when Docker is
  wedged. `msmtp-mta` installed too (provides a `sendmail` symlink). AppArmor profile **declined**
  at install (the module is disabled on this host anyway — `/sys/module/apparmor/parameters/enabled`
  = `N`); avoided so a confinement rule can't silently block the alert path.
- (Pi firmware/EEPROM tooling is part of Pi OS; `rpi-imager` was used only on a separate
  bootstrap drive during install and is not part of the running server.)
---
 
## Portainer (standalone container, not a stack)
 
| Field | Value |
|-------|-------|
| Image | `portainer/portainer-ce:latest` |
| Run | `docker run` (not a compose stack) |
| Ports | 8000 (agent tunnel), 9443 (HTTPS UI) |
| Volume | `portainer_data:/data` + `/var/run/docker.sock` |
| Restart | always |
| Logging | json-file, `--log-opt max-size=10m --log-opt max-file=3` (set 2026-06-19; standalone container can't take a stack `logging:` block, so the caps are run-flags — re-add them if ever recreated) |
 
---
 
## Stack: `media`
 
All download/indexer apps share gluetun's network namespace
(`network_mode: "service:gluetun"`) and exit through AirVPN. Jellyfin and Jellyseerr are
user-facing and stay **off** the VPN.
 
### gluetun (VPN + kill switch)
- Image: `qmcgaw/gluetun:latest`
- `cap_add: NET_ADMIN`, `devices: /dev/net/tun`
- Env:
  - `VPN_SERVICE_PROVIDER=airvpn`, `VPN_TYPE=wireguard`
  - `WIREGUARD_PRIVATE_KEY` / `WIREGUARD_PRESHARED_KEY` / `WIREGUARD_ADDRESSES` (from AirVPN Config Generator)
  - `SERVER_COUNTRIES=Netherlands,Bulgaria,Belgium,Romania,Switzerland` (2026-06-19: widened
    from `Netherlands` so AirVPN can rotate exits without faulting the VPN monitor; current
    observed exit still Lelystad, NL — `213.152.186.116`. NB: as of 2026-06-22 the VPN monitor
    no longer asserts exit country — it's a liveness check (gluetun container health), decoupled
    from this list so the set can change freely. See **Stack: `monitoring`** → gluetun monitor.)
  - `FIREWALL_VPN_INPUT_PORTS=17159` (AirVPN forwarded port, reserved in AirVPN panel for all devices)
  - `FIREWALL_OUTBOUND_SUBNETS=10.0.1.0/24,10.0.69.0/24,100.64.0.0/10` (office + home LAN + Tailscale; pre-loaded for the move)
  - `TZ=Europe/Athens`
- Volume: `/opt/arr/gluetun:/gluetun`
- **Published ports for the behind-VPN apps** (declared here, not on the apps themselves):
  8080 qBittorrent · 9696 Prowlarr · 8989 Sonarr · 7878 Radarr · 6767 Bazarr.
  FlareSolverr (8191) is **not** published — internal only.
### qBittorrent
- Image: `lscr.io/linuxserver/qbittorrent:latest`
- `network_mode: "service:gluetun"`, PUID/PGID 1000, `WEBUI_PORT=8080`
- Volumes: `/opt/arr/qbittorrent:/config`, `/data/torrents:/data/torrents`
- **Critical in-app settings:**
  - Listening port = **17159**; "Use UPnP/NAT-PMP" = **OFF**
  - Advanced → Network Interface = **`tun0`**; bind to **All IPv4 addresses**
    (this fixed DHT=0 / stuck-on-metadata / firewalled — do not revert to "Any interface")
  - DHT / PeX / LSD = on; anonymous mode = off; encryption = allowed
  - Default save path `/data/torrents`
  - Download categories: Sonarr uses `tv-sonarr`, Radarr uses `radarr`
  - **Seeding limits (2026-06-19):** remove torrent **and its files** at ratio 2 OR 2 weeks
    (20160 min) total seeding, whichever first — the reclaim lever for watched-and-deleted
    titles. Hardlink-safe for arr-imported titles (library copy survives); a manually-added,
    never-imported torrent lives only in `/data/torrents`, so removal deletes its sole copy.
  - **Pre-allocate disk space for all files = ON (2026-06-19):** the in-app disk-full guard.
    On `ext4` (confirmed via `findmnt -no FSTYPE /`) this is a `fallocate` extent reservation,
    **not** a zero-fill, so **no extra write/wear** on the QLC drive. Reserves a torrent's full
    size at add-time, so an oversized grab onto a too-full disk **fails cleanly at the start**
    instead of dying mid-write (the worst failure mode on a single QLC disk). NB: this is
    fail-fast-at-overflow, **not** a free-space *threshold* guard — qBittorrent has **no**
    "pause when free space below X" setting (long-open upstream FRs #10098/#10645/#23572,
    unimplemented). So it complements, not replaces, the host `df` alert (roadmap #3): the
    `df` alert is the trend/headroom detector, pre-allocation is the clean-failure backstop at
    the exact moment of overflow. Verified working: adding a 2.9 GiB torrent moved `df` Used by
    the full size at add-time.
  - **`linux-iso` category — seeding exempt (2026-06-19):** distro ISOs (Ubuntu, Mint, Kali)
    kept seeded to give back to the projects, organised under a `linux-iso` category. These are
    the manually-added, never-imported case — sole copy lives in `/data/torrents`, no
    `/data/media` hardlink — so the global "remove + files" seeding limit would delete them.
    Exempted via a **per-torrent** share-limit override set to **no limit** (ratio and time), so
    the global rule never applies. NB: per-category share limits exist in qBittorrent's data
    model but the **WebUI can't edit them** (desktop GUI / `categories.json` only — confirmed on
    5.2.1), so the override is per-torrent; the category itself is organisational.
  - **Global upload rate limit = 15000 KiB/s (~half of the 250 Mbit up line, 2026-06-19):** not
    there to protect other devices — the upstream **Flint 2 router runs QoS** with VPN/proxy at
    its lowest of three tiers, so seeding already yields to calls/gaming under contention. The
    cap does the two things QoS structurally can't: (1) bufferbloat insurance (strict priority
    reorders packets but doesn't keep the upstream buffer shallow — Flint 2 SQM would, but it
    can't run alongside QoS), and (2) protects **our own downloads**, which ride the same
    WireGuard tunnel and so land in the same lowest QoS tier — nothing but qBittorrent arbitrates
    seed-vs-grab inside the tunnel, so capping upload keeps seeding from choking incoming ACKs.
    Download limit left unlimited. (WireGuard adds a few % overhead, so real WAN upload sits just
    above the cap.)
### Prowlarr
- Image: `lscr.io/linuxserver/prowlarr:latest`
- `network_mode: "service:gluetun"`, PUID/PGID 1000
- Volume: `/opt/arr/prowlarr:/config`
- FlareSolverr added as an indexer proxy at `http://localhost:8191`, tagged; tag applied
  to indexers needing Cloudflare bypass. Syncs indexers to Sonarr/Radarr as "Apps."
### Sonarr / Radarr
- Images: `lscr.io/linuxserver/sonarr:latest`, `lscr.io/linuxserver/radarr:latest`
- `network_mode: "service:gluetun"`, PUID/PGID 1000
- Volumes: `/opt/arr/<app>:/config`, `/data:/data`
- Download client: qBittorrent at host `localhost`, port `8080` (shared namespace)
- Root folders: `/data/media/tv` (Sonarr), `/data/media/movies` (Radarr)
- **Quality intent:** 1080p only (WEB-DL/WEBRip/HDTV/Bluray-1080p); **no** Remux or 2160p;
  size caps in quality definitions (~3–5 GB/hr) to exclude bloated encodes; custom format
  nudging H.264/x264 positive for direct-play. Delete after watching; preload full
  seasons / complete series / movie-sequel collections for binge (tag to keep).
### Bazarr
- Image: `lscr.io/linuxserver/bazarr:latest`
- `network_mode: "service:gluetun"`, PUID/PGID 1000
- Volume: `/opt/arr/bazarr:/config`, `/data/media:/data/media`
- Connected to Sonarr/Radarr. Language profile: Greek + English, **at least one**
  required (cutoff satisfied by either), not both forced.
- Connected to Sonarr/Radarr. One language profile: Greek + English, both required — 
  Cutoff left empty (no named "None" in this build; you deselect to clear it), so Bazarr never stops at one language and fetches both el + en. 
  Set as default for Series + Movies and reassigned to existing items. (2026-06-20: changed from "either satisfies.")
- Gotchas banked: cutoff any stops at the first language found (one-or-the-other); 
  a single-language cutoff locks to that one — empty is the only setting that forces both. 
  And two separate single-language profiles do not stack: each title takes exactly one profile, so that route yields one-or-the-other, not both.
### FlareSolverr
- Image: `ghcr.io/flaresolverr/flaresolverr:latest`
- `network_mode: "service:gluetun"` (must share gluetun's IP so Cloudflare clearance
  matches the indexer query IP)
- Env: `LOG_LEVEL=info`, `TZ=Europe/Athens`. **No** PUID/PGID (image doesn't use them).
- Memory-hungriest container (spawns Chromium per request); fine on 8 GB.
### Jellyfin (NOT behind VPN)
- Image: `lscr.io/linuxserver/jellyfin:latest`
- PUID/PGID 1000, port **8096**
- Volumes: `/opt/arr/jellyfin:/config`, `/data/media:/data/media:ro`
- No hardware transcoding (Pi 5 has no HW encoder). Configured/used for **direct play** —
  set clients to Original quality.
- **`.keep` sentinel files at each library root** (`/data/media/movies/.keep`,
  `/data/media/tv/.keep`, added 2026-06-26): keep the library folders from ever being
  *completely* empty. Jellyfin's scanner **skips an empty library root and won't prune**
  deleted titles (empty-folder safety guard — treats zero-items as a possibly-unmounted
  volume, to avoid wiping the library on a mount failure). The consume-and-delete model
  drains libraries to zero regularly, so without the sentinel a drained library keeps showing
  ghost entries no matter how often the scan runs. Dotfile → not parsed as media. Created
  host-side (`/data/media` is rw there; the container mount is `:ro`). See ISSUES.md #2.
- **Library reconciler = the scheduled `Scan Media Library` task**, not real-time monitoring
  (inotify is unreliable for deletions over bind mounts). Deletions surface on the next
  scheduled scan — given the `.keep` sentinel above, which is what makes that reliable on the
  empty-folder edge case.  
### Seerr — formerly Jellyseerr (NOT behind VPN)
- Image: `ghcr.io/seerr-team/seerr:latest` — **migrated 2026-07-08** from
  `fallenbagel/jellyseerr:latest`. Jellyseerr and Overseerr merged into the unified **Seerr**
  project; the old image is frozen (no new releases, incl. security patches). The in-place DB
  migration ran automatically on first start — verified: UI on 3.3.0, request history intact,
  Kuma green, Homepage widget counts correct.
- **Container/service name deliberately kept `jellyseerr`** — the backup stop list, the Kuma
  monitor, the Homepage widget, and these docs all reference it by name, and Docker doesn't
  care that name and image differ. Renaming would mean touching the backup script for zero gain.
- **`init: true` is required** (compose) — the Seerr image ships no init process; without the
  flag, signal handling breaks (zombie processes, ungraceful stops). Per the official
  migration guide.
- Env: `LOG_LEVEL=info`, `TZ=Europe/Athens`. Port **5055** (unchanged — so Kuma/Homepage URLs
  were untouched). Runs as the `node` user (UID 1000), genuinely **non-root** (the old image
  ran parts as root).
- Volume: `/opt/arr/jellyseerr:/app/config` — **entire tree must be owned 1000:1000.** Gotcha
  banked: the old image had written `db/` as **root**; caught by a pre-migration
  `ls -ldn` check and fixed with `chown -R 1000:1000` *before* switching. A root-owned `db/`
  would have failed the non-root Seerr container at startup.
- Service connections unchanged (stored in Seerr's DB, survived migration):
  - Jellyfin → `http://jellyfin:8096`
  - Radarr → `http://gluetun:7878` · Sonarr → `http://gluetun:8989` (via the gluetun container
    name + API keys, as before)
  - Built-in TMDB integration — no user-supplied key.
- Homepage `type: jellyseerr` widget **verified compatible** with Seerr's API post-migration
  (Seerr kept `/api/v1`).
---
 
## Stack: `monitoring`
 
### Glances (live metrics)
- Image: `nicolargo/glances:latest`
- `network_mode: host`, `pid: host`, `GLANCES_OPT=-w` (web mode)
- Volumes: `/var/run/docker.sock:ro`, `/:/host:ro`
- Web UI on **61208**. Shows CPU, RAM, temperatures, disk usage/IO, per-process.
### Scrutiny (drive SMART health)
- Image: `ghcr.io/analogj/scrutiny:master-omnibus`
- `cap_add: SYS_RAWIO` **and** `SYS_ADMIN` (SYS_ADMIN required to read the NVMe SMART
  health log — without it temperature/power-on come back empty and status shows "Failed")
- Port **8082:8080**
- Devices: `/dev/nvme0n1`, `/dev/nvme0`
- Volumes: `/opt/scrutiny/config`, `/opt/scrutiny/influxdb`, `/run/udev:ro`
- Note: if device status reads "Failed" while all attributes are green, it's a stale
  record — trash the device in the UI and re-run `scrutiny-collector-metrics run`.
- **Notifications (active):** email via Gmail, set in `/opt/scrutiny/config/scrutiny.yaml`
  under `notify.urls`. Scrutiny shells out to **shoutrrr**, so the value is a shoutrrr SMTP
  URL:
  `smtp://<user>%40gmail.com:<app-password>@smtp.gmail.com:587/?fromAddress=<user>@gmail.com&toAddresses=<user>%2Bpiarr@gmail.com&fromName=pi-arr&encryption=ExplicitTLS&usestarttls=yes`
  - Gmail needs an **app password** (2FA on); the normal password is rejected over SMTP.
  - **URL-encode the userinfo:** `@`→`%40` in the username (an unescaped `@` gives a
    `net/url: invalid userinfo` parse error and the container **restart-loops**); `+`→`%2B`
    in the plus-addressed recipient.
  - `fromName=pi-arr` makes alerts self-identify per box; `+piarr` enables Gmail filtering.
  - The yaml is `chmod 600` (holds the app password); the container runs as root, so it can
    still read it. Keys in `notify.urls` query string (`fromAddress`/`toAddresses`) keep a
    literal `@` — only the userinfo before `@host` needs encoding.
  - Test: `curl -X POST http://localhost:8082/api/health/notify`. **A `{"success":true}`
    response is NOT proof of delivery** — confirm by inbox; on failure read the container
    logs for the `Sending notifications to smtp://…` line (shows the real auth/parse error).
  - **`notify.level` is removed in this image.** It moved to the web UI; a stray `level:` key
    fails config validation and restart-loops the container (this bit us once). Current UI
    settings: **Device Status Threshold = Both**, **Notify Filter Attributes = All** — the
    most-sensitive pairing = earliest warning, the right call on a single-disk SPOF.
### Uptime Kuma (service up/down monitoring)
- Image: `louislam/uptime-kuma:2` — pinned to the **v2 major** (gets patches, no surprise v3
  jump; arm64 supported). Added to the `monitoring` stack alongside Glances + Scrutiny.
- Port **3001:3001**; volumes `/opt/uptime-kuma:/app/data` (SQLite `kuma.db` + history) **and**
  `/var/run/docker.sock:/var/run/docker.sock:ro` (added 2026-06-22 for the gluetun Docker
  Container monitor — see below). `restart: unless-stopped`. First-run DB choice = **SQLite**
  (right for a handful of monitors on one box; MariaDB is for large deployments and only adds
  failure surface here). **NB on the socket:** it grants Kuma full Docker-daemon control (the
  `:ro` flag locks the socket *file*, not the API — not a real safeguard); acceptable here only
  because Kuma is never published to the WAN (Tailscale/LAN only) and Portainer already mounts it.
- **Monitor addressing:** Uptime Kuma is a bridge container, so `localhost` means *itself*,
  not the Pi. All HTTP monitors target the **Tailscale host IP** `http://100.115.36.108:<port>`
  — location-independent (survives the office→home move), and it's also how the VPN-gated apps
  are reached (they're published on the host by gluetun, so from here they're just host ports).
  Probe-verified against Jellyfin before building the rest.
- **HTTP(s) monitors (8):**
  - Jellyfin → `:8096/health` · Jellyseerr → `:5055`
  - qBittorrent → `:8080` · Bazarr → `:6767`
  - Prowlarr → `:9696/ping` · Sonarr → `:8989/ping` · Radarr → `:7878/ping`
    (`/ping` = unauthenticated 200 health check; if a URL base is set and `/ping` 404s, drop
    it for the bare base URL, which also returns 200)
  - WUD → `:3002/health` (added 2026-07-07 — the same endpoint WUD's own image HEALTHCHECK
    curls). Retries 2; on the published status page. Deliberately **not** in the backup
    maintenance window: WUD isn't on the backup stop list, so it rides through the 04:00
    backup running — suppressing it would mask real signal. Known limit, accepted: the probe
    catches the container being down, not WUD silently failing to *detect* updates.
  - **FlareSolverr (8191) excluded** — internal-only, never published, unreachable from
    outside gluetun's namespace; its health is implied by Prowlarr working.
- **Per-monitor retries = 2** (v2 defaults *new* monitors to 0, so a single blip would fire an
  alert). Interval left at 60 s.
- **gluetun-namespace signature:** the five VPN-gated apps share gluetun's namespace, so a
  gluetun drop shows as **all five going red at once** — read that as "gluetun," not five
  separate faults. A *direct* gluetun check also exists via the **`pi-arr gluetun (VPN)`** Docker
  Container monitor (see below), so the all-five-red pattern remains the in-Kuma tell while the
  container-health monitor adds the dedicated signal (retired roadmap #2's loose end via #8).
- **Notifications:** Email (SMTP) via the same Gmail, set in Settings → Notifications. Uptime
  Kuma uses its **own** SMTP client (nodemailer) via a plain form — **not** a shoutrrr URL, so
  **no URL-encoding** (literal `@`, no `%40`). Hostname `smtp.gmail.com`, port **587**,
  Security **STARTTLS**, username/From/To = the Gmail address, password = the app password.
  - **"Apply on all existing monitors" must be ON** — otherwise the channel attaches to no
    monitors and nothing ever emails (silent gap). "Default Enabled" ON so new monitors inherit
    it.
  - Custom Subject/Body left **blank** (default names the monitor + status). v2 templating is
    **LiquidJS** (lowercase `{{ name }}`, `{{ status }}`); the old uppercase `{{STATUS}}`-style
    placeholders render as literal text — avoid.
- **Push monitor `pi-arr disk space` (added 2026-06-19):** the WARN half of the low-disk-space
  alert (roadmap #3). Monitor Type = **Push**; the host script (`pi-arr-disk-alert.sh`) pushes
  `up`/`down` with a free-space message each run. Kuma has **no native host-disk monitor** (that
  FR was closed as not-planned), so Push is the mechanism. **Heartbeat Interval = 1800 s**,
  deliberately 2× the script's 15-min timer cadence: long enough not to flap on one missed run,
  short enough that a *dead script* flips it `Down` (so the alerter's own death surfaces). Uses
  the same Gmail notification channel as the others (so the WARN email needs no new secret).
- **Docker Container monitor `pi-arr gluetun (VPN)` (added 2026-06-22, replaced the
  `pi-arr vpn exit` Push monitor):** the dedicated gluetun/VPN check (roadmap #8). Monitor
  Type = **Docker Container**, Container Name = `gluetun`, via a Docker Host (connection type
  **Socket**, `/var/run/docker.sock`) registered in Kuma's settings. Reads gluetun's
  `State.Health.Status` — i.e. gluetun's own active egress healthcheck (ICMP/DNS every minute,
  TCP+TLS dial every 5 min through the tunnel), so "green" means real tunnel egress worked
  recently, not merely that the container is running. **Liveness only — no country assertion**
  (decoupled from `SERVER_COUNTRIES`; the country/leak check was dropped 2026-06-22). Interval
  60 s, **Retries = 2** (v2 still defaults new monitors to 0 — the silent-no-alert trap); same
  Gmail channel (no new secret). **Why this needs v2:** Kuma counts a running, healthchecked
  container `up` *only* when health = `healthy`, else `down` (PR #4372); v1 reported unhealthy as
  *pending* and never alerted.
### WUD — What's Up Docker (image-update notifications)
- Image: `ghcr.io/getwud/wud:latest` (arm64 supported). Added 2026-07-07 (roadmap #5).
  **Notify-only by construction:** the SMTP trigger is the only trigger defined — no update
  triggers exist, so WUD cannot recreate containers. Updates are performed deliberately in
  Portainer (workflow below).
- Port **3002:3000** (host 3000 is Homepage's; WUD's internal port stays 3000). **No basic
  auth** on the UI (anonymous) — same exposure posture as Homepage: Tailscale/LAN only,
  never WAN.
- Volumes: `/var/run/docker.sock:ro` (same caveat as Kuma's socket — `:ro` locks the file,
  not the API) and `/opt/wud:/store` (state store, so a recreate doesn't re-announce every
  already-known update). `TZ=Europe/Athens`, `restart: unless-stopped`, shared log cap.
- **Watcher:** the default `local` socket watcher, `WUD_WATCHER_LOCAL_CRON=30 6 * * *`
  (daily 06:30 Athens — clear of the 04:00 backup window). All 15 containers watched
  (WATCHBYDEFAULT). **Digest watching is automatic:** WUD enables it by default for
  non-semver tags, and everything here runs `:latest` — no labels needed on any container.
  Consequence: detection reports "new digest," not a version delta — read the project's
  release notes for what changed. Daily cadence keeps registry pull-API quota pressure
  negligible (most images are lscr.io/ghcr.io; only ~5 on Docker Hub).
- **Email trigger:** `WUD_TRIGGER_SMTP_GMAIL_*` → `smtp.gmail.com` port **465**,
  `TLS_ENABLED=true`. NB that flag means **implicit TLS**, not STARTTLS — don't pair it
  with 587 (the other three Gmail senders on this box use 587/STARTTLS; WUD uses the other
  door, both fine with Google). Uses `FROM_ADDRESS` (bare `FROM` is deprecated);
  `FROM_NAME=pi-arr`; To = the `+piarr` plus-address. Credentials injected as Portainer
  stack env (`WUD_GMAIL_USER/PASS/TO`) — same pattern as the Homepage `HOMEPAGE_VAR_*`
  secrets, keeping the app password out of the YAML. It's the shared Gmail app password —
  the **fourth on-box copy**; see the auth map.
- **Update workflow:** email / Homepage tile → WUD UI (`:3002`) to see what's pending →
  **any app except gluetun:** Portainer → the container → **Recreate** with "Re-pull image"
  (namespace apps rejoin gluetun's running namespace, same as after the nightly backup
  restarts); confirm the Kuma monitor green after. **gluetun: never the per-container
  recreate** — attended **stack-level** update only (Stacks → media → update with re-pull),
  so `depends_on: service_healthy` governs ordering; verify the kill switch after (exit IP
  still AirVPN). NB a stack-level re-pull updates *every* service in that stack with a
  newer image, not just the one you came for.
- **Rollback:** the restic backup reverts *config only* — it cannot roll back an image
  (`:latest` still resolves to the new one after a restore). Image rollback = pin the
  previous version tag and redeploy, or recreate against the old image ID while it's
  still on disk un-pruned.
- Clearing is automatic: on the next scan the running digest matches latest and the
  pending count drops — no acknowledge step.
- **Blind spot banked (2026-07-08):** digest watching detects *new* digests — it cannot detect
  a repo that has gone quiet because the project **moved or was renamed**. `fallenbagel/jellyseerr`
  froze when Jellyseerr merged into Seerr, and WUD reported nothing wrong for weeks — the exact
  failure #5 exists to prevent, defeated silently. A "suspiciously long time since any update"
  is a signal to check the project's homepage, not a comfort. (Found via the Seerr version
  banner in the app's own UI, not via monitoring.)  
---
 
## Stack: `dashboard`
 
Homepage (`gethomepage`) landing dashboard — a single info strip + service tiles for the whole
box. Added 2026-06-23. **Socket-free by decision:** no `docker.sock` mount — it uses service
widgets + ping, **not** container discovery (so it never gets Docker-daemon control, unlike
Kuma/Portainer). Authoring source for the compose + the two config YAMLs is in this repo under
`dashboard/` (see `dashboard/README.md`); on the box the stack lives in Portainer and the YAMLs
live in `/opt/homepage`.
 
### Homepage
- Image: `ghcr.io/gethomepage/homepage:latest` (arm64 supported). New Portainer stack `dashboard`,
  single service.
- `PUID/PGID=1000`, `TZ=Europe/Athens`, `restart: unless-stopped`, the shared log cap
  (`json-file`, `max-size: 10m`, `max-file: 3`).
- Port **3000:3000** (verified free on the host first).
- Volume `/opt/homepage:/app/config`, owned `1000:1000`. YAML-only (no DB).
- **`HOMEPAGE_ALLOWED_HOSTS` (required — Homepage refuses connections without it):**
  comma-separated `host:port`, no spaces. Currently `100.115.36.108:3000,10.0.1.59:3000`
  (Tailscale + current LAN); **add `10.0.69.x:3000` after the office→home move.**
- **Addressing rule (every widget + link):** `http://100.115.36.108:<port>` — Tailscale host IP,
  **never** container names, and **no trailing slash** (a trailing `/` breaks widgets — the #1
  cause of "API Error").
- **Secrets:** kept out of the YAML via Homepage's env-var substitution — values injected as
  `HOMEPAGE_VAR_*` (stack env) and referenced as `{{HOMEPAGE_VAR_NAME}}`. They live only on the
  box (Portainer stack env). If ever inlined instead, `chmod 600` the YAMLs like the Scrutiny one.
### `widgets.yaml` (top info strip)
- **Glances** info widget at `http://100.115.36.108:61208`, showing `cpu`, `mem`, `cputemp`,
  `disk: /`. The `version` field **must match the running Glances major version** (required for
  v4+, defaults to 3 if omitted) — set per the container, don't assume.
- A **datetime** widget.
### `services.yaml` groups
- **Media** — Jellyfin (`type: jellyfin`, **new** API key created in Jellyfin → Dashboard →
  Advanced → API Keys), Jellyseerr (`type: jellyseerr`, key from Settings → General).
- **Acquisition** — qBittorrent (`type: qbittorrent`, WebUI **username + password**, not a key),
  Sonarr/Radarr (`type: sonarr`/`radarr`, existing API keys), Prowlarr + Bazarr as plain links
  with a `ping` status dot. (NB: `ping` targets the host IP, so it confirms the Pi is reachable,
  not the app itself — all the VPN-gated apps share this one published host IP.)
- **Operations** — Uptime Kuma rollup (`type: uptimekuma`, `url: …:3001`, `slug:` = the published
  status-page slug, `fields: ["up","down","uptime","incident"]`), Scrutiny (`type: scrutiny`,
  `url: …:8082`, no key), WUD (`type: whatsupdocker`, `url: …:3002`,
  `fields: ["monitoring","updates"]` — watched/pending-update counts; no credential fields,
  WUD runs without auth), Portainer as a link.
### Kuma prerequisite
- A single **published** Uptime Kuma status page backs the rollup tile: all monitors (8 HTTP + the
  `pi-arr gluetun (VPN)` Docker-Container monitor + the `pi-arr disk space` Push monitor) in one
  group; the tile reads that page via its slug.
---
 
## Logging / log rotation (all containers)
 
Per-service log caps so container logs can't fill the root partition (roadmap #3, 2026-06-19).
Chose compose-level `logging:` over `/etc/docker/daemon.json` deliberately — see the gotcha below.
 
- **Both stacks:** a shared YAML anchor at the top of each compose, with `logging: *default-logging`
  on every service:
  ```yaml
  x-logging: &default-logging
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
  ```
- **Portainer (standalone):** can't take a stack `logging:` block, so the same caps are
  `docker run` flags (`--log-opt max-size=10m --log-opt max-file=3`) — see the Portainer table.
- Result: all 14 containers cap at 10 MB × 3 = 30 MB each (~420 MB worst case).
- **Why not `daemon.json`:** (1) daemon log-opts apply only to **newly created** containers and
  need a full **daemon restart** to load — and a daemon restart force-cycles gluetun, with the
  namespace apps **not** ordered by `depends_on` (that's Compose-only, not honoured on daemon
  restart / reboot). (2) `daemon.json` is a host file **outside the restic backup set**, so the
  caps wouldn't survive a bare-metal restore; per-service `logging:` lives in the backed-up
  compose. Applying it was a normal Portainer stack redeploy (compose changed → services
  recreated); gluetun's recreate was attended and Compose-orchestrated, kill switch verified.
---
 
## Low-disk-space alert (host-level)
 
Dual-layer free-space alert on `/` (single filesystem — OS + all data). Roadmap #3, 2026-06-19.
Host script + systemd timer, mirroring the backup machinery; decoupled from Docker on purpose.
 
- **Script:** `/usr/local/sbin/pi-arr-disk-alert.sh` (root, 700). Measures free GB on `/` via
  `df -B1` (byte-exact, no column-counting), then branches:
  - **OK** (> 90 G free) → push `up` heartbeat to Kuma.
  - **WARN** (≤ 90 G, ~19% free) → push `down` to Kuma → WARN email via Kuma's channel.
  - **CRITICAL** (≤ 35 G, ~7.5% free) → push `down` to Kuma **and** send a host email via
    **msmtp** (subject `[pi-arr] CRITICAL: low disk space`). Both channels fire at CRIT.
  - Thresholds are constants at the top of the script. Set conservatively for the **QLC** drive
    (keep headroom — see Host → Storage). Each send helper ends in `|| echo … >&2` so a failure
    in one channel never aborts the run (and never suppresses the other) under `set -e`.
- **Timer:** `pi-arr-disk-alert.timer` → `pi-arr-disk-alert.service` (oneshot), every **15 min**
  (`OnBootSec=5min`, `OnUnitActiveSec=15min`, `Persistent=true`). The service depends on
  `network-online.target` but **deliberately NOT** `docker.service` — the msmtp path must fire
  even when Docker is wedged.
- **msmtp:** config at `/etc/msmtprc` (root, **600**) → `smtp.gmail.com:587`, STARTTLS, Gmail
  **app password** (this is the **third** on-box copy of that password — see auth map).
  `logfile /var/log/msmtp.log`. `From: pi-arr <…>`, `To: …+piarr@gmail.com` (threads/filters with
  the other pi-arr mail). NB: a zero exit is **not** proof of delivery — confirm by inbox / the
  log's `250` line, same lesson as the Scrutiny test.
- **Kuma side:** the `pi-arr disk space` Push monitor, heartbeat 1800 s (see Uptime Kuma section).
- **Verified** by a real fill-test (`fallocate` on `/`) crossing WARN then CRITICAL on genuine
  `df` output — both channels confirmed, then fillers removed and space reclaimed to baseline.
---
 
## VPN exit check — retired 2026-06-22 (now a Kuma Docker Container monitor)
 
The host-level VPN exit check (script `pi-arr-vpn-check.sh` + systemd timer + Kuma Push monitor,
roadmap #8) was **removed 2026-06-22**. It joined a throwaway `curlimages/curl` container to
gluetun's namespace, queried `ipinfo.io`, and asserted the exit country. Why it was replaced:
- **Throttling:** AirVPN's shared exit IP collectively blew ipinfo's unauthenticated limit
  (1,000 req/day, **per IP, shared across all users on that exit**), so the check intermittently
  got a 429 with no parseable country and false-flagged `down` ("egress OK but no country").
- **Country coupling:** asserting the exit country tied the monitor to `SERVER_COUNTRIES`, which
  is meant to change freely.
 
Replaced by the **`pi-arr gluetun (VPN)` Docker Container monitor** (see **Stack: `monitoring`**
→ Uptime Kuma): it reads gluetun's own `State.Health.Status` via the Docker socket — the same
egress-liveness signal (gluetun's healthcheck actively pings/DNS-queries each minute and
TCP+TLS-dials every 5 min through the tunnel), with no external lookup, nothing to rate-limit,
and gluetun never touched. **Dropped in the swap:** the country/leak assertion — liveness only
now. **Deleted:** `/usr/local/sbin/pi-arr-vpn-check.sh`,
`/etc/systemd/system/pi-arr-vpn-check.{service,timer}`, the embedded Kuma push token (died with
the script), and the pre-pulled `curlimages/curl` image.
---
 
## Backups — off-box config snapshots
 
Encrypted, deduplicated snapshots of the **config** (not media) to **Google Drive**, run by a
host-level systemd timer as **root**. Decoupled from Docker on purpose (survives a wedged
daemon). Roadmap item #1; restore verified to scratch and over live.
 
### Tooling (host binaries, not containers)
- **restic** 0.18.0, repo format v2 / compressed — `apt install` then `restic self-update`.
- **rclone** (official install script, current arm64) — provides the Drive backend; restic
  has no native Drive backend and shells out to rclone (`rclone:<remote>:<path>`).
### Repository
| Field | Value |
|-------|-------|
| Backend | Google Drive via rclone remote `gdrive` |
| Scope | `drive.file` — least privilege; rclone sees **only files it created**, not the rest of the Drive |
| OAuth | own Google Cloud client (Desktop app), **published to production** (Testing mode expires tokens after 7 days) |
| Repo | `rclone:gdrive:rclone/restic-pi-arr` (ID `bd9ae25b`) |
 
### What's captured
- `/opt/arr` — all app configs.
- `/opt/homepage` — the Homepage dashboard config (YAML-only). **Added to the include set
  2026-06-23** when the `dashboard` stack was built — it sits outside `/opt/arr`, so it had to be
  added explicitly to the script's path list or it wouldn't be captured. `--group-by host`
  retention absorbs the changed path-set automatically. YAML-only (no DB), so it does **not** join
  the pre-backup container-stop list — safe to snapshot live.
- `/var/lib/docker/volumes/portainer_data/_data` — the Portainer volume, which holds the
  **stack definitions** at `compose/<id>/docker-compose.yml`. Because the compose was pasted
  into Portainer's editor, the gluetun **WireGuard secret is inline in there** — this is the
  crown jewel. `/opt/arr` + these compose files rebuild the box on a fresh Portainer without
  needing Portainer's own DB.
- **Excluded:** `/opt/arr/jellyfin/cache` (~5.9 GB of regenerable transcodes — see ISSUES.md #1).
- **Not added:** `/opt/wud` (WUD's state store) — regenerable (WUD rebuilds it by rescanning;
  worst case is a repeated update notification), so deliberately outside the set.
### Schedule & consistency
- `pi-arr-backup.timer` → `pi-arr-backup.service` (oneshot) → `/usr/local/sbin/pi-arr-backup.sh`.
- Fires `*-*-* 04:00` Athens, `RandomizedDelaySec=15min`, `Persistent=true` (runs a missed
  backup after downtime).
- For a mid-write-safe snapshot the script **stops the eight app containers** (bazarr,
  qbittorrent, sonarr, radarr, prowlarr, jellyfin, jellyseerr, portainer), backs up, then
  restarts them. **gluetun is left running** — it has no DB and must never cycle (kill switch
  stays up; apps rejoin its namespace on restart). Downtime ≈ 30–60 s. A `trap` guarantees
  the apps restart even if the backup fails or the job is killed.
- Retention (run after restart; repo-only, no downtime): `forget --group-by host
  --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --prune`. `--group-by host` keeps all
  pi-arr snapshots on one timeline regardless of path-set changes.
### Credentials — where they live
- **restic repo password** — the one unrecoverable secret. Master copy is **off-box** (in
  george's password manager); on-box at `/root/.config/restic/password` (root, 600) for
  unattended runs. That file is outside the backup set, so it never lands in the repo.
- **rclone configs — two independent copies, by necessity.** `/home/george/.config/rclone/`
  for interactive use; `/root/.config/rclone/` for the backup job. They must be separate
  because rclone **rewrites its config on every token refresh (~hourly)** — a shared file run
  under sudo gets chowned to root and locks george out. Both hold the same token; root's is
  the one the job depends on.
### Restore (disaster recovery)
On any machine with restic + rclone and the two secrets (repo password + an rclone config
carrying a valid token, or re-auth of the same OAuth client):
 
```bash
sudo RCLONE_CONFIG=/root/.config/rclone/rclone.conf \
  restic -r rclone:gdrive:rclone/restic-pi-arr \
  --password-file /root/.config/restic/password \
  restore <snapshot-id> --target <path>
```
 
`--target /` restores in place (stop the apps first; add `--exclude /opt/arr/gluetun` while
the VPN is running). Recreate the two stacks from the restored compose files, restore
`/opt/arr` alongside.
 
### Gotchas banked
- **A `drive.file` repo is visible only to the token that created it** — every restic/rclone
  op must use a config carrying that token, or the repo appears not to exist.
- **`sudo restic` uses *root's* default rclone config**, not george's — point it with
  `RCLONE_CONFIG=` (the script exports this).
- **Restore over live defaults to overwrite, not delete** — changed files revert, newer extra
  files are left in place. Use `--delete` only after a `--dry-run` (and it requires an
  `--include`/`--exclude` when `--target /`).
- **Still unrehearsed:** zero-to-restored on *bare metal* (fresh box, nothing but the repo +
  the two passwords). The mechanism is proven; the from-scratch drill is the final check.
---
 
## Filesystem layout (on the single NVMe)
 
```
/data
├── torrents/{movies,tv}     # qBittorrent download targets
└── media/{movies,tv}        # Sonarr/Radarr libraries; Jellyfin reads here
/opt/arr/{gluetun,qbittorrent,prowlarr,sonarr,radarr,bazarr,jellyfin,jellyseerr}
/opt/scrutiny/{config,influxdb}                    # config/ holds scrutiny.yaml (notify config, 600)
/opt/uptime-kuma                                   # Uptime Kuma data (SQLite kuma.db + history)
/opt/homepage                                      # Homepage dashboard config (widgets.yaml, services.yaml; owned 1000:1000)
/opt/wud                                           # WUD state store (regenerable; deliberately outside the backup set)
 
# off-box backup machinery (host-level)
/usr/local/sbin/pi-arr-backup.sh                   # backup wrapper (root, 700)
/etc/systemd/system/pi-arr-backup.{service,timer}
/root/.config/restic/password                      # repo password (root, 600)
/root/.config/rclone/rclone.conf                   # backup job's rclone config
/home/george/.config/rclone/rclone.conf            # george's rclone config (interactive)
 
# low-disk-space alert machinery (host-level)
/usr/local/sbin/pi-arr-disk-alert.sh               # disk-alert script (root, 700)
/etc/systemd/system/pi-arr-disk-alert.{service,timer}
/etc/msmtprc                                       # msmtp config + Gmail app password (root, 600)
/var/log/msmtp.log                                 # msmtp send log (real SMTP errors land here)
```
- `/data` and its subtrees are owned `1000:1000`.
- Downloads and media live on the **same filesystem** so Sonarr/Radarr move grabs by
  hardlink (no double-write, no extra space). Keep this property if storage is ever added.
  **Verified 2026-06-19:** an imported title shows the **same inode** under both
  `/data/torrents` and `/data/media` with **link count 2** (checked via
  `find /data/media -links +1` then `find /data/torrents -inum <inode>`) — one physical
  file, two names. This is what makes the qBittorrent "remove torrent + files" seeding limit
  safe for arr-imported titles: it drops the torrent's link, and the library copy survives.
---
 
## Service access (via Tailscale `100.115.36.108`, or LAN IP)
 
| Service | Port | Notes |
|---------|------|-------|
| Portainer | 9443 | HTTPS |
| Homepage | 3000 | landing dashboard (`dashboard` stack) |
| Jellyfin | 8096 | media playback |
| Jellyseerr | 5055 | requests / discovery |
| qBittorrent | 8080 | behind VPN (published on gluetun) |
| Prowlarr | 9696 | behind VPN |
| Sonarr | 8989 | behind VPN |
| Radarr | 7878 | behind VPN |
| Bazarr | 6767 | behind VPN |
| Glances | 61208 | metrics |
| Scrutiny | 8082 | drive health |
| Uptime Kuma | 3001 | service up/down monitoring |
| WUD | 3002 | image-update notifications (`monitoring` stack) |
| FlareSolverr | 8191 | internal only (not published) |
 
---
 
## API keys / auth map (where to find them)
 
- qBittorrent: authenticated by **WebUI username/password** (no API key).
- Each *arr app's API key: app → Settings → General. Used by:
  - Prowlarr → Sonarr & Radarr
  - Bazarr → Sonarr & Radarr
  - Jellyseerr → Sonarr & Radarr
  - Homepage → Sonarr & Radarr (read-only widget queries; reuses the existing keys)
- **Jellyfin API key (Homepage)** — a **new** key created in Jellyfin → Dashboard → Advanced →
  API Keys for the Homepage `jellyfin` widget (the box's first Jellyfin API key; Jellyseerr uses
  Jellyfin's auth, not an API key). Revoke/regenerate from the same screen.
- **Homepage widget secrets** — Jellyfin/Jellyseerr/Sonarr/Radarr keys + the qBittorrent WebUI
  username/password are kept out of the YAML via env-var substitution: injected as `HOMEPAGE_VAR_*`
  through the `dashboard` Portainer stack env and referenced as `{{HOMEPAGE_VAR_NAME}}` in
  `/opt/homepage/services.yaml`. They live only on the box (Portainer stack env). No new external
  credential — they're copies/uses of keys already documented above.
- VPN credentials: in the gluetun service environment (WireGuard keys/addresses).
- Notification email (Scrutiny + Uptime Kuma + host msmtp + WUD): a Gmail **app password**
  (2FA enabled). Four on-box copies, by necessity: inline in the shoutrrr URL in
  `/opt/scrutiny/config/scrutiny.yaml` (Scrutiny); in Uptime Kuma's SQLite DB
  (`/opt/uptime-kuma/kuma.db`); in `/etc/msmtprc` (root, 600) for the host low-disk-space
  CRITICAL alert; and in the `monitoring` Portainer stack env (`WUD_GMAIL_PASS`) for WUD's
  update-notification emails (2026-07-07). Master copy in george's password manager.
  Revoke/rotate via Google Account → Security → App passwords (rotating means updating all
  four on-box copies).
- Backup secrets (restic repo password, rclone OAuth token): see **Backups → Credentials** above.
- **Uptime Kuma push token** — the `pi-arr disk space` Push monitor has a token embedded in its
  host script (`pi-arr-disk-alert.sh`, root/700). Not a credential to any external service — the
  token only lets a local caller post heartbeats to that one monitor — but kept out of
  world-readable files all the same. Regenerate from the monitor's page in Uptime Kuma if ever
  needed. (The former `pi-arr vpn exit` monitor's token was retired with that monitor on
  2026-06-22 — see **VPN exit check — retired**.)
