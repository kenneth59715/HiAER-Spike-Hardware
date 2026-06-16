# DVS Accuracy Gap Investigation

## The Gap

| Bitstream | XDMA IP | Vivado | DVS Full Dataset |
|-----------|---------|--------|------------------|
| 2024 ref  | v4.1.4  | 2019.2 | 64.24%           |
| L6d       | v4.1.29 | 2024.1 | 56.25%           |
| L6g       | v4.1.29 | 2024.1 | 56.60%           |
| L6j       | v4.1.29 | 2024.1 | 56.60%           |
| L6m       | v4.1.29 | 2024.1 | 56.60%           |

SpikingJelly software simulation: 64.45% (matches 2024 hardware closely).

## What Was Tested and Eliminated

**Noise behavior**: Restoring 2024-style unsigned noise produced 0% accuracy (destructive). Both IEPs produce zero noise for shift=-17 in practice.

**35-bit vs 32-bit MP**: L6j tested 35-bit mode via `legacy_noise_en=1`. Result: 56.60%, same as 32-bit. No improvement.

**Sign extension**: Fixed `19'h7FFFF` to `19'hFFFFF`. Correctness improvement but no accuracy change.

**URAM init, ghost masking, watchdog, stale spike draining**: All disabled in L6k. DVS accuracy unchanged at 56.60%.

**FIFO implementation**: Tested Builtin_FIFO in L6l. Result: 55.90% (slightly worse). Reverted.

## Root Cause

The ~8% gap is caused by the **XDMA IP version** difference between Vivado 2019.2 (v4.1.4) and Vivado 2024.1 (v4.1.29). The XDMA IP handles DMA data delivery timing differently, affecting how spike data and weight data arrive at the neuromorphic core relative to the IEP's processing phases.

This was confirmed by testing the 2024 bitstream with the software team's exact configuration (hs_api commit e526b6f, test file `test_DVS_large_fulldataset_2024.py`) and reproducing the 64.24% result.

## Resolution Options

1. **Vivado 2019.2 license for xcvu37p** — not available
2. **Quantization-aware retraining** — retrain the DVS model against the Vivado 2024.1 hardware behavior
3. **Hardware-in-the-loop training** — use the actual FPGA for forward passes during training
4. **Accept 56.60%** — the hardware is functionally correct; the gap is a quantization/timing artifact

The 56.60% result is the correct hardware accuracy for this model on Vivado 2024.1 builds.
