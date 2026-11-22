# ZCU104 FPGA AES-128-GCM Image Protection

Hardware/software co-design for authenticated image encryption on the AMD/Xilinx ZCU104 Zynq UltraScale+ MPSoC. A Cortex-A53 application transfers 256 × 256 RGB888 images through AXI DMA to a custom 128-bit AES-GCM AXI4-Stream accelerator in programmable logic.

The final implementation provides confidentiality, a 128-bit authentication tag, ciphertext-tamper detection, and controlled plaintext release. FPGA ciphertext and tags were independently checked against Python's `AESGCM` implementation.

## Verified outcome

| Test | Measured result |
|---|---:|
| Physical board | ZCU104 (`xczu7ev-ffvc1156-2-e`) |
| Toolchain | Vivado and Vitis Embedded 2026.1 |
| PL clock | 75.002 MHz |
| Image format | 256 × 256 RGB888, 196,608 bytes |
| Dataset | 10 images + 1 same-image/new-IV transaction |
| Unique 96-bit IVs | 11/11 |
| FPGA ciphertext matched Python AESGCM | 11/11 PASS |
| FPGA 128-bit tag matched Python AESGCM | 11/11 PASS |
| Exact recovery | 11/11 PASS |
| Modified ciphertext rejected | 11/11 PASS |
| Rejected output buffer zeroized | 11/11 PASS |
| Unauthenticated plaintext released | 0/11 |

## Architecture

```mermaid
flowchart LR
    Host["Laptop: image, IV, verification"] -->|UART| A53["Cortex-A53 bare-metal app"]
    A53 <--> DDR["PS DDR buffers"]
    DDR -->|AXI DMA MM2S, 128-bit| GCM["AES-128-GCM AXI4-Stream RTL"]
    GCM -->|AXI DMA S2MM, 128-bit| DDR
    A53 -->|AXI4-Lite control| GCM
```

Data path: PS DDR → AXI DMA → AES-GCM → AXI DMA → PS DDR. The processor supplies the key, 96-bit IV, direction and length through AXI4-Lite registers. On decryption, software checks the hardware authentication decision before transmitting plaintext. A rejected output buffer is zeroized.

## Measured CTR-DMA versus GCM-DMA

| Metric | AES-CTR DMA | AES-GCM DMA | GCM change |
|---|---:|---:|---:|
| Encryption latency | 2,186 µs | 3,010 µs | +37.7% |
| Encryption throughput | 89,939 KB/s | 65,318 KB/s | −27.4% |
| Decryption latency | 2,175 µs | 2,999 µs | +37.9% |
| Decryption throughput | 90,394 KB/s | 65,557 KB/s | −27.5% |
| LUTs | 17,016 | 19,593 | +15.1% |
| Flip-flops | 11,129 | 12,587 | +13.1% |
| BRAM equivalent | 5 × 36K | 5 × 36K | 0% |
| DSPs | 0 | 0 | 0 |
| WNS at 75.002 MHz | +1.304 ns | +0.187 ns | Timing met |
| Vivado estimated power | 3.539 W | 3.625 W | +2.4% |
| Ciphertext tamper detection | No | Yes | Security gain |

![DMA throughput comparison](docs/images/dma_throughput_comparison.png)

![Security-property comparison](docs/images/security_property_comparison.png)

## Cortex-A53 software AES-GCM baseline

A separate software-only benchmark was run on the ZCU104 Cortex-A53 using Mbed TLS 3.6.7. Each operation processed a 4,096-byte payload with AES-128-GCM, a 96-bit IV, 16-byte AAD and a 16-byte authentication tag. The benchmark used 5 warm-up iterations and 100 measured iterations, timed with the ARM generic counter at 100 MHz.

| Metric | Software encryption | Software decryption |
|---|---:|---:|
| Average latency (UART-reported) | 1,681 µs | 1,680 µs |
| Average counter ticks | 168,104 | 168,069 |
| Throughput | 2,379 KB/s | 2,379 KB/s |
| Functional verification | PASS | PASS |

Encryption/decryption recovery passed, and a modified authentication tag was rejected. See [CPU_SOFTWARE_BASELINE.md](CPU_SOFTWARE_BASELINE.md) and [the UART transcript](results/a53_software_gcm/a53_software_gcm_uart_transcript.txt).

**Measurement-scope note:** this CPU benchmark times Mbed TLS AES-GCM operations on a 4 KiB payload. The FPGA protected-chunk measurements include protected-buffer handling, GCM preparation, DMA/replay and zeroization for encryption; FPGA decryption is split into authentication-only and release phases. These timing boundaries are different, so these results should not be presented as a controlled CPU-versus-FPGA speedup.

## Image-security measurements

Across the 10-image AES-GCM dataset:

- mean combined ciphertext entropy: **7.999052 bits/byte**;
- mean horizontal, vertical and diagonal correlations: **−0.000839**, **−0.001305**, and **+0.001588**;
- same preprocessed image with two fresh IVs: **NPCR 99.593099%**, **UACI 33.458699%**;
- all 11 IVs, tags and ciphertext hashes were unique.

These image statistics describe the measured ciphertext. Cryptographic correctness is supported more strongly by the standard RTL tests, exact Python AESGCM agreement, unique-IV checks, and authenticated tamper-rejection experiment.

![GCM statistical results](docs/images/gcm_statistical_results.png)

## Repository layout

```text
hardware/
  rtl/                         Original AXI4-Lite CTR baseline
  gcm_dma/rtl/                 AES, GHASH and AES-GCM AXI4-Stream RTL
  gcm_dma/sim/                 NIST/reference RTL testbenches
  gcm_dma/scripts/             Self-contained Vivado 2026.1 flow
  gcm_protected_chunk/         Protected-chunk BRAM architecture and Vivado flow
software/
  vitis/                       Original fixed-image baseline
  vitis_gcm/single_image/      Dynamic authenticated-image application
  vitis_gcm/batch/             Multi-image benchmark application
  vitis_gcm/protected_chunk/   Cortex-A53 protected-chunk bare-metal application
host/gcm/                      Python single-image and batch tools
host/gcm/protected_chunk/      CHK3 32-byte header framing and host test tools
results/
  ctr_dma_run1/                Preserved CTR-DMA evidence
  gcm_batch_run1/              Preserved GCM evidence and reports
  comparison/                  Recomputed JSON summary and evidence hashes
docs/                          Security model and paper experiment plan
tools/analyze_results.py       Reproduces tables, hashes and figures
```

## Protected-chunk BRAM optimization

The protected-chunk co-design milestone eliminates external DDR release before authentication tag verification. Captured chunk ciphertext is held in an on-chip buffer (`ProtectedChunkMemory_BRAM.v`) and replayed for authentication-only calculation while output DMA is disarmed. Decryption and DMA plaintext release occur only after authentication passes.

The before-and-after Vivado utilization reports show the protected buffer wrapper decreasing from **58,392 to 371 LUTs**, from **33,421 to 238 FFs**, and from **0 to 2 RAMB36 blocks**. In the optimized hierarchy, the memory child uses 86 LUTs and 2 RAMB36 blocks. The full-design totals in that report pair are 78,133 and 20,112 LUTs; these are report totals, not a claim that every change in the design came from the buffer refactor. Physical ZCU104 testing passed at 17×19 (1/1), 256×256 (48/48), and 512×512 (192/192) chunks. The paired evidence and scope notes are in [the BRAM comparison folder](results/protected_chunk_bram_comparison/).

For full architectural, protocol, and resource details, see [docs/PROTECTED_CHUNK.md](docs/PROTECTED_CHUNK.md).

## Visual Demonstration — Original, Encrypted and Recovered Images

The following outputs were captured from the ZCU104 protected-chunk FPGA implementation. The recovered images demonstrate reconstruction after authenticated decryption.

### 256 × 256 RGB Image

| Original Input | Encrypted Visualization | Recovered Output |
|---|---|---|
| ![Original 256x256](results/visual_demo/256x256/input.png) | ![Encrypted 256x256](results/visual_demo/256x256/encrypted.png) | ![Recovered 256x256](results/visual_demo/256x256/recovered.png) |

### 512 × 512 RGB Image

| Original Input | Encrypted Visualization | Recovered Output |
|---|---|---|
| ![Original 512x512](results/visual_demo/512x512/input.png) | ![Encrypted 512x512](results/visual_demo/512x512/encrypted.png) | ![Recovered 512x512](results/visual_demo/512x512/recovered.png) |

### Additional Test — 17 × 19

- [Original image](results/visual_demo/17x19/input.png)
- [Encrypted visualization](results/visual_demo/17x19/encrypted.png)
- [Recovered image](results/visual_demo/17x19/recovered.png)

### Raw Evidence

Each test directory also contains the raw ciphertext, recovered plaintext, protected-chunk JSON summary and UART execution transcript.

See [all visual demonstration evidence](results/visual_demo/).

**Note:** `encrypted.png` is a visualization of encrypted data. The actual encrypted byte stream is `ciphertext.bin`.

## Reproduce the analysis

```powershell
py -m pip install -r requirements-analysis.txt
py tools\analyze_results.py
```

The script reads the preserved raw files, recomputes the comparison, creates SHA-256 checksums for the evidence, and regenerates the figures. It does not alter the raw measurements.

## Build and run

Follow [BUILD_AND_RUN.md](BUILD_AND_RUN.md). The short sequence is:

1. Run the RTL validation testbench.
2. Run the three Vivado scripts under `hardware/gcm_dma/scripts` (or `hardware/gcm_protected_chunk/scripts`).
3. Create a Vitis platform from the exported XSA.
4. Build the Cortex-A53 application.
5. Start the matching Python host tool before launching the Vitis application.
6. Require `OVERALL: PASS` before accepting a run.

## Documentation

- [Detailed measured results](RESULTS.md)
- [Cortex-A53 software baseline](CPU_SOFTWARE_BASELINE.md)
- [Software benchmark UART transcript](results/a53_software_gcm/a53_software_gcm_uart_transcript.txt)
- [Build and run guide](BUILD_AND_RUN.md)
- [AES-GCM protected-chunk milestone](docs/PROTECTED_CHUNK.md)
- [Security model and limitations](docs/SECURITY_MODEL.md)
- [Paper experiment roadmap](docs/PAPER_EXPERIMENT_PLAN.md)
- [Machine-readable verified summary](results/comparison/verified_summary.json)

## Current limitations

- The proof of concept supports 256 × 256 RGB888 payloads (up to 512 × 512 on protected-chunk); it is not a DICOM parser.
- Additional authenticated data (AAD) limitations apply to the baseline `gcm_dma` path; the protected-chunk path incorporates CHK3 AAD framing.
- IVs are randomly generated by the host; production use needs persistent uniqueness enforcement for every key.
- The demonstration key is not stored in a secure hardware key vault.
- DDR plaintext release before tag verification applies to the baseline `gcm_dma` path; the protected-chunk path holds unauthenticated ciphertext in on-chip BRAM and releases plaintext only after authentication succeeds.
- Power values are vectorless Vivado estimates with medium confidence, not physical rail measurements.
- The present dataset has 10 images; a publication study should use a larger, licensed medical-image dataset and repeated trials.

## Academic scope

This repository is a verified engineering research prototype, not a clinical security product. The included NIST/demo key is intentionally public and must never be used as a production secret. Do not publish private patient images, real secret keys, generated Vivado workspaces, bitstreams, XSA files, or unrelated local logs. Only redistribute sample images whose licensing permits public use.

## License

See [LICENSE](LICENSE).


