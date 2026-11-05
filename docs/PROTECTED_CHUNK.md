# AES-GCM protected-chunk milestone

The protected-chunk milestone removes the external-memory time-of-check/time-of-use assumption from the two-pass AES-GCM release path. Each CHK3 transaction is captured once into a 4096-byte on-chip buffer. Capture then locks; the same bytes are replayed for authentication-only mode and, only after a valid tag, replayed again for decryption and plaintext release.

## Security behavior

1. The host sends a 32-byte canonical CHK3 header and RGB888 chunk payload (`host/gcm/protected_chunk/chk3_protocol.py`).
2. The firmware (`software/vitis_gcm/protected_chunk/main_gcm_dma_protected_chunk.c`, `software/vitis_gcm/protected_chunk/platform.h`) captures the chunk and checks the locked-buffer state.
3. Authentication-only GCM runs with S2MM unarmed, so no plaintext destination exists while the tag is checked.
4. A valid tag permits a second replay and plaintext DMA release.
5. Modified ciphertext or AAD is rejected before decryption; rejected data and the protected buffer are zeroized.
6. A fresh session nonce and monotonically ordered chunk index provide unique per-chunk IVs within the volatile session.

## Board evidence

The physical ZCU104 runs passed at all three tested sizes:

| Size | Chunks | Recovery | Tamper rejection | Overall |
|---|---:|---|---|---|
| 17×19 | 1 | PASS | PASS | PASS |
| 256×256 | 48 | PASS | PASS | PASS |
| 512×512 | 192 | PASS | PASS | PASS |

The 512×512 run accepted all 192 chunks and ended with `ZCU104_AES_GCM_PROTECTED_CHUNK_DONE ACCEPTED 192`.

## Reproduction

Run the RTL validation script under `hardware/gcm_protected_chunk/sim/`, build the ZCU104 project with the three scripts under `hardware/gcm_protected_chunk/scripts/`, then build the Vitis application from the exported XSA (`software/vitis_gcm/protected_chunk/`). Start the host script before launching the application:

```powershell
py host/gcm/protected_chunk/test_chunk_protocol_offline.py
py host/gcm/protected_chunk/run_gcm_protected_chunk.py `
  "C:/path/to/dataset/image.png" COMx `
  --size 256x256 --sequence 101 --output protected_chunk_256_run1
```

Replace `COMx` with the serial port assigned to the board on your computer.

## BRAM optimization validation

The protected ciphertext storage was refactored into a synchronous simple-dual-port memory (`ProtectedChunkMemory_BRAM.v`) to enable block-RAM inference.

| Metric | Pre-BRAM buffer wrapper | BRAM-optimized buffer wrapper | Optimized memory child |
|---|---:|---:|---:|
| LUTs | 58,392 | 371 | 86 |
| FFs | 33,421 | 238 | 0 |
| RAMB36 | 0 | 2 | 2 |
| RAMB18 | 0 | 0 | 0 |

The full-design totals in the same utilization report pair are 78,133 LUTs, 46,149 FFs, 4 RAMB36, and 2 RAMB18 before; and 20,112 LUTs, 12,966 FFs, 6 RAMB36, and 2 RAMB18 after. These are report totals and are not attributed solely to the buffer refactor. The archived pair contains utilization reports only, so WNS/WHS and before/after power comparisons are not claimed here.

The optimized bitstream passed physical ZCU104 validation for:

- 17x19 image: 1/1 chunk PASS
- 256x256 image: 48/48 chunks PASS
- 512x512 image: 192/192 chunks PASS

Ciphertext tampering and AAD tampering were rejected before plaintext release, and rejected buffers were zeroized.

## Resource and scope notes

Vivado power reports are estimates, not physical board rail measurements. The BRAM comparison archive does not include paired timing or power reports; refer to the separately scoped implementation evidence below rather than inferring timing or power changes from utilization reports.

The prototype uses a demonstration AES key, volatile session freshness state, RGB888 input only, and a 4096-byte chunk limit. It does not claim resistance to side-channel or fault-injection attacks and is not a clinical product.

### Vivado utilization

The before-and-after Vivado utilization reports show the protected buffer wrapper (`protected_chunk_buffer_0`) decreasing from 58,392 LUTs and 33,421 FFs, with 0 RAMB36 blocks, to 371 LUTs and 238 FFs, with 2 RAMB36 blocks. In the optimized hierarchy, the memory child uses 86 LUTs and 2 RAMB36 blocks; the rest of the wrapper uses 285 LUTs. Full-design totals are reported in the archived reports and are not attributed solely to this buffer change.

The memory SmartConnect (`system_mem_smc`) uses 2,059 LUTs: 1,518 logic LUTs, 532 LUTRAMs, and 9 SRLs. This is its utilization in the final design; the report does not establish that it represents a 2,059-LUT increase caused by the BRAM optimization.

## Physical board evidence files

Board-run summaries and UART transcripts are stored under `results/protected_chunk_board_evidence/`. The accompanying `SHA256SUMS.txt` records their SHA-256 hashes. Per-chunk binary dumps are excluded from the repository.
