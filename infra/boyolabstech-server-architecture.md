# *.boyolabstech.com — Server Architecture Overview
**Date:** 2026-07-20 (backdated — written 2026-08-06, see note below)

## Context & Intent

This entry is deliberately backdated. The bulk of the `*.boyolabstech.com`
infrastructure — the tunnel architecture, the self-healing service pattern, most
of the individual apps — was built between roughly 2026-07-20 and early August,
before [the decision to make documentation a default habit](../standing-documentation-practice.md) made writing this stuff down a default
habit rather than something requested case by case. Boyo Labs has since settled
on research logs as effectively its actual output — not a sales pitch, a running
record of what got built and why — and a body of infrastructure work that
predates that habit was sitting undocumented. This entry closes that gap: a
single overview of the server architecture as it stands, dated to when the
underlying work actually happened rather than when it was finally written up.

In the interest of not handing anyone a map of a live, internet-facing personal
server, this entry intentionally omits internal ports, tunnel identifiers,
software version numbers, LAN/IP details, auth secrets, and physical location —
none of that changes the architecture story, and all of it is either irrelevant
to an outside reader or actively useful only to someone trying to poke at the
box.

## The Execution

**Ingress model.** `boyolabstech.com` is a single personal Debian-ish box running
under one unprivileged user, exposing a handful of small Flask apps to the
internet through **named Cloudflare Tunnels** — one tunnel per subdomain, each
proxying to a Flask app that only ever binds to `localhost`. Nothing on the box
binds to a public interface directly; the tunnel daemon is the sole ingress
path. There is no port-forwarding, no public IP exposure, and no direct-connect
fallback if the tunnel software itself is down — which is a deliberate
trade-off (smaller attack surface) rather than an oversight.

**Service inventory.** Roughly a dozen subdomains are live at any given time,
covering: a public homepage/dashboard, an income-portfolio tracker, a 3D-print
queue/upload frontend, a browser-based shell, a WebRTC remote-desktop stream, a
personal file server, a local image-generation tool, and a public read-only
education transcript. Two of these (the homepage and the education transcript)
are intentionally public with no login at all; everything else sits behind a
custom PIN-and-session gate — a numeric pad built to behave well on mobile
rather than the browser's native Basic Auth popup — since they expose personal
data (finances, files, a shell) rather than being meant for outside visitors.

**Self-healing over systemd.** This box doesn't run systemd, so every service
is started by a small shell script (`start-<service>.sh`) launched at boot, each
one idempotent (checks whether its process is already running before doing
anything) so the same script can safely be re-invoked by a recurring cron job
without spawning duplicates. A separate watchdog process restarts any tunnel
whose local process has died. This pattern exists because it was tested by
failure: one service that hadn't yet been given a recurring watchdog died
silently overnight and stayed down for roughly 13 hours before anyone noticed —
after which every service on the box was audited and given the same coverage.

**Ownership discipline.** Everything runs as the unprivileged user, never root.
Editing or running any app's files as root — even briefly, during a manual fix
— leaves root-owned files behind that the normal user's cron-launched process
can no longer write to, silently breaking it. The fix that stuck: any script
that detects it was accidentally invoked as root re-asserts correct ownership
on its own files before doing anything else, then re-executes itself as the
correct user.

**Local-only supporting services.** A handful of processes back the public apps
without ever being exposed themselves: 3D-printer control software, a webcam
streamer, a local LLM used by one app for auto-summarization, and a WebRTC relay
used only when two devices on the same home network need to negotiate a
connection. None of these are reachable from outside the LAN.

**Backups.** A nightly job snapshots the box's application data and config to a
local archive; a separate scheduled job encrypts the latest snapshot and emails
it to the owner's own inbox, so a "known good state" exists off-box without
standing up separate backup infrastructure.

## Findings

The recurring lesson across almost every piece of this, in one form or another:
**default to the boring, resilient option.** Tunnel-only ingress instead of any
direct exposure. Idempotent polling scripts instead of a process supervisor the
box doesn't have. Config-driven auto-discovery (the homepage reads tunnel
config files directly rather than maintaining its own hardcoded service list)
instead of a list that quietly drifts out of date. None of these are exotic —
they're the same shape of decision made repeatedly: whatever requires the least
ongoing attention to stay correct wins, because the actual maintainer is one
person checking in occasionally, not an ops team.

The other throughline is that most outages here were caused by *missing*
resilience in one specific spot rather than a design flaw in the overall
pattern — a service without a watchdog, a script edited as root, a tunnel that
died with nothing set up to notice. The architecture itself has held up; the
gaps have consistently been "this one thing hadn't been given the same
treatment as everything else yet."
