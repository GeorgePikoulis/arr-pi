# Project: pi-arr media server — operating notes for future sessions

General guidance for how to help on this project. Not a task list — these are habits
and reflexes that paid off (or would have) during the initial build. Read alongside
`INVENTORY.md`, which holds the current concrete state.

## What this box is

A single Raspberry Pi 5 (aarch64) running a self-hosted media stack in Docker, managed
through Portainer. One NVMe drive holds the OS *and* all data — there is no second disk.
Everything that touches the internet for downloading rides an always-on AirVPN tunnel
(gluetun) with a kill switch. See `INVENTORY.md` for specifics.

## How George likes to work

- **One detail at a time.** He explicitly steers sessions step by step and dislikes
  "running away" with a huge multi-part answer when he asked about one thing. Match that
  pace. Finish the current item, confirm it works, then ask what's next.
- **He reports what he sees.** When he pastes an error, a status icon, or CLI output,
  that's primary evidence — believe it and verify on his terms (ask for the output,
  reproduce, look closer) rather than asserting from priors that he must be mistaken. He
  has usually already checked the obvious thing; don't re-explain it.
- **He values correct over fast.** A slower verified answer beats a confident wrong one,
  especially because his feedback loops cost real cycles on hardware.

## Reflexes that save time here

- **Try a clean restart before a deep dive.** A full stack/container reboot cleared a
  torrent issue that looked like a deep config problem. When something is stuck in a way
  that doesn't match the config, restart the relevant container(s) or stack first — it's
  cheap and rules out stale runtime state before you open a rabbit hole.
- **Diagnose the layer beneath the symptom.** Surface symptoms here repeatedly had a
  root one level down (a "firewalled" torrent was really an interface-binding/DNS issue;
  a "Failed" drive was really a missing capability, then a stale status record). Confirm
  the mechanism is even active before tuning its parameters.
- **A silent no-op is a red flag.** No error + no effect usually means a mismatch one
  level down, not a value that needs nudging.
- **Distinguish "can't read / stale" from "real failure."** Several alarming states were
  tooling artifacts, not real problems. Get the ground-truth source (e.g. the device's
  own self-assessment) before treating an alarm as real.

## Things that are easy to get wrong on this box

- **It's ARM64.** Every image must support `linux/arm64`. Verify arch support for any new
  container before recommending it — don't assume from an amd64 example.
- **The Pi 5 has no hardware video encoder.** Jellyfin transcoding is CPU-only. Steer
  toward direct play (1080p, client-friendly codecs); don't lean on transcoding.
- **Version/arch/current-state facts decay.** Image tags, compose syntax, provider
  options, product details — verify against current sources rather than recalling,
  especially for anything load-bearing. Plausible-looking is not the same as correct.
- **Cost of being wrong is asymmetric.** The VPN kill switch and the single shared disk
  are the high-stakes pieces. Before committing changes that affect either (leak risk,
  filling/corrupting the root disk), verify the load-bearing facts and prefer changes
  that keep an escape hatch open.

## Access & networking

- **Reach everything via the Tailscale IP, not the LAN IP.** The Tailscale address is
  location-independent and survives the planned office→home move (LAN subnet changes from
  10.0.1.0/24 to 10.0.69.0/24). Bookmarks and service URLs should use Tailscale.
- **Apps behind the VPN are reached via the `gluetun` container**, not their own names —
  they share gluetun's network namespace and have no separate network identity.
- **WireGuard is a silent protocol.** gluetun reporting "healthy" does not prove peer
  traffic flows. If torrents misbehave, confirm actual traffic, not just tunnel status.

## Housekeeping

- Prefer graceful shutdowns/reboots (`sudo poweroff`) over pulling power — single NVMe
  holds OS + data, and unsafe shutdowns mid-write are the main avoidable risk.
- Watch free space on the root disk; a runaway season-pack grab or forgotten seeding can
  fill the partition the OS lives on. qBittorrent has a low-space pause guard; Glances and
  Scrutiny show capacity and drive health.

## When updating these files

Keep `INVENTORY.md` factual and current (what's installed, where, which options). Keep
this file about *approach*. If a future fix reveals a new recurring gotcha, add it to the
"easy to get wrong" section rather than burying it in the inventory.