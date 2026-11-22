# Cortex-A53 Software AES-128-GCM Baseline

## 1. Objective
Measure software AES-128-GCM encryption and decryption on the ZCU104 Cortex-A53. This benchmark provides a software reference measurement; it is not a controlled performance comparison against the FPGA implementation because the payload sizes and timing boundaries differ.

## 2. Test Configuration

| Parameter | Configuration |
|---|---|
| Board | AMD/Xilinx ZCU104 |
| Processor | ARM Cortex-A53 |
| Software library | Mbed TLS 3.6.7 |
| Algorithm | AES-128-GCM |
| Key size | 128 bits |
| IV size | 96 bits |
| AAD size | 16 bytes |
| Authentication tag | 16 bytes |
| Payload per operation | 4096 bytes |
| Warm-up iterations | 5 |
| Measured iterations | 100 |
| Timer | ARM generic counter |
| Counter frequency | 100 MHz |

## 3. Measured Results

| Metric | Encryption | Decryption |
|---|---:|---:|
| Average latency | 1681.04 us | 1680.69 us |
| Throughput | 2379 KB/s | 2379 KB/s |

Mean counter ticks:
- Encryption: 168104 ticks
- Decryption: 168069 ticks

## 4. Functional Verification

| Test | Result |
|---|---|
| AES-GCM encryption | PASS |
| AES-GCM decryption | PASS |
| Plaintext recovery | PASS |
| Modified authentication tag rejection | PASS |

## 5. Comparison Notes

The CPU baseline measures Mbed TLS software AES-GCM operations on a 4096-byte payload.

The FPGA protected-chunk implementation uses DMA and a protected-buffer workflow. Its encryption timing includes buffer capture, GCM preparation, DMA processing, replay, and zeroization. FPGA decryption is measured in separate authentication-only and release phases.

Therefore, the CPU and FPGA measurements have different timing boundaries. Their latency values should not be presented as a direct, fully controlled speedup comparison.

## 6. Reproducibility

The benchmark uses 5 warm-up iterations followed by 100 measured iterations. Timing is performed using the ARM generic counter, with UART output outside the measured operation.

The benchmark uses a demonstration key. It is intended for performance evaluation and not production cryptographic key management.
