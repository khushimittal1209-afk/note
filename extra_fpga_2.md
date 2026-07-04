# FPGA / RTL / Verification Tooling — Interview-Ready Notes
## PART 2: Quartus → UVM (Intro) → CDC → Reset Synchronizers → Final Interview Cheat-Sheet

> **Recap of Part 1:** Tcl (EDA scripting language), ModelSim (HDL simulator: compile → elaborate → simulate, waveform debugging).
>
> **This part covers:** Quartus (FPGA implementation flow), UVM introduction (why it exists, basic building blocks), Clock Domain Crossing (CDC), Reset Synchronizers — plus a final consolidated interview cheat-sheet for all six topics.

---

# 3. Quartus (Intel/Altera FPGA Design Software)

## 3.1 What It Is

**Quartus (Quartus Prime)** is Intel's (formerly Altera's) FPGA design software suite. It takes your RTL design all the way from source code to an actual **bitstream/programming file** that configures a real FPGA chip. (Xilinx's equivalent tool is **Vivado**.)

## 3.2 Why It's Used in FPGA Flows

RTL simulation (ModelSim) only proves your design is **functionally** correct. To actually run your design on physical FPGA hardware, you need a tool that:
- **Synthesizes** RTL into a netlist of FPGA-specific primitives (LUTs, flip-flops, DSP blocks, block RAMs)
- **Places and routes** that netlist onto the actual physical FPGA fabric
- Checks **timing** (does the design meet the target clock frequency on real silicon?)
- Generates the final **programming file** to load onto the chip

Quartus does all of this in one integrated tool.

## 3.3 Basic FPGA Project Flow in Quartus

```
1. Create Project (select FPGA device/family)
        ↓
2. Add RTL Source Files (.v / .vhd)
        ↓
3. Add Constraints (.sdc for timing, pin assignments)
        ↓
4. Analysis & Synthesis (RTL → generic gate-level netlist using FPGA primitives)
        ↓
5. Fitter (Place & Route) — maps netlist onto physical FPGA resources (LUTs, DSPs, I/O pins)
        ↓
6. Timing Analysis (TimeQuest) — checks setup/hold across the FPGA fabric (same STA concepts from earlier notes!)
        ↓
7. Assembler — generates the Programming File (.sof / .pof)
        ↓
8. Programmer — loads the file onto the FPGA board via USB-Blaster/JTAG
```

## 3.4 Step-by-Step Detail

### Step 1 — Create Project
Choose the exact FPGA part number (e.g., Cyclone V, Arria 10) — this matters because different FPGA families have different amounts of LUTs, DSP blocks, memory, and I/O standards.

### Step 2 — Add Files
```
Project → Add/Remove Files in Project → browse and add .v/.vhd files
```
All RTL source files (design + top-level wrapper) go here. Testbenches are **not** added here — Quartus doesn't simulate; that's ModelSim's job.

### Step 3 — Pin Assignment
Every top-level port (clock, reset, LEDs, switches, I/O) must be assigned to a **physical FPGA pin** matching the board schematic. Done via:
- **Pin Planner** (GUI tool), or
- A `.qsf` (Quartus Settings File) with lines like:
```tcl
set_location_assignment PIN_A8 -to clk_50mhz
set_location_assignment PIN_B10 -to led[0]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to led[0]
```

### Step 4 — Compile (Synthesis + Fit + Timing + Assembler, all in one click)
```
Processing → Start Compilation
```
This single action runs Analysis & Synthesis → Fitter → Timing Analysis → Assembler in sequence.

### Step 5 — Timing Analysis (TimeQuest)
Uses the **exact same STA concepts** covered in the earlier timing-analysis notes (setup/hold, slack, critical path) — but applied specifically to the FPGA's internal fabric (LUT delays, routing delays between logic elements) instead of an ASIC standard-cell library.

### Step 6 — Generate Programming File
After a clean compile, Quartus generates:
- `.sof` (SRAM Object File) — used for temporary/volatile configuration (loaded via JTAG, lost on power-off)
- `.pof` (Programmer Object File) — used to permanently program the FPGA's configuration flash memory

### Step 7 — Program the FPGA
```
Tools → Programmer → select .sof/.pof file → Start
```
Loads the design onto the physical board through a JTAG/USB-Blaster cable.

## 3.5 Practical Example (Conceptual .qsf snippet)

```tcl
set_global_assignment -name FAMILY "Cyclone V"
set_global_assignment -name TOP_LEVEL_ENTITY top
set_global_assignment -name VERILOG_FILE top.v
set_global_assignment -name VERILOG_FILE alu.v
set_global_assignment -name SDC_FILE constraints.sdc

set_location_assignment PIN_AF14 -to clk
set_location_assignment PIN_AA12 -to rst_n
```

(Notice: `.qsf` files are also **Tcl syntax** — reinforcing why Tcl fundamentals from Part 1 matter here too.)

## 3.6 Minimum Core Concepts You Must Know

- Quartus = **FPGA implementation tool**: RTL → synthesis → place & route → bitstream, for Intel/Altera FPGAs.
- The flow: **Synthesis → Fitter (Place & Route) → Timing Analysis (TimeQuest) → Assembler → Programmer.**
- Pin assignment maps logical ports to physical FPGA pins — mandatory, board-specific.
- `.sof` = temporary/volatile config file; `.pof` = permanent flash config file.
- TimeQuest performs STA **on the FPGA fabric** — same setup/hold/slack concepts you already know, just a different physical target than ASIC standard cells.
- Quartus does **not simulate** — ModelSim (or another simulator) is used separately/beforehand for functional verification.

## 3.7 Connection to RTL / Simulation / Synthesis / Verification

| Stage | Relationship |
|---|---|
| RTL | Quartus consumes the same RTL that was already simulated/verified in ModelSim |
| Simulation | Should happen **before** Quartus compilation — catching functional bugs is far cheaper in simulation than after synthesis/fitting |
| Synthesis | Quartus's "Analysis & Synthesis" step is FPGA-specific synthesis (maps to LUTs/FFs/DSPs, unlike ASIC synthesis which maps to standard cells) |
| Timing / STA | TimeQuest inside Quartus applies the exact STA principles (arrival/required time, slack, setup/hold) to the FPGA's actual physical fabric post-place-and-route |
| Verification | Post-Quartus, engineers often also do **gate-level simulation** or **on-hardware validation** using the generated bitstream, as a final sanity check beyond RTL simulation |

---

# 4. UVM (Universal Verification Methodology) — Introduction Only

## 4.1 What It Is

**UVM** is a **standardized, class-based verification methodology** built on top of SystemVerilog. It provides a structured, reusable framework for building **testbenches** — much more sophisticated than the simple testbenches shown in the ModelSim section (Part 1).

> Important distinction: UVM is **not a tool** (like ModelSim/Quartus) — it's a **methodology/library** (a set of base classes and conventions) that runs *on top of* a SystemVerilog simulator (like ModelSim, VCS, or Xcelium).

## 4.2 Why It Exists (The Problem It Solves)

Simple directed testbenches (Part 1 style) work fine for small designs, but for complex chips:
- You need **randomized, constrained stimulus** to hit corner cases a human wouldn't think to write by hand.
- You need **reusable** verification components across different projects/blocks (don't rewrite a monitor/scoreboard every time).
- You need a way to automatically **check** results (self-checking testbenches) instead of manually eyeballing waveforms for every test.
- Large verification teams need a **common structure** so engineers can understand and maintain each other's testbenches.

UVM solves all of this by providing standard base classes for stimulus generation, driving, monitoring, checking, and reporting — used industry-wide, so verification engineers moving between companies/projects can quickly understand any UVM-based testbench.

## 4.3 Basic UVM Testbench Architecture (Building Blocks)

```
                     ┌─────────────┐
                     │  Test       │  (configures and starts everything)
                     └──────┬──────┘
                            │
                     ┌──────▼──────┐
                     │ Environment │  (top-level container)
                     └──────┬──────┘
              ┌─────────────┼──────────────┐
        ┌─────▼─────┐ ┌─────▼─────┐  ┌─────▼──────┐
        │   Agent   │ │ Scoreboard│  │  (more     │
        │           │ │           │  │  agents…)  │
        └─────┬─────┘ └───────────┘  └────────────┘
       ┌───────┼────────┐
 ┌─────▼───┐ ┌─▼──────┐ ┌▼────────┐
 │Sequencer│ │ Driver │ │ Monitor │
 └─────────┘ └────┬───┘ └────┬────┘
                   │          │
              ┌────▼──────────▼────┐
              │        DUT          │
              └─────────────────────┘
```

### Key Components (What Each One Does)

| Component | Role |
|---|---|
| **Sequence** | Defines a series of stimulus transactions (e.g., "send 10 random read/write requests") — the "what to test" |
| **Sequencer** | Manages/arbitrates sequences and feeds transactions to the driver |
| **Driver** | Converts abstract transactions into actual pin-level signal wiggling on the DUT's interface |
| **Monitor** | Passively observes DUT interface signals, reconstructs transactions, and forwards them for checking (does not drive anything — read-only) |
| **Agent** | Bundles Sequencer + Driver + Monitor for one particular interface (e.g., one AXI port) |
| **Scoreboard** | Receives transactions from monitor(s), compares actual DUT behavior against expected/reference behavior, flags mismatches automatically |
| **Environment (env)** | Container that instantiates and connects agents, scoreboard, and other components |
| **Test** | Top-level class that builds the environment, configures it, and kicks off the sequences |

## 4.4 UVM Phases (Very Brief)

UVM testbenches execute through standardized **phases**, ensuring all components build/connect/run in a predictable order:

| Phase | Purpose (simplified) |
|---|---|
| `build_phase` | Construct/instantiate all components (like a constructor step) |
| `connect_phase` | Wire up TLM ports/connections between components (e.g., monitor → scoreboard) |
| `run_phase` | The actual test execution — stimulus generation, driving, checking (this is where "time" advances) |
| `report_phase` | Print final summary (pass/fail, coverage stats) at the end |

*(There are more phases in full UVM, but these four are enough to discuss confidently at intro level.)*

## 4.5 Practical Example (Conceptual, Simplified — Not Full Syntax)

Think of testing a simple adder DUT:
1. **Sequence** generates random pairs of numbers to add.
2. **Driver** drives those numbers onto the DUT's input pins, cycle by cycle.
3. **Monitor** watches the DUT's inputs and output, packaging them into a "transaction."
4. **Scoreboard** takes that transaction, independently computes `expected = a + b`, and compares against the DUT's actual output — flagging an error if they don't match.
5. **Test** class ties it all together and says "run 1000 such random additions."

## 4.6 Minimum Core Concepts You Must Know

- UVM = **class-based, reusable verification methodology** built on SystemVerilog — not a standalone tool.
- Core building blocks: **Sequence, Sequencer, Driver, Monitor, Agent, Scoreboard, Environment, Test.**
- Driver **writes** to the DUT; Monitor **only observes** (passive, never drives signals).
- Scoreboard = **automatic self-checking** (compares actual vs expected — no manual waveform eyeballing needed).
- UVM runs through structured **phases** (`build`, `connect`, `run`, `report`) so complex environments assemble predictably.
- Main benefit over simple testbenches: **reusability, randomization, and automatic checking** at scale.

## 4.7 Connection to RTL / Simulation / Synthesis / Verification

| Stage | Relationship |
|---|---|
| RTL | UVM's entire purpose is verifying RTL correctness, just with a far more structured, scalable methodology than a simple directed testbench |
| Simulation | UVM testbenches are still SystemVerilog code — compiled and run by the **same simulators** (ModelSim/VCS/Xcelium) discussed in Part 1 |
| Synthesis | UVM is verification-only — it's never synthesized; only the DUT (RTL) is synthesized, the UVM environment stays purely in simulation |
| Verification | UVM **is** the modern industry-standard verification methodology for any non-trivial ASIC/complex FPGA design — almost every verification job posting mentions UVM |

---

# 5. Clock Domain Crossing (CDC)

## 5.1 What It Is

**Clock Domain Crossing (CDC)** refers to any signal that travels from a flip-flop clocked by **one clock** into a flip-flop clocked by a **different, asynchronous (unrelated) clock**. Since the two clocks have no fixed phase relationship, standard STA setup/hold analysis (which assumes a known clock relationship) **does not apply** in the traditional sense — this is exactly why such paths are typically marked as **false paths** in STA (recall Part 2 of the STA notes).

## 5.2 Why It's Dangerous

Even though STA ignores these paths (correctly, from a setup/hold-timing-equation perspective), the signal still has to physically cross from one clock domain to another in real silicon — and this crossing has its **own** unique failure mode:

**Metastability.** If a signal changes right at the moment the receiving flip-flop's clock edge samples it (violating the flip-flop's setup/hold window relative to the *receiving* clock, which is entirely possible since the clocks are unrelated), the flip-flop's output can enter a **metastable state** — hovering at an invalid voltage level between 0 and 1 for an unpredictable amount of time before resolving to either 0 or 1 (randomly). This unpredictable, corrupted value can propagate into the rest of the design, causing **functional failure** that's extremely hard to reproduce/debug (since it's probabilistic, not deterministic).

> **Analogy:** Imagine trying to jump onto a moving carousel that's spinning at a completely different, unrelated speed from your own footsteps. If you time your jump right at the edge of a gap, you might land awkwardly half-on/half-off — unstable, and it takes a moment to know which side you'll end up settling on. That's metastability.

## 5.3 Why STA Alone Isn't Sufient — CDC Needs Separate, Structural Fixes

Since metastability is a **probabilistic, physical phenomenon** — not a deterministic timing/logic issue — no amount of STA analysis "fixes" it. Instead, CDC-crossing signals need special **synchronizer circuits** to reduce the *probability* of metastability propagating to a functionally negligible level (never truly zero, but astronomically low — measured via **MTBF: Mean Time Between Failures**).

## 5.4 Basic Ideas of Synchronization

### 5.4.1 Two-Flop (Double) Synchronizer — For Single-Bit Signals

```
                 clkB domain
async_signal ──►[FF1]──►[FF2]──► synchronized_signal
                  │D  Q   │D  Q
                 clkB    clkB
```

```verilog
always @(posedge clkB) begin
  ff1 <= async_signal;   // may go metastable — this is expected/allowed
  ff2 <= ff1;            // gives ff1 time to resolve before being used
end
```

- **FF1** may go metastable (that's expected and OK) — but it's given a **full clock cycle** to resolve before **FF2** samples it.
- **FF2's** output is (with extremely high probability) a clean, valid, stable logic value.
- This is the **most common single-bit CDC synchronizer** — often just called a "2-flop synchronizer" or "double-flop synchronizer."
- Sometimes a **3-flop synchronizer** is used for extra margin at very high frequencies / very tight MTBF requirements.

### 5.4.2 Why Two-Flop Sync Is NOT Enough for Multi-Bit Buses

If you naively 2-flop-synchronize **each bit** of a multi-bit bus independently, different bits can resolve metastability at different times / land on different randomly-resolved values in different clock cycles — causing the receiving domain to briefly see a **completely wrong, jumbled intermediate value** (not just one wrong bit — a nonsensical combination). This is a very common interview trick question.

**Fixes for multi-bit CDC:**
- **Gray code encoding** — for counters/pointers (e.g., FIFO read/write pointers) — ensures only **one bit changes at a time** between consecutive values, so even if synchronization catches the transition mid-change, only one bit is ambiguous, never a jumbled multi-bit value.
- **Handshake-based synchronization** (req/ack protocol) — for full data buses — the sending domain asserts a request, waits for an acknowledge from the receiving domain (both passed through 2-flop synchronizers) before considering the data safely transferred; this guarantees data is **held stable** long enough to be safely captured.
- **Asynchronous FIFO** — the most common real-world solution for crossing wide data buses between clock domains; internally uses Gray-coded pointers + 2-flop synchronizers for the pointer values, while the actual data is written/read via dual-port memory (no metastability risk on the data itself, since each side only ever accesses memory using its own local, already-safe pointer logic).

## 5.5 Minimum Core Concepts You Must Know

- CDC = signal crossing from one clock domain to an **unrelated/asynchronous** clock domain.
- The core danger is **metastability**, not a normal setup/hold violation.
- STA marks these paths as **false paths** (no fixed clock relationship to check against) — but that does **not** mean the crossing is safe; it just means STA setup/hold equations don't apply — separate CDC-specific verification and synchronizer circuits are required.
- **Single-bit signal fix:** 2-flop (or 3-flop) synchronizer.
- **Multi-bit signal fix:** Gray coding (for counters/pointers) or handshake protocols or asynchronous FIFOs (for data buses) — never just independently 2-flop-sync each bit of a bus.
- Metastability risk is measured via **MTBF (Mean Time Between Failures)** — never fully eliminated, only reduced to a negligible probability.

## 5.6 Connection to RTL / Simulation / Synthesis / Verification

| Stage | Relationship |
|---|---|
| RTL | Synchronizers (2-flop, async FIFO) are themselves written in RTL — a core RTL design skill |
| Simulation | Standard RTL simulation (ModelSim) generally **cannot model metastability realistically** (it's a physical analog effect) — dedicated **CDC verification tools** (e.g., Synopsys SpyGlass CDC, Cadence CDC) statically analyze the design to find unsynchronized crossings |
| Synthesis/STA | CDC paths are typically excluded from setup/hold STA via `set_false_path`, but must be separately verified for **correct synchronizer structure**, not just excluded and forgotten |
| Verification | CDC-specific static/formal verification is now a standard, separate sign-off step in any multi-clock-domain ASIC/FPGA design — a very commonly discussed interview topic |

---

# 6. Reset Synchronizers

## 6.1 What It Is

A **reset synchronizer** is a small circuit that takes an **asynchronous reset signal** (which can assert at any random time, not aligned to any clock edge) and ensures it is **released (de-asserted) synchronously** with respect to the local clock domain — this specific technique is called **"asynchronous assert, synchronous de-assert."**

## 6.2 Why Resets Need Synchronization

- **Reset assertion** should ideally happen **immediately**, regardless of clock state — e.g., a power-on reset or emergency reset should not have to "wait" for a clock edge — so reset assertion is intentionally kept **asynchronous** (fast, immediate).
- **Reset de-assertion (release)**, however, is dangerous if it happens asynchronously: if reset is released at a moment that violates the recovery/removal timing requirement of flip-flops in the design (very similar in spirit to setup/hold, but specifically for the reset pin), some flip-flops might come out of reset **one clock cycle later than others**, or even go **metastable** on release — causing different parts of the design to start in inconsistent states.

> **Analogy:** Asserting reset immediately is like slamming on the brakes the instant you see danger — no need to "wait for the right moment." But *releasing* the brakes must be done in a controlled, well-timed way — releasing them jerkily/randomly could cause the car (or in this case, different flip-flops in your design) to lurch forward at different, inconsistent times.

## 6.3 Recovery and Removal Time (The Reset Equivalent of Setup/Hold)

- **Recovery time:** Minimum time the asynchronous reset must be **de-asserted (stable)** before the next active clock edge, for the flip-flop to correctly come out of reset — this is the reset-pin equivalent of **setup time**.
- **Removal time:** Minimum time the reset must remain **stable after** an active clock edge — the reset-pin equivalent of **hold time**.

If reset releases at a "bad" moment relative to the clock edge (violating recovery/removal time), the flip-flop's reset-release behavior itself can become metastable.

## 6.4 Basic Reset Synchronizer Circuit

```
                clk domain
async_rst_n ──►[FF1]──►[FF2]──► sync_rst_n  (used inside the design)
        (async clear)  │D Q    │D Q
         │              clk     clk
         └──────────────┴───────┘
         (async reset input to BOTH flops directly — asserts immediately)
```

```verilog
module reset_sync (
    input  wire clk,
    input  wire async_rst_n,   // active-low asynchronous reset input
    output wire sync_rst_n     // synchronized reset output
);
    reg rff1, rff2;

    always @(posedge clk or negedge async_rst_n) begin
        if (!async_rst_n) begin
            rff1 <= 1'b0;   // ASYNC assertion — immediate, no clock wait
            rff2 <= 1'b0;
        end else begin
            rff1 <= 1'b1;   // SYNC de-assertion — takes 2 clock cycles to propagate 1
            rff2 <= rff1;
        end
    end

    assign sync_rst_n = rff2;
endmodule
```

- **Assertion:** The `negedge async_rst_n` in the sensitivity list means both flops clear **immediately**, asynchronously — no waiting for a clock edge.
- **De-assertion:** Once `async_rst_n` goes high again, it takes **two clock edges** for `sync_rst_n` to go high — giving FF1 a chance to resolve any metastability before FF2 (and hence the rest of the design) sees the reset release.

## 6.5 Why This Matters for Multi-Flop / Full-Chip Designs

Every flip-flop that uses `sync_rst_n` (instead of the raw asynchronous reset) will release from reset on the **exact same clock edge**, guaranteeing a clean, consistent, glitch-free start state across the entire design — critical for avoiding subtle bugs where some registers "wake up" one cycle before others.

## 6.6 Minimum Core Concepts You Must Know

- **Asynchronous assertion, synchronous de-assertion** is the standard safe reset practice.
- Assertion is immediate (doesn't wait for clock) — important for fast, guaranteed reset response.
- De-assertion is synchronized (via a small 2-flop synchronizer structure) so all logic exits reset cleanly, aligned to the clock, with no metastability risk on release.
- **Recovery time / Removal time** are the reset-pin equivalents of setup/hold time.
- Without a reset synchronizer, different flip-flops could exit reset on different cycles or metastably — a subtle, hard-to-debug class of bug.

## 6.7 Connection to RTL / Simulation / Synthesis / Verification

| Stage | Relationship |
|---|---|
| RTL | Reset synchronizer is itself a small, standard RTL block — every real design includes one per clock domain |
| Simulation | Simple RTL simulation can verify the **logical** assert/de-assert behavior, but (like CDC) cannot model the actual metastability physics — that risk is reduced by design/structure, not "proven safe" by simulation alone |
| Synthesis/STA | The 2-flop reset synchronizer itself needs correct recovery/removal timing verified by STA (`set_false_path` is sometimes wrongly applied here — recovery/removal checks on reset synchronizer flops should generally still be timed, not blanket-excluded) |
| Verification | CDC and reset-domain-crossing checks are usually combined into the same static "CDC/RDC (Reset Domain Crossing)" sign-off verification step in real projects |

---

# 7. Final Consolidated Interview Cheat-Sheet (All Six Topics)

## 7.1 Most Important Interview Talking Points

| Topic | One "confident answer" you should be able to give instantly |
|---|---|
| **Tcl** | "Tcl is the scripting language used to automate and control EDA tools — SDC constraint files, ModelSim `.do` files, and Quartus `.qsf` files are all Tcl syntax." |
| **ModelSim** | "ModelSim simulates RTL through compile → elaborate → simulate; I use `vlog`/`vcom`, `vsim`, `run`, and the waveform viewer to verify functional correctness before synthesis." |
| **Quartus** | "Quartus takes RTL through FPGA-specific synthesis, place & route (Fitter), STA (TimeQuest), and generates a `.sof`/`.pof` bitstream to program the FPGA." |
| **UVM** | "UVM is a class-based, reusable SystemVerilog verification methodology with standard components — driver, monitor, sequencer, scoreboard — enabling randomized stimulus and automatic self-checking at scale." |
| **CDC** | "CDC is a signal crossing between asynchronous clock domains; the danger is metastability, fixed via 2-flop synchronizers for single bits, and Gray coding / handshakes / async FIFOs for multi-bit buses." |
| **Reset Sync** | "Safe reset practice is asynchronous assertion (immediate) with synchronous de-assertion (via a 2-flop synchronizer), so the whole design exits reset cleanly on the same clock edge without metastability." |

## 7.2 Must-Remember Commands / Concepts Per Topic

**Tcl**
- `set`, `$var`, `[expr {...}]`, `if/elseif/else`, `for`, `foreach`, `proc`
- Remember: SDC and `.qsf`/`.do` files = Tcl syntax

**ModelSim**
- `vlib work`, `vlog file.v`, `vcom file.vhd`, `vsim work.tb_top`, `run 100ns` / `run -all`, `add wave -r /*`
- Flow: **Compile → Elaborate → Simulate**

**Quartus**
- Flow: **Synthesis → Fitter (Place & Route) → TimeQuest (STA) → Assembler → Programmer**
- Files: `.qsf` (settings/pin assignment), `.sdc` (timing constraints), `.sof` (volatile bitstream), `.pof` (flash bitstream)

**UVM**
- Building blocks: **Sequence → Sequencer → Driver → DUT; Monitor → Scoreboard**
- Phases: `build_phase`, `connect_phase`, `run_phase`, `report_phase`
- Driver = writes/drives; Monitor = passive/observes only

**CDC**
- Core danger: **Metastability**
- Fixes: **2-flop synchronizer** (single bit), **Gray code / handshake / async FIFO** (multi-bit)
- Metric: **MTBF (Mean Time Between Failures)**

**Reset Synchronizer**
- Rule: **Asynchronous assert, synchronous de-assert**
- Reset-pin timing equivalents: **Recovery time (≈setup)**, **Removal time (≈hold)**
- Structure: 2-flop synchronizer, async reset directly wired to both flops' clear input

---

**This completes the two-part FPGA/Verification tooling notes.** You now have interview-confident coverage of Tcl, ModelSim, Quartus, UVM basics, CDC, and Reset Synchronizers, all connected back to the RTL → Simulation → Synthesis → Verification flow.

If it would help, I can next put together: **(a)** a small set of realistic interview Q&A (rapid-fire style) across all six topics, **(b)** a simple end-to-end mini-project outline (write RTL → simulate in ModelSim → synthesize in Quartus) to practice the full flow hands-on, or **(c)** a deeper look at UVM sequences/scoreboards if you want to go one level beyond intro. Just let me know.
