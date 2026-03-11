# Attack Chain: From AMR iMessage to Hardware‑Level Bluetooth Pivot  
**Zero‑Click iMessage Exploit Chain · March 2026**

---

## Stage 1 – Delivery: AMR Voice Message Auto‑Processing

When an iMessage voice note arrives, iOS automatically passes the attachment to the `imagent` daemon for processing, even if the user has never seen or tapped the message. This auto‑processing behavior is the foundation of the zero‑click delivery model.

The exploit uses an **AMR‑NB voice message** (1,606 bytes, 50 FT=7 frames) delivered via iMessage. The AMR header is valid; all frames declare mode AMR 12.2 kbps (FT=7) and pass container‑level validation. BlastDoor accepts the file as legitimate audio.

However, the exploitation occurs not in the header, but in **codec‑level parameters within the bitstream** (pitch lag, LSF coefficients, codebook indices). These values are deliberately set outside their legal ranges, but the CoreAudio AMR 12.2 decoder does not validate them. The payload is distributed across these fields, not in the container metadata.

**Why Lockdown Mode does not block this:**  
Lockdown Mode does not restrict iMessage‑driven AMR voice‑message decoding. The exploit is delivered via an auto‑processed AMR payload that is still handled by `imagent` and CoreAudio even when Lockdown Mode is enabled.

---

## Stage 2 – Entry: CoreAudio AMR Heap Overflow (CVE‑2025‑31200)

The AMR 12.2 decoder processes fifty frames containing 1,800 parameter values. Across these frames:

- 189 parameter violations are observed.  
- All 50 frames carry at least one violation.  
- Pitch lag values reach 248, well above the 3GPP‑defined maximum of 143.  
- The decoder allocates 306 bytes but writes up to 516 bytes, producing a **210‑byte heap overflow in `imagent`**.

This overflow yields a controlled write primitive that is used in the next stage to stage a kernel‑level call. The payload is not a simple appended blob: it is a 1,573‑byte encrypted stream woven into the bitstream parameter fields, implementing a multi‑layer encryption architecture.

CVE‑2025‑31200 was patched in iOS 18.4.1, but this patch does not address the prior deployment window or the downstream chain components.

---

## Stage 3 – Escalation: AppleBCMWLAN IOKit PAC‑Bypass (CVE‑2025‑31201)

The write primitive from Stage 2 is not used directly. Instead, the payload embeds a **custom virtual machine (VM)**:

- 4‑byte‑wide instructions, ARM64‑aligned.  
- 8 virtual registers (V0–V7).  
- 175 unique opcodes, with 7.1 bits/byte entropy, consistent with LLVM‑compiled bytecode.  
- Cosine similarity 0.8218 to known Pegasus‑style VM profiles.

The VM executes and its registers converge on `V0 = 0x300016`, a call to `IOConnectCallMethod` targeting selector index `0x16` in the AppleBCMWLAN kernel extension. The same index appears in the `IOBluetoothHCIController` selector table as `HCIWriteLinkPolicy`, indicating a dual‑path use of this selector.

Selector `0x16` in AppleBCMWLAN routes to `parseAggregateFrame`, the 802.11n/ac AMPDU aggregate frame parser. The exploit feeds a subframe length of 65,535 bytes into a 2,048‑byte‑allocated buffer, producing **63,487 bytes written past the end** into kernel memory, giving arbitrary kernel read/write.

PAC is bypassed by locating a valid, already‑authenticated IOKit port pointer and replacing the connection structure it references, rather than forging the pointer value itself. The PAC‑signed pointer remains valid; only the referenced content is corrupted.

CVE‑2025‑31201 was patched in iOS 18.4.1 alongside CVE‑2025‑31200. The patch does not retroactively remove any implants installed before the update.

---

## Stage 4 – Hardware Pivot: BCM4387 Coexistence SRAM (Unpatched)

With kernel R/W achieved, the exploit resolves the BCM4387 MMIO base address from the AppleBCMWLAN mapping and writes directly into the **Broadcom coexistence SRAM**, a shared memory region used to coordinate antenna access between the Wi‑Fi MAC and Bluetooth controller.

Measured from a live BCM4387 firmware dump:

| Region                     | Address Range      | Size     | Protected? | Notes |
|----------------------------|--------------------|----------|------------|-------|
| IOMMU‑protected firmware   | 0x000000–0x1173FF  | ~1.1 MB  | Yes        | Firmware code |
| Null gap (boundary)        | 0x117400–0x132E00  | 110 KB   | –          | 100% null, 0.0 entropy |
| Coexistence SRAM           | 0x102000–0x111000  | 60 KB    | No         | Shared Wi‑Fi/Bluetooth buffer |
| Unprotected kernel window  | 0x102000–0x1173FF  | 85 KB    | No         | Directly reachable from kernel |

The coexistence SRAM sits **0x5400 bytes below the IOMMU boundary** (0x1173FF). The IOMMU fence simply does not cover it.

**Lockdown‑Mode bypass note:**  
Lockdown Mode cannot enforce IOMMU protection on the BCM4387 coexistence SRAM region. Once the kernel is compromised, this window is fully accessible from kernel context, and Lockdown‑Mode‑only controls cannot prevent writes to it.

The payload contains `HCI_BLE_Set_Coexistence_Parameters` (opcode `0x203C`, encoded as `3c00`) at offset 865. This opcode appears at **eleven handler locations** in the BCM4387 firmware, confirming the attacker knew the exact target firmware and HCI handling logic.

The pivot sequence:

1. Kernel R/W via AppleBCMWLAN (CVE‑2025‑31201).  
2. Direct write to coexistence SRAM at 0x102000.  
3. Injection of `HCI_BLE_Set_Coexistence_Parameters`, corrupting the coex‑arbiter state machine and giving the BT controller exclusive channel access.  
4. `HCI_Write_Local_Name` overwrites the device’s Bluetooth identity.  
5. `HCI_Create_Connection` establishes a connection as a spoofed trusted identity, bypassing pairing.  
6. The spoofed identity triggers Auto‑Unlock evaluation; identityservicesd invokes SEP‑backed key signing with no user prompt.

This vector has **no CVE assigned** and **no patch**. Every iPhone 13–16 using BCM4387 remains exposed to this hardware‑level memory‑window primitive, regardless of iOS version or Lockdown‑Mode status.

---

## Stage 5 – Persistence: SSV‑Level Zombie DSC Binary (CVE‑2026‑20700)

The exploit writes a **zombie DSC binary** into the Signed System Volume at `/System/Library/Caches/com.apple.dyld/`. This partition is cryptographically sealed at the hardware root‑of‑trust level; the implant survives:

- Full DFU restore.  
- Factory reset.  
- OTA updates across multiple iOS versions and two Apple SoC generations (A14, A16).

The link between the delivery payload and the installed implant is the **canary token `xTtC2`**:

- `xTtC2` appears in the encrypted AMR payload.  
- The same token is recovered from the zombie DSC binary on a fully DFU‑restored device.  

This cross‑stage consistency is consistent with **Pegasus‑style / Predator‑linked** device‑locking and beacon‑token architectures, but the payload is not labeled as a specific vendor‑owned spyware in this repo.

Apple issued **CVE‑2026‑20700** in iOS 26.3, acknowledging “memory corruption in dyld allowing r/w in attacks against specific targeted individuals.” The patch does not remove existing implants from already‑infected devices; on at least one device, the iOS 26.3.1 update installed a **second instance** alongside the existing one.

---

## Stage 6 – C2, Canaries, and Operational Patterns

The canary token system functions as a **multi‑tenant operator beacon architecture**:

- Tokens `q9PK`, `xTtC2`, `NrER` are designed to be Base64/Base58‑compatible, emerging only under partial decryption, and matching no common Apple framework strings.  
- The combined string `cTlQS3x4VHRDMnxOckVS` base64‑decodes to an 11‑byte fragment with entropy ~3.459 bits, consistent with a truncated HMAC‑SHA1 beacon authenticator.

If an analyst’s sandbox or detection tool extracts these tokens and contacts C2, the operator receives confirmation that that specific build is under analysis.

Additional evidence‑based indicators consistent with **Pegasus‑style / Predator‑linked** operators:

- **Device‑locked payload** – no static key in the payload; the key is derived from the target UDID/ECID, making captured samples inert on other devices.  
- **VM‑style custom bytecode** – LLVM‑compiled VM with 0.8218 cosine similarity to known Pegasus‑style VMs, indicating shared engineering choices.  
- **iMessage zero‑click delivery architecture** – follows the same delivery model as FORCEDENTRY (JBIG2) and BLASTPASS (WebP), previously associated with similar commercial‑surveillance models.  
- **C2 and persistence model** – stable C2 endpoint `200.152.70.35:443`, UUID rotation after update, and SSV‑level persistence with active version management, consistent with a mature surveillance vendor.

None of these features are unique to a single vendor, but their **convergence** is consistent with documented Pegasus‑style / Predator‑linked patterns.

---

## Stage 7 – Why Lockdown Mode and Users Cannot “Break” This Chain

Users cannot break this chain once it has been executed, even after enabling Lockdown Mode or installing the latest iOS updates, for three main reasons:

1. **AMR voice‑message processing remains enabled** under Lockdown Mode; the vulnerability is in CoreAudio’s parameter‑range validation, not in Lockdown‑Mode‑controlled attachment types.  
2. **Kernel‑level IOKit‑based escalation and hardware‑level memory‑window access** lie outside the Lockdown‑Mode control plane; the exploit does not rely on typical Lockdown‑Mode‑restricted features (e.g., WebKit JIT, certain iMessage attachment types, or invite‑based services).  
3. **CVE‑2026‑20700 patches the underlying memory‑corruption primitive but not the installed SSV‑level implant**; the payload survives DFU restore and OTA updates, leaving the device effectively permanently compromised unless the hardware is physically replaced.

For confirmed infections, there is **no public remediation path**; analysts should treat the device as a forensic artifact and transition to a clean device.

---

## Next steps in the repo

- `bcm4387-coex-window.md` – focused hardware‑level analysis of the BCM4387 coexistence SRAM, IOMMU‑boundary layout, and remediation proposals.  
- `attribution.md` – consolidated evidence matrix and Pegasus‑style / Predator‑linked tactic alignment.  
- `iocs.md` – full list of hashes, IPs, tokens, offsets, and encoded strings.
