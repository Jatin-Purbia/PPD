# AGENT TASK PROMPT
## Aspen Plus Steady-State Flowsheet — Integrated Ammonia Plant (100 TPD)
### Version 1.0 | Process Simulation Brief

---

## ROLE & OBJECTIVE

You are a **senior process simulation engineer** with deep expertise in Aspen Plus (v12 or later). Your task is to build a **complete, converged, steady-state Aspen Plus simulation** of an integrated ammonia production plant with a combined production capacity of **100 metric tonnes per day (TPD) of ammonia**, split into:

- **Gaseous ammonia (NH₃ gas)** product stream
- **Liquid ammonia (NH₃ liquid)** product stream

The flowsheet must be fully integrated, mass-balanced, energy-balanced, and ready for equipment sizing and utility load extraction. All four deliverables listed below must be produced in a single simulation file.

---

## DELIVERABLES

### D1 — Complete Integrated Flowsheet (Aspen Plus .apwz file)
A single Aspen Plus file containing all sections below, with all streams connected and all recycles converged. The flowsheet must be navigable via clearly labelled sections using the Aspen Plus flowsheet annotation feature.

### D2 — Equipment List with Technical Specifications
Export a complete equipment summary table (via Aspen Plus Results → Equipment Summary, supplemented by manual entries) containing: tag number, equipment type, key design parameters, operating conditions, material of construction notes, and sizing basis. Format as an Excel or in-simulation report.

### D3 — Heat Exchanger Utility Load Summary
For every heat exchanger in the flowsheet: hot-side inlet/outlet T and flowrate, cold-side inlet/outlet T and flowrate, duty (MW or Gcal/hr), utility type (HP steam, CW, refrigerant, flue gas, process-to-process), LMTD, and U×A product. Export from Aspen Plus HX Summary.

### D4 — Stream Table (Full Material Balance)
Complete stream table for all process streams: name, phase, temperature (°C), pressure (bar), molar flow (kmol/hr), mass flow (kg/hr), and component mole fractions. Export from Aspen Plus → Report → Stream Summary.

---

## PROCESS SCOPE & SECTIONS TO MODEL

### SECTION A — FEED PREPARATION

#### A1. Feed Gas Specification
- **Feed**: Natural gas (primary) with optional naphtha capability
- **NG composition (mole %)**: CH₄ = 88%, C₂H₆ = 6%, C₃H₈ = 3%, n-C₄H₁₀ = 1%, CO₂ = 1%, N₂ = 1%
- **Feed pressure**: 40 bar(g)
- **Feed temperature**: 40°C (ambient)
- **Total feed rate**: Size to achieve 100 TPD NH₃ (solver must back-calculate)

#### A2. Desulphurisation Section
Model as two sequential reactor blocks:

**Block R-3201 — Hydrodesulphuriser (HDS)**
- Type: `REquil` or `RGibbs`
- Catalyst: Ni-Mo (TK-251), 9.0 m³ per vessel
- Operating T: 380°C, P: 35 bar(g)
- Feed: NG + recycle H₂ (H₂/feed ratio = 0.03 mol/mol)
- Reactions to model:
  - RSH + H₂ → RH + H₂S
  - COS + H₂ → CO + H₂S
  - C₄H₄S + 4H₂ → C₄H₁₀ + H₂S
- Specify: S conversion = 99.95% (overall organic S → H₂S)

**Block R-3202A/B — ZnO Absorbers (model as Sep or RStoic)**
- Two vessels in series (model as sequential absorbers)
- ZnO loading: 30 m³ per vessel
- Reaction: ZnO + H₂S → ZnS + H₂O (irreversible, fractional conversion = 1.0 for H₂S)
- Outlet specification: S content < 0.05 ppm (use Design Spec to confirm)
- **IMPORTANT**: Treat as a simple separator for H₂S in simulation (ZnO is a solid sorbent, not a reactive liquid)

**Block E-3204 — NG Preheater**
- Type: Heater block
- Duty: Heat NG from 40°C to 380°C
- Utility: Flue gas waste heat (model as a fired heater utility or as a process stream if flue gas is modelled)

#### A3. Adiabatic Prereformer
**Block R-3206 — Prereformer**
- Type: `RGibbs` (adiabatic: Q = 0)
- Catalyst: RKNGR-7H, 23.4 m³ total (2 beds: 2350 mm + 1450 mm depth)
- Inlet T: 490°C, P: 34 bar(g)
- Steam-to-carbon ratio (W/C): 3.3 mol/mol (add process steam before E-3201)
- Feed: Desulphurised NG + steam
- Expected result: All C₂₊ → CH₄, temperature change ≈ adiabatic (near-zero net heat)
- **NG-only bypass**: Model a Selector/Splitter block (B-3206BYP) that allows 100% bypass of R-3206 for pure methane feeds; set active stream to R-3206 path for this simulation.

**Block E-3201 — Feed/Steam Preheater Coil**
- Type: Heater block
- Duty: Heat NG + steam mixture from ~150°C to 490°C
- Source: H-3201 convection section flue gas (waste heat recovery)

---

### SECTION B — REACTION SECTION

#### B1. Primary Reformer
**Block H-3201 — Primary Reformer (Fired)**
- Type: `RGibbs` (isothermal approximation at outlet T) OR use a `REquil` block with built-in equilibrium reactions
- Catalyst: R-67-7H (pre-reduced Ni), 42.8 m³
- Tubes: 288 tubes (informational; do not need to model individually)
- **Inlet**: 490°C, 34 bar(g), W/C = 3.3
- **Outlet T**: 800°C (specify as isothermal outlet or use design spec to reach target CH₄)
- **Outlet P**: 32.5 bar(g) (allow ~1.5 bar pressure drop)
- **Target outlet composition (dry, mole %)**: H₂ ≈ 56%, CO ≈ 13%, CO₂ ≈ 8%, CH₄ ≈ 10–11%, N₂ ≈ 0.5–1%
- Reactions:
  - CH₄ + H₂O ⇌ CO + 3H₂ (primary, endothermic)
  - CH₄ + 2H₂O ⇌ CO₂ + 4H₂
  - CO + H₂O ⇌ CO₂ + H₂ (WGS, simultaneous)
- **Fired duty**: Calculate from energy balance; report as "Primary Reformer Fuel Load (Gcal/hr)"
- Model convection section WHR as separate Heater blocks (E-3204, E-3201) that receive heat from a pseudo-flue gas stream

#### B2. Secondary Reformer
**Block R-3203 — Secondary Reformer**
- Type: Two-zone model:
  - Zone 1 (combustion): `RStoic` block — combust injected air (O₂ fraction reacts with H₂ and CH₄)
  - Zone 2 (catalyst bed): `RGibbs` block — equilibrium at outlet T
- Catalyst: RKS-2-7H, 39 m³
- **Inlet (Zone 1)**: Reformed gas from H-3201 at 800°C + Process air
- **Process air**: Compressed air stream; set N₂ flow so that final syngas H₂/N₂ ratio = 3.0 (use Design Spec)
- **Air composition (mole %)**: N₂ = 78.09%, O₂ = 20.95%, Ar = 0.93%, CO₂ = 0.03%
- **Combustion zone T**: 1100–1200°C (verify; constraint: do not exceed 1400°C)
- **Outlet T**: 990°C (set Zone 2 outlet T)
- **Outlet P**: 31.5 bar(g)
- **Outlet target (dry)**: CH₄ ≈ 0.30%, CO ≈ 13%, CO₂ ≈ 7.3%, N₂ ≈ 22%
- **Design Spec required**: Manipulate air flow rate to achieve H₂/N₂ = 3.00 ± 0.01 in final syngas

**Block E-3206 — Secondary Reformer Waste Heat Boiler**
- Type: HeatX or MHeatX
- Shell side (hot): Process gas from R-3203, 990°C → 350°C
- Tube side (cold): BFW → HP steam at 125 bar(g), 510°C
- Duty: Calculate; this is a primary steam generator
- LMTD correction factor: 0.9 (assume counter-current)

#### B3. CO Shift Conversion — High Temperature

**Block E-3207 — Shift Feed Cooler/Preheater**
- Type: Heater or HeatX
- Cool process gas from E-3206 outlet to HT shift inlet T (350°C)

**Block R-3204 — HT Shift Converter**
- Type: `RGibbs` or `REquil`
- Catalyst: SK201-2 (Fe/Cr), 92.28 m³ total
- **Inlet T**: 350°C, P: 31.0 bar(g)
- **Outlet T**: ~440°C (adiabatic temperature rise from exothermic WGS)
- Set adiabatic (Q = 0) — outlet T is a result, not a specification
- Reaction: CO + H₂O ⇌ CO₂ + H₂ (WGS equilibrium at outlet T)
- **Target outlet CO (dry)**: ~4.0 mol%

#### B4. CO Shift Conversion — Low Temperature

**Block E-3208 — Interstage Cooler (HT → LT Shift)**
- Type: HeatX
- Hot side: HT Shift outlet (~440°C) → LT Shift inlet (200°C)
- Cold side: BFW preheating or process steam generation
- Duty: Calculate

**Block R-3205 — LT Shift Converter**
- Type: `RGibbs` or `REquil`
- Catalyst: LK-801-S (Cu/Zn), 122 m³ main bed + LSK Cl-guard, 6.1 m³ (top layer — model as a single combined bed)
- **Inlet T**: 200°C, P: 30.5 bar(g)
- Set adiabatic (Q = 0)
- Reaction: CO + H₂O ⇌ CO₂ + H₂ (WGS equilibrium at outlet T ~230°C)
- **Target outlet CO (dry)**: ~0.30 mol%

**Block E-3209 — LT Shift Outlet Cooler**
- Type: Heater or HeatX
- Cool process gas from ~230°C to 107°C (GV absorber inlet)

---

### SECTION C — CO₂ REMOVAL (GV / BENFIELD SYSTEM)

Model as a simplified absorber/stripper using the **Electrolyte NRTL** or **ENRTL-RK** property method for this section only. Alternatively, use a `Sep` block with specified CO₂ removal efficiency if electrolyte convergence is not achievable.

**Preferred approach**: Use `RadFrac` for absorber (F-3303) and stripper (F-3301/F-3302).

**Block F-3303 — CO₂ Absorber**
- Type: `RadFrac` (packed column) or `Sep` block
- Solvent: 30 wt% K₂CO₃ aqueous solution + glycine activator + DEA + V₂O₅ inhibitor
  - Model solvent as: K₂CO₃(aq) at 30 wt%, temperature 106°C (semilean, lower) and 70°C (lean, upper)
  - If using `Sep` block: specify CO₂ removal = 99.8% (achieve outlet < 0.03 mol% CO₂)
- **Column**: 5 packed beds, stainless steel random packing
- **Inlet gas**: LT Shift outlet gas at 107°C, 30 bar(g)
- **Outlet gas**: CO₂ < 0.03 mol% → methanator feed
- **CO₂ product**: Gaseous CO₂ overhead from regenerator → export to urea plant (report as separate product stream: S-CO2-EXPORT at ~41,500 Nm³/hr)

**Block F-3301 — HP Regenerator**
- Type: `RadFrac` stripped or Flash2 block
- Operating P: 1.0 bar(g)
- Strips bulk CO₂ from rich solution → semilean solution
- Thermal input: LP steam reboiler

**Block F-3302 — LP Regenerator**
- Type: Flash2 or `RadFrac`
- Operating P: 0.1 bar(g) (subatmospheric)
- Uses flash energy for deeper stripping → lean solution returned to F-3303

**Block TX-3301 — Hydraulic Turbine (Pressure Recovery)**
- Type: `Compr` block (turbine mode, isentropic efficiency = 0.75)
- Inlet: Rich K₂CO₃ solution at 30 bar(g)
- Outlet: 1.0 bar(g)
- **Shaft work recovered**: Connect work stream to P-3301 pump (partial duty credit)

---

### SECTION D — METHANATION

**Block E-3311 — Feed/Effluent Gas-Gas Heat Exchanger**
- Type: `HeatX`
- Hot side: Methanator outlet (~339°C) → 107°C
- Cold side: Purified gas from F-3303 → 290°C (methanator inlet)

**Block R-3311 — Methanator**
- Type: `RGibbs` or `REquil` (two beds — model as single adiabatic block)
- Catalyst: PK-5 (~27% Ni), 60 m³ total (2 beds)
- **Inlet T**: 290–320°C (set via E-3311 design spec), P: 30.0 bar(g)
- **Maximum T**: 420°C (verify; use `Sensitivity` to confirm no overtemperature)
- Reactions (set to equilibrium):
  - CO + 3H₂ → CH₄ + H₂O
  - CO₂ + 4H₂ → CH₄ + 2H₂O
- **Outlet spec**: CO + CO₂ < 10 ppm combined (verify with stream results)
- **Temperature diagnostic**: Report ΔT across bed; normal = 18–19°C for <0.03% CO₂ feed

**Block E-3312 — Methanator Outlet Cooler**
- Type: Heater or HeatX
- Cool from ~107°C (after E-3311) → 38°C
- Utility: Cooling water

**Block B-3311 — Knockout Drum (Condensate Separator)**
- Type: Flash2
- T = 38°C, P = 29.5 bar(g)
- Separate condensed water from dry syngas
- Liquid: Send to condensate handling
- Vapour: Dry syngas → synthesis compression

---

### SECTION E — SYNTHESIS GAS COMPRESSION

**Block K-3401 — Syngas Compressor (Multi-stage)**
- Type: `MCompr` block (3 stages with interstage cooling)
- Inlet: Dry syngas from B-3311 at 38°C, 29.5 bar(g)
- Outlet: 225 bar(g)
- Isentropic efficiency per stage: 0.78
- Mechanical efficiency: 0.96
- Interstage coolers (E-3401A/B/C): Cool to 40°C after each stage; model as Heater blocks; utility = cooling water
- Interstage knockout drums (B-3401A/B/C): Flash2 blocks at 40°C to remove condensate
- **Report**: Total shaft power (MW), interstage pressures, interstage cooler duties

---

### SECTION F — AMMONIA SYNTHESIS LOOP

**Loop components** (all at ~220 bar(g) unless noted):

**Block H-3501 — Loop Startup Heater / Trim Heater**
- Type: Heater
- Duty: Trim feed syngas to 360°C (synthesis converter inlet minimum)
- Utility: HP steam (startup only; normally bypassed once loop is self-sustaining)

**Block R-3501 — Ammonia Synthesis Converter (Radial Flow)**
- Type: `RGibbs` (two-bed, with inter-bed cold shot modelled as a mixer + splitter)
- Catalyst: KMIR (promoted Fe), 109.3 m³ total
  - Bed 1: 29.0 m³
  - Bed 2: 80.3 m³
- **Inlet T**: 360–400°C (Bed 1), P: 220 bar(g)
- **Bed 1 outlet T**: ~500–510°C
- **Cold shot system**: Model as:
  1. Split a fraction of make-up syngas (cold shot stream)
  2. Mix cold shot with Bed 1 outlet to achieve 370°C Bed 2 inlet
  3. Use `Design Spec` to determine cold shot fraction
- **Bed 2 outlet T**: ~455°C
- **Per-pass conversion**: ~28–32% (N₂ basis) — result of RGibbs equilibrium at conditions
- **Hot spot constraint**: Verify T in both beds ≤ 520°C (use Sensitivity block)
- Reaction: N₂ + 3H₂ ⇌ 2NH₃ (ΔH = –92 kJ/mol)

**Block E-3501 — Converter Outlet WHB (Waste Heat Boiler)**
- Type: HeatX
- Hot side: R-3501 outlet, 455°C → 320°C
- Cold side: BFW → HP steam at 125 bar(g), 510°C
- Duty: Calculate (significant HP steam generation)

**Blocks E-3502 through E-3507 — Loop Cooler Train**
Model as sequential Heater blocks or HeatX blocks:
- E-3502: Process-to-process (cool outlet gas, heat make-up syngas)
- E-3503: Air cooler (if applicable) or trim cooler
- E-3504: CW cooler to ~40°C
- E-3506: Ammonia refrigerant chiller to 18.8°C
- E-3507: Ammonia refrigerant chiller to 12°C

**Block B-3501 — High Pressure NH₃ Separator**
- Type: Flash2
- T = 12°C, P = 220 bar(g)
- Vapour: Recycle gas (H₂, N₂, CH₄, Ar, residual NH₃) → recycle compressor
- Liquid: Crude liquid NH₃ (contains dissolved gases) → letdown

**Block K-3501 — Recycle Gas Compressor**
- Type: `Compr` (centrifugal, isentropic efficiency = 0.80)
- Inlet: Recycle gas from B-3501 vapour
- Outlet: 220 bar(g)
- Mix outlet with make-up syngas before H-3501

**Loop convergence**: Use Aspen Plus `Convergence` → `Tear` stream on the recycle gas line. Target tolerance: 0.01% on all stream flows.

---

### SECTION G — PURGE, WASH & SEPARATION

**Block E-3511 — Purge Gas Chiller**
- Type: Heater
- Cool purge gas from ~18.8°C to –25°C
- Utility: Refrigerant (NH₃ refrigeration system — model as a simple utility at –30°C)

**Block B-3511 — Chilled Purge Separator**
- Type: Flash2
- T = –25°C, P = 215 bar(g)
- Liquid NH₃ recovered → return to B-3501
- Vapour → water wash absorbers

**Block F-3522 — Low-Pressure Ammonia Wash Absorber**
- Type: `RadFrac` or `Sep` block
- P: 15 bar(g) (letdown purge gas)
- Solvent: Demineralised water at 43°C (lean), recirculated
- Specify: NH₃ in exit gas < 0.02 mol%
- Overhead gas (H₂ + N₂ + CH₄ + Ar): Route to H-3201 as fuel supplement
- Bottom (dilute NH₃ water): Route to F-3521 distillation

**Block F-3523 — High-Pressure Inert Gas Wash Absorber**
- Type: `RadFrac` or `Sep` block
- P: 81 bar(g) (letdown / inert purge from loop)
- Same wash water circuit as F-3522
- Overhead gas: Route to H-3201 fuel

**Block F-3521 — Ammonia Distillation Column**
- Type: `RadFrac`
- 20 theoretical trays (or equivalent packed height)
- **Operating P**: 25 bar(g) (so overhead NH₃ condenses at ~60°C with cooling water)
- **Overhead**: Liquid NH₃, purity > 99.9 vol% → product split
- **Bottoms**: Lean water (< 0.05 mol% NH₃) → recycle to F-3522/F-3523
- **Condenser**: E-3521 — cooling water utility (CW at 30°C in, 45°C out)
- **Reboiler**: E-3522 — LP steam utility

**Product Split:**
From F-3521 overhead liquid NH₃:
- Stream S-NH3-GAS: Flash to ~5 bar(g) at 20°C → gaseous NH₃ product (report flow in Nm³/hr and kg/hr)
- Stream S-NH3-LIQ: Subcool to –33°C and/or maintain at 12 bar(g) at ambient → liquid NH₃ product (kg/hr)
- Ratio between gas/liquid products: set by Design Spec targeting total = 100 TPD combined

---

### SECTION H — CONDENSATE STRIPPING

**Block F-3321 — Process Condensate Stripper**
- Type: `RadFrac` or simple steam-stripping column
- Feed: Accumulated condensate from B-3311, interstage knockout drums, and any other aqueous knockout streams
- Operating P: 38 bar(g)
- Stripping agent: Live LP steam injection (bottom)
- **Overhead**: Steam + NH₃ + CO₂ + methanol → route to H-3201 reformer inlet (join before E-3201)
- **Bottoms**: Stripped condensate at ~180°C → cool to 45°C via E-3321 (HeatX with BFW or CW) → demineraliser feed

**Block B-3202 — Overhead Knockout Drum**
- Type: Flash2
- T = 180°C, P = 38 bar(g)
- Liquid: Return to F-3321
- Vapour: Clean steam → reformer

---

## PROPERTY METHOD SPECIFICATIONS

| Section | Property Method | Reason |
|---------|----------------|---------|
| Feed prep, reforming, shift | **PR-BM** (Peng-Robinson with Boston-Mathias) | High-T, high-P gas-phase systems |
| CO₂ removal (GV system) | **ENRTL-RK** or **ElecNRTL** | Electrolyte K₂CO₃/KHCO₃ system |
| Methanation | **PR-BM** | High-T gas phase |
| Synthesis loop, compression | **PR-BM** | High-P (220 bar) gas/liquid NH₃ |
| NH₃ distillation | **NRTL** or **SRK** | Liquid-vapour NH₃-H₂O system |
| Condensate stripping | **NRTL** | Dilute aqueous NH₃/CO₂ system |

**Component list (register all in Aspen Plus):**
H₂, N₂, CH₄, CO, CO₂, H₂O, NH₃, Ar, C₂H₆, C₃H₈, n-C₄H₁₀, H₂S, CH₃OH (methanol trace in condensate), ZnO (solid, inert — register for mass balance only)

---

## DESIGN SPECIFICATIONS (MANDATORY)

The following `Design Specs` must be active and converged in the simulation:

| DS-ID | Variable Manipulated | Target | Tolerance |
|-------|---------------------|--------|-----------|
| DS-01 | NG feed flow rate | Total NH₃ production = 100 TPD | ±0.5 TPD |
| DS-02 | Process air flow to R-3203 | H₂/N₂ ratio in synthesis loop feed = 3.00 | ±0.01 |
| DS-03 | Process steam flow | W/C ratio at R-3206/H-3201 inlet = 3.30 | ±0.02 |
| DS-04 | Cold shot split fraction to R-3501 Bed 2 | Bed 2 inlet T = 370°C | ±2°C |
| DS-05 | Purge split fraction from synthesis loop | Ar + CH₄ inert level in loop = 12–15 mol% | ±1 mol% |
| DS-06 | H-3201 outlet T (via firing rate) | R-3203 inlet CH₄ = 10.5 mol% (dry) | ±0.3 mol% |

---

## SENSITIVITY ANALYSES (REQUIRED IN FILE)

Set up but do not necessarily run — mark as "inactive" in final file:

| SA-ID | Independent Variable | Dependent Variable | Range |
|-------|---------------------|-------------------|-------|
| SA-01 | H-3201 outlet T | CH₄ slip at R-3203 outlet | 770–830°C |
| SA-02 | Synthesis P (R-3501) | Per-pass NH₃ conversion | 180–250 bar |
| SA-03 | Purge split fraction | NH₃ production rate vs. H₂ loss | 0.01–0.10 |
| SA-04 | R-3311 inlet T | Methanator outlet CO+CO₂ | 280–330°C |

---

## HEAT EXCHANGER SUMMARY — REQUIRED OUTPUTS (D3)

For every HX block in the flowsheet, report the following in a consolidated table:

| Column | Units |
|--------|-------|
| Tag number | — |
| Equipment name | — |
| Hot side fluid | — |
| Hot side Tin | °C |
| Hot side Tout | °C |
| Hot side flow | kg/hr |
| Cold side fluid | — |
| Cold side Tin | °C |
| Cold side Tout | °C |
| Cold side flow | kg/hr |
| Duty | Gcal/hr and MW |
| LMTD | °C |
| U assumed | kcal/m²·hr·°C |
| Area calculated | m² |
| Utility type | HP steam / CW / Refrigerant / Flue gas / Process-to-process |

---

## COMPLETE EQUIPMENT LIST — REQUIRED SPECIFICATIONS (D2)

The following tags must appear in the equipment summary. Add any additional equipment identified during simulation.

| Tag | Type | Key Design Parameter |
|-----|------|----------------------|
| E-3204 | Shell & tube HX | NG preheater, flue gas / NG |
| R-3201 | Fixed-bed reactor | HDS, Ni-Mo, 9.0 m³, 380°C, 35 bar |
| R-3202A | Fixed-bed reactor | ZnO absorber, 30 m³ |
| R-3202B | Fixed-bed reactor | ZnO absorber, 30 m³ (series with A) |
| E-3201 | Coil (convection section) | Feed/steam preheater, flue gas |
| R-3206 | Fixed-bed reactor | Prereformer, RKNGR-7H, 23.4 m³, 490°C |
| H-3201 | Fired tubular furnace | Primary reformer, 288 tubes, R-67-7H, 42.8 m³, 800°C |
| R-3203 | Refractory-lined reactor | Secondary reformer, RKS-2-7H, 39 m³, 990°C |
| E-3206 | Waste heat boiler | Reformate cooler / HP steam gen, 990°C → 350°C |
| R-3204 | Fixed-bed reactor | HT shift, SK201-2 Fe/Cr, 92.28 m³, 350–440°C |
| E-3208 | Shell & tube HX | HT/LT shift interstage cooler |
| R-3205 | Fixed-bed reactor | LT shift, LK-801-S Cu/Zn, 122 m³, 200–230°C |
| E-3209 | Shell & tube HX | LT shift outlet cooler |
| F-3303 | Packed absorber | CO₂ absorber, 5 SS beds, K₂CO₃ system |
| F-3301 | Packed column | HP regenerator, 1.0 bar |
| F-3302 | Packed column | LP regenerator, 0.1 bar |
| TX-3301 | Hydraulic turbine | Rich solution pressure recovery, η=0.75 |
| P-3301 | Centrifugal pump | Lean solution recirculation |
| E-3311 | Shell & tube HX | Methanator feed/effluent HX |
| R-3311 | Fixed-bed reactor | Methanator, PK-5 Ni, 60 m³, 290–420°C |
| E-3312 | Shell & tube HX | Methanator outlet cooler, CW |
| B-3311 | Knockout drum | Condensate separator, 38°C |
| K-3401 | Centrifugal compressor (3-stage) | Syngas compressor, 29.5 → 225 bar |
| E-3401A/B/C | Shell & tube HX | Interstage coolers, CW |
| B-3401A/B/C | Knockout drum | Interstage condensate separators |
| H-3501 | Electric / steam heater | Loop trim heater, 360°C |
| R-3501 | Radial flow converter | NH₃ converter, KMIR Fe, 109.3 m³, 220 bar, ≤520°C |
| E-3501 | Waste heat boiler | Converter effluent WHB, HP steam gen |
| E-3502–E-3507 | HX train | Loop cooling train (various utilities) |
| B-3501 | HP separator | NH₃ / gas separator, 12°C, 220 bar |
| K-3501 | Centrifugal compressor | Recycle compressor, 220 bar |
| E-3511 | Refrigerant chiller | Purge gas chiller, –25°C |
| B-3511 | Separator | Chilled purge separator |
| F-3522 | Packed absorber | Low-P NH₃ wash, 15 bar, DM water |
| F-3523 | Packed absorber | High-P NH₃ wash, 81 bar, DM water |
| F-3521 | Distillation column | NH₃ distillation, 20 trays, 25 bar |
| E-3521 | Condenser | NH₃ column overhead condenser, CW |
| E-3522 | Reboiler | NH₃ column reboiler, LP steam |
| F-3321 | Stripping column | Condensate stripper, 38 bar |
| E-3321 | Shell & tube HX | Stripped condensate cooler |
| B-3202 | Knockout drum | Stripper overhead separator |

---

## CONVERGENCE STRATEGY

Work through the flowsheet in this order to achieve initial convergence before enabling all recycles:

1. **Open-loop pass**: Fix all recycle streams with estimates; converge feed prep → reforming → shift → CO₂ removal → methanation in sequence.
2. **Synthesis loop**: Close the synthesis loop recycle tear stream using `Wegstein` method. Initial estimate: recycle gas = 75% N₂+H₂, 12% CH₄, 3% Ar, 10% NH₃.
3. **Purge and wash**: Close the NH₃ wash water recycle.
4. **All Design Specs active**: Run with all DS active simultaneously; use `Sequential Modular` convergence.
5. **Final check**: Verify global mass balance — nitrogen atoms in = nitrogen atoms in product NH₃ + purge.

If the GV electrolyte section does not converge, substitute with a `Sep` block specifying:
- CO₂ removal fraction = 0.998
- H₂O removal fraction = 0.10 (condensation)
- All other components pass through

---

## VALIDATION CHECKS (RUN BEFORE SUBMITTING)

The simulation is considered complete when ALL of the following are satisfied:

- [ ] Total NH₃ production = 100.0 ± 0.5 TPD (gaseous + liquid combined)
- [ ] H₂/N₂ molar ratio in synthesis converter feed = 3.00 ± 0.02
- [ ] S content after R-3202B < 0.05 ppm (wt)
- [ ] CH₄ at H-3201 outlet (dry) = 10–11 mol%
- [ ] CH₄ at R-3203 outlet (dry) < 0.35 mol%
- [ ] CO at R-3204 outlet (dry) < 4.5 mol%
- [ ] CO at R-3205 outlet (dry) < 0.35 mol%
- [ ] CO₂ at F-3303 outlet < 0.03 mol%
- [ ] CO + CO₂ at R-3311 outlet < 10 ppm
- [ ] NH₃ converter hot spot T ≤ 520°C (both beds)
- [ ] Inert level (Ar + CH₄) in synthesis loop = 12–15 mol%
- [ ] NH₃ product purity (F-3521 overhead) > 99.9 vol%
- [ ] Global nitrogen balance closure > 99.5%
- [ ] Global carbon balance closure > 99.5%
- [ ] All Design Specs show "Converged" status
- [ ] No blocks show "Error" or "Warning — not converged" status

---

## OUTPUT FILES REQUIRED

1. **NH3_Plant_Simulation.apwz** — Main Aspen Plus file (all sections, all design specs, all sensitivity blocks)
2. **NH3_Stream_Table.xlsx** — Full stream summary exported from Aspen Plus
3. **NH3_Equipment_List.xlsx** — Equipment list with technical specs (tag, type, size, T, P, duty, material note)
4. **NH3_HX_Utility_Summary.xlsx** — Heat exchanger utility load table (all HX blocks)
5. **NH3_Energy_Balance_Summary.xlsx** — Summary of major utility consumers and producers:
   - H-3201 fired duty (Gcal/hr)
   - Total HP steam generated (kg/hr) from WHBs
   - Total cooling water duty (Gcal/hr)
   - Refrigeration duty (Gcal/hr at –30°C level)
   - Total compression power (MW): K-3401 + K-3501
   - Net energy exported (HP steam to battery limit)

---

## NOTES FOR AGENT

- **Do not use shortcut methods** (`DSTWU`, `SCFrac`) for any column — use `RadFrac` throughout.
- **Always report stream results on both molar and mass basis** — plant operators work in kg/hr and TPD.
- **Label every stream** with a descriptive tag: e.g., `S-NG-FEED`, `S-SYNGAS-COMP-IN`, `S-NH3-PRODUCT-GAS`.
- **Add Aspen Plus annotations** on the flowsheet: section headers (A, B, C), equipment tag labels, and stream composition callouts for key streams.
- **Property method switching**: Use the `Property Method Override` feature on individual blocks where the section-specific method differs from the global method.
- **Refrigeration system**: Model the NH₃ refrigeration loop for chilling as a utility specification (set T = –30°C, P = cooling water condensing equivalent) rather than a full refrigeration cycle, unless specifically requested.
- **Flue gas from H-3201**: Model flue gas heat recovery by creating a pseudo-flue gas stream and routing it through E-3204 and E-3201 as a hot utility stream; do not model combustion air + burner explicitly unless the agent has specific instruction to do so.

---

*End of Agent Prompt — Version 1.0*
*Source data: NH₃ Plant Process Logic Guide (10-step flowsheet)*
*Target capacity: 100 TPD total ammonia (gaseous + liquid)*