# Static Timing Analysis (STA) — Complete Study Notes
## PART 1: Fundamentals → Setup/Hold → Arrival/Required Time → Slack → Critical Path → Violations

> **Scope of Part 1:** Timing Analysis Fundamentals, STA overview, Propagation Delay, Contamination Delay, Setup Time, Hold Time, Arrival Time, Required Time, Slack, Critical Path, Maximum Clock Frequency, Timing Violations, and an introduction to Timing Report terms.
>
> **Scope of Part 2 (next):** Clock Skew, Clock Uncertainty/Jitter, Register-to-Register / Input-to-Register / Register-to-Output timing in depth, Multi-cycle Paths, False Paths, Timing Closure, Physical Design impact, Setup & Hold Fixes, Interconnect vs Logic Delay, Path-based timing, full Report interpretation, and Interview Q&A.

---

## 1. Timing Analysis Fundamentals

### 1.1 What is Timing Analysis?

Every digital circuit is built from **combinational logic** (gates, MUXes, adders) and **sequential elements** (flip-flops, latches) that store state. Data moves through the circuit in discrete "hops" — from one flip-flop, through combinational logic, into the next flip-flop — once every clock cycle.

**Timing analysis** is the process of verifying that:
1. Data launched from a flip-flop on one clock edge **arrives early enough** to be correctly captured by the next flip-flop on the following active clock edge (this is the **setup** requirement), and
2. Data does **not arrive too early** and corrupt the value that is currently being captured (this is the **hold** requirement).

If either of these is violated, the circuit may compute wrong logical values, or work correctly only sometimes (a "flaky" chip) — an extremely dangerous class of bug because it can be data-dependent, voltage-dependent, or temperature-dependent, and may not show up in every test run.

### 1.2 Why is Timing Analysis Needed?

- A physical circuit is not instantaneous — every wire and gate has **delay** caused by resistance, capacitance, and switching characteristics of transistors.
- As clock frequency increases (performance target), the "time budget" available for data to travel from one register to the next **shrinks**.
- Process, Voltage, and Temperature (**PVT**) variations, manufacturing variation, and design margins all eat into this budget.
- Without formally verifying timing, a chip could be functionally correct in RTL simulation (which is a **zero-delay** or **unit-delay** abstraction) but **fail in silicon** because gate/wire delays were not accounted for.

### 1.3 Two Approaches to Timing Verification

| Approach | Description | Pros | Cons |
|---|---|---|---|
| **Dynamic Timing Simulation (DTS)** | Apply test vectors, simulate with real delays (SPICE/gate-level sim), check waveforms | Very accurate | Coverage-limited; cannot exhaustively test every path; extremely slow for full-chip |
| **Static Timing Analysis (STA)** | Analyze *all* paths in the design without applying any input vectors, using delay models | Exhaustive (100% path coverage), fast, scalable to full chip | Doesn't know "functional" false paths automatically (needs exceptions); pessimistic in some scenarios |

STA has become the **industry-standard sign-off methodology** because it is exhaustive and fast — it checks every single timing path in the design (millions of paths in modern SoCs) without needing test vectors.

### 1.4 Key Idea Behind STA

STA breaks the whole chip into a **timing graph**:
- **Nodes** = pins of gates / ports
- **Edges** = delay arcs (through a gate, or through a wire/net)

STA then finds:
- The **arrival time** of a signal at every node (worst-case latest, and best-case earliest)
- The **required time** at every node (the deadline based on the next flip-flop's setup/hold requirement)
- The **slack** = required time − arrival time (or vice versa) at every endpoint

If slack is negative anywhere → **timing violation** → chip may not work at the targeted frequency.

---

## 2. Static Timing Analysis (STA) — Deep Dive

### 2.1 What STA Actually Computes

For **every possible path** from a **startpoint** (a flip-flop clock pin / input port) to an **endpoint** (a flip-flop data pin / output port), STA computes:

1. **Data Arrival Time (AT)** – when the data signal actually arrives at the endpoint, in the worst case (max delay analysis) and best case (min delay analysis)
2. **Data Required Time (RT)** – the time by which data *must* arrive to be captured correctly
3. **Slack = RT − AT**

STA does this using **static delay models** (lookup tables / equations, e.g., from Liberty `.lib` files) for every cell and interconnect — it does **not** simulate actual logic values (0/1 patterns). This is why it's called "static" — the analysis is independent of the functional vectors being applied.

### 2.2 Corners: Why "Static" Doesn't Mean "One Number"

Delay of a gate depends on:
- **Process (P):** manufacturing variation (fast/slow silicon)
- **Voltage (V):** Vdd variation
- **Temperature (T):** operating temperature

STA is run across multiple **PVT corners**:
- **Setup analysis** is checked at the **slowest** corner (worst-case max delay): e.g., **SS (Slow-Slow process), low voltage, high temperature** — called **SSLowVoltHighTemp**, generally the "**slow corner**."
- **Hold analysis** is checked at the **fastest** corner (worst-case min delay): e.g., **FF (Fast-Fast process), high voltage, low temperature** — the "**fast corner**."

> **Interview point:** *Why is setup checked at the slow corner and hold at the fast corner?*
> Setup fails when data is **too slow** to arrive → check the corner that makes logic **slowest** (max delay) → worst case is the **slow corner**.
> Hold fails when data is **too fast** to arrive → check the corner that makes logic **fastest** (min delay) → worst case is the **fast corner**.

### 2.3 STA Flow Overview (where it sits in ASIC design)

```
RTL Design → Synthesis (gate-level netlist) → Floorplan → Placement 
→ CTS (Clock Tree Synthesis) → Routing → Post-Route STA (Sign-off)
                                        ↑
                    STA run at EVERY stage (pre-layout & post-layout)
```

- **Pre-layout STA:** Uses estimated wire-load models (no real placement/routing yet) — approximate.
- **Post-layout STA (sign-off):** Uses actual extracted parasitics (RC from real wires after routing) — this is the **golden, accurate** STA used for tape-out sign-off.

### 2.4 Inputs Required for STA

| Input | Purpose |
|---|---|
| Gate-level netlist | Structural connectivity of the design |
| Standard Cell Liberty (`.lib`) files | Delay/timing models of each cell, at each PVT corner |
| SDC (Synopsys Design Constraints) | Clock definitions, I/O delays, false paths, multi-cycle paths, exceptions |
| Parasitics (SPEF/DSPEF) | Extracted RC values of interconnect (post-route) |
| SDF (optional) | Standard Delay Format — annotated delays for simulation cross-check |

### 2.5 Tools

Industry-standard STA tools: **PrimeTime (Synopsys)**, **Tempus (Cadence)**, **OpenSTA (open-source)**. All follow the same conceptual model described in these notes.

---

## 3. Propagation Delay

### 3.1 Definition

**Propagation delay (Tpd / Tpcq for flip-flops)** is the time taken for a signal change at the **input** of a logic element (gate or flip-flop) to produce the corresponding change at its **output**, measured under the **worst-case (maximum)** delay condition — i.e., **assuming everything is as slow as it can be**.

For a flip-flop specifically, this is called:
**Clock-to-Q delay (Tcq)** or **Tpcq (max)** — the max time from the active clock edge until the output Q changes.

### 3.2 Why It Matters

Propagation delay represents the **worst-case (slowest)** timing behavior of a cell/path. It is used in **setup (max delay) analysis**, because setup analysis asks: *"What is the LATEST that data could possibly arrive?"*

### 3.3 Where Propagation Delay Comes From

1. **Cell (gate) delay** — intrinsic switching delay of a transistor-level gate, dependent on input slew and output load (capacitance)
2. **Interconnect (wire/net) delay** — RC delay of the metal wire connecting cells (covered in depth in Part 2)

Total path delay = Σ (cell delays) + Σ (net/wire delays) along the path.

### 3.4 Standard Measurement Convention

Propagation delay is typically measured from the **50% point** of the input transition to the **50% point** of the output transition (standard industry convention for cell characterization in Liberty files).

```
Input:   ────╲___________
                50%
Output:  ─────────╲_______
                    50%
              |<-- Tpd -->|
```

---

## 4. Contamination Delay

### 4.1 Definition

**Contamination delay (Tcd / Tccq for flip-flops)** is the **minimum** time for a signal change at the input to start causing a change at the output — i.e., the **earliest / fastest** the output could possibly start changing.

For a flip-flop: **Contamination Clock-to-Q delay (Tccq / Tcq-min)** — the **minimum** time after the clock edge before Q could possibly start changing (even a glitch).

### 4.2 Why It Matters

Contamination delay represents the **best-case (fastest)** timing behavior. It is used in **hold (min delay) analysis**, because hold analysis asks: *"What is the EARLIEST that data could possibly arrive (and potentially corrupt the previous value being latched)?"*

### 4.3 Propagation vs Contamination Delay — Side-by-Side

| Aspect | Propagation Delay (Tpd) | Contamination Delay (Tcd) |
|---|---|---|
| Meaning | Max/worst-case delay | Min/best-case delay |
| Used for | Setup analysis | Hold analysis |
| Corner used | Slow corner (SS, low V, high T) | Fast corner (FF, high V, low T) |
| Question answered | "Latest the output can change?" | "Earliest the output can start changing?" |
| Relation | Tcd ≤ Tpd always | Tcd ≤ Tpd always |

> **Golden Rule:** `Contamination delay ≤ Propagation delay` for any real gate or path — the output cannot start changing *before* the earliest possible time (Tcd) and is guaranteed to have fully changed by the *latest* possible time (Tpd).

---

## 5. Setup Time

### 5.1 Definition

**Setup time (Tsu)** of a flip-flop is the minimum amount of time **before** the active clock edge during which the data (D) input must be **stable** (not changing) for the flip-flop to reliably and correctly capture that data.

```
        Tsu
   |<-------->|
   ____________         
D  ____________\_____________ (must be stable here)
                             ↑
                          CLK edge
```

### 5.2 Why It Exists (Physical Intuition)

Internally, a flip-flop is built from cascaded latches/transmission gates. The data has to propagate through internal nodes and settle **before** the clock edge triggers the capture — if data is still transitioning when the clock edge arrives, the flip-flop may enter a **metastable** state (output hangs between 0 and 1, taking unpredictable/long time to resolve).

### 5.3 Setup Time and Why It Governs Maximum Frequency

Setup time defines how much time before the clock edge the data must be ready. Combined with combinational path delay, it determines the **maximum operating frequency** of the design (explained fully in Section 8).

### 5.4 Where It Appears in Reports

In a PrimeTime/Tempus timing report, setup checks appear as:
```
Startpoint: reg_A (rising edge-triggered flip-flop clocked by CLK)
Endpoint:   reg_B (rising edge-triggered flip-flop clocked by CLK)
Path Group: CLK
Path Type:  max   <-- "max" = setup check
```

---

## 6. Hold Time

### 6.1 Definition

**Hold time (Th)** of a flip-flop is the minimum amount of time **after** the active clock edge during which the data (D) input must remain **stable** for the flip-flop to reliably capture it.

```
                Th
           |<-------->|
D  ________/______________________ (must remain stable here)
           ↑
        CLK edge
```

### 6.2 Why It Exists

Even after the clock edge triggers capture, the internal latch/sense-amp of the flip-flop needs a small additional window of stable data to fully and correctly latch the value internally, due to internal signal propagation and charge-sharing effects.

### 6.3 Hold Time and Why It Is Independent of Clock Period

Unlike setup, **hold time checks do NOT depend on clock frequency** — hold is a check between two **immediately adjacent edges** (same edge, launch and capture), so slowing down the clock **does not fix a hold violation**. This is a critical interview concept (detailed derivation in Section 8).

### 6.4 Where It Appears in Reports

```
Path Type: min   <-- "min" = hold check
```

### 6.5 Setup vs Hold — Side-by-Side Comparison

| Aspect | Setup Time | Hold Time |
|---|---|---|
| Window checked | Before clock edge | After clock edge |
| Violated by | Data arriving **too late** (slow paths) | Data arriving **too early** (fast paths) |
| Delay type used | Max delay (propagation) | Min delay (contamination) |
| Corner | Slow (SS, low V, high T) | Fast (FF, high V, low T) |
| Depends on clock period? | **Yes** | **No** |
| Fixed by | Reducing logic delay, increasing clock period, pipelining | Adding delay (buffers) in the fast path |
| Also called | Max-delay / long-path check | Min-delay / short-path check |
| Path type in report | `max` | `min` |

---

## 7. Arrival Time and Required Time

This is the mathematical heart of STA. Every setup/hold check reduces to comparing **Arrival Time** vs **Required Time**.

### 7.1 Data Arrival Time (AT)

**Definition:** The actual time at which a data signal arrives at a given pin, measured from a reference clock edge (usually time = 0 at the launching clock edge).

**For a register-to-register path:**

```
AT (at capture flop D pin) = T_launch_clock_edge 
                            + Tclk-to-Q (of launching flop)
                            + Tcombinational (logic + interconnect delay)
```

Where:
- `T_launch_clock_edge` = the time at which the clock edge arrives at the **launching** flip-flop (0, if launch clock is ideal and starts the reference)
- `Tclk-to-Q` = propagation delay of the launching flip-flop (max, for setup)
- `Tcombinational` = sum of all gate delays + net delays along the data path

**Simple worked example:**
- Clock launches at t = 0 ns
- Launch flop Tcq (max) = 0.2 ns
- Combinational logic delay = 1.5 ns
- **Data Arrival Time = 0 + 0.2 + 1.5 = 1.7 ns**

### 7.2 Data Required Time (RT)

**Definition:** The latest time (for setup) by which data must arrive at the capturing flip-flop's D pin so it can be safely and correctly captured on the next active clock edge.

**For setup (max) analysis:**

```
RT (setup) = T_capture_clock_edge + Tclock_path_to_capture_flop 
           − Tsetup (of capturing flop)
           − (any margin: uncertainty, skew — see Part 2)
```

In the simplest single-clock-domain, ideal-clock case:

```
RT (setup) = Tclk_period − Tsu
```

**Simple worked example:**
- Clock period = 2 ns
- Setup time of capture flop = 0.1 ns
- **Required Time = 2 − 0.1 = 1.9 ns**

### 7.3 Putting It Together — The Fundamental Setup Equation

```
Tclk_period ≥ Tcq(max) + Tcombinational(max) + Tsetup
```

This single equation is **the most important equation in all of STA** — nearly every setup-related interview question traces back to this.

### 7.4 The Fundamental Hold Equation

```
Tccq(min) + Tcombinational(min) ≥ Thold
```

Notice: **no clock period term appears** — this is why hold violations are independent of frequency.

### 7.5 Arrival/Required Time — Full Numerical Example (Setup)

**Given:**
- Clock period T = 3 ns
- Launch flop Tcq(max) = 0.25 ns
- Combinational path delay (max) = 2.0 ns
- Capture flop Tsetup = 0.15 ns
- Assume ideal clock (no skew/jitter for now — covered in Part 2)

**Step 1 — Data Arrival Time:**
```
AT = Tcq(max) + Tcombo(max) = 0.25 + 2.0 = 2.25 ns
```

**Step 2 — Data Required Time:**
```
RT = T − Tsetup = 3 − 0.15 = 2.85 ns
```

**Step 3 — Slack:**
```
Slack = RT − AT = 2.85 − 2.25 = +0.60 ns   → Setup MET (positive slack)
```

### 7.6 Same Example — Hold Check

**Given (additionally):**
- Launch flop Tccq(min) = 0.10 ns
- Combinational path delay (min) = 0.30 ns
- Capture flop Thold = 0.12 ns

**Step 1 — Data Arrival Time (min):**
```
AT(min) = Tccq(min) + Tcombo(min) = 0.10 + 0.30 = 0.40 ns
```

**Step 2 — Data Required Time (hold):**
```
RT(hold) = Thold = 0.12 ns   (measured from the SAME capture clock edge, at t=0 relative offset)
```

**Step 3 — Hold Slack:**
```
Slack = AT(min) − RT(hold) = 0.40 − 0.12 = +0.28 ns → Hold MET
```

> **Note the reversal:** For setup, Slack = RT − AT. For hold, Slack = AT − RT. This is because setup cares about an **upper bound** on arrival (data must not be later than RT), while hold cares about a **lower bound** on arrival (data must not be earlier than RT). Full explanation in Section 9.

---

## 8. Slack

### 8.1 Definition

**Slack** is the timing margin — how much "spare time" exists between when data is required and when it actually arrives.

```
Setup Slack = Data Required Time (max) − Data Arrival Time (max)
Hold  Slack = Data Arrival Time (min)  − Data Required Time (min)
```

### 8.2 Positive vs Negative Slack — Interpretation

| Slack Value | Meaning | Design Status |
|---|---|---|
| **Positive slack** | Data arrives **earlier** than required (setup) / **later** than the minimum bound (hold) | Timing **met** — path has margin to spare |
| **Zero slack** | Data arrives exactly at the deadline | Timing **just met** — no margin (risky, sensitive to variation) |
| **Negative slack** | Data arrives **later** than required (setup) / **earlier** than allowed (hold) | **Timing violation** — must be fixed before sign-off |

### 8.3 Why Slack Matters for Signoff

- **Worst Negative Slack (WNS):** The single worst (most negative) slack value across the entire chip for a given check (setup or hold). Chip **cannot tape out** with WNS < 0 unless waived/fixed.
- **Total Negative Slack (TNS):** Sum of all negative slacks across all failing endpoints — indicates how *widespread* the violation is (many small violations vs one big one).
- Sign-off criteria typically require: **WNS ≥ 0** and **TNS = 0** (or within an agreed margin) for both setup and hold, across all corners.

### 8.4 Slack Numerical Example (extending 7.5)

Suppose combinational delay increases to 3.2 ns due to a routing congestion issue post-layout (was estimated at 2.0 ns pre-layout):

```
AT = Tcq(max) + Tcombo(max) = 0.25 + 3.2 = 3.45 ns
RT = T − Tsetup = 3 − 0.15 = 2.85 ns

Slack = RT − AT = 2.85 − 3.45 = −0.60 ns   → SETUP VIOLATION
```

This −0.60 ns must be recovered through timing fixes (buffer insertion, gate sizing, logic restructuring, or — if allowed — relaxing the clock period). Detailed fixing strategies are in **Part 2**.

---

## 9. Why Setup and Hold Use Opposite Slack Formulas (Deep Intuition)

This is a common interview trip-up point, so let's build the intuition carefully using a timing diagram description.

Think of the capture flip-flop's valid data window relative to the clock edge:

```
     ← Tsetup →|← Thold →
   ─────────────┼─────────────
                CLK edge (t = 0 reference)
   [data must be stable in this ENTIRE window]
```

- For **setup**, we are checking: *"Did data arrive before the window started (i.e., before t = −Tsetup relative to the capture edge)?"* → so we compare **Required (deadline) ≥ Arrival**. If arrival creeps later and later, eventually it violates → **Slack = Required − Arrival**, and slack goes negative when data is *late*.

- For **hold**, we are checking: *"Did data stay stable past the end of the window (i.e., past t = +Thold)?"* Here, the concern is data arriving *too early* — specifically, **new** data from the *next* launch overwriting the *current* captured value before the hold window closes. So we compare **Arrival ≥ Required (minimum bound)** → **Slack = Arrival − Required**, and slack goes negative when data is *early*.

This is the fundamental reason setup and hold slack formulas are "flipped" relative to each other — they protect against dangers on opposite ends of the clock edge.

---

## 10. Critical Path

### 10.1 Definition

The **critical path** is the specific timing path in the entire design that has the **worst (least, potentially most negative) slack** among all paths, for a given analysis (setup or hold).

- It represents the **bottleneck** of the design — the path that limits how fast the chip can run (for setup) or that is at greatest risk of a race condition (for hold).

### 10.2 Why It's Called "Critical"

Because it is the path STA/designers must focus on first — fixing any other path won't help overall timing if the critical path still fails; the critical path **determines the achievable clock frequency** (see Section 11).

### 10.3 Identifying Critical Paths in Reports

Tools report the **top N worst paths** sorted by slack, typically via commands like `report_timing` (PrimeTime) — the very first (worst) path listed is the critical path.

```
Startpoint: U1/reg_A/CK (rising edge)
Endpoint:   U5/reg_D/D  (rising edge)
Path Group: core_clk
Path Type:  max
------------------------------------------------
   Point                      Incr       Path
------------------------------------------------
   clock core_clk (rise)      0.00       0.00
   U1/reg_A/CK (DFF)          0.00       0.00
   U1/reg_A/Q  (DFF)          0.18       0.18
   U2/A1 (AND2)               0.09       0.27
   U2/Z  (AND2)               0.11       0.38
   ... (more logic stages)
   U5/reg_D/D  (DFF)          0.05       1.92
------------------------------------------------
   data arrival time                     1.92

   clock core_clk (rise)      2.00       2.00
   U5/reg_D/CK (DFF)          0.00       2.00
   library setup              -0.08      1.92
------------------------------------------------
   data required time                    1.92
------------------------------------------------
   slack (MET)                           0.00
```

### 10.4 Multiple Critical Paths

Real designs have thousands of paths close to the worst slack value — these are collectively called **near-critical paths**. Fixing the single worst path often just promotes the *next* path to "critical" status — this is why timing closure is iterative (Part 2, Section on Timing Closure).

---

## 11. Maximum Clock Frequency

### 11.1 Derivation from the Setup Equation

Recall:
```
Tclk_period ≥ Tcq(max) + Tcombo(max) + Tsetup
```

Rearranged to find the **minimum allowed clock period**:
```
Tclk_period(min) = Tcq(max) + Tcombo(max) + Tsetup
```

And therefore the **maximum achievable clock frequency**:
```
Fmax = 1 / Tclk_period(min)
     = 1 / [ Tcq(max) + Tcombo(max) + Tsetup ]
```

### 11.2 Why the Critical Path Sets Fmax

`Tcombo(max)` in the equation above must be the **worst-case (largest)** combinational delay across **all register-to-register paths** in the design — because *every* path must individually satisfy the setup equation. The path with the **largest** `Tcombo(max)` is precisely the **critical path**, and it is this path's delay that sets the floor on `Tclk_period(min)`, hence capping `Fmax`.

> **Interview-ready one-liner:** *"The maximum frequency of a chip is limited by its slowest (critical) register-to-register path, because the clock period must be long enough to accommodate the worst-case delay of every single path in the design — you can't run the clock faster than what your slowest path allows."*

### 11.3 Worked Numerical Example

Given a design with 4 register-to-register paths with the following combinational delays (assume same Tcq = 0.2 ns, Tsetup = 0.1 ns for all flops):

| Path | Tcombo(max) | Tclk_period(min) needed | Fmax for this path alone |
|---|---|---|---|
| Path 1 | 1.0 ns | 0.2+1.0+0.1 = 1.3 ns | 769 MHz |
| Path 2 | 1.8 ns | 0.2+1.8+0.1 = 2.1 ns | 476 MHz |
| Path 3 (**critical**) | 2.5 ns | 0.2+2.5+0.1 = 2.8 ns | **357 MHz** |
| Path 4 | 1.2 ns | 0.2+1.2+0.1 = 1.5 ns | 667 MHz |

**Overall chip Fmax = 357 MHz** (set entirely by Path 3, the critical path) — even though 3 out of 4 paths could easily run much faster.

### 11.4 Practical Implication for RTL / Physical Design

- RTL coding style directly affects `Tcombo` — e.g., deep combinational logic chains (long chains of adders/muxes without pipelining) create long paths that become critical.
- **Pipelining** (inserting registers to break long combinational chains) is the primary RTL-level technique to increase Fmax — it trades latency (more cycles) for throughput (higher frequency).
- Physical design (placement/routing) affects `Tcombo` through **interconnect delay** — poor placement (cells far apart) increases wire delay and can turn an otherwise-fine path into the new critical path (deep dive in Part 2).

---

## 12. Timing Violation

### 12.1 Definition

A **timing violation** occurs when the slack (setup or hold) at any endpoint in the design is **negative**, meaning the fundamental timing equation (setup or hold) is not satisfied.

### 12.2 Setup Violation

**Cause:** Data arrives too late — combinational delay too large, clock period too short, Tcq too large, or (as we'll see in Part 2) excessive clock skew/uncertainty eating into the available time budget.

**Symptom in silicon:** The capturing flop may sample **stale/incorrect data**, or enter **metastability**.

**Typical fixes (introduced here, detailed in Part 2):**
- Reduce logic depth / use faster (higher-drive) cells
- Reduce interconnect delay via better placement / buffer insertion
- Increase clock period (reduce frequency) — often not acceptable if performance is a spec requirement
- Use useful skew to borrow time from an adjacent path
- Logic restructuring / re-pipelining

### 12.3 Hold Violation

**Cause:** Data arrives too early relative to the capturing edge — combinational path is too fast (very few logic stages, short wires), especially dangerous in **paths with zero or very short logic between two flops on the same clock edge**.

**Symptom in silicon:** The capturing flop may sample the **new (next cycle's) data** instead of the intended current-cycle data — a race condition. This is often catastrophic because, unlike setup, **it cannot be fixed by changing the clock frequency** — the chip will fail at *any* frequency, including DC (0 Hz)!

**Typical fixes (introduced here, detailed in Part 2):**
- Insert delay buffers in the fast path to intentionally slow it down
- Increase drive strength mismatch fixes
- Use hold-fixing cells (delay buffers, useful skew)

### 12.4 Setup vs Hold Violation — Quick Comparison

| | Setup Violation | Hold Violation |
|---|---|---|
| Root cause | Path too slow | Path too fast |
| Depends on clock frequency | Yes — can sometimes fix by slowing clock | No — persists at any frequency |
| Fix direction | Speed up the path / relax period | Slow down the path (add delay) |
| Severity if unfixed | Functional failure at target frequency | Functional failure at **any** frequency — must always be fixed |
| When checked in flow | Increasingly critical as frequency target rises | Must be clean at every stage, independent of frequency target |

> **Interview gold nugget:** *"Hold violations must be fixed before tape-out no matter what, because they represent a fundamental race condition that exists regardless of clock speed. Setup violations, in contrast, are frequency-dependent and can sometimes be tolerated by relaxing performance targets."*

---

## 13. Introduction to Common Timing Report Terms (Preview)

(Full deep-dive with annotated real report walkthroughs is in **Part 2**, but here are the essential terms to know now so Part 1 is self-contained.)

| Term | Meaning |
|---|---|
| **Startpoint** | Where the timing path begins (a clock pin of a launch flop, or an input port) |
| **Endpoint** | Where the timing path ends (a data pin of a capture flop, or an output port) |
| **Path Group** | The clock domain the path belongs to |
| **Path Type: max** | Setup check |
| **Path Type: min** | Hold check |
| **Incr** | Incremental delay contributed by that specific stage |
| **Path (cumulative)** | Running total of delay up to that point |
| **Data Arrival Time** | Explained in Section 7 |
| **Data Required Time** | Explained in Section 7 |
| **Slack** | Explained in Section 8; `(MET)` = positive/zero, `(VIOLATED)` = negative |

---

## Part 1 Summary Table (Quick Revision)

| Concept | Core Equation |
|---|---|
| Setup timing equation | `Tclk ≥ Tcq(max) + Tcombo(max) + Tsetup` |
| Hold timing equation | `Tccq(min) + Tcombo(min) ≥ Thold` |
| Data Arrival Time (setup) | `AT = Tcq(max) + Tcombo(max)` |
| Data Required Time (setup) | `RT = Tclk − Tsetup` |
| Setup Slack | `Slack = RT − AT` |
| Hold Slack | `Slack = AT(min) − Thold` |
| Max Frequency | `Fmax = 1 / [Tcq(max) + Tcombo(max)_critical + Tsetup]` |

---

**End of Part 1.** Say **"next"** when you're ready for **Part 2**: Clock Skew, Clock Uncertainty/Jitter, Register-to-Register / Input-to-Register / Register-to-Output timing in depth, Multi-cycle Paths, False Paths, Timing Closure, Physical Design impact on timing, Setup & Hold fixes, Interconnect vs Logic delay, Path-based timing understanding, full report interpretation, and interview Q&A.
