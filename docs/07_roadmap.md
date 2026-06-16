# Roadmap — From 16-Core to 40-FPGA Cluster

## Current State

L6m is the verified single-core baseline (42/42 tests, 56.60% DVS). The 16-core multicore_2 bitstream (with tdest routing fix) is building on crisdsc2. After multicore_2 validation, the development proceeds through three phases.

## Phase 1: Validate multicore_2 (16-core, current)

**Goal**: All 16 cores pass 42/42 hardware tests + DVS on core 0.

After `sixteen_core_top_multicore_2.bit` build completes:
```bash
scp multi_core.runs/impl_1/sixteen_core_top.bit crisdsc0:/bitstreams/sixteen_core_top_multicore_2.bit
cd /home/omowuyi
bash run_all_tests_multicore.sh
```

The test script runs cores 0-15 sequentially (one at a time) because the shared PCIe/XDMA link has only 1 H2C + 1 C2H channel. The `switch_16_1` merges all core responses, so parallel processes deadlock.

**Success criteria**: 42/42 on all 16 cores, DVS 56.60% on core 0.

## Phase 2: NoC Integration (Inter-Core Communication)

**Goal**: Enable spike routing between cores on the same FPGA.

The NoC module (`cores_with_noc.v`, designed by Paresh Kurdekar Vasanth) connects between the EEP of each core and a shared routing fabric. When a neuron on core A spikes and has a post-synaptic target on core B, the spike is routed through the NoC instead of going back through the host.

**RTL integration points**:
- Each core's EEP already has NoC FIFO ports (added in L6d): `noc_rx_data`, `noc_rx_valid`, `noc_tx_data`, `noc_tx_ready`
- The `external_events_processor_simple.v` has BRAM write multiplexing for PCIe and NoC event sources
- After Phase 2 completes, the EEP checks if the NoC FIFO is empty before deciding Phase 3 vs loop back

**Software changes needed**:
- Network compiler must partition neurons across cores and generate inter-core routing tables
- `connectome_utils` partitioning library (currently disabled: "partitioning library failed to load, multicore disabled")
- Each core's HBM stores only its local synapses; remote spikes routed via NoC

**Validation**: Run 42 hardware tests on each core individually, then run a multi-core network that requires inter-core spike routing.

**Resource headroom**: With 16 cores at ~65% LUT, ~72% BRAM, ~27% URAM, there is room for NoC (~40-50K LUTs, ~30 BRAM).

## Phase 3: Firefly Inter-FPGA Communication

**Goal**: Enable spike routing between FPGAs within a server.

**Hardware**: 8 FPGAs per server, 5 servers = 40 FPGAs total.

Firefly uses Aurora IP over GTY transceivers for high-speed serial links between FPGAs. Each FPGA-to-FPGA link carries serialized spike packets with source/destination core addressing.

**Architecture**:
```
Server 1 (8 FPGAs, 128 cores)
  FPGA 0 ←→ FPGA 1 ←→ ... ←→ FPGA 7
    │           │                  │
    16 cores    16 cores          16 cores
    each        each              each

Server 2 (8 FPGAs, 128 cores)
  ...

Server 5 (8 FPGAs, 128 cores)
  ...

Total: 40 FPGAs × 16 cores = 640 neuromorphic cores
```

**RTL integration**:
- Aurora IP instantiation (8 GTY transceivers per FPGA)
- Firefly packet format: source_fpga, source_core, dest_fpga, dest_core, spike_address
- Routing table in each FPGA: local spikes go to NoC, remote spikes go to Firefly TX
- Firefly RX feeds into NoC for delivery to local cores

**Resource requirement**: ~20-30K LUTs, ~20 BRAM, 8 GTY transceivers per FPGA. Fits within remaining headroom after 16 cores + NoC.

**Software changes**:
- Global network partitioner: assigns neuron groups to specific FPGA/core pairs
- Inter-FPGA routing table generation
- Distributed execution coordinator

## Phase 4: 5-Server Cluster (Future)

**Goal**: Full 40-FPGA distributed neuromorphic system.

Inter-server communication uses Ethernet or InfiniBand between server NICs, with a lightweight protocol for spike packet forwarding. This adds latency compared to intra-server Firefly links, so the network partitioner should minimize cross-server traffic.

## Resource Utilization Summary

| Configuration     | LUTs       | BRAM       | URAM      | GTY  |
|-------------------|------------|------------|-----------|------|
| 1 core            | 100K (8%)  | 149 (7%)   | 16 (2%)   | 0    |
| 16 cores          | ~700K (54%)| ~1340 (66%)| 256 (27%) | 0    |
| 16 cores + NoC    | ~750K (58%)| ~1370 (68%)| 256 (27%) | 0    |
| 16 cores+NoC+Fire | ~780K (60%)| ~1390 (69%)| 256 (27%) | 8    |
| Available (VU37P) | 1,303K     | 2,016      | 960       | 32+  |
