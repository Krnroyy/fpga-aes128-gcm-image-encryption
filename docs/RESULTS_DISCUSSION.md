# Results and Discussion

This discussion interprets the measurements recorded in [RESULTS.md](../RESULTS.md). It distinguishes functional security evidence, baseline performance cost, protected-chunk implementation results and the separate Cortex-A53 software benchmark.

## 1. Functional correctness and authenticated release

The baseline GCM-DMA batch processed 11 transactions. FPGA ciphertext and 128-bit authentication tags matched Python AESGCM for all 11 transactions, and all 11 valid decryptions recovered the original input exactly. All 11 modified-ciphertext cases were rejected; all rejected output buffers were zeroized; firmware released unauthenticated plaintext in 0/11 cases.

This is stronger evidence of functional correctness than image statistics alone because it compares exact cryptographic outputs against an independent implementation and exercises an adversarial ciphertext modification.

The protected-chunk path extends the release policy. Its board tests passed for 1, 48 and 192 chunks at 17Ã—19, 256Ã—256 and 512Ã—512 image sizes, respectively. Authentication-only processing runs while S2MM is disarmed; only a valid tag permits replay for decryption and plaintext DMA release. The protected-chunk tests also cover modified AAD. These claims apply to the protected-chunk implementation and should not be retroactively attributed to the baseline GCM-DMA path.

## 2. Cost of authentication relative to CTR-DMA

The baseline comparison quantifies the additional cost of GCM relative to CTR-DMA for 196,608-byte images:

| Metric | CTR-DMA | GCM-DMA | Change |
|---|---:|---:|---:|
| Encryption latency | 2,186 Âµs | 3,010 Âµs | +37.7% |
| Encryption throughput | 89,939 KB/s | 65,318 KB/s | âˆ’27.4% |
| Decryption latency | 2,175 Âµs | 2,999 Âµs | +37.9% |
| Decryption throughput | 90,394 KB/s | 65,557 KB/s | âˆ’27.5% |
| LUTs | 17,016 | 19,593 | +15.1% |
| Flip-flops | 11,129 | 12,587 | +13.1% |
| BRAM equivalent | 5 Ã— 36K | 5 Ã— 36K | No change |
| DSPs | 0 | 0 | No change |

GCM's latency and resource overhead is the measured engineering trade-off for authenticated encryption. CTR alone provides confidentiality but not integrity: the controlled CTR tamper test demonstrated that a ciphertext modification changes the corresponding recovered plaintext without an authentication failure. GCM's tag rejects the tested modification.

The throughput decrease should not be interpreted as an unexplained inefficiency alone: GCM performs GHASH-based authentication in addition to AES counter-mode processing. The reported numbers describe this implementation, board, payload size and measurement setup; they are not universal AES-GCM performance figures.

## 3. Protected-chunk architecture and BRAM optimization

The protected-chunk design changes where unauthenticated data is held. Rather than decrypting into a DDR destination before the authentication decision, it captures ciphertext into locked on-chip BRAM, authenticates with S2MM disarmed, and replays the same bytes for decryption only after tag validation.

The protected-buffer wrapper utilization comparison reports a reduction from 58,392 to 371 LUTs and from 33,421 to 238 flip-flops, while using two RAMB36 blocks. The optimized memory child uses 86 LUTs and two RAMB36 blocks. These values describe the wrapper hierarchy.

The paired full-design totals also changed from 78,133 to 20,112 LUTs and from 46,149 to 12,966 flip-flops. Those are report totals and must not be attributed solely to the buffer refactor. The before/after archive contains utilization reports only, so it does not establish a timing or power improvement caused by BRAM optimization.

## 4. Final protected-chunk implementation PPA

The final protected-chunk implementation report records:

- 20,112 total CLB LUTs (8.73%) and 12,966 flip-flops (2.81%);
- 6 RAMB36 and 2 RAMB18 blocks, with no URAM or DSP;
- WNS +0.975 ns, TNS 0 ns, WHS +0.002 ns and THS 0 ns;
- 42,054 timing endpoints with zero setup/hold failing endpoints;
- 24,970 fully routed nets and zero routing errors.

The critical path is reported at 12.122 ns from `key_reg3_reg[13]/C` to `ciphertext_reg[124]/D`. Routing accounts for 8.350 ns (68.883%) and logic for 3.772 ns (31.117%), across 45 logic levels. This indicates that interconnect is the larger component of this reported path and is a reasonable target for future timing optimization. It does not by itself prove that a particular RTL rewrite will improve timing.

The final Vivado power estimate is 3.602 W (2.909 W dynamic, 0.693 W static; medium confidence). It is vectorless, uses no activity file and includes an estimated 2.680 W PS8 contribution. It is neither measured board power nor PL-only power. Physical rail measurements and activity-informed estimates would be needed for a stronger energy-efficiency claim.

## 5. Cortex-A53 software benchmark: what can and cannot be concluded

The Mbed TLS 3.6.7 software benchmark measured 1681.04 Âµs encryption and 1680.69 Âµs decryption for a 4096-byte payload, at 2379 KB/s for each operation. Functional recovery and modified-tag rejection passed.

These values are useful as a reproducible software reference point, but they cannot be compared directly with the FPGA image measurements to claim a speedup. The software payload is 4 KiB, whereas the baseline FPGA image transaction is 196,608 bytes. In addition, the software timer covers the Mbed TLS operation, while the protected-chunk FPGA timing includes buffer handling and DMA/replay stages, with decryption divided into authentication and release phases.

A valid head-to-head comparison would require the same payload sizes, key/IV/AAD/tag configuration, warm-up policy, cache policy, and explicitly matched timing boundaries. The next experiment should time software and FPGA paths over identical buffers and report both kernel-only and end-to-end measurements.

## 6. Ciphertext statistics

The 10-image dataset produced mean combined ciphertext entropy of 7.999052 bits/byte and near-zero mean horizontal, vertical and diagonal correlations. Encrypting the same preprocessed image with two different IVs produced NPCR 99.593099% and UACI 33.458699%.

These values characterize the measured ciphertext distribution and IV sensitivity. They are not a proof of AES-GCM security, do not replace standard test vectors, and do not demonstrate a plaintext-avalanche property. The exact Python reference agreement and authenticated tamper tests are the more direct functional evidence.

## 7. Limitations and next experiments

The main limitations are experimental breadth and measurement comparability, not a missing core architecture:

1. Expand the dataset from 10 images to a licensed, citable 50â€“100 image set.
2. Run at least three independent board passes per image and preserve raw timer ticks.
3. Report mean, standard deviation, range and 95% confidence intervals.
4. Compare ARM software, CTR-DMA and GCM-DMA with identical payloads and matched timing boundaries.
5. Add wrong-key, malformed packet, replay/IV-reuse, additional AAD-field and tag-bit-position tests.
6. Measure physical board-rail power where possible; otherwise retain the vectorless-estimate label.
7. Report energy per byte and per image only when power and timing use a clearly defined common boundary.

## 8. Defensible conclusion

The measurements support the following bounded conclusion:

> A ZCU104 prototype demonstrated AES-128-GCM image encryption with exact ciphertext and tag agreement against Python AESGCM, successful authenticated recovery, and rejection of the tested ciphertext modifications. The protected-chunk implementation additionally authenticated its tested CHK3 metadata and withheld plaintext DMA release until tag verification succeeded. Relative to the measured CTR-DMA baseline, GCM incurred approximately 27% lower throughput and approximately 15% more LUT usage in the recorded implementation. The final protected-chunk design met the reported 75.002 MHz timing target. Further dataset expansion, repeated runs, matched software/FPGA timing boundaries and physical power measurements are needed for publication-grade performance claims.

For experiment details and evidence paths, see [EXPERIMENT_METHODOLOGY.md](EXPERIMENT_METHODOLOGY.md) and [RESULTS.md](../RESULTS.md).

