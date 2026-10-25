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

## Offline ZCU104 implementation results

Built for `xczu7ev-ffvc1156-2-e` with Vivado 2026.1. Bitstream generation completed with zero DRC errors. All 24,970 routable nets were routed with zero routing errors.

The design met its 75 MHz timing constraint: setup WNS was +0.975 ns and hold WHS was +0.002 ns, with no failing endpoints. Hierarchical utilization reported 371 LUTs, 238 FFs, and 2 RAMB36 blocks for the protected chunk buffer; the AES-GCM block used 12,931 LUTs and 3,803 FFs.

Vivado estimated total on-chip power at 3.602 W using vector-less activity. The report warns that reset activity may make this estimate inaccurate. Physical-board validation and power measurement are pending access to the ZCU104.

## Post-route hold-path review

Reviewed the five worst hold paths from the routed Vivado implementation using `report_timing -delay_type min -max_paths 5 -sort_by slack`. All five were reported as MET. The worst path had +0.002 ns slack at the slow process corner; the other four were +0.004 ns, +0.004 ns, +0.006 ns, and +0.007 ns.

The worst path runs from an AXI DMA S2MM skid-buffer register to a memory subsystem write skid-buffer register. It is in the generated DMA/memory infrastructure, not the protected-chunk or AES-GCM RTL. The margin is small but positive in this implementation report; no RTL change was made. Board validation remains pending access to the ZCU104.

### Device-level utilization

For the routed `system_wrapper` design on `xczu7ev-ffvc1156-2-e`, Vivado 2026.1 reported: CLB LUTs 20,112/230,400 (8.73%); CLB registers 12,966/460,800 (2.81%); block RAM tiles 7/312 (2.24%), comprising 6 RAMB36 and 2 RAMB18; URAM 0/96; DSPs 0/1,728. These are whole-design totals, including the processing system, DMA, interconnect, and custom logic. The RAMB36, RAMB18, and block RAM tile percentages are separate report views and should not be added together.
