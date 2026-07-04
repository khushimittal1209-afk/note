# Computer Architecture — Deep Study Notes
## Part 1 of 4: Fundamentals, Performance, ISA, Datapath & Control Unit

**Primary references used for this part:**
- Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*, 5th Ed. — Chapter 1 (Fundamentals of Quantitative Design and Analysis) and Appendix A (Instruction Set Principles).
- Furber, *ARM System-on-Chip Architecture* — Chapter 1 (An Introduction to Processor Design, featuring the MU0 teaching processor) and Chapter 2 (The ARM Programmer's Model).
- Supplementary framing in the style of Patterson & Hennessy's *Computer Organization and Design* and Neso Academy's COA lecture sequence, for beginner-friendly build-up.

**Series roadmap:**
- **Part 1 (this file):** Architecture fundamentals → Performance metrics → ISA → CPU datapath → Control unit → Assembly-level view.
- **Part 2:** Pipelining, all hazard types, forwarding/hazard detection, branch handling & prediction.
- **Part 3:** Cache memory, mapping techniques, replacement policies, write policies, locality, memory hierarchy.
- **Part 4:** Main memory & secondary storage, virtual memory, memory bandwidth/latency, multiprocessing basics.

---

## 1. Computer Architecture Fundamentals

### 1.1 What It Is
Computer architecture is the discipline of designing the interface and internal organization of a computer system so that it meets functional, performance, cost, power, and reliability goals simultaneously. Hennessy & Patterson frame this as **"defining computer architecture"** in the broad sense — not just the instruction set, but the full set of design decisions across the hardware/software boundary: instruction set architecture, organization (microarchitecture), and hardware implementation.

Three distinct but related layers, and it's essential to keep them separate in your head:

| Layer | What it answers | Example |
|---|---|---|
| **Instruction Set Architecture (ISA)** | What operations can software *ask* the hardware to do? | "there is an ADD instruction that takes two registers" |
| **Microarchitecture / Organization** | *How* does the hardware implement the ISA internally? | pipelined vs. non-pipelined, cache sizes, number of ALUs |
| **Implementation (hardware realization)** | The actual logic design, circuit design, physical layout | transistor sizing, clock distribution, floorplanning |

The same ISA (e.g., ARM, x86, RISC-V) can have wildly different organizations and implementations across generations and vendors — this is precisely why "architecture" and "implementation" must be kept conceptually distinct.

### 1.2 How It Works — Classes of Computers and Design Forces
Modern architecture design is driven by the target application class, because different classes optimize for different things:
- **Personal Mobile Devices (PMDs):** cost and energy-constrained, code size matters (less memory = cheaper, lower energy).
- **Desktop computing:** balance of price-performance, integer and floating-point performance matter, code size largely ignored.
- **Servers:** throughput, availability, and scalability dominate; floating point is less critical than integer/string performance, but still present.
- **Warehouse-scale / cloud computing:** cost-per-operation at massive scale, energy efficiency, and fault tolerance across thousands of nodes.
- **Embedded systems:** real-time constraints, extreme cost/power sensitivity.

Underlying every design decision are a handful of recurring **quantitative principles** (deep-dived in Section 2 below): take advantage of parallelism, exploit the principle of locality, and focus design effort on the common case.

### 1.3 Why It's Important
Every subsequent topic in this series (performance analysis, ISA design, pipelining, caching, memory hierarchy, multiprocessing) is really just architecture responding to one of two forces: **technology trends** (transistors get cheaper and more numerous, but wires and memory don't get proportionally faster) and **application demand** (software wants ever more performance, parallelism, and lower energy per operation). Understanding architecture as "an engineering response to these two forces" is what separates rote memorization from real understanding.

### 1.4 Where It's Used
Literally every digital computing device: smartphones (ARM-based SoCs), laptops/servers (x86/ARM/RISC-V), embedded microcontrollers, GPUs, and warehouse-scale datacenters. The same core vocabulary (ISA, pipeline, cache, hazard, hierarchy) applies at every scale, just with different constraints dominating.

### 1.5 Common Pitfalls
- Treating "architecture" and "implementation" as the same thing (e.g., assuming two chips with the same ISA must perform identically).
- Ignoring that architectural choices are always trade-offs, not universal improvements — a design that's excellent for a PMD (small code, low power) may be poor for a server (throughput-focused).
- Forgetting that software (compilers, OS) co-designs with hardware; ISA decisions are made *with* compiler writers in mind (see Appendix A crosscutting issues, Section 3 below).

### 1.6 Key Terms & Definitions
- **ISA (Instruction Set Architecture):** the programmer/compiler-visible contract between hardware and software.
- **Microarchitecture / Organization:** the internal design used to implement an ISA (pipeline depth, cache hierarchy, execution units).
- **Implementation:** the actual logic/circuit/physical realization.
- **PMD, WSC:** Personal Mobile Device; Warehouse-Scale Computer — the two extremes Hennessy & Patterson emphasize as driving modern design.

---

## 2. Performance and Performance Metrics

### 2.1 What It Is, Beginner Level
"Faster" means different things to different people. H&P's own framing: a desktop user cares about **response time** (a.k.a. execution time / latency) — time between start and completion of one task. A server operator cares about **throughput** — total work completed per unit time (e.g., transactions/hour). These are related but *not* the same metric, and a change that improves one can sometimes hurt the other (e.g., adding more overlapped requests can improve throughput while individual response time gets slightly worse due to contention).

**"X is n times faster than Y"** is formally defined via execution time:
```
Execution_time(Y) / Execution_time(X) = n
```
which, since performance is the reciprocal of execution time, is equivalent to `Performance(X)/Performance(Y) = n`.

### 2.2 CPU Time — the Core Formula
The textbook decomposes CPU time into three orthogonal factors:

```
CPU time = Instruction Count (IC) × Cycles Per Instruction (CPI) × Clock Cycle Time
```
equivalently
```
CPU time = (CPU clock cycles for a program) / (Clock rate)
```

Each factor is influenced by different parts of the system:
| Factor | Influenced by |
|---|---|
| Clock cycle time | Hardware technology, circuit/physical implementation |
| CPI | Organization (microarchitecture) and ISA |
| Instruction count (IC) | ISA and compiler technology |

**Crucial exam-style insight:** a 10% improvement in *any one* of the three factors gives a 10% improvement in CPU time — but the factors are **not independent** in practice; e.g., an ISA change that reduces instruction count might increase average CPI.

When instruction mix matters, CPI can be broken into per-instruction-class contributions:
```
CPU clock cycles = Σ (IC_i × CPI_i)     for each instruction class i
CPI = Σ (IC_i / Instruction Count) × CPI_i
```
This "weighted CPI" form is what H&P call the **processor performance equation**, and it is the standard tool for comparing two microarchitectural design alternatives when you can measure per-instruction-class frequencies and CPIs (e.g., via simulation or hardware performance counters).

### 2.3 Worked Example — Processor Performance Equation
*(Adapted directly from the textbook's own worked example, Section 1.9.)*
Given measurements:
- Frequency of FP operations = 25%, average CPI of FP ops = 4.0
- Average CPI of other instructions = 1.33
- Frequency of FPSQR = 2%, CPI of FPSQR = 20

**Step 1 — baseline CPI:**
`CPI_original = (4 × 0.25) + (1.33 × 0.75) = 2.0`

**Step 2 — Option A: speed up FPSQR only (CPI: 20 → 2):**
`CPI_new = CPI_original − 0.02 × (20 − 2) = 2.0 − 0.36 = 1.64`

**Step 3 — Option B: speed up all FP instructions (CPI: 4.0 → 2.5):**
`CPI_new_FP = (0.75 × 1.33) + (0.25 × 2.5) = 1.625`

**Step 4 — compare via speedup** (clock rate and IC unchanged, so speedup = ratio of CPIs):
`Speedup_FP = CPI_original / CPI_new_FP = 2.00 / 1.625 = 1.23`
`Speedup_FPSQR = CPI_original / CPI_new = 2.00 / 1.64 = 1.22`

**Conclusion:** improving all FP instructions is marginally better than improving only FPSQR, because it touches a larger fraction of execution — exactly the "focus on the common case" principle at work quantitatively.

### 2.4 Amdahl's Law — Deep Dive
Amdahl's Law formalizes "focus on the common case." If an enhancement speeds up only a *fraction* of execution, overall speedup is capped.

```
Speedup_overall = 1 / [ (1 − Fraction_enhanced) + (Fraction_enhanced / Speedup_enhanced) ]
```
- `Fraction_enhanced`: portion of *original* execution time that can use the enhancement (always ≤ 1).
- `Speedup_enhanced`: how much faster that portion runs *if used exclusively in enhanced mode*.

**Worked example (from the text):** A new processor is 10× faster on computation; original processor spends 40% of time computing, 60% waiting on I/O.
```
Fraction_enhanced = 0.4, Speedup_enhanced = 10
Speedup_overall = 1 / [0.6 + 0.4/10] = 1 / 0.64 ≈ 1.56
```
Even a 10× local speedup yields only 1.56× overall — because I/O (60% of time) is untouched. **This is the single most important intuition Amdahl's Law teaches: the un-accelerated portion becomes the bottleneck and dominates as the accelerated portion approaches "instant."**

**Corollary (important for interviews/exams):** if a fraction *f* can never be sped up, overall speedup can never exceed `1 / (1 − f)`, no matter how much the rest is accelerated.

**Common mistake to flag explicitly (the book calls this out directly):** confusing "fraction of *original* time that could use the enhancement" with "fraction of *new* time spent in the enhanced mode" — these are different denominators and swapping them gives a wrong answer.

### 2.5 Benchmarks — How Performance Is Actually Measured
The book's position: **the only trustworthy performance metric is execution time of real programs.** Everything else is a proxy that can be gamed.
- **Kernels** — small extracted hot loops of real applications.
- **Toy programs** — simple teaching programs (e.g., quicksort) — not representative.
- **Synthetic benchmarks** — artificial programs designed to "look like" real workloads (e.g., Dhrystone) — historically prone to compiler gaming.
- **Benchmark suites** (e.g., SPEC CPU2006 = 12 integer + 17 floating-point real programs) — reduce single-benchmark gaming risk by aggregating many real programs; SPEC uses **geometric mean** of normalized ratios specifically because it is consistent regardless of which machine is used as the reference (unlike an arithmetic mean of ratios, which is not).

### 2.6 Numerical / Solved Problems
1. **CPI and CPU time.** A program executes `10^9` instructions on a machine running at 2 GHz with an average CPI of 1.5. Find CPU time.
   `CPU time = IC × CPI × cycle_time = 10^9 × 1.5 × (1/2×10^9) = 0.75 s`
2. **Speedup via Amdahl's Law.** An enhancement affects 30% of execution and offers a 5× local speedup. Find overall speedup.
   `Speedup = 1 / [0.7 + 0.3/5] = 1 / 0.76 ≈ 1.32`
3. **MIPS-style mixed-CPI problem.** Instruction mix: 50% ALU (CPI=1), 30% Load/Store (CPI=4), 20% Branch (CPI=2). Find average CPI.
   `CPI = 0.5(1) + 0.3(4) + 0.2(2) = 0.5 + 1.2 + 0.4 = 2.1`

### 2.7 Diagrams / Conceptual View
```
                        CPU TIME
                 ┌───────────┬───────────┬─────────────┐
                 │ Instr.    │   CPI     │ Clock Cycle │
                 │  Count    │           │    Time     │
                 └───────────┴───────────┴─────────────┘
                        ↑           ↑            ↑
                     ISA +     Micro-        Circuit /
                   Compiler   architecture   fabrication
                                + ISA         technology
```
```
Amdahl's Law — diminishing returns as Speedup_enhanced → ∞:
Speedup_overall → 1 / (1 − Fraction_enhanced)      (a hard ceiling)
```

### 2.8 Connections to Other Topics
- CPI is where **pipelining** (Part 2) directly attacks performance — ideal pipelining drives CPI toward 1 by overlapping instructions; hazards are precisely the things that push CPI back up via stalls.
- Instruction count is where **ISA design** (Section 3) matters — CISC-style rich instructions can lower IC at the cost of higher per-instruction CPI, while RISC does the opposite.
- Clock cycle time is limited by the **critical path** through the datapath and, in pipelined machines, by the depth of pipeline stages — a direct bridge to Part 2.
- Amdahl's Law reappears fundamentally in **multiprocessing** (Part 4) when reasoning about the speedup limits of parallelizing only part of a program across many cores.

### 2.9 How It Appears in Modern Processors
Modern CPUs report **IPC (instructions per clock)**, the inverse of CPI, as a headline efficiency metric (e.g., "This core achieves 3+ IPC on integer workloads"). Real-world benchmark suites (SPEC CPU, geekbench, industry TPC benchmarks for servers) are the direct descendants of the SPEC methodology described above. Performance counters embedded in real silicon (as H&P note) let engineers directly measure IC and cycle counts per code region — the same processor performance equation is still the working tool architects use today.

### 2.10 Key Terms & Definitions
- **Response time / execution time / latency:** time to complete one task.
- **Throughput:** work completed per unit time.
- **CPI (Cycles Per Instruction):** average clock cycles consumed per instruction.
- **IPC:** 1/CPI — instructions completed per clock cycle.
- **Speedup:** ratio of old to new execution time (or new to old performance).
- **Amdahl's Law:** speedup limit imposed by the un-accelerated fraction of a task.
- **SPEC:** Standard Performance Evaluation Corporation — industry-standard real-program benchmark suites.
- **Geometric mean:** the mean used for aggregating normalized SPECRatios, chosen because it is reference-machine-independent.

---

## 3. Instruction Set Architecture (ISA)

### 3.1 What It Is
The ISA is, in H&P's words, **"the portion of the computer visible to the programmer or compiler writer."** It is the contract: what instructions exist, what operands they take, how memory is addressed, how control flow works, and how instructions are encoded in bits. Two processors with completely different internal hardware (organization) can implement the *same* ISA and run the exact same binaries.

### 3.2 Classifying ISAs — Where Do Operands Live?
The most basic ISA classification question: **where are the operands for an ALU operation?**

| Class | Operand location | Example code for `C = A + B` |
|---|---|---|
| **Stack** | implicitly top-of-stack | `Push A; Push B; Add; Pop C` |
| **Accumulator** | one implicit accumulator register | `Load A; Add B; Store C` |
| **Register-memory** | one operand register, one memory | `Load R1,A; Add R3,R1,B; Store R3,C` |
| **Register-register / load-store** | all operands in registers | `Load R1,A; Load R2,B; Add R3,R1,R2; Store R3,C` |

Load-store (register-register) is what **virtually every architecture designed after 1980 uses** (MIPS, ARM, SPARC, Alpha, PowerPC, RISC-V) because:
1. Registers are physically faster to access than memory.
2. Registers are far more efficient for a compiler to allocate variables into — expressions can be evaluated in any convenient order, unlike a stack machine which is forced into one evaluation order.
3. Using registers for variables reduces memory traffic, speeds up programs, and improves code density (a register needs fewer encoding bits than a memory address).

### 3.3 Memory-Operand Taxonomy
A more precise classification counts **how many operands of a typical ALU instruction may be memory addresses**:

| # memory addresses | Max total operands | Architecture type | Examples |
|---|---|---|---|
| 0 | 3 | Load-store | Alpha, ARM, MIPS, PowerPC, SPARC |
| 1 | 2 | Register-memory | IBM 360/370, Intel x86, Motorola 68000 |
| 2 | 2 | Memory-memory | VAX (also has 3-operand forms) |
| 3 | 3 | Memory-memory | VAX |

**Trade-off table (RISC-style load-store vs. register-memory vs. memory-memory), summarized from H&P:**

| Type | Advantage | Disadvantage |
|---|---|---|
| Register-register (load-store) | Simple, fixed-length encoding; simple to pipeline | Higher instruction count than architectures that allow memory operands directly |
| Register-memory | Data can be accessed without a separate load; encoding fairly simple | Operands are not equivalent since a source in this form is destroyed; encoding a register number and a memory address in each instruction may restrict register count |
| Memory-memory | Most compact, doesn't waste registers for temporaries | Large variation in instruction size and work per instruction; can create memory bottleneck |

### 3.4 RISC vs. CISC — The Big Picture
- **CISC (Complex Instruction Set Computer):** rich instruction repertoire, variable-length encoding, instructions can directly operate on memory — historically motivated by expensive memory and a desire to reduce instruction count when compilers were primitive. Example lineage: VAX, x86.
- **RISC (Reduced Instruction Set Computer):** simple, fixed-length instructions, load-store only, designed to be easy to pipeline and to let the compiler do more of the optimization work. Example lineage: MIPS, ARM, SPARC, PowerPC, RISC-V.

Modern x86 chips are architecturally CISC on the outside but internally translate to RISC-like **micro-ops** and execute them on a RISC-style pipelined core — H&P explicitly flag this: "Intel to use a RISC instruction set internally while supporting an 80x86 instruction set externally." This is a critical modern-processor insight: **ISA (what you see) and organization (how it's executed) can diverge completely.**

### 3.5 Addressing Modes and Operand Types (Beginner → Advanced)
- **Immediate:** the operand is a constant encoded directly in the instruction (`ADD R1, R2, #5`).
- **Register:** operand is in a register.
- **Displacement / base+offset:** effective address = register value + constant offset — the dominant addressing mode for load/store in RISC ISAs, since most memory accesses are to structures/arrays near a base pointer.
- **Direct/absolute, indirect, indexed, auto-increment/decrement:** additional modes found in richer (often CISC) ISAs — each adds hardware/decoding complexity but can reduce instruction count for specific access patterns (e.g., array traversal).

Operand data types typically include byte, half-word, word, and double-word integers, and single/double precision floating point — modern ISAs also add SIMD/vector-width operand types (see Part 4/multiprocessing discussion of data-level parallelism).

### 3.6 Instructions for Control Flow
All ISAs need a way to alter the sequential flow of execution:
- **Conditional branches:** test a condition (register value, flag, or explicit compare) and jump if true (e.g., `BEQ`, `BNE`).
- **Unconditional jumps:** always transfer control.
- **Procedure call/return:** save a return address (link register or stack) and later transfer back — e.g., ARM's "branch and link."
- **Supervisor calls / traps:** controlled transfer into privileged/OS code.

The instruction set principles appendix stresses that branch/jump behavior and its addressing (PC-relative vs. absolute) matters a great deal for both code density and, critically, for **pipelining** (Part 2) — PC-relative branches are position-independent and simpler for hardware to compute early.

### 3.7 Encoding an Instruction Set
Two broad approaches:
- **Fixed-length encoding** (typical RISC choice, e.g., all ARM instructions are 32 bits, all classic MIPS instructions are 32 bits): simplifies instruction fetch and decode, which is essential for clean pipelining, at some cost to code density.
- **Variable-length encoding** (typical CISC choice, e.g., x86): improves code density (common short instructions take fewer bytes) at the cost of much more complex, multi-cycle instruction decoding — a major reason CISC decode stages are historically deeper/more complex pipeline stages.

ARM additionally offers **Thumb**, a compressed 16-bit encoding of a subset of the ARM instruction set, specifically to improve code density for embedded/PMD use cases without abandoning the underlying 32-bit RISC core — a direct real-world illustration of the code-size-vs-simplicity trade-off.

### 3.8 Worked / Numerical Example — Instruction Count Impact of ISA Choice
Suppose a stack machine needs 4 instructions (`Push A; Push B; Add; Pop C`) to compute `C = A+B`, while a load-store machine needs 4 as well but with different per-instruction cost (loads/stores cost 2 cycles, ALU ops cost 1 cycle in a simple non-pipelined model):
- Stack machine CPI-weighted cycles: Push(2)+Push(2)+Add(1)+Pop(2) = 7 cycles.
- Load-store machine: Load(2)+Load(2)+Add(1)+Store(2) = 7 cycles.
Even though both have the same instruction count here, in general stack/accumulator machines require *more* total instructions for anything beyond trivial expressions (e.g., `(A*B)-(B*C)-(A*D)` forces one fixed evaluation order and repeated loads on a stack machine, but can be freely reordered and register-allocated on a load-store machine) — this is exactly why register machines win once compiler optimization matters.

### 3.9 Diagrams
```
Stack:        [TOS]->[B]->[A]        Accumulator:  ACC <-> ALU <-> MEM
              ALU pops 2, pushes 1                  (1 implicit operand register)

Register-register / load-store:
   MEM <-> [R0..R31] <-> ALU
   (all ALU operands/results are registers; MEM touched only by explicit
    LOAD/STORE instructions)
```
```
Fixed-length instruction (ARM-style, 32 bits):
[ 31    28 | 27      20 | 19  16 | 15  12 | 11        0 ]
[  cond    |    opcode   |  Rn    |  Rd    |  operand2   ]
-> uniform width means the decode stage can always know instruction
   boundaries immediately, which pipelines cleanly.
```

### 3.10 Connections to Other Topics
- Instruction count (IC) in the CPU performance equation (Section 2) is directly determined by ISA richness.
- Fixed-length, load-store ISA design is what makes clean 5-stage pipelining (Part 2) practical — this is not a coincidence; RISC was co-designed with pipelining in mind.
- Load-store architecture is precisely why cache/memory hierarchy (Part 3) design can be reasoned about cleanly: all memory traffic funnels through a small, well-defined set of Load/Store instructions.

### 3.11 How It Appears in Modern Processors
- **ARM (AArch32/AArch64):** fixed 32-bit instructions (plus Thumb-2 variable compressed encoding), load-store, large register file, condition-code execution on (nearly) every instruction in AArch32.
- **x86-64:** variable-length CISC encoding at the ISA level, but internally cracked into RISC-like micro-ops for out-of-order execution.
- **RISC-V:** a modern, modular, open ISA explicitly designed around the same load-store, fixed-length-with-compressed-extension philosophy as ARM/MIPS.

### 3.12 Common Pitfalls
- Assuming "RISC = fewer instructions to write in assembly" — it actually usually means *more* instructions (higher IC) but lower CPI and simpler, faster hardware; the net CPU-time effect is what matters, not raw instruction count.
- Confusing ISA-level CISC/RISC labeling with actual internal execution style (modern x86 is RISC-like internally).
- Forgetting that addressing-mode and encoding choices are compiler-facing decisions as much as hardware decisions (Appendix A explicitly frames this as "the role of compilers").

### 3.13 Key Terms & Definitions
- **ISA:** instruction set architecture, the hardware/software contract.
- **Load-store architecture:** ALU instructions only touch registers; memory is touched only by explicit load/store instructions.
- **Addressing mode:** the method used to specify an operand's location.
- **RISC / CISC:** reduced vs. complex instruction set computer design philosophies.
- **Micro-ops:** internal RISC-like operations that CISC ISAs (e.g., x86) are translated into before execution.
- **Thumb:** ARM's compressed 16-bit instruction encoding for code density.

---

## 4. CPU Datapath Basics

### 4.1 What It Is
The **datapath** is the collection of hardware components that carry, store, and process data — registers, ALUs, multiplexers, and the buses connecting them. As Furber's *ARM SoC Architecture* puts it: "All the components carrying, storing or processing many bits in parallel" belong to the datapath, and are typically designed at the **register transfer level (RTL)** — reasoning in terms of registers, multiplexers, and the transfers between them each clock cycle, rather than individual gates.

### 4.2 How It Works — Learning via MU0 (a Minimal Teaching CPU)
The clearest way to internalize datapath design is via a minimal processor. Furber's book uses **MU0**, a teaching CPU historically used at the University of Manchester, built from exactly five components:
- **PC (Program Counter):** holds the address of the current instruction.
- **ACC (Accumulator):** holds the working data value.
- **ALU (Arithmetic-Logic Unit):** performs add/subtract/increment/pass-through operations.
- **IR (Instruction Register):** holds the instruction currently being executed.
- **Instruction decode and control logic:** interprets the IR and drives the datapath.

MU0 is a 16-bit machine, 16-bit instructions (4-bit opcode + 12-bit address field), addressing up to 4,096 16-bit memory locations. Its instruction set includes operations like `ACC := ACC + mem16[S]` ("add memory contents at address S into the accumulator").

**Key datapath design principle used by MU0** (directly generalizable to any simple CPU): **the number of clock cycles an instruction takes is determined by the number of memory accesses it must make**, because memory access is the design's limiting resource. Instructions needing an operand (add/subtract/load/store-type) take 2 cycles (fetch instruction, then fetch/store operand); instructions with no memory operand (e.g., increment accumulator) take 1 cycle.

**Two-stage instruction execution pattern (general and reusable across any simple CPU):**
1. **Operand access + operation:** issue the address from IR, read/write memory, combine with ACC in the ALU, write back to ACC (or store ACC to memory).
2. **Next-instruction fetch:** issue PC (or IR-held address for a jump) to fetch the next instruction, and simultaneously increment PC using the same ALU (a beginner "aha": you often don't need a *dedicated* PC-incrementer if the main ALU is free during the fetch cycle).

### 4.3 Why It's Important
Every CPU, no matter how advanced (superscalar, out-of-order, deeply pipelined), is still fundamentally organized around this same idea: registers hold state, an ALU computes new values, multiplexers route data, and control logic sequences the whole thing correctly, cycle by cycle. Understanding a minimal datapath like MU0 gives you the mental model to reason about vastly more complex real CPUs (the ARM pipeline in Part 2 is a direct, more elaborate descendant of exactly this structure).

### 4.4 Where It's Used
Every processor core — from an 8-bit microcontroller to a superscalar server CPU — has a datapath at its heart. GPUs replicate simplified datapaths thousands of times in parallel (SIMD lanes); DSPs specialize the ALU for multiply-accumulate operations; the same RTL "registers + ALU + muxes" vocabulary applies everywhere.

### 4.5 Numerical / Worked Example
MU0's `ADD` instruction (opcode implying `ACC := ACC + mem16[S]`) at address 0 in memory, operating on data at address 5:
- **Cycle 1:** IR already holds the fetched ADD instruction (address field S=5); issue address 5 to memory, read `mem[5]`, feed into ALU with ACC, compute `ACC + mem[5]`, write result back into ACC.
- **Cycle 2:** issue PC (currently pointing to address 1, the next instruction) to fetch the next instruction into IR; simultaneously increment PC (0+1=1 → becomes 2) using the ALU.

This 2-cycle pattern repeats for every operand-using instruction; the total program execution time is (number of operand instructions × 2 + number of non-operand instructions × 1) clock cycles — a direct, concrete instance of the CPU-time formula from Section 2 (`IC × CPI × cycle time`, here with per-instruction-class CPI of 1 or 2).

### 4.6 Diagrams
```
                    ┌───────────────────────────────────────┐
                    │              CONTROL LOGIC             │
                    │      (decodes IR, drives datapath)     │
                    └───────────────┬─────────────────────────┘
                                     │ control signals
   ┌──────┐   ┌──────┐   ┌───────┐  ▼   ┌──────┐
   │  PC  │   │  IR  │   │  ACC  │◄────►│ ALU  │
   └───┬──┘   └───┬──┘   └───┬───┘      └──┬───┘
       │  (mux selects address source)     │
       └────────────┬───────────────────────┘
                     ▼
              ┌─────────────┐
              │   MEMORY    │
              └─────────────┘
```
```
Two-phase MU0 instruction execution timeline:
clk   __|‾|__|‾|__|‾|__|‾|__
      |<-- op+access -->|<-- next fetch + PC incr -->|
       (cycle 1: 1 instr)     (cycle 2: next instr's fetch)
```

### 4.7 Connections to Other Topics
- The datapath is exactly what gets **pipelined** in Part 2 — a pipelined CPU is essentially this same fetch/decode/execute/memory/writeback structure, but with multiple instructions occupying different stages of the datapath simultaneously.
- The ALU's operations (add, subtract, compare) are what determine per-instruction CPI in the performance equation (Section 2).
- The datapath's memory-access behavior is the origin point for everything in the memory hierarchy discussion (Part 3): every load/store instruction is a datapath-initiated memory transaction.

### 4.8 How It Appears in Modern Processors
Real CPUs (e.g., the ARM organizations covered in Part 2, or a modern out-of-order x86 core) still use the same primitives — registers, ALUs, multiplexers — just replicated (multiple ALUs for superscalar issue), pipelined (stages inserted between register boundaries), and augmented with much larger register files, forwarding networks, and reservation stations. The MU0 model is the "hello world" of datapath design that scales conceptually all the way up.

### 4.9 Common Pitfalls
- Thinking the ALU alone "is" the CPU — the ALU is only useful when correctly wired via muxes/registers to receive the right operands each cycle, controlled by the control unit.
- Missing that memory access is very often the true limiting factor for cycle count in a simple, non-pipelined CPU (MU0's own design principle) — this motivates caches and pipelining later.
- Forgetting the two-stage (operand-then-fetch) execution pattern generalizes, in more elaborate form, to the classic 5-stage RISC pipeline (IF-ID-EX-MEM-WB) covered in Part 2.

### 4.10 Key Terms & Definitions
- **Datapath:** the hardware that carries/stores/processes data (registers, ALU, multiplexers, buses).
- **RTL (Register Transfer Level):** a design abstraction describing behavior in terms of register-to-register data transfers each clock cycle.
- **ACC (Accumulator):** a single implicit working register (used in simple/accumulator-style machines).
- **IR (Instruction Register):** holds the instruction currently being decoded/executed.
- **PC (Program Counter):** holds the address of the current/next instruction to fetch.

---

## 5. Control Unit Basics

### 5.1 What It Is
The **control unit** is the part of the CPU that is *not* the datapath: it decodes the current instruction and generates the correct sequence of control signals to make the datapath perform the intended operation, cycle by cycle. Furber's book states this precisely: "Everything that does not fit comfortably into the datapath will be considered part of the control logic and will be designed using a finite state machine (FSM) approach" — directly connecting control unit design to the FSM material most digital-design courses (and interview prep) already cover.

### 5.2 How It Works
For MU0, the control unit only needs **two states**: `fetch` and `execute` — one bit of state (`Ex/ft`) is sufficient. The control logic:
1. Looks at the current state and the opcode bits from IR.
2. Produces the correct levels on all datapath control signals for this cycle: register clock-enables (e.g., `PCce`, `IRce`, `ACCce`), ALU function-select lines, multiplexer select lines, and memory request/read-write signals (`MEMrq`, `RnW`).
3. Also consumes status signals *from* the datapath — e.g., whether ACC is zero or negative — to correctly implement conditional-jump instructions.

**Practical simplification example (straight from the text):** in MU0's control table, `PCce` and `IRce` (clock-enables for the PC and instruction register) turn out to always be identical, because whenever a new instruction is being fetched, the ALU is simultaneously computing the next PC value, and both should be latched together — so the two control signals can simply be merged into one wire. Similarly, whenever the accumulator drives the data bus outward (`ACCoe` high, i.e. a store), the memory must be doing a write (`RnW` low) — so one signal can be derived from the other via a single inverter. **This is a concrete, real example of using logic minimization (from your digital-design background) directly inside control-unit design.**

Once the control table is built, it can be implemented either as a **PLA (programmable logic array)** — a direct, systematic hardware structure for implementing arbitrary combinational truth tables — or synthesized into standard combinational gates. Two broad implementation philosophies exist for any control unit:
- **Hardwired control:** the FSM is implemented directly in combinational logic + a state register — fast, but inflexible and harder to modify/debug (this is what MU0 uses).
- **Microprogrammed control:** the control logic is itself a small stored program (microcode) executed by a simple internal sequencer — more flexible/easier to change/debug/extend (mentioned by Furber as an alternative MU0 implementation approach used historically, and famous historically in CISC machines like the VAX).

### 5.3 Why It's Important
Interviewers and textbooks alike use control-unit design as the definitive test of whether you can connect **FSM theory** (Moore/Mealy, state encoding, next-state logic — see the companion FPGA/RTL notes for a hardware-level deep-dive on this exact topic) to **real processor design**. A CPU's control unit is, quite literally, one large FSM whose "outputs" are the enable/select signals driving an entire datapath.

### 5.4 Where It's Used
Every processor without exception has a control unit, whether hardwired (typical in modern high-performance RISC pipelines, where speed is paramount) or microprogrammed (historically common in CISC machines needing to support large, irregular instruction sets economically; modern x86 chips still use a hybrid — hardwired control for common simple instructions, microcode ROM fallback for complex/rare instructions).

### 5.5 Numerical / Structural Example — Control Table Reasoning
For MU0's `ADD` instruction, sketch the control signal values per cycle:

| Cycle | State | PCce/IRce | ACCce | ALU func | MEMrq | RnW |
|---|---|---|---|---|---|---|
| 1 | execute | 0 | 1 (latch new ACC) | A+B | 1 (read) | 1 (read) |
| 2 | fetch | 1 (latch new PC+IR) | 0 | B+1 (PC increment) | 1 (read) | 1 (read) |

This table is exactly the kind of "control logic in tabular form" the textbook describes — and it is directly analogous to the next-state/output tables used when designing any FSM in Verilog (3-always-block style: state register, next-state logic, output logic).

### 5.6 Diagrams
```
                     ┌───────────────────────────────┐
   opcode bits  ───► │                                 │
   (from IR)         │        CONTROL UNIT (FSM)       │───► control signals to datapath
   status flags ───► │   states: FETCH ⇄ EXECUTE       │     (PCce, IRce, ACCce, ALU func,
   (Z, N from ALU)   │                                 │      mux selects, MEMrq, RnW)
                     └───────────────────────────────┘
```
```
MU0 control FSM (2-state):
   ┌────────┐  instr. latched   ┌─────────┐
   │ FETCH  │ ─────────────────►│ EXECUTE │
   └────────┘◄───────────────── └─────────┘
        ▲       next fetch issued
        │
      reset
```

### 5.7 Connections to Other Topics
- This is exactly the FSM design pattern (state register + next-state combinational logic + output logic) that appears throughout digital design and RTL interviews — control unit design *is* FSM design, applied to an entire CPU.
- In a pipelined CPU (Part 2), control becomes distributed — each pipeline stage effectively needs its own local control signals derived from the instruction as it flows through, and hazard detection logic (Part 2) is really an *extension* of the control unit's job: deciding when to stall, forward, or flush.
- The hardwired-vs-microprogrammed trade-off directly echoes the RISC-vs-CISC ISA discussion in Section 3 — simple, regular RISC ISAs are what make hardwired control practical and fast.

### 5.8 How It Appears in Modern Processors
Modern high-performance cores use **hardwired control** for the vast majority of common instructions (for speed), but retain a **microcode ROM** for rare/complex operations (string operations, certain system instructions) — a hybrid directly descended from the philosophies described above. Out-of-order superscalar cores extend the control unit concept enormously: instruction scheduling, register renaming, and reorder-buffer management are all, at their core, elaborate FSM/control-logic problems layered on top of the same fetch-decode-execute skeleton.

### 5.9 Common Pitfalls
- Believing the control unit and datapath are designed independently — in practice they're co-designed; datapath choices (e.g., "can the ALU double as a PC incrementer?") directly shape how simple or complex the control FSM needs to be.
- Not recognizing that "which cycles does this instruction need, and what must be true in each" is exactly the analysis interviewers want when they ask you to "design a simple CPU control unit" — build the control table before trying to write RTL.
- Assuming microprogrammed control is "obsolete" — it's alive and well as a fallback path in modern CISC-derived processors.

### 5.10 Key Terms & Definitions
- **Control unit:** the FSM-based logic that decodes instructions and drives datapath control signals.
- **Hardwired control:** control logic implemented directly as combinational logic + state register.
- **Microprogrammed control:** control logic implemented as a small stored microcode program executed by an internal sequencer.
- **PLA (Programmable Logic Array):** a systematic hardware structure for implementing arbitrary combinational control tables.
- **Control signal:** an individual enable/select/function-code wire driven by the control unit into the datapath (e.g., `PCce`, `RnW`, ALU function select).

---

## 6. Assembly-Level Understanding

### 6.1 What It Is
Assembly language is the human-readable, near-1:1 textual representation of machine instructions — the level at which you can directly observe how a compiler (or you, by hand) expresses an algorithm in terms of the ISA's actual registers, addressing modes, and control-flow instructions covered in Section 3.

### 6.2 How It Works — ARM as the Working Example
Using the ARM programmer's model (Furber, Ch. 2) as the concrete reference:
- **Registers visible to user-level code:** 15 general-purpose 32-bit registers (`r0`–`r14`), the program counter (`r15`), and the **CPSR** (Current Program Status Register).
- **CPSR condition flags** — set by ALU operations that update flags, and consumed by conditional branches/execution:
  - **N (Negative):** result's top bit was 1.
  - **Z (Zero):** result was all-zero.
  - **C (Carry):** an arithmetic/shift carry-out occurred.
  - **V (oVerflow):** signed arithmetic overflow into the sign bit.
- **Instruction categories** (a clean, memorizable taxonomy): (1) **data processing** — register-only ALU ops; (2) **data transfer** — load/store between register and memory; (3) **control flow** — branch, branch-and-link (call), supervisor call (trap).
- **3-address format:** ARM data-processing instructions specify two source registers and a (possibly different) destination register independently, e.g., `ADD R3, R1, R2` means `R3 := R1 + R2` — contrast this with 2-address forms (`ADD R1, R2` meaning `R1 := R1+R2`, destroying a source) found in many CISC ISAs.
- **Conditional execution:** uniquely (among mainstream ISAs) essentially every ARM instruction, not just branches, can carry a condition code and execute conditionally — a compact way to avoid short forward branches (e.g., `ADDEQ R1, R2, R3` only executes if the Z flag is set).

### 6.3 Why It's Important
Reading assembly is how you verify — and debug — that a compiler (or your own RTL/microarchitectural reasoning) is doing what you think. It is also the concrete bridge between the abstract ISA principles of Section 3 and the datapath/control unit machinery of Sections 4–5: every assembly instruction is a request for the control unit to sequence the datapath through a specific set of register transfers.

### 6.4 Where It's Used
Compiler back-ends emit it; performance engineers read it to understand why code is slow (e.g., unexpected spills to memory, missed vectorization); OS/driver/embedded developers sometimes write it directly for tight control over timing or hardware access; security researchers reverse-engineer it.

### 6.5 Worked Example — Translating C to Assembly
C source:
```c
int C = A + B;
```
ARM assembly (assuming A, B, C are memory variables with addresses already loaded into base registers, or simply values already in registers):
```asm
LDR R0, [R4]      ; R0 := mem[R4]   (load A, R4 holds &A)
LDR R1, [R5]      ; R1 := mem[R5]   (load B, R5 holds &B)
ADD R2, R0, R1    ; R2 := R0 + R1   (data processing, 3-address)
STR R2, [R6]      ; mem[R6] := R2   (store C, R6 holds &C)
```
Compare directly against the four ISA classes from Section 3.2 — this is precisely the **register (load-store)** column of Figure A.2 in the textbook, and it's worth explicitly re-deriving the stack/accumulator/register-memory equivalents from Section 3.2 as a self-test.

**Conditional branch example** — a simple `if (R0 == 0) goto label;`:
```asm
CMP R0, #0        ; sets Z flag if R0 == 0
BEQ label         ; branch if Z flag set
```

### 6.6 Diagrams
```
C source:  C = A + B;
                │  compiler
                ▼
Assembly:  LDR R0,[R4]  LDR R1,[R5]  ADD R2,R0,R1  STR R2,[R6]
                │  assembler
                ▼
Machine code (32-bit ARM instruction words, hex):
  0xE5940000   0xE5951000   0xE0802001   0xE5862000
```

### 6.7 Numerical / Solved Problem
How many total memory accesses does the 4-instruction example above require, and how does that map to CPU time using Section 2's formula? Two instruction fetches for `LDR`s, one for `ADD`, one for `STR` = 4 instruction fetches; plus 2 data loads + 1 data store = 3 data accesses. Total = 7 memory accesses for this 4-instruction sequence — directly illustrating why memory system performance (Part 3) so strongly dominates real CPU time, exactly as MU0's datapath design principle (Section 4.2) anticipated in miniature.

### 6.8 Connections to Other Topics
- Every assembly instruction is a "test case" for the control unit's FSM (Section 5) — tracing an assembly program by hand is equivalent to tracing the control unit's state and signal values cycle by cycle.
- Register usage patterns in real assembly code (which registers are read/written, how close together) are precisely what create the **data hazards** central to Part 2's pipelining discussion — e.g., the `ADD R2,R0,R1` instruction above depends on both preceding loads completing first.
- Load/Store instruction frequency and address patterns in real assembly directly drive **locality of reference** and cache behavior (Part 3).

### 6.9 How It Appears in Modern Processors
Modern compilers (GCC, LLVM/Clang) generate assembly for ARM, x86-64, and RISC-V using exactly this register-allocation-centric, load-store-aware reasoning, now heavily augmented by instruction scheduling to avoid pipeline hazards (a preview of Part 2) and to maximize cache locality (a preview of Part 3). Tools like `objdump`, Compiler Explorer, and `perf` let engineers inspect exactly this level of representation on real hardware today.

### 6.10 Common Pitfalls
- Assuming assembly directly reflects execution order/timing on a modern superscalar, out-of-order core — it doesn't; assembly (and even machine code) is just the *architectural* instruction stream, while the actual execution order inside the hardware can be reordered extensively (this distinction becomes central in Part 2).
- Confusing condition-flag-setting instructions with condition-flag-*consuming* instructions — e.g., forgetting that not all ARM ALU instructions update CPSR unless explicitly suffixed (`ADDS` vs. `ADD`).
- Treating 2-address and 3-address instruction forms as interchangeable when hand-translating C to assembly — 2-address forms destroy a source operand, which can silently change program semantics if you're not careful.

### 6.11 Key Terms & Definitions
- **CPSR:** ARM's Current Program Status Register, holding condition flags (N, Z, C, V) and processor mode/interrupt-enable bits.
- **3-address instruction:** an instruction with two independent source operands and a separate destination.
- **Conditional execution:** the ability of an instruction (beyond just branches) to execute only if a condition code matches.
- **Assembler:** the tool translating human-readable assembly mnemonics into machine code (binary instruction words).
- **Data processing / data transfer / control flow instructions:** ARM's three-way taxonomy of instruction categories (Section 6.2).

---

## Bridge to Part 2
Part 1 established the vocabulary and mental models: what architecture *is*, how to *measure* performance quantitatively (CPI, Amdahl's Law), how instructions are *designed* (ISA), and how a simple CPU *executes* them cycle by cycle (datapath + control unit + assembly). Part 2 takes this same MU0-style datapath and asks the natural next question: **what happens if we let multiple instructions occupy different parts of the datapath at the same time?** That is pipelining — and it immediately creates the central engineering problem of the entire course: **hazards**.

*(End of Part 1. Say "next" for Part 2: Pipelining, Pipeline Hazards, Hazard Detection & Forwarding, Branch Handling & Prediction.)*
