# Boyo Labs: The Research Logs

*Part of [Boyo Labs](https://www.boyolabstech.com).*

## 📖 Log Index

★ = Featured — a personal favorite among these entries, not a quality/importance ranking of the rest.

| Date | Category | Entry | Summary |
|------|----------|-------|---------|
| 2026-08-19 | Hardware | [ddr5-sodimm-adapter-am5-experiment](hardware/ddr5-sodimm-adapter-am5-experiment.md) | Tried adding cheap SO-DIMM-to-DIMM adapters to reach 32GB by mixing 2 native UDIMM with 2 laptop SO-DIMMs across all 4 slots on an AM5 board — flagged as the highest-risk config going in, and it failed to POST as expected; reverted to the stable native config. |
| 2026-08-13 | Software | ★ [browser-slicer-deep-dive](software/browser-slicer-deep-dive.md) | How a browser upload flow became a full headless slicing pipeline — auto-orientation, infill, tree supports — plus a real research finding: a hand-rolled settings resolver silently falling back to wrong defaults, causing three separate print-quality bugs before the pattern was recognized. |
| 2026-08-10 | Showcase | ★ [boyoapps](showcases/boyoapps/README.md) | A terminal app launcher/scaffolding tool with a built-in curses Python code editor, purpose-built for creating and editing small scripts entirely from an SSH session — source code and reproducible master prompt included. |
| 2026-08-10 | Showcase | ★ [portfolio](showcases/portfolio/README.md) | A plain-text (no curses) terminal dashboard for a personal income-portfolio web app — live totals, braille-rendered charts, positions, 401k tracking, milestones, news, and a heatmap, all typeable from a phone SSH client. |
| 2026-08-06 | Infra | [bios-ac-power-recovery](infra/bios-ac-power-recovery.md) | Configured the BIOS to auto-recover the server after a power outage while still respecting an intentional shutdown for maintenance — one setting handles both, plus a documentation-gap finding on the vendor's side. |
| 2026-08-06 | Software | ★ [ram-crisis-fleet-vs-single-machine-tco](software/ram-crisis-fleet-vs-single-machine-tco.md) | Rigorous TCO simulation: old-hardware fleet vs. one modern machine during the 2026 DRAM shortage — fleet wins on CapEx *and* compute-per-dollar, but a real crossover point (as soon as ~1.2yr) exists once power draw is counted. |
| 2026-08-05 | AI Research | [ansi-escape-animation-attempt](ai-research/ansi-escape-animation-attempt.md) | Tested whether raw ANSI escape codes smuggled into a chat response could animate a terminal live — negative result, rendering layer strips them. |
| 2026-07-20 | Infra | [boyolabstech-server-architecture](infra/boyolabstech-server-architecture.md) | *(backdated, written 2026-08-06)* Overview of the `*.boyolabstech.com` server: named-tunnel-only ingress, PIN-gated apps, self-healing cron-driven services, and the outage-driven lessons behind the pattern. |
| 2026-05-19 | Hardware | [hyperx-keyboard-issue](hardware/hyperx-keyboard-issue.md) | HyperX keyboard went completely dead (no USB presence at all) after an RGB-control troubleshooting session — suspected firmware wipe. |
