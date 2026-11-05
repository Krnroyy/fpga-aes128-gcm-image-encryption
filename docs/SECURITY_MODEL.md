# Security Model and Limitations

## Protected asset

The protected asset is the RGB image payload transferred between the laptop, Cortex-A53 memory and programmable-logic accelerator. This repository contains two related AES-GCM data paths: the baseline `gcm_dma` path and the protected-chunk path. Their plaintext-release behavior differs, so security claims below identify the path they describe.

## Adversary considered

The experiments consider an attacker who can observe or modify ciphertext while it is stored or transmitted. The attacker does not know the 128-bit AES key and cannot modify the trusted FPGA configuration or running control software.

## Baseline `gcm_dma` path

- FPGA ciphertext and 128-bit tags matched Python `AESGCM` for the 11 tested transactions.
- Modified ciphertext was rejected by the GCM tag check in the tested transactions.
- The application transmitted recovered plaintext only after successful authentication.
- After a failed authentication, the reserved DDR output buffer was zeroized. Unauthenticated decrypted bytes could temporarily occupy that buffer before the software checked the authentication result; the baseline path does not provide on-chip quarantine through tag verification.
- The baseline core does not authenticate image identifiers, dimensions or other metadata as GCM additional authenticated data (AAD).

## Protected-chunk path

- Each CHK3 transaction is captured once into a 4096-byte on-chip buffer, then locked against further writes.
- Authentication-only GCM runs with S2MM disarmed. The same protected ciphertext is replayed for decryption only after a valid tag.
- The 16-byte authenticated metadata field within the 32-byte CHK3 header is supplied as AAD. Tests rejected modified ciphertext and modified AAD before decryption or plaintext release.
- Rejected data and the protected buffer are zeroized. A fresh session nonce and ordered chunk index provide per-chunk IV uniqueness within the volatile session; they do not provide persistent freshness across resets.
- Physical ZCU104 tests passed for 17×19 (1/1 chunk), 256×256 (48/48), and 512×512 (192/192) images.

## Why CTR alone was insufficient

CTR mode encrypts by XORing plaintext with a generated keystream. A ciphertext bit flip therefore causes a predictable plaintext bit flip after decryption. The controlled ZCU104 CTR experiment changed one ciphertext byte and observed exactly one changed plaintext byte, with no authentication error. GCM retains CTR-like confidentiality while GHASH and the tag detect the tested modification.

## Trust boundaries

The Cortex-A53 software, PL bitstream, AES key-loading path and board are trusted. UART and ciphertext storage may be untrusted. The Python program is used as an independent experimental oracle, not as part of the deployed trust boundary.

## Limitations requiring careful claims

1. The project does not prove resistance against side-channel attacks, voltage or clock fault injection, bitstream replacement, or malicious privileged software.
2. The demonstration key is not protected by a PUF, eFUSE, BBRAM or hardware security module.
3. Host-generated random IVs passed uniqueness checks in the baseline experiment. The protected-chunk path enforces ordered chunk use within a volatile session. Production use needs persistent IV allocation that prevents reuse under the same key, including across resets.
4. The baseline `gcm_dma` core does not use AAD. The protected-chunk path authenticates the 16-byte CHK3 metadata field (header bytes 4–19). Other image or DICOM metadata fields are covered only if represented in those authenticated bytes.
5. On the baseline path, unauthenticated decrypted bytes may temporarily occupy the reserved DDR output buffer, but firmware does not transmit them and zeroizes the buffer after authentication failure. The protected-chunk path keeps ciphertext in locked on-chip storage during authentication and does not arm S2MM until authentication succeeds.
6. Statistical image metrics do not establish cryptographic security. Standard conformance, reference agreement, nonce discipline and adversarial tests are stronger evidence.
7. This is not a clinical workflow or regulatory validation.

## Safe publication language

Use: “The prototype experimentally demonstrated authenticated encryption, exact reference agreement and rejection of the tested ciphertext modifications on ZCU104. The protected-chunk path authenticated each tested 16-byte CHK3 metadata field and withheld plaintext release until authentication passed.”

Avoid: “The system is unbreakable,” “clinically secure,” or “GCM prevents every possible attack.”
