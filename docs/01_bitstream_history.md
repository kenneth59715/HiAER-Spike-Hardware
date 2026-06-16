# Bitstream History — Every Version, Edit, and Software Configuration

This document tracks every bitstream from the 2024 reference through L6m and into multicore. For each version: what RTL changed, what software was needed, and how to reproduce tests.

---

## 2024 Reference Bitstream (August 2024)

**File**: `multi_neuron_type_param_mem_fix_08132024.bit`
**Location**: `/bitstreams/multi_neuron_type_param_mem_fix_08132024.bit` on crisdsc0
**Vivado**: 2019.2 | **XDMA IP**: v4.1.4

**Test Results**: DVS full dataset = 64.24% (hardware), 64.45% (SpikingJelly sim)

**IEP characteristics (original 2024 behavior)**:
- Unsigned `shift_param`: `reg [5:0] shift_param;`
- 35-bit membrane potential: full [34:0] of URAM used for MP
- Unsigned threshold comparison (no `$signed()`)
- No refractory period, no synaptic delay hardware, no URAM init
- No ghost neuron masking, no watchdog timer
- Leak uses arithmetic right shift (`>>>`)
- `stopAddr` encoded as raw 17-bit value in `write_neuron_type`

**Software for 2024 test**:
- hs_api: commit `e526b6fdcfea4cd0c0d2e7b16cb0134839dcd830` (testing-suite branch)
- Test file: `test_DVS_large_fulldataset_2024.py`
- Pickle: `DVS_model_config_shift=0.pkl` (109,615 neurons)

**Why this matters**: This is the accuracy target. The ~8% gap to 56.60% was traced to XDMA IP version difference (v4.1.4 vs v4.1.29), not RTL behavior.

---

## L6 (February 2025)

**HW Tests**: 42/42 (with older software branches, incompatible with current testing-suite)

Original L6 passed all tests but with older hs_api/hs_bridge branches no longer compatible with testing-suite/marchChange software.

---

## L6a

**HW Tests**: Non-functional for networks with 100+ axons (hangs after HBM DMA writes)

**Root cause**: Four bypass-related residuals from earlier bypass DMA path experiment (bypass_clk_conv, bypass_dwidth_conv IPs) corrupted DMA transfers for larger networks.

---

## L6c — Backup RTL Restored

**HW Tests**: 31/42

**RTL Changes**: Five files restored from TRANSFER_multi_core backup:
1. `internal_events_processor.v`
2. `command_interpreter.v`
3. `external_events_processor_simple.v`
4. `hbm_processor.v`
5. `single_core.sv`

**Why 31/42**: Unsigned threshold comparisons caused false spikes for negative membrane potentials.

**Software**: hs_api at ad34a7b (testing-suite), hs_bridge at 1e3a114, connectome_utils at 181f8a8 (dev). dmadump `.so` rebuilt.

---

## L6d — Signed Threshold Fix

**HW Tests**: 41/42 (`test_synaptic_delay` fails — no software delay yet)
**DVS Accuracy**: 56.25%

### IEP RTL Changes from L6c

**Change 1 — Signed threshold comparison (Phase 0, lines 852 and 880)**:
```verilog
// BEFORE (L6c): unsigned comparison
if (uram_rmwmem_upper[i][31:0] > threshold_param[31:0])

// AFTER (L6d): signed comparison
if ($signed(uram_rmwmem_upper[i][31:0]) > $signed(threshold_param[31:0]))
```
Applied to BOTH upper and lower half-word paths. Without `$signed()`, negative membrane potentials appeared as large positive values, causing false spikes.

**Change 2 — Signed shift_param declaration**:
```verilog
// BEFORE: reg [5:0] shift_param;             // unsigned
// AFTER:  reg signed [5:0] shift_param;      // signed
```
With signed declaration, `shift=-17` sets `noise_disabled=true` (zero noise). In 2024 unsigned IEP, `shift=-17` = 47 unsigned, which left-shifted noise by 30.

**Change 3 — 32-bit membrane potential mode**:
- URAM bits [31:0] = membrane potential
- URAM bits [34:32] = refractory counter (3-bit)
- URAM bit [35] = spike flag
- Leak uses `$unsigned(MP) >> leak_param` (logical shift)

**Change 4 — Added infrastructure (not yet active)**:
- Refractory period support: neuron_param_mem bits [16:14]
- Delay value register: bits [20:17]
- Dual synapse enable: bit [13]
- Shadow URAM offset: bits [12:9]
- Soft reset enable: bit [8]
- URAM initialization state (STATE_INIT_URAM)
- Ghost neuron masking, watchdog timer

### Software for L6d

- hs_api: fb811b4 commit
- hs_bridge: commit 1e3a114
- connectome_utils: dev branch, commit 181f8a8
- Backup: `/home/omowuyi/L6d_backup_software/` (files suffixed `_L6d`)
- fpga_controller.py already had 13-bit boundary encoding and parameter fields
- neuron_models.py had `get_shadow_uram_offset()` using **single quotes** in `getattr()`

**Tests fixed**: `test_LIF_neuron_negative_input`, `test_2layers_no_input`
**Still failing**: `test_synaptic_delay`

---

## L6g — Legacy Noise Enable (Bit [7])

**HW Tests**: 42/42 (first version to pass all tests)
**DVS Accuracy**: 56.60%

### IEP RTL Changes from L6d

**Change 1 — Added `legacy_noise_en_param` read from neuron_param_mem bit [7]**:
```verilog
reg legacy_noise_en_param;
// In parameter read block:
legacy_noise_en_param <= dout_neuron_param_mem[7];
```

**Change 2 — Conditional noise and MP mode**:

When `legacy_noise_en=0` (default, used by all 42 hardware tests):
- 32-bit MP in [31:0], refractory counter in [34:32]
- Signed shift: shift=-17 disables noise
- Leak: `$unsigned(MP) >> leak_param`

When `legacy_noise_en=1` (used by DVS inference):
- 35-bit MP in [34:0], no refractory counter
- Unsigned shift treatment for noise calculation
- Leak: `MP >>> leak_param` (arithmetic shift)

### Software Changes (via `patch_software_crisdsc0.py`)

- `fpga_controller.py`: Added `legacy_noise_en=0` parameter to `write_neuron_type()`, encoded at bit [7]
- `network.py`: Both write_neuron_type calls (lines 76 AND 87) pass `legacy_noise_en=neuronModel.get_legacy_noise_en()`
- `neuron_models.py`: Added to LIF_neuron and ANN_neuron: `def get_legacy_noise_en(self): return getattr(self, 'legacy_noise_en', 0)` (single quotes)
- DVS tests: shift=0 converted to shift=-17, `legacy_noise_en=1` set before CRI_network creation

---

## L6j — Sign Extension Fix

**HW Tests**: 42/42
**DVS Accuracy**: 56.60%

### IEP RTL Changes from L6g

**Change 1 — Sign extension in Phase 2**:
```verilog
// BEFORE: 19'h7FFFF  (wrong for negative weights)
// AFTER:  19'hFFFFF  (proper sign extension)
```

**Change 2 — L6d noise for all modes**: The "2024 noise restoration" attempt was abandoned after testing showed destructive results (0% accuracy with large noise enabled).

### Software

Same as L6g — no changes needed.

---

## L6k — Feature Disable Test

**HW Tests**: 33/42 | **DVS**: 56.60%

Disabled URAM reinit, ghost masking, watchdog, stale spike draining. No accuracy change confirmed these features are not the gap source.

---

## L6l — FIFO Configuration Test

**DVS**: 55.90%

Changed hbmdata_FIFO from Distributed RAM to Builtin_FIFO. Slight accuracy decrease. Reverted.

---

## L6m — Regenerated FIFO IP (Current Single-Core Baseline)

**HW Tests**: 42/42
**DVS Accuracy**: 56.60%
**File**: `sixteen_core_top_L6m.bit`

### Changes from L6j

- Same RTL logic (no behavioral changes)
- Regenerated `hbmdata_FIFO_512_wide` IP in Vivado 2024.1
- FIFO uses `srst` (synchronous reset) instead of `rst` (asynchronous reset)
- `Xilinx_IP_wrappers.sv` updated: `srst` + `wr_rst_busy` + `rd_rst_busy` connections

### Complete Software Configuration for L6m

Working state backed up at `/home/omowuyi/L6j_working_software/` on crisdsc0:

**api.py** (hs_api):
- Testing-suite infrastructure + fb811b4 test file integration
- Software delay: `_preprocess_delayed_synapses()` creates synthetic `_DELAY_` axons
- `_delay_queue` managed in `step()` with `max(0, dv-1)` timing
- Delay queue in BOTH `membranePotential==True` and `else` paths
- 3-element connection tuples: `(target_neuron, weight, delayed_flag)`

**neuron_models.py** (hs_api):
- L6d base + `get_shadow_uram_offset()` and `get_legacy_noise_en()` on ALL neuron classes
- Uses **single quotes** in `getattr()` calls

**fpga_controller.py** (hs_bridge):
- L6d base (1e3a114) + full parameter encoding:
  - `command[-34:-21]` = `(stopAddr+15)//16` (13-bit boundary)
  - `command[-21:-17]` = delay_value (4 bits)
  - `command[-17:-14]` = refractory_max (3 bits)
  - `command[-14:-13]` = dual_synapse_en (1 bit)
  - `command[-13:-9]` = shadow_uram_offset (4 bits)
  - `command[-8:-7]` = legacy_noise_en (1 bit)

**network.py** (hs_bridge):
- L6d base (1e3a114) + full parameter passing on BOTH write_neuron_type calls

**test_bitstream_hardware_fast.py**: From fb811b4 (NOT e526b6f testing-suite version)

**DVS test runtime patches**: shift=0 to -17, legacy_noise_en=1 before CRI_network

**Dependency commits**:
- hs_api: testing-suite branch e526b6f infrastructure, fb811b4 test files
- hs_bridge: 1e3a114 with patches
- connectome_utils: dev branch, 181f8a8

---

## 16-Core Multicore — multicore_1

**File**: `sixteen_core_top_multicore_1.bit`
**HW Tests**: 42/42 core 0 only; cores 1-15 fail (tdest routing bug)

### RTL Changes from L6m (via `patch_16core.py`)

1. `sixteen_core_top.v`: Generate loop `j<1` to `j<16`
2. `sixteen_core_top.v`: Added `.CORE_ID(j[3:0])` parameter
3. `sixteen_core_top.v`: Removed switch port tie-offs
4. `sixteen_core_top.v`: HBM tie-offs `j=1..31` to `j=16..31`
5. `sixteen_core_top.v`: Removed `user_irq_per_core[15:1] = 15'b0`

### The tdest Routing Bug

`pcie_tdest_generator` reads `tdata[503:499]` but software has two coreID encodings:
- Format A (`clear`, `execute`): `coreBits = np.binary_repr(coreID,5)+'000'` — coreID at [503:499]
- Format B (`write_neuron_type`): `np.binary_repr(coreID,8)` — coreID at [499:496]

For core 0: both produce `00000000`, works fine. For core 1+: programming goes to wrong core.

---

## 16-Core Multicore — multicore_2 (Building)

**File**: `sixteen_core_top_multicore_2.bit`
**Status**: Building on crisdsc2

### RTL Fix — tdest generator (`switch_1_32.sv`)

```verilog
// BEFORE (multicore_1): assign m.tdest = s.tdata[503:499];
// AFTER  (multicore_2): assign m.tdest = {1'b0, s.tdata[499:496]};
```

### Software Fix — fpga_controller.py

```python
# BEFORE: coreBits = np.binary_repr(coreID, 5) + '000'     # [503:499]
# AFTER:  coreBits = '000' + np.binary_repr(coreID, 5)      # [499:496]
```

After build completes:
```bash
scp multi_core.runs/impl_1/sixteen_core_top.bit crisdsc0:/bitstreams/sixteen_core_top_multicore_2.bit
# Then on crisdsc0:
bash run_all_tests_multicore.sh
```

**Parallel limitation**: Only 1 H2C + 1 C2H DMA channel. `switch_16_1` merges responses — multiple processes deadlock. Sequential per-core testing works; true parallel requires single-process orchestrator.
