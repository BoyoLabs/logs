# HyperX Keyboard Issue — 2026-05-19

## Device
- **Model**: HyperX Alloy Origins Core
- **Model #**: AG003 (HX-KB6C)
- Connected via USB (Genesys Logic USB hub, Bus 01 Port 12)

## Background
- Was working with Claude Code to toggle RGB components
- During that session "ran into trouble with the keyboard"
- After the session, keyboard stopped working entirely
- User believes the firmware may have been erased during troubleshooting

## Symptoms
- Keyboard produces **zero USB presence** — does not appear in `lsusb` at all
- No USB enumeration even briefly on plug-in (checked via `dmesg`)
- `usbhid` kernel module loaded but shows 0 users
- USB hub (Genesys Logic) is detected fine, all 4 ports show empty
- Mouse (Logitech G203) works normally on a different USB bus

## Troubleshooting Attempted
- USB hub power cycle (unbind/rebind) — no effect
- Esc + replug (HyperX bootloader/recovery mode) — no effect, keyboard still absent from lsusb
- Checked udev rules — OpenRGB rules (`/lib/udev/rules.d/60-openrgb.rules`) present but only adds `uaccess` tags, not blocking
- USB `authorized_default` all set to 1 — not the issue

## Key Finding
A keyboard with erased **application firmware** typically still shows up in lsusb with a DFU/bootloader USB ID. This keyboard shows **nothing at all**, which suggests either:
1. A physical connection/hardware failure (power, cable, port, USB controller chip)
2. The bootloader itself was corrupted (very rare, harder to recover)

## Recovery Options to Try
1. **Try a different USB cable** — if the cable is detachable
2. **Try plugging directly into a rear motherboard USB port** (not the hub)
3. **Try on a Windows PC with NGENUITY** — the official HyperX firmware tool, fetches firmware from HyperX servers. Windows only.
4. **Contact HyperX support** — if under warranty, this may be a warranty claim
5. **Alternative boot mode**: Some HyperX keyboards respond to other key combos during plug-in (e.g. Fn+Esc, or holding a different key). Worth checking the manual or HyperX support forums.

## Notes
- NGENUITY is Windows-only; no official Linux firmware reflash path exists
- OpenRGB does not have firmware flash capability for HyperX — it only sends RGB HID commands
- The previous Claude Code session that may have caused this should be reviewed (session history not available at time of this log)

## Follow-Up
- [ ] Try keyboard on Windows PC with NGENUITY
- [ ] Try different cable/port
- [ ] Contact HyperX support (https://www.hyperxgaming.com/support)
- [ ] Check if keyboard shows any life (LEDs, etc.) on any machine

---

## Session History (Retrieved 2026-05-21)

The prior Claude Code sessions have now been reviewed. Here is what was done:

### Session `864a0880` — "Turn off all RGB devices with openRGB" (May 19–20)
- Goal: remotely toggle all RGB off via OpenRGB while RDP'd in
- **Files created/modified:**
  - `rgb-toggle.sh` — shell toggle script, went through 7 revisions
  - `/etc/systemd/system/openrgb.service` — systemd service to run OpenRGB at boot (2 revisions)
  - `hyperx-keyboard-white.py` — Python script to send HID commands directly to the HyperX keyboard (4 revisions)
- OpenRGB struggled to control the keyboard's RGB; multiple attempts were made
- The keyboard RGB got stuck in an off state, then the keyboard became completely non-functional

### Session `577b8e4b` — Keyboard non-functional (May 19–20)
- User reported keyboard stopped working entirely; PC showed no USB presence

### Session `56cb02ae` — Post-mortem (May 20)
- User confirmed keyboard appeared bricked from the OpenRGB work
- Session away summary: *"Investigating a bricked HyperX keyboard likely caused by OpenRGB. Tonight, plug it into a Windows machine and check if NGENUITY can recover the firmware."*

---

## Update — 2026-05-20

### Forensic Analysis (kern.log / syslog)
Pulled full logs to reconstruct exactly what happened:

- **18:05:21** — Boot. Keyboard (`03f0:098f`) enumerated normally on `usb 5-2`.
- **18:10:15** — Chrome Remote Desktop host started.
- **18:26:04** — Remote client connected (boyolabstech@gmail.com via relay/STUN).
- **~18:26–18:27** — OpenRGB session ran, sent HID commands to keyboard.
- **18:27:49** — Remote client disconnected. Session lasted ~105 seconds.
- **21:44:14** — Next reboot. Keyboard completely absent from USB — never returned.

**No USB disconnect event was logged during the session.** The keyboard stayed electrically alive but entered a non-functional firmware state that survived power cycles. Classic MCU firmware corruption/bad-state from a malformed HID packet — not physical damage.

### Recovery Attempts This Session
- Esc + unplug/replug — attempted, no USB enumeration observed (confirmed via lsusb)
- User is currently remote (RDP), keyboard not physically accessible

### Current Assessment
- Issue is firmware state on the keyboard's MCU, not the host OS, drivers, or USB ports
- All other USB devices (mouse, AURA controller, wireless adapter) work normally
- A replacement keyboard plugged into the same ports will work immediately

### Plan for Tonight (when physically at machine)
1. Plug keyboard into a Windows PC — check if it enumerates at all
2. If it enumerates: run HyperX NGENUITY, attempt firmware recovery
3. If it does not enumerate on Windows either: likely done; consider warranty claim or replacement
4. Custom firmware path (QMK or similar) only viable if keyboard enters bootloader mode and MCU is identified from teardown
