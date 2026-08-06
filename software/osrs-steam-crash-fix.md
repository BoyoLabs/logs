# OSRS / Steam Crash Fix — 2026-06-10

## Issue
Old School RuneScape (Steam, AppID 1343370) was not launching. User reported it "keeps crashing."

## Root Cause
Steam was stuck in a `ShowInterstitials` state — a dialog requiring user interaction (age gate, EULA, or update notice) had appeared on screen and was blocking the game from launching. Multiple redundant launch attempts had also queued up, compounding the stuck state.

Relevant log entry from `~/.local/share/Steam/logs/console_log.txt`:
```
[2026-06-10 08:15:41] GameAction [AppID 1343370, ActionID 2] : LaunchApp waiting for user response to ShowInterstitials ""
```

## Resolution
Shut down Steam gracefully via `steam -shutdown`, then killed remaining processes:
```
sudo -u boyo DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus steam -shutdown
pkill -u boyo -f steam
```
All Steam and OSRS processes confirmed terminated. User relaunched from desktop shortcut successfully.

## Notes
- Game runs via Proton Experimental (11.0-100) since it's a Windows client (`osclient.exe`)
- One prior crash dump found from 2026-06-09 16:22 (`assert_20260609162240_8.dmp`) — but that was `gameoverlayui`, not OSRS itself
- `mmap() failed: Cannot allocate memory` errors appeared in logs on 2026-06-09 15:25 — worth monitoring if crashes recur
- System RAM at time of investigation: 14 GiB total, ~9 GiB available — not memory-constrained
