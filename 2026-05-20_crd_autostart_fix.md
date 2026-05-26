# CRD Autostart Fix — 2026-05-20

## Problem
Chrome Remote Desktop was not reliably starting on boot. After a reboot, the machine was often unreachable for 5–10+ minutes, and occasionally never came back without manual intervention.

## System Context
- **Init system**: SysV init (not systemd) — MX Linux / Liquorix kernel
- **Display manager**: SDDM with no auto-login configured
- **CRD version**: 148.0.7778.58
- **User**: boyo (UID 1000)

## Root Causes Found

### 1. Missing `XDG_RUNTIME_DIR` at early boot
`/run/user/1000` does not exist until a user session is opened (by PAM/elogind/SDDM). On a headless machine with no auto-login, it was never created before CRD started. The CRD C++ host binary (`libremoting_core.so`) hits DCHECK assertions when `XDG_RUNTIME_DIR` is unset/missing, causing repeated `int3` trap crashes visible in syslog.

Crash pattern observed in syslog (seconds since boot):
- [9s], [13s], [18s], [23s], [28s], [33s], [93s] — CRD crashing and retrying with backoff
- Eventually stabilized after ~8 minutes once the environment settled

### 2. Duplicate startup mechanisms racing against each other
Two separate mechanisms were both trying to start CRD:
- **`@reboot` cron** (as user boyo) — fired ~7–9 seconds after boot, before environment was ready
- **`/etc/rc.local`** — fired later in the boot sequence, after `$all` services

This caused race conditions and the @reboot cron was consistently starting CRD too early.

### 3. `start_crd.sh` missing required environment variables
`HOME`, `USER`, and `XDG_RUNTIME_DIR` were not explicitly set, leaving the CRD process dependent on whatever environment `su` or cron provided.

## Why `--child-process` is Used
`chrome-remote-desktop --start` (without `--child-process`) calls `systemctl start chrome-remote-desktop@boyo`, which fails on a non-systemd system. Using `--start --child-process` bypasses this and runs CRD directly as the daemon. This is the correct workaround for SysV init.

## Changes Made

### `/etc/rc.local`
Added pre-creation of `/run/user/1000` as root before starting CRD, and passes `XDG_RUNTIME_DIR` to the su invocation:
```sh
if [ ! -d /run/user/1000 ]; then
    mkdir -p /run/user/1000
    chown boyo:boyo /run/user/1000
    chmod 700 /run/user/1000
fi
su -s /bin/bash boyo -c 'XDG_RUNTIME_DIR=/run/user/1000 bash /home/boyo/start_crd.sh'
```

### `/home/boyo/start_crd.sh`
Added explicit `HOME`, `USER`, and `XDG_RUNTIME_DIR` exports. Changed log from truncate (`>`) to append (`>>`):
```bash
export HOME=/home/boyo
export USER=boyo
export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/1000}"
```

### Crontab (boyo)
Removed the `@reboot /home/boyo/start_crd.sh` entry. rc.local is now the sole startup mechanism — it runs later in the boot sequence when the system is more stable.

## Expected Behavior After Fix
- rc.local runs near end of boot (after `$all` services)
- `/run/user/1000` is created by root if it doesn't already exist
- CRD starts with a proper environment on first try — no crash/retry loop
- CRD should be reachable within ~1–2 minutes of boot rather than 5–10+

## Follow-Up
- [ ] Reboot and verify CRD comes up cleanly on first attempt
- [ ] Check `/home/boyo/crd-boot-log.txt` after reboot — should not show repeated crash restarts
