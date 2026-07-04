# Static Timing Analysis (STA) — Complete Study Notes
## PART 2: Clock Skew → Uncertainty/Jitter → Path Types → Exceptions → Physical Design → Fixes → Interview Prep

> **Recap of Part 1:** Timing fundamentals, STA overview, propagation/contamination delay, setup/hold time, arrival/required time, slack, critical path, max frequency, timing violations.
>
> **This part covers:** Clock Skew, Clock Uncertainty/Jitter, Register-to-Register / Input-to-Register / Register-to-Output timing (in depth, with the full equations including skew and uncertainty), Multi-cycle Paths, False Paths, Common Timing Report Terms (full walkthrough), Physical Design impact on timing, Setup & Hold fixes, Interconnect vs Logic delay, Path-based understanding, and an Interview Q&A bank.

---

## 14. Clock Skew

### 14.1 Definition

**Clock skew** is the difference in **arrival time of the same clock edge** at two different flip-flops (the launching flop and the capturing flop), caused by differences in clock network delay (different wire lengths/buffers from the clock source to each flop).

```
Clock Source
     │
     ├──── (path A, delay = 1.0 ns) ──── Launch Flop (reg_A)
     │
     └──── (path B, delay = 1.3 ns) ──── Capture Flop (reg_B)

Skew = Delay(path B) − Delay(path A) = 1.3 − 1.0 = 0.3 ns
```

Ideally, the clock would arrive at every flip-flop at exactly the same instant (a "zero-skew" clock tree). In reality, the **Clock Tree Synthesis (CTS)** step tries to balance this, but perfect balance is never achieved, especially across a large die.

### 14.2 Types of Skew

| Type | Description |
|---|---|
| **Global skew** | Difference in clock arrival across the entire chip / clock domain |
| **Local skew** | Difference in clock arrival between two adjacent, timing-related flops (what actually matters for a specific path's setup/hold check) |
| **Positive skew** | Capture clock arrives **later** than launch clock (capture edge delayed relative to launch edge) |
| **Negative skew** | Capture clock arrives **earlier** than launch clock |
| **Useful skew** | Deliberately introduced skew (intentional, via CTS constraints) to **help** a specific critical path meet timing by "borrowing" time from an adjacent, non-critical path |

### 14.3 Effect of Skew on Setup Analysis

Revisiting the setup equation from Part 1, now including skew:

```
Tclk_period + Skew ≥ Tcq(max) + Tcombo(max) + Tsetup

  where Skew = T_capture_clock_arrival − T_launch_clock_arrival
```

- If skew is **positive** (capture clock arrives later than launch clock) → this **relaxes** the setup requirement → **effectively gives the data path more time** → helps setup.
- If skew is **negative** (capture clock arrives earlier than launch clock) → this **tightens** the setup requirement → **effectively less time** for data → hurts setup.

**Setup Slack equation with skew:**
```
Setup Slack = (Tclk_period + Skew) − [Tcq(max) + Tcombo(max) + Tsetup]
```

### 14.4 Effect of Skew on Hold Analysis

For hold, the relationship **reverses**:

```
Tccq(min) + Tcombo(min) ≥ Thold + Skew
```

- **Positive skew** (capture clock later than launch) → **hurts** hold (makes the effective hold window at the capture flop shift later, closer to when new data has already started to arrive) → **increases hold violation risk**.
- **Negative skew** → **helps** hold.

**Hold Slack equation with skew:**
```
Hold Slack = [Tccq(min) + Tcombo(min)] − [Thold + Skew]
```

### 14.5 Setup vs Hold — Opposite Sensitivity to Skew (Critical Interview Concept)

| | Setup | Hold |
|---|---|---|
| Positive skew (capture later than launch) | **Helps** (more time available) | **Hurts** (less margin, race risk) |
| Negative skew (capture earlier than launch) | **Hurts** (less time available) | **Helps** (more margin) |

> **Interview one-liner:** *"Skew that helps setup always hurts hold, and vice versa — this is exactly why CTS engineers use 'useful skew' carefully: borrowing skew to fix a setup-critical path can create a hold violation on the very same path, so both checks must be re-verified together."*

### 14.6 Numerical Example — Setup with Skew

**Given:**
- Clock period = 2.0 ns
- Launch clock arrives at t = 0
- Capture clock arrives at t = 0.15 ns (positive skew = +0.15 ns, capture is later)
- Tcq(max) = 0.2 ns, Tcombo(max) = 1.5 ns, Tsetup = 0.1 ns

```
Setup Slack = (Tclk + Skew) − (Tcq + Tcombo + Tsetup)
            = (2.0 + 0.15) − (0.2 + 1.5 + 0.1)
            = 2.15 − 1.8
            = +0.35 ns → MET (skew helped, compared to +0.20 ns without skew)
```

### 14.7 Numerical Example — Hold with the Same Skew

**Given (same skew = +0.15 ns):**
- Tccq(min) = 0.1 ns, Tcombo(min) = 0.2 ns, Thold = 0.08 ns

```
Hold Slack = (Tccq(min) + Tcombo(min)) − (Thold + Skew)
           = (0.1 + 0.2) − (0.08 + 0.15)
           = 0.3 − 0.23
           = +0.07 ns → MET, but margin shrank from what it would've been with zero skew (0.3 − 0.08 = 0.22 ns)
```

This clearly shows the **same skew helped setup (+0.15 ns extra margin) but hurt hold (−0.15 ns less margin)** — a direct trade-off.

### 14.8 Where Skew Appears in Reports

```
   clock core_clk (rise edge)         2.00      2.00
   clock network delay (launch)       0.45      0.45
   ...
   clock core_clk (rise edge)         2.00      2.00
   clock network delay (capture)      0.60      0.60      <- different from launch = skew source
```

The tool computes skew implicitly as the difference between the two clock network delay values shown in the launch and capture clock paths of the timing report.

---

## 15. Clock Uncertainty / Jitter

### 15.1 Definition

**Clock uncertainty** is a designer-specified timing margin (set via SDC: `set_clock_uncertainty`) that is **subtracted** from the timing budget to account for unpredictable, unmodeled variations in the clock signal, primarily:

1. **Jitter** — cycle-to-cycle variation in clock period, caused by noise in the PLL/clock generation circuit (electrical, not a static/structural delay — cannot be precisely modeled statically)
2. **Skew that hasn't been finalized yet** (in early-stage STA before CTS, an "estimated skew" margin is added as part of uncertainty)
3. **OCV (On-Chip Variation) margins** — extra pessimism for the possibility that different parts of the chip experience slightly different PVT conditions simultaneously (deep dive in Section 19)

### 15.2 Types of Jitter

| Type | Description |
|---|---|
| **Period jitter** | Variation in a single clock period vs the nominal/ideal period |
| **Cycle-to-cycle jitter** | Variation between consecutive clock periods |
| **Long-term/accumulated jitter** | Drift accumulated over many cycles |

### 15.3 Why Uncertainty is Needed (Even Though It's "Pessimistic")

STA is a **static, deterministic** analysis — it cannot model random electrical noise directly. Instead, the design margins for jitter/uncertainty are captured as a **fixed timing "tax"** applied uniformly to every setup/hold check, ensuring the design is robust even under the worst realistic clock noise.

### 15.4 Setup Equation With Uncertainty

```
Tclk_period + Skew − Uncertainty ≥ Tcq(max) + Tcombo(max) + Tsetup

Setup Slack = (Tclk_period + Skew − Uncertainty) − (Tcq(max) + Tcombo(max) + Tsetup)
```

Uncertainty **always reduces** the available time budget for setup (it's a pure pessimism margin subtracted from the "good" side).

### 15.5 Hold Equation With Uncertainty

```
Tccq(min) + Tcombo(min) ≥ Thold + Skew + Uncertainty

Hold Slack = (Tccq(min) + Tcombo(min)) − (Thold + Skew + Uncertainty)
```

For hold, uncertainty is **added** to the required side — it also always makes hold **harder to meet** (this is different from skew, which can help one check while hurting the other; uncertainty is pessimistic for **both** setup and hold simultaneously).

### 15.6 Numerical Example

**Given:**
- Clock period = 2.0 ns, Skew = 0 (ideal, for simplicity), Uncertainty = 0.08 ns
- Tcq(max) = 0.2 ns, Tcombo(max) = 1.6 ns, Tsetup = 0.1 ns

```
Setup Slack = (2.0 + 0 − 0.08) − (0.2 + 1.6 + 0.1)
            = 1.92 − 1.9
            = +0.02 ns → MET, but barely (uncertainty ate 0.08 ns of margin)
```

Without uncertainty, slack would have been 2.0 − 1.9 = **+0.10 ns** — so uncertainty directly cost this path **0.08 ns** of margin, nearly causing a violation.

### 15.7 Where Uncertainty is Set (SDC Command)

```tcl
set_clock_uncertainty -setup 0.08 [get_clocks core_clk]
set_clock_uncertainty -hold  0.05 [get_clocks core_clk]
```

### 15.8 Skew vs Uncertainty — Comparison

| | Clock Skew | Clock Uncertainty |
|---|---|---|
| Source | Structural (clock tree imbalance) | Electrical/statistical (jitter, unmodeled variation) + early-stage margin |
| Sign | Can be positive or negative | Always treated as a margin that hurts (subtracted from budget) |
| Effect on setup | Can help or hurt | Always hurts (reduces available time) |
| Effect on hold | Can help or hurt (opposite of setup) | Always hurts (adds to required time) |
| Computed by tool automatically? | Yes, from actual clock tree delays (post-CTS) | No — manually specified by designer via SDC |
| Reduces as flow progresses? | Refined/reduced after CTS optimization | Usually reduced in later stages once real jitter data / balanced clock tree is available |

---

## 16. Register-to-Register Timing (Deep Dive)

### 16.1 Definition

The most common and fundamental timing path type: data launched from one flip-flop's output, through combinational logic, into another flip-flop's input — both flops clocked by the same or related clock.

### 16.2 Full Setup Equation (All Terms Combined)

```
Tclk_period ≥ Tcq(max) + Tcombo(max) + Tsetup − Skew + Uncertainty
```
(Rearranged so all "cost" terms are on the right, showing that a positive skew value reduces the effective right-hand side, i.e. helps.)

Equivalently, as commonly written in STA literature:

```
Data Required Time = Tclk_period + T_capture_clock_latency − Tsetup − Uncertainty
Data Arrival Time   = T_launch_clock_latency + Tcq(max) + Tcombo(max)

Setup Slack = Data Required Time − Data Arrival Time
```

### 16.3 Full Hold Equation (All Terms Combined)

```
Data Arrival Time (min) = T_launch_clock_latency + Tccq(min) + Tcombo(min)
Data Required Time (hold) = T_capture_clock_latency + Thold + Uncertainty

Hold Slack = Data Arrival Time(min) − Data Required Time(hold)
```

### 16.4 Why Reg-to-Reg is the "Default" Path Type

Most of a synchronous design's logic consists of these paths. STA tools automatically identify all reg-to-reg paths from the netlist + SDC clock definitions, with no special constraints needed (unlike I/O paths, which require explicit input/output delay constraints — see Sections 17–18).

### 16.5 Complete Worked Example (Realistic, With All Margins)

**Given:**
- Clock period = 2.5 ns
- Launch clock network latency = 0.30 ns, Capture clock network latency = 0.42 ns → **Skew = +0.12 ns**
- Setup uncertainty = 0.05 ns
- Tcq(max) = 0.18 ns, Tcombo(max) = 1.65 ns, Tsetup = 0.09 ns

**Data Arrival Time:**
```
AT = 0.30 (launch clock latency) + 0.18 (Tcq) + 1.65 (Tcombo) = 2.13 ns
```

**Data Required Time:**
```
RT = 2.5 (period) + 0.42 (capture clock latency) − 0.09 (Tsetup) − 0.05 (uncertainty) = 2.78 ns
```

**Setup Slack:**
```
Slack = RT − AT = 2.78 − 2.13 = +0.65 ns → MET
```

*(Note: clock latencies are added directly to both sides in tool reports because the tool tracks absolute time from a common t=0 reference — the "+Skew" simplified formula in 16.2 is the same thing algebraically simplified.)*

---

## 17. Input-to-Register Timing

### 17.1 Definition

A timing path that starts at a **primary input port** of the chip (not a flip-flop) and ends at a **register (flip-flop)** inside the design. Since STA has no visibility into what happens **outside** the chip (in the external circuit driving this input), the designer must **explicitly specify** the assumed timing behavior of the external driver via an **input delay constraint**.

### 17.2 Why Input Delay Constraint is Needed

Without it, STA has no starting "arrival time" reference for the input signal, and cannot know how much of the clock period is "already consumed" before the signal even enters the chip (e.g., time spent in an external buffer, board trace, or another chip's output flop).

### 17.3 SDC Command

```tcl
set_input_delay -clock CLK -max 0.8 [get_ports data_in]
set_input_delay -clock CLK -min 0.2 [get_ports data_in]
```

This tells STA: *"Assume the external world can deliver a valid signal at `data_in` no later than 0.8 ns after the CLK edge (max, for setup), and no earlier than 0.2 ns after the CLK edge (min, for hold)."*

### 17.4 Setup Equation for Input-to-Register Path

```
Data Arrival Time = Input_Delay(max) + Tcombo(max)  [inside the chip, from port to reg D pin]
Data Required Time = Tclk_period − Tsetup − Uncertainty  [same as reg-to-reg]

Setup Slack = Data Required Time − Data Arrival Time
```

### 17.5 Numerical Example

**Given:**
- Clock period = 3.0 ns
- `set_input_delay -max 0.7` on `data_in`
- Internal combinational delay from port to reg D pin (max) = 1.2 ns
- Tsetup = 0.1 ns, Uncertainty = 0.05 ns

```
AT = 0.7 (input delay) + 1.2 (combo) = 1.9 ns
RT = 3.0 − 0.1 − 0.05 = 2.85 ns

Setup Slack = 2.85 − 1.9 = +0.95 ns → MET
```

### 17.6 Physical Meaning of Input Delay

Input delay essentially models "how much of the clock period has already been used up by the external device driving this pin, before the signal even reaches our chip boundary." A large input delay value effectively **shrinks the time budget** available for on-chip logic on that path.

---

## 18. Register-to-Output Timing

### 18.1 Definition

A timing path that starts at a **flip-flop** inside the design and ends at a **primary output port**. Just like input paths need an input delay, output paths need an **output delay constraint** to model the timing requirement of the external circuit receiving this output.

### 18.2 SDC Command

```tcl
set_output_delay -clock CLK -max 0.6 [get_ports data_out]
set_output_delay -clock CLK -min 0.15 [get_ports data_out]
```

This tells STA: *"The external receiving device needs the output signal to arrive with at least 0.6 ns of margin before the next capturing clock edge on the external side (max, for setup), and the signal must not change earlier than 0.15 ns relative to hold requirements on the external side (min, for hold)."*

### 18.3 Setup Equation for Register-to-Output Path

```
Data Arrival Time = Tcq(max) + Tcombo(max)  [from reg Q pin to output port]
Data Required Time = Tclk_period − Output_Delay(max) − Uncertainty

Setup Slack = Data Required Time − Data Arrival Time
```

### 18.4 Numerical Example

**Given:**
- Clock period = 2.5 ns
- Tcq(max) = 0.2 ns, Tcombo(max) (reg to output port) = 1.1 ns
- `set_output_delay -max 0.5` on `data_out`
- Uncertainty = 0.03 ns

```
AT = 0.2 + 1.1 = 1.3 ns
RT = 2.5 − 0.5 − 0.03 = 1.97 ns

Setup Slack = 1.97 − 1.3 = +0.67 ns → MET
```

### 18.5 The Four Fundamental Path Types — Summary Table

| # | Path Type | Startpoint | Endpoint | Extra Constraint Needed |
|---|---|---|---|---|
| 1 | Register-to-Register | Flop (CK) | Flop (D) | None (uses clock definition only) |
| 2 | Input-to-Register | Input port | Flop (D) | `set_input_delay` |
| 3 | Register-to-Output | Flop (CK) | Output port | `set_output_delay` |
| 4 | Input-to-Output | Input port | Output port | Both `set_input_delay` and `set_output_delay` (combinational-only path, no clock relationship — often analyzed as a pure combinational max-delay path) |

### 18.6 RTL / FPGA Connection

- **I/O timing constraints are especially critical in FPGA design** — Vivado/Quartus timing closure heavily depends on correct input/output delay constraints matching the actual board-level timing (e.g., DDR interfaces, SPI, high-speed serial I/O).
- Forgetting to constrain an I/O path is one of the most common **STA setup mistakes** — the tool will treat unconstrained paths as **"false paths" by omission**, silently skipping them, which can mask real violations (dangerous in sign-off).

---

## 19. On-Chip Variation (OCV) — Brief Note (Context for Skew/Uncertainty)

Modern STA doesn't just use one single "slow" and one single "fast" number for the whole chip — because even within the **same corner**, different transistors on the same die can vary slightly due to local process gradients. This is handled via:

- **Derate factors** (e.g., `set_timing_derate`) that adjust cell delays up (for launch path, pessimistically slower) and down (for capture path, pessimistically faster) simultaneously in the **same** setup check — called **AOCV/POCV (Advanced/Parametric OCV)** in modern flows.
- This adds *additional* pessimism beyond simple skew/uncertainty, particularly important at advanced nodes (16nm and below).

*(This is an advanced topic — good to mention in interviews for depth, but the core mental model of skew + uncertainty from Sections 14–15 covers what's needed for solid fundamentals.)*

---

## 20. Multi-Cycle Paths (MCP)

### 20.1 Definition

A **multi-cycle path** is a timing path that, by design intent, is **allowed multiple clock cycles** to complete — rather than being required to complete within a single clock period. This is a **timing exception** that must be explicitly declared to the STA tool; otherwise, the tool will (correctly, by default) assume every path must complete in **one cycle** and may report a false violation.

### 20.2 Why Multi-Cycle Paths Exist

Some data paths are functionally designed to only need to settle every N cycles — for example:
- A slow control register updated only once every 4 cycles
- A divider or multiplier whose output is only sampled after several cycles of computation
- Data paths gated by an enable that's only asserted periodically

### 20.3 SDC Command

```tcl
set_multicycle_path 2 -setup -from [get_pins regA/CK] -to [get_pins regB/D]
set_multicycle_path 1 -hold  -from [get_pins regA/CK] -to [get_pins regB/D]
```

### 20.4 Modified Setup Equation for a 2-Cycle MCP

```
N × Tclk_period ≥ Tcq(max) + Tcombo(max) + Tsetup − Skew + Uncertainty
```

Where N = number of cycles allowed (here, 2). This **relaxes** the setup requirement by giving `N × Tclk_period` instead of just one period.

### 20.5 Numerical Example

**Given:** Clock period = 1.0 ns, Multicycle setup factor N = 3, Tcombo(max) = 2.4 ns, Tcq(max) = 0.15 ns, Tsetup = 0.08 ns

**Without MCP (would incorrectly show a violation):**
```
Required (1 cycle) = 1.0 − 0.08 = 0.92 ns
Arrival = 0.15 + 2.4 = 2.55 ns
Slack = 0.92 − 2.55 = −1.63 ns → FALSE VIOLATION
```

**With correct MCP=3 declared:**
```
Required (3 cycles) = 3.0 − 0.08 = 2.92 ns
Arrival = 2.55 ns
Slack = 2.92 − 2.55 = +0.37 ns → Correctly MET
```

### 20.6 Important Hold Consideration for MCP

When a setup multicycle is applied, the **hold check reference edge also shifts** — by default, most tools move the hold check to the *cycle just before* the new (relaxed) setup capture edge, **not** back to the original single-cycle edge. This is a very common designer mistake: applying only a setup MCP without also explicitly considering/declaring the hold MCP can create a **new, unintended hold violation risk** because the hold check window has effectively widened. Always specify both setup and hold multicycle values deliberately (as shown in Section 20.3), and verify with the tool's actual computed check.

---

## 21. False Paths

### 21.1 Definition

A **false path** is a timing path that exists **structurally** in the netlist (STA can trace a route through gates from a startpoint to an endpoint) but **can never actually be sensitized/exercised functionally** in real operation — i.e., it's not a real functional data path, so it should be **excluded entirely** from timing analysis (not just relaxed, like a multi-cycle path).

### 21.2 Common Reasons a Path is False

1. **Asynchronous clock domain crossings (CDC)** — paths between two flops clocked by unrelated/asynchronous clocks; setup/hold relationship is meaningless because there's no fixed phase relationship (handled instead by synchronizers, not by STA timing closure).
2. **Static configuration/scan/test-mode paths** — e.g., a path only active during scan-shift mode, or a mode-select signal that is tied to a constant in functional mode.
3. **Mutually exclusive paths** — e.g., two paths that can never both be active due to mutually exclusive select/enable logic (common in muxed datapaths).
4. **Paths through unused/reset-only logic**.

### 21.3 SDC Command

```tcl
set_false_path -from [get_clocks clkA] -to [get_clocks clkB]
set_false_path -through [get_pins u_mux/S]
```

### 21.4 Effect on STA

Once declared, the STA tool **completely ignores** that path for both setup and hold checks — it will not appear in violation reports at all, and (critically) it also won't consume tool runtime/analysis resources on it.

### 21.5 Multi-Cycle Path vs False Path — Comparison

| | Multi-Cycle Path | False Path |
|---|---|---|
| Meaning | Path is real, but allowed more than 1 cycle | Path is not functionally meaningful at all |
| STA behavior | Relaxes the required time by N cycles | Completely excludes the path from analysis |
| Common use case | Slow/periodically-updated datapaths | CDC, scan/test paths, mutually exclusive logic |
| Risk if misused | Can hide a real violation if N is set too generously | Can hide a REAL functional bug if misapplied to a path that actually matters — very dangerous if wrong |
| SDC command | `set_multicycle_path` | `set_false_path` |

> **Interview caution:** *Both exceptions are powerful but dangerous — a wrongly applied false path or overly generous multicycle path can mask a genuine silicon bug. Sign-off reviews specifically audit every SDC exception for justification. This is one of the most scrutinized parts of the constraints file during tape-out review.*

### 21.6 Case-by-Case (Design) Exceptions — Related Concept

Similar in spirit but distinct: `set_case_analysis` fixes a signal to a constant (0 or 1) for analysis purposes (e.g., a mode pin tied off in the actual application), which lets the tool automatically prune/simplify paths that become non-functional under that fixed condition — often used together with or instead of manually specified false paths.

---

## 22. Interconnect Delay and Logic (Cell) Delay

### 22.1 Definition

Total path delay in any stage is the sum of:
```
Total Delay = Logic (Cell) Delay + Interconnect (Net/Wire) Delay
```

- **Logic/Cell delay:** Intrinsic delay of the standard cell (gate) itself — determined by transistor sizing, input slew, and output load.
- **Interconnect delay:** Delay of the metal wire connecting one cell's output to the next cell's input — determined by wire resistance (R) and capacitance (C), i.e., RC delay.

### 22.2 Why This Split Matters

- **Pre-layout (pre-placement) STA** uses only *estimated* wire delay (via wire-load models or zero-wire-delay assumptions) — because actual physical wire routes don't exist yet.
- **Post-layout STA** uses *extracted* parasitics (SPEF file, from actual routed metal) — this is the **accurate, sign-off-quality** delay number.
- As technology nodes shrink, **interconnect delay increasingly dominates** over logic delay (wires don't scale as favorably as transistors) — in advanced nodes, a large fraction of critical path delay can be **wire delay**, not gate delay.

### 22.3 Elmore Delay Model (Simplified Interconnect Delay Estimate)

A commonly taught simplified model for RC wire delay:

```
Tdelay ≈ 0.69 × R × C     (for a simple RC step response, time to reach 50%)
```

Where R = total wire resistance, C = total wire + downstream gate input capacitance. Real STA tools use much more sophisticated models (effective capacitance, non-linear delay models / NLDM, or current-source models / CCS from Liberty), but Elmore delay is the standard **first-principles teaching model**.

### 22.4 Numerical Example

Given a wire with R = 200 Ω, C = 15 fF:
```
Tdelay ≈ 0.69 × 200 × 15×10⁻¹⁵ = 0.69 × 3000×10⁻¹⁵ = 2.07×10⁻¹² s ≈ 2.07 ps
```

(Real interconnect segments in advanced nodes can range from a few ps for short local wires up to 50-100+ ps for long global routes without buffering — which is why **buffer insertion** is used to break up long wires, discussed next.)

### 22.5 How Physical Design Reduces Interconnect Delay

1. **Buffer/repeater insertion** — breaks a long RC wire into multiple shorter segments, each individually driven by a buffer; total delay scales roughly **linearly** with number of buffered segments instead of **quadratically** with raw (unbuffered) wire length (since RC delay ∝ length²) — a major reason why unbuffered long wires are so much worse than buffered ones.
2. **Better placement** — placing logically-connected cells physically close together shortens wire length directly.
3. **Wire sizing/spacing, layer assignment** — using wider/higher metal layers (lower R) for critical long nets.
4. **Reducing fanout** — fewer downstream loads on a net reduces the effective capacitance the driving cell must charge/discharge.

---

## 23. Path-Based Understanding of Timing (Putting It All Together)

### 23.1 The "Path" as the Fundamental Unit of STA

Every single timing check in STA is fundamentally about **one path**: a sequence of connected pins (through cells and nets) from a **startpoint** to an **endpoint**, evaluated against a required time derived from the relevant clock(s).

### 23.2 Anatomy of a Full Timing Path Report (Fully Annotated)

```
Startpoint: regA (rising edge-triggered flip-flop clocked by clk)
Endpoint:   regB (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type:  max                                    <- SETUP check

Point                                Incr      Path
-----------------------------------------------------
clock clk (rise edge)                0.00      0.00     <- ideal/source clock time
clock network delay (launch)         0.35      0.35     <- clock tree delay to regA
regA/CK (DFF)                        0.00      0.35
regA/Q (DFF)                         0.18      0.53     <- Tcq(max) of launch flop
U1/Z (BUF)                           0.06      0.59     <- cell delay
net1 (net)                           0.04      0.63     <- interconnect delay
U2/Z (AND2)                          0.09      0.72     <- cell delay
net2 (net)                           0.03      0.75     <- interconnect delay
regB/D (DFF)                         0.00      0.75
-----------------------------------------------------
data arrival time                              0.75

clock clk (rise edge)                1.00      1.00     <- next clock edge (period = 1.00 ns)
clock network delay (capture)        0.40      1.40     <- clock tree delay to regB
regB/CK (DFF)                        0.00      1.40
library setup time                  -0.05      1.35     <- Tsetup subtracted
-----------------------------------------------------
data required time                             1.35
-----------------------------------------------------
slack (MET)                                    0.60     <- 1.35 − 0.75
```

### 23.3 Reading This Report — Step by Step

1. **Top section (data arrival time):** Traces the actual physical path the data takes, accumulating delay stage by stage, starting from the launch clock edge.
2. **Bottom section (data required time):** Traces the capture clock's path to the capture flop, then subtracts the setup time margin — giving the deadline.
3. **Slack = required − arrival.**
4. Every **cell** and **net** entry shown separately lets you see exactly how much delay comes from **logic** vs **interconnect** — critical for debugging *which* stage to optimize.

### 23.4 Debugging a Path from a Report (Practical Skill)

When a path shows negative slack, a working engineer looks at the **Incr** column to find the largest individual contributors:
- If a **net (interconnect)** delay is unusually large → physical design issue (bad placement, long unbuffered route) → fix via buffering/placement.
- If a **cell (logic)** delay is unusually large → possibly a weak/undersized gate, or too much fanout/load → fix via upsizing the cell or reducing fanout.
- If **many small stages** add up → the logic is simply too deep → fix via RTL restructuring/pipelining.

---

## 24. Physical Design Impact on Timing (Consolidated View)

| Physical Design Stage | Timing Impact |
|---|---|
| **Floorplanning** | Determines macro/block placement — poor floorplan → long cross-chip routes → large interconnect delay on critical paths |
| **Placement** | Determines standard cell locations — directly controls wire length between connected cells |
| **Clock Tree Synthesis (CTS)** | Determines clock latency & skew at every flop — directly affects setup/hold slack (Section 14) |
| **Routing** | Determines actual wire geometry (R, C) — the real, final interconnect delay used in sign-off STA |
| **Post-route optimization (ECO)** | Cell sizing, buffer insertion, spare-cell usage to fix late-found violations without full re-placement/re-route |

### 24.1 Why Timing Can "Get Worse" After Layout

Pre-layout STA (using wire-load models) is only an **estimate**. After actual placement and routing, real wire lengths/parasitics are known — commonly **longer and more capacitive** than pre-layout estimates for congested designs, meaning **paths that looked fine pre-layout may become critical or even violating post-layout**. This is why **post-layout (sign-off) STA is always the final authority**, and why physical design + STA are run **iteratively** together (not as one-shot separate steps).

---

## 25. Setup and Hold Fixes (Practical Design Flow Techniques)

### 25.1 Setup Violation Fixes

| Technique | How It Helps |
|---|---|
| **Gate upsizing** | Larger drive strength cells switch faster → reduces cell delay |
| **Buffer insertion / logic restructuring** | Reduces long wire delay, or restructures logic to reduce depth |
| **Reduce fanout** (buffer/clone driving cell) | Less capacitive load per driver → faster switching |
| **Re-placement of cells** (physically move closer) | Shortens interconnect delay |
| **Pipelining (adding a register stage)** | Splits one long combinational path into two shorter paths — trades latency for frequency headroom |
| **Useful skew (via CTS)** | Deliberately delay the capture clock relative to launch clock on this specific path to add margin |
| **Relaxing clock period (if allowed)** | Directly increases available time — often not acceptable if performance target is fixed |
| **Logic re-synthesis / re-mapping** | Choosing a different, faster gate implementation of the same logic function |

### 25.2 Hold Violation Fixes

| Technique | How It Helps |
|---|---|
| **Delay buffer insertion in the fast path** | Deliberately slows down data arrival to satisfy `Tccq(min) + Tcombo(min) ≥ Thold` |
| **Downsizing a cell (reduce drive strength)** | Slower switching → adds delay (used carefully; can affect other paths) |
| **Useful skew (opposite direction from setup fix)** | Advance the capture clock relative to launch to widen the hold margin |
| **Increasing load on the driving cell** | Increases delay (used sparingly) |

> **Important trade-off note:** Hold fixes (adding delay) must be applied **very carefully** because they can potentially **reintroduce a setup violation** if too much delay is added, and — since hold fixes are usually done via small delay/buffer insertion cells inserted **after** placement/routing (ECO stage) — they must not disturb the already-closed floorplan/routing significantly. This is why hold fixing is typically one of the **very last steps** in the physical design flow, done via **ECO (Engineering Change Order)**, after setup and placement/routing are already largely finalized (since hold is independent of frequency and clock-tree-final-shape-dependent).

### 25.3 Why Hold Fixing Happens Late, Setup Fixing Happens Early

- Setup timing depends heavily on **clock period target** and overall logic structure → addressed early (synthesis, floorplanning) and refined through placement.
- Hold timing depends on the **final clock tree** (post-CTS skew values) → cannot be reliably fixed until the clock tree is essentially finalized, hence hold fixing is one of the last steps (post-CTS / post-route ECO).

---

## 26. Timing Closure

### 26.1 Definition

**Timing closure** is the overall iterative process of converging the design to a state where **all setup and hold checks pass (zero or acceptable margin of negative slack)** across **all corners, all modes, and all path groups**, ready for sign-off / tape-out.

### 26.2 Typical Timing Closure Flow

```
Synthesis (initial gate-level netlist, pre-layout STA)
        ↓
Floorplan + Placement → Post-placement STA (early physical estimate)
        ↓
Clock Tree Synthesis (CTS) → Post-CTS STA (real clock skew known now)
        ↓
Routing → Post-route STA (SIGN-OFF QUALITY — real parasitics)
        ↓
   [If violations found]
        ↓
ECO (Engineering Change Order): buffer insert/resize/re-route locally
        ↓
Re-run STA → repeat until clean
```

### 26.3 Signoff Criteria (Typical)

- **WNS (Worst Negative Slack) ≥ 0** for setup and hold, across all corners
- **TNS (Total Negative Slack) = 0** (or within an agreed, waived threshold)
- Clean across **all defined path groups / modes** (e.g., functional mode, scan/test mode, low-power mode)
- Confirmed with **on-chip variation (OCV/AOCV)** margins applied
- All SDC exceptions (false paths, multicycle paths) reviewed and justified

### 26.4 Why Timing Closure Is Iterative, Not One-Shot

Fixing one critical path (e.g., via buffer insertion) changes local placement/loading, which can shift which path is now the *next* worst — and physical changes (ECO) can themselves introduce **new** hold violations on adjacent paths. This is why real chips go through **multiple STA-ECO iterations** before sign-off.

---

## 27. Full Consolidated Equation Reference Sheet

| Check | Equation |
|---|---|
| Setup (basic) | `Tclk ≥ Tcq(max) + Tcombo(max) + Tsetup` |
| Hold (basic) | `Tccq(min) + Tcombo(min) ≥ Thold` |
| Setup (with skew & uncertainty) | `Tclk + Skew − Uncertainty ≥ Tcq(max) + Tcombo(max) + Tsetup` |
| Hold (with skew & uncertainty) | `Tccq(min) + Tcombo(min) ≥ Thold + Skew + Uncertainty` |
| Setup Slack | `Slack = RT − AT` |
| Hold Slack | `Slack = AT(min) − RT(hold)` |
| Max Frequency | `Fmax = 1 / [Tcq(max) + Tcombo(max)_critical + Tsetup]` |
| Multi-cycle setup (N cycles) | `N × Tclk ≥ Tcq(max) + Tcombo(max) + Tsetup − Skew + Uncertainty` |
| Elmore wire delay (approx.) | `Tdelay ≈ 0.69 × R × C` |

---

## 28. Interview Q&A Bank

**Q1. Why is setup checked at the slow corner and hold at the fast corner?**
A: Setup fails when data is too slow → must check the corner producing max (worst) delay = slow corner. Hold fails when data is too fast → must check the corner producing min (best/fastest) delay = fast corner.

**Q2. Can a hold violation be fixed by slowing down the clock?**
A: No. Hold violations are independent of clock period — the hold equation has no clock period term. A hold violation exists at every possible frequency, including DC.

**Q3. What happens to slack if I add a buffer directly in a setup-critical path?**
A: It typically increases delay (if inserted purely for routing) — could worsen setup slack unless the buffer improves signal integrity/drive strength enough to net-reduce delay. In general, buffers are added in setup-critical paths only when they reduce net delay (e.g., breaking up an over-long unbuffered wire) — random buffer insertion can hurt setup.

**Q4. Why does positive skew help setup but hurt hold?**
A: Positive skew delays the capture clock edge relative to launch, giving the data path more time to arrive before the deadline → helps setup. But that same delayed capture edge means the hold check window (measured from the capture edge) also shifts later, making it easier for new data to arrive before the hold window closes → hurts hold.

**Q5. What is the difference between a false path and a multicycle path?**
A: A false path is not a real functional path at all and is completely excluded from timing analysis. A multicycle path is a real functional path that is intentionally allowed more than one clock cycle to complete, and its required time is relaxed by a factor of N cycles rather than excluded.

**Q6. Why is hold fixing done late in the flow (post-CTS/post-route) while setup is optimized early?**
A: Hold checks depend on the final, real clock skew values, which are only accurately known after Clock Tree Synthesis and routing are essentially complete. Setup can be roughly addressed earlier because it's primarily governed by logic depth and target clock period, refined progressively as placement data becomes available.

**Q7. If a design has zero setup violations but a large negative hold slack, is the chip functional?**
A: No — a hold violation represents a race condition that will cause functional failure (wrong data captured) regardless of frequency; it must be fixed before tape-out. Unlike setup, there's no "just run it slower" workaround.

**Q8. What determines the maximum frequency of a chip?**
A: The critical path — the register-to-register path with the largest total delay (Tcq + Tcombo + Tsetup, adjusted for skew/uncertainty) — since every path must individually satisfy the setup timing equation, the single worst path sets the minimum allowable clock period, and hence caps the maximum frequency.

**Q9. Why does interconnect delay matter more in advanced technology nodes?**
A: Transistor (gate) delays shrink favorably with each new process node, but wire resistance/capacitance per unit length does not scale as well — so as nodes shrink, a larger proportion of total path delay comes from interconnect (wires) rather than logic gates, making physical design (placement/routing/buffering) increasingly critical to timing closure.

**Q10. What's the danger of an incorrectly specified false path?**
A: If a path that is actually functionally real and exercised is mistakenly excluded via `set_false_path`, STA will never flag a real violation on it — this can let a genuine silicon timing bug slip through to tape-out completely undetected, since STA is exhaustive only over paths it's told to check.

**Q11. What is on-chip variation (OCV) and why is a single "slow corner" not always sufficient?**
A: Even within the same nominal PVT corner, transistors across a large die can vary slightly due to local process gradients, so the launch and capture paths of the SAME check might not experience identical delay conditions in reality. OCV/derating applies extra pessimism (slowing the launch path further, speeding the capture path further, or vice versa for hold) within a single setup/hold check to guard against this intra-die variation.

**Q12. Why do you need `set_input_delay` / `set_output_delay` for I/O paths but not for reg-to-reg paths?**
A: Reg-to-reg paths have both a known launch and known capture flop with a fully defined clock relationship, so STA can compute arrival/required time from the clock definition alone. I/O paths involve external circuitry that STA has no visibility into, so the designer must explicitly model the assumed external timing behavior via input/output delay constraints — otherwise STA has no reference point for arrival (at inputs) or deadline (at outputs).

---

## Part 2 Summary — Full Topic Checklist Covered

- [x] Clock Skew (types, effect on setup/hold, useful skew)
- [x] Clock Uncertainty / Jitter (types, always-pessimistic nature, SDC)
- [x] Register-to-Register timing (full equation with skew & uncertainty)
- [x] Input-to-Register timing (`set_input_delay`)
- [x] Register-to-Output timing (`set_output_delay`)
- [x] Multi-cycle Paths (`set_multicycle_path`, setup + hold interplay)
- [x] False Paths (`set_false_path`, comparison to MCP)
- [x] Common Timing Report Terms (fully annotated real report)
- [x] Physical Design impact on timing (floorplan/placement/CTS/routing)
- [x] Setup and Hold fixes (practical techniques, why hold fixed late)
- [x] Interconnect delay vs Logic delay (Elmore model, buffering)
- [x] Path-based understanding of timing (full annotated walkthrough)
- [x] Timing Closure (flow, sign-off criteria, iterative nature)
- [x] Interview Q&A bank (12 high-value questions with model answers)

---

**This completes the two-part STA study notes.** Recommended next study steps:
1. Re-derive every equation in Section 27 from memory without looking.
2. Practice reading real `report_timing` outputs (PrimeTime/Tempus/OpenSTA) and identify AT, RT, slack, and skew contributions yourself.
2. Work through the numerical examples in both parts by changing one variable at a time (e.g., "what if Tcombo increases by 0.3 ns — does it still meet setup? What about hold?") to build strong intuition.
3. If you'd like, I can also generate: **(a)** a set of practice problems with answer keys, **(b)** a condensed one-page cheat sheet for quick revision, or **(c)** FPGA-specific (Vivado/Quartus) timing constraint examples — just let me know.
