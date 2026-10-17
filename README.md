# Adjustable-Low-Dropout-Regulator-with-Miller-Compensation

A 1.0 V-input, **digitally adjustable low-dropout regulator** that delivers **0.6 V or 0.8 V at up to 20 mA**, selected by one control bit. The repo contains the schematic, a reusable error-amplifier sub-circuit, and a set of testbenches for loop stability, regulation, dropout, startup and transients.

| | |
|---|---|
| Input | 1.0 V |
| Output | **0.6 V** (`CTRL`=0) / **0.8 V** (`CTRL`=1) |
| Max load | 20 mA |
| Output capacitor | 100 pF |
| Reference | 0.6 V |
| Compensation | 30 pF Miller capacitor (gate → output) |
| Devices | Generic Level-1 MOSFET models (not a foundry PDK) |

---

## 1. Architecture

```

```

| Block | Role |
|---|---|
| **EA** (error amp) | NMOS differential pair with PMOS mirror load; 1 µA reference mirrored ×10 → 10 µA tail |
| **M1** | PMOS pass device, W/L = 800 µm / 0.1 µm, carries all load current |
| **C2** | 30 pF Miller capacitor from M1 gate to OUT (pole splitting) |
| **R3 / R4** | Feedback divider, 20 kΩ (top) / 60 kΩ (bottom) |
| **M2S** | NMOS switch in series with R4; `CTRL` opens or closes the bottom leg |

**Loop action:** if OUT drops, `fb` drops, the EA output (M1 gate) falls, M1 conducts more, and OUT recovers.

---

## 2. Design equations and worked examples

### 2.1 Output selection

`Vout = Vref · (1 + R3/R4)`

- `CTRL=1` (M2S on): feedback factor B = R4/(R3+R4) = 60/80 = 0.75  
  → Vout = 0.6 × (1 + 20/60) = **0.8 V**
- `CTRL=0` (M2S off): R4 leg is open, no divider current, B = 1 (unity-gain buffer)  
  → Vout = **0.6 V**
- Divider current in 0.8 V mode: 0.8 V / 80 kΩ = **10 µA**

### 2.2 Power and efficiency

An LDO dissipates `(VDD − Vout) · Iload`, so efficiency ≈ Vout/VDD.

| Mode | Loss at 20 mA | Efficiency |
|---|---|---|
| 0.6 V | 0.4 V × 20 mA = 8 mW | ≈ 60 % |
| 0.8 V | 0.2 V × 20 mA = 4 mW | ≈ 80 % |

### 2.3 Gain needed for accuracy

Closed loop: `Vout/Vref = A / (1 + A·B)`. Allowing about 11 mV of error:

- B = 1, Vout ≥ 0.589 V → ratio 0.9817 → A = 0.9817/(1 − 0.9817) ≈ **54 (34.7 dB)**
- B = 0.75, Vout ≥ 0.789 V → ratio 1.315 → A = 1.315/(1 − 0.75 × 1.315) ≈ **96 (39.6 dB)**

The 0.8 V mode is the tighter requirement, so the loop needs ≳ 40 dB at all load and corner conditions.

### 2.4 Stability (Miller compensation)

With C2 across the pass device:

- Loop unity-gain frequency: `UGF ≈ gm1 · B / (2π · Cc)`
- Non-dominant pole: `p2 ≈ gm_pass / (2π · CL)`

**Example (0.6 V mode):** gm1 ≈ 346 µS, Cc = 30 pF, B = 1  
→ UGF ≈ 346 µ / (2π × 30 p) ≈ **1.8 MHz**, which matches the simulated 1.8 MHz.

The load pole `1/(2π · Rout · CL)` moves about 1000× between 20 µA and 20 mA, which is why stability is checked at both load extremes in every corner.

---

## 3. Repository layout

```
LDO_TOP.asc             Top-level schematic (open this first)
EA.asy                  Error-amp block symbol
pmos_g.asy, nmos_g.asy  MOSFET symbols used by the schematic
ldo_ea.lib              Generic MOSFET models + EA sub-circuit (used by LDO_TOP.asc)
ldo_core.lib            Flat LDO sub-circuit LDO_ADJ + models (used by the testbenches)
tb_loop_gain.cir        AC loop gain / phase margin (L-C loop-break)
tb_startup_adjust.cir   Power-up, then CTRL 0→1 (0.6 V → 0.8 V)
tb_dc_sweeps.cir        Vout vs VDD: line regulation and dropout, both modes
tb_load_reg.cir         DC load regulation, 1 → 20 mA
tb_line_transient.cir   VDD step response
tb_load_transient.cir   10 mA ↔ 20 mA load step
README.md
```

Keep all files in one folder: the schematic and testbenches find the symbols and `.lib` files by relative path.

---

## 4. Running it

1. **Schematic:** open `LDO_TOP.asc` in LTspice and run. It simulates power-up, then switches `CTRL` from 0 to 1 at 200 µs. Probe `OUT`.
2. **Testbenches:** open any `tb_*.cir` and run.
3. **Corners:** set `.param corner` (−1 = FF, 0 = TT, +1 = SS). It shifts Vth by ±50 mV and scales Kp by ∓20 %. Several testbenches already `.step` over it.
4. **Output mode:** change `ctl` (0 or 1) in the testbench, or the `Vctrl` source in the schematic.
5. **Loop gain:** plot `-V(fbtap)/V(vfb)` in dB and degrees. **Phase margin = 180° + phase at the 0 dB crossing.**

Sub-circuit pins for `LDO_ADJ`: `vdd vref ctrl vout fbtap vfb`. Short `fbtap` to `vfb` for normal operation; break them apart for loop gain.

---

## 5. Results (typical and corner sweep)

Generic models, VDD = 1.0 V, CL = 100 pF.

| Metric | 0.6 V mode | 0.8 V mode |
|---|---|---|
| Output at 20 mA (TT) | 0.5997 V | 0.7995 V |
| Load regulation, 1 → 20 mA | −0.46 mV | −0.63 mV |
| Line regulation, 0.9 → 1.15 V at 20 mA | ≈ 0.06 mV/V | ≈ 0.9 mV/V |
| Dropout at 20 mA (Vout = 98 % of target) | ≈ 195 mV | ≈ 80 mV |
| Loop-gain UGF (TT / SS / FF, both loads) | 1.6 – 2.0 MHz | 0.9 – 1.5 MHz |
| **Phase margin, all corners, 20 µA and 20 mA** | **74° – 89°** | **80° – 89°** |
| Load step 10 → 20 mA in 100 ns | ≈ −43 mV undershoot, ≈ +44 mV on release | — |
| VDD step 0.9 ↔ 1.0 V | ≈ ±9 mV | — |
| Startup | no overshoot | no overshoot |

---

## 6. Verification status and limitations

- **Verified in ngspice 42**, not in LTspice itself. The netlists were simulated there with LTspice-only directives (`.step`, `.meas`, the `.tran 0 …` form) stripped or converted. The schematic's connectivity was checked by rebuilding its netlist from `LDO_TOP.asc` and simulating that. If LTspice reports a `.meas` or `.step` error, the simulation still runs; use the waveform viewer instead.
- **Generic Level-1 models** (Vth ≈ 0.35 V, Kp 300 µA/V² N and 100 µA/V² P, λ = 0.1). Absolute gain, leakage and dropout will differ on a real process node.
- **Corners are emulated** by shifting Vth and Kp. **Temperature is not modeled.**
- Not yet covered: PSRR, output noise, quiescent current versus load, Monte Carlo, and layout / post-layout.

## 7. Roadmap

- [ ] PSRR and output-noise testbenches
- [ ] Quiescent current versus load
- [ ] Temperature-aware models and Monte Carlo mismatch
- [ ] Port the design to a real PDK (e.g. 28 nm) and compare
- [ ] Cap-less / small-C_L variant

## Credits

Specifications and topology follow the adjustable-LDO reference design from the IIT Hyderabad × VSD × Synopsys *Cloud-Based Analog IC Design Hackathon* (Abhijeet Patnaik). LTspice implementation and documentation: `<your name>`.

## License

`<choose a license, e.g. MIT>`
