# pi-arr — known issues & investigations
 
Anomalies noticed in passing that aren't urgent enough to derail the task in hand,
but shouldn't be lost. Each item: what was seen, why it's odd, and where to start.
Pick one up when there's a session for it — don't let them sidetrack roadmap work
that's already in progress.
 
Status key: `[ ]` open · `[~]` investigating · `[x]` resolved (note how)
 
---
 
## [x] 1. Jellyfin transcode cache at 5.9 GB on a direct-play box — RESOLVED (2026-06-19): benign (audio remux), not video transcode
 
**Resolution (2026-06-19):** Diagnosed from the ffmpeg transcode logs (ground truth —
each log opens with the full ffmpeg command line, which states the operation outright).
Read all six surviving `FFmpeg.*.log` files under `/opt/arr/jellyfin/log/`. **All six the
same shape: `-codec:v:0 copy` (video copied, NOT re-encoded), `-map -0:s` (subtitles
excluded), no burn-in filter (`subtitles=` / `overlay` / `hwupload`) anywhere.** The
expensive operations the box can't afford — video re-encode, subtitle burn-in — did NOT
happen on any logged cache contributor.
 
What the cache actually is: HLS fMP4 segments from **video-copy + audio-remux**
DirectStreams. The trigger is **source audio the client can't passthrough** — the dominant
contributor was the long Amélie (2001) playback whose default track is **French DTS 5.1**,
which Jellyfin converts to stereo AAC (`-codec:a:0 libfdk_aac -ac 2`). That audio-only
re-encode forces the title out of pure direct-play into DirectStream and spills copied-video
segments into `transcodes/`. Audio remux is trivial CPU work on the Pi 5 — exactly the cheap
operation we're fine with. 5.9 GB was a few long playbacks accumulating, not heavy work.
 
**Correction to the original hypothesis:** the predicted "likeliest culprit," subtitle
burn-in, was **NOT** what happened — every log explicitly excluded subtitles (`-map -0:s`).
The original "contrary to intent" alarm rested on a false premise (cache size ⇒ video
transcoding). Don't re-chase burn-in for this cache; the evidence said audio-DTS remux.
 
**Caveat (honest scope):** this characterizes the *logged* playbacks only. Jellyfin prunes
transcode logs after a few days, so cache segments older than these six have no surviving
log. No signal of an outlier (six-for-six benign, largest log matches what'd dominate 5.9 GB),
but "no burn-in ever happened" isn't provable from logs that no longer exist.
 
**Cache now safe to clear on its own terms** — it's regenerable HLS working space, not
evidence of a misconfiguration still to diagnose. (`/opt/arr/jellyfin/cache/transcodes` —
already excluded from the restic backup set.) It'll regrow modestly as DTS-source titles get
played; that's expected, not a regression.
 
**Keeping it clean (forward-looking — no server "never transcode video" switch exists;
the server transcodes when a *client* asks or hits a server limit):**
- **Clients on Original / Auto, no bitrate cap.** A client capped below a file's video
  bitrate (Amélie was ~14 Mbit/s video) forces a real video re-encode to fit. This is the
  top lever.
- **Prefer text subtitles; avoid selecting image-based (PGS/VOBSUB) or unsupported styled
  ASS at playback** — selecting one forces burn-in = full video transcode regardless of
  every other setting. Live risk given the Bazarr Greek+English profile.
- **Server (Dashboard → Playback → Transcoding):** leave hardware acceleration **None**
  (Pi 5 has no encoder; a misconfigured HW setting fails worse than CPU). Per-user transcode
  limits exist (Dashboard → Users → Playback) but are a blunt wall — leave off unless
  enforcing a hard policy.
- **Verification that beats all settings:** Dashboard → click an **active playback session**
  during real playback → shows Direct Play / DirectStream / Transcode **and the reason**.
  If "video" ever appears in the transcode reason, that's the moment to chase a client
  setting — it names the cause live, same as the ffmpeg log did.
**Possible (optional) upstream tweak — not a fix:** to reduce even the cheap audio remux,
nudge arr custom formats to prefer releases carrying AC3/E-AC3/AAC over DTS-only. Pure
optimization; current behaviour is harmless.
 
---

## [x] 2. Jellyfin won't prune a deleted title when its library folder goes completely empty — RESOLVED (2026-06-26): empty-folder safety guard; fixed with a `.keep` sentinel

**Seen (2026-06-26):** Amélie (the only movie) was deleted at the filesystem level
(deliberate — consume-and-delete). `ls /data/media/movies` was empty and Radarr showed no
movies, yet Jellyfin still showed Amélie on Home (My Media / Continue Watching / Recently
Added) and its full detail page. The scheduled **Scan Media Library** task had already run
~17 h earlier — *after* the deletion — and had not removed it. So "the next scan will
prune it" was wrong; a scan had already run and the item survived.

**Root cause:** Jellyfin's empty-folder protection. When a library's root folder is
**completely empty**, the scanner skips it and refuses to prune the items that used to live
there — by design, so an unmounted/unavailable volume doesn't wipe the library (losing watch
progress/metadata). "Folder has zero items" is indistinguishable from "the mount fell off,"
so it errs toward keeping the DB rows. Confirmed against current Jellyfin issues/docs, not
just inferred. This is why scan *frequency* is irrelevant: the scan itself declines to touch
an empty folder regardless of cadence.

Why it bites this box specifically: the consume-and-delete model regularly drains a library
to **zero** items. A library with other titles still in it prunes a single deletion fine
(root isn't empty); the guard triggers only on drain-to-empty — exactly the recurring
end-state here.

**Fix (applied):** a permanent sentinel file at each library root so the folder is **never**
completely empty:

---
 
<details>
<summary>Original investigation note (kept for the record — hypothesis was wrong, see resolution above)</summary>
**Seen:** 2026-06-16, while sizing `/opt/arr` for off-box config backups.
`du` showed `/opt/arr/jellyfin/cache/transcodes` = **5.9 GB** — essentially the
entire Jellyfin config dir (everything else under it was ~15 MB). Excluded from the
backup, so this is purely a local-disk / behaviour question, not a backup one.
 
**Why it's odd:** this box is set up for **direct play** — clients on Original
quality, and the Pi 5 has **no hardware video encoder**, so any transcoding is slow
CPU work we explicitly want to avoid. A transcode cache that large means a
non-trivial amount of transcoding has happened, contrary to intent.
 
**Candidate causes (check, don't assume):**
- **Subtitle burn-in** — likeliest culprit on an otherwise direct-play setup.
  Image-based subtitles (PGS / VOBSUB) can't be sent as text and force a full video
  transcode to burn them in. The Bazarr profile pulls Greek + English subs, so this
  is the first thing to look at.
- A client (or app) not actually set to Original / direct-play, quietly requesting
  transcodes.
- A specific container / video codec / audio format some client can't direct-play.
- Stale temp from interrupted transcodes (cache not cleared after a crash).
**Where to start:** Jellyfin Dashboard → watch active sessions during playback and
read the "Playback Info" transcode reason. If it cites subtitles, that confirms
burn-in. The cache is safe to clear (it regenerates), but clearing it before finding
the cause just defers the question — find the *why* first.
 
</details>
 
