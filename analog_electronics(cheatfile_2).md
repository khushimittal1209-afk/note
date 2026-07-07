# Analog Electronics — OA Cheat Sheet (Part 2/3)
## Transistor Family: BJT, Biasing, Operating Regions, CE/CB/CC, Load Line, MOSFET, MOSFET Regions/Characteristics, CMOS

---

## 1. BJT (Bipolar Junction Transistor)
- **Concept:** Current-controlled 3-terminal device (Emitter, Base, Collector) — a small base current controls a much larger collector current. Two PN junctions back-to-back (NPN or PNP).
- **Formulas:**
  - `I_E = I_B + I_C`
  - Current gain: `β (h_FE) = I_C/I_B` (typical 50–300); `α = I_C/I_E = β/(β+1)` (typical 0.95–0.99).
  - `I_C = βI_B`, and relation: `β = α/(1−α)`.
- **Curve:** Output characteristics — family of I_C vs V_CE curves for different I_B values; curves are flat (active region) after initial knee, showing near-constant I_C for a given I_B.
```
 I_C
  |   IB=40µA  ______
  |   IB=30µA ______
  |   IB=20µA ______
  |   IB=10µA ______
  |__________________ V_CE
   (knee)  saturation → active (flat) → breakdown
```
- **Circuit:** NPN symbol — arrow on emitter points OUT (away from base); PNP — arrow points IN. Basic circuit: `V_CC → R_C → Collector; Base → R_B → V_BB; Emitter → GND`.
- **MCQ patterns:** "Given I_B and β, find I_C and I_E", "identify NPN vs PNP from arrow direction", "find α given β or vice versa".
- **Traps:** Mixing up NPN (arrow out = "Not Pointing iN") vs PNP (arrow in = "Points iN Proudly"); using α and β formulas interchangeably without converting.
- **Memory trick:** NPN → "Not Pointing iN" (arrow away from base); current gain: "β is Big, α is Almost 1".
- **Application:** Amplifiers, switches, oscillators — foundation of analog and digital circuits before MOSFETs dominated.

---

## 2. Biasing (BJT DC Biasing)
- **Concept:** Setting a stable DC operating point (Q-point) using resistors so the transistor operates in the active region despite temperature/β variations.
- **Formulas:**
  - Fixed bias: `I_B = (V_CC − V_BE)/R_B` (simple but poor stability — depends directly on β).
  - Voltage divider bias (most common, most stable): `V_B = V_CC·R2/(R1+R2)`, `I_E ≈ (V_B − V_BE)/R_E`, `I_C ≈ I_E`.
  - Stability factor `S = ΔI_C/ΔI_CO` — lower S = more stable (voltage divider has lowest S).
- **Curve:** Q-point plotted on output characteristics — should sit near mid-point of load line for maximum symmetric swing (avoid clipping).
- **Circuit:** Voltage divider bias: `R1` (VCC to Base), `R2` (Base to GND), `R_C` (VCC to Collector), `R_E` (Emitter to GND) — self-stabilizing via R_E (emitter feedback).
- **MCQ patterns:** "Find Q-point (I_C, V_CE) given bias resistors", "which biasing method is most stable/least stable?", "effect of R_E on stability (increases stability)".
- **Traps:** Forgetting V_BE (0.7V) drop when finding V_E or I_E; assuming fixed bias is preferred (it's actually the LEAST stable — very β-dependent).
- **Memory trick:** "Voltage divider bias = most stable, Fixed bias = least stable, R_E = self-correcting feedback."
- **Application:** Setting reliable operating conditions in amplifier design regardless of transistor-to-transistor β variation.

---

## 3. Operating Regions (BJT)
- **Concept:** BJT has 4 regions based on junction bias states — determines whether it acts as amplifier, switch (ON/OFF), or is damaged.
- **Formulas/Conditions:**
  | Region | Base-Emitter Junction | Base-Collector Junction | Use |
  |---|---|---|---|
  | Cutoff | Reverse | Reverse | Switch OFF (I_C≈0) |
  | Active | Forward | Reverse | Amplification (linear) |
  | Saturation | Forward | Forward | Switch ON (V_CE≈0.2V, max I_C) |
  | Breakdown | — | Excess reverse V | Damage (avoid) |
- **Curve:** On output characteristics — Cutoff = bottom (I_C≈0 line), Saturation = near-vertical rise close to V_CE≈0, Active = the flat, evenly spaced region in between.
- **Circuit:** Same BJT circuit; region depends on bias voltages applied, not circuit topology.
- **MCQ patterns:** "Identify BJT region given V_BE, V_CE values", "for amplifier operation, which region must BJT stay in?" (Active), "for switching applications?" (Cutoff & Saturation only).
- **Traps:** Thinking saturation means "fully ON forever" without checking V_CE (should be near 0.2V, NOT 0V exactly, and NOT negative); assuming active region is used for switches (switches use cutoff/saturation, NOT active).
- **Memory trick:** "Amplifiers live in Active; Switches jump between Cutoff and Saturation."
- **Application:** Active → analog amplifiers; Cutoff/Saturation → digital switching, logic gates (BJT-based).

---

## 4. CE, CB, CC Configurations
- **Concept:** Three ways to connect a BJT in a circuit, based on which terminal is common to both input and output — each gives different gain/impedance characteristics.
- **Formulas (typical qualitative values):**
  | Config | Voltage Gain | Current Gain | Input Z | Output Z | Phase shift |
  |---|---|---|---|---|---|
  | CE (Common Emitter) | High | High | Medium | Medium-High | 180° |
  | CB (Common Base) | High | ≈1 (=α) | Low | High | 0° |
  | CC (Common Collector / Emitter Follower) | ≈1 | High | High | Low | 0° |
- **Curve:** CE — inverted (180° out of phase) amplified sine output. CB/CC — in-phase output, CC output nearly same amplitude as input (follows input, hence "follower").
- **Circuit:** CE = signal in at Base, out at Collector, Emitter grounded/common. CB = signal in at Emitter, out at Collector, Base grounded/common (often via capacitor). CC = signal in at Base, out at Emitter, Collector grounded/common (tied to VCC via no signal-path resistor).
- **MCQ patterns:** "Identify configuration from circuit diagram (which terminal is grounded/common)", "which configuration is used for impedance matching/buffering?" (CC), "which gives voltage AND current gain?" (CE), "which has 180° phase shift?" (CE only).
- **Traps:** Assuming all configs invert phase (only CE does); confusing "common" terminal meaning — it's the terminal shared between input and output loop, not necessarily grounded to 0V DC.
- **Memory trick:** "CE = most versatile (used most), CB = current buffer/high-freq, CC = voltage follower/buffer (impedance matching)."
- **Application:** CE → general-purpose amplification; CB → high-frequency circuits, current buffer; CC → impedance matching, output stage buffer (emitter follower).

---

## 5. Load Line Analysis
- **Concept:** A straight line drawn on the transistor's output characteristics representing all possible (I_C, V_CE) combinations for a given circuit (R_C, V_CC) — intersection with a specific I_B curve gives the Q-point.
- **Formulas:**
  - DC load line equation: `V_CE = V_CC − I_C·R_C`.
  - Y-intercept (I_C axis): `I_C = V_CC/R_C` (when V_CE=0, saturation point).
  - X-intercept (V_CE axis): `V_CE = V_CC` (when I_C=0, cutoff point).
- **Curve:** Straight line from `(V_CE=0, I_C=V_CC/R_C)` to `(V_CE=V_CC, I_C=0)`, superimposed on the I_C-V_CE output characteristic curves. Q-point = intersection with the biasing I_B curve.
```
 I_C
 Vcc/Rc |\
        | \
        |  \  ← load line
   Q----|---\●  (Q-point: intersection with IB curve)
        |    \
        |_____\____ V_CE
              Vcc
```
- **Circuit:** Same basic CE amplifier circuit (V_CC, R_C, transistor) — load line depends only on V_CC and R_C values.
- **MCQ patterns:** "Draw/identify load line given V_CC and R_C", "find Q-point given I_B and load line", "find max symmetric swing given Q-point position on load line".
- **Traps:** Forgetting AC load line differs from DC load line if there's a separate AC load resistance (R_C || R_L); placing Q-point too close to saturation or cutoff (causes one-sided clipping, not max swing).
- **Memory trick:** "Load line: two endpoints — Saturation (max I_C, V_CE=0) and Cutoff (I_C=0, V_CE=V_CC). Q-point in the middle = best amplifier design."
- **Application:** Amplifier design — ensures maximum undistorted output swing by centering Q-point on load line.

---

## 6. MOSFET
- **Concept:** Voltage-controlled 3-terminal device (Gate, Source, Drain) — gate voltage creates/controls a conducting channel between source and drain via electric field (no gate current in steady state, unlike BJT's base current).
- **Formulas:**
  - Threshold voltage `V_TH` — minimum V_GS to turn on channel (enhancement mode).
  - Overdrive voltage: `V_OV = V_GS − V_TH`.
  - Saturation current (square-law): `I_D = ½k_n(W/L)(V_GS−V_TH)²(1+λV_DS)`, often simplified `I_D = k(V_GS−V_TH)²`.
- **Curve:** Transfer characteristic (I_D vs V_GS) — no current until V_TH, then parabolic (square-law) rise. Output characteristic (I_D vs V_DS) — similar family-of-curves shape to BJT but curves parameterized by V_GS instead of I_B.
- **Circuit:** Gate is insulated (via oxide layer — MOS = Metal-Oxide-Semiconductor) → draws essentially zero DC gate current → very high input impedance.
- **MCQ patterns:** "Find I_D given V_GS, V_TH, and k", "why does MOSFET have higher input impedance than BJT?" (insulated gate, no gate current), identify enhancement vs depletion type.
- **Traps:** Assuming gate draws current like BJT base (it does NOT in DC/steady-state — only displacement current during switching); forgetting square-law relationship (doubling V_OV quadruples I_D, not doubles).
- **Memory trick:** "MOSFET = Voltage-controlled, BJT = Current-controlled." Gate is like a capacitor plate — no DC current flows through it.
- **Application:** Digital logic (CMOS), switching power supplies, low-power amplifiers — dominant device in modern ICs due to high input impedance and low power.

---

## 7. MOSFET Regions of Operation
- **Concept:** MOSFET has 3 operating regions depending on V_GS and V_DS — determines whether it acts as OFF switch, variable resistor, or constant-current amplifier.
- **Formulas/Conditions:**
  | Region | Condition | Behavior |
  |---|---|---|
  | Cutoff | `V_GS < V_TH` | OFF, I_D ≈ 0 |
  | Triode/Linear | `V_GS > V_TH` AND `V_DS < V_GS−V_TH` | Acts as voltage-controlled resistor |
  | Saturation | `V_GS > V_TH` AND `V_DS ≥ V_GS−V_TH` | Acts as constant-current source (used for amplification) |
- **Curve:** On I_D-V_DS output curve — Triode = initial rising linear portion (near origin); Saturation = flat plateau region (curves become horizontal).
```
 I_D
     |        _________ (saturation, flat)
     |      /
     |    /   (triode, linear rise)
     |__/___________________ V_DS
        ↑
   V_DS = V_GS - V_TH (boundary)
```
- **Circuit:** Same NMOS/PMOS circuit — region is set by applied gate and drain voltages relative to source.
- **MCQ patterns:** "Identify MOSFET region given V_GS, V_DS, V_TH values", "which region is used for amplifiers?" (Saturation — same naming confusion as BJT: MOSFET's 'saturation' = BJT's 'active'!), "which region for switches?" (Cutoff & Triode).
- **Traps:** **CRITICAL NAMING TRAP:** MOSFET "saturation" region = constant current = analogous to BJT's ACTIVE region (used for amplification); MOSFET "triode" region = resistive = analogous to BJT's SATURATION region (used for switching ON). The names are swapped compared to BJT — a very common MCQ trap!
- **Memory trick:** "MOSFET Saturation = Amplifier region (opposite naming from BJT!). Triode = ON-switch/resistor region."
- **Application:** Saturation region → analog amplifiers; Triode + Cutoff → digital switches (CMOS logic).

---

## 8. MOSFET Characteristics (Types)
- **Concept:** MOSFETs come in Enhancement mode (channel must be created by V_GS) and Depletion mode (channel exists by default, can be depleted) — each in NMOS or PMOS polarity.
- **Formulas:**
  - Enhancement NMOS: OFF at V_GS=0, turns ON when `V_GS > +V_TH`.
  - Depletion NMOS: ON at V_GS=0, turns OFF when `V_GS < −V_TH` (negative threshold).
- **Curve:** Enhancement — transfer curve starts at 0 and only rises after V_TH (curve entirely in one quadrant). Depletion — transfer curve already has current at V_GS=0, extending into negative V_GS territory before hitting 0.
- **Circuit:** Symbol difference — enhancement MOSFET has broken/segmented channel line in symbol; depletion MOSFET has solid unbroken channel line.
- **MCQ patterns:** "Identify enhancement vs depletion from transfer curve or symbol", "which type is OFF by default?" (Enhancement), "which type conducts at V_GS=0?" (Depletion).
- **Traps:** Assuming all MOSFETs are enhancement type (depletion type does exist and behaves oppositely at V_GS=0); confusing symbol line style (solid=depletion, broken=enhancement).
- **Memory trick:** "Enhancement = Effort needed to turn ON (needs +V_GS). Depletion = Default ON, needs effort to turn OFF."
- **Application:** Enhancement mode dominates modern digital logic (CMOS); depletion mode used in some analog/RF and older NMOS logic designs.

---

## 9. CMOS Basics
- **Concept:** Complementary MOS — uses PMOS and NMOS transistor pairs together so that in any static logic state, only one type conducts — ideal for very low static power consumption.
- **Formulas:**
  - Static power ≈ 0 (only switching/dynamic power and small leakage matter): `P_dynamic = C·V²·f`.
  - PMOS conducts when input=LOW (0); NMOS conducts when input=HIGH (1) — complementary switching.
- **Curve:** Voltage Transfer Characteristic (VTC) of CMOS inverter — sharp transition region between V_OH (output high) and V_OL (output low), giving good noise margin; near-rail-to-rail output swing.
- **Circuit:** CMOS Inverter — PMOS on top (source to V_DD), NMOS on bottom (source to GND), both gates tied to input, both drains tied to output.
```
      V_DD
       |
     [PMOS] --- gate=input
       |
    ---o--- output
       |
     [NMOS] --- gate=input
       |
      GND
```
- **MCQ patterns:** "Identify which transistor conducts for given input (0 or 1) in CMOS inverter", "why is CMOS preferred over NMOS-only logic?" (lower static power), find output logic level for given input.
- **Traps:** Forgetting that only ONE of PMOS/NMOS conducts at a time in steady state (during switching, brief moment both conduct → causes small "shoot-through" dynamic power spike); assuming CMOS has zero power always (dynamic/switching power still exists, scales with frequency).
- **Memory trick:** "CMOS Inverter: Input LOW → PMOS ON → Output HIGH. Input HIGH → NMOS ON → Output LOW" (it's an inverter, so output flips input).
- **Application:** Virtually all modern digital ICs (processors, memory, logic gates) — chosen for low static power and good noise immunity.

---

## 🔑 Comparison Tables

### BJT vs MOSFET
| Aspect | BJT | MOSFET |
|---|---|---|
| Control | Current-controlled (I_B) | Voltage-controlled (V_GS) |
| Input impedance | Moderate (draws base current) | Very high (gate is insulated, ~no current) |
| Carriers | Both electrons & holes (bipolar) | Single carrier type (unipolar) |
| Speed/switching | Slower (minority carrier storage) | Faster (majority carriers only) |
| Key equation | I_C = βI_B | I_D = k(V_GS−V_TH)² |
| Power consumption | Higher (continuous base current) | Lower (esp. in CMOS static state) |
| Common use | Analog amplifiers, high-current apps | Digital ICs (CMOS), low-power/high-density |

### CE vs CB vs CC
| Aspect | CE | CB | CC |
|---|---|---|---|
| Common terminal | Emitter | Base | Collector |
| Voltage gain | High | High | ≈1 |
| Current gain | High (β) | ≈1 (α) | High (β+1) |
| Input impedance | Medium | Low | High |
| Output impedance | Medium-High | High | Low |
| Phase shift | 180° | 0° | 0° |
| Main use | General amplification | High-frequency/current buffer | Impedance matching/buffer (follower) |

---

**End of Part 2** — reply "next" to continue with Part 3: Op-amp, Inverting/Non-inverting Amplifier, Integrator, Differentiator, Comparator, Schmitt Trigger, Active Filters, Differential Amplifier, Differential Gain, CMRR.
