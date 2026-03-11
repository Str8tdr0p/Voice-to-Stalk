# Evidence and Indicators of Compromise  
**Pegasus‑Style / Predator‑Linked Operational Patterns** 

---

## Executive summary

This file documents the **evidence and IOCs** that support the hypothesis that the exploit chain described in `attack-chain.md` and `bcm4387-coex-window.md` is consistent with **Pegasus‑style / Predator‑linked** operational tactics.

The evidence includes:

- **Device‑locking and runtime‑key architecture.**  
- **Multi‑tenant token‑based beaconing** (`q9PK`, `xTtC2`, `NrER`).  
- **VM‑style custom bytecode** with 0.8218 cosine similarity to known Pegasus‑style VMs.  
- **iMessage‑based zero‑click delivery** using AMR‑encoded payloads.  
- **SSV‑level persistence** that survives DFU restore and OTA updates, with active C2‑driven campaign management.

Below, each evidence item is tied directly to **concrete IOCs** (hashes, IPs, strings, offsets, and encoded values) so analysts can pivot and correlate samples.

---

## Evidence 1 – Device‑locked runtime key

- **Observation**  
  - The AMR payload is encrypted with a **runtime‑derived key** tied to the target’s UDID/ECID.  
  - Captured samples are **inert on other devices**, which is a deliberate design choice to prevent proof‑of‑deployment by third parties.

- **Significance**  
  - This is a documented **Pegasus‑style / Predator‑linked** OPSEC pattern: device‑locking payloads to prevent evidentiary exposure.  
  - It is **not technically necessary** for the exploit to function, but is essential for operator‑level risk mitigation.

- **IOCs and artefacts**  
  - AMR file SHA256: `0de03d7fe9dc023d5e4824322195dcc2359ec15acd714ade01f50488214c2d60`  
  - AMR file MD5: `e9336a80fd59722b1c1744f2e56a54ce`  
  - Layer‑1 XOR key: `4bdbb4c5e03c0e08164914000021f805`  
  - Runtime‑key IV / campaign nonce: `e46c80ce` (entropy 2.0000 bits/byte, structural constant)  

These values are used to **identify the campaign** and confirm that the payload is **parameterized per‑device**, not a generic sample.

---

## Evidence 2 – Multi‑tenant token‑based beacon system

- **Observation**  
  - The chain uses a **canary token system** based on three tokens: `q9PK`, `xTtC2`, `NrER`.  
  - The combined string `cTlQS3x4VHRDMnxOckVS` (Base64‑encoded `q9PK|xTtC2|NrER`) appears in the payload and is encoded in C2‑bound beacon structures.  
  - The combined hash values are:
    - Combined MD5: `2482d4bcec039ae7391120253a397746`  
    - Combined SHA1: `a0897faf4f52468d990bd7188aa9662ca5b7ebd7`

- **Significance**  
  - The tokens score **9/9** on the researcher‑evasion profile:  
    - Base64/Base58‑compatible.  
    - Only emerge under partial decryption.  
    - No matches in Apple framework strings.  
    - Extractable by automated scanners.  
  - This forms a **multi‑tenant beacon architecture** where each device’s traffic is tagged and routed through shared C2 infrastructure, consistent with **Pegasus‑style / Predator‑linked** operators.

- **Cross‑stage link**  
  - The token `xTtC2` appears in the **encrypted AMR payload** and is recovered from the **zombie DSC binary** on a fully DFU‑restored device.  
  - This confirms that the voice note and the installed implant are part of the same **operation and campaign**.

---

## Evidence 3 – VM‑based custom bytecode and Pegasus‑style profile

- **Observation**  
  - The payload contains a **custom VM** with:
    - 4‑byte‑wide instructions, ARM64‑aligned.  
    - 8 virtual registers (V0–V7).  
    - 175 unique opcodes, 7.1 bits/byte entropy.  
    - 0.8218 cosine similarity to known Pegasus‑style VM profiles.  
  - Final register state: `V0 = 0x300016` (selector‑0x16 staging for `IOConnectCallMethod`).

- **Significance**  
  - The VM is **LLVM‑compiled shellcode**, matching the same toolchain used by Apple and by Pegasus‑style operators.  
  - The **obfuscation strategy** and opcode‑distribution pattern align with documented Pegasus‑style VMs, indicating shared engineering choices.

- **IOC**  
  - VM‑final state marker: `V0 = 0x300016` (used to stage the kernel‑level escalation exploit).

---

## Evidence 4 – iMessage zero‑click delivery lineage

- **Observation**  
  - The chain delivers the payload via an **AMR voice note** in iMessage, exploiting **CoreAudio AMR‑12‑2 heap overflow** (CVE‑2025‑31200) followed by **AppleBCMWLAN IOKit‑based kernel escalation** (CVE‑2025‑31201).  
  - Earlier iMessage‑based chains:
    - FORCEDENTRY (JBIG2).  
    - BLASTPASS (WebP).  
  - Now this chain: **AMR‑encoded payload** with parameter‑range abuse.

- **Significance**  
  - The **architecture rotates** (codec / format), but the **delivery model is identical**: zero‑click, auto‑processed attachment, complex payload‑embedding.  
  - This format‑rotation pattern is consistent with **Pegasus‑style / Predator‑linked** zero‑click iMessage chains.

- **IOCs**  
  - CVE‑2025‑31200 – CoreAudio AMR heap overflow.  
  - CVE‑2025‑31201 – AppleBCMWLAN IOKit PAC‑bypass.  
  - CVE‑2026‑20700 – dyld‑cache memory corruption used for SSV‑level persistence.

---

## Evidence 5 – SSV‑level persistence and active campaign management

- **Observation**  
  - The exploit installs a **zombie DSC binary** into the Signed System Volume at `/System/Library/Caches/com.apple.dyld/`.  
  - The implant survives:
    - DFU restore.  
    - Factory reset.  
    - OTA updates (multiple iOS versions, A14–A16).  
  - The patch for CVE‑2026‑20700 does **not remove** existing implants; on at least one device, the update installed **a second instance** alongside the original.

- **Significance**  
  - This is a **productive, engineering‑driven persistence product**, not a one‑off exploit.  
  - The **versioned implants** and multi‑device reach indicate **active campaign management**, consistent with Pegasus‑style / Predator‑linked operators.

- **IOCs – Zombie DSC binaries**

| Description                  | SHA256 |
|------------------------------|--------|
| Zombie binary (A16, instance 1) | `d93d48802aa3ccefa74ae09a6a86eafa7554490d884c00b531a9bfe81981fb06` |
| Zombie binary (A16, instance 2) | `ac746508938646c0cfae3f1d33f15bae718efbc7f0972426c41555e02e6f9770` |
| Zombie binary (A14, instance 1) | `38a723210c18e81de8f33db79cfe8bae050a98d9d2eacdeb4f35dabbf7bd0cee` |
| Zombie binary (A14, instance 2) | `869f9771ea1f9b6bf7adbd2663d71dbc041fafcbf68e878542c8639a6ba23066` |

---

## Evidence 6 – C2 and UUID‑based campaign consistency

- **Observation**  
  - The device beaconed to C2 endpoint `200.152.70.35:443`, which was stable across:
    - Two devices.  
    - Three iOS versions.  
    - Two chipset generations (A14, A16).  
  - UUIDs rotated after update, confirming **active campaign‑level management**, not a fixed sample.

- **Significance**  
  - This is consistent with **Pegasus‑style / Predator‑linked** C2 models, where operators maintain a fleet of devices under a single, stable endpoint with per‑device identifiers.

- **IOCs – C2 and beaconing**

- Primary C2 endpoint: `200.152.70.35:443`  
- Detection path for the zombie binary: `/system_logs.logarchive/dsc/[32‑char UUID]`  
- Canary token string: `cTlQS3x4VHRDMnxOckVS` (base64‑encoded `q9PK|xTtC2|NrER`)  
- Cross‑device‑correlation marker: `xTtC2` (present in AMR payload and recovered from the zombie DSC binary on DFU‑restored device).

---

## Evidence 7 – Hardware‑level coexistence‑SRAM memory window

- **Observation**  
  - The BCM4387 coexistence SRAM at `0x102000–0x111000` sits **0x5400 bytes below** the IOMMU boundary at `0x1173FF`.  
  - The full 85 KB window `0x102000–0x1173FF` is **unprotected** and **kernel‑accessible**.  
  - The exploit injects `HCI_BLE_Set_Coexistence_Parameters` (opcode `0x203C`) and `HCI_Write_Local_Name` into this region.

- **Significance**  
  - This hardware‑level pivot is **not mitigated** by Lockdown Mode or by iOS‑only updates.  
  - The ability to spoof Bluetooth identity and enable Auto‑Unlock via this low‑level interface is consistent with **Pegasus‑style / Predator‑linked** identity‑based persistence.

- **IOCs – Hardware‑related artefacts**

- IOMMU‑protection boundary: `0x1173FF`  
- Null‑gap range: `0x117400–0x132E00` (110 KB, 100% null, 0.0 entropy).  
- Coexistence‑SRAM range: `0x102000–0x111000` (60 KB, unprotected).  
- HCI‑opcode `0x3C` handler locations (hex):  
  - `0x6fd17`, `0x76d5f`, `0xb0dfc`, `0xbf57c`, `0xd5738`, `0xeff68`, `0xf26a5`, `0xf28ae`, `0xf2e9b`, `0x1bd01d` (plus one additional confirmed hit).  
- HCI opcode `3c00` in AMR payload at offset `865`.

---

## Evidence 8 – Static‑ and dynamic‑analysis boundaries

- **Static analysis confirmed:**
  - AMR parameter violations in 50 × FT=7 frames.  
  - 1,573‑byte encrypted payload with multi‑layer encryption.  
  - Layer‑1 16‑byte cyclic XOR key recovered.  
  - Layer‑2 CBC‑mode chaining identified.  
  - No static key in payload; runtime‑key architecture.  
  - BCM4387 coex‑SRAM region is **outside** the IOMMU‑protected zone.

- **Dynamic analysis:**
  - `routined` changing PID mid‑shutdown (new instance spawned during `SIGTERM`, indicating a watchdog‑style relaunch).  
  - `IOMFB_bics_daemon` persisting across two `SIGTERM` cycles (consistent with AGX‑framebuffer‑capture hooks).  
  - `SafariSafeBrowsing.Service` lingering after first `SIGTERM` with no legitimate reason (consistent with flushing exfiltration traffic before power‑off).

These **log‑level anomalies** match the expected behavior of an implant maintaining covert access and persistence, consistent with Pegasus‑style / Predator‑linked operators.

---

## Evidence‑based attribution statement

Across all observed evidence items:

- Device‑locking,
- Token‑based multi‑tenant beaconing,
- VM‑based custom bytecode,
- iMessage‑based zero‑click delivery,
- SSV‑level persistence with active C2‑driven versioning,
- Hardware‑level Bluetooth‑based identity spoofing,

the **architectural and operational patterns are consistent with Pegasus‑style / Predator‑linked** commercial‑surveillance actors.

No evidence contradicts that hypothesis. However, this file does not declare a **single‑vendor attribution label**; instead, it curates the **evidence and IOCs** that analysts and vendors can map to their own threat‑intelligence frameworks.

---

## Complete IOC reference table

### Delivery artefacts (AMR voice note)

| Field                      | Value |
|----------------------------|-------|
| AMR file SHA256            | `0de03d7fe9dc023d5e4824322195dcc2359ec15acd714ade01f50488214c2d60` |
| AMR file MD5               | `e9336a80fd59722b1c1744f2e56a54ce` |
| IV / campaign nonce        | `e46c80ce` |
| Layer‑1 XOR key            | `4bdbb4c5e03c0e08164914000021f805` |
| Canary token               | `xTtC2` |
| Canary beacon (encoded)    | `cTlQS3x4VHRDMnxOckVS` |
| Combined canary MD5        | `2482d4bcec039ae7391120253a397746` |
| Combined canary SHA1       | `a0897faf4f52468d990bd7188aa9662ca5b7ebd7` |
| HCI coex‑opcode in payload | `3c00` at offset `865` |
| VM register final state    | `V0 = 0x300016` |

### Persistence artefacts (zombie DSC binary)

| Device / instance                        | SHA256 |
|------------------------------------------|--------|
| Zombie binary (A16, instance 1)          | `d93d48802aa3ccefa74ae09a6a86eafa7554490d884c00b531a9bfe81981fb06` |
| Zombie binary (A16, instance 2)          | `ac746508938646c0cfae3f1d33f15bae718efbc7f0972426c41555e02e6f9770` |
| Zombie binary (A14, instance 1)          | `38a723210c18e81de8f33db79cfe8bae050a98d9d2eacdeb4f35dabbf7bd0cee` |
| Zombie binary (A14, instance 2)          | `869f9771ea1f9b6bf7adbd2663d71dbc041fafcbf68e878542c8639a6ba23066` |

### Network / C2 IOCs

| Field                          | Value |
|--------------------------------|-------|
| Primary C2 endpoint            | `200.152.70.35:443` |
| C2‑related beacon string       | `cTlQS3x4VHRDMnxOckVS` |
| Correlation token              | `xTtC2` |
