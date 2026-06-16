# Lessons Learned

Critical lessons from months of debugging. Read before making any changes.

## 1. Never Mix Files from Different Git Versions

The hs_api, hs_bridge, and connectome_utils repos have interdependent APIs. Mixing a test file from commit A with an api.py from commit B causes subtle failures (wrong connection formats, missing methods, changed fixture signatures). The testing-suite branch test file (e526b6f) is incompatible with the fb811b4 api.py for refractory and negative-input tests.

**Rule**: Always restore the COMPLETE working state from `/home/omowuyi/L6j_working_software/`. Never cherry-pick individual files.

## 2. Always Backup BEFORE Switching Branches

The L6j backup was captured AFTER switching to testing-suite branch, losing the actual working state. The correct configuration had to be reconstructed from the L6d backup + patches, which took many hours.

**Rule**: Backup first, verify it works, THEN make changes.

## 3. Single Quotes in neuron_models.py

The L6d backup uses single quotes in `getattr`: `getattr(self, 'delay_value', 0)`. Patches searching for double quotes fail silently, methods are never added, and all tests fail with AttributeError.

**Rule**: Use `grep` to check quote style before writing any patch.

## 4. XDMA IP Version Determines DVS Accuracy

The ~8% DVS gap (64.24% to 56.60%) is caused by XDMA v4.1.4 (Vivado 2019.2) vs v4.1.29 (Vivado 2024.1). This was confirmed by exhaustively testing every IEP behavioral difference (noise, 35-bit MP, sign extension, URAM init, ghost masking, watchdog, FIFO type). None closed the gap.

**Rule**: Accept 56.60% as the ceiling for Vivado 2024.1 builds. Improving requires a Vivado 2019.2 license (unavailable for xcvu37p) or model retraining.

## 5. Software Delay Timing: max(0, dv-1)

FPGA Phase 2 delivers weights AFTER Phase 0 threshold check. A delayed spike must be injected one step before the expected spike time. The delay queue countdown uses `max(0, dv-1)`.

## 6. coreID Encoding Must Match tdest Generator

The `pcie_tdest_generator` reads specific bits from byte 1 of DMA packets. ALL fpga_controller.py functions must place coreID in the same bit positions. A mismatch routes programming commands to the wrong core while execution goes to the right one.

**Current standard**: coreID in bits [499:496] (lower 4 bits of byte 1). Software uses `'000'+np.binary_repr(coreID,5)`.

## 7. Parallel Multi-Process FPGA Access Deadlocks

Multiple processes reading/writing `/dev/adxdma0*` simultaneously deadlock. The `switch_16_1` merges all responses — process A's DMA read can consume process B's response.

**Rule**: Test cores sequentially, or build a single-process orchestrator.

## 8. Reboot After Failed Parallel Tests

Zombie pytest processes hold the FPGA in a bad state. Always `sudo reboot` crisdsc0 before retesting after any parallel execution failure.

## 9. Build vs Test Machine Separation

crisdsc2 = Vivado builds ONLY (no FPGA hardware). crisdsc0 = testing ONLY (no Vivado). Bitstreams transfer via `scp crisdsc2:path crisdsc0:/bitstreams/`.

## 10. PCIe Re-enumeration After Every Flash

```bash
sudo /bitstreams/scripts/flash.sh /bitstreams/<bitstream>.bit
sleep 3
echo 1 | sudo tee /sys/bus/pci/rescan
sleep 3
sudo modprobe adxdma
echo "4144 0902" | sudo tee /sys/bus/pci/drivers/adxdma/new_id
sudo chmod 666 /dev/adxdma0*
```
