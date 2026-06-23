# pi-arr — uptime & reliability roadmap
 
Prioritized backlog for hardening the media server over time. Ordered by impact against
this box's specific failure modes: **single NVMe (OS + data, no redundancy)**, **silent
VPN kill switch**, **root disk filling**, and **unsafe shutdown**. One item per session —
finish, verify, mark done, move on.
 
Reminders that apply to every item below:
- **ARM64 only.** Confirm `linux/arm64` support for any new image before deploying.
- **Reach services via Tailscale** (`100.115.36.108`), not the LAN IP.
- **Apps behind the VPN are addressed via the `gluetun` container**, not their own names.
- Prefer changes that keep an escape hatch open; verify load-bearing facts before committing.
Status key: `[ ]` not started · `[~]` in progress · `[x]` done
 
---
 
## [x] 1. Off-box config backups — DONE (2026-06-17)
 
**Targets:** single-disk total loss / corruption.
Built as **restic → rclone → Google Drive** (chose Drive over the original "another machine
over Tailscale" idea — cloud is off-box and needs no second always-on host). Encrypted repo,
own published OAuth client, least-privilege `drive.file` scope. A root systemd timer fires
nightly at 04:00, quiescing the app stack (gluetun left up) for a consistent snapshot of
`/opt/arr` + the Portainer volume — which captures the stack definitions and the inline
gluetun WireGuard secret. Retention 7 daily / 4 weekly / 12 monthly. **Restore verified
twice** — to a scratch path (compose byte-identical) and over live (test markers reverted,
apps healthy). Full config in INVENTORY.md → **Backups**.
 
Two honest loose ends, carried forward:
- **Unattended but silent.** The job logs failures loudly to the journal, but nothing
  *pushes* a failure to you yet — that notification gap is item #2, the natural partner.
- **Bare metal unrehearsed.** Restores so far ran on an otherwise-intact box. Zero-to-restored
  on a fresh device (nothing but the repo + the two passwords) is the eventual final drill.
## [x] 2. Enable existing alerting + add uptime monitoring — DONE (2026-06-18)
 
**Targets:** silent failures; drive (the SPOF) degrading unnoticed.
Both halves built and verified by test email.
- **Scrutiny notifications** now active — email via Gmail SMTP (Scrutiny shells out to
  shoutrrr). `fromName=pi-arr` so alerts self-identify per box; `+piarr` plus-address for inbox
  filtering. Sensitivity left at its most-aggressive setting (Dashboard Settings → **Device
  Status Threshold = Both**, **Notify Filter Attributes = All**) for earliest warning on the
  SPOF. NB: the old `notify.level` yaml key is **removed** in this image — it now lives in the
  UI, and a stray `level:` key fails validation and restart-loops the container.
- **Uptime Kuma** (`louislam/uptime-kuma:2`) deployed in the `monitoring` stack on port 3001;
  SQLite backend. HTTP monitors for the seven web-facing services, all addressed via the
  **Tailscale host IP** (`100.115.36.108:<port>`) — survives the move and is also how the
  VPN-gated apps are reached (published on the host by gluetun). Per-monitor retries set to 2
  to ride out blips. Email notifications wired to the same Gmail (its own SMTP form, not
  shoutrrr — so no URL-encoding). FlareSolverr deliberately not monitored (internal-only).
- Full config + gotchas in INVENTORY.md → **Stack: `monitoring`** (Scrutiny notifications,
  Uptime Kuma) and **API keys / auth map**.
One loose end carried forward — **resolved in #8 (2026-06-19):**
- **gluetun has no dedicated monitor.** The five VPN-gated apps all share its namespace, so a
  gluetun drop surfaces as all five going red at once — a recognizable signature, not five
  separate faults, but not the same as watching gluetun directly. A direct check was originally
  scoped as "publish the control port" (a deliberate media-stack change), but #8 took the
  lower-risk namespace-curl route instead — a Kuma Push monitor that tests real egress without
  touching the stack. See #8.
## [x] 3. Docker log rotation + free-space alert — DONE (2026-06-19)
 
**Targets:** root partition filling and taking the OS down.
Both halves built and verified (log caps by `docker inspect`; the alert by a real fill-test
across both thresholds on genuine `df` output).
 
- **Log rotation — per-service `logging:`, not `daemon.json` (Option B).** A shared YAML
  anchor (`x-logging: &default-logging` → json-file, `max-size: 10m`, `max-file: 3`) applied
  to every service in both stacks, plus per-container `--log-opt` flags on standalone
  Portainer. All 13 containers now cap at 10 MB × 3 = 30 MB each (~390 MB worst case, trivial
  on 469 G). Chose compose-level over `daemon.json` because: `daemon.json` log-opts need a full
  daemon restart (which force-cycles gluetun and isn't ordered by `depends_on`) and don't apply
  retroactively; and the per-service blocks live in the backed-up compose, so the caps survive
  a bare-metal restore. gluetun was recreated as part of the media-stack redeploy — attended,
  Compose-orchestrated (so `depends_on: service_healthy` held), kill switch verified intact
  (exit IP still AirVPN) afterward.
- **Free-space alert — dual-layer, one host script.** `/usr/local/sbin/pi-arr-disk-alert.sh`
  (root, 700) on a 15-min systemd timer measures free space on `/` (single filesystem; OS +
  all data). **WARN ≤ 90 G** (~19% free) → push `down` to a Kuma Push monitor. **CRITICAL
  ≤ 35 G** (~7.5% free) → Kuma `down` **plus** a host email via **msmtp** (Docker-independent —
  the backstop that still fires when Docker is wedged). Thresholds set conservatively for the
  **QLC** drive (small DRAM-less QLC degrades in speed/endurance as it fills, so keep extra
  headroom). Full config in INVENTORY.md → **Low-disk-space alert** and **Logging / log
  rotation**.
Notes / gotchas banked (see INVENTORY.md):
- The Kuma Push monitor's heartbeat interval (1800 s) is deliberately **2× the timer cadence**
  (15 min) — long enough not to flap on a single missed run, short enough that a *dead script*
  flips the monitor `Down` so the alerter's own failure surfaces.
- The alert service must **not** depend on `docker.service` — that would defeat the
  Docker-independent msmtp backstop.
- Discovered en route: the **host** timezone was `Europe/London`, not `Europe/Athens` like the
  stack — corrected with `timedatectl set-timezone Europe/Athens` (instant was always correct;
  only display/stamping was off). Also: at the time of #3, qBittorrent had no in-app disk
  guard, so this `df` alert was the only line. Resolved in #7 (2026-06-19): **pre-allocation**
  is now the in-app complement — but note it's a fail-fast-at-overflow backstop, **not** a
  free-space threshold (no such setting exists in qBittorrent). So the `df` alert remains the
  trend/headroom detector; pre-allocation is the clean-failure backstop at the moment a grab
  won't fit. Two complementary lines, not one guarding the other.
## [ ] 4. Graceful shutdown on power loss (UPS + NUT)
 
**Targets:** unsafe mid-write shutdown — the main avoidable risk on a single-disk box.
- Small consumer UPS + Network UPS Tools (`nut`, apt) so the Pi detects "on battery" and
  runs a clean `poweroff` before dying.
- Only hardware item on this list.
## [ ] 5. Image-update notifications (not auto-apply)
 
**Targets:** surprise breakage from `:latest` pulls.
- **Diun** notifies on new image digests so updates are applied deliberately (verify its
  arm64 tag at deploy).
- **Avoid Watchtower-style auto-update** here — silent updates to gluetun or the
  shared-namespace apps can take a VPN-gated stack down unattended.
- Alternative / stronger: pin services to specific version tags.
## [ ] 6. Auto-recovery (autoheal) — with a caveat
 
**Targets:** containers stuck in an unhealthy state.
- `willfarrell/autoheal` restarts unhealthy containers.
- **Caveat:** the arr apps share gluetun's network namespace. If gluetun is recreated,
  those apps lose their network and must be restarted too — naïve autoheal on gluetun can
  strand them and make things worse. Only deploy with that dependency handled. Later step,
  not a quick win.
## [x] 7. qBittorrent download hygiene + disk-usage prevention — DONE (2026-06-19)
 
**Targets:** the root disk filling — *prevention* (cause) to complement #3's alert (detection).
qBittorrent is the only unbounded data generator on the box; everything else is now capped.
There is **no native "max total disk" knob** in qBittorrent (open upstream FRs, unimplemented),
and the separate-partition trick that *would* give a hard ceiling is ruled out because it breaks
the same-filesystem hardlink design (imports would become copy+delete, double space + I/O). So
the levers are behavioural, plus one in-app fail-fast guard. All built and verified:
- **Seeding limits (set 2026-06-19):** remove torrent **and its files** at **ratio 2** OR
  **2 weeks** (20160 min) total seeding, whichever first. Good for reclaiming watched-and-deleted
  titles whose torrent was still holding the data via its `/data/torrents` link.
- **Edge case to respect:** a torrent added *manually and never imported* lives **only** in
  `/data/torrents` (no `/data/media` hardlink), so removal deletes its sole copy. Preloads that
  go through Sonarr/Radarr are safe (the library copy survives); direct grab-and-watch has a
  2-week margin.
- **Private trackers:** check their minimum seed-time/ratio rules against ratio 2 / 2 weeks to
  avoid hit-and-run penalties.
- **Hardlinking — VERIFIED (2026-06-19):** an imported title shows the **same inode** with
  **link count 2** across `/data/torrents` and `/data/media`. Checked with
  `find /data/media -links +1` (found two 2-link media files) then
  `find /data/torrents -inum <inode>` (each inode's partner sits under `/data/torrents`). One
  physical file, two names — so the seeding-limit "remove + files" drops the torrent's link and
  the library copy survives. The whole safety argument above now rests on confirmed ground.
- **Pre-allocation — ENABLED (2026-06-19), the in-app disk-full guard.** Options → Downloads →
  "Pre-allocate disk space for all files" = ON. Reserves a torrent's full size at add-time, so
  an oversized grab onto a too-full disk **fails cleanly at the start** rather than dying
  mid-write — the worst failure mode on a single QLC disk. On `ext4` (confirmed) this is a
  `fallocate` extent reservation, **not** a zero-fill, so **no extra QLC wear**. Verified: adding
  a 2.9 GiB torrent moved `df` Used by the full size immediately.
  - **Correction banked:** there is **no** "pause when free space below X" *threshold* setting in
    qBittorrent — that's a long-open, unimplemented upstream FR (#10098 / #10645 / #23572), not a
    box we'd left unticked. Earlier notes (incl. PROJECT_NOTES) calling it a "low-space pause
    guard" were wrong. Pre-allocation is fail-fast-*at-overflow*, which **complements** #3's `df`
    alert (the trend/headroom detector) rather than being the threshold floor once imagined. The
    two together are the prevention story; there is no further in-app knob to add.
- **Why it matters more here:** the drive is small **QLC** (Micron 2500-family, 232-layer QLC,
  DRAM-less/HMB). Sustained writes onto a near-full QLC drive are slow (pSLC cache collapses) and
  wear-heavy — so proactively reclaiming space pays off more than it would on a roomy TLC drive.
## [x] 8. Dedicated gluetun exit monitor — DONE (2026-06-19; reimplemented 2026-06-22)
 
**Update (2026-06-22) — reimplemented; country/leak assertion dropped.** The original
implementation below (a throwaway curl container joined to gluetun's namespace, querying
`ipinfo.io` and asserting the exit country) was **replaced**. AirVPN's shared exit IP
collectively blew ipinfo's unauthenticated rate limit (1,000 req/day, *per IP, shared across all
users on that exit*), so the check kept getting throttled — a 429 with no parseable country —
and false-flagged `down` ("egress OK but no country"). Rather than paper over it with an ipinfo
token, the goal was narrowed to **liveness only**: inside gluetun's kill-switched namespace there
is no "leak to ISP" path to detect anyway, and asserting country needlessly coupled the monitor
to `SERVER_COUNTRIES` (meant to change freely). New implementation: a Kuma **Docker Container
monitor** (`pi-arr gluetun (VPN)`) reads gluetun's own `State.Health.Status` via the Docker
socket — gluetun's healthcheck is itself an active egress test (ICMP/DNS every minute, TCP+TLS
dial every 5 min through the tunnel), so this is a real liveness signal with no external lookup,
nothing to rate-limit, and gluetun untouched. Works because Kuma v2 marks a healthchecked
container `down` when unhealthy (v1 reported it only as *pending* and never alerted). The host
script, its systemd timer/service, the embedded push token, and the pre-pulled `curlimages/curl`
image were all removed. Current config in INVENTORY.md → **Stack: `monitoring`** (Uptime Kuma) and
**VPN exit check — retired**. *(Original 2026-06-19 write-up kept below for the record.)*
 
**Targets:** silent VPN failure / wrong-exit / leak going unnoticed — item #2's carried-forward
loose end *and* the bottom-of-list "VPN traffic/leak check," which turned out to be one job.
A host-level check mirroring the disk-alert machinery, deliberately chosen over the originally
penciled "publish gluetun's control port" route — since gluetun **v3.40.1 the control server
requires auth** (a `config.toml` role), host 8000 collides with Portainer's agent tunnel, and
the change needs a media-stack redeploy that cycles gluetun. The namespace-curl route touches
nothing in the stack and tests *real egress*.
- `/usr/local/sbin/pi-arr-vpn-check.sh` (root, 700) runs a throwaway `curlimages/curl` container
  joined to gluetun's network namespace (`docker run --rm --network container:gluetun …`),
  queries the public IP (`ipinfo.io/json`), and asserts the exit country is in the allow-list
  **{NL, BG, BE, RO, CH}** — mirroring `SERVER_COUNTRIES` (widened to those five on 2026-06-19).
  **Fail-safe:** anything short of a valid IP in an allowed country pushes `down`, with a message
  distinguishing tunnel-down vs ipinfo error vs wrong-country.
- The check runs **inside** the namespace (tests real tunnel egress, fails closed on the kill
  switch); the Kuma push runs over the **host** network so a tunnel drop is still actively
  reported (not just a missed heartbeat).
- systemd timer at **5-min** cadence (`After=docker.service`, **not** `Requires=`) → Kuma Push
  monitor `pi-arr vpn exit`, heartbeat 600 s (2× cadence). **Verified 2026-06-19:** manual run +
  timer both flip the monitor green with `exit OK: NL <ip>`.
- Full config in INVENTORY.md → **VPN exit check** and **Stack: `monitoring`** (Uptime Kuma).
Retired: item #2's "gluetun has no dedicated monitor" loose end, and the bottom-of-list VPN
leak-check bullet.
 
---
 
 
 
- **Healthchecks** on the arr services (gluetun already has one) so monitoring/autoheal
  have real signal.
- **VPN traffic/leak check — addressed via #8, simplified 2026-06-22.** Originally the
  `pi-arr vpn exit` check asserted the exit country from inside gluetun's namespace; reimplemented
  2026-06-22 as a liveness-only Kuma Docker Container monitor reading gluetun's health (see the #8
  update). The kill switch already makes an ISP-leak path impossible *inside* the namespace, so
  tunnel-liveness is the meaningful signal — WireGuard's silence no longer hides a dead tunnel.
  The explicit country/leak assertion was intentionally dropped.