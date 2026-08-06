# Boyo Labs: The Research Logs

> "The log is the laboratory."

This repository serves as the definitive, immutable record of engineering exploration, architectural design, and daily technical execution at Boyo Labs. 

In this environment, we operate under a foundational philosophy: **the logs of what we are doing and working on are the ultimate product.** This repository is our private literature base—preserving the raw context, intellectual property, and thought processes behind every experiment.

---

## 🔬 Core Philosophies

* **The "Literature" Mindset:** Logs are treated like research papers. They detail hypotheses, methodologies, terminal workflows, and conclusions.
* **Preserving Negative Results:** Code that fails or projects that break are never discarded. They are thoroughly documented so we understand *why* they didn't work and retain the underlying logic.
* **Capturing "Dark Matter" Work:** Hours spent wrestling with Bash pipelines, configuring daemons, or tweaking network bridges are given full visibility. If it took mental lifting, it gets logged.

---

## 📂 Repository Structure

The `/logs` directory is a flat, searchable repository of knowledge. Each file is named directly after the topic or concept explored within it.

```text
EXAMPLE: 

├── README.md
└── logs/
    ├── webrtc-mesh-anchor-nodes.md
    ├── bash-automation-and-permissions.md
    ├── docker-network-routing.md
    └── [descriptor-of-the-log].md
```

Filenames are the explanation — no date prefix. The date lives inside the file
itself (title or a `Date:` field near the top), not in the filename.

## 🛠️ Log Elements
To maintain analytical consistency, logs in this repository generally capture:

* Context & Intent: What problem are we solving, or what concept are we exploring?

* The Execution: The raw terminal commands, scripts, and environment setups used during the session.

* The Findings: The data on what worked, what broke, and the core logic discovered along the way.

This repository functions as a searchable, personal knowledge base. Built collaboratively with AI; maintained for the future.

---

## 📖 Log Index

| Date | Entry | Summary |
|------|-------|---------|
| 2026-05-19 | [hyperx-keyboard-issue](hyperx-keyboard-issue.md) | HyperX keyboard went completely dead (no USB presence at all) after an RGB-control troubleshooting session — suspected firmware wipe. |
| 2026-05-20 | [camera-issue](camera-issue.md) | Webcam that previously worked stopped functioning; investigation into the cause. |
| 2026-05-20 | [crd-autostart-fix](crd-autostart-fix.md) | Chrome Remote Desktop wasn't reliably starting on boot, leaving the machine unreachable for 5–10+ minutes after a reboot; fixed. |
| 2026-05-20 | [crd-outage](crd-outage.md) | ~30-minute CRD outage traced to a service restart plus WiFi instability while roaming between access points. |
| 2026-05-20 | [network-outage](network-outage.md) | Brief WiFi drop from the adapter roaming between access points — not a DHCP or config issue. |
| 2026-05-20 | [webcam-utility](webcam-utility.md) | Built a single-file HTML utility for webcam monitoring, photo capture, and video recording. |
| 2026-05-22 | [gpu-usage-research](gpu-usage-research.md) | Mapped which GPU (Intel Arc B580 vs. AMD integrated) actually handles rendering during an RDP session. |
| 2026-05-23 | [neptune-3-pro-led-control](neptune-3-pro-led-control.md) | Controlling the Elegoo Neptune 3 Pro's LEDs directly over USB serial. |
| 2026-05-25 | [zero-infrastructure-server-mesh-design](zero-infrastructure-server-mesh-design.md) | Theoretical architecture for a device-independent personal server, built entirely on GitHub + browser runtimes instead of local infra. |
| 2026-05-26 | [markdown-parsing-collisions](markdown-parsing-collisions.md) | Why raw, nested markdown code blocks get corrupted when copy-pasted out of an AI web interface's own markdown renderer. |
| 2026-05-28 | [robinhood-agent-setup](robinhood-agent-setup.md) | Stood up an MCP-based agentic trading tool suite against a live Robinhood account. |
| 2026-06-10 | [osrs-steam-crash-fix](osrs-steam-crash-fix.md) | OSRS wouldn't launch via Steam — traced to a blocking interstitial dialog stuck waiting for input. |
| 2026-08-05 | [ansi-escape-animation-attempt](ansi-escape-animation-attempt.md) | Tested whether raw ANSI escape codes smuggled into a chat response could animate a terminal live — negative result, rendering layer strips them. |
| 2026-08-05 | [standing-documentation-practice](standing-documentation-practice.md) | The decision to make logging CS-related work a default, proactive habit rather than something requested case by case. |
