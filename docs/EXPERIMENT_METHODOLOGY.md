# Experimental Methodology

This document records the scope and pass criteria of the experiments already represented in the repository. It distinguishes the baseline CTR/GCM-DMA measurements, protected-chunk board validation and Cortex-A53 software benchmark. These are related experiments, not one controlled head-to-head benchmark.

## 1. Platform and implementation

| Item | Configuration |
|---|---|
| Board | AMD/Xilinx ZCU104 |
| Device | XCZU7EV-FFVC1156-2-E |
| Tools | Vivado and Vitis Embedded 2026.1 |
| PL clock | 75.002 MHz |
| Processor | Cortex-A53, bare-metal application |
| Accelerator interface | 128-bit AXI4-Stream |
| Control interface | AXI4-Lite |
| Payload movement | AXI DMA between PL and PS DDR |
| Reference implementation | Python `cryptography` AESGCM |
| Cipher | AES-128-GCM |
| IV | 96 bits |
| Tag | 128 bits |

## 2. Baseline CTR-DMA versus GCM-DMA experiment

The baseline performance and functional dataset used 256 Ã— 256 RGB888 images (196,608 bytes per image): 10 dataset images plus one repeat of the first image using a new IV, for 11 transactions.

### Functional procedure

For each transaction:

1. Prepare the image payload and a fresh 96-bit IV.
2. Run the FPGA encryption path.
3. Compare FPGA ciphertext and 128-bit tag against an independent Python AESGCM result.
4. Run authenticated decryption and compare recovered bytes with the original input.
5. Modify ciphertext and check that authentication rejects the modified transaction.
6. Check that the rejected output buffer is zeroized and that firmware does not release unauthenticated plaintext.

The recorded batch result was 11/11 for unique IVs, ciphertext agreement, tag agreement, exact recovery, ciphertext tamper rejection and rejected-buffer zeroization. Unauthenticated plaintext release was 0/11.

The controlled CTR tamper experiment is separate: CTR has no authentication tag, so a ciphertext bit modification produces a corresponding plaintext bit modification without an authentication error.

### Performance procedure and reported values

The recorded transaction size was 196,608 bytes and the PL clock was 75.002 MHz. The repository's measured summaries are:

| Metric | CTR-DMA | GCM-DMA |
|---|---:|---:|
| Encryption latency | 2,186 Âµs | 3,010 Âµs |
| Encryption throughput | 89,939 KB/s | 65,318 KB/s |
| Decryption latency | 2,175 Âµs | 2,999 Âµs |
| Decryption throughput | 90,394 KB/s | 65,557 KB/s |

The archived GCM transcript also contains 100 MHz counter data. Rounded whole-microsecond summaries should not be interpreted as evidence of zero timing variation.

## 3. Protected-chunk validation

The protected-chunk architecture uses a locked 4096-byte on-chip buffer. A transaction consists of a 32-byte CHK3 header and chunk data; a 16-byte metadata field is authenticated as AAD.

### Procedure

1. Transfer a CHK3 transaction from the host to the Cortex-A53 application.
2. Capture the ciphertext chunk once into protected BRAM and lock the buffer.
3. Run authentication-only GCM with S2MM disarmed.
4. On a failed tag, reject the transaction and zeroize the rejected data and protected buffer.
5. On a valid tag, replay the same protected ciphertext for decryption and release plaintext through DMA.
6. Compare the recovered output with the original and verify the transaction summary and UART result.

### Board validation results

| Image dimensions | Chunks | Result |
|---|---:|---|
| 17 Ã— 19 | 1 | 1/1 PASS |
| 256 Ã— 256 | 48 | 48/48 PASS |
| 512 Ã— 512 | 192 | 192/192 PASS |

Ciphertext and AAD tampering were rejected before decryption/plaintext release in the recorded protected-chunk tests. The board evidence is archived under `results/protected_chunk_board_evidence/` and the visual test outputs are under `results/visual_demo/`.

## 4. Cortex-A53 software AES-GCM benchmark

A separate Vitis bare-metal application used Mbed TLS 3.6.7 on the Cortex-A53.

| Parameter | Value |
|---|---:|
| Payload | 4096 bytes |
| AES key | 128 bits |
| IV | 96 bits |
| AAD | 16 bytes |
| Tag | 16 bytes |
| Warm-up iterations | 5 |
| Measured iterations | 100 |
| ARM generic counter | 100 MHz |

Recorded averages:

| Metric | Encryption | Decryption |
|---|---:|---:|
| Average latency | 1681.04 Âµs | 1680.69 Âµs |
| Mean counter ticks | 168104 | 168069 |
| Throughput | 2379 KB/s | 2379 KB/s |

Encryption, decryption, exact recovery and modified-tag rejection passed. UART output is outside the measured operation.

**Important comparison limitation:** the software benchmark times Mbed TLS AES-GCM operations on 4 KiB. The FPGA protected-chunk workflow includes protected-buffer capture, GCM preparation, DMA/replay and zeroization for encryption; FPGA decryption is split into authentication-only and release phases. The payload sizes and timing boundaries differ. Do not calculate or advertise a direct FPGA speedup from these values.

## 5. Implementation and power methodology

The baseline CTR-DMA/GCM-DMA implementation comparison and final protected-chunk implementation PPA are separate report sets.

- CTR-DMA versus GCM-DMA reports are used for the baseline area, timing and estimated-power comparison.
- The final protected-chunk PPA values describe the final routed protected-chunk design.
- The protected-buffer before/after BRAM optimization evidence contains utilization reports only; it does not support a before/after timing or power claim.
- Vivado power values are vectorless estimates without an activity file. They are not physical board-rail measurements. The final protected-chunk estimate includes a PS8 contribution and must not be described as PL-only power.

## 6. Acceptance criteria

| Experiment | Acceptance criterion |
|---|---|
| Reference agreement | FPGA ciphertext and tag exactly match Python AESGCM |
| Valid decryption | Recovered bytes exactly equal original |
| Ciphertext tamper | Invalid authentication; no plaintext release |
| AAD tamper (protected-chunk) | Invalid authentication; no plaintext release |
| Rejected buffer handling | Required output/protected buffer zeroization observed |
| Nonce discipline | No duplicate IV in the recorded baseline batch; ordered per-chunk IV use within volatile protected-chunk session |
| Timing | No negative setup/hold slack in the cited final implementation report |
| Physical routing | Zero routing errors in cited final implementation report |

## 7. Reproduction and evidence

Use the commands and project scripts in [BUILD_AND_RUN.md](../BUILD_AND_RUN.md). The analysis script can regenerate the archived data summaries and figures:

```powershell
py -m pip install -r requirements-analysis.txt
py tools\analyze_results.py
```

Key evidence locations:

- `results/gcm_batch_run1/`: baseline GCM batch data and reports
- `results/ctr_dma_run1/`: CTR baseline data and reports
- `results/a53_software_gcm/`: Cortex-A53 software benchmark transcript
- `results/protected_chunk_board_evidence/`: protected-chunk board summaries and UART transcripts
- `results/visual_demo/`: input, encrypted-data visualization, recovered images and raw evidence
- [RESULTS.md](../RESULTS.md): consolidated recorded results
- [PAPER_EXPERIMENT_PLAN.md](PAPER_EXPERIMENT_PLAN.md): additional work needed for publication-grade evaluation

## 8. Work still needed for a publication study

The current measurements are valid engineering prototype evidence, but not a full statistical study. The existing roadmap recommends a licensed and citable 50â€“100 image dataset, at least three independent board passes per image, raw timer ticks, confidence intervals, comparable timing boundaries across baselines, broader malformed/replay/wrong-key tests, and physical rail power or carefully labelled estimated power.

