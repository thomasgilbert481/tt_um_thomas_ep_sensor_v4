# v4 pre-tapeout simulation checklist

Tracking the 13-item sign-off checklist for v4 Recipe B before submission.

## Tier 1 — must do before tape-out

### 1. Full RC-extracted PEX across the cv-code sweep
**Status:** ⏳ IN PROGRESS — Magic ext2spice on v4 GDS.
**Tool:** `magic -dnull -noconsole -T sky130A` with `extract all; ext2spice lvs; ext2spice -p`
**What to deliver:** netlist with parasitics (R + C), re-run 256-code cv sweep in ngspice transient + FFT, confirm Δf(ε=0) ≈ 0, β = √(C₁/(C₂+Cc)) preserved at EP.

### 2. Mutual inductance M₁₂ between L₁ and L₂ — flagged HARDEST risk
**Status:** PARTIAL — v2 Neumann integration gave K = −0.032 (3-turn geometry pending recompute).
**Why critical:** 130×130 µm spirals in a 334×450 µm tile at 2.7 GHz can hit |k| = 5-15% of L. Above 0.02 you break Zhao chiral-EP topology.
**v4-specific check needed:** the new 3-turn w=15 spirals have different L (0.6 nH vs 1.35 nH) so M₁₂ rescales. Re-run Neumann for v4 geometry.
**Tools:** FastHenry, ASITIC, or OpenEMS. Already have figure-8 geometry generator (`figure8_design.py`) that delivers |k| = 0.001 if K_M12 turns out >0.02 in v4.

### 3. Pad + bondwire + ESD-diode model in testbench
**Status:** ✓ PARTIAL — `ep_sensor_v4_aggressive.spice` already includes:
  - L_BOND = 1.5 nH (bondwire)
  - R_BOND = 0.05 Ω (bondwire skin loss)
  - C_PAD = 125 fF (pad oxide)
  - 50 Ω VNA source (R_VNA)
**Missing:** ESD diode model (~300 fF capacitance + ~5 mA breakdown). Add as `Cesd v_pad1n2 GND 300f` parallel.
**Also missing:** PCB launch S-parameters. Approximate as 5 mm 50-Ω microstrip = ~30 ps delay + 0.5 dB loss at 5 GHz.

### 4. 5-corner × 3-temp × 3-voltage sweep (45 corners)
**Status:** PARTIAL — v2 ran 5 corners × 3 temps × 3 voltages = 45 corners earlier (`silicon_v2_corners.py`).
**v4-specific check needed:** re-run for the 3-turn spiral geometry + f₀=4.6 GHz operating point.
**Critical metrics per corner:**
  - (a) buffer Z_out ≪ Z_Cc at 4.6 GHz?  (Z_Cc = 1/(2π·4.6G·1.36p) = 25 Ω; Z_out diff-pair ~ 200 Ω; FAIL at SS-cold expected)
  - (b) f₀ within VNA bandwidth (4-5.5 GHz tunable target)
  - (c) EP coalescence at ε=0 preserved

### 5. Process + device-mismatch Monte Carlo
**Status:** PARTIAL — v2 had 500-trial global MC but lacked local-mismatch variation between L₁/L₂ branches.
**v4 plan:** use sky130's `mc_g_mm_subckt.spice` mismatch library, vary:
  - NMOS Vt and β PER DEVICE (XM1 vs XM2, XMp1 vs XMp2)
  - cap_mim mismatch per unit cap in cv-array (b0..b7 each independent)
  - Spiral L1 vs L2 independent (process mismatch + lithography)
**Trial count:** 2000 (statistical convergence at the 1σ level).

## Tier 2 — strongly recommended

### 6. Spiral Q & SRF via full-wave EM
**Status:** NOT DONE for v4 geometry. v2 had FastHenry quasi-static (Q = ωL/Rs only, no eddy current or substrate-coupling Q derating).
**Tool:** OpenEMS (free FDTD), or EMX (Synopsys), or Sonnet.
**For v4:** verify L=0.6 nH at 4.6 GHz, Q ≥ 10, SRF > 8 GHz.

### 7. Phase noise / .noise analysis
**Status:** NOT DONE.
**Plan:** `.noise V(V1) Vin DEC 200 1MEG 10GIG`, integrate output noise over ±1 MHz around each split peak. Tells you minimum resolvable Δf in post-fab measurement.

### 8. cv-array DNL / INL
**Status:** PARTIAL — `cv_array_dnl_inl.py` from v2 work showed code 127→128 has -1260 fF jump (MSB transition glitch).
**v4 plan:** re-run with extracted v4 netlist; verify glitch is at same code; document the spec sheet.

### 9. Substrate coupling
**Status:** NOT DONE.
**Plan:** model p-substrate as 10 Ω·sq sheet with sparse taps; inject 100 mV at digital-supply node; verify <100 µV at V1/V2_in.

### 10. VDPWR ripple injection
**Status:** PARTIAL — v2 had basic PSRR check showing V1 follows VDPWR ~1:1 (high PSRR sensitivity).
**v4 plan:** add 10 mV ripple at 100 MHz, 1 MHz, 10 kHz; observe drift in Z_out and f₀ shift.

## Tier 3 — nice to have

### 11. Startup transient (VDPWR 0 → 1.8 V over 100 µs)
**Status:** NOT DONE for v4.
**Plan:** `.tran 0.1u 200u` with VDPWR ramp; verify no latch-up, source-follower biases cleanly.

### 12. cv-array switch Ron impact
**Status:** NOT DONE.
**Plan:** at LSB (20 fF), the NFET Ron should be ≪ 1/(2πf·C) = 1/(2π·4.6G·20f) = 1.7 kΩ. Verify Ron < 200 Ω, Coff < 4 fF.

### 13. Pre-tapeout falsifiable spec sheet
**Status:** PARTIAL — `V4_README.md` has predicted f₀, Q, κ_eff, R². Need to extend with corner spread.

---

## Recommended priority (1 week of sim time budget)

| Day | Task | Outcome |
|---|---|---|
| 1 | Magic PEX extract on v4 GDS | RC-extracted SPICE netlist |
| 2 | Re-run ε sweep on PEX netlist; document drift in κ, Δf₀ | item 1 ✓ |
| 3 | Neumann/FastHenry M₁₂ on v4 3-turn geometry | item 2 ✓ |
| 4 | 45-corner sweep on PEX netlist | item 4 ✓ |
| 5 | Mismatch MC with local variation | item 5 ✓ |
| 6 | items 7, 8, 10 noise + DNL/INL + PSRR | tier 2 ✓ |
| 7 | items 11, 12, 13 startup + Ron + spec sheet | tier 3 ✓ |

The PEX-and-corners pipeline (days 1-5) is the must-have. Days 6-7 catch
bring-up surprises but don't gate tapeout.
