# `dashboard` stack — Homepage

Deploy-ready config for a Homepage dashboard on **pi-arr** (Pi 5, aarch64, Docker via
Portainer). Image `ghcr.io/gethomepage/homepage:latest` (arm64 supported). Single service,
published on **3000**, **socket-free** by decision (no `docker.sock` mount — service widgets
+ ping, not container discovery).

These files are the authoring source. On the box, the running stack lives in Portainer (and
its compose is captured by the Portainer-volume backup); the two config YAMLs live in
`/opt/homepage`. Copy from here to there.

```
dashboard/
├── docker-compose.yml   # paste into the Portainer "dashboard" stack
├── .env.example         # the HOMEPAGE_VAR_* secrets to fill in (Portainer env editor)
├── README.md            # this file
└── config/
    ├── widgets.yaml      -> /opt/homepage/widgets.yaml
    └── services.yaml     -> /opt/homepage/services.yaml
```

## Golden rules (banked gotchas)

- **Addressing:** every widget/link uses `http://100.115.36.108:<port>` (Tailscale host IP) —
  **never** container names, and **no trailing slash** (a trailing `/` breaks widgets; it's the
  #1 cause of "API Error").
- **`HOMEPAGE_ALLOWED_HOSTS` is required** — without it Homepage refuses connections.
  Comma-separated `host:port`, no spaces. Add `10.0.69.x:3000` after the office→home move.
- **Secrets stay out of YAML** — injected as `HOMEPAGE_VAR_*` and referenced as
  `{{HOMEPAGE_VAR_NAME}}`. They live only on the box.

## Manual steps that need you (on the box / in the apps)

I built the config from here, but these require box/app access and your secrets:

1. **Kuma status page (do FIRST — services.yaml needs the slug).** Uptime Kuma → Status Pages →
   create **one published** status page. Add **all** monitors into a single group: the 7 HTTP
   monitors + the `pi-arr gluetun (VPN)` Docker-Container monitor + the `pi-arr disk space` Push
   monitor. Publish, then note the **slug** (lower-case, the part after `/status/`). Put it in
   `config/services.yaml` → Uptime Kuma → `slug:` (replace `CHANGEME-kuma-status-slug`).
2. **Verify port 3000 is free** on the host (should be): `sudo ss -ltnp | grep ':3000'` → no output.
3. **Create `/opt/homepage` owned by 1000:1000:**
   `sudo mkdir -p /opt/homepage && sudo chown -R 1000:1000 /opt/homepage`, then drop
   `widgets.yaml` and `services.yaml` in it.
4. **Confirm the Glances major version** and set `version:` in `widgets.yaml` (3 or 4 — don't
   assume): `docker exec glances glances --version` (or check the image tag). I defaulted it to 4.
5. **Jellyfin API key (NEW):** Jellyfin → Dashboard → Advanced → API Keys → create one for
   Homepage. Also gather: Jellyseerr key (Settings → General), Sonarr/Radarr existing keys
   (Settings → General), qBittorrent WebUI **username + password**. Put them in the stack env
   per `.env.example`.
6. **Deploy** the stack in Portainer (name it `dashboard`), with the env vars set.
7. **Backup include — don't skip.** `/opt/homepage` is **outside** the current restic include
   set (`/opt/arr` + the Portainer volume), so add `/opt/homepage` to the path list in
   `/usr/local/sbin/pi-arr-backup.sh`. `--group-by host` retention absorbs the changed path-set
   automatically. Homepage is YAML-only (no DB), so it does **not** need adding to the
   pre-backup container-stop list — safe to snapshot live.

## Verify

- Every widget loads without "API Error"; the Uptime Kuma tile shows real up/down counts.
- For any widget that errors, check in order: **trailing slash** on the url, the
  **`HOMEPAGE_ALLOWED_HOSTS`** entry, and **reachability from inside the container**
  (`docker exec homepage wget -qO- http://100.115.36.108:<port>`).

## Secrets alternative (if you inline instead of env-var substitution)

Recommended path is env-var substitution (above). If you inline keys into the YAML instead,
`chmod 600` the files (as done for the Scrutiny yaml). Either way they live only in `/opt/homepage`.
