# 16-Core Multicore Design

## Data Path

```
Host PCIe ──> XDMA ──> pcie2fifos ──> async_fifo ──> pcie_tdest_generator
                                                            │
                                                      tdest = tdata[499:496]
                                                            │
                                                      switch_1_16
                                               ┌───────────┴───────────┐
                                            Core 0               Core 15
                                              │                      │
                                          HBM Port 0            HBM Port 15
                                              │                      │
                                            Core 0               Core 15
                                              │                      │
                                          switch_16_1
                                               │
                                  pcie2fifos ──> XDMA ──> Host PCIe
```

## Command Routing

Every DMA packet is 512 bits (64 bytes). Byte 1 (bits [503:496]) contains the coreID. The `pcie_tdest_generator` extracts the lower 4 bits:

```verilog
module pcie_tdest_generator(
    AXIStream_simple.Slave s,
    AXIStream.Master m
);
    assign m.tdest = {1'b0, s.tdata[499:496]};  // 4-bit coreID
endmodule
```

Software encodes coreID consistently:
```python
# Bit-array functions (clear, execute, flush_spikes):
coreBits = '000' + np.binary_repr(coreID, 5)  # coreID in bits [499:496]

# Byte-array functions (write_neuron_type, write_synapse_row):
commandPrefix = [2, coreID] + [0]*27           # coreID as raw byte value
```

Both place coreID in bits [499:496], matching the tdest generator.

## Bitstream Versions

**multicore_1** (`sixteen_core_top_multicore_1.bit`): tdest reads `tdata[503:499]` and software Format A uses `np.binary_repr(coreID,5)+'000'`. Works for core 0, fails for cores 1-15 because programming commands (Format B at [499:496]) route to wrong core.

**multicore_2** (`sixteen_core_top_multicore_2.bit`): tdest reads `{1'b0, tdata[499:496]}` and software uses `'000'+np.binary_repr(coreID,5)`. All functions align on [499:496].

## Sequential Per-Core Testing

```bash
for i in $(seq 0 15); do
    echo "=== Testing Core $i ==="
    HIAER_CORE_ID=$i pytest test_bitstream_hardware_fast.py -v
done
```

## Resource Utilization

| Resource   | 1 Core      | 16 Cores (est.) | Available   | Headroom |
|------------|-------------|-----------------|-------------|----------|
| LUTs       | 100K (8%)   | ~700K (54%)     | 1,303K      | 46%      |
| Registers  | 78K (3%)    | ~830K (32%)     | 2,607K      | 68%      |
| BRAM       | 149 (7%)    | ~1,340 (66%)    | 2,016       | 34%      |
| URAM       | 16 (2%)     | 256 (27%)       | 960         | 73%      |

32 cores would NOT fit (LUTs overflow). 16 is the maximum per FPGA.
