# 🧪 LAB LOG: Architectural Design Session
**Project Concept:** The Zero-Infrastructure, Device-Independent Personal Server Mesh
**Date:** May 25, 2026  
**Status:** Theoretical Architecture & Constraint Mapping  

---

## 1. Executive Summary & Core Philosophy
The objective of this architectural exploration is to fundamentally redefine the traditional "home server" model for a single-user workflow. Instead of relying on dedicated, high-maintenance local infrastructure (hypervisors, open inbound ports, static IP allocations, or dynamic DNS daemons), the proposed paradigm shifts the server's execution completely into the user's **version-controlled cloud workspace (GitHub)** and **universal client runtime environments (the browser)**.

### Core Axioms:
1. **Device Independence:** Physical hardware represents modular thin-client "windows" into the workspace. The ultimate workbench is a static repository layout.
2. **Compute vs. Storage Separation:** High-availability file storage and codebase distribution are completely offloaded to public static hosting (GitHub Pages). Physical local iron is utilized purely for raw compute power and local resource interaction (e.g., Python hardware automation).
3. **Zero-Trust Network Footprint:** The home network must remain invisible to the public WAN. No ports are forwarded, and no static IP addresses are used.

---

## 2. System Overview & The Shift to Device Independence

Moving away from a traditional mobile-centric or hardware-tethered development setup, this paradigm treats standard GitHub.com accounts as the absolute laboratory space. Devices like smartphones, handheld Linux platforms, or remote work terminals are entirely decoupled from the actual operating system environment, serving as interchangeable portals to execute or monitor projects. 

### The RDP/Server Bridge Implementation
For specialized, bare-metal tasks—such as direct hardware control for 3D printing automation written in Python—the system avoids browser-sandbox limitations by running specialized execution code on a high-powered home computing asset. Remote Desktop Protocol (RDP) traffic is funneled through encrypted tunnels to access this heavy compute resource on demand from any modular client screen, entirely keeping gaming, scripting, and development environments completely isolated from enterprise firewalls.

---

## 3. Proposed Technical Architecture: The Browser-Based "Server Node"

The system utilizes a dual-mode Single Page Application (SPA) deployed as a static frontend. When executed on an always-on physical machine at home, the app is initialized in **Server Mode**, mutating a browser tab into a persistent data-and-routing daemon.

```text
+------------------------------------------------------------+
|                  UNIVERSAL CLIENT RUNTIME                  |
|       (Static Single Page Application via GitHub Pages)      |
+------------------------------------------------------------+
                             │
              Toggled via UI Initialization
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
     [ SERVER MODE ]                [ CLIENT MODE ]
  (Executed on Home Iron)        (Executed on Mobile/Deck)
              │                             │
    • Mounts OPFS Database                  • Generates Client State
    • Runs WebRTC Listener                  • Initiates Peer Request
              │                             │
              └──────────────┬──────────────┘
                             │
                P2P Network Initialization
                             │
                             ▼
         +---------------------------------------+
         |      DIRECT WEBRTC DATA CHANNEL       |
         |  (Encrypted, Zero-Port, Local-First)  |
         +---------------------------------------+
