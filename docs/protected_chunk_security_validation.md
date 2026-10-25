# Protected Chunk Security Validation

## Purpose

This simulation checks that ciphertext authenticated by the AES-GCM engine cannot be overwritten before the later decrypt and release pass. The protected chunk buffer locks the captured bytes after capture.

## Test procedure

The Vivado validation script runs two simulations:

1. `tb_ProtectedChunkBuffer_AXIS` checks capture, locking, blocked writes, replay, zeroization, malformed chunks, and reset scrubbing.
2. `tb_AES_GCM_ProtectedChunk` checks the authenticated two-pass AES-GCM flow, including an attempted overwrite between authentication and release.

Run it from the Vivado Tcl Console:

```tcl
set script {C:/vlsi/fpga-aes128-gcm-image-encryption-review/hardware/gcm_protected_chunk/sim/run_protected_chunk_validation_2026_1.tcl}
if {![file exists $script]} { error "Validation script not found: $script" }
source $script
## Per-cycle write-protection check

The integration test holds an overwrite attempt valid for three clock cycles after authentication. On each rising edge, it checks that \capture_tready\ and the buffer's internal \memory_write_en\ are low. It then verifies that STATUS bit 9 (\write_blocked\) is set.

## Checker sensitivity mutation test

A temporary copy of the buffer RTL was changed to assert memory_write_en when the buffer was locked and input valid was asserted. The integration test detected the injected defect at 4296 ns (TREADY=0 MEM_WRITE_EN=1) and stopped before release. This confirms the test detects this specific injected bug; it is simulation evidence, not a formal proof.
