# Circuit Analysis — OA Cheat Sheet (Part 1/2)
*KCL → Norton | Formula-first, exam-speed revision*

---

## 1. KCL (Kirchhoff's Current Law)
- **Formula:** ΣI_in = ΣI_out at any node.
- **Concept:** Charge can't accumulate at a node — current in = current out.
- **Patterns:** "Find unknown branch current", node current tables, current divider setups.
- **Method:** Assign directions, write ΣI = 0, solve algebraically (don't overthink sign — flip if answer negative).
- **Mistakes:** Missing a branch at the node; inconsistent sign convention across steps.
- **Shortcut:** Current divider: `I₁ = I_total × R₂/(R₁+R₂)` (2 parallel R's).

---

## 2. KVL (Kirchhoff's Voltage Law)
- **Formula:** ΣV around any closed loop = 0.
- **Concept:** Energy conservation — sum of drops = sum of rises in a loop.
- **Patterns:** Single-loop circuits, series R with sources, "find voltage across element X".
- **Method:** Pick loop direction, drop = +IR (if traversing + to −), sum to zero, solve.
- **Mistakes:** Sign error entering a source (check + terminal first); forgetting a component in the loop.
- **Shortcut:** Voltage divider: `V₁ = V_total × R₁/(R₁+R₂)` (series).

---

## 3. Node (Nodal) Analysis
- **Formula:** At each non-reference node: Σ(V_node − V_adj)/R = Σ I_injected.
- **Concept:** Solve for node voltages w.r.t. ground; fewer equations than mesh if many parallel branches.
- **Patterns:** "Find V at node A/B", circuits with current sources, multiple parallel resistors.
- **Method:** Pick ground (usually most-connected node) → write KCL per node in terms of V → solve linear system (2×2 by hand, or Cramer's rule).
- **Mistakes:** Forgetting supernode when a voltage source floats between two non-ref nodes; wrong current direction assumption (doesn't matter if consistent).
- **Shortcut:** Supernode = merge nodes across floating V-source; add constraint equation V₁ − V₂ = V_source.

---

## 4. Mesh (Loop) Analysis
- **Formula:** Per mesh: Σ(IR drops around loop) = Σ V_sources.
- **Concept:** Solve for loop currents; best when circuit is planar with many series elements/voltage sources.
- **Patterns:** "Find mesh current I₁/I₂", multi-loop resistor networks, shared-branch problems.
- **Method:** Define clockwise mesh currents → KVL per mesh (shared resistor gets difference of two mesh currents) → solve system.
- **Mistakes:** Sign error on shared resistor term; missing supermesh when a current source is shared between two meshes.
- **Shortcut:** Supermesh = combine meshes sharing a current source; add constraint I₁ − I₂ = I_source.

---

## 5. Superposition
- **Formula:** Response = Σ (response due to each independent source acting alone).
- **Concept:** Linear circuits only; turn off all other independent sources one at a time.
- **Patterns:** "Find current/voltage due to multiple sources", verify linearity.
- **Method:** Kill other sources (V-source → short, I-source → open) → solve simple circuit → repeat for each source → add results algebraically (respect direction/sign).
- **Mistakes:** Forgetting dependent sources must STAY (never killed); open vs short confusion.
- **Shortcut:** Only kill *independent* sources — dependent sources always remain active.

---

## 6. Thevenin's Theorem
- **Formula:** V_th = open-circuit voltage at terminals; R_th = equivalent resistance seen from terminals (sources killed).
- **Concept:** Any linear 2-terminal network → single V_th in series with R_th.
- **Patterns:** "Find Thevenin equivalent", "find current through load R_L", load-varying problems.
- **Method:** Remove load → find V_oc (=V_th) → kill sources, find R_eq (=R_th) → reattach load: `I_L = V_th/(R_th+R_L)`.
- **Mistakes:** Computing R_th without killing sources; forgetting dependent-source case needs test source method (R_th = V_test/I_test).
- **Shortcut:** If dependent sources present → apply 1V test source at terminals, R_th = 1/I_test (kill only independent sources).

---

## 7. Norton's Theorem
- **Formula:** I_N = short-circuit current at terminals; R_N = R_th (same equivalent resistance).
- **Concept:** Dual of Thevenin — current source I_N parallel with R_N.
- **Patterns:** "Find Norton equivalent", source transformation problems.
- **Method:** Short the terminals → find I_sc (=I_N) → R_N = R_th (same method as above) → `V_th = I_N × R_N` links both models.
- **Mistakes:** Confusing open-circuit (Thevenin) vs short-circuit (Norton) conditions.
- **Shortcut:** Source transform: `V_th/R_th = I_N`, and `I_N × R_N = V_th` — convert instantly between the two.
# Circuit Analysis — OA Cheat Sheet (Part 2/2)
*Reciprocity → Two-Port Networks | Formula-first, exam-speed revision*

---

## 8. Reciprocity Theorem
- **Formula:** For single-source linear networks: `V/I` ratio unchanged if source and response (ammeter) positions are swapped.
- **Concept:** Applies only to circuits with ONE independent source and only R, L, C (no dependent sources).
- **Patterns:** "If source and meter are interchanged, will reading change?" (usually a yes/no or verify-value question).
- **Method:** Check network is reciprocal (passive, linear, no dependent sources) → ratio of excitation to response stays same after swap.
- **Mistakes:** Applying it to circuits with dependent sources or multiple sources — theorem fails there.
- **Shortcut:** Reciprocal ⇒ transfer impedance Z₁₂ = Z₂₁ (symmetric Z or Y matrix).

---

## 9. Maximum Power Transfer
- **Formula:** Max power to load when `R_L = R_th` (DC) or `Z_L = Z_th*` (conjugate match, AC).
- **Concept:** Load draws max power when matched to source's internal (Thevenin) resistance/impedance.
- **Patterns:** "Find R_L for max power", "find max power delivered", AC impedance matching.
- **Method:** Find R_th/Z_th from source side → set R_L = R_th (or Z_L = conjugate of Z_th) → `P_max = V_th²/(4R_th)`.
- **Mistakes:** Using R_L = R_th for AC (should be complex conjugate match, not just magnitude); forgetting factor of 4 in P_max formula.
- **Shortcut:** `P_max = V_th² / (4·R_th)` — memorize directly, avoid recomputation.

---

## 10. Phasors
- **Formula:** `v(t) = V_m cos(ωt+φ) ⇔ V = V_m∠φ` (phasor domain).
- **Concept:** Converts sinusoidal steady-state signals into complex numbers — turns calculus into algebra.
- **Patterns:** Convert time-domain ↔ phasor, add/subtract sinusoids, find phase difference.
- **Method:** Write magnitude and phase directly from cosine reference → use complex arithmetic (rectangular for +/−, polar for ×/÷) → convert back if asked for time-domain.
- **Mistakes:** Mixing sine and cosine reference (convert sin→cos: `sin(x) = cos(x−90°)`); forgetting ω is same for all elements in steady state (don't carry it in phasor).
- **Shortcut:** `sin(ωt) = cos(ωt − 90°)`; polar mult/div, rectangular add/sub.

---

## 11. AC Circuit Analysis
- **Formula:** Ohm's law extends: `V = IZ`, apply KCL/KVL directly with complex Z.
- **Concept:** Same DC techniques (node/mesh/Thevenin) apply — just use complex impedance instead of R.
- **Patterns:** "Find current/voltage phasor", AC node/mesh analysis, power in AC circuits.
- **Method:** Convert all sources/elements to phasor + impedance form → solve like DC circuit with complex numbers → convert back to time domain if needed.
- **Mistakes:** Forgetting Z_L and Z_C are frequency-dependent (recompute if ω changes); real vs complex power confusion.
- **Shortcut:** Avg power `P = ½ V_m I_m cos(θᵥ−θᵢ) = V_rms I_rms cos(θ)`.

---

## 12. RL, RC, RLC Circuits (Transient + Steady State)
- **Formula:**
  - Impedances: `Z_R = R`, `Z_L = jωL`, `Z_C = 1/(jωC)`.
  - Time constant: `τ = L/R` (RL), `τ = RC` (RC).
  - Natural response: `x(t) = x(∞) + [x(0)−x(∞)]e^(−t/τ)`.
- **Concept:** First-order (RL/RC) → exponential charge/discharge; second-order (RLC) → depends on damping.
- **Patterns:** "Find v(t)/i(t) after switch closes", "find time constant", step response.
- **Method:** Find initial value x(0⁻)=x(0⁺) [inductor current / cap voltage continuous] → find final value x(∞) (steady state, L→short, C→open) → find τ → plug into formula.
- **Mistakes:** Using wrong initial condition (voltage across C and current through L don't jump instantly); mixing up L→short/C→open at t=∞ vs t=0.
- **Shortcut:** At t=0⁺: inductor = open (if no initial current) / behaves as current source; capacitor = short (if uncharged). At t=∞: inductor = short, capacitor = open.

---

## 13. Resonance (Series & Parallel RLC)
- **Formula:** `ω₀ = 1/√(LC)`, `f₀ = 1/(2π√(LC))`.
- **Concept:** At resonance, `X_L = X_C`, so net reactance = 0 → circuit is purely resistive.
- **Patterns:** "Find resonant frequency", "find bandwidth/Q factor", "find impedance at resonance".
- **Method:** Set `ωL = 1/(ωC)` → solve for ω₀ → at resonance Z = R (series) or Z = R (parallel, max impedance case) → compute Q, BW from there.
- **Mistakes:** Confusing series resonance (Z minimum = R) with parallel resonance (Z maximum = R); wrong Q formula for series vs parallel.
- **Shortcut:** `Q = ω₀L/R = 1/(ω₀RC)` (series); `BW = ω₀/Q = R/L` (series, rad/s).

---

## 14. Laplace Transform (Circuit Application)
- **Formula:** `V=IZ(s)` with `Z_R=R`, `Z_L=sL`, `Z_C=1/(sC)`; include initial-condition sources if nonzero (`sL·i(0)` for inductor, `v(0)/s` for capacitor).
- **Concept:** Converts differential equations into algebraic s-domain equations; handles transients + initial conditions cleanly.
- **Patterns:** "Find transfer function H(s)", "find v(t) using Laplace", pole-zero/stability questions.
- **Method:** Transform circuit to s-domain (impedances + IC sources) → solve via node/mesh/Thevenin like a resistive circuit → inverse Laplace (partial fractions) back to t-domain if required.
- **Mistakes:** Forgetting initial-condition source terms; sign error in capacitor IC term (`v(0)/s` is a voltage SOURCE in series, oriented per v(0) polarity).
- **Shortcut:** Final Value Theorem: `x(∞) = lim_{s→0} sX(s)`; Initial Value Theorem: `x(0⁺) = lim_{s→∞} sX(s)`.

---

## 15. Two-Port Networks
- **Formula (Z-parameters):** `V₁ = Z₁₁I₁ + Z₁₂I₂`, `V₂ = Z₂₁I₁ + Z₂₂I₂`.
- **Concept:** Characterizes a 2-port black box via parameter sets (Z, Y, h, ABCD) relating port voltages/currents.
- **Patterns:** "Find Z/Y/h parameters", "identify reciprocal/symmetric network", convert between parameter sets.
- **Method:** For Z-params: `Z₁₁ = V₁/I₁|I₂=0` (port 2 open), `Z₁₂=V₁/I₂|I₁=0`, etc. — apply open/short conditions per parameter definition, solve directly.
- **Mistakes:** Using short-circuit condition for Z-params (should be OPEN) or open-circuit for Y-params (should be SHORT); mixing ABCD sign convention (I₂ often defined flowing OUT of port 2).
- **Shortcut:** Reciprocal network ⇒ `Z₁₂=Z₂₁`, `Y₁₂=Y₂₁`, `AD−BC=1` (ABCD). Symmetric ⇒ additionally `Z₁₁=Z₂₂` (or `A=D`).

---

## 🔑 Compact Quick-Reference Table

| Topic | Key Formula |
|---|---|
| KCL | ΣI_in = ΣI_out |
| KVL | ΣV_loop = 0 |
| Voltage divider | V₁ = V·R₁/(R₁+R₂) |
| Current divider | I₁ = I·R₂/(R₁+R₂) |
| Thevenin | I_L = V_th/(R_th+R_L) |
| Norton↔Thevenin | V_th = I_N·R_N |
| Max power | P_max = V_th²/(4R_th); R_L=R_th (DC), Z_L=Z_th* (AC) |
| Phasor | v=V_m cos(ωt+φ) ⇔ V_m∠φ |
| Impedances | Z_R=R, Z_L=jωL, Z_C=1/(jωC) |
| Avg AC power | P = V_rms·I_rms·cos(θ) |
| RL/RC transient | x(t)=x(∞)+[x(0)-x(∞)]e^(−t/τ); τ=L/R or RC |
| Resonance | ω₀=1/√(LC); Q=ω₀L/R (series) |
| Laplace impedances | Z_L=sL, Z_C=1/(sC) |
| FVT / IVT | x(∞)=lim_{s→0}sX(s); x(0⁺)=lim_{s→∞}sX(s) |
| Two-port (Z) | Z₁₁=V₁/I₁\|₂ₒₚₑₙ, reciprocal: Z₁₂=Z₂₁ |

---

## 🎯 Sample OA Question Types (map to method)
- **Find equivalent resistance** → series/parallel reduction, or Thevenin R_th (kill sources).
- **Solve node voltages** → Node analysis (§3), watch for supernodes.
- **Find mesh currents** → Mesh analysis (§4), watch for supermeshes.
- **Compute power** → P=VI (DC) or P=V_rms·I_rms·cosθ (AC); use P_max=V_th²/4R_th for matching problems.
- **Find resonance frequency** → ω₀=1/√(LC), f₀=ω₀/2π.
- **Use Thevenin/Norton equivalent** → reduce circuit to V_th/R_th or I_N/R_N, reattach load.
- **Calculate impedance/reactance** → Z_L=jωL, Z_C=1/(jωC); combine like resistors (series add, parallel reciprocal).
- **Identify transfer function / two-port params** → H(s)=Output(s)/Input(s) via Laplace; Z/Y/h params via open/short port conditions.

---
**End of Part 2 — full cheat sheet complete.**
---

**End of Part 1** — reply to continue with Part 2 (Reciprocity → Two-Port Networks + full quick-reference table).
