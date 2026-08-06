# Elegoo Neptune 3 Pro — LED Control via USB

**Date:** 2026-05-23  
**Device:** Elegoo Neptune 3 Pro 3D Printer  
**Connection:** USB serial cable → `/dev/ttyUSB0`  
**Baud Rate:** 115200  
**Firmware:** Marlin (MKS Robin Nano board)

---

## How It Works

The printer exposes a serial interface over USB. G-code commands are sent as plain text lines (newline-terminated) using any serial tool (Python `pyserial`, `screen`, `minicom`, etc.).

The LED strip is controlled with the **M355** G-code command.

---

## G-Code Commands

| Action            | Command       | Notes                          |
|-------------------|---------------|--------------------------------|
| Turn on (full)    | `M355 S1`     | Sets brightness to 255         |
| Turn off          | `M355 S0`     | Confirmed response: `Case light: OFF` |
| Set brightness    | `M355 P<val>` | `val` = 0–255 (e.g. `P80` ≈ 31%) |

**Note:** `S` is the on/off toggle. `P` sets the brightness level independently. `M42` (pin control) returned `Unknown command` on this firmware — use `M355` only.

---

## Python Snippet

```python
import serial
import time

def send_gcode(port='/dev/ttyUSB0', baud=115200, command='M355 S1'):
    with serial.Serial(port, baud, timeout=2) as ser:
        time.sleep(2)  # wait for printer ready
        ser.flushInput()
        ser.write(f'{command}\n'.encode())
        time.sleep(0.5)
        return ser.read_all().decode(errors='ignore').strip()

# Examples
print(send_gcode(command='M355 S1'))      # full brightness on
print(send_gcode(command='M355 P80'))     # dim to ~31%
print(send_gcode(command='M355 S0'))      # off
```

---

## Notes for AI Automation

- Always include a `time.sleep(2)` after opening the serial port — the printer resets on connect and needs a moment.
- Flush input before sending to avoid reading stale responses.
- The printer echoes back the result: `echo:Case light: 255`, `echo:Case light: 80`, or `echo:Case light: OFF` — parse this to confirm the command was applied.
- Good webcam dimming range: **P60–P100** depending on ambient light.
