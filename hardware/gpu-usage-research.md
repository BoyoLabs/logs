# GPU Usage Research - 2026-05-22

## Scenario
User is RDP'd into the machine using a separate "dummy" display.

## Hardware Configuration
- **Discrete GPU:** Intel Arc B580 (`card0`)
- **Integrated GPU:** AMD Radeon Graphics (`card1`)

## Findings
- **Current Usage:** The RDP session is bound to the Intel Arc B580.
- **RuneScape:** Verified to be running within the session, therefore utilizing the Intel Arc B580.
- **OnShape (WebGL):** Since the session compositor is on the Intel Arc, the browser and its WebGL renderers will use the discrete GPU by default.

## Recommendations for GPU Enforcement
- **OpenGL:** Use `DRI_PRIME=0` to ensure the Intel Arc is used.
- **Vulkan:** Use `VK_ICD_FILENAMES` pointing to the Intel ICD.
- **Browser:** Enable Hardware Acceleration in settings.
