# Analog Electronics — OA Cheat Sheet (Part 1/3)
## Diode Family: Diodes, PN Junction, Zener, LED, Photodiode, Clippers, Clampers, Rectifiers

---

## 1. Diodes (General / PN Diode)
- **Concept:** A diode conducts current in one direction only (anode→cathode) once forward voltage exceeds cut-in voltage. Acts like a voltage-controlled switch.
- **Formulas:**
  - Shockley eqn: `I = I_s(e^(V/ηV_T) − 1)`, `V_T ≈ 26mV` at room temp.
  - Cut-in voltage: Si ≈ 0.7V, Ge ≈ 0.3V.
  - Ideal diode: 0Ω forward (short), ∞Ω reverse (open).
- **Curve:** I-V characteristic — flat (~0) below cut-in, then sharp exponential rise in forward bias; near-zero leakage current in reverse bias until breakdown.
```
   I
   |            _/
   |          _/
   |________ /______________ V
   |      cut-in (0.7V)
```
- **Circuit:** Series diode + resistor + source: `Source(+) → Diode → R → Source(−)`. Current flows only if forward-biased.
- **MCQ patterns:** "Find current through diode using ideal/practical/2nd-approx model", "is diode ON or OFF", identify bias direction from circuit polarity.
- **Traps:** Forgetting to check polarity before assuming diode conducts; using ideal model when question specifies "practical diode" (must include 0.7V drop).
- **Memory trick:** "Diode = one-way valve for current" — arrow in symbol points in the direction of conventional current flow (forward).
- **Application:** Rectification, signal demodulation, protection circuits (flyback diode).

---

## 2. PN Junction
- **Concept:** Formed by joining P-type (holes) and N-type (electrons) semiconductor. Diffusion creates a depletion region with a built-in potential barrier.
- **Formulas:**
  - Barrier potential: Si ≈ 0.7V, Ge ≈ 0.3V.
  - Depletion width increases with reverse bias, decreases with forward bias.
- **Curve:** Energy band diagram — bands bend at junction; depletion region shown as a barrier zone with no free carriers.
```
  P-side          Depletion         N-side
 [+holes]  |||||| barrier ||||||  [-electrons]
           <-- widens in reverse, shrinks in forward -->
```
- **Circuit:** Not a standalone circuit — it's the internal structure of the diode.
- **MCQ patterns:** "What happens to depletion width under forward/reverse bias?", "which carriers dominate current in each region?", drift vs diffusion current identification.
- **Traps:** Confusing forward bias (narrows depletion, lowers barrier) with reverse bias (widens depletion, raises barrier); mixing up majority vs minority carrier current direction.
- **Memory trick:** Forward bias = "Free flow" (narrow barrier); Reverse bias = "Restricted" (wide barrier).
- **Application:** Foundation for all diode, BJT, and MOSFET behavior.

---

## 3. Zener Diode
- **Concept:** Heavily doped diode designed to operate in reverse breakdown region without damage — used for voltage regulation.
- **Formulas:**
  - Zener regulator: `V_out = V_Z` (constant, as long as I_Z stays between I_Zmin and I_Zmax).
  - Series resistor: `R_S = (V_in − V_Z)/I_Z`.
  - Power rating check: `P_Z = V_Z × I_Z ≤ P_Zmax`.
- **Curve:** I-V curve — normal diode forward curve PLUS a sharp, near-vertical breakdown knee in reverse region at `V_Z` (current rises steeply at constant voltage).
```
 Reverse            Forward
   I                   I
   |                   |    _/
   |                   |  _/
---+------ V   ...   --+-/-------- V
   | \                 |
   |  \___ (breakdown at -V_Z, steep knee)
```
- **Circuit:** Reverse-biased Zener in series with resistor across supply, load in parallel with Zener → simple shunt voltage regulator.
- **MCQ patterns:** "Find load voltage using Zener regulator", "find minimum/maximum load current for regulation", "identify Zener region of operation on graph".
- **Traps:** Zener is deliberately operated in REVERSE breakdown (not forward!) — many forget this is intentional, not a fault; forgetting to check I_Z is within rated range (regulation fails outside range).
- **Memory trick:** "Zener = Reverse hero" — it does its job in reverse bias, unlike normal diodes.
- **Application:** Voltage regulators, reference voltage sources, overvoltage protection (clipping circuits).

---

## 4. LED (Light Emitting Diode)
- **Concept:** Forward-biased PN junction made of compound semiconductor (GaAs, GaP etc.) that emits light when electrons recombine with holes, releasing energy as photons.
- **Formulas:**
  - Photon energy: `E = hf = hc/λ` (wavelength depends on bandgap energy).
  - Series resistor sizing: `R = (V_supply − V_LED)/I_LED`.
- **Curve:** Similar forward I-V curve to normal diode but higher cut-in voltage (~1.8–3.3V depending on color); negligible/unsafe reverse breakdown (must avoid reverse voltage).
- **Circuit:** LED in series with current-limiting resistor across DC supply — polarity matters (longer lead = anode).
```
  V_supply(+) ---[ R ]---|>|--- GND
                        LED (anode-cathode, forward biased)
```
- **MCQ patterns:** "Find resistor value for desired LED current", "why can't LED be used as rectifier?" (low reverse breakdown), color vs bandgap energy relation.
- **Traps:** Assuming LED forward voltage = 0.7V like Si diode (it's higher, varies by color: red≈1.8V, blue/white≈3–3.3V); forgetting current-limiting resistor is mandatory (steep I-V curve → no resistor = burnout).
- **Memory trick:** "Bigger bandgap → shorter wavelength → higher V_f" (Blue LED has higher V_f than Red).
- **Application:** Indicator lights, displays, optocouplers, IR remotes.

---

## 5. Photodiode
- **Concept:** Reverse-biased PN junction that generates current proportional to incident light intensity (photogenerated carriers).
- **Formulas:**
  - `I_total = I_dark + I_photo`, where `I_photo ∝ light intensity`.
  - Operates in reverse bias — current flows in reverse direction, magnitude controlled by light.
- **Curve:** Family of I-V curves in the reverse-bias quadrant — parallel horizontal lines shifting to more negative current as light intensity increases.
```
   I
   |____________________ V (forward, mostly unused)
   |‾‾‾‾ dark current
   |____ low light
   |________ high light   (all in reverse/-I region)
```
- **Circuit:** Reverse-biased photodiode in series with resistor — light intensity modulates voltage across resistor (light sensor).
- **MCQ patterns:** "Identify biasing mode for photodiode (reverse)", "how does output current vary with light intensity?", compare with LED (opposite function).
- **Traps:** Confusing photodiode (reverse-biased, senses light → produces current) with LED (forward-biased, current → emits light) — functional opposites; assuming photodiode current is large (it's typically µA range).
- **Memory trick:** "Photo-IN, LED-OUT" — Photodiode converts light IN to current; LED converts current to light OUT.
- **Application:** Light sensors, optical receivers, solar cells (photovoltaic mode), camera sensors.

---

## 6. Clippers (Limiters)
- **Concept:** Circuits that "clip"/cut off part of a waveform above and/or below a certain reference voltage, using diodes — do NOT shift DC level, only reshape amplitude.
- **Formulas:**
  - Clipping level ≈ `V_ref ± V_diode(0.7V)` depending on diode orientation and added bias.
  - Positive clipper removes + portion above threshold; negative clipper removes − portion below threshold.
- **Curve:** Sine input → output waveform flattened (clipped) above/below the threshold level; unclipped portion retains original shape.
```
 Input (sine):        Output (positive clipper at +V):
    /\    /\             __    __
   /  \  /  \      →    /  \  /  \
  /    \/    \         /    \/    \    (top flattened at +V)
```
- **Circuit:** Diode in parallel (shunt) with load, often with a bias battery in series with diode to set clip level; series resistor limits current.
- **MCQ patterns:** "Find clipped output waveform for given input and diode orientation", "identify clip level given bias voltage", series vs shunt clipper identification.
- **Traps:** Forgetting to add/subtract 0.7V diode drop to the bias reference when finding actual clip level; confusing which half-cycle gets clipped based on diode direction (draw current path to check).
- **Memory trick:** "Clippers reshape amplitude only — they do NOT create a DC shift" (that's a clamper's job).
- **Application:** Waveform shaping, protection (limiting voltage to safe range), noise limiters.

---

## 7. Clampers (DC Restorers)
- **Concept:** Circuits that shift the ENTIRE waveform up or down by adding/restoring a DC level, using a diode + capacitor — preserves original waveform SHAPE, only shifts the reference level.
- **Formulas:**
  - Capacitor charges to peak input voltage (approx `V_C = V_m − 0.7V` for practical diode).
  - Output = Input waveform + DC shift (from capacitor).
- **Curve:** Sine input (centered at 0) → output shifted entirely above or below 0, shape unchanged.
```
 Input (sine, centered 0):     Output (positive clamper, shifted up):
      /\                              /\
  ___/  \___  0V line          ______/  \______
      \  /                            \  /       (whole wave lifted above 0)
       \/                              \/
```
- **Circuit:** Series capacitor + shunt diode across output (diode direction sets shift direction) — often called a "DC restorer".
- **MCQ patterns:** "Find peak output voltage after clamping", "identify positive vs negative clamper from diode direction", "will waveform shape change?" (No — only level shifts).
- **Traps:** Confusing clamper with clipper (clamper = shifts level, retains shape; clipper = cuts shape, no shift); forgetting capacitor must be large enough to hold charge across the cycle (RC >> T).
- **Memory trick:** "Clampers CLAMP the wave to a new baseline but keep its shape — think 'elevator', not 'guillotine'."
- **Application:** TV/video sync restoration, level-shifting circuits, peak detector variants.

---

## 8. Rectifiers
- **Concept:** Converts AC input to pulsating DC output using diode(s) — first stage of most power supplies (before filtering/regulation).
- **Formulas:**
  - Half-wave: `V_dc = V_m/π`, `V_rms = V_m/2`, ripple factor `r ≈ 1.21`.
  - Full-wave (center-tap or bridge): `V_dc = 2V_m/π`, `V_rms = V_m/√2`, ripple factor `r ≈ 0.482`.
  - Efficiency: Half-wave ≈ 40.6%, Full-wave ≈ 81.2%.
  - PIV: Half-wave/center-tap = `2V_m`; Bridge = `V_m`.
- **Curve:**
```
 Half-wave output:            Full-wave output:
    _      _                    _   _   _   _
   / \    / \                  / \ / \ / \ / \
__/   \__/   \__ 0        ____/   X   X   \____ 0
 (only + half passes)      (both halves rectified, all positive)
```
- **Circuit:** Half-wave = single diode in series with load across AC source. Full-wave bridge = 4 diodes in bridge configuration, 2 conduct per half-cycle.
- **MCQ patterns:** "Find V_dc/V_rms/ripple factor/efficiency given V_m", "identify rectifier type from output waveform", "find PIV rating needed for diode", compare half-wave vs full-wave performance.
- **Traps:** Using wrong PIV formula (bridge PIV is HALF of center-tap PIV for same V_m — commonly tested); forgetting ripple factor is much worse for half-wave (~2.5× that of full-wave); mixing up which diodes conduct in which half-cycle in bridge rectifier.
- **Memory trick:** "Full-wave = Full efficiency (81%), Half-wave = Half of that, roughly (40%)."
- **Application:** DC power supplies, battery chargers, AM signal demodulation (envelope detection uses half-wave + capacitor).

---

## 🔑 Comparison Tables

### Zener Diode vs Regular Diode
| Aspect | Regular Diode | Zener Diode |
|---|---|---|
| Normal operating region | Forward bias | Reverse breakdown |
| Reverse breakdown | Destructive (avoid) | Designed for it (safe) |
| Doping | Normal | Heavily doped (sharp breakdown) |
| Main use | Rectification, switching | Voltage regulation, reference |
| Key parameter | V_forward (0.7V) | V_Z (breakdown voltage) |

### Rectifier vs Clipper vs Clamper
| Aspect | Rectifier | Clipper | Clamper |
|---|---|---|---|
| Purpose | AC → pulsating DC | Cut off part of waveform | Shift DC level of waveform |
| Shape change? | Yes (only + or both halves pass) | Yes (flattens beyond threshold) | No (shape preserved) |
| DC level shift? | Creates DC component | No shift, just cuts | Yes — that's its main job |
| Key component | Diode(s) only | Diode (+ optional bias) | Diode + Capacitor |
| Typical use | Power supplies | Waveform limiting/protection | DC restoration, level shifting |

---

**End of Part 1** — reply "next" to continue with Part 2: BJT, Biasing, Operating Regions, CE/CB/CC, Load Line, MOSFET, MOSFET Regions/Characteristics, CMOS.
