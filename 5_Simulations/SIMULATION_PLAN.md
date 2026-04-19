# AERIS-10 Multi-Level Simulation Environment — Development Plan

**System:** AERIS-10 10.5 GHz X-band PLFM phased-array radar  
**Toolchain:** 100% open-source (Verilator, cocotb, Python/numpy/scipy, ngspice)  
**Goal:** Understand AERIS-10 architecture through layered simulation; contribute improvements to upstream OSS repo

---

## 1. Simulation Hierarchy

```
L4  System-level          5_Simulations/system/
    Radar equation · PLFM waveform · range-Doppler map · CA/SO/GO-CFAR · array patterns

L3  Subsystem-level       5_Simulations/subsystem/
    TX/RX chain integration · end-to-end noise figure · phase noise → resolution impact

L2+L1  RF/Analog + Digital Co-simulation   9_Firmware/9_2_FPGA/cocotb/
    RF/analog behavioral models (floating-point) used as cocotb stimulus drivers
    that inject realistic impairments into RTL simulated under Verilator
```

### Signal flow in co-simulation

```
Radar Scene (Python)
      │  target returns + clutter + noise floor
      ▼
RF/Analog Drivers  (cocotb/models/rf_analog/)
  ├─ pll.py          ADF4382A: Leeson phase noise, fractional spurs
  ├─ mixer.py        LT5552:   conversion gain, NF, IIP3, image rejection
  ├─ pa.py           QPA2962:  AM/AM, AM/PM, P1dB compression
  ├─ lna.py          ADTR1107: NF, gain, OIP3
  ├─ phase_shifter.py ADAR1000: 5.625° quantization step + RMS phase error
  └─ adc.py          AD9484:   ENOB, thermal noise, clock jitter
      │  impaired 14-bit ADC sample stream (integer, matches hardware format)
      ▼
RTL Under Test  (Verilator via cocotb)
  ddc_400m.v → matched_filter_*.v → doppler_processor.v → cfar_ca.v
      │
      ▼
Verification
  RTL output compared against fpga_model.py golden reference
  Detection performance asserted to degrade predictably with each impairment
```

---

## 2. Directory Layout

```
9_Firmware/9_2_FPGA/
├── cocotb/                              ← NEW (all phases)
│   ├── Makefile                         Phase 0: top-level dispatcher
│   ├── common.mk                        Phase 0: shared Verilator/cocotb flags
│   ├── requirements.txt                 Phase 0: Python dependencies
│   ├── duts/                            Phase 0: per-DUT variable files
│   │   ├── cfar_ca.mk                   Phase 0 (smoke test)
│   │   ├── ddc.mk                       Phase 2
│   │   ├── matched_filter.mk            Phase 2
│   │   ├── fft.mk                       Phase 2
│   │   └── integration.mk              Phase 4
│   ├── stubs/                           Phase 0: Xilinx primitive stubs
│   │   ├── xilinx_prims.v               BUFG, IBUFDS, ODDR etc. (for top-level sims)
│   │   └── README.md
│   ├── models/
│   │   ├── digital/                     Phase 2: port/extend fpga_model.py
│   │   │   ├── __init__.py
│   │   │   ├── ddc_model.py
│   │   │   ├── matched_filter_model.py
│   │   │   ├── fft_model.py
│   │   │   └── cfar_model.py
│   │   └── rf_analog/                   Phase 3: RF behavioral drivers
│   │       ├── __init__.py
│   │       ├── pll.py                   ADF4382A
│   │       ├── mixer.py                 LT5552
│   │       ├── pa.py                    QPA2962
│   │       ├── lna.py                   ADTR1107
│   │       ├── phase_shifter.py         ADAR1000
│   │       ├── adc.py                   AD9484
│   │       └── dac.py                   AD9708
│   └── tests/
│       ├── common.py                    Phase 0: shared cocotb coroutines
│       ├── test_cfar_smoke.py           Phase 0: infrastructure smoke test
│       ├── test_ddc.py                  Phase 2
│       ├── test_matched_filter.py       Phase 2
│       ├── test_fft.py                  Phase 2
│       ├── test_cfar.py                 Phase 2
│       ├── test_mti.py                  Phase 2
│       ├── test_chirp_controller.py     Phase 2
│       ├── test_rf_impairments.py       Phase 4: parametrized impairment sweeps
│       └── test_e2e.py                  Phase 4: scene→RF→RTL→detection
└── tb/cosim/                            ← UNCHANGED — read-only reference

5_Simulations/
├── system/                              ← NEW Phase 1 (L4)
│   ├── __init__.py
│   ├── radar_params.py                  AERIS-10 system constants
│   ├── radar_equation.py               SNR, range, sensitivity
│   ├── waveform.py                      PLFM chirp: BW, PRI, duty cycle, resolution
│   ├── range_doppler.py                2D range-Doppler map
│   ├── detection.py                    Pd, Pfa, CFAR threshold vs. alpha
│   ├── beamforming.py                  16×8 / 32×16 array, ADAR1000 quantization
│   └── tests/
│       ├── __init__.py
│       └── test_*.py
├── subsystem/                           ← NEW Phase 5 (L3)
│   ├── __init__.py
│   ├── tx_chain.py
│   ├── rx_chain.py
│   ├── noise_figure.py
│   └── tests/
│       ├── __init__.py
│       └── test_*.py
├── Antenna/                             ← existing — unchanged
├── DAC_ReconstructionFilter/            ← existing — unchanged
├── IF_BPF/                              ← existing — unchanged
├── Matlab/                              ← existing — unchanged
└── RF switch/                           ← existing — unchanged
```

---

## 3. Phased Delivery

### Phase 0 — Verilator + cocotb Infrastructure
**Status:** Not started  
**Goal:** Prove the toolchain end-to-end with a single smoke test on `cfar_ca.v`

Deliverables:
- `cocotb/requirements.txt` — pinned deps (cocotb, numpy, scipy, pytest)
- `cocotb/Makefile` + `cocotb/common.mk` — `make DUT=<name>` dispatches per-DUT builds
- `cocotb/duts/cfar_ca.mk` — sources, toplevel, module for CFAR smoke test
- `cocotb/stubs/` skeleton + `README.md` explaining when stubs are needed
- `cocotb/tests/common.py` — `start_clock()`, `reset_dut()`, `await_idle()` coroutines
- `cocotb/tests/test_cfar_smoke.py` — single-target detection smoke test on `cfar_ca.v`
- `__init__.py` stubs in all `models/` subdirectories
- `.gitignore` entries for `cocotb/build/`

Invoke: `cd 9_Firmware/9_2_FPGA/cocotb && make DUT=cfar_ca`

Verilator flags: `--trace-fst -DSIMULATION -Wall -Wno-fatal`

### Phase 1 — L4 System-Level Python Simulation
**Status:** Not started  
**Goal:** Parameterized radar equation and waveform analysis for AERIS-10

Deliverables:
- `system/radar_params.py` — all AERIS-10 constants (f0, BW, PRI, Pt, Gt, Gr, NF, losses)
- `system/radar_equation.py` — SNR(R), max range, sensitivity floor
- `system/waveform.py` — PLFM chirp synthesis, range resolution, ambiguity function
- `system/range_doppler.py` — 2D range-Doppler map with configurable scene
- `system/detection.py` — Pd/Pfa curves, CA/SO/GO-CFAR threshold vs. alpha
- `system/beamforming.py` — phased array patterns, ADAR1000 phase quantization effect
- pytest unit tests for each module

Upstream PR candidate: adds `5_Simulations/system/` as educational simulation resource.

### Phase 2 — L1 Digital Co-simulation (Core DSP)
**Status:** Not started  
**Goal:** cocotb/Verilator testbenches for all FPGA DSP modules

Key design decision: `tb/cosim/fpga_model.py` is ported and modularised into
`cocotb/models/digital/` rather than rewritten. Existing `tb/cosim/*.hex` fixtures
are loaded via `numpy.fromfile()` — no data regeneration needed.

Deliverables:
- `models/digital/ddc_model.py` — NCO + CIC + FIR Python reference
- `models/digital/matched_filter_model.py`
- `models/digital/fft_model.py`
- `models/digital/cfar_model.py`
- `cocotb/duts/ddc.mk`, `matched_filter.mk`, `fft.mk`
- `tests/test_ddc.py` — DDC RTL vs. model, parametrized over FTW values
- `tests/test_matched_filter.py` — multi-segment, short/long chirp, reuse cosim hex
- `tests/test_fft.py` — stationary/moving/two-target Doppler scenarios
- `tests/test_cfar.py` — CA/SO/GO-CFAR modes, edge handling, detect_count validation
- `tests/test_mti.py` — clutter cancellation verification
- `tests/test_chirp_controller.py` — PLFM LUT output integrity

Upstream PR candidate: replaces fragile DPI-C dependency with portable cocotb.

### Phase 3 — L2 RF/Analog Behavioral Models
**Status:** Not started  
**Goal:** Floating-point Python models of each RF component, parametrized from datasheets

Each model exposes:
- A `generate(signal, params)` function for standalone use (L3 subsystem)
- A cocotb `Driver` class for injection into RTL simulations (L4 co-sim)

Components:

| File | Component | Key parameters |
|------|-----------|----------------|
| `pll.py` | ADF4382A | Phase noise floor, Leeson freq, fractional spur level |
| `mixer.py` | LT5552 | Conversion gain, NF, IIP3, IIP2, image rejection |
| `pa.py` | QPA2962 | Psat, P1dB, AM/AM polynomial, AM/PM, NF |
| `lna.py` | ADTR1107 | NF, gain, OIP3, operating frequency |
| `phase_shifter.py` | ADAR1000 | Step size (5.625°), RMS phase error, gain flatness |
| `adc.py` | AD9484 | ENOB vs. frequency, thermal noise, clock jitter sensitivity |
| `dac.py` | AD9708 | SFDR, reconstruction filter response, update rate |

Each file includes pytest unit tests verifying parameter-vs-output relationships
(e.g., increasing IIP3 reduces IM3 products, higher ENOB reduces quantization noise).

Upstream PR candidate: new `cocotb/models/rf_analog/` modeling layer.

### Phase 4 — L2+L1 Impairment Co-simulation
**Status:** Not started  
**Goal:** Quantify how RF impairments degrade end-to-end detection performance

Deliverables:
- `tests/test_rf_impairments.py` — parametrized sweeps:
  - Phase noise level → Doppler velocity resolution degradation
  - ADC ENOB → SNR floor, spur-free dynamic range
  - PA P1dB compression → CFAR false alarm rate increase
  - ADAR1000 phase quantization → beamforming sidelobe level
- `tests/test_e2e.py` — full pipeline: scene → RF drivers → RTL → detection assertion
- Each test produces a CSV result file (local only, not committed)

Engineering questions answered by this phase:
- What ADC ENOB is the actual system bottleneck?
- How much PLL phase noise can be tolerated before Doppler bins merge?
- Does PA compression at full power cause false alarms in CFAR?

Upstream PR candidate: unique contribution — no other OSS radar project has this level of RF+RTL co-simulation.

### Phase 5 — L3 Subsystem Chain
**Status:** Not started  
**Goal:** Integrate L2 RF models into full TX/RX chain Python simulation

Deliverables:
- `subsystem/tx_chain.py` — chirp gen → DAC → mixer → PA → antenna
- `subsystem/rx_chain.py` — antenna → LNA → mixer → IF filter → ADC
- `subsystem/noise_figure.py` — Friis formula end-to-end NF budget, cross-check vs. L4 radar equation
- Sensitivity analysis: which component dominates NF?

Upstream PR candidate: adds `5_Simulations/subsystem/` with quantitative NF budget.

### Phase 6 — Control-Plane RTL (Advanced)
**Status:** Not started  
**Goal:** Testbenches for SPI peripherals and USB interface

Deliverables:
- `tests/test_adar1000_spi.py` — ADAR1000 SPI register write/readback
- `tests/test_adf4382a_spi.py` — ADF4382A frequency programming sequence
- `tests/test_usb_ft2232h.py` — USB 2.0 packet framing (11-byte radar packet)
- `tests/test_power_seq.py` — power-on sequencing state machine validation

---

## 4. Upstream PR Strategy

| Phase | PR scope | Value to upstream |
|-------|----------|-------------------|
| 1 | `5_Simulations/system/` | Educational: parameterized radar equation and waveform analysis |
| 2 | `9_Firmware/9_2_FPGA/cocotb/` (core DSP tests) | Quality: portable cocotb replaces DPI-C fragility |
| 3 | `cocotb/models/rf_analog/` | Novel: datasheet-based RF behavioral models |
| 4 | `test_rf_impairments.py` + `test_e2e.py` | Unique: RF+RTL impairment co-simulation |
| 5 | `5_Simulations/subsystem/` | Educational: quantitative NF budget and sensitivity analysis |

PRs should be submitted independently per phase. Each is self-contained.

---

## 5. Key Implementation Notes

### Verilator-specific
- Always compile with `-DSIMULATION` — required for `adc_clk_mmcm.v` passthrough and BRAM init blocks
- `--trace-fst` for waveform dumps (FST is faster/smaller than VCD); disable in CI
- `--coverage` to track which RTL lines are exercised
- `--Wno-fatal` to treat lint warnings as warnings not errors during early development
- Xilinx primitive stubs in `stubs/xilinx_prims.v` are only needed for top-level integration tests (Phase 6); individual DSP modules elaborate cleanly without them

### Reusing existing assets
- `tb/cosim/fpga_model.py` → modularize into `models/digital/` (do not copy verbatim; refactor into class-per-module)
- `tb/cosim/*.hex` → load with `numpy.frombuffer(open(f,'rb').read(), dtype=numpy.int16)` in Phase 2 tests
- `tb/cosim/radar_scene.py` → import directly from Phase 4 `test_e2e.py`
- Existing `5_Simulations/IF_BPF/*.sch` → drive with ngspice in Phase 3 `mixer.py` for IF filter response

### RF model implementation notes
- All RF/analog models are purely floating-point (numpy complex128)
- Models are stateless functions where possible; stateful (e.g., PLL with memory) use a class
- Phase noise model: generate colored noise via power spectral density shaping in frequency domain, then IFFT to time domain
- ADC jitter model: apply aperture jitter as a phase error on the sampling clock, compute SNR degradation analytically per `SNR_jitter = -20*log10(2*pi*f_in*t_jitter)`
- PA compression: use a memoryless polynomial model AM/AM: `Vout = a1*Vin + a3*Vin^3` with coefficients fitted to QPA2962 P1dB spec

---

## 6. Verification Philosophy

This is an **OSS learning and research project**, not a safety-critical or high-volume production target.
Verification effort is sized accordingly:

### Primary approach: simulation-based verification
- **Comprehensive cocotb testbenches** with realistic stimuli (RF impairments, radar scenes)
- **Golden reference cross-checks** — RTL output vs. `fpga_model.py` bit-accurate Python model
- **Parametrized sweeps** — Phase 4 impairment analysis (Pd/Pfa vs. ADC ENOB, phase noise, PA compression)
- **End-to-end scene validation** — Phase 4 `test_e2e.py`: scene → RF drivers → RTL → detection assertion
- **Coverage tracking** — Verilator `--coverage` to identify untested RTL regions

### Light-touch assertions (OK)
Simple SVA assertions to catch obvious bugs — add sparingly where high-value:
- Frame-level invariants (e.g., `detect_count ≤ MAX_TARGETS`)
- AXI handshake sanity (no `tvalid && !tready` deadlock for long stretches)
- Configuration guards (e.g., `guard_cells + train_cells ≤ window_size`)
- Pipeline stage balance (e.g., output valid only after N cycles of input valid)

Keep assertions behavioral and readable. Do not invest in exhaustive SVA coverage.

### Out of scope for this repo
The following are **deliberately not pursued upstream**:
- **Exhaustive formal verification / model checking** — appropriate for safety-critical (DO-254, medical, aerospace) and high-volume silicon; not cost-effective here
- **Equivalence checking** (RTL vs. netlist) — no tape-out, no netlist
- **UVM/SystemVerilog verification environments** — cocotb is the chosen framework; keep single-stack simplicity
- **Machine-learning-augmented modeling** (NN behavioral PA, EM surrogate, RD-map classifier, anomaly detector) — reserved for a separate private repository for potential commercial/patent exploration. Do not add a "Phase 7" to this plan.

Rationale: time invested in comprehensive E2E simulation and golden-reference cross-checks produces faster learning, catches 95%+ of real bugs, and delivers clear upstream PR value. Formal proofs and ML models are interesting but orthogonal to the goals of this repo.

---

*Last updated: 2026-04-19*  
*See `CLAUDE.md` at repo root for Claude Code session instructions.*
