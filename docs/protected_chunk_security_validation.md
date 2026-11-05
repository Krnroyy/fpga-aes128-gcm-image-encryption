# Protected-Chunk RTL and Implementation Validation

This page separates three kinds of evidence that apply to related, but distinct, design snapshots: protected-chunk RTL simulation, the base protected-chunk physical board runs, and the later PCF cancellation candidate implementation.

## Protected-chunk RTL simulation

The Vivado validation script runs two simulations:

1. `tb_ProtectedChunkBuffer_AXIS` checks capture, locking, blocked writes, replay, zeroization, malformed chunks, and reset scrubbing.
2. `tb_AES_GCM_ProtectedChunk` checks the authenticated two-pass AES-GCM flow, including an attempted overwrite between authentication and release.

From the repository root, run the script in the Vivado Tcl Console:

```tcl
set repo {C:/path/to/fpga-aes128-gcm-image-encryption}
set script [file join $repo hardware gcm_protected_chunk sim run_protected_chunk_validation_2026_1.tcl]
if {![file exists $script]} { error "Validation script not found: $script" }
source $script
```

The integration test holds an overwrite attempt valid for three clock cycles after authentication. On each checked rising edge, it requires `capture_tready` and the internal `memory_write_en` to be low. This is a directed RTL simulation check, not a formal proof. No mutation-test result is claimed here because its raw log is not included in the repository.

## Base protected-chunk physical board evidence

The archived ZCU104 board summaries and UART transcripts record passing runs for 17×19 (1/1 chunk), 256×256 (48/48), and 512×512 (192/192). Each summary records exact full-image recovery and per-chunk tamper/AAD rejection checks. See [`results/protected_chunk_board_evidence/`](../results/protected_chunk_board_evidence/).

These results apply to the base protected-chunk board test runs represented by these records. They do not establish board validation of the separate cancellation candidate described below.

## Separate PCF cancellation-candidate implementation

The reports in [`research/pcf/evidence/cancel_candidate_implementation/`](../research/pcf/evidence/cancel_candidate_implementation/) describe a later cancellation-candidate implementation, not the base protected-chunk BRAM comparison. Its routed reports record:

- Device `xczu7ev-ffvc1156-2-e`, Vivado 2026.1, PL clock 75.002 MHz.
- Setup WNS +0.846 ns and hold WHS +0.010 ns, with no failing setup or hold endpoints.
- 20,249 LUTs and 12,972 FFs for the full design; 6 RAMB36 and 2 RAMB18.
- 427 LUTs, 239 FFs, and 2 RAMB36 for the protected-buffer wrapper.
- Estimated on-chip power 3.590 W (2.897 W dynamic, 0.693 W static), with medium confidence. This is a Vivado estimate, not a physical board power measurement.
- 25,324 of 25,324 routable nets fully routed, with zero routing errors.

The repository's PCF snapshot says physical board validation of this cancellation candidate is not established. Earlier base protected-chunk board passes must not be attributed to the candidate. See [`research/pcf/README.md`](../research/pcf/README.md).

## BRAM comparison scope

The archived before/after utilization reports for the BRAM optimization are in [`results/protected_chunk_bram_comparison/`](../results/protected_chunk_bram_comparison/). They support the wrapper-level LUT/FF/BRAM comparison. That folder does not contain paired timing or power reports, so it does not support a before/after timing or power claim. The cancellation-candidate timing and power reports above are for a distinct implementation snapshot and are not substitutes for those missing paired reports.
