# AERIS-10 PLFM Radar — Claude Code Instructions

## Repository identity
- **Project:** AERIS-10 (10.5 GHz X-band PLFM phased-array radar)
- **Fork of:** `https://github.com/NawfalMotii79/PLFM_RADAR` (upstream OSS)
- **Goal:** Learn the system through simulation, contribute improvements back upstream via PR

## Language rule
All implementation files and documentation **must be in English** — the repo targets upstream OSS contribution.

## Active development: multi-level simulation environment
See **`5_Simulations/SIMULATION_PLAN.md`** for the full architecture and phased plan.

Current phase: **Phase 0 — cocotb + Verilator infrastructure** (not yet started).

## Toolchain (100% OSS — no exceptions)
| Layer | Tools |
|-------|-------|
| RTL simulation | Verilator ≥ 5.x |
| Testbench framework | cocotb ≥ 1.9 |
| Test runner | pytest ≥ 7.4 |
| Signal processing | numpy, scipy |
| Plots (on-demand only, never committed) | matplotlib |
| SPICE (IF filters) | ngspice |

Do **not** suggest MATLAB, LTspice, Cadence, or any commercial tool.

## Directory layout for simulation work
```
9_Firmware/9_2_FPGA/
├── cocotb/          ← new cocotb+Verilator testbenches (all phases)
└── tb/cosim/        ← existing DPI-C golden refs — DO NOT modify

5_Simulations/
├── system/          ← L4 Python system-level (Phase 1)
├── subsystem/       ← L3 Python subsystem chain (Phase 5)
└── [existing dirs]  ← Antenna, IF_BPF, Matlab, RF switch — do not reorganise
```

## Hard constraints
- `5_Simulations/sys_sim/` **has been removed** — moved to private repo. Do not recreate it.
- `tb/cosim/` is read-only reference material. Extend in `cocotb/`, never overwrite.
- Simulation figures are generated on-demand locally; never commit matplotlib output to `docs/`.
- RF/analog behavioral models use **floating-point** throughout (not fixed-point).
- RF/analog models live in `cocotb/models/rf_analog/` and act as cocotb stimulus drivers, injecting impairments into the RTL under test.

## Verification philosophy
- **Simulation-first.** Comprehensive cocotb testbenches, golden-reference cross-checks vs. `fpga_model.py`, parametrized impairment sweeps, and E2E scene-to-detection validation. This is an OSS learning repo, not safety-critical silicon.
- **Light SVA assertions OK** for frame invariants, AXI handshake sanity, and config guards — do NOT invest in exhaustive formal verification, model checking, or UVM.
- **Machine-learning-augmented modeling (NN behavioral PA, EM surrogate, RD-map classifier, anomaly detector) is out of scope upstream** — reserved for a separate private repo for potential commercial/patent exploration. Do not add a Phase 7 or NN components to `SIMULATION_PLAN.md`.

## Key RTL facts (saves re-reading schematics)
- FPGA: Xilinx Artix-7 XC7A50T (production) / XC7A200T (dev)
- Signal chain: ADC AD9484 → DDC (`ddc_400m.v`) → Matched Filter → FFT (`doppler_processor.v`) → CFAR (`cfar_ca.v`) → USB out
- `xfft_16.v` is pure behavioral RTL (no Xilinx IP) — Verilator can elaborate it directly
- `adc_clk_mmcm.v` has a built-in `` `ifdef SIMULATION `` passthrough — compile with `-DSIMULATION`
- Compile flag `-DSIMULATION` must always be set for cocotb/Verilator builds
- Existing `tb/cosim/fpga_model.py` is a bit-accurate Python model of the full DSP chain — reuse as golden reference in `cocotb/models/digital/`
- Existing `tb/cosim/*.hex` fixture files can be loaded directly via `numpy.fromfile()` in cocotb tests
