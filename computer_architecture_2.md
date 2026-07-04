# Computer Architecture — Deep Study Notes
## Part 2 of 4: Pipelining, Pipeline Hazards, Hazard Detection & Forwarding, Branch Handling & Prediction

**Primary references used for this part:**
- Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*, 5th Ed. — Appendix C (Pipelining: Basic and Intermediate Concepts), Sections C.1–C.2.
- Furber, *ARM System-on-Chip Architecture* — Chapter 4 (ARM Organization and Implementation: 3-stage and 5-stage pipeline organizations).
- Supplementary framing in the style of Patterson & Hennessy's *Computer Organization and Design* and Neso Academy's COA lecture sequence.

**Series roadmap:**
- Part 1: Architecture fundamentals, performance metrics, ISA, CPU datapath, control unit, assembly-level view. *(done)*
- **Part 2 (this file):** Pipelining → Pipeline hazards → Data/Structural/Control hazards → Hazard detection & forwarding → Branch handling & prediction.
- Part 3: Cache memory, mapping techniques, replacement policies, write policies, locality, memory hierarchy.
- Part 4: Main memory & secondary storage, virtual memory, memory bandwidth/latency, multiprocessing basics.

---

## 1. Pipelining

### 1.1 What It Is
Pipelining is an **implementation technique** — invisible to the programmer — in which multiple instructions are overlapped in execution, exploiting the fact that the steps needed to execute *any* instruction (fetch, decode, execute, memory access, write back) can be performed in parallel with the corresponding steps of *neighboring* instructions, as long as they use different hardware resources at any given moment. H&P's own analogy: **a pipeline is like an automobile assembly line** — many stations, each doing a different part of the job, all working on different cars simultaneously.

### 1.2 How It Works — The Classic 5-Stage RISC Pipeline
Building directly on Part 1's MU0/datapath discussion, the textbook's default unpipelined RISC implementation takes at most 5 clock cycles per instruction:

| Stage | Name | What happens |
|---|---|---|
| 1 | **IF** — Instruction Fetch | Send PC to memory, fetch instruction, compute PC+4 |
| 2 | **ID** — Instruction Decode / Register Fetch | Decode opcode, read register operands, sign-extend immediate, compute branch target (in parallel, via **fixed-field decoding**, since register-specifier fields sit at the same bit positions across instruction types in a RISC ISA) |
| 3 | **EX** — Execute / Effective Address | ALU computes either the effective memory address (loads/stores), the ALU result (register-register op), or an ALU-immediate result |
| 4 | **MEM** — Memory Access | Load reads memory, or store writes memory |
| 5 | **WB** — Write-Back | Result (from ALU or memory) written into the register file |

Pipelining this design costs "almost no changes" — you simply **start a new instruction every clock cycle**, so at any given moment five different instructions occupy the five stages simultaneously:

```
                       Clock cycle number
Instr.       1      2      3      4      5      6      7      8      9
i           IF     ID     EX    MEM     WB
i+1                IF     ID     EX    MEM     WB
i+2                       IF     ID     EX    MEM     WB
i+3                              IF     ID     EX    MEM     WB
i+4                                     IF     ID     EX    MEM     WB
```
If an instruction is started every cycle, performance can approach **5× that of the unpipelined implementation** — the number of pipeline stages.

### 1.3 Why It's Important
Pipelining is, in the textbook's own words, **"the key implementation technique used to make fast CPUs"** today. It attacks exactly one of the three terms in the CPU-time equation from Part 1 (`IC × CPI × cycle time`) — depending on your baseline, pipelining is viewed either as **reducing CPI** (starting from a multi-cycle-per-instruction design) or as **reducing clock cycle time** (starting from a single-long-cycle design). The textbook's primary framing throughout is the CPI view.

### 1.4 Ideal Speedup Formula
If pipeline stages are perfectly balanced (equal work per stage) and there is no pipelining overhead:
```
Time per instruction (pipelined) = Time per instruction (unpipelined) / Number of pipe stages
```
so ideal speedup = number of pipeline stages = **pipeline depth**. More generally, with stalls:
```
Speedup = Pipeline depth / (1 + Pipeline stall cycles per instruction)
```
and, since the ideal pipelined CPI is 1:
```
CPI(pipelined) = 1 + Pipeline stall cycles per instruction
```

### 1.5 Numerical / Solved Problem — Pipelining Overhead
If an unpipelined instruction takes 4.4 ns and pipelining introduces 0.2 ns of overhead per stage on top of a 1 ns "useful work" slowest stage, the pipelined cycle time is `1 + 0.2 = 1.2 ns`, giving:
```
Speedup = 4.4 / 1.2 = 3.7×    (not the "ideal" 5× a naive depth-based estimate might suggest)
```
**This is a direct, concrete instance of Amdahl's Law from Part 1**: fixed per-stage overhead is a portion of time that can never be reduced by adding more pipeline depth, so it caps the achievable speedup.

### 1.6 Diagrams
```
UNPIPELINED (each instruction fully completes before the next starts):
 Instr i:   [IF][ID][EX][MEM][WB]
 Instr i+1:                          [IF][ID][EX][MEM][WB]
 -> 5 cycles per instruction, one at a time

PIPELINED (new instruction starts every cycle):
 Instr i:   [IF][ID][EX][MEM][WB]
 Instr i+1:     [IF][ID][EX][MEM][WB]
 Instr i+2:         [IF][ID][EX][MEM][WB]
 -> throughput approaches 1 instruction completed per cycle
```

### 1.7 Connections to Other Topics
- Pipelining is precisely what turns the single MU0-style datapath from Part 1 into a *shared, time-multiplexed* structure — the same ALU, register file, and memory port are now used by five different instructions in five different pipeline stages during any one clock cycle.
- Pipelining is what makes hazards *possible* in the first place — an unpipelined machine, by construction, never has two instructions "in flight" simultaneously, so RAW/structural/control conflicts as pipeline hazards simply don't arise; the rest of Part 2 is entirely about the problems pipelining itself creates.
- The ideal-CPI-of-1 target directly continues the CPI discussion from Part 1's performance equation.

### 1.8 How It Appears in Modern Processors
Every mainstream CPU core today is pipelined — often far deeper than 5 stages (e.g., historically, the Pentium 4 had a 20+ stage pipeline to chase very high clock rates, at the cost of a much higher branch misprediction penalty; see Section 5). ARM's own product line is a clean real-world case study of pipeline depth as a design lever: **early ARM cores (ARM6/ARM7) used a simple 3-stage pipeline** (Fetch, Decode, Execute), while **later cores (e.g., ARM9TDMI) moved to a 5-stage pipeline** (Fetch, Decode, Execute, Buffer/Data, Write-back) specifically to raise clock frequency and reduce CPI, at the direct cost of needing more forwarding hardware (Section 4).

### 1.9 Common Pitfalls
- Assuming pipelining always yields exactly "N× speedup" for an N-stage pipeline — real designs never achieve this due to stage-imbalance and stalls (Sections 2–5).
- Forgetting that pipelining is about **throughput**, not **latency** — any *individual* instruction still takes the same number of cycles (or more, with stalls) to fully complete; what improves is how often a *new* instruction can start/finish.
- Believing deeper pipelines are strictly better — deeper pipelines raise clock frequency but also increase the *penalty* (in cycles) for any hazard, especially control hazards (Section 5) — a core architectural trade-off.

### 1.10 Key Terms & Definitions
- **Pipe stage / pipe segment:** one step of instruction processing, overlapped with other instructions' stages.
- **Pipeline depth:** the number of pipe stages.
- **Throughput:** instructions completed per unit time (what pipelining improves).
- **Latency:** time for one instruction to fully complete (unchanged or worsened by pipelining).
- **Fixed-field decoding:** decoding registers/immediates in parallel because their bit positions are fixed across instruction types — a RISC ISA property (Part 1, Section 3) that *enables* clean pipelining.

---

## 2. Pipeline Hazards (Overview)

### 2.1 What It Is
A **hazard** is any situation that prevents the next instruction in the stream from executing in its designated clock cycle, forcing the pipeline to **stall**. H&P define exactly three classes:

| Hazard type | Cause |
|---|---|
| **Structural** | Hardware resource conflict — two instructions in the pipeline simultaneously need the same physical resource |
| **Data** | An instruction needs a result that a previous, still-in-flight instruction hasn't produced yet |
| **Control** | A branch or other PC-changing instruction hasn't yet resolved its outcome/target, so the fetch stage doesn't know what to fetch next |

### 2.2 How It Works — The Stall Rule
When any instruction stalls, **every later instruction (behind it, less far along) also stalls**, while **every earlier instruction (ahead of it, further along) continues** — otherwise the hazard could never clear. No new instructions are fetched during the stall. A stall cycle is often visualized as a **"bubble"** — it occupies a pipeline slot but carries no useful work, floating through the pipeline exactly like an inserted no-op.

### 2.3 Why It's Important
Hazards are the entire reason pipelined CPI is `1 + stall cycles per instruction` rather than a flat 1 — **all real pipeline performance analysis is hazard analysis.** Every hazard subtype has its own detection logic, penalty, and (partial) mitigation technique, covered in Sections 3–6 below.

### 2.4 Where It's Used / Appears
Every pipelined processor, from a simple 5-stage RISC core to a modern deeply-pipelined out-of-order superscalar CPU, must solve all three hazard classes — the mitigation techniques scale in sophistication (simple stalling → forwarding → dynamic scheduling/out-of-order execution → branch prediction with speculative execution) but the underlying three-way taxonomy never changes.

### 2.5 Diagrams
```
Generic stall behavior:
Instr i     : [IF][ID][EX][MEM][WB]
Instr i+1   :     [IF][ID][stall][EX][MEM][WB]      <- stalled, held back
Instr i+2   :         [IF][stall][ID][EX][MEM][WB]  <- also stalled
                            ^
                     "bubble" injected here — no useful work this cycle
```

### 2.6 Connections to Other Topics
- Structural hazards connect directly back to the datapath resource design from Part 1 (e.g., a single shared memory port for both instructions and data, exactly like MU0's memory-limited design principle).
- Data hazards connect to the assembly-level register dependency patterns discussed in Part 1, Section 6.
- Control hazards connect to the ISA's branch/control-flow instruction design from Part 1, Section 3.

### 2.7 Common Pitfalls
- Treating "hazard" and "bug" as the same thing — a hazard is a *timing/performance* problem the hardware (or compiler) must correctly handle; if handled correctly, the program's result is unaffected, only its speed.
- Forgetting that hazards are fundamentally created by pipelining itself — they don't exist in a purely sequential, unpipelined execution model.

### 2.8 Key Terms & Definitions
- **Hazard:** any condition that would cause incorrect execution if the pipeline proceeded normally, requiring a stall or other correction.
- **Stall / bubble:** an inserted "do-nothing" cycle used to delay an instruction until a hazard clears.
- **Pipeline interlock:** the hardware that detects a hazard and enforces the necessary stall.

---

## 3. Structural Hazards

### 3.1 What It Is
A structural hazard arises when **the hardware cannot support the combination of instructions currently in the pipeline simultaneously**, because some functional unit is not fully pipelined or some resource hasn't been duplicated enough (e.g., only one register-file write port, but the pipeline momentarily wants two writes in the same cycle).

### 3.2 How It Works — The Classic Example: Shared Memory Port
If a design uses **one single memory** for both instructions and data (rather than separate instruction/data memories or caches), then whenever an instruction accesses data memory, it directly **conflicts with a later instruction's need to fetch from memory** in that same cycle:

```
Time (clock cycles):        CC1   CC2   CC3   CC4   CC5   CC6   CC7   CC8
Load               :        IF    ID    EX    MEM   WB
Instr i+1          :              IF    ID    EX    MEM   WB
Instr i+2          :                    IF    ID    EX    MEM   WB
Instr i+3          :                          IF <-- wants to fetch, but
                                               Load's MEM stage is using
                                               the shared memory port here!
```
The fix: **stall the pipeline for one cycle** when the conflict occurs — the fetch of instruction i+3 is delayed exactly one cycle (a bubble), while instructions i+1 and i+2 (already further along) continue normally.

### 3.3 Why It's Important
Structural hazards are fundamentally a **cost vs. performance trade-off** decision made by the *designer*, not an unavoidable consequence of the ISA or algorithm. A designer *could* eliminate this particular hazard entirely by fully duplicating the memory system (separate instruction and data caches) — but that costs more chip area, more memory bandwidth, and more pins. If the hazard is rare in practice, it may simply not be worth the extra hardware cost to avoid it.

### 3.4 Numerical / Solved Problem
**Setup:** data references are 40% of the instruction mix; ideal pipelined CPI (ignoring the structural hazard) is 1; the hazard-affected design can run its clock 1.05× faster than the hazard-free design (because it saves the cost/complexity of a second memory port).
```
Average instruction time (hazard-free) = 1 × Clock_ideal
Average instruction time (with hazard) = (1 + 0.40 × 1) × (Clock_ideal / 1.05)
                                        = 1.40 × Clock_ideal / 1.05
                                        = 1.33 × Clock_ideal
```
**Conclusion:** the hazard-free design is roughly **1.3× faster** — despite its slower clock, avoiding the structural hazard wins overall. (This is the exact textbook worked example.)

### 3.5 Diagrams
```
Structural hazard from a single memory port:
Load        : IF  ID  EX  MEM  WB
Instr i+1   :     IF  ID  EX   MEM  WB
Instr i+2   :         IF  ID  EX    MEM  WB
Instr i+3   :             stall IF  ID    EX   MEM  WB
                            ^
                  fetch delayed one cycle — Load's MEM access
                  "steals" the memory port this cycle
```

### 3.6 Connections to Other Topics
- Structural hazards are resolved architecturally by exactly the memory-hierarchy techniques covered in Part 3: **split instruction/data caches (Harvard-style access at the cache level)** eliminate this specific hazard, at the cost of the extra hardware discussed above.
- The general principle — "duplicate a resource, or accept a stall" — reappears identically when reasoning about register-file write ports, multi-cycle functional units (e.g., a non-pipelined floating-point multiplier), and any other shared hardware resource.

### 3.7 How It Appears in Modern Processors
Virtually every modern CPU uses **separate L1 instruction and data caches** specifically to avoid this exact structural hazard (this is precisely the "split cache" solution the textbook references, and is directly why ARM's 5-stage-and-deeper cores, discussed in Section 1.8, moved to separate instruction/data memories — the textbook calls this the **"memory bottleneck"**, closely related to the classic **von Neumann bottleneck**). Non-pipelined, multi-cycle functional units (e.g., an unpipelined divider) remain a real, deliberately-accepted structural hazard source in many modern designs, since fully pipelining every single functional unit (especially FP multiply/divide) is often not worth the transistor cost.

### 3.8 Common Pitfalls
- Assuming structural hazards are always "bad design" — sometimes accepting a rare structural hazard is the *correct* engineering trade-off (Section 3.4's numerical example notwithstanding — the specific numbers matter).
- Forgetting that structural hazards, unlike data hazards, are entirely eliminable by throwing hardware (duplication/full pipelining) at the problem — the only real question is whether it's worth the cost.

### 3.9 Key Terms & Definitions
- **Structural hazard:** a resource conflict caused by insufficient hardware duplication/pipelining.
- **Split cache / Harvard-style cache access:** separate instruction and data caches, the standard fix for the shared-memory structural hazard.
- **Von Neumann / memory bottleneck:** the fundamental performance limit imposed by a single shared instruction-and-data memory path.

---

## 4. Data Hazards

### 4.1 What It Is
A data hazard occurs when **pipelining changes the relative order of register reads/writes** compared to sequential (unpipelined) execution, so a later instruction could read a stale/wrong value of a register that an earlier, still-in-flight instruction hasn't yet written back.

### 4.2 How It Works — The Classic RAW Example
```asm
DADD  R1, R2, R3      ; R1 := R2 + R3
DSUB  R4, R1, R5      ; uses R1 -- needs DADD's result
AND   R6, R1, R7      ; uses R1 -- needs DADD's result
OR    R8, R1, R9      ; uses R1 -- needs DADD's result
XOR   R10, R1, R11    ; uses R1 -- needs DADD's result
```
`DADD` doesn't write R1 into the register file until its **WB** stage (cycle 5), but `DSUB` reads registers during **its** ID stage (cycle 3) — two full cycles too early. Without any correction, `DSUB` and `AND` would silently read the *old*, wrong value of R1. (The textbook flags an important subtlety here: the "wrong" value read isn't even deterministic in general — e.g., if an interrupt occurred between DADD and DSUB, DADD's WB might complete and change what DSUB reads — clearly unacceptable behavior that hardware must prevent.)

This specific pattern is called a **RAW (Read-After-Write) hazard** — the most common and important data hazard type (WAR and WAW hazards also exist in more complex, out-of-order pipelines, but do not occur in this simple in-order 5-stage pipeline since reads always happen before writes in program order there).

### 4.3 Why It's Important
Data hazards are the single most frequent hazard type in real code (register dependencies between nearby instructions are extremely common — almost every instruction consumes a value some nearby instruction just produced) — this makes their mitigation technique, **forwarding** (Section 6), one of the most important pieces of hardware in any real pipelined CPU.

### 4.4 Numerical / Solved Problem — Where Exactly Is the Conflict?
Continuing the example above, mapped onto pipeline cycles:
```
              CC1  CC2  CC3  CC4  CC5  CC6
DADD R1,..    IF   ID   EX   MEM  WB
DSUB ..,R1    -    IF   ID   EX   MEM  WB
AND  ..,R1    -    -    IF   ID   EX   MEM
OR   ..,R1    -    -    -    IF   ID   EX
XOR  ..,R1    -    -    -    -    IF   ID
```
- DADD writes R1 in **WB = cycle 5**.
- DSUB reads R1 in **ID = cycle 3** → needs value **2 cycles early**.
- AND reads R1 in **ID = cycle 4** → needs value **1 cycle early**.
- OR reads R1 in **ID = cycle 5** → same cycle as the write; **fine if register reads happen in the second half of the cycle and writes in the first half** (a standard convention that resolves this specific case "for free," without any forwarding hardware at all).
- XOR reads R1 in **ID = cycle 6** → strictly after the write; **no hazard at all.**

### 4.5 Diagrams
```
DADD writes R1 →  ...ID  EX  MEM  [WB]
                                    │  (write happens here, cycle 5)
DSUB needs R1  →  IF [ID] EX  MEM  WB
                       │ (read happens here, cycle 3 — TWO cycles too early)
AND  needs R1  →      IF [ID] EX  MEM WB
                            │ (read here, cycle 4 — ONE cycle too early)
OR   needs R1  →          IF [ID] EX  MEM WB
                                │ (read here, cycle 5 — same cycle as write;
                                   resolved by read-2nd-half/write-1st-half convention)
```

### 4.6 Connections to Other Topics
- Data hazards are resolved primarily by **forwarding/bypassing** (Section 6) and, where forwarding is physically impossible (the load-use case), by a **pipeline interlock stall** (also Section 6).
- Register dependency chains that create data hazards are exactly the assembly-level register-usage patterns discussed in Part 1, Section 6.9 — reading real compiled assembly for hazard patterns is a genuinely useful debugging/analysis skill.
- Compilers actively **schedule instructions to minimize data hazards** — e.g., reordering independent instructions between a load and its first use — directly connecting back to ISA/compiler co-design themes from Part 1.

### 4.7 How It Appears in Modern Processors
All modern pipelined CPUs implement extensive forwarding networks (Section 6). Deeper, superscalar, out-of-order cores go much further: **register renaming** eliminates false (WAR/WAW) dependencies entirely, and **dynamic scheduling** (Tomasulo's algorithm, scoreboarding — beyond this appendix-level scope but built directly on top of it) lets independent instructions execute out of order specifically to hide data hazard latency.

### 4.8 Common Pitfalls
- Assuming all data hazards can be fixed by forwarding — the load-use hazard (Section 6.4) fundamentally cannot be, because it would require "forwarding backward in time."
- Forgetting that data hazards are a *property of the dynamic instruction stream*, not just the static code — the same code can have different hazard behavior depending on which instructions are actually adjacent in the pipeline at any moment (relevant once branches/speculation are involved).

### 4.9 Key Terms & Definitions
- **RAW (Read-After-Write) hazard:** an instruction needs a value before a preceding instruction has written it — the only data hazard type possible in a simple in-order pipeline.
- **WAR / WAW hazards:** Write-After-Read / Write-After-Write hazards — possible only when instructions can complete out of program order (relevant to advanced dynamically-scheduled pipelines, not the basic 5-stage design).

---

## 5. Control Hazards

### 5.1 What It Is
A control hazard arises from pipelining **branches and other PC-changing instructions**: until a branch is resolved (its condition evaluated and its target computed), the fetch stage doesn't know for certain which instruction to fetch next.

### 5.2 How It Works — The Baseline Cost
In the classic 5-stage pipeline, a branch's condition and target aren't known until the end of its **ID** stage. The simplest possible handling: **redo the fetch once the branch is decoded** — the instruction fetched immediately after the branch is simply discarded (a wasted "stall" cycle that does no useful work):
```
Branch instruction  : IF   ID   EX   MEM  WB
Branch successor    :      IF   IF   ID   EX   MEM  WB
                                ^ refetched once target is known
```
The textbook notes plainly: **"One stall cycle for every branch will yield a performance loss of 10% to 30% depending on the branch frequency"** — control hazards, unmitigated, are typically *more* costly to overall performance than data hazards, precisely because branches are so frequent (SPEC branch frequencies range roughly 3%–24% of instructions) and the default penalty applies to every single one.

### 5.3 Why It's Important
Because branch frequency is high and the naive penalty is large, control-hazard mitigation (Section 7: branch prediction) is one of the most heavily engineered parts of any modern high-performance CPU — often the single biggest differentiator in real-world integer-code performance between two otherwise similar microarchitectures.

### 5.4 The Four Classic Static (Compile-Time) Schemes
| Scheme | Idea | Penalty behavior |
|---|---|---|
| **Flush/freeze pipeline** | Stall fetch until branch resolves | Fixed penalty every branch, taken or not |
| **Predicted untaken** | Keep fetching sequentially; if actually taken, squash the fetched instruction and refetch at target | Zero penalty if correctly untaken; penalty only when actually taken |
| **Predicted taken** | Fetch from the (once known) target immediately | Only helps if the target address is known *before* the outcome — not useful in the basic 5-stage pipeline, but valuable in deeper pipelines with implicit condition codes |
| **Delayed branch** | Compiler places a useful, always-executed instruction in the "branch delay slot" right after the branch | Effectively hides one cycle of penalty *if* the compiler can find a valid instruction to fill the slot |

**Delayed branch, illustrated (single delay slot, MIPS-style):**
```
branch instruction
sequential successor      <- the "delay slot" instruction; ALWAYS executes,
                              whether or not the branch is taken
branch target (if taken)  <- OR fall-through (if not taken)
```
Compilers fill the delay slot three ways, in order of preference: **(a) from before the branch** (an independent instruction moved forward — the best option, wastes nothing); **(b) from the branch target** (useful when the branch is highly likely taken, e.g., loop branches — the target instruction must be duplicated since it may still be reached another way); **(c) from the fall-through path** (useful when the branch is likely untaken) — legal only if executing the moved instruction causes no harm when the branch goes the "wrong" way.

### 5.5 Numerical / Solved Problem — Branch Penalty and Effective CPI
For a deeper pipeline (e.g., MIPS R4000-style) where the branch target isn't known for 3 stages, penalties differ by scheme and by branch outcome:

| Scheme | Penalty: unconditional | Penalty: untaken | Penalty: taken |
|---|---|---|---|
| Flush pipeline | 2 | 3 | 3 |
| Predicted taken | 2 | 3 | 2 |
| Predicted untaken | 2 | 0 | 3 |

Given frequencies: unconditional = 4%, untaken conditional = 6%, taken conditional = 10% (total branches = 20%), the **added CPI from branches** for each scheme:
```
Stall-pipeline:      0.04(2) + 0.06(3) + 0.10(3) = 0.08+0.18+0.30 = 0.56
Predicted taken:     0.04(2) + 0.06(3) + 0.10(2) = 0.08+0.18+0.20 = 0.46
Predicted untaken:   0.04(2) + 0.06(0) + 0.10(3) = 0.08+0.00+0.30 = 0.38
```
**Conclusion:** with a base CPI of 1, the predicted-untaken scheme gives an ideal pipeline **1.56/1.38 ≈ 1.13× faster** than naive stall-pipeline, and a fully-ideal (zero branch penalty) pipeline would be **1.56× faster** than stall-pipeline — a direct, quantified illustration of why control-hazard mitigation matters so much.

The general formula tying this back to Section 1.4's speedup equation:
```
Pipeline speedup = Pipeline depth / (1 + Branch frequency × Branch penalty)
```

### 5.6 Diagrams
```
Predicted-untaken scheme:
  Untaken (correct):  IF ID EX MEM WB   (fetches continue normally, zero penalty)
                         [i+1] IF ID EX MEM WB

  Taken (misprediction):  IF ID EX MEM WB
                             [wrongly fetched i+1]  IF -> idle -> idle -> idle -> idle (squashed)
                             [branch target]              IF   ID   EX   MEM  WB
```

### 5.7 Connections to Other Topics
- Control hazards are the direct motivation for **branch prediction** (Section 7), the far more powerful and universal modern solution compared to the static compile-time schemes above.
- The branch-target and branch-condition computation happening "early" (during ID) is only possible because of the RISC ISA properties from Part 1 (fixed instruction format, PC-relative addressing, simple register-vs-zero or register-vs-register comparisons) — a CISC ISA with more complex condition codes and addressing typically pushes this resolution later, increasing the effective penalty.
- Deeper pipelines (Section 1.8's ARM 5-stage example, or historically the Pentium 4's very deep pipeline) directly increase control-hazard *penalty* (more cycles before resolution) even as they may improve raw clock speed — a fundamental architecture trade-off.

### 5.8 How It Appears in Modern Processors
Static schemes (flush, predicted-untaken, delayed branch) were standard in early RISC designs (classic MIPS, early SPARC) but have been almost entirely superseded in high-performance cores by **dynamic branch prediction** (Section 7), because static schemes cannot adapt to a branch's actual runtime behavior. Delayed branches, in particular, have fallen out of favor in modern deep, wide, out-of-order designs because they complicate exception handling and interact poorly with variable-latency, deeply-pipelined implementations of the *same* ISA across generations — a concern the ARM designers explicitly flagged as a reason to *avoid* delayed branches from the start (Part 1's ARM material notes this directly: avoiding delayed branches "simplifies re-implementing the architecture with a different pipeline").

### 5.9 Common Pitfalls
- Assuming "predicted taken" is always better than "predicted untaken" — in the basic 5-stage pipeline they have identical worst-case penalty, since the target isn't known any earlier than the outcome; predicted-taken only helps in pipelines where the target is resolved before the condition.
- Forgetting that unconditional branches/jumps still cost a penalty in the simplest schemes — only prediction (or delay slots) can hide it.
- Misreading a delayed-branch code sequence — the delay-slot instruction executes **unconditionally**, regardless of whether the branch itself is taken.

### 5.10 Key Terms & Definitions
- **Control hazard:** a pipeline hazard caused by not yet knowing a branch's outcome/target.
- **Branch penalty:** the number of stall/squashed cycles incurred per branch under a given handling scheme.
- **Delay slot:** the instruction position immediately after a branch that always executes, used by delayed-branch schemes.
- **Taken / untaken (fall-through) branch:** whether the branch changes the PC to its target or simply continues sequentially.

---

## 6. Hazard Detection and Forwarding

### 6.1 What It Is
**Forwarding (a.k.a. bypassing or short-circuiting)** is the hardware technique that resolves most data hazards *without* stalling, by routing a result directly from the pipeline stage that produced it to the pipeline stage that needs it — **before** that result has been written back to the register file through the normal path.

### 6.2 How It Works — Forwarding Paths
**Key insight:** the DADD/DSUB example from Section 4 doesn't actually need R1's value written into the register file for DSUB to use it correctly — it just needs the *value*, available as soon as DADD's ALU produces it (end of DADD's EX stage), routed directly into DSUB's ALU input (start of DSUB's EX stage):
```asm
DADD R1, R2, R3
DSUB R4, R1, R5     ; forwarded directly from DADD's EX/MEM pipeline register
AND  R6, R1, R7     ; forwarded directly from DADD's MEM/WB pipeline register
OR   R8, R1, R9     ; resolved "for free" via read-2nd-half/write-1st-half register file convention
```
Forwarding generalizes beyond ALU-to-ALU: it also connects, e.g., a load's memory output directly to a subsequent store's data input (`DADD R1,R2,R3 / LD R4,0(R1) / SD R4,12(R1)` — here the ALU output is forwarded to the load/store address calculation, and the loaded value is forwarded from the memory-read output to the memory-write input).

### 6.3 Why It's Important
Forwarding is what allows a pipelined CPU to achieve near-ideal CPI in the extremely common case of back-to-back dependent ALU instructions — **without it, nearly every real program would stall constantly**, since register dependencies between adjacent instructions are the norm, not the exception, in compiled code.

### 6.4 The One Case Forwarding Cannot Fix — the Load-Use Hazard
```asm
LD   R1, 0(R2)
DSUB R4, R1, R5     ; needs R1 -- but LD doesn't have it until MEM (cycle 4)!
AND  R6, R1, R7     ; can be forwarded (needs R1 at EX = cycle 4... let's check)
OR   R8, R1, R9     ; resolved via register file, no forwarding needed
```
The textbook is blunt about why this specific case is unfixable by forwarding alone: **"the value used by the DSUB instruction... forwarding backward in time... a capability not yet available to computer designers!"** The load doesn't produce its result until the *end* of its MEM stage (cycle 4), but DSUB needs that value at the *start* of its EX stage (also cycle 4) — the data simply isn't ready yet, no matter how fast the forwarding wire is. AND, one instruction further back, *can* be forwarded to (its EX stage is one cycle later than DSUB's), and OR needs no forwarding at all (resolved through the register file).

**The fix: a pipeline interlock** — dedicated hazard-detection hardware that recognizes this specific pattern (a load immediately followed by a dependent instruction) and **inserts exactly one stall cycle**, after which forwarding resolves the rest normally:
```
LD   R1,0(R2)   IF  ID  EX  MEM  WB
DSUB R4,R1,R5       IF  ID  stall EX  MEM  WB
AND  R6,R1,R7           IF  stall ID  EX   MEM  WB
OR   R8,R1,R9               stall IF  ID    EX   MEM  WB
```

### 6.5 Numerical / Solved Problem
If load-use hazards occur in 20% of instructions and each costs exactly 1 stall cycle, added CPI from this source alone = `0.20 × 1 = 0.20`. Combined with a base ideal CPI of 1, `CPI = 1.20` from this hazard type alone — directly pluggable into the Section 1.4 speedup formula.

### 6.6 Diagrams
```
Forwarding paths (conceptual):
   EX/MEM pipeline reg ──┐
   MEM/WB pipeline reg ──┼──► ALU input muxes (feed EX stage of dependent instr)
   Register file (2nd-half read / 1st-half write) ──┘

Load-use hazard (cannot be forwarded — "needs data from the future"):
   LD:    ...  [EX][MEM]───────┐  (data ready only at END of MEM)
   DSUB:      [ID][EX]◄────────┘  (data NEEDED at START of EX — impossible without a stall)
```

### 6.7 Connections to Other Topics
- Forwarding hardware complexity scales directly with pipeline depth — this is exactly why ARM's move from a 3-stage to a 5-stage pipeline (Part 2, Section 1.8) required substantially more forwarding paths: with execution spread across three stages (Execute, Buffer/Data, Write-back) rather than one, results must be forwarded from any of three intermediate result registers to any of three source-operand reads. The ARM9TDMI's own documentation confirms this exact load-use hazard is unavoidable even with full forwarding — "the processor cannot avoid a one-cycle stall" for a load immediately followed by a dependent instruction, precisely matching the textbook's MIPS analysis.
- The load-use hazard is a strong, concrete argument for **compiler instruction scheduling** (placing independent, useful work between a load and its first use) — a real, practical technique compilers use specifically to hide this exact hazard type.

### 6.8 How It Appears in Modern Processors
All modern pipelined CPUs implement extensive forwarding/bypass networks as standard, foundational hardware. The load-use hazard specifically remains a real, named hazard in modern pipeline documentation and compiler-scheduling guidance (exactly as shown for both the textbook's MIPS pipeline and the real ARM9TDMI). Deeper modern pipelines with longer cache-access latency for loads (see Part 3) can have a **multi-cycle** load-use penalty rather than just one cycle, making compiler/hardware scheduling around loads even more valuable.

### 6.9 Common Pitfalls
- Assuming more forwarding paths always eliminate all stalls — the load-use case is a hard physical limit (data not yet produced), not a hardware-design oversight.
- Forgetting that forwarding requires **hazard detection logic** to actively recognize the dependency (matching destination and source register numbers across pipeline stages) and correctly select which forwarding path (if any) to mux in — this detection logic is itself a nontrivial combinational block, closely related in spirit to the priority-encoder/comparator logic covered in digital-design fundamentals.
- Confusing "forwarding" (routing a value early) with "stalling" (waiting for a value) — real pipelines use *both*, forwarding wherever timing allows and stalling only in the residual cases (like load-use) where forwarding is physically impossible.

### 6.10 Key Terms & Definitions
- **Forwarding / bypassing / short-circuiting:** routing a result directly from its producing pipeline stage to a consuming pipeline stage, ahead of the normal register-file write-back.
- **Pipeline interlock:** hazard-detection hardware that stalls the pipeline exactly as long as needed when forwarding alone cannot resolve a hazard.
- **Load-use hazard:** the specific data hazard where an instruction immediately following a load needs that load's result before it is available, even with full forwarding.

---

## 7. Branch Handling and Branch Prediction Basics

### 7.1 What It Is
Branch prediction is the far more powerful, adaptive alternative to the static compile-time schemes of Section 5: instead of a fixed compile-time policy, the hardware (or profile-informed compiler) **guesses** a branch's outcome, and only pays the misprediction penalty when the guess is wrong.

### 7.2 Static Branch Prediction (Profile-Based)
Uses **profiling information from earlier program runs** to bias the compile-time prediction per branch, exploiting the empirical observation that **individual branches are often strongly, bimodally biased toward taken or untaken.** Measured on SPEC89: floating-point programs average **9% misprediction**, integer programs average **15% misprediction** — worse for integer code, which the textbook notes is a real limitation of static schemes, since integer programs both mispredict more *and* branch more frequently.

### 7.3 Dynamic Branch Prediction — the Branch-Prediction Buffer
The simplest dynamic scheme: a small memory (the **branch-prediction buffer / branch history table**), indexed by the low-order bits of the branch's own address, storing a bit that records whether that branch was **recently taken or not**. Critically, **this buffer has no tags** — it's "effectively a cache where every access is a hit": if the index happens to collide with a different branch, the prediction is simply a "hint" that might be wrong, harmless to correctness, only affecting how often the hint helps.

### 7.4 The 1-Bit Predictor's Weakness, and the 2-Bit Fix
A single prediction bit has an annoying failure mode: **even a branch that is almost always taken will still mispredict twice in a row** whenever it happens to be untaken once (loop exit), because the single bit flips immediately. The standard fix is a **2-bit saturating counter** per entry — a branch must be wrong *twice in a row* before the prediction actually flips:

```
                Taken
      ┌──────────────────────┐
      │                      ▼
  ┌────────┐  Not taken  ┌────────┐
  │  11    │◄────────────│  10    │
  │Predict │             │Predict │
  │ taken  │────────────►│ taken  │
  └────────┘   Taken      └────────┘
      ▲ Taken                 │ Not taken
      │                       ▼
  ┌────────┐  Not taken  ┌────────┐
  │  01    │────────────►│  00    │
  │Predict │◄────────────│Predict │
  │untaken │   Taken      │untaken│
  └────────┘             └────────┘
```
This is a **direct, real-world FSM** — precisely the Moore/Mealy state-machine material from digital design, applied to a genuine, ubiquitous piece of CPU hardware. States 11/10 both predict *taken*; states 01/00 both predict *untaken*; a single wrong outcome only moves one step toward the opposite prediction, while two wrong outcomes in a row are required to actually flip the prediction. (More generally, an **n-bit saturating counter** predicts taken whenever its value is ≥ half its maximum, but studies show 2-bit predictors capture nearly all the benefit, so nearly all real systems use exactly 2 bits per entry.)

### 7.5 Numerical / Solved Problem — 2-Bit Predictor Accuracy
For SPEC89 with a 4096-entry 2-bit prediction buffer, measured misprediction rates: integer benchmarks average **≈11%**, floating-point benchmarks average **≈4%** — meaningfully better than the profile-based static scheme's 15%/9% averages, illustrating why dynamic prediction became the industry standard.

**CPI contribution example:** if branch frequency is 20% and misprediction rate is 11% with a 3-cycle misprediction penalty:
```
Added CPI from branches = Branch frequency × Misprediction rate × Penalty
                         = 0.20 × 0.11 × 3 ≈ 0.066
```
— compare this to the ~0.38–0.56 added CPI from the naive static schemes in Section 5.5: dynamic prediction, when it works well, is dramatically cheaper.

### 7.6 Diagrams
```
Branch-prediction buffer access (conceptual):

  branch instruction address
            │
            ▼
   ┌─────────────────────┐
   │  low-order address   │──► index ──► ┌───────────────┐
   │        bits           │             │ 2-bit counter  │──► prediction (taken/untaken)
   └─────────────────────┘             │  entries table  │
                                         └───────────────┘
                                                │
                                    actual outcome (after resolution)
                                                │
                                                ▼
                                  update counter (FSM transition, Section 7.4)
```

### 7.7 Connections to Other Topics
- The 2-bit predictor is literally an FSM whose states, transitions, and "output" (predicted direction) map directly onto the Moore-machine coding style from digital-design fundamentals — a strong cross-course connection worth explicitly drawing in an interview or exam answer.
- Branch prediction accuracy interacts directly with pipeline depth (Section 1.8/5.7): the deeper the pipeline, the larger the misprediction penalty, so deeper pipelines demand *more* accurate prediction just to break even — a core reason very deep pipelines (e.g., historical Pentium 4) invested heavily in sophisticated branch predictors.
- Branch behavior (frequency, predictability) is inseparable from ISA-level control-flow instruction design (Part 1, Section 3.6) — e.g., how conditions are expressed (condition codes vs. register-vs-zero comparisons) affects how early a branch can be resolved and thus how large the "natural" penalty is before any prediction is even applied.

### 7.8 How It Appears in Modern Processors
Modern high-performance cores use vastly more sophisticated dynamic predictors than the simple 2-bit counter — **correlating/global-history predictors, tournament predictors (blending multiple predictor types), and dedicated branch-target buffers (BTBs)** that predict *both* direction and target address, often combined with a separate **return-address stack** for accurate call/return prediction. All of these are direct engineering descendants of the basic branch-prediction-buffer/2-bit-counter idea covered here — the fundamental FSM concept scales up in scope (more history bits, more table entries, multiple cooperating predictors) but not in kind.

### 7.9 Common Pitfalls
- Assuming a "tagless" prediction buffer is unsafe — it's not, because a wrong prediction (from an aliasing collision) is only ever a *performance* hint; a misprediction is always caught and corrected once the actual branch resolves, exactly like any other misprediction.
- Forgetting that static (profile-based) and dynamic (hardware buffer) prediction are not mutually exclusive — many real compiler/hardware combinations use both, with static hints informing the initial/cold-start prediction before the dynamic predictor "warms up" with real runtime history.
- Confusing branch *direction* prediction (taken/untaken) with branch *target* prediction (where the branch actually goes) — modern hardware predicts both, but the classic 2-bit counter covered here handles only direction.

### 7.10 Key Terms & Definitions
- **Branch-prediction buffer / branch history table:** a small, tagless memory indexed by branch address, storing prediction state.
- **2-bit saturating counter:** a 4-state FSM predictor requiring two consecutive wrong outcomes to flip its prediction.
- **Misprediction penalty:** the number of cycles wasted (squashed instructions) when a prediction turns out wrong.
- **Profile-based (static) prediction:** compile-time prediction informed by prior program-run measurements.
- **Branch-target buffer (BTB):** modern hardware that predicts not just direction but the actual target address of a branch.

---

## Bridge to Part 3
Part 2 showed how pipelining creates hazards, and how forwarding, stalling, and prediction keep CPI close to its ideal value of 1 despite them. Notice how often **memory** kept appearing as the real bottleneck underneath these hazards — the structural hazard from a shared memory port, the multi-cycle load-use hazard, even the "memory bottleneck" driving ARM's move to a 5-stage pipeline with separate instruction/data memories. Part 3 goes directly to the source of that bottleneck: **why memory is slow relative to the CPU, and how cache memory, locality of reference, and the memory hierarchy are engineered to hide that gap.**

*(End of Part 2. Say "next" for Part 3: Cache Memory, Cache Mapping Techniques, Cache Replacement Policies, Write Policies, Locality of Reference, and Memory Hierarchy.)*
