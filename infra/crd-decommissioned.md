# Chrome Remote Desktop Decommissioned
**Date:** 2026-08-06

## Context & Intent

Chrome Remote Desktop (CRD) was this box's original GUI access path (see the
earlier `crd-outage` and `crd-autostart-fix` entries in this folder). Once
`rdp.boyolabstech.com` — a self-built WebRTC desktop stream over the box's own
Cloudflare Tunnel — matured into a genuine replacement, the question came up:
does `rdp.boyolabstech.com` even depend on CRD, or are they independent? They
turned out to be fully independent (different display server, different
underlying tech), which meant CRD was no longer doing anything the newer
service didn't already cover, and it was retired.

## The Execution

1. **Verified CRD wasn't even running.** Before touching anything, checked
   live process state and CRD's own logs. Its most recent activity was a
   clean shutdown on **2026-07-17** — it had already dropped out of the boot
   sequence (no entry left in `/etc/rc.local` or cron) roughly three weeks
   before this cleanup, unnoticed, because `rdp.boyolabstech.com` had already
   taken over in practice. This was cleanup of an already-dead path, not a
   live cutover.

2. **Removed the package.** `chrome-remote-desktop` was installed as a real
   `apt`/`dpkg` package. Its pre-removal script unconditionally calls
   `systemctl stop`, which fails hard on this box since it doesn't run
   systemd (same category of gotcha documented elsewhere for `ollama` and
   `cloudflared`, which is exactly why those are installed as standalone
   binaries instead). Worked around by patching the packaged `prerm` script
   in `/var/lib/dpkg/info/` to tolerate the failed `systemctl` call
   (`|| true`) rather than aborting the whole removal, then purged normally.
   Left-over backup binaries from a past manual edit were also cleaned out
   of `/opt/google/chrome-remote-desktop`.

3. **Removed the per-user config and leftover scripts/logs** — the CRD host
   config, its old startup script, and its boot/manual logs, none of which
   are managed by `apt` since they live under the user's home directory.

## Findings

**A real mistake worth recording, not smoothing over:** reading the CRD host
config to check its contents printed its private key and OAuth refresh token
in plaintext into a terminal session. Nothing was committed or pushed
anywhere — it stayed local to that one session — but it's exactly the kind of
thing this repo's own security-sweep practice exists to catch before it
becomes a real leak. The token is moot now that the host itself is
deregistered/removed, but the process lesson stands: config files that might
hold credentials should be listed/grepped for structure first, not `cat`'d
wholesale, especially on a box that's already been bitten once by secrets
sitting in plaintext files (see this repo's security-sweep note in
`standing-documentation-practice`-adjacent practice).

**Second finding, unrelated to the mistake above:** CRD's architecture turned
out to be genuinely independent of `rdp.boyolabstech.com` — CRD ran its own
separate virtual display, not the physical SDDM/Xorg session the newer
service captures. That independence was real while CRD was alive, but stopped
mattering in practice three weeks before anyone noticed, which says more
about how naturally the newer service replaced it than about the old
service's design.
