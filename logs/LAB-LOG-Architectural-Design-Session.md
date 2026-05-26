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

## 2. Proposed Architecture: The Browser-Based "Server Node"

The system utilizes a dual-mode Single Page Application (SPA) deployed as a static frontend. When executed on an always-on physical machine at home, the app is initialized in **Server Mode**, mutating a browser tab into a persistent data-and-routing daemon.

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

### Data Persistence Layer
* **Technology:** Origin Private File System (OPFS) & WebAssembly SQLite.
* **Mechanism:** The persistent Server Node requests permanent origin retention via `navigator.storage.persist()`. Incoming binary updates streams received from client nodes are written directly to disk via the low-level `FileSystemAccessAPI`. 
* **Quota:** Up to 60% of the host machine's physical storage volume is made accessible to the sandboxed runtime, providing hundreds of gigabytes of zero-config file system access directly within a browser frame.

---

## 4. The Core Engineering Constraint: NAT Traversal vs. Purity

The primary technical bottleneck discovered during architectural analysis centers on **WebRTC Signaling**. Standard WebRTC requires the exchange of Session Description Protocol (SDP) payloads containing ephemeral routing sockets and dynamic cryptographic signatures (`ice-ufrag`/`ice-pwd`). 

Because a home router shifts its temporary external connection ports constantly, two browsers must trade their highly volatile network fingerprints to open a connection. To orchestrate this handshake globally without a central command server, a series of design trade-offs were evaluated against the strict requirement for **total serverless purity**:

| Signaling Mechanism | Structural Friction | Dependency | Architectural Purity |
| :--- | :--- | :--- | :--- |
| **Manual Data Exchange** (QR / Text Blocks) | **High:** Requires physical proximity or manual copy-paste of ephemeral tokens before roaming. | None | **100% Pure** (No third-party compute or network routing required). |
| **Local mDNS Broadcast** (LAN Only) | **High:** Connections drop the moment a client moves to an outside network (e.g., cell towers or office firewalls). | None | **100% Pure** (Constrained to Local Area Network). |
| **Tertiary Key-Value/API Storage** | **Low:** Allows a static user password to fetch dynamic coordinates automatically. | Third-Party API Bucket | **Pseudo-Backend:** Reintroduces a remote dependency / point of failure. |
| **Public Open Protocols** (Matrix/MQTT) | **Low:** Completely automated backend handshake utilizing outbound connections. | Public Infrastructure Mesh | **Decentralized Network Share:** Piggybacks on public communication relays. |

---

## 5. Next Phase Action Items & Prototyping Path
To validate the browser-as-a-server thesis without compromising on architectural constraints, early-stage development will split into two isolated lab tracks:

1. **The Pure Local Sandbox:** Build the vanilla WebRTC `RTCDataChannel` logic and verify data-rate throughput when streaming raw binary file pieces straight into the `FileSystemWritableFileStream` of the OPFS interface on a local loopback.
2. **Handshake Streamlining:** Design the cleanest possible interface for managing manual SDP token transport, optimizing for low-overhead string injection to minimize friction during off-grid or remote testing windows.





