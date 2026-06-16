# HiAER-Spike Hardware Platform

**Neuromorphic Computing on Xilinx VU37P FPGA (ADM-PCIE-9H7)**

## Overview

HiAER-Spike implements spiking neural networks with leaky integrate-and-fire (LIF) neurons on FPGA. The system supports configurable synaptic weights, refractory periods, synaptic delays, noise injection, and multi-core scaling.

This repository documents every hardware bitstream version, every RTL edit, the corresponding software configuration for each version, and the multicore scaling path from single-core through 16-core to the full 40-FPGA cluster.

## Architecture

Each neuromorphic core contains: an Internal Events Processor (IEP) for neuron evaluation, a Command Interpreter (CI) for DMA packet parsing, an External Events Processor (EEP) for spike I/O, an HBM Processor for synapse weight storage, 16 URAMs for membrane potential storage, and Parameter Memory (BRAM) for neuron type configurations.

The host communicates via PCIe Gen3 x16 through XDMA, with an AXI-Stream switch fabric routing commands to individual cores by extracting the coreID from DMA packet byte 1.

## Current Status

| Bitstream       | Cores | HW Tests    | DVS Full  | Status                           |
|-----------------|-------|-------------|-----------|----------------------------------|
| L6m             | 1     | 42/42       | 56.60%    | Verified single-core baseline    |
| multicore_1     | 16    | 42/42 core0 | 56.60%    | tdest bug for cores 1-15         |
| multicore_2     | 16    | pending     | pending   | tdest fix applied, building      |
| 2024 reference  | 1     | N/A         | 64.24%    | Accuracy target (XDMA v4.1.4)   |

## Documentation

See `docs/` for complete details:
- `01_bitstream_history.md` — Every bitstream version with RTL edits and software config
- `02_iep_modifications.md` — All IEP RTL changes with Verilog code
- `03_neuron_parameter_encoding.md` — write_neuron_type bit field layout
- `04_software_configuration.md` — Working software state and patches
- `05_multicore_design.md` — 16-core architecture, routing, and validation
- `06_dvs_accuracy.md` — DVS accuracy gap root cause analysis
- `07_roadmap.md` — NoC, Firefly, 40-FPGA cluster plan
- `08_lessons_learned.md` — Critical debugging lessons

## Machines

| Machine   | Role                                | Key Software            |
|-----------|-------------------------------------|-------------------------|
| crisdsc2  | Vivado builds, RTL development      | Vivado 2024.1           |
| crisdsc0  | FPGA testing, software development  | Python 3.10, ADXDMA     |

## Quick Start

### Flash and test single-core (L6m):
```bash
# On crisdsc0:
cd /home/omowuyi
bash run_all_tests.sh L6m
```

### Flash and test 16-core (multicore_2):
```bash
# On crisdsc0:
cd /home/omowuyi
bash run_all_tests_multicore.sh
```

## Contacts

- **Omowuyi Olajide** (omowuyi@gmail.com) — PhD student, hardware lead
- **Leif Gibb** — Faculty advisor
- **Gwen Fawrseren** — Software/testing
- **Christopher Deng** — Testing suite
- **Kenneth Yoshimoto** — SDSC/NSG infrastructure
- **Paresh Kurdekar Vasanth** — Multicore NoC
