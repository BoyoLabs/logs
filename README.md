# Boyo Labs: The Research Logs

## 📖 Log Index

★ = Featured — a personal favorite among these entries, not a quality/importance ranking of the rest.

| Date | Category | Entry | Summary |
|------|----------|-------|---------|
| 2026-08-10 | Showcase | ★ [boyoapps](showcases/boyoapps/README.md) | A terminal app launcher/scaffolding tool with a built-in curses Python code editor, purpose-built for creating and editing small scripts entirely from an SSH session — source code and reproducible master prompt included. |
| 2026-08-10 | Showcase | ★ [portfolio](showcases/portfolio/README.md) | A plain-text (no curses) terminal dashboard for a personal income-portfolio web app — live totals, braille-rendered charts, positions, 401k tracking, milestones, news, and a heatmap, all typeable from a phone SSH client. |
| 2026-08-06 | Infra | [bios-ac-power-recovery](infra/bios-ac-power-recovery.md) | Configured the BIOS to auto-recover the server after a power outage while still respecting an intentional shutdown for maintenance — one setting handles both, plus a documentation-gap finding on the vendor's side. |
| 2026-08-06 | Software | ★ [ram-crisis-fleet-vs-single-machine-tco](software/ram-crisis-fleet-vs-single-machine-tco.md) | Rigorous TCO simulation: old-hardware fleet vs. one modern machine during the 2026 DRAM shortage — fleet wins on CapEx *and* compute-per-dollar, but a real crossover point (as soon as ~1.2yr) exists once power draw is counted. |
| 2026-08-06 | Infra | [crd-decommissioned](infra/crd-decommissioned.md) | Chrome Remote Desktop retired once another existing remote-access service fully replaced it — turned out CRD had already silently stopped running weeks earlier. Includes a real security-lesson finding. |
| 2026-08-05 | — | [standing-documentation-practice](standing-documentation-practice.md) | The decision to make logging CS-related work a default, proactive habit rather than something requested case by case. |
| 2026-08-05 | Showcase | [edu](showcases/edu/README.md) | Self-directed continuing-education system with a public, tamper-evident transcript — entry includes the reproducible master prompt used to build it. |
| 2026-08-05 | AI Research | [ansi-escape-animation-attempt](ai-research/ansi-escape-animation-attempt.md) | Tested whether raw ANSI escape codes smuggled into a chat response could animate a terminal live — negative result, rendering layer strips them. |
| 2026-07-20 | Infra | [boyolabstech-server-architecture](infra/boyolabstech-server-architecture.md) | *(backdated, written 2026-08-06)* Overview of the `*.boyolabstech.com` server: named-tunnel-only ingress, PIN-gated apps, self-healing cron-driven services, and the outage-driven lessons behind the pattern. |
| 2026-06-10 | Software | [osrs-steam-crash-fix](software/osrs-steam-crash-fix.md) | OSRS wouldn't launch via Steam — traced to a blocking interstitial dialog stuck waiting for input. |
| 2026-05-28 | Infra | [robinhood-agent-setup](infra/robinhood-agent-setup.md) | Stood up an MCP-based agentic trading tool suite against a live Robinhood account. |
| 2026-05-26 | AI Research | [markdown-parsing-collisions](ai-research/markdown-parsing-collisions.md) | Why raw, nested markdown code blocks get corrupted when copy-pasted out of an AI web interface's own markdown renderer. |
| 2026-05-25 | Software | [zero-infrastructure-server-mesh-design](software/zero-infrastructure-server-mesh-design.md) | Theoretical architecture for a device-independent personal server, built entirely on GitHub + browser runtimes instead of local infra. |
| 2026-05-23 | Hardware | [neptune-3-pro-led-control](hardware/neptune-3-pro-led-control.md) | Controlling the Elegoo Neptune 3 Pro's LEDs directly over USB serial. |
| 2026-05-22 | Hardware | [gpu-usage-research](hardware/gpu-usage-research.md) | Mapped which GPU (Intel Arc B580 vs. AMD integrated) actually handles rendering during an RDP session. |
| 2026-05-20 | Software | [webcam-utility](software/webcam-utility.md) | Built a single-file HTML utility for webcam monitoring, photo capture, and video recording. |
| 2026-05-20 | Infra | [network-outage](infra/network-outage.md) | Brief WiFi drop from the adapter roaming between access points — not a DHCP or config issue. |
| 2026-05-20 | Infra | [crd-outage](infra/crd-outage.md) | ~30-minute CRD outage traced to a service restart plus WiFi instability while roaming between access points. |
| 2026-05-20 | Infra | [crd-autostart-fix](infra/crd-autostart-fix.md) | Chrome Remote Desktop wasn't reliably starting on boot, leaving the machine unreachable for 5–10+ minutes after a reboot; fixed. |
| 2026-05-20 | Hardware | [camera-issue](hardware/camera-issue.md) | Webcam that previously worked stopped functioning; investigation into the cause. |
| 2026-05-19 | Hardware | [hyperx-keyboard-issue](hardware/hyperx-keyboard-issue.md) | HyperX keyboard went completely dead (no USB presence at all) after an RGB-control troubleshooting session — suspected firmware wipe. |
