# Network Outage — 2026-05-20

## Summary
Brief WiFi disruption at ~6:24 PM due to the wireless adapter roaming between access points. No DHCP exhaustion or configuration issue on the machine itself.

## Timeline (Local Time, CDT)

| Time | Event |
|------|-------|
| 17:57:42 | CRD client disconnected (may be related to degraded signal ahead of roam) |
| 18:24:32 | WiFi connection to AP `94:a6:7e:4b:d8:14` lost |
| 18:24:33 | `wpa_supplicant` disconnected (reason=4: disassociated due to inactivity) |
| 18:24:40 | Attempting to authenticate with `94:a6:7e:4b:d8:15` (5240 MHz) |
| 18:24:41 | Associated and connected to `94:a6:7e:4b:d8:15` |
| 18:24:41 | WPA key negotiation completed — back online |

**Total downtime**: ~8–9 seconds

## Root Cause
WiFi roaming event between two BSSIDs on the same network ("the real internet"):
- `94:a6:7e:4b:d8:14` → `94:a6:7e:4b:d8:15`

These are two radios on the same router or mesh system (same OUI `94:a6:7e`). The adapter lost the connection to one radio and reconnected to the other. Reason code 4 ("disassociated due to inactivity") typically indicates the AP deauthenticated the client, possibly due to signal handoff logic.

## What It Was NOT
- Not DHCP lease exhaustion — machine held a valid lease (`192.168.50.140`, valid_lft ~24h) throughout
- Not caused by any system configuration changes made during this session
- Not a persistent outage — fully self-resolved in under 10 seconds

## Notes
- Signal strength just before the roam: -53 dBm (marginal)
- Signal after reconnect: -52 dBm
- This same roaming pattern (between these two BSSIDs) was also observed during the May 19 CRD outage incident

## Possible Mitigations
- If roaming is happening too aggressively (or not aggressively enough), tuning the roaming threshold in NetworkManager may help
- Check if the router/mesh firmware has an 802.11r (fast BSS transition) option — enables faster roaming with less disruption
