# BRAM optimization utilization comparison

These Vivado hierarchical utilization reports compare the protected chunk design before and after BRAM inference.

- `pre_bram_utilization.rpt`: protected buffer wrapper uses 58,392 LUTs, 33,421 FFs, and 0 RAMB36; full design uses 78,133 LUTs and 46,149 FFs.
- `bram_optimized_utilization.rpt`: protected buffer wrapper uses 371 LUTs, 238 FFs, and 2 RAMB36; full design uses 20,112 LUTs and 12,966 FFs.
- The optimized memory child uses 86 LUTs and 2 RAMB36. The wrapper's other logic uses 285 LUTs.
- `SHA256SUMS.txt` contains SHA-256 hashes for both reports.

The separate `system_mem_smc` utilization is reported in the final design; these reports do not attribute its LUT count to the BRAM optimization.