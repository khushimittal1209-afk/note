# ASIC Design Flow — Complete Study Notes (RTL to Signoff)
## PART 1 of 3 — Specification, Architecture, RTL Design, Functional Verification, Synthesis, Netlist Generation, DFT Basics

---

# 0. The Big Picture — Full Flow at a Glance

```
Specification
      |
      v
Architecture Planning
      |
      v
RTL Design (Verilog/VHDL)
      |
      v
RTL Simulation / Functional Verification
      |
      v
Logic Synthesis  --------------------------> Netlist Generation
      |                                              |
      v                                              v
DFT Insertion (scan chains, BIST)  <------------------
      |
      v
Floorplanning  ---> Power Planning ---> Placement ---> Clock Tree Synthesis (CTS) ---> Routing
      |
      v
Parasitic Extraction (RC extraction)
      |
      v
Static Timing Analysis (STA)  <------> ECO (Engineering Change Order) [loop back if violations]
      |
      v
Signoff Checks: DRC, LVS, IR Drop, EM, Antenna, LEC (Logical Equivalence Checking)
      |
      v
GDSII Generation / Tape-out
      |
      v
Fabrication (Foundry) --> Packaging --> Testing --> Silicon
```

**Front-end** (this Part 1): Specification → Architecture → RTL → Functional Verification → Synthesis → Netlist → DFT.
**Back-end / Physical Design** (Part 2): Floorplanning → Power Planning → Placement → CTS → Routing → Parasitic Extraction.
**Signoff** (Part 3): STA → ECO → DRC/LVS → IR Drop/EM → GDSII/Tape-out.

This overall RTL-to-GDSII split (**front-end** deals with logical correctness and behavior; **back-end** deals with physical realization on silicon) is the most fundamental mental model in ASIC design, and interviewers frequently start by asking you to explain this split before drilling into specifics.

---

# 1. Specification

## 1.1 What the Stage Is

The **specification** is the formal, detailed document describing exactly what the chip (or IP block) must do: functional behavior, interfaces/protocols supported, performance targets (clock frequency, throughput, latency), power budget, area budget, process node/technology target, and operating conditions (voltage, temperature ranges — PVT corners).

## 1.2 Why It Is Needed

Every downstream decision — architecture, RTL micro-architecture, synthesis constraints, physical design floorplan sizing — traces back to requirements defined here. A wrong or incomplete specification is the most expensive class of mistake in chip design, because errors discovered late (post-tape-out) require an entirely new, extremely costly fabrication run ("re-spin"), whereas a specification error caught early costs only design time.

## 1.3 Inputs

- Market/product requirements (e.g., "must support DDR4-3200," "must run at 1.2 GHz in a 7nm process," "must fit in a 2W power envelope").
- Interoperability standards (protocol specs: AXI, PCIe, USB, Ethernet, etc.) the design must comply with.
- Target process technology node and foundry PDK (Process Design Kit) constraints.

## 1.4 Outputs

- **Functional specification document** (behavior, register maps, interface protocols, corner cases).
- **Micro-architecture-level performance/power/area (PPA) targets.**
- **Verification plan skeleton** — what needs to be proven correct, informing the eventual testbench/coverage plan.

## 1.5 What Problems Can Appear at This Stage

- **Ambiguous or incomplete requirements** — e.g., unclear reset behavior, undefined edge-case protocol behavior — that surface much later as functional bugs or verification gaps.
- **Unrealistic PPA targets** — a specified frequency/power/area combination that is not achievable in the target process node, discovered only after significant RTL/synthesis effort.
- **Specification drift** — requirements changing mid-project without full re-propagation through architecture/RTL/verification plans, causing silent inconsistencies.

## 1.6 How It Connects to the Next Stage

The specification directly shapes **architecture planning** (Section 2) — every architectural decision (pipeline depth, memory hierarchy, clocking scheme) is a response to specification requirements, particularly the PPA targets.

## 1.7 Practical Industry Usage

Real projects maintain specifications as living documents, often with formal **requirement tracking** (each requirement given an ID, traced through design and verification test plans to ensure closure) — especially critical in safety- or mission-critical chip development (automotive, aerospace) where requirement traceability may be an audit requirement.

## 1.8 Verification/Debugging Implications

The **verification plan** (what scenarios/coverage must be demonstrated) is derived directly from the specification — an incomplete specification produces an incomplete verification plan, which is a common root cause of bugs escaping to silicon (the bug was never something anyone thought to check, because the spec never called it out).

## 1.9 Timing Implications

The target clock frequency specified here becomes the fundamental input to every future timing constraint (Sections on Synthesis and STA, later) — it is the number that ultimately drives every `create_clock` and every synthesis timing-driven decision downstream.

## 1.10 Physical Design Implications

Area and power budgets specified here directly bound the floorplan size and power grid design decisions made much later (Part 2) — a chip specified with an unrealistically small area target relative to its functional requirements will manifest as a floorplanning/congestion crisis much further down the flow.

## 1.11 How to Explain It in an Interview

*"Specification is where we define what the chip must do and how well it must perform — functionality, frequency, power, and area targets — before any RTL is written. Getting this right early is critical because errors caught here are cheap to fix, while errors discovered after tape-out require an expensive re-spin."*

## 1.12 Interview Questions — Easy to Advanced

**Q (Easy):** What is the difference between a functional specification and a micro-architecture specification?
**A:** A functional spec describes *what* the design must do (behavior, interfaces, protocol compliance) without prescribing *how* it's implemented internally; a micro-architecture spec describes the internal implementation approach (pipeline stages, module partitioning) chosen to meet the functional spec's requirements within PPA targets.

**Q (Medium):** Why is a specification error more costly the later it's discovered?
**A:** Because every downstream stage (RTL, verification, synthesis, physical design) builds on assumptions from the spec — a late-discovered error can invalidate work across multiple stages, and if discovered post-tape-out, requires a full re-fabrication cycle costing significant time and money (potentially millions of dollars and months of schedule for a re-spin at advanced nodes).

**Q (Advanced):** How do requirement traceability practices help prevent verification escapes?
**A:** By explicitly linking every specification requirement to specific test cases/coverage points, traceability ensures no requirement is silently unverified — a gap analysis (requirement with no linked test) directly flags verification plan holes before tape-out, rather than relying on ad-hoc verification judgment.

---

# 2. Architecture Planning

## 2.1 What the Stage Is

Architecture planning translates specification requirements into a **high-level implementation approach**: pipeline structure, module partitioning, memory architecture, clocking/reset strategy, interconnect topology (buses/NoC), and power management scheme (power domains, clock/power gating strategy).

## 2.2 Why It Is Needed

The architecture determines whether the PPA targets from the specification are even **achievable** — this stage is where major, hard-to-reverse structural decisions are made (e.g., "do we need a 5-stage or 8-stage pipeline to hit this frequency," "do we need multiple power domains to hit this power budget").

## 2.3 Inputs

- The finalized (or near-final) specification.
- Process technology characteristics (achievable frequency/power at the target node, standard cell library characteristics).
- IP reuse considerations (existing verified blocks that can be integrated rather than designed from scratch).

## 2.4 Outputs

- **Block-level architecture diagram** — module partitioning and interconnect.
- **Clocking and reset architecture** — number of clock domains, PLL/clock generation scheme, reset distribution strategy.
- **Power architecture** — power domains, planned use of clock gating/power gating/multi-Vt cells for power reduction.
- **Memory architecture** — SRAM/register file sizing and placement strategy at a conceptual level.

## 2.5 What Problems Can Appear at This Stage

- **Pipeline depth vs. frequency mismatch** — insufficient pipelining chosen for the target frequency, discovered only during synthesis/STA much later, forcing a costly architectural rework.
- **Clock domain proliferation without a clear CDC (clock domain crossing) strategy** — leads to widespread, hard-to-fix CDC bugs discovered late in verification (Digital Circuits Section 18 concepts apply directly here at chip scale).
- **Underestimating interconnect/routing congestion** at the architecture stage — a module partition that looks clean logically can create physically difficult floorplans later.

## 2.6 How It Connects to the Next Stage

Architecture decisions become **RTL module boundaries and micro-architecture** (Section 3) — the architecture is the blueprint; RTL is its concrete implementation.

## 2.7 Practical Industry Usage

Real teams often build **performance/power models** (sometimes in C/C++/SystemC, or spreadsheet-based estimation) at this stage to validate architectural choices before committing to RTL — RTL development is expensive, so architectural mistakes are best caught via faster, higher-level modeling first.

## 2.8 Verification/Debugging Implications

Architecture decisions directly shape the **verification environment structure** — module boundaries typically map to verification unit-test boundaries, and clock domain architecture directly determines where CDC verification effort (Digital Circuits Section 17-18) must be concentrated.

## 2.9 Timing Implications

Pipeline depth is the single most direct architectural lever over achievable clock frequency — more pipeline stages generally allow shorter combinational paths per stage (Digital Circuits Section 24.4), enabling higher `f_max`, at the cost of latency and potentially added area/complexity.

## 2.10 Physical Design Implications

Module partitioning directly determines the eventual **floorplan hierarchy** (Part 2, Section on Floorplanning) — architecturally-tightly-coupled modules should generally be physically co-located; poor partitioning here creates avoidable routing congestion and timing closure difficulty at the back-end.

## 2.11 How to Explain It in an Interview

*"Architecture planning is where we decide how to structurally implement the specification — pipelining, clock domains, memory organization, and power domains — because these high-level decisions bound what's achievable later; you cannot fix an under-pipelined design at the synthesis stage without a major rework."*

## 2.12 Interview Questions

**Q (Easy):** Why is pipelining used in chip architecture?
**A:** To break long combinational logic paths into shorter segments across multiple clock cycles, allowing a higher clock frequency (Digital Circuits Section 24.4), at the cost of added latency (more cycles to produce a result).

**Q (Medium):** What's the risk of deciding on clock domains without a clear CDC strategy upfront?
**A:** Ad-hoc, unplanned clock domain crossings tend to proliferate as the design grows, and each crossing needs correct synchronization (2-FF synchronizers, async FIFOs) — without early planning, these get added reactively/inconsistently, leading to widespread CDC bugs that are notoriously hard to find (simulation-clean, silicon-only failures) and expensive to fix late.

**Q (Advanced):** How does architecture-level module partitioning affect physical design outcomes?
**A:** Module boundaries chosen at the architecture stage typically become floorplan partition boundaries; tightly-coupled logic split across far-apart modules can create long, congested routes and timing-critical paths crossing large physical distances, whereas partitioning aligned with actual data-flow/communication patterns tends to produce naturally more routable, timing-friendly floorplans.

---

# 3. RTL Design

## 3.1 What the Stage Is

RTL (Register Transfer Level) design is writing the actual synthesizable Verilog/VHDL/SystemVerilog code implementing the architecture — describing the design as register-to-register data flow and the combinational logic between them (Digital Circuits Section 21.2's core RTL mental model applies directly here at full-chip scale).

## 3.2 Why It Is Needed

RTL is the actual, concrete, machine-processable description of the design's behavior — everything before this stage (spec, architecture) is descriptive/planning; RTL is the first artifact that can actually be simulated, synthesized, and eventually turned into silicon.

## 3.3 Inputs

- The architecture specification (Section 2's outputs).
- Coding guidelines/style standards (company-specific lint rules, naming conventions, CDC-safe coding patterns).
- Reused/third-party IP RTL (memory compilers' generated views, standard IP blocks).

## 3.4 Outputs

- **Synthesizable RTL source files** (the design itself).
- **RTL-level documentation** (module interfaces, register maps, timing assumptions documented for downstream teams).
- **Lint reports** — static analysis results checking for common coding issues before simulation even begins.

## 3.5 What Problems Can Appear at This Stage

- **Latch inference** from incomplete `case`/`if` branches (Digital Circuits Section 7.5, 20.7) — one of the most common, most-checked-for RTL coding bugs.
- **Blocking/non-blocking assignment misuse** (Digital Circuits Section 22.2) causing simulation-synthesis mismatches.
- **Unsynchronized CDC paths** — signals crossing clock domains without proper synchronization (Digital Circuits Sections 17-18), often the single largest category of RTL bugs found via dedicated CDC-checking tools rather than simulation.
- **Non-synthesizable constructs** accidentally used in what's intended as synthesizable RTL (e.g., delays, certain system tasks).

## 3.6 How It Connects to the Next Stage

RTL is directly consumed by **RTL simulation** (Section 4) for functional verification, and later by **synthesis** (Section 5) for technology mapping — the same RTL source feeds both, which is precisely why simulation-synthesis consistency (writing clean, unambiguous, synthesizable RTL) matters so much.

## 3.7 Practical Industry Usage

Real teams enforce RTL coding guidelines via **automated linting** (tools like Synopsys SpyGlass, industry-standard lint rule sets) run continuously during development, catching latch inference, CDC issues, and style violations before they reach simulation/synthesis — lint is treated as a mandatory gate, not an optional nicety, in serious industry flows.

## 3.8 Verification/Debugging Implications

RTL bugs are cheapest to fix at this stage — this is why heavy investment in **early, fast feedback loops** (lint, quick smoke-test simulations) is standard practice; catching a bug in RTL review/lint costs minutes, catching the same bug post-synthesis costs hours, and catching it post-tape-out costs a re-spin.

## 3.9 Timing Implications

RTL structure (logic depth between registers, Digital Circuits Section 7.4) is the primary lever engineers have over achievable timing — this is set here, long before physical design; a poorly-pipelined RTL block cannot be timing-closed by physical design effort alone.

## 3.10 Physical Design Implications

RTL hierarchy (module boundaries) becomes the basis for synthesis hierarchy and, subsequently, floorplan partitioning (Part 2) — RTL written with physical-design awareness (e.g., keeping tightly-coupled logic within a module rather than spread across many small modules with excessive cross-module signaling) tends to produce more physically-implementable designs.

## 3.11 How to Explain It in an Interview

*"RTL design is writing the synthesizable Verilog/VHDL describing the architecture's behavior at a register-transfer level — every clock cycle, combinational logic computes new values from current register state, and registers capture them on the clock edge. Getting RTL coding discipline right — avoiding latches, correct blocking/non-blocking usage, proper CDC handling — is foundational, because these issues are cheap to fix here and very expensive to fix later."*

## 3.12 Interview Questions

**Q (Easy):** What's the difference between blocking and non-blocking assignment, and when should each be used?
**A:** Blocking (`=`) executes immediately in program order; non-blocking (`<=`) schedules updates to occur at the end of the simulation time step. Use non-blocking for sequential/clocked logic (`always @(posedge clk)`), blocking for combinational logic (`always @(*)`) — misuse causes race conditions and simulation-vs-synthesis mismatches (Digital Circuits Section 22.2).

**Q (Medium):** Why does an incomplete `case` statement infer a latch, and why is this usually a bug?
**A:** If not every possible input combination assigns a value to the output, the synthesis tool must "remember" the previous value for unhandled cases, inferring a level-sensitive latch to hold that state — this is almost always unintended in combinational logic and creates timing analysis complications and potential hazards (Digital Circuits Sections 7.5, 12).

**Q (Advanced):** How does poor RTL module partitioning affect physical design outcomes downstream, even if functionally correct?
**A:** Even functionally-correct RTL, if partitioned without physical-design awareness, can create excessive cross-module signal traffic that later manifests as floorplan congestion, long routes, and difficult timing closure — because module boundaries chosen in RTL typically propagate into synthesis and floorplan hierarchy; RTL and physical design are not fully independent concerns despite being handled by different teams/tools.

---

# 4. RTL Simulation / Functional Verification

## 4.1 What the Stage Is

Functional verification is the process of proving (to a practically-achievable confidence level, since exhaustive proof is generally infeasible for real designs) that the RTL correctly implements the specification, via simulation (and increasingly, formal verification techniques) against a comprehensive test plan.

## 4.2 Why It Is Needed

RTL is complex enough that manual review alone cannot reliably catch functional bugs — dedicated, systematic verification (testbenches, coverage, assertions) is the primary defense against functional bugs escaping to silicon, where they are enormously more expensive to fix.

## 4.3 Inputs

- The RTL design (Section 3's output).
- The verification plan (derived from the specification, Section 1.8).
- Verification IP (VIP) for standard protocols (e.g., AXI VIP, PCIe VIP) — pre-built, verified testbench components for standard interfaces.

## 4.4 Outputs

- **Simulation results / pass-fail logs.**
- **Coverage reports** (code coverage and functional coverage, Digital Circuits Section 25.4).
- **Bug reports/regression database** tracking found-and-fixed issues.
- A **"verified" RTL** signoff gate before proceeding to synthesis (in disciplined flows, RTL is not handed to synthesis/physical design until verification milestones are met).

## 4.5 What Problems Can Appear at This Stage

- **Functional bugs** of every kind: incorrect protocol handling, incorrect corner-case behavior, arithmetic errors, FSM bugs (Digital Circuits Section 20.7).
- **Incomplete coverage** — code/functional coverage gaps indicating untested scenarios, a leading indicator of latent bugs (Digital Circuits Section 25.4).
- **CDC bugs** invisible to standard functional simulation (Digital Circuits Sections 17-18) — requiring dedicated CDC verification tools, not caught by ordinary testbenches.
- **Testbench bugs** — the testbench itself can be wrong, leading to false confidence (a "passing" test that isn't actually checking what it claims to check).

## 4.6 How It Connects to the Next Stage

Verified RTL is handed to **synthesis** (Section 5) — critically, verification must continue *after* synthesis too (via gate-level simulation and equivalence checking, discussed in later sections), since synthesis and physical design can introduce their own classes of issues (Section 14.2's simulation-synthesis mismatch discussion from the Vivado notes applies conceptually here as well, generalized to ASIC).

## 4.7 Practical Industry Usage

Real verification environments use **UVM (Universal Verification Methodology)** — a standardized, layered SystemVerilog-based testbench architecture (drivers, monitors, scoreboards, sequences) — enabling reusable, scalable verification across large designs and teams. Formal verification (model checking, formal property verification) is increasingly used to complement simulation, particularly for control-logic-heavy blocks and CDC verification, where exhaustive proof is more tractable than for full datapath designs.

## 4.8 Verification/Debugging Implications

(This entire stage is the verification/debugging topic.) Key principle: **verification effort typically exceeds RTL design effort** in real projects (a very commonly cited industry ratio, sometimes 2-3x or more design engineering time) — this stage is not a "quick check" but a substantial, structured engineering discipline in its own right.

## 4.9 Timing Implications

Standard functional simulation uses **idealized, zero or unit-delay timing models** — it verifies logical correctness, not real silicon timing (this becomes relevant much later at STA, Part 3) — a common early-career misunderstanding is believing "simulation passing" says anything about real timing correctness; it does not.

## 4.10 Physical Design Implications

None directly at this stage, but a design with poor/incomplete verification coverage entering physical design carries hidden functional risk that physical design changes (buffer insertion, ECO fixes, Part 3) can sometimes inadvertently expose or interact with — reinforcing why solid verification closure before physical design handoff matters.

## 4.11 How to Explain It in an Interview

*"Functional verification proves the RTL correctly implements the specification through simulation against a structured test plan, using coverage metrics to measure completeness. It's typically the largest engineering effort in the whole flow, because functional bugs are dramatically cheaper to fix here than after synthesis or, worst case, after tape-out."*

## 4.12 Interview Questions

**Q (Easy):** What's the difference between code coverage and functional coverage?
**A:** Code coverage measures whether lines/branches/expressions in the RTL executed during simulation; functional coverage measures whether specific, meaningful scenarios (as defined by the verification engineer, e.g., every FSM state, boundary conditions) were actually exercised — 100% code coverage does not imply the design was tested under all functionally important conditions (Digital Circuits Section 25.4).

**Q (Medium):** Why can't standard RTL simulation catch CDC bugs?
**A:** Standard simulation typically uses zero-delay or simplified timing models and doesn't model the real, variable-phase relationship between independent clock domains — a CDC bug (missing synchronizer) can appear to "work" in simulation because the simulator's idealized clock edges never actually create the metastability-risk timing relationship that would occur in real silicon; dedicated CDC static-checking tools are needed instead (Digital Circuits Section 17.6).

**Q (Advanced):** Why might a design with high code coverage still have significant functional bugs escape to silicon?
**A:** Code coverage only confirms code was *executed*, not that it was exercised under all functionally meaningful input combinations/sequences/corner cases — a line can execute correctly under one stimulus but fail under a different, untested stimulus that never appeared in the test suite; functional coverage and rigorous corner-case/negative testing (not just "does the happy path work") are needed to close this gap, and even then, verification is fundamentally a confidence-building (not exhaustive-proof) activity for real designs, which is why formal verification is increasingly used to complement simulation for high-risk logic.

---

# 5. Synthesis

## 5.1 What the Stage Is

Synthesis transforms verified RTL into a **gate-level netlist** built from actual standard cells (from the target technology library) — logically equivalent to the RTL, but expressed in terms of real, fabricatable logic gates, flip-flops, and their interconnections.

## 5.2 Why It Is Needed

RTL is technology-independent (abstract Verilog/VHDL); silicon fabrication requires an actual, physical gate-level circuit built from the specific standard cells available in the target process technology — synthesis is the bridge between these two worlds.

## 5.3 Inputs

- **Synthesizable RTL** (Section 3's output, verified per Section 4).
- **Standard cell library** (`.lib`/Liberty timing/power models) for the target process node — describes every available gate (AND2, NAND3, DFF, etc.), its delay/power/area characteristics across PVT corners.
- **Design constraints (SDC — Synopsys Design Constraints)** — clock definitions, I/O timing, false/multicycle paths (directly analogous to XDC concepts from the Vivado FPGA notes, since XDC is itself SDC-based).

## 5.4 Outputs

- **Gate-level netlist** (structural Verilog, in terms of standard cells).
- **SDF (Standard Delay Format)** file — timing information for the synthesized netlist, usable for gate-level simulation.
- **Post-synthesis timing/area/power reports.**

## 5.5 Synthesis Sub-Steps (Conceptual — Same Spirit as FPGA Synthesis, Vivado Notes Section 3.2)

```
RTL parsing & elaboration
        |
        v
Technology-independent optimization (Boolean minimization, FSM extraction/re-encoding)
        |
        v
Technology mapping (map to actual standard cells: AND, OR, NAND, MUX, DFF from the target library)
        |
        v
Timing-driven optimization (gate sizing, buffer insertion, logic restructuring to meet SDC constraints)
        |
        v
Gate-level netlist output
```

**Key ASIC-specific difference from FPGA synthesis:** ASIC synthesis maps to **standard cells from a library with many size/drive-strength variants of the same logical gate** (e.g., dozens of different-strength inverters, NAND gates), and synthesis must choose the right variant (**gate sizing**) for each instance to meet timing — a much richer optimization space than FPGA's fixed LUT/FF fabric.

## 5.6 What Problems Can Appear at This Stage

- **Timing violations** — paths not meeting the SDC-specified frequency (setup violations) even after synthesis-stage optimization.
- **Area/power blowup** — synthesis choosing larger/faster cells extensively to meet aggressive timing, exceeding area or power budgets (a classic **timing vs. area vs. power** three-way trade-off).
- **Latch inference surviving into the netlist** (if not caught during RTL lint/simulation, Section 3.5) — now visible in synthesis reports/warnings as an actual synthesized latch cell.
- **Multi-driven nets, unconnected ports** — structural netlist issues flagged by synthesis-stage checks.

## 5.7 How It Connects to the Next Stage

The synthesized netlist proceeds to **DFT insertion** (Section 7) and then **physical design / floorplanning** (Part 2) — synthesis timing at this stage is still an **estimate** (wire-load models or, in modern flows, more sophisticated pre-placement parasitic estimation), not the final, signoff-accurate timing that only comes after real placement/routing (directly analogous to Vivado FPGA notes Sections 3.8/7.3 and 9.6).

## 5.8 Practical Industry Usage

Industry-standard synthesis tools include **Synopsys Design Compiler (DC)**, **Cadence Genus**, and increasingly **physically-aware synthesis** flows (e.g., Synopsys DC Topographical Mode, Cadence Genus with placement awareness) that use rough floorplan/placement information *during* synthesis to produce more physically-realistic timing estimates, reducing the synthesis-to-placement timing surprise gap.

## 5.9 Verification/Debugging Implications

**Logical Equivalence Checking (LEC)** — using tools like Synopsys Formality or Cadence Conformal — formally proves the synthesized gate-level netlist is functionally equivalent to the original RTL, without needing to re-run functional simulation on the netlist. This is standard, mandatory practice in real ASIC flows precisely because synthesis performs extensive optimization/restructuring, and a purely simulation-based re-verification of the netlist would be far too slow/incomplete to build confidence at full-chip scale.

## 5.10 Timing Implications

Post-synthesis timing reports use either **wire-load models** (statistical estimates of interconnect delay based on fanout/net size, used in older/simpler flows) or **placement-aware estimation** (modern flows) — in both cases, this remains an **estimate**; the same fundamental principle from the FPGA notes applies at ASIC scale, arguably even more strongly: **real, signoff-accurate timing only exists after real physical implementation and parasitic extraction** (Part 2/3).

## 5.11 Physical Design Implications

Gate sizing and buffering decisions made during synthesis directly affect the netlist's physical implementability — an over-aggressively-sized netlist (many large, high-drive-strength cells chosen purely to hit an ambitious timing target) can create area and routing congestion problems that only fully manifest during placement (Part 2).

## 5.12 How to Explain It in an Interview

*"Synthesis converts RTL into a gate-level netlist made of real standard cells from the target technology library, using the design's timing constraints (SDC) to drive gate selection and sizing decisions. It's the first stage where the design becomes 'real' in terms of actual, fabricatable logic — but timing at this stage is still an estimate, since real interconnect delay isn't known until after physical placement and routing."*

## 5.13 Interview Questions

**Q (Easy):** What is gate sizing, and why does it matter?
**A:** Gate sizing is selecting the appropriately-sized (drive-strength) variant of a logical gate from the standard cell library for each instance in the netlist — larger/stronger gates switch faster (helping timing) but consume more area and power; synthesis (and later, physical optimization) balances this trade-off per-instance based on timing criticality.

**Q (Medium):** Why is post-synthesis timing considered only an estimate, not signoff?
**A:** Because real interconnect (wire) delay depends on the actual physical placement and routing of cells, which isn't known until after physical design — synthesis uses either statistical wire-load models or rough placement-aware estimation, both of which can diverge meaningfully from actual post-route parasitics, especially at advanced process nodes where wire delay often dominates gate delay.

**Q (Advanced):** Why is Logical Equivalence Checking (LEC) used instead of re-simulating the gate-level netlist to verify synthesis correctness?
**A:** LEC formally proves logical equivalence between RTL and netlist across all possible input combinations using formal methods (structural/BDD/SAT-based comparison), which is exhaustive and fast compared to simulation, which can only check the specific stimulus patterns exercised by the testbench; for a full-chip-scale netlist, achieving equivalent confidence via simulation alone would require infeasibly large simulation effort, so LEC is the practical, industry-standard solution for this specific class of correctness question (does synthesis preserve RTL behavior), while simulation remains essential for other purposes (verifying RTL against spec in the first place).

---

# 6. Netlist Generation

## 6.1 What the Stage Is

While closely tied to synthesis (Section 5), "netlist generation" is worth treating as a distinct conceptual checkpoint: it's the finalized, structural gate-level description of the design — every cell instance, every net (wire) connecting them — that becomes the definitive input artifact for all subsequent physical design and signoff stages.

## 6.2 Why It Is Needed

The netlist is the **single source of truth for logical connectivity** from this point forward — physical design tools (Part 2) don't look back at RTL; they operate entirely on the netlist (plus constraints), which is why netlist correctness (verified via LEC, Section 5.9) is such a critical gate.

## 6.3 Inputs

Synthesis results (Section 5) — the netlist is essentially synthesis's primary output, formalized and prepared for handoff.

## 6.4 Outputs

- **Structural Verilog netlist** (or an equivalent format) — the definitive logical connectivity description.
- **Constraint files (SDC)** carried forward, often refined/annotated further for physical design (e.g., adding physical-only constraints not relevant at pure synthesis).

## 6.5 What Problems Can Appear at This Stage

- **Netlist-RTL functional mismatches** if LEC wasn't run or reveals a discrepancy (Section 5.9) — must be resolved before proceeding, as it indicates synthesis introduced an unintended functional change.
- **Missing or inconsistent constraints** carried from synthesis to physical design — a constraint present at synthesis but accidentally dropped before physical design handoff causes downstream tools to optimize/analyze incorrectly.

## 6.6 How It Connects to the Next Stage

The netlist (plus DFT-inserted scan structures, Section 7) becomes the direct input to **floorplanning** (Part 2) — this is the formal RTL-to-physical-design handoff point in the flow.

## 6.7 Practical Industry Usage

Real flows treat netlist handoff as a formal **checkpoint/gate** with a defined checklist (LEC clean, no unresolved synthesis warnings, constraints fully carried forward, DFT structures inserted and verified) before physical design teams accept the netlist — miscommunication or incomplete handoff at this exact boundary is a common real-world source of wasted physical design iteration.

## 6.8 Interview Questions

**Q (Easy):** Why is the netlist, not the RTL, used as the input to physical design tools?
**A:** Physical design tools work with actual standard cells and their physical/electrical characteristics — the netlist provides exactly this (technology-mapped gates and their connectivity), whereas RTL is technology-independent and doesn't directly describe fabricatable circuit elements.

**Q (Medium):** What could go wrong if constraints aren't correctly carried forward from synthesis to physical design?
**A:** Physical design tools (placement, routing, CTS) use timing constraints to guide every optimization decision — missing or inconsistent constraints mean the physical design tool might optimize the wrong paths, under-prioritize genuinely critical timing paths, or fail to correctly exclude false/multicycle paths from analysis, producing misleading intermediate results and wasted iteration.

---

# 7. DFT Basics (Design for Testability)

## 7.1 What the Stage Is

**DFT (Design for Test)** inserts additional structures into the netlist specifically to enable **manufacturing test** — the process of testing every physically fabricated chip (not the design, but each individual manufactured unit) for fabrication defects, since silicon manufacturing is never 100% defect-free.

## 7.2 Why It Is Needed

Without DFT structures, testing whether an individual fabricated chip is defect-free would require controlling and observing every internal flip-flop directly through primary I/O pins — practically impossible for any design with more than a handful of internal registers, since the number of possible internal states vastly exceeds what limited external pins could exhaustively exercise and observe in reasonable test time.

## 7.3 Core DFT Technique: Scan Chain Insertion

DFT tools replace ("stitch") ordinary flip-flops with **scan flip-flops** — functionally identical D flip-flops with an added **scan mode**, in which they're connected together into a long shift register (**scan chain**) during test mode, allowing external test equipment to:
1. **Shift in** any arbitrary internal state via the scan chain (serial shift-in, like a giant shift register, Digital Circuits Section 14.2).
2. **Capture** one clock cycle of normal functional operation.
3. **Shift out** the resulting state for comparison against expected results.

This converts the otherwise-infeasible "control and observe every internal register" problem into a manageable serial shift-based test.

```
Normal functional D flip-flop:
     D --->[FF]---> Q

Scan flip-flop (added scan input + scan enable mux):
     D  ---\
            (MUX)---> [FF] ---> Q
     SI ---/    ^
                SE (scan enable: 0=functional, 1=scan/test mode)
```

## 7.4 Other Common DFT Structures

| Structure | Purpose |
|---|---|
| **Scan chains** | As above — the fundamental DFT technique for sequential logic testability |
| **BIST (Built-In Self-Test)**, esp. **MBIST** for memories | Dedicated on-chip test logic that autonomously tests memory arrays (BRAMs/SRAMs) for manufacturing defects, since scan-chain-based testing is inefficient for large regular memory arrays |
| **Boundary Scan (JTAG / IEEE 1149.1)** | Standardized test access structure at chip I/O boundaries, enabling board-level interconnect testing and, commonly, access to internal scan chains/debug features via a standard JTAG interface |
| **ATPG (Automatic Test Pattern Generation)** | Not a DFT structure itself, but the tool-driven process of generating the actual test vectors (patterns to shift in/capture/shift out) that exercise scan chains to detect specific fault models (e.g., stuck-at faults) |

## 7.5 Inputs

- The synthesized, functionally-verified netlist (Sections 5-6).
- **DFT architecture plan** — number of scan chains, chain length targets, test clock/test mode pin planning (usually decided during architecture planning, Section 2, since DFT has its own area/routing/pin-count implications that need early consideration).

## 7.6 Outputs

- **DFT-inserted (scan-stitched) netlist.**
- **Test patterns** (from ATPG) used later at the manufacturing test stage (post-fabrication, not part of the design flow itself, but the direct deliverable this stage exists to enable).
- **Fault coverage report** — the percentage of modeled possible manufacturing defects (typically **stuck-at fault** and increasingly **transition/delay fault** models) that the generated test patterns can actually detect.

## 7.7 What Problems Can Appear at This Stage

- **Low fault coverage** — insufficient scan chain coverage or poorly-testable logic structures (e.g., certain clock-gating or asynchronous logic that's inherently harder to control/observe via scan) leaving some fraction of potential manufacturing defects undetectable.
- **Scan chain timing/routing overhead** — added scan routing and multiplexers introduce area overhead and can affect timing (the scan mux adds a small delay to the functional data path, Digital Circuits Section 4.6's general fan-in/gate-delay principle applies) — must be accounted for in STA (Part 3).
- **DFT-functional interaction bugs** — e.g., a design with internal asynchronous logic or unusual clocking that doesn't play well with standard scan methodology, requiring special DFT handling.

## 7.8 How It Connects to the Next Stage

The DFT-inserted netlist proceeds to **physical design** (Part 2) exactly like an ordinary netlist — scan flip-flops are placed and routed just like regular flip-flops, with scan chain connectivity typically optimized during placement (scan chain reordering to minimize routing, a physical-design-aware DFT optimization).

## 7.9 Practical Industry Usage

Virtually **all** modern commercial ASICs include DFT — it is not optional for any chip expecting reasonable manufacturing yield/test economics. Tools like **Synopsys DFTMAX/TetraMAX** and **Cadence Modus** are industry-standard for scan insertion and ATPG.

## 7.10 Verification/Debugging Implications

DFT structures themselves must be verified (does scan mode correctly shift/capture without disturbing functional behavior when scan enable is de-asserted?) — a **separate verification concern** from functional verification (Section 4), often requiring dedicated DFT-specific testbenches/simulation.

## 7.11 Timing Implications

The scan mux added to every scan flip-flop's data input adds a small but real delay to the functional path — this must be included in STA (Part 3) timing analysis; forgetting to account for scan-related delay is a common, checkable mistake.

## 7.12 Physical Design Implications

Scan chain **reordering** (choosing which scan flip-flop connects to which in the shift chain) is often performed physically-aware during/after placement specifically to minimize the routing overhead/length of scan chain interconnect, since the *logical* order of flip-flops in a scan chain (which one shifts to which) doesn't need to match any functional relationship — this is a physical-design-driven optimization freedom unique to DFT structures.

## 7.13 How to Explain It in an Interview

*"DFT inserts scan chains and other test structures into the netlist so that each individually fabricated chip can be tested for manufacturing defects after fabrication — since silicon fabrication is never perfectly defect-free, we need an efficient way to control and observe internal state through limited I/O pins, and scan chains solve this by turning internal flip-flops into a giant shiftable register during test mode."*

## 7.14 Interview Questions

**Q (Easy):** What problem does DFT/scan insertion solve?
**A:** It solves the problem of testing fabricated chips for manufacturing defects when internal flip-flops aren't directly accessible from external pins — scan chains let external test equipment shift arbitrary state into and out of internal registers serially.

**Q (Medium):** Why is MBIST used for memories instead of ordinary scan-chain-based testing?
**A:** Memory arrays are large, regular structures with defect patterns and testing needs (e.g., detecting specific memory-cell-level fault types like stuck bits, coupling faults) that are inefficiently tested via generic scan-and-observe; MBIST embeds dedicated, autonomous test logic specifically designed for the regular structure and fault models of memory arrays, testing them much faster and more thoroughly than generic scan-chain approaches would.

**Q (Advanced):** Why is scan chain order (which flip-flop connects to which in the shift chain) typically decided based on physical placement rather than logical/functional grouping?
**A:** The order in which flip-flops are visited in a scan shift chain has no functional significance (it only matters during test mode, not normal operation) — so physical design tools are free to reorder the chain purely to minimize routing distance/congestion between physically-adjacent scan flip-flops, an optimization freedom that wouldn't exist if chain order were functionally constrained; this typically happens as a physically-aware DFT step during or shortly after placement.

---

*(End of Part 1 — Specification, Architecture Planning, RTL Design, RTL Simulation/Functional Verification, Synthesis, Netlist Generation, DFT Basics. Part 2 will cover the Physical Design / Back-End stages: Floorplanning, Power Planning, Placement, Clock Tree Synthesis (CTS), Routing, and Parasitic Extraction. Reply "continue" or "part 2" to proceed.)*
