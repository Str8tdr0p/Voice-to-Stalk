# Zero‑Click iMessage Exploit Chain — AMR to BCM4387 Hardware Pivot
Public Disclosure · March 2026

---

## Brief overview

This research documents a zero‑click iMessage exploit chain that leverages an **AMR voice message** to achieve **iOS kernel read/write**, then pivots via the **BCM4387 coexistence SRAM** into Bluetooth‑based identity spoofing and **Signed System Volume (SSV)‑level persistence**.

The chain demonstrates operational patterns consistent with commercial surveillance vendors:
- Device‑locked payloads derived from UDID/ECID.
- Multi‑tenant, token‑based beaconing to shared C2 infrastructure.
- iMessage‑based zero‑click delivery with codec‑level payload embedding.
- SSV‑level persistence that survives DFU restore and OTA updates.

Affected devices: **iPhone 13–16 (A15–A18)** using the BCM4387 Wi‑Fi/Bluetooth combo chip. The BCM4387 coexistence memory window remains **unpatched** and **outside the IOMMU boundary**, and the **CVE‑2026‑20700 dyld‑cache‑based implant survives DFU restore and OTA updates**, with no public remediation available for already‑infected devices. The chain is **not blocked** by Lockdown Mode.

---

## Attack chain at a glance

- **Delivery** – iMessage auto‑processes an AMR voice note; no user action required.  
- **CoreAudio exploitation** – illegal AMR parameter values trigger a 210‑byte heap overflow in `imagent` (CVE‑2025‑31200).  
- **Kernel escalation** – a custom LLVM‑compiled VM abuses AppleBCMWLAN IOKit selector `0x16`, achieving kernel R/W via a 63 KB buffer overflow with PAC‑bypass (CVE‑2025‑31201).  
- **Hardware pivot** – the kernel writes directly into the BCM4387 coexistence SRAM (0x102000–0x111000), below the IOMMU boundary, injecting HCI commands and spoofing the device’s Bluetooth identity.  
- **Persistence** – the exploit installs a zombie DSC binary into the Signed System Volume (`/System/Library/Caches/com.apple.dyld/`), surviving DFU and OTA updates (CVE‑2026‑20700).  
- **C2 / Actions** – the device beaconed to C2 endpoint `200.152.70.35:443` using multi‑tenant canary‑token beacons such as `xTtC2`, tying the AMR payload to the installed implant.

**Key unresolved gaps:**  
- BCM4387 coexistence SRAM is outside the IOMMU boundary and writable from kernel context; **no patch currently exists**.  
- The CVE‑2026‑20700 dyld‑cache‑based implant **survives DFU restore and OTA updates**, with no public remediation available for already‑infected devices.

---

## Lockdown Mode does not block this chain

Lockdown Mode does not block this chain because it targets surfaces it is not designed to restrict. The exploit uses an auto‑processed AMR voice message in iMessage (unaffected by Lockdown Mode restrictions), a kernel‑level IOKit PAC‑bypass in AppleBCMWLAN, and a hardware‑level BCM4387 coexistence SRAM window below the IOMMU boundary. Once the kernel is compromised, the attacker pivots into Bluetooth HCI injection and installs an SSV‑level persistence implant that survives DFU and OTA updates; none of these layers are mitigated by Lockdown‑Mode‑only controls.

---

## Operational patterns

Several architectural choices in this chain are consistent with documented commercial surveillance tooling:

- **Device‑locked payload** – runtime key derived from target UDID/ECID, making captured samples inert on other devices.
- **Multi‑tenant token‑based beaconing** – tokens `q9PK`, `xTtC2`, `NrER` used to track analysis and correlate campaigns.
- **VM and compiler profile** – LLVM‑compiled custom VM with 0.8218 cosine similarity to known commercial surveillance VMs.
- **Delivery architecture** – follows the same iMessage zero‑click model as FORCEDENTRY (JBIG2) and BLASTPASS (WebP), using AMR as the codec vector.
- **C2 and persistence model** – stable C2 endpoint, campaign‑level UUID rotation, and SSV‑level persistence with active version management.

Attribution is not declared in this repo. The evidence is structured so that analysts can map it to open‑source threat‑actor reporting independently.

---

## Indicators of Compromise

For a full list of file hashes, IPs, canary tokens, and crypto artefacts, see:

- `iocs.md` – exhaustive list of hashes, IPs, tokens, offsets, and encoded beacon strings.

---

## Recommendations

- For confirmed or suspected infected devices: **DFU restore does not remove the implant**. There is no reliable remediation path short of hardware replacement. Treat the device as permanently compromised and transition to a clean device.  
- Use the  detection tool [ZombieHunter](https://github.com/JGoyd/ZombieHunter)  to assess infection status.  
- Preserve the device as forensic evidence; it may contain additional telemetry and samples relevant to attribution and vendor disclosure.

---

## Repo structure

- `README.md` – high‑level brief for policy‑technical and technical audiences.  
- `attack-chain.md` – detailed stage‑by‑stage breakdown of the exploit chain.  
- `bcm4387-coex-window.md` – hardware‑level analysis of the BCM4387 coexistence SRAM and IOMMU‑bypass.   
- `iocs.md` – comprehensive list of hashes, IPs, tokens, offsets, and encoded strings.  
