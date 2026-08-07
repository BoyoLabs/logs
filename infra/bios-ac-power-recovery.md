# BIOS AC Power-Loss Recovery for Server Automation
**Date:** 2026-08-06

## Context & Intent

This box runs as an always-on personal server: a headless boot chain already
brings every self-hosted service back up unattended after a reboot, with a
recurring watchdog job that self-heals anything that dies later. That whole
chain assumes the machine actually powers itself back on, though — and by
default, most desktop-class hardware does not do that after a real outage. The
goal was to close that one remaining gap for server automation specifically:
recover on its own after a power loss, without needing anyone to walk over and
press the button, while still respecting an intentional shutdown for
maintenance rather than fighting it back on at the next power blip.

## The Execution

Checked the existing boot-time recovery first, since that was the more likely
gap. It wasn't: the full service stack already comes up unattended and
headless (no GUI session dependency), and a cron-driven watchdog restarts
anything that dies after boot too. That confirmed the only missing piece was
firmware-level — nothing in the OS can make a machine physically power itself
on again after mains power returns; that decision is made entirely by the
motherboard before any OS ever loads.

The board is a recent ASUS AM5 (B850 chipset) model. The standard location for
this setting on ASUS boards — `Advanced → APM Configuration → Restore AC Power
Loss` — doesn't exist on this particular BIOS. Neither ASUS's own support FAQ
nor the printed user manual for this board resolved it: the manual for this
generation has dropped its BIOS reference chapter entirely, and the FAQ only
documents the older menu layout from other board families. ASUS has
reorganized the BIOS menu structure for this line and, as far as could be
found, hasn't published anything reflecting where the setting actually lives
now. Ended up locating it directly in the BIOS itself rather than through any
vendor documentation.

Set to **Last State** (the same setting sometimes labeled "AC BACK" or "Former
State" on raw AMI firmware, depending on how much an OEM has re-skinned it).

## Findings

**Last State is the right answer for the two-sided requirement, and the
mechanism is worth understanding rather than just trusting the label.** The
BIOS doesn't record *why* power was lost — only whether the machine was on or
off at the instant AC power was removed:

- Machine is running, an outage kills power, power comes back → last known
  state was "on" → it boots itself. This is the automated recovery half.
- Machine is deliberately shut down for maintenance first → last known state
  is "off" → it stays off through any further power cycling (a blip, or even
  physically unplugging and replugging it) until someone presses the button.
  This is the maintenance-safety half, and it falls out of the same setting
  for free — no separate switch needed.

**Gotcha found in the process, worth flagging for later:** a clean OS shutdown
also leaves the last-known-state as "off," identically to a maintenance
shutdown. That means this setting can't double as a way to remotely power the
box back on after an intentional shutdown — e.g. cycling power to it via a
smart plug wouldn't work, because as far as the BIOS is concerned nothing
changed about *why* it's off. Wake-on-LAN from another always-on device on the
same network, or a smart relay wired directly to the physical power-button
header (which simulates an actual button press rather than an AC power event),
are the two options that would actually solve *that* problem — a different
one from what this entry covers, and not implemented here.

**Vendor documentation has quietly gotten worse for this.** Both the FAQ and
the current-generation printed manual for this board no longer cover this at
the level of detail they used to for older boards. The BIOS's own built-in
navigation ended up more reliable than the vendor's official docs for finding
a setting that vendor was the one who moved.
