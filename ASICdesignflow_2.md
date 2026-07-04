# ASIC Design Flow — Complete Study Notes (RTL to Signoff)
## PART 2 of 3 — Floorplanning, Power Planning, Placement, Clock Tree Synthesis, Routing, Parasitic Extraction

---

# 8. Floorplanning

## 8.1 What the Stage Is

Floorplanning is the first physical design step: deciding the **physical die size and shape**, placing major blocks (macros — large hierarchical sub-blocks, memories, analog IP) at specific physical locations, defining I/O pin locations around the die periphery, and establishing the overall physical structure (power domains, physical hierarchy) that all subsequent physical design stages build upon.

## 8.2 Why It Is Needed

Every subsequent stage (power planning, placement, CTS, routing) operates within the physical boundaries and macro locations established here — a poor floorplan (badly-placed macros, insufficient area for routing, poor I/O placement relative to internal logic) creates problems that ripple through the entire remaining flow and are extremely costly to fix later (often requiring a full floorplan redo, cascading back through every downstream stage).

## 8.3 Inputs

- The DFT-inserted netlist (Part 1, Section 7).
- **Die size / area budget** (from specification/architecture, Part 1 Sections 1-2).
- **Macro/hard-IP list and sizes** (memory compilers' generated macro dimensions, analog IP block footprints).
- I/O pad list and any pre-determined pin location constraints (often driven by package/PCB routing considerations decided even earlier, at the architecture/package co-design stage).

## 8.4 Outputs

- **Floorplan (die size, macro placement, I/O pin locations)** — typically saved as a physical design database (e.g., DEF — Design Exchange Format).
- **Power domain boundaries** (if using multiple voltage/power domains for power management, tying back to architecture planning's power strategy, Part 1 Section 2.4).
- **Initial estimate of routing congestion/utilization** based on the chosen floorplan.

## 8.5 Key Floorplanning Concepts

| Concept | Meaning |
|---|---|
| **Core area** | The region inside the die where standard cells/macros are placed |
| **I/O ring / pad ring** | The peripheral region containing I/O pads/pins |
| **Aspect ratio** | Ratio of die height to width — affects routing and area efficiency; extreme aspect ratios (very tall/thin) are generally harder to route well |
| **Utilization** | Percentage of core area actually occupied by standard cells/macros vs. left for routing — real designs typically target 60-80% utilization, **not 100%**, leaving room for routing and timing-closure-driven optimization (directly analogous to the FPGA utilization margin discussion, Vivado notes Section 11.4) |
| **Macro placement / macro halo** | Placing large hard blocks (memories, analog IP) and reserving a keep-out margin (halo) around them for routing/power-strap clearance |
| **Channel** | Routing space deliberately left between macros/blocks |

## 8.6 What Problems Can Appear at This Stage

- **Poor macro placement** creating routing congestion "hotspots" (e.g., macros placed too close together, blocking routing channels needed for signals passing between other logic).
- **Excessive die area** (wasted cost — chips are priced roughly by area at a given process node) or **insufficient area** (forces unrealistically high utilization, causing severe downstream congestion/timing problems).
- **I/O placement mismatched with internal logic layout** — forcing unnecessarily long internal routes from I/O-adjacent logic to the actual functional blocks using that I/O, hurting both timing and congestion.
- **Aspect ratio poorly suited to the package/board** — floorplan decisions must also consider how the die will be packaged and connected to the PCB.

## 8.7 How It Connects to the Next Stage

The floorplan (macro/IO locations, die size, power domain boundaries) is the physical canvas that **power planning** (Section 9) and then **placement** (Section 10) work within — nothing in subsequent stages can move macros or change the die size without effectively redoing the floorplan.

## 8.8 Practical Industry Usage

Floorplanning is often iterative and partially manual/expert-driven even in modern flows — while tools (Synopsys IC Compiler II, Cadence Innovus) provide automated floorplanning assistance, experienced physical design engineers make deliberate, judgment-based macro placement decisions based on data-flow understanding (which blocks communicate heavily with which) that automated tools don't always capture well on their own.

## 8.9 Verification/Debugging Implications

Floorplan quality is typically assessed via **early congestion/timing estimation tools** run directly on the floorplan (before full placement/routing) — if these early estimates show severe problems, it's far cheaper to revise the floorplan now than to discover the same problem after a full placement/routing run (which can take hours to days for large designs).

## 8.10 Timing Implications

Floorplanning fundamentally sets the **physical distances** that signals must travel — this directly bounds achievable timing, since wire delay scales with distance (and, at advanced nodes, wire delay increasingly dominates gate delay, as emphasized repeatedly in the FPGA notes' discussion of route delay, Vivado notes Section 7.3/9.6 — the same physics applies, arguably more critically, in ASIC design). A floorplan that places communicating logic far apart cannot be timing-closed later no matter how good subsequent placement/routing optimization is.

## 8.11 Physical Design Implications

(This entire stage is fundamentally a physical-design topic.) Floorplanning decisions also directly affect **power delivery network design** (Section 9) — macro placement determines where power straps/rings must route around or through, and **thermal considerations** (dense clusters of high-power macros can create local hotspots) increasingly factor into modern floorplanning decisions, especially at advanced nodes with high power density.

## 8.12 How to Explain It in an Interview

*"Floorplanning is where we decide the physical die layout — die size, where large macros and memories go, and where I/O pins are located — because this becomes the physical canvas every later stage works within. A bad floorplan, like macros placed too close together blocking routing channels, creates congestion and timing problems that are very expensive to fix later, since fixing them often means redoing the floorplan and repeating every downstream stage."*

## 8.13 Interview Questions

**Q (Easy):** Why don't real designs target 100% core utilization?
**A:** Some area must be left unoccupied by standard cells specifically for routing (wires need physical space to be drawn) and for the physical optimization flexibility placement/CTS/routing tools need to achieve timing closure — designs targeting near-100% utilization typically suffer severe routing congestion and timing closure difficulty (directly analogous to the FPGA utilization discussion, Vivado notes Section 11.4).

**Q (Medium):** How does macro placement affect routing congestion?
**A:** Macros are large, "opaque" regions that ordinary standard-cell routing cannot pass through — placing macros such that they block natural routing paths between other logic blocks (e.g., two macros placed adjacent to each other with no channel between them, when logic on either side needs to communicate) forces routing to detour, consuming extra routing resources and potentially creating severe local congestion.

**Q (Advanced):** Why is floorplanning still often partially manual/expert-driven even with sophisticated automated tools available?
**A:** Automated floorplanning tools optimize based on generally-applicable heuristics/algorithms (minimizing wire length, balancing utilization), but they don't always fully capture design-specific data-flow understanding — e.g., which specific blocks have particularly timing-critical or high-bandwidth communication paths between them that should be prioritized for physical proximity even at some cost to other, less-critical metrics; experienced engineers apply this design-specific judgment to guide/override automated placement in ways that generic algorithms may not discover on their own, especially for architecturally-critical communication paths.

---

# 9. Power Planning

## 9.1 What the Stage Is

Power planning designs the **power delivery network (PDN)** — the physical grid of power (VDD) and ground (VSS) metal routing that distributes power from the chip's external power pins to every standard cell and macro across the die, ensuring every gate receives adequate voltage to switch correctly and reliably.

## 9.2 Why It Is Needed

Every gate on the chip needs power to operate — if the power delivery network is inadequately designed, gates far from power pins experience **IR drop** (voltage droop due to resistance in the power routing, Section covered in depth in Part 3) severe enough to cause timing failures or even functional failures, and current density limits (electromigration, also Part 3) can cause long-term reliability failures if power routing is undersized for the current it must carry.

## 9.3 Inputs

- The floorplan (Section 8's output) — power planning is designed around the established macro/die layout.
- **Power budget and current estimates** per region of the design (from architecture-level power analysis, Part 1 Section 2.4, refined with more detailed post-synthesis power estimates).
- Technology library power/electrical rules (maximum current density per metal layer, from the foundry PDK).

## 9.4 Outputs

- **Power grid/mesh design** — physical VDD/VSS metal routing structure (typically implemented as **rings** around macros/die periphery and a **mesh/grid** of straps across the core area, connecting down to individual standard cell power rails via vias).
- **Power domain implementation details** — physical realization of any multiple-voltage-domain architecture decided earlier (level shifters at domain boundaries, power switch/header cell placement for power-gated domains).

## 9.5 Key Power Planning Concepts

| Concept | Meaning |
|---|---|
| **Power ring** | A ring of power/ground metal routing around the die periphery or around individual macros, providing a robust power source at that boundary |
| **Power straps/mesh** | A grid of power/ground metal lines running across the core area at regular intervals, connecting the ring(s) to the standard cell rows |
| **Standard cell power rail** | The lowest-level power/ground metal running along each row of standard cells, connecting individual cells to the strap/mesh above |
| **Power gating** | Deliberately switching off power to unused/idle blocks (via header/footer power switch cells) to reduce leakage power — a major power-reduction technique in modern low-power designs |
| **Multi-Vt cells (multi-threshold-voltage)** | Using standard cells built with different transistor threshold voltages (low-Vt: faster but leakier; high-Vt: slower but lower-leakage) selectively across the design — timing-critical paths use low-Vt cells, non-critical paths use high-Vt cells to save leakage power, a very standard modern low-power optimization technique applied during synthesis/placement |

## 9.6 What Problems Can Appear at This Stage

- **Insufficient power grid density** in high-current-demand regions — leading to excessive IR drop (Part 3) discovered later during signoff power integrity analysis, potentially requiring a power grid redesign (an expensive rework, since power straps consume routing resources that then aren't available for signal routing).
- **Power grid consuming too much routing resource**, leaving insufficient room for signal routing — a direct trade-off against the routing congestion concerns from floorplanning (Section 8).
- **Power domain boundary errors** — missing/incorrectly-placed level shifters or isolation cells at voltage domain crossings, causing functional or electrical failures.

## 9.7 How It Connects to the Next Stage

The power grid must be substantially in place **before or during placement** (Section 10), since standard cells need to connect to power rails as they're placed — power planning and placement are tightly interleaved in practice, not strictly sequential.

## 9.8 Practical Industry Usage

Modern advanced-node designs increasingly require **sophisticated, multi-layer power grids** with careful IR-drop-aware and electromigration-aware design from the start — power integrity has become a first-class signoff concern (Part 3) comparable in importance to timing signoff, particularly as chips have grown in power density.

## 9.9 Verification/Debugging Implications

Power grid quality is checked via **static and dynamic IR drop analysis** (Part 3) and **electromigration analysis** (Part 3) — these are dedicated signoff checks specifically targeting power delivery network adequacy, run after enough of the design is placed/routed to have a realistic current distribution estimate.

## 9.10 Timing Implications

IR drop directly slows down gate switching speed (a gate receiving less than its nominal supply voltage switches more slowly) — this is why IR-drop-aware STA (accounting for voltage droop in timing analysis, sometimes called **voltage-aware STA**) is standard practice in advanced-node signoff (Part 3 covers this in depth).

## 9.11 Physical Design Implications

(This entire stage is a physical design topic.) The power grid physically competes for routing resources with signal routing — this is why power planning and floorplanning (Section 8) are tightly coupled, and why power grid design decisions made early significantly constrain later routing (Section 12) options.

## 9.12 How to Explain It in an Interview

*"Power planning designs the physical grid of power and ground metal that delivers power from the chip's pins to every gate on the die. It matters because inadequate power delivery causes IR drop — voltage droop that slows down or even breaks gate switching — and because power routing itself consumes physical routing resources that compete with signal routing, so it has to be carefully balanced against the floorplan and later routing needs."*

## 9.13 Interview Questions

**Q (Easy):** What is the difference between a power ring and a power mesh/strap?
**A:** A power ring runs around the periphery of the die or a macro, providing a robust power source at that boundary; power straps/mesh are grids of power/ground routing that run across the core area at regular intervals, distributing power from the ring inward to every standard cell row.

**Q (Medium):** Why is power gating used, and what additional design elements does it require?
**A:** Power gating switches off power to idle/unused blocks to reduce leakage power (a significant fraction of total power in advanced process nodes even when logic isn't actively switching) — it requires power switch/header cells to control the power connection, and typically **isolation cells** at the boundary of the power-gated domain (to hold outputs at a defined, safe value while the domain is powered off, preventing floating/glitchy signals from propagating to always-on logic) and **retention flip-flops** if state must be preserved across a power-down cycle.

**Q (Advanced):** Why is power grid design a trade-off against routing congestion, and how does this connect back to floorplanning decisions?
**A:** Both power delivery and signal routing consume the same finite physical metal routing resources on each layer — a denser power grid (more/wider straps) provides better IR drop margin and current-carrying capacity but leaves less routing resource for signals, potentially causing signal routing congestion; floorplanning decisions (macro placement, utilization target, Section 8) directly affect how much routing headroom exists in the first place, so power planning, floorplanning, and eventual routing congestion are all interdependent — a change in one often requires re-evaluating the others, which is why physical design is often iterative rather than a strictly one-pass-per-stage process.

---

# 10. Placement

## 10.1 What the Stage Is

Placement assigns a specific physical (x,y) location to every standard cell (LUT-equivalent logic gates, flip-flops) in the netlist, within the floorplan established earlier (Section 8), aiming to minimize timing-critical path delay and routing congestion while respecting the physical power grid (Section 9) and any macro/blockage constraints.

## 10.2 How It Works (Conceptual)

Placement algorithms (in tools like Cadence Innovus, Synopsys IC Compiler II) use a combination of **analytical placement** (solving a mathematical optimization treating the netlist as a system of "forces" pulling connected cells together, similar in spirit to physical spring-force models) and **timing-driven refinement** (iteratively adjusting placement of cells on timing-critical paths to reduce their physical distance and thus wire delay), all constrained to legal, non-overlapping positions aligned to the standard cell rows established during floorplanning.

```
Global placement (rough, analytical positioning, may have minor overlaps)
        |
        v
Legalization (remove overlaps, snap cells to legal row/site positions)
        |
        v
Detailed placement (local optimization/refinement for timing and congestion)
```

## 10.3 Why It Matters

Placement is the stage where **most of the actual physical distance between communicating gates gets determined** — this is the single biggest lever over real wire/route delay (Section 8.10's point about wire delay dominance applies directly), making placement quality one of the most consequential steps for achieving timing closure.

## 10.4 Inputs

- The floorplan with power grid (Sections 8-9's outputs).
- The (DFT-inserted) netlist and timing constraints (SDC, Part 1 Sections 6-7).

## 10.5 Outputs

- **Placed netlist** — every standard cell now has a specific physical location (typically stored in DEF format alongside the floorplan data).
- **Post-placement timing/congestion estimates** — now based on **real physical distances** (much more accurate than pre-placement synthesis estimates, though still not fully final since routing hasn't happened yet).

## 10.6 What Problems Can Appear at This Stage

- **Congestion** — too many cells/nets concentrated in a small physical region, exceeding available routing resource in that area, discovered via congestion maps/reports.
- **Timing-critical paths physically far apart** despite placement's best efforts — sometimes an inherent consequence of poor floorplanning (Section 8) or RTL/synthesis hierarchy choices (Part 1) that placement alone cannot fully overcome.
- **Placement legalization pushing cells away from their analytically-optimal positions** to resolve overlaps, sometimes degrading timing from what a purely analytical (but illegal/overlapping) placement would have predicted.

## 10.7 How It Connects to the Next Stage

Placed cell locations directly feed **Clock Tree Synthesis** (Section 11), which needs to know exactly where every flip-flop's clock pin is located to build a balanced clock distribution network, and subsequently **routing** (Section 12), which must connect every net between the now-fixed cell locations.

## 10.8 Practical Industry Usage

Modern **physically-aware, timing-driven placement** is standard — placement and a form of preliminary/estimated routing/timing analysis are often run together iteratively (sometimes called **placement optimization** or incremental placement refinement loops) to converge on a placement that's genuinely likely to route and time-close well, rather than a purely analytically-optimal-but-unroutable placement.

## 10.9 Verification/Debugging Implications

**Congestion maps** (visual heat-maps of routing demand vs. available resource per region, directly analogous to the FPGA notes' `report_design_analysis -congestion`, Vivado notes Section 9.8) are the primary diagnostic tool at this stage — a "hot" congestion region typically requires either floorplan adjustment (Section 8, if severe) or localized placement density adjustment (utilization target reduction in that region).

## 10.10 Timing Implications

Post-placement timing is meaningfully more accurate than post-synthesis timing (real cell locations are now known), but **still not signoff-final** — actual wire routing (Section 12) and its resulting parasitics (Section 13) remain unknown until later, and can still shift results, particularly for very long or congested nets where the actual routed path may deviate significantly from the straight-line estimate placement tools use internally.

## 10.11 Physical Design Implications

(This entire stage is fundamentally physical design.) Placement density (utilization) directly trades off against timing/congestion outcomes, echoing the same principle from floorplanning (Section 8.5) at a finer-grained level — very locally dense placement regions are common sources of both congestion and IR drop (Section 9) hotspots.

## 10.12 How to Explain It in an Interview

*"Placement assigns physical locations to every standard cell within the floorplan, using timing-driven algorithms that try to place communicating, timing-critical cells physically close together to minimize wire delay, while respecting congestion and power grid constraints. It's one of the most consequential stages for timing closure because it determines the real physical distances signals must travel, which is often the dominant component of total path delay at advanced process nodes."*

## 10.13 Interview Questions

**Q (Easy):** What is the difference between global placement and detailed placement?
**A:** Global placement produces a rough, analytically-optimized cell positioning that may have minor illegal overlaps; detailed placement (after legalization removes overlaps) performs finer, local optimization to refine timing and congestion within the constraints of legal, non-overlapping cell positions.

**Q (Medium):** Why can a placement that looks good analytically still cause routing problems?
**A:** Analytical placement algorithms typically estimate wire length/connectivity cost using simplified models (e.g., half-perimeter wire length estimates) rather than fully simulating actual detailed routing — a placement that minimizes this estimated cost can still create local congestion hotspots that the simplified model didn't fully capture, especially in regions with many overlapping high-fanout nets or where legalization had to significantly perturb the analytically-ideal positions.

**Q (Advanced):** Why is placement often run iteratively with preliminary timing/routing estimation rather than as a single, one-pass optimization?
**A:** Placement decisions interact with downstream routing and timing outcomes in ways that simplified analytical cost models can't fully predict upfront — running incremental estimation (e.g., a fast global route estimate, or timing analysis using post-placement Steiner-tree-based wire estimates) after an initial placement pass reveals where the placement's assumptions diverged from reality, allowing targeted refinement (e.g., locally spreading out a congested region, or pulling a specific critical-path cell closer to its timing partner) rather than accepting a placement based purely on first-pass analytical estimates that may prove suboptimal once more realistic feedback is available.

---

# 11. Clock Tree Synthesis (CTS)

## 11.1 What the Stage Is

Clock Tree Synthesis builds the actual physical clock distribution network — a tree (or mesh) of buffers and wiring — that delivers the clock signal from its single source point to every sequential element (flip-flop) across the entire chip, with the specific goal of minimizing **clock skew** (Digital Circuits Section 18.1) and controlling **clock latency/insertion delay**.

## 11.2 Why It Is Needed

Before CTS, the clock is typically just an "ideal" logical signal in the netlist with no physical realization — but a single clock source cannot, in physical reality, reach thousands to millions of flip-flops across an entire die simultaneously; a dedicated, carefully-buffered, balanced distribution tree is required to keep skew within acceptable bounds and to ensure the clock signal itself has clean, sufficiently-fast edges everywhere it's needed (since a single unbuffered wire driving huge fan-out would have severely degraded signal integrity/edge rate over a large die).

## 11.3 How It Works (Conceptual)

```
                     Clock Source (PLL output or clock pin)
                              |
                        [Root buffer]
                        /            \
                  [Buffer]          [Buffer]
                  /       \          /       \
            [Buffer]  [Buffer]  [Buffer]  [Buffer]
              /  \      /  \      /  \      /  \
            FF   FF   FF   FF   FF   FF   FF   FF
```

CTS tools (Cadence Innovus, Synopsys IC Compiler II) build this buffered tree structure automatically, choosing buffer sizes, insertion points, and (critically) the physical routing topology, specifically targeting:
- **Minimizing skew** — ensuring the clock edge arrives at every flip-flop at as close to the same time as possible (Digital Circuits Section 18.1's skew concept, now made concrete and controllable via deliberate, physically-engineered buffer tree design, rather than being an incidental/undesired effect as it might be in an unplanned distribution).
- **Meeting target insertion delay/latency** — the total delay from clock source to each flip-flop, sometimes deliberately tuned to help meet timing (e.g., **useful skew** techniques, discussed below).

## 11.4 Useful Skew (Advanced Concept)

While excessive/uncontrolled skew is harmful (Digital Circuits Section 18.1), **deliberately introduced, controlled skew** can sometimes be used as a timing-closure technique: if a setup-critical path exists between two flip-flops, deliberately delaying the *capturing* flip-flop's clock arrival slightly (positive skew in the direction of data flow) effectively grants that path extra time budget, at the cost of "borrowing" time from the *next* stage's timing budget (and requiring careful, corresponding hold-check verification, since hold timing is affected in the opposite direction, per Digital Circuits Sections 16.6/18.1's skew-hold interaction).

## 11.5 Inputs

- The placed netlist (Section 10's output) — CTS needs to know exact flip-flop locations to build an appropriately-routed tree.
- **Clock tree synthesis constraints** — target skew, target latency, buffer/inverter cell selection guidance from the technology library.

## 11.6 Outputs

- **Clock tree netlist** (buffers inserted, clock routing topology defined) — merged into the overall placed design.
- **Post-CTS clock skew and latency reports.**

## 11.7 What Problems Can Appear at This Stage

- **Excessive residual skew** — if the physical clock distribution (constrained by floorplan/macro obstacles, Section 8) cannot achieve target skew, causing setup/hold timing difficulty on paths crossing regions with mismatched clock arrival times.
- **Clock tree power consumption** — clock trees, since they toggle every single cycle across the entire distribution network, are often one of the **largest single power consumers** in a digital chip; CTS must balance skew/latency optimization against this significant power cost (fewer, larger buffers vs. more numerous, smaller buffers is a real power-vs-skew trade-off).
- **Insufficient buffer insertion points due to congestion/blockages** — macros or heavily congested regions can make ideal clock tree routing difficult, forcing detours that increase latency/skew.

## 11.8 How It Connects to the Next Stage

After CTS, the design proceeds to **routing** (Section 12) for the remaining (non-clock) signal nets — clock nets are typically routed with **higher priority and special rules** (e.g., shielding, dedicated routing layers) either during CTS itself or immediately following it, given the clock signal's outsized importance to overall timing correctness.

## 11.9 Practical Industry Usage

Real designs sometimes use a **clock mesh** (a grid rather than a strict tree) for extremely skew-sensitive, high-performance designs (e.g., high-end CPU cores) — a mesh provides multiple parallel paths to each flip-flop, averaging out local variation for even lower skew, at a significant area/power cost, making it a choice reserved for the most timing-critical designs/regions rather than a universal default.

## 11.10 Verification/Debugging Implications

**Clock tree debugging** typically involves examining **skew reports across the full chip** (identifying specific regions/flip-flop pairs with unexpectedly high skew) and correlating with the physical clock tree routing (is the tree routing through a congested/obstacle-heavy region, causing an unbalanced branch length?).

## 11.11 Timing Implications

(This entire stage is fundamentally a timing-focused topic.) CTS quality directly determines the **clock skew and clock uncertainty** terms used in all subsequent STA (Part 3) — a poorly-built clock tree directly worsens both setup and hold margins across the design, exactly per the Digital Circuits Section 18.1 skew-timing relationship, now made concrete with real, extracted (post-CTS, and later post-route) skew values rather than the generic uncertainty margins used pre-CTS.

## 11.12 Physical Design Implications

Clock tree buffers and their routing consume real area and routing resource, and their placement/routing must navigate the same floorplan/macro/congestion landscape as everything else (Sections 8-10) — CTS is not performed in isolation from the physical realities established earlier in the flow.

## 11.13 How to Explain It in an Interview

*"Clock Tree Synthesis builds the actual physical buffer tree that distributes the clock signal from its source to every flip-flop on the chip, specifically engineered to minimize skew — the difference in clock arrival time at different flip-flops — because excessive skew directly eats into setup and hold timing margins. It's also one of the largest power consumers on the chip since every clock buffer toggles every single cycle, so CTS has to balance skew minimization against power."*

## 11.14 Interview Questions

**Q (Easy):** Why can't the clock signal just be routed as an ordinary wire to every flip-flop?
**A:** A single unbuffered wire driving potentially millions of flip-flops across a large die would have severely degraded signal edge rate (due to the combined capacitive load) and would accumulate very large, uncontrolled, non-uniform delay across the die, causing excessive and unpredictable clock skew — a deliberately-engineered buffered tree is needed to distribute the clock with controlled, balanced delay and clean edges everywhere.

**Q (Medium):** What is useful skew, and why must hold timing be re-checked carefully when it's used?
**A:** Useful skew deliberately introduces controlled clock arrival time differences to grant extra setup timing margin to a specific critical path (by delaying the capturing flip-flop's clock edge relative to the launching one) — but this same skew, applied in the same direction, makes the *hold* timing check on that same path (or potentially an adjacent path sharing the same clock branch) more difficult (Digital Circuits Section 18.1's point that positive skew helps setup but hurts hold), so any intentional skew insertion must be verified not to introduce new hold violations elsewhere.

**Q (Advanced):** Why might a high-performance CPU design use a clock mesh instead of a clock tree, despite the area/power cost?
**A:** A clock mesh provides multiple redundant parallel paths from source to each flip-flop, which averages out local process variation, on-chip variation (OCV), and other non-idealities that would otherwise create localized, unpredictable skew in a strict tree topology (where each flip-flop has exactly one path back to the source, so any local delay variation on that unique path directly and fully affects that flip-flop's skew) — for extremely timing-critical, high-frequency designs where even small skew degradation translates to meaningful frequency loss, the mesh's superior skew robustness can justify its substantially higher area and power cost, whereas for the majority of more moderate-performance designs, a standard buffered tree remains the more efficient default choice.

---

# 12. Routing

## 12.1 What the Stage Is

Routing determines the actual physical wire paths, across the chip's available metal layers, connecting every net in the netlist according to the connectivity established since synthesis — turning the placed-but-unconnected cells (Section 10) plus the clock tree (Section 11) into a fully physically realized, electrically connected circuit.

## 12.2 How It Works (Conceptual)

```
Global routing (coarse-grained: which general region/channel each net will use, congestion-aware planning)
        |
        v
Track assignment (assigning each net to a specific routing track within its planned region)
        |
        v
Detailed routing (exact metal layer, via, and geometric wire path for every net, obeying all DRC rules)
        |
        v
Routed design (fully physically connected, ready for parasitic extraction)
```

Routing uses **multiple metal layers** (modern advanced-node processes commonly have 10+ metal layers), with lower layers typically used for short, local connections (finer pitch, higher resistance per unit length) and upper layers for longer, chip-wide connections (coarser pitch, lower resistance, often used for power routing and long signal nets/clock trees) — layer assignment is itself an important routing decision affecting both timing and congestion.

## 12.3 Why It Matters

Routing is the stage that **finally, fully determines** the actual physical wire geometry (and thus, in combination with parasitic extraction, Section 13, the actual real RC delay) for every net in the design — this is precisely why, as emphasized throughout these notes, only **post-route** timing analysis is considered truly signoff-accurate.

## 12.4 Inputs

- The placed netlist with clock tree (Sections 10-11's outputs).
- **DRC (Design Rule Check) rules** from the foundry PDK — minimum wire width/spacing, via rules, layer-specific constraints that every routed wire must obey to be manufacturable.

## 12.5 Outputs

- **Fully routed design** (complete physical layout, typically in a physical database format like DEF, eventually contributing to the final GDSII, Part 3).
- **Routing/DRC violation reports** — any remaining unresolved rule violations that must be fixed before proceeding.

## 12.6 What Problems Can Appear at This Stage

- **Routing congestion causing incomplete routing** ("unrouted nets") — if the placement/floorplan left insufficient routing resource in some region, the router may simply be unable to complete all connections, requiring a return to placement or even floorplanning (an expensive backward iteration).
- **DRC violations** — spacing/width violations, especially in extremely dense/congested regions where the router is forced into tight, rule-violating geometries to complete a connection.
- **Crosstalk / signal integrity issues** — closely-routed parallel wires can capacitively couple, causing **crosstalk noise** (unwanted signal induced on a "victim" net by a switching "aggressor" net) that can, in severe cases, cause functional glitches or additional timing delay/uncertainty (crosstalk-induced delay effects are a standard, dedicated signoff analysis category, related to but distinct from ordinary RC-delay timing analysis).
- **Antenna violations** — during fabrication, long metal wire segments can accumulate charge during plasma-etching manufacturing steps before the transistor they connect to is fully protected, potentially damaging thin gate oxide; **antenna rules** limit metal length/area ratios relative to connected gate size, and antenna diodes are sometimes added as a fix (this is a manufacturing-process-driven concern, checked as part of signoff DRC, Part 3).

## 12.7 How It Connects to the Next Stage

The fully routed design proceeds to **parasitic extraction** (Section 13), which computes the real electrical (resistance/capacitance) characteristics of every routed wire — the essential input that finally enables fully accurate, signoff-grade STA (Part 3).

## 12.8 Practical Industry Usage

Modern routers (Cadence Innovus, Synopsys IC Compiler II) are highly automated and DRC-correct-by-construction for the vast majority of nets, but **timing-critical nets** often receive special, targeted routing attention (e.g., wider wires or extra shielding to reduce resistance/crosstalk on specific critical paths) — a combination of automated routing plus targeted, timing-driven manual/scripted guidance for the most critical portions of the design.

## 12.9 Verification/Debugging Implications

**Routing DRC** (a distinct check from the later, comprehensive signoff DRC, Part 3, though closely related) is typically run and cleaned up iteratively during/immediately after routing — leaving DRC cleanup until the very end of the flow is poor practice, since early DRC issues often reveal congestion problems better addressed by returning to placement/floorplanning rather than accepting increasingly severe rule-bending as routing gets "squeezed" later.

## 12.10 Timing Implications

(Central to this stage, as emphasized throughout.) Post-route timing, informed by parasitic extraction (Section 13), is the **first fully signoff-accurate timing view** in the entire flow — every prior timing report (synthesis, post-placement) was, to varying degrees, an estimate.

## 12.11 Physical Design Implications

(This entire stage is fundamentally physical design.) Routing layer assignment, wire width, and shielding decisions directly trade off against both area/congestion and timing/signal-integrity — e.g., using a wider wire reduces resistance (helping timing) but consumes more routing resource (potentially worsening congestion elsewhere).

## 12.12 How to Explain It in an Interview

*"Routing determines the actual physical wire paths connecting every net in the design across the chip's metal layers, obeying foundry design rules for manufacturability. It's the stage where the design becomes fully physically complete, and it's why post-route timing — combined with parasitic extraction — is the first truly signoff-accurate timing view in the whole flow, since only now do we know the real wire geometry and thus real wire delay and crosstalk behavior."*

## 12.13 Interview Questions

**Q (Easy):** What is the difference between global routing and detailed routing?
**A:** Global routing makes coarse-grained decisions about which general region/channel each net will use, primarily for congestion planning; detailed routing then determines the exact metal layer, via placement, and precise geometric path for every net, fully obeying manufacturing design rules.

**Q (Medium):** What is crosstalk, and why does it matter in routing?
**A:** Crosstalk is unwanted capacitive (and to a lesser extent inductive) coupling between physically adjacent parallel wires, where a switching "aggressor" net induces noise or additional delay on a nearby "victim" net — it matters because in dense, advanced-node routing, wires are packed close enough together that this coupling can cause functional glitches (if severe enough to flip a logic value) or measurably affect timing (crosstalk-induced delay uncertainty), requiring dedicated signal-integrity-aware routing techniques (extra spacing, shielding wires) and signoff analysis.

**Q (Advanced):** Why might a router fail to complete all connections (leave "unrouted nets"), and why is this often better fixed by returning to an earlier stage rather than forcing the router to try harder?
**A:** Unrouted nets typically indicate that the local routing resource (available tracks across the relevant metal layers in that region) is insufficient for the routing demand created by the placement/floorplan decisions made earlier — simply forcing the router to attempt harder in an already over-congested region tends to produce DRC-violating "illegal" routes (if the tool is configured to prioritize completion over rule compliance) rather than a genuine fix, since the fundamental resource shortage isn't a routing-algorithm limitation but a structural placement/floorplan density issue; the more robust fix is usually to reduce local placement density (spread out the congested region) or, if the congestion is floorplan-driven (e.g., macros placed too close together), revise the floorplan itself.

---

# 13. Parasitic Extraction

## 13.1 What the Stage Is

Parasitic extraction (often called **RC extraction**) computes the actual electrical **resistance (R) and capacitance (C)** — and at very advanced nodes/high frequencies, sometimes **inductance (L)** — of every physically routed wire and via in the design, based on the real, final routed geometry (Section 12) and the foundry's physical/electrical models for each metal layer.

## 13.2 Why It Is Needed

Digital timing analysis (Digital Circuits Section 16, and STA generally, Part 3) fundamentally depends on knowing real signal propagation delay — and at the physical level, this delay is a direct function of the resistance and capacitance of the wire (and the driving gate's output resistance and the receiving gate's input capacitance) — parasitic extraction is what converts the abstract, geometric routed layout into the concrete electrical (RC) values that timing analysis tools actually need to compute delay.

## 13.3 How It Works (Conceptual)

Extraction tools (Synopsys StarRC, Cadence Quantus) analyze the exact 3D geometry of every routed wire segment and via (width, length, thickness, spacing to neighboring wires on the same and adjacent layers) combined with the foundry's process-specific per-layer resistance-per-square and capacitance-per-unit-area/fringe/coupling models, producing a detailed parasitic netlist, typically in **SPEF (Standard Parasitic Exchange Format)**.

## 13.4 Types of Extracted Capacitance

| Capacitance Type | Description |
|---|---|
| **Ground/substrate capacitance** | Capacitance from the wire to the substrate/ground plane |
| **Coupling capacitance** | Capacitance between adjacent wires (the physical basis of the crosstalk phenomenon, Section 12.6) — this is the component that makes accurate extraction particularly important at advanced nodes, where wire spacing is small enough that coupling capacitance can rival or exceed ground capacitance |
| **Fringe capacitance** | Edge-effect capacitance beyond simple parallel-plate approximation, significant for narrow, tall modern wire cross-sections |

## 13.5 Inputs

- The fully routed physical design (Section 12's output) — extraction needs the **real, final** wire geometry, not an estimate.
- Foundry-provided **technology files** describing per-layer physical/electrical characteristics.

## 13.6 Outputs

- **SPEF file** (parasitic data: R and C values for every net segment) — the essential input, alongside the netlist and Liberty timing library, for **signoff STA** (Part 3).

## 13.7 What Problems Can Appear at This Stage

- **Excessive coupling capacitance on tightly-packed nets** — revealing crosstalk risk that may require routing rework (added spacing/shielding, Section 12.6) if it significantly impacts timing or signal integrity.
- **Unexpectedly high resistance on long, narrow, or high-via-count nets** — can reveal that a particular net's routing (e.g., forced through many layer transitions due to congestion) is contributing more delay than anticipated, sometimes triggering a targeted re-route of that specific net.
- **Extraction accuracy/runtime trade-offs** — full, highly detailed 3D field-solver-based extraction is extremely accurate but computationally expensive; faster, simplified extraction models trade some accuracy for speed, and choosing the right extraction accuracy level for a given design/stage (e.g., faster/rougher during early iteration, full accuracy for final signoff) is a real, practical engineering decision.

## 13.8 How It Connects to the Next Stage

The SPEF parasitic data, combined with the gate-level netlist and the Liberty (`.lib`) timing library, is fed directly into **signoff Static Timing Analysis** (Part 3) — this is the definitive, final timing analysis of the entire flow.

## 13.9 Practical Industry Usage

Extraction accuracy requirements have increased dramatically at advanced process nodes (7nm, 5nm, and below), where wire parasitics (particularly coupling capacitance from extremely tightly-packed, tall/narrow modern metal wire cross-sections) dominate total path delay even more than at older nodes — this is a major reason why physical design and timing closure have become progressively more challenging (and schedule-consuming) as process nodes have advanced.

## 13.10 Verification/Debugging Implications

Extracted parasitics are sometimes **cross-checked/correlated** against silicon measurement data from earlier test chips or prior similar designs in the same process node, to validate that the extraction tool/models are accurately predicting real silicon behavior — a real, practical industry concern given how central extraction accuracy is to overall timing closure confidence.

## 13.11 Timing Implications

(This entire stage exists to enable accurate timing analysis.) Extraction is the direct bridge between "we have a physically routed design" and "we can compute real, trustworthy path delays" — without accurate extraction, even a perfectly-routed design cannot be correctly timing-signed-off.

## 13.12 Physical Design Implications

Extraction results sometimes feed back into physical design as an **ECO (Engineering Change Order, Part 3)** trigger — if extraction reveals a specific net's parasitics are significantly worse than the pre-route estimate assumed, and this causes a timing violation, a targeted physical fix (re-route that specific net, adjust its shielding/spacing) may be needed, closing a feedback loop back to routing (Section 12) or even placement (Section 10) for that specific local area.

## 13.13 How to Explain It in an Interview

*"Parasitic extraction computes the real resistance and capacitance of every routed wire based on its actual final physical geometry, producing a SPEF file. This is essential because timing delay is fundamentally a function of these RC values, combined with the driving gate's characteristics — extraction is what turns the physically-routed layout into the electrical data that signoff STA actually needs to compute accurate, trustworthy path delays."*

## 13.14 Interview Questions

**Q (Easy):** What does SPEF stand for and what does it contain?
**A:** Standard Parasitic Exchange Format — it contains the extracted resistance and capacitance values for every net in the routed design, used as direct input to timing analysis tools.

**Q (Medium):** Why has parasitic extraction accuracy become more critical at advanced process nodes?
**A:** As process nodes shrink, wires become narrower and taller (to maintain reasonable resistance despite shrinking width) and are packed more closely together, which increases coupling capacitance relative to ground capacitance — wire-dominated (rather than gate-dominated) delay becomes increasingly significant, meaning even small inaccuracies in extracted parasitics translate into larger relative timing errors compared to older, more gate-delay-dominated process nodes.

**Q (Advanced):** Why might a design team use faster, less accurate extraction during early physical design iterations, but require full, high-accuracy extraction for final signoff?
**A:** Full, detailed (e.g., field-solver-based) extraction is computationally expensive and can take significant runtime on large designs, making it impractical to run after every minor placement/routing iteration during active physical design work when many iterations are needed to converge; faster, simplified extraction models provide "good enough" relative accuracy to guide iterative optimization decisions (is this change trending better or worse) at much lower computational cost, while the final, signoff-quality timing verdict — which directly gates tape-out — requires the full accuracy of detailed extraction to ensure no significant timing risk is being missed due to extraction approximation error.

---

*(End of Part 2 — Floorplanning, Power Planning, Placement, Clock Tree Synthesis, Routing, Parasitic Extraction. Part 3 will cover the Signoff stages: Static Timing Analysis (STA), ECO Basics, Signoff Checks, DRC/LVS Basics, IR Drop and EM Awareness, and GDSII/Tape-out — plus a consolidated interview preparation summary. Reply "continue" or "part 3" to proceed.)*
