# Analog Electronics — OA Cheat Sheet (Part 3/3)
## Op-Amp Family: Op-amp, Inverting/Non-inverting Amp, Integrator, Differentiator, Comparator, Schmitt Trigger, Active Filters, Differential Amplifier, Differential Gain, CMRR

---

## 1. Op-Amp (Operational Amplifier) Basics
- **Concept:** High-gain DC-coupled differential amplifier IC with two inputs (inverting − , non-inverting +) and one output. Ideal op-amp assumptions make circuit analysis simple.
- **Formulas (Ideal Op-Amp Rules):**
  - Infinite open-loop gain `A_OL → ∞`.
  - Infinite input impedance → **no current flows into either input terminal**.
  - Zero output impedance.
  - **Virtual short rule** (with negative feedback): `V+ = V−`.
- **Curve:** Open-loop gain vs frequency — very high DC gain (~10⁵–10⁶) that rolls off at low frequency (dominant pole), reaching unity-gain at the Gain-Bandwidth Product (GBW).
- **Circuit:** Triangle symbol, `+` and `−` inputs, single output, powered by dual supply (+V_CC/−V_EE) or single supply.
- **MCQ patterns:** "Apply virtual short/virtual ground to solve for V_out", "why is input current assumed zero?", "find closed-loop gain given feedback network".
- **Traps:** Forgetting virtual short ONLY applies with negative feedback present (open-loop or positive feedback → output just saturates to ±V_sat); assuming real op-amp has literally zero input current (it's just very small, not exactly 0, but treated as 0 for ideal analysis).
- **Memory trick:** "No current in, no voltage difference between inputs (with feedback) — that's the golden rule of ideal op-amps."
- **Application:** Building block for all analog signal processing — amplifiers, filters, oscillators, comparators, ADC/DAC circuits.

---

## 2. Inverting Amplifier
- **Concept:** Input signal applied to inverting (−) terminal through R_in; non-inverting (+) terminal grounded; feedback resistor R_f sets gain. Output is 180° out of phase with input.
- **Formulas:**
  - `A_v = V_out/V_in = −R_f/R_in` (negative sign = inversion).
  - Input impedance ≈ `R_in` (NOT infinite — key difference from non-inverting).
- **Curve:** Sine input → amplified AND inverted sine output (180° phase shift) — peak becomes trough and vice versa.
```
 V_in  (sine, 0-centered)      V_out (inverted, amplified)
    /\        /\                    \    /
   /  \      /  \        →           \  /
      \    /                  \/    \/
```
- **Circuit:** `V_in → R_in → (−) input node → op-amp; R_f connects (−) node to V_out; (+) input grounded.`
```
       Rf
    +--/\/\--+
    |        |
Vin-/\/\-----+---(-)  \
    Rin          op-amp}---- Vout
             +---(+)  /
             |
            GND
```
- **MCQ patterns:** "Find V_out given V_in, R_f, R_in", "find gain given resistor ratio", "why is virtual ground at (−) input, not literal ground?" (0V due to virtual short since (+) is grounded).
- **Traps:** Forgetting the negative sign in gain formula (very common — leads to wrong output polarity); confusing input impedance (it's ≈R_in, finite — NOT high like non-inverting config).
- **Memory trick:** "Inverting gain = −R_f/R_in — negative sign is non-negotiable, always flips phase 180°."
- **Application:** Audio mixers, signal inversion, precision gain stages, summing amplifiers (multiple R_in branches).

---

## 3. Non-Inverting Amplifier
- **Concept:** Input signal applied directly to non-inverting (+) terminal; feedback network (R_f, R_g) from output to inverting (−) terminal sets gain. Output is IN PHASE with input.
- **Formulas:**
  - `A_v = V_out/V_in = 1 + R_f/R_g` (always ≥ 1, always positive — no inversion).
  - Input impedance ≈ very high (≈ op-amp's own input impedance, since signal goes straight into + input, no loading resistor in signal path).
  - Special case: **Voltage follower/buffer** — `R_f=0, R_g=∞` (or just direct feedback wire) → `A_v = 1`.
- **Curve:** Sine input → amplified sine output, SAME phase (no inversion), just scaled in amplitude.
```
 V_in                          V_out (same phase, larger amplitude)
    /\        /\                   /\          /\
   /  \      /  \        →        /  \        /  \
```
- **Circuit:** `V_in → (+) input directly; V_out → R_f → (−) input node → R_g → GND.`
```
Vin ----(+) \
             }---- Vout ---+--- Rf ---+
        (-)  /                        |
             +-------- Rg -------- (-) node
             |
            GND
```
- **MCQ patterns:** "Find gain given R_f, R_g", "identify voltage follower configuration (R_f=0 or direct feedback)", "compare input impedance with inverting config" (much higher here).
- **Traps:** Forgetting the "+1" in the gain formula (common error: writing gain as just R_f/R_g, missing the added 1); assuming gain can be less than 1 (impossible in this basic topology — minimum gain is 1, at voltage follower).
- **Memory trick:** "Non-inverting gain = 1 + R_f/R_g — always at least 1, never inverts, never less than unity."
- **Application:** Buffer stages (impedance matching, isolating source from load), sensor signal conditioning where high input impedance is critical.

---

## 4. Integrator
- **Concept:** Op-amp circuit that produces an output proportional to the TIME INTEGRAL of the input signal — replaces feedback resistor with a capacitor.
- **Formulas:**
  - `V_out = −(1/RC)∫V_in dt`
  - For sine input: acts as a low-pass filter, gain falls at −20dB/decade with increasing frequency.
- **Curve:** Square wave input → **triangular wave** output (integral of a constant is a ramp). Sine input → cosine output (90° phase-shifted, and inverted).
```
 V_in (square wave):        V_out (triangular wave):
  __    __                      /\    /\
 |  |  |  |          →         /  \  /  \
 |  |__|  |__                 /    \/    \
```
- **Circuit:** `V_in → R → (−) input node; Capacitor C connects (−) node to V_out; (+) input grounded.` (Same as inverting amp, but R_f replaced by C.)
- **MCQ patterns:** "Given square wave input, sketch/identify triangular output", "find output expression given V_in(t)", "identify integrator vs differentiator from circuit (R-then-C = integrator)".
- **Traps:** Forgetting the negative sign in output formula; forgetting integrator behaves as LOW-pass filter (attenuates high frequencies) — opposite of differentiator.
- **Memory trick:** "Integrator: R in, C in feedback → square wave IN → triangle wave OUT. Acts as a low-pass filter."
- **Application:** Analog computers, waveform generators (triangle wave generation), ramp generators, PID controllers (the "I" term).

---

## 5. Differentiator
- **Concept:** Op-amp circuit that produces an output proportional to the RATE OF CHANGE (derivative) of the input signal — capacitor at input, resistor in feedback (opposite of integrator).
- **Formulas:**
  - `V_out = −RC(dV_in/dt)`
  - Acts as a high-pass filter, gain rises at +20dB/decade with increasing frequency (until real op-amp limitations kick in).
- **Curve:** Triangular wave input → **square wave** output (derivative of a linear ramp is a constant). Square wave input → sharp spikes/pulses at each transition (derivative of a step is an impulse-like spike).
```
 V_in (triangular wave):        V_out (square wave):
    /\    /\                       __    __
   /  \  /  \           →      ___|  |__|  |___
  /    \/    \
```
- **Circuit:** `V_in → C → (−) input node; Resistor R connects (−) node to V_out; (+) input grounded.` (Same as inverting amp, but R_in replaced by C.)
- **MCQ patterns:** "Given triangular input, sketch/identify square wave output", "given square wave input, identify spike/pulse output", "identify differentiator vs integrator from circuit (C-then-R = differentiator)".
- **Traps:** Forgetting negative sign; confusing which component comes first (C at INPUT = differentiator; C in FEEDBACK = integrator — easy to mix up); differentiators are noise-sensitive/unstable at high frequency in practice (often need a small series resistor to tame this — sometimes tested conceptually).
- **Memory trick:** "Differentiator: C in, R in feedback → triangle wave IN → square wave OUT. Acts as a high-pass filter. (Opposite of integrator!)"
- **Application:** Edge/transition detectors, rate-of-change sensors (e.g., FM demodulation), wave-shaping circuits, PID controllers (the "D" term).

---

## 6. Comparator
- **Concept:** Op-amp used in OPEN-LOOP (no feedback) to compare two voltages — output slams to one of two saturation levels (+V_sat or −V_sat) depending on which input is larger. Essentially a 1-bit ADC.
- **Formulas:**
  - If `V+ > V−` → `V_out = +V_sat` (≈+V_CC).
  - If `V+ < V−` → `V_out = −V_sat` (≈−V_EE, or 0 in single-supply).
  - No linear region — output is always at one rail or the other (except during the near-instantaneous transition).
- **Curve:** Output is a two-level (binary) square-wave-like signal — switches abruptly between +V_sat and −V_sat exactly when input crosses the reference level, with NO hysteresis (switches back and forth right at threshold, vulnerable to noise-induced multiple crossings/"chatter").
```
 V_in (sine) crossing V_ref:      V_out (square, switches AT V_ref):
     /\   /\                        __    __
 ---/--\-/--\--- V_ref    →     ___|  |__|  |___
   /    X    \                  (chatter possible if V_in noisy near V_ref)
```
- **Circuit:** `V_in → (+) input; V_ref → (−) input; NO feedback resistor at all (open-loop).`
- **MCQ patterns:** "Given V_in and V_ref waveforms, sketch comparator output", "why does comparator output only have 2 levels?" (open-loop, saturates), "what problem occurs with noisy input near threshold?" (chatter/multiple triggering).
- **Traps:** Adding feedback resistor by mistake (that would make it an amplifier, not a comparator — comparator MUST be open-loop); forgetting real op-amps used as comparators are slow — dedicated comparator ICs are faster and preferred in practice.
- **Memory trick:** "Comparator = No feedback = output ALWAYS at a rail (+V_sat or −V_sat). Simplest 1-bit decision maker."
- **Application:** Zero-crossing detectors, level detectors, simple ADC building block, over/under-voltage alarms.

---

## 7. Schmitt Trigger
- **Concept:** A comparator WITH POSITIVE FEEDBACK, creating two different threshold levels (hysteresis) instead of one — solves the "chatter" problem of a plain comparator with noisy inputs.
- **Formulas:**
  - Upper Threshold Voltage: `V_UT = +V_sat·(R1/(R1+R2))` (for inverting Schmitt trigger with output feedback via R1/R2 divider).
  - Lower Threshold Voltage: `V_LT = −V_sat·(R1/(R1+R2))`.
  - Hysteresis width: `V_H = V_UT − V_LT`.
- **Curve:** Hysteresis loop (V_out vs V_in) — a rectangular loop shape; output switches HIGH only when input rises above V_UT, and switches LOW only when input falls below V_LT (different thresholds for rising vs falling).
```
  V_out
  +Vsat |________         ________
        |        \       /
        |         \     /
  -Vsat |          \___/
        +-----------------------  V_in
             V_LT      V_UT
     (switches high going up at V_UT, low going down at V_LT)
```
- **Circuit:** Comparator circuit PLUS a positive feedback resistor divider (R1, R2) from output back to the (+) input — this is what creates the two thresholds.
- **MCQ patterns:** "Find V_UT and V_LT given R1, R2, V_sat", "why is Schmitt trigger preferred over plain comparator for noisy signals?" (hysteresis prevents chatter), "identify positive feedback path in given circuit".
- **Traps:** Confusing this with plain comparator (KEY DIFFERENCE: Schmitt trigger uses POSITIVE feedback, comparator uses NONE); forgetting the two thresholds are different — a single-threshold assumption is a common wrong answer trap.
- **Memory trick:** "Schmitt Trigger = Comparator + Positive Feedback + Hysteresis (2 thresholds) = Noise-immune switching."
- **Application:** Noise-immune switching (converting noisy/slow-edge signals into clean digital transitions), oscillators (relaxation oscillator), debouncing.

---

## 8. Active Filters
- **Concept:** Filters built using op-amps + R/C (no inductors needed) — provide gain AND filtering in one stage, unlike passive RC filters which only attenuate.
- **Formulas:**
  - Cutoff frequency (1st order): `f_c = 1/(2πRC)`.
  - Roll-off rate: 1st order = ±20dB/decade; 2nd order = ±40dB/decade (per added RC stage).
  - Types: Low-Pass (passes below f_c), High-Pass (passes above f_c), Band-Pass (passes a band between f_L and f_H), Band-Stop/Notch (rejects a band).
- **Curve:** Bode magnitude plot — Low-pass: flat then falls off after f_c. High-pass: rises then flat after f_c. Band-pass: peak between two cutoff frequencies, falls off on both sides.
```
 Low-pass:            High-pass:            Band-pass:
 Gain                 Gain                  Gain
  ‾‾‾\                    /‾‾‾                 /\
      \___ f            _/                    /  \
       f_c               f_c                 f_L f_H
```
- **Circuit:** Low-pass active filter = non-inverting/inverting op-amp with RC low-pass network at input (or C in feedback for inverting-integrator-style). High-pass = swap R and C positions.
- **MCQ patterns:** "Find cutoff frequency given R, C", "identify filter type from Bode plot or circuit", "why use active filter instead of passive?" (provides gain, avoids loading effects due to low output impedance/high input impedance of op-amp).
- **Traps:** Forgetting active filters need external power (op-amp supply) — unlike passive RC filters; confusing roll-off order (each additional RC STAGE adds another 20dB/decade, not automatically 40dB with one stage).
- **Memory trick:** "Active filter = Passive RC filter + Op-amp buffer/gain — same f_c formula (1/2πRC), but now with gain and no loading effect."
- **Application:** Audio equalizers, anti-aliasing filters before ADC, signal conditioning, noise rejection.

---

## 9. Differential Amplifier
- **Concept:** Amplifies the DIFFERENCE between two input signals while ideally rejecting any signal common to both inputs (common-mode signal, e.g., noise picked up equally on both lines).
- **Formulas:**
  - `V_out = A_d(V1 − V2)` where `A_d` = differential gain.
  - For a basic 4-resistor differential amp with matched ratios: `A_d = R_f/R_in` (same ratio on both input branches).
- **Curve:** Output responds only to (V1 − V2); if V1=V2 (pure common-mode input), ideal output = 0.
- **Circuit:** Two inputs, each through matched input resistors to (+) and (−) op-amp terminals; feedback resistor from output to (−) input; matching resistor from (+) input to ground — resistor ratios must match on both sides for good common-mode rejection.
```
 V1 --Rin--(-) \
               } ---- Vout
 V2 --Rin--(+) /
       |
      R (to GND, matches Rf/Rin ratio)
    with Rf from Vout back to (-) node
```
- **MCQ patterns:** "Find V_out given V1, V2, and resistor values", "why must resistor ratios match exactly?" (mismatch → poor CMRR, common-mode signal leaks through), identify differential amp from 4-resistor + op-amp topology.
- **Traps:** Forgetting resistor mismatch directly degrades CMRR (a very common conceptual MCQ); assuming this circuit has same high input impedance as non-inverting amp (input impedance here is lower, set by R_in on each side).
- **Memory trick:** "Differential amp: amplifies the DIFFERENCE, ideally rejects what's COMMON to both inputs — resistor matching is everything."
- **Application:** Instrumentation amplifiers (biomedical signals like ECG), noise-rejecting sensor interfaces, audio balanced line receivers.

---

## 10. Differential Gain
- **Concept:** The gain applied specifically to the DIFFERENCE between the two input signals — the "wanted" gain of a differential/instrumentation amplifier.
- **Formulas:**
  - `A_d = V_out/(V1−V2)` (ideally, this is the ONLY gain that should exist).
  - For standard differential amp: `A_d = R_f/R_in` (with matched resistor ratios on both sides).
- **Curve:** Linear relationship — output scales directly with the difference signal (V1−V2); should be independent of the common-mode/average level of V1 and V2.
- **Circuit:** Same differential amplifier circuit as above — A_d is simply the differential-mode transfer function of that circuit.
- **MCQ patterns:** "Calculate differential gain given resistor values", "distinguish differential gain from common-mode gain in a given output expression".
- **Traps:** Confusing differential gain (wanted, usually LARGE) with common-mode gain (unwanted, should be near ZERO in an ideal circuit) — these are two separate numbers, often tested together with CMRR.
- **Memory trick:** "Differential gain = the amplifier doing its JOB (amplifying the real signal difference)."
- **Application:** Core specification for instrumentation amplifiers — determines how strongly the true signal-of-interest gets amplified.

---

## 11. CMRR (Common-Mode Rejection Ratio)
- **Concept:** A figure of merit measuring how well a differential amplifier rejects common-mode signals (noise/interference present equally on both inputs) relative to how well it amplifies the wanted differential signal. HIGHER CMRR = BETTER noise rejection.
- **Formulas:**
  - `CMRR = A_d/A_cm` (ratio of differential gain to common-mode gain).
  - In dB: `CMRR(dB) = 20log₁₀(A_d/A_cm)`.
  - Ideal op-amp: `A_cm = 0` → `CMRR = ∞` (never achieved in practice; real op-amps have CMRR typically 80–120 dB).
- **Curve:** CMRR vs frequency — typically high (good) at DC/low frequency, DEGRADES (decreases) as frequency increases — a commonly tested trend.
- **Circuit:** Same differential/instrumentation amplifier circuit — CMRR is a derived performance SPEC of that circuit, not a separate topology.
- **MCQ patterns:** "Calculate CMRR given A_d and A_cm", "convert CMRR to/from dB", "why does higher resistor mismatch reduce CMRR?" (introduces non-zero A_cm), "does CMRR improve or worsen at high frequency?" (worsens/decreases).
- **Traps:** Forgetting CMRR is a RATIO (higher is better) — don't confuse with A_cm itself (where LOWER is better); mixing up the dB conversion (use 20log, not 10log, since it's a ratio of gains/voltages, not power).
- **Memory trick:** "CMRR = Differential gain ÷ Common-mode gain. High CMRR = the amp ignores noise well. Think of CMRR as a 'rejection score' — bigger is always better."
- **Application:** Key spec for instrumentation amplifiers, biomedical amplifiers (ECG/EEG), any application needing to extract a small signal from a noisy/common-mode-heavy environment.

---

## 🔑 Comparison Tables

### Inverting vs Non-Inverting Amplifier
| Aspect | Inverting | Non-Inverting |
|---|---|---|
| Gain formula | −R_f/R_in | 1 + R_f/R_g |
| Phase shift | 180° (inverted) | 0° (in phase) |
| Input impedance | ≈R_in (moderate/low) | Very high |
| Minimum gain magnitude | Can be < 1 (attenuator possible) | Always ≥ 1 |
| Special case | — | Voltage follower (R_f=0 → A_v=1) |

### Comparator vs Schmitt Trigger
| Aspect | Comparator | Schmitt Trigger |
|---|---|---|
| Feedback | None (open-loop) | Positive feedback |
| Thresholds | Single threshold | Two thresholds (V_UT, V_LT) — hysteresis |
| Noise immunity | Poor (chatter near threshold) | Good (hysteresis prevents chatter) |
| Output | Switches at exact V_ref crossing | Switches at different points going up vs down |

### Integrator vs Differentiator
| Aspect | Integrator | Differentiator |
|---|---|---|
| Component placement | R at input, C in feedback | C at input, R in feedback |
| Math function | Output ∝ ∫V_in dt | Output ∝ dV_in/dt |
| Filter behavior | Low-pass | High-pass |
| Square wave input → output | Triangular wave | Spikes/pulses |
| Triangular wave input → output | Parabolic-ish curve | Square wave |

### Differential Gain vs CMRR
| Aspect | Differential Gain (A_d) | CMRR |
|---|---|---|
| What it measures | Gain applied to (V1−V2), the wanted signal | Ratio of A_d to common-mode gain A_cm |
| Ideal value | As designed (finite, deliberate) | Infinite (∞) |
| Formula | R_f/R_in (matched resistor differential amp) | A_d/A_cm, or 20log(A_d/A_cm) in dB |
| Larger = better? | Depends on design need | YES — always want CMRR as large as possible |
| What degrades it | Circuit design choice (not a "flaw") | Resistor mismatch, non-ideal op-amp, high frequency |

---

**End of Part 3 — full Analog Electronics OA cheat sheet complete (all 3 parts).**
