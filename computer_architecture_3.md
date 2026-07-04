# Computer Architecture — Deep Study Notes
## Part 3 of 4: Locality, Memory Hierarchy, Cache Memory, Mapping, Replacement & Write Policies

**Primary references used for this part:**
- Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*, 5th Ed. — Appendix B (Review of Memory Hierarchy), Sections B.1–B.2.
- Supplementary framing in the style of Patterson & Hennessy's *Computer Organization and Design* (Ch. 7) and Neso Academy's COA lecture sequence.

**Note on ordering:** the topic list groups these six ideas as Cache Memory → Mapping → Replacement → Write Policies → Locality → Hierarchy. This part covers them in the more pedagogically natural build-up order instead: **Locality of Reference → Memory Hierarchy → Cache Memory basics → Mapping Techniques → Replacement Policies → Write Policies** — each concept is a prerequisite for the next, and the textbook itself follows this same dependency order.

**Series roadmap:**
- Part 1: Architecture fundamentals, performance metrics, ISA, CPU datapath, control unit, assembly-level view. *(done)*
- Part 2: Pipelining, pipeline hazards, hazard detection & forwarding, branch handling & prediction. *(done)*
- **Part 3 (this file):** Locality of reference → Memory hierarchy → Cache memory → Cache mapping techniques → Cache replacement policies → Write policies.
- Part 4: Main memory & secondary storage, virtual memory, memory bandwidth/latency, multiprocessing basics.

---

## 1. Locality of Reference

### 1.1 What It Is
Locality of reference is the empirical observation that **programs don't access memory uniformly at random** — they tend to reuse the same data and instructions repeatedly, and to access nearby memory locations close together in time. It is *the* foundational principle that makes the entire memory hierarchy (and caching in general) work at all — every technique in this part exists to exploit it.

Two distinct flavors:
- **Temporal locality:** if a memory location was referenced recently, it is likely to be referenced again soon (e.g., a loop counter, a frequently called function, a hot variable in a tight loop).
- **Spatial locality:** if a memory location was referenced, nearby locations are likely to be referenced soon after (e.g., sequential instruction fetch, array traversal, struct field access).

### 1.2 How It Works — Why Caching Exploits It Directly
When a cache miss occurs, hardware doesn't fetch just the single requested word — it fetches an entire **block (or line)** of contiguous memory around it. This is a direct, deliberate exploitation of spatial locality: since nearby data is likely to be needed soon, bringing in the whole block up front amortizes the cost of the slow lower-level memory access across many future fast hits. Temporal locality is what justifies *keeping* that block in the fast cache at all — the same block, once fetched, is likely to satisfy several future accesses before it's ever evicted.

### 1.3 Why It's Important
Every performance number in this entire part — hit rate, miss rate, average memory access time — is fundamentally a *measurement of how well a given program exhibits locality*, combined with how well a given cache design is built to exploit it. Without locality, caches would provide essentially no benefit; memory hierarchy design as a discipline would not exist.

### 1.4 Practical Examples
- **Temporal:** a loop variable `i` incremented every iteration; a function called repeatedly in a hot path; the top-of-stack during recursive calls.
- **Spatial:** iterating over an array (`for i in arr: ...`); sequential instruction execution (the PC almost always moves forward by one instruction width, except at branches); reading consecutive fields of a struct/record.
- **Poor locality (cache-hostile) example:** traversing a large linked list scattered randomly across memory, or striding through a large 2D array in the *wrong* dimension (column-major access on a row-major-stored array) — each access likely misses because neither temporal nor spatial locality is present.

### 1.5 Numerical Example
Suppose a program strides through a 2D array of 1000×1000 32-bit integers, stored row-major (each row contiguous in memory), with a cache block size of 64 bytes (16 integers per block).
- **Row-major traversal (`for i: for j: A[i][j]`):** each block of 16 consecutive elements is fully consumed before moving on → excellent spatial locality → roughly 1 miss per 16 accesses (miss rate ≈ 6.25%, ignoring capacity effects).
- **Column-major traversal (`for j: for i: A[i][j]`)** on the same row-major array: consecutive accesses jump 1000 elements (4000 bytes) apart, far outside any one cache block → essentially every access misses (miss rate near 100%, capacity permitting).
This is a completely real, common bug pattern in performance-sensitive code (e.g., naive matrix multiplication) and a favorite "why is my code slow" interview/exam question.

### 1.6 Diagrams
```
Spatial locality (good):           Spatial locality (poor, e.g. column-major on row-major array):
Access sequence: A[0] A[1] A[2] A[3] ...     Access sequence: A[0] A[1000] A[2000] ...
                 └──── one cache block ────┘                  each access -> different, distant block
                 (fetched once, reused 15 more times)          (near-zero reuse per fetch)
```
```
Temporal locality:
time ──►  x  y  z  x  y  z  x  y  z  ...
          (x, y, z reused repeatedly -> stay "hot" in cache)
```

### 1.7 Connections to Other Topics
- Locality is *why* pipelining's forwarding (Part 2) and caching (this Part) both work — both techniques bet on "the near future looks like the recent past/present," just applied to registers vs. memory.
- Cache block size (Section 3) is a direct, tunable knob for how aggressively a design exploits *spatial* locality; associativity and replacement policy (Sections 4–5) are knobs for exploiting *temporal* locality.
- Locality reappears at every level of the hierarchy (Section 2) — the same argument justifying an L1 cache in front of main memory justifies main memory (DRAM) in front of disk, and justifies virtual memory paging (Part 4).

### 1.8 How It Appears in Modern Processors
Every level of a modern cache hierarchy (L1/L2/L3), the TLB (Part 4), branch-prediction structures (Part 2, since recently-taken branches are likely taken again — literally temporal locality applied to control flow), and even OS-level file-system caches and CDN/web caches, are all direct applications of this exact same principle. Compilers actively restructure code (loop tiling/blocking, loop interchange) specifically to *improve* a program's locality for a given cache hierarchy — a real, practical optimization technique directly motivated by the numerical example above.

### 1.9 Common Pitfalls
- Thinking "cache" only means the CPU hardware cache — locality-driven caching is a universal computing pattern (web caches, DNS caches, disk buffer caches).
- Forgetting that locality is a *program property*, not a hardware guarantee — a poorly-structured algorithm (e.g., pointer-chasing through a randomly shuffled linked list) can defeat even a very large, sophisticated cache.
- Confusing spatial locality (nearby addresses) with temporal locality (same address again soon) — they motivate different design decisions (block size vs. replacement policy, respectively).

### 1.10 Key Terms & Definitions
- **Temporal locality:** recently accessed items are likely to be accessed again soon.
- **Spatial locality:** items near a recently accessed item are likely to be accessed soon.
- **Locality of reference:** the umbrella term covering both.

---

## 2. Memory Hierarchy

### 2.1 What It Is
The memory hierarchy is the layered organization of storage technologies — from tiny, extremely fast registers down to enormous, extremely slow disk/secondary storage — designed so that, thanks to locality, the *effective* speed experienced by the processor is close to that of the fastest (smallest, most expensive) level, while the *effective* capacity is close to that of the largest (slowest, cheapest) level.

### 2.2 How It Works — The Canonical Levels
| Level | Name | Typical size | Access time | Bandwidth | Managed by |
|---|---|---|---|---|---|
| 1 | Registers | < 1 KB | 0.15–0.30 ns | 100,000–1,000,000 MB/s | Compiler |
| 2 | Cache (SRAM) | 32 KB–8 MB | 0.5–15 ns | 10,000–40,000 MB/s | Hardware |
| 3 | Main memory (DRAM) | < 512 GB | 30–200 ns | 5,000–20,000 MB/s | Operating system |
| 4 | Disk storage | > 1 TB | ~5,000,000 ns | 50–500 MB/s | OS / operator |

*(Figures are as reported in the textbook's 2006-era reference table — the qualitative pattern of "smaller & faster near the CPU, larger & slower far away" is what matters and remains true today, even as absolute numbers have improved by orders of magnitude — see Section 2.7.)*

**The core design bet:** every level is backed by (i.e., effectively a cache for) the level below it — cache is backed by main memory, main memory is backed by disk. As you move away from the processor, capacity grows, cost-per-bit falls, and access time grows dramatically — by roughly 4–5 orders of magnitude from registers to disk.

### 2.3 Why It's Important
Without a hierarchy, a system would have to choose between: (a) an all-fast, all-small memory (blazing speed, but you can't fit real programs/data), or (b) an all-large, all-slow memory (huge capacity, but every access is catastrophically slow). The hierarchy gives you *both*, at the cost of managing which data lives where — exactly what caches (hardware-managed) and virtual memory (OS-managed) do.

### 2.4 Multilevel Hierarchies — Extending AMAT
Real systems stack multiple cache levels (L1, L2, sometimes L3) between the processor and main memory. The **Average Memory Access Time (AMAT)** formula extends naturally:
```
AMAT = HitTime_L1 + MissRate_L1 × (HitTime_L2 + MissRate_L2 × MissPenalty_L2)
```
Each level "absorbs" most of the misses from the level above it — an L1 miss doesn't necessarily go all the way to main memory; it first checks L2, and only a genuine L2 miss pays the full main-memory penalty. This is the direct multilevel generalization of the single-level AMAT formula covered in Section 3.

### 2.5 Numerical / Solved Problem
A system has L1 hit time = 1 ns, L1 miss rate = 5%; L2 hit time = 10 ns, L2 miss rate (local, i.e. of L1 misses) = 20%; L2 miss penalty (main memory access) = 100 ns. Find AMAT:
```
AMAT = 1 + 0.05 × (10 + 0.20 × 100)
     = 1 + 0.05 × (10 + 20)
     = 1 + 0.05 × 30
     = 1 + 1.5 = 2.5 ns
```
Compare this to a **no-L2** system with the same L1 but going straight to a 100 ns main memory on any L1 miss: `AMAT = 1 + 0.05 × 100 = 6.0 ns` — the L2 cache more than **halves** average access time by catching 80% of what would otherwise be full 100 ns main-memory trips.

### 2.6 Diagrams
```
CPU
 │
 ▼
[ Registers ]  <1 KB,   ~0.2 ns    ▲ faster, smaller, more expensive per bit
 │  miss
 ▼
[ L1 Cache  ]  ~32 KB,  ~1 ns      │
 │  miss                          │
 ▼                                │
[ L2/L3 Cache] ~1-8 MB, ~10 ns    │
 │  miss                          │
 ▼                                │
[ Main Memory (DRAM) ] <512 GB, ~100 ns
 │  page fault                    ▼
 ▼                          slower, larger, cheaper per bit
[ Disk / SSD Storage ] >1 TB, ~ms-scale
```

### 2.7 Connections to Other Topics
- The hierarchy is a direct, physical embodiment of exploiting locality (Section 1) at every possible scale.
- Everything in Section 3 onward (cache basics, mapping, replacement, write policy) is really "how do we manage the *cache level* of this hierarchy well?" — while Part 4 asks the same question one level down, for main memory and virtual memory management.
- The AMAT formula here is the direct ancestor of the CPU-time-with-cache formula (Section 3.5) that ties memory hierarchy performance back to the Part 1 CPU performance equation.

### 2.8 How It Appears in Modern Processors
Modern CPUs universally implement **3 levels of cache** (L1 split I/D-cache per core, L2 per core or per small cluster, L3 shared across all cores) sitting in front of DRAM main memory, which itself sits in front of SSD/HDD secondary storage — exactly this hierarchy, just with dramatically improved absolute numbers (modern L1 access ~1 ns or less, DRAM ~50-100 ns, NVMe SSD ~10-100 μs, HDD ~ms). Register allocation (compiler-managed, per Section 2.2's "managed by" row) remains a first-class compiler optimization problem exactly as the table indicates.

### 2.9 Common Pitfalls
- Assuming "bigger cache is always better" — bigger caches often mean *slower* hit times (more hardware to search), which is exactly why multiple levels exist rather than just one huge L1.
- Forgetting that *every* level of the hierarchy is being managed by *something* (compiler, hardware, OS) — "the hierarchy" isn't automatic; each layer's management policy (register allocation, cache replacement, virtual memory paging) is a real, separately-designed algorithm.
- Ignoring that hierarchy access-time gaps (e.g., DRAM-to-disk) are far larger than cache-level gaps — which is exactly why virtual memory page faults are handled entirely differently (in software/OS, Part 4) than cache misses (in hardware).

### 2.10 Key Terms & Definitions
- **Memory hierarchy:** the layered set of storage technologies from registers to disk, exploiting locality to approximate the speed of the fastest level at the cost of the cheapest level.
- **AMAT (Average Memory Access Time):** the expected time per memory access, accounting for hit time, miss rate, and miss penalty (possibly recursively across multiple levels).
- **Backing store:** the next, larger/slower level that a given level is a "cache" for (e.g., main memory is the backing store for an L1 cache).

---

## 3. Cache Memory

### 3.1 What It Is
A cache is the first (fastest, smallest) level of memory hierarchy the processor encounters — a small, hardware-managed SRAM structure that holds a subset of main memory's contents, chosen (thanks to locality) to be the data most likely to be needed soon.

### 3.2 Key Vocabulary
- **Cache hit:** the requested data is found in the cache.
- **Cache miss:** the requested data is *not* found; a **block (line)** — a fixed-size chunk of contiguous memory containing the requested word — must be fetched from a lower level and placed into the cache.
- **Hit time:** time to access data already in the cache (includes tag comparison).
- **Miss penalty:** the extra time required to service a miss (fetch the block from the next level).
- **Miss rate:** fraction of accesses that miss.

### 3.3 The Four Fundamental Cache Design Questions
Every cache design, at any level of the hierarchy, is fully characterized by how it answers these four questions (this exact framework generalizes cleanly to virtual memory in Part 4 as well):

| # | Question | Answered in |
|---|---|---|
| Q1 | Where can a block be placed? | Section 4 (Mapping Techniques) |
| Q2 | How is a block found? | Section 4 (tag/index/offset) |
| Q3 | Which block is replaced on a miss? | Section 5 (Replacement Policies) |
| Q4 | What happens on a write? | Section 6 (Write Policies) |

### 3.4 How Addresses Map to Cache Structure
A memory address is split into three fields for a set-associative or direct-mapped cache:
```
| Tag | Index | Block offset |
```
- **Block offset:** selects the specific byte/word *within* the block.
- **Index:** selects *which set* of the cache to search (not needed for fully associative caches, which have no index field).
- **Tag:** compared against the stored tag(s) in that set to determine hit/miss.

**Important subtlety directly from the text:** neither the offset nor (for a matched set) the index needs to be stored/compared as part of the tag — the offset is irrelevant to whether the *block* is present (it only picks bytes within an already-identified block), and the index is redundant once you've already used it to pick the set (any block correctly stored in set 0 must, by construction, have an index field equal to 0). This optimization shrinks the tag storage and comparison hardware.

### 3.5 Cache Performance — Connecting Back to Part 1's CPU-Time Equation
```
CPU execution time = (CPU clock cycles + Memory stall cycles) × Clock cycle time
Memory stall cycles = IC × (Memory accesses / Instruction) × Miss rate × Miss penalty
Average memory access time (AMAT) = Hit time + Miss rate × Miss penalty
```

### 3.6 Numerical / Solved Problem — Impact of Cache Misses on CPU Time
**Setup:** ideal CPI (all hits) = 1.0; 50% of instructions are loads/stores; miss penalty = 25 cycles; miss rate = 2%. How much faster would the machine be with a perfect (always-hit) cache?
```
Memory accesses per instruction = 1 (instruction fetch) + 0.5 (data) = 1.5
Memory stall cycles (per instruction, in units of IC) = 1.5 × 0.02 × 25 = 0.75
CPU time (with cache)   = IC × (1.0 + 0.75) × Clock cycle = 1.75 × IC × Clock cycle
CPU time (perfect cache) = IC × 1.0 × Clock cycle
Speedup of perfect cache = 1.75 / 1.0 = 1.75×
```

**A more dramatic version of the same idea**, using measured 30 misses per 1000 instructions instead of a per-reference miss rate (equivalent formulation): CPI grows from an ideal 1.00 to 7.00 with a realistic cache (miss penalty 200 cycles) — and would balloon to roughly **301** with *no* cache/memory hierarchy at all (every access paying the full 200-cycle penalty) — a **40×+** slowdown. This single number is the strongest possible argument for why memory hierarchy design matters as much as (or more than) almost anything else in a modern CPU.

**Second important insight (an "Amdahl's Law strikes again" moment):** cache misses hurt *proportionally more* on processors with low CPI and high clock rates — a fixed number of miss cycles is a *larger fraction* of a fast, efficient pipeline's time than of a slow, inefficient one. This is exactly why cache design gets more, not less, critical as core pipelines get faster and more efficient.

### 3.7 Split (Instruction/Data) vs. Unified Caches
- **Split cache:** separate instruction-cache and data-cache, avoiding the Part 2 structural hazard of a single shared memory port for fetch and load/store in the same cycle. Also allows independently tuning each cache's size/associativity/policy.
- **Unified cache:** one cache serves both instructions and data — simpler, but reintroduces a structural hazard risk when both an instruction fetch and a data access are needed in the same cycle.

**Worked comparison (from the text's own methodology):** for a 74%-instruction / 26%-data reference mix, comparing two 16 KB split caches against one 32 KB unified cache of equal *total* size: the unified cache can have a slightly *better* combined miss rate, yet the split-cache design still wins on **average memory access time** overall, precisely because it avoids the structural-hazard-driven extra cycle unified caches pay when both an instruction and a data access are needed simultaneously.

### 3.8 Diagrams
```
Cache access flow:
   CPU address
        │
        ▼
  ┌───────────────┐    index selects set   ┌───────────────────┐
  │  Tag | Index | │ ───────────────────►  │  compare tag(s)    │──► HIT (data returned)
  │  Offset        │                        │  in selected set   │
  └───────────────┘                        └───────────────────┘
                                                     │ no match
                                                     ▼
                                              MISS -> fetch block
                                              from next level
```

### 3.9 Connections to Other Topics
- Cache design is the direct architectural response to the structural and load-use hazards from Part 2 — split caches specifically exist to avoid a Part-2-style structural hazard.
- Every cache performance number here plugs directly back into the Part 1 CPU performance equation via memory stall cycles — this is the concrete bridge between "abstract performance formula" and "real memory system engineering."
- Cache design questions (placement, identification, replacement, write) recur almost unchanged when discussing virtual memory (Part 4) — a page table is conceptually a fully-associative, software-managed "cache" of the disk's contents in main memory.

### 3.10 How It Appears in Modern Processors
Every modern CPU core has split L1 instruction and data caches (exactly per Section 3.7's reasoning), unified L2/L3 caches (where the structural-hazard concern is less pressing since they're not accessed every single cycle), and dedicated hardware performance counters to directly measure hit/miss rates in real time — precisely the tools needed to apply the Section 3.6-style formulas to real running software.

### 3.11 Common Pitfalls
- Optimizing for miss rate alone — Section 3.7's split-vs-unified comparison is a direct, textbook-sourced counterexample showing a *design with a slightly worse combined miss rate* can still have *better* overall average memory access time and CPU performance.
- Forgetting that "hit time" is itself a real, nontrivial cost paid on **every single access**, not just misses — a cache design that lowers miss rate by adding complexity (e.g., higher associativity) but raises hit time can be a net loss (this is the exact subject of Section 4's set-associative worked example).

### 3.12 Key Terms & Definitions
- **Block / line:** the fixed-size unit of data transferred between cache and the next level on a miss.
- **Hit time / miss penalty / miss rate:** as defined above; the three quantities in the AMAT formula.
- **Split cache / unified cache:** separate vs. shared instruction and data caches.
- **AMAT:** Average Memory Access Time.

---

## 4. Cache Mapping Techniques

### 4.1 What It Is
Cache mapping is the answer to **Q1 (where can a block be placed?)** and directly determines **Q2 (how is a block found?)**. It is the single biggest structural decision in cache design, trading off hardware simplicity/speed against flexibility/miss rate.

### 4.2 How It Works — The Three (or Really, One) Organization
| Type | Rule | Blocks per set |
|---|---|---|
| **Direct mapped** | `(Block address) MOD (Number of blocks in cache)` — each block has exactly **one** possible location | 1 |
| **Set associative** (n-way) | `(Block address) MOD (Number of sets)` selects a *set*; block can go **anywhere within that set** | n |
| **Fully associative** | Block can go **anywhere** in the entire cache | all blocks (= 1 set) |

**Key conceptual unification (directly from the text):** these are not three unrelated designs but **one continuum** — direct mapped is simply "1-way set associative," and fully associative (with m total blocks) is equivalent to "m-way set associative." In practice, the vast majority of real processor caches use **direct-mapped, 2-way, or 4-way set-associative** organizations — the extremes (pure direct-mapped for L1, or fully associative for very small structures like a TLB) are used only in specific niches.

### 4.3 Worked Example — Tracing a Block Placement
Cache with 8 block frames; memory has 32 blocks; consider **block 12** from memory:
- **Fully associative:** block 12 can go into *any* of the 8 frames.
- **Direct mapped:** block 12 can only go into frame `12 MOD 8 = 4`.
- **2-way set associative (4 sets):** block 12 maps to set `12 MOD 4 = 0`; within set 0 (which holds 2 blocks), it can go into either of the 2 frames in that set.

### 4.4 The Address Breakdown, Revisited
As associativity increases (for a *fixed total cache size*), the number of sets shrinks, so the **index field shrinks** and the **tag field grows** correspondingly — at the fully-associative extreme, there is no index field at all (every tag must be checked against every stored block).

**Index size formula (directly usable in exam/interview numeric problems):**
```
2^index_bits = Cache size / (Block size × Set associativity)
```

### 4.5 Worked Numerical Example — the AMD Opteron Data Cache
A real, concrete industry example: a 64 KB (65,536-byte), 2-way set-associative data cache with 64-byte blocks.
```
2^index = 65,536 / (64 × 2) = 512 = 2^9  →  index = 9 bits
Block offset = log2(64) = 6 bits
Physical address width = 40 bits (this example)
Tag width = 40 − 9 − 6 = 25 bits
```
This matches exactly how the four-question framework (Section 3.3) plays out on a shipping, real-world processor design: 512 sets, 2 ways per set, LRU replacement (Section 5), write-back with write-allocate (Section 6).

### 4.6 Trade-off Analysis: Direct-Mapped vs. Set-Associative
Higher associativity generally **lowers miss rate** (more flexibility about where a block can go reduces "conflict misses" — two frequently-used blocks that happen to map to the same set/index no longer necessarily evict each other) but **raises hit time**, because a multiplexor/comparator must select among multiple candidate blocks in the matched set.

**Worked numerical comparison (from the text):** 128 KB caches, 64-byte blocks, CPI (perfect cache) = 1.6, base clock cycle = 0.35 ns, 1.4 memory references/instruction, miss penalty ≈ 65 ns for either design; assume the 2-way design must stretch its clock cycle 1.35× to accommodate the extra selection hardware; direct-mapped miss rate = 2.1%, 2-way miss rate = 1.9%.
```
AMAT(1-way) = 0.35 + 0.021×65 = 1.72 ns
AMAT(2-way) = 0.35×1.35 + 0.019×65 = 1.71 ns      → 2-way looks (barely) better on AMAT

CPUtime(1-way) = IC×[1.6×0.35 + 0.021×1.4×65]  = 2.47×IC
CPUtime(2-way) = IC×[1.6×0.35×1.35 + 0.019×1.4×65] = 2.49×IC   → 1-way is actually (barely) faster!
```
**The critical lesson:** AMAT and full CPU-time comparisons can disagree — here, the 2-way design's *lower miss rate* is more than offset by its *slower clock cycle applying to every single instruction*, not just memory accesses. Since actual CPU execution time is the real bottom-line goal, and direct-mapped is simpler/cheaper to build, the direct-mapped cache is judged the better design in this specific example — a genuinely counterintuitive, important result.

### 4.7 Diagrams
```
Direct mapped (8 frames):        2-way set associative (4 sets × 2 ways):     Fully associative:
block 12 -> frame (12 mod 8)=4   block 12 -> set (12 mod 4)=0, either way     block 12 -> anywhere

Frame: 0 1 2 3 4 5 6 7           Set:    0     1     2     3
                 ▲                     [_ _] [_ _] [_ _] [_ _]
             block 12 goes here          ▲
                                    block 12 -> either slot here
```

### 4.8 Connections to Other Topics
- Associativity directly determines how much replacement-policy choice (Section 5) even matters — a direct-mapped cache (1-way) has *no* replacement decision to make at all (Section 5.2).
- The tag/index/offset address breakdown here is structurally identical to the page-table lookup breakdown used in virtual memory (Part 4) — same four-question framework, same trade-offs, different implementation technology (hardware cache vs. OS-managed page tables).
- Mapping choice interacts directly with the earlier structural-hazard discussion (Part 2): a cache with a multiplexed, multi-cycle hit path (higher associativity) can itself become a new source of pipeline stalls if not carefully designed.

### 4.9 How It Appears in Modern Processors
Modern L1 caches are typically **4-way to 8-way set associative** (balancing hit-time and miss-rate concerns using far more transistor budget than was available in the era of the textbook's Opteron example); L2/L3 caches often use even higher associativity (8-way, 16-way) since their hit time is less on the critical path of every single instruction. Fully-associative structures remain common for small, latency-tolerant structures like TLBs (Part 4) where the total entry count is small enough that fully-associative search is cheap.

### 4.10 Common Pitfalls
- Assuming higher associativity is a "free" improvement — Section 4.6 is a direct, textbook-derived counterexample.
- Forgetting the index/tag-width trade-off — increasing associativity at fixed cache size *shrinks* the index and *grows* the tag, which is a real hardware cost even before considering the extra comparators/multiplexors.
- Miscomputing index bits — always derive them from `Cache size / (Block size × Associativity)`, not from cache size alone.

### 4.11 Key Terms & Definitions
- **Direct mapped:** 1-way set associative; each block has exactly one possible cache location.
- **Set associative (n-way):** each block maps to one set, but can occupy any of n ways within that set.
- **Fully associative:** any block can occupy any cache location; no index field.
- **Conflict miss:** a miss caused by two blocks mapping to the same set/frame, evicting each other, that would not occur with higher associativity.

---

## 5. Cache Replacement Policies

### 5.1 What It Is
The replacement policy answers **Q3**: when a miss occurs in a set/cache that already has candidate blocks (any associativity above direct-mapped), **which existing block gets evicted** to make room for the new one?

### 5.2 How It Works — The Three Standard Strategies
| Policy | Idea | Hardware cost |
|---|---|---|
| **Random** | Pick a candidate block at random (often pseudo-random for reproducible debugging) | Very low |
| **LRU (Least Recently Used)** | Evict the block that hasn't been accessed for the longest time — a direct application of temporal locality's corollary: if recently-used blocks are likely reused, the least-recently-used block is the best eviction candidate | Grows expensive fast as associativity increases; usually only *approximated* in real hardware (**pseudo-LRU**) |
| **FIFO (First In, First Out)** | Evict the oldest-loaded block, regardless of recent access pattern — a cheaper approximation of LRU | Low-moderate |

**Direct-mapped caches have no replacement decision at all** — there is exactly one possible frame for any given block, so "replacement policy" is vacuous (this is precisely why direct-mapped hardware is so simple, as flagged already in Section 4.6).

**A concrete pseudo-LRU scheme** (used in real hardware): keep one bit per way in each set; whenever a way is accessed, its bit is set; if all bits in the set become set, reset all except the most-recently-set one; on a miss, evict from among the ways whose bit is currently *off* (breaking ties randomly if more than one qualifies). This approximates true LRU without needing a full access-order timestamp per block.

### 5.3 Why It's Important
Replacement policy choice measurably affects miss rate, and — thanks to the "double-barreled" impact of misses noted in Section 3.6 (misses hurt low-CPI, high-clock designs proportionally more) — replacement-policy quality genuinely matters for real system performance, though the *size* of that effect shrinks as caches get larger (Section 5.5).

### 5.4 Numerical / Solved Data (SPEC2000, Alpha, 64-byte blocks) — Misses per 1000 Instructions
| Size | 2-way LRU | 2-way Random | 2-way FIFO | 4-way LRU | 4-way Random | 4-way FIFO | 8-way LRU | 8-way Random | 8-way FIFO |
|---|---|---|---|---|---|---|---|---|---|
| 16 KB | 114.1 | 117.3 | 115.5 | 111.7 | 115.1 | 113.3 | 109.0 | 111.8 | 110.4 |
| 64 KB | 103.4 | 104.3 | 103.9 | 102.4 | 102.3 | 103.1 | 99.7 | 100.5 | 100.3 |
| 256 KB | 92.2 | 92.1 | 92.5 | 92.1 | 92.1 | 92.5 | 92.1 | 92.1 | 92.5 |

**Key reading of this table:** LRU consistently beats Random and FIFO at *small* cache sizes (where replacement decisions matter most, because capacity pressure is high), but **the gap essentially disappears at large cache sizes** (256 KB) — with enough capacity, almost nothing meaningful gets evicted prematurely regardless of policy, so the "smarter" policy stops mattering. FIFO generally sits between LRU and Random in the small/medium range.

### 5.5 Diagrams
```
LRU state tracking (2-way set, access sequence A, B, A, C):
  Access A: [A*, - ]     (A becomes MRU)
  Access B: [A , B*]     (B becomes MRU, A now LRU)
  Access A: [A*, B ]     (A becomes MRU again, B now LRU)
  Access C: MISS -> evict B (the LRU block) -> [A , C*]
```

### 5.6 Connections to Other Topics
- Replacement policy is only a meaningful design axis once associativity > 1 (Section 4) — direct-mapped caches skip this question entirely.
- Replacement policy interacts with write policy (Section 6): if the evicted block is **dirty** (modified under a write-back policy), it must be written back to the next level before the frame can be reused — an extra step replacement logic must trigger.
- The same LRU/random/FIFO taxonomy reappears, largely unchanged, for **page replacement in virtual memory** (Part 4) — the core idea (evict the block least likely to be needed soon) is identical; only the vastly larger miss penalty (a page fault vs. a cache miss) changes which policy sophistication is worth the engineering cost.

### 5.7 How It Appears in Modern Processors
Real hardware almost universally implements **pseudo-LRU** (true LRU's per-access bookkeeping becomes prohibitively expensive as associativity rises past 4–8 ways) or, in some designs, simple random replacement for very high-associativity structures. The AMD Opteron example from Section 4.5 explicitly uses **true LRU** at 2-way associativity (cheap enough to track exactly with 2 ways), updating one LRU bit on every access, and evicting whichever way holds the least-recently-referenced block on a miss.

### 5.8 Common Pitfalls
- Assuming LRU is always meaningfully better — Section 5.4's own data shows the benefit shrinks to nearly zero at large cache sizes.
- Forgetting that "true LRU" hardware cost grows rapidly with associativity (tracking a full recency ordering among n ways requires roughly log2(n!) bits, not just n bits) — this is exactly why pseudo-LRU approximations dominate real designs beyond very low associativity.
- Overlooking the write-back interaction (Section 5.6) — replacement isn't "free" the instant a candidate is chosen if that candidate is dirty.

### 5.9 Key Terms & Definitions
- **Replacement policy:** the rule for choosing which cache block to evict on a miss, when more than one candidate exists.
- **LRU (Least Recently Used):** evict the block unused for the longest time.
- **Pseudo-LRU:** a cheaper hardware approximation of true LRU using per-way access bits.
- **FIFO:** evict the oldest-loaded block regardless of access recency.
- **Random replacement:** evict a (pseudo-)randomly chosen candidate block.

---

## 6. Write Policies

### 6.1 What It Is
Write policy answers **Q4**: what does the cache do when the *processor writes* to a location — does it update just the cache, just the lower level, or both, and what happens if the write misses?

### 6.2 Why Writes Are Harder Than Reads
Reads dominate cache traffic (~93% of MIPS-style memory traffic per typical instruction mixes, since *all* instruction fetches are reads and only a minority of instructions are stores) and are the easy case: a read can begin pulling data out of the cache **in parallel with** the tag comparison, since if it turns out to be a miss, no harm is done by having "peeked" at (and discarded) the wrong data. **Writes cannot use this shortcut** — you must confirm a hit *before* modifying anything, since an incorrect speculative write would corrupt data with no way to "unsee" it. This tag-check-then-modify serialization is exactly why writes are structurally slower than reads in cache design.

### 6.3 The Two Write-Hit Policies
| Policy | Behavior | Pros | Cons |
|---|---|---|---|
| **Write-through** | Every write updates *both* the cache and the next lower level immediately | Cache always "clean"; lower level always has the current value (simplifies coherency for multiprocessors/I/O); simpler to implement | Uses much more memory bandwidth (every write, not just evictions, travels down the hierarchy); processor may **write-stall** waiting for the lower-level write, mitigated by a **write buffer** |
| **Write-back** | Writes only update the cache; the modified block is written to the lower level only when it's *evicted* | Writes happen at cache speed; multiple writes to the same block cost only one eventual write-back; much less memory bandwidth/power — attractive for multiprocessors and embedded/low-power designs | Lower level can hold stale data for a while (complicates coherency); requires a **dirty bit** to track which blocks actually need writing back on eviction |

The **dirty bit** is the key optimization that makes write-back efficient: a clean (unmodified) evicted block never needs to be written back at all, since the lower level already has an identical copy.

### 6.4 The Two Write-Miss Policies
| Policy | Behavior |
|---|---|
| **Write allocate** | On a write miss, the block is first loaded into the cache (like a read miss), then the write proceeds as a hit — write misses behave like read misses |
| **No-write allocate** | On a write miss, the block is *not* brought into the cache at all — the write goes directly to the lower level, bypassing the cache entirely |

**Typical real-world pairing (and why):** write-back caches almost always pair with write-allocate (betting that a block just written to will likely be written/read again soon — hoping to capture subsequent accesses in the fast cache). Write-through caches often pair with no-write-allocate (since every write must go to the lower level regardless under write-through, there's less benefit to also paying the cost of pulling the block into the cache).

### 6.5 Numerical / Solved Problem — Write-Allocate vs. No-Write-Allocate
**Setup:** a fully-associative write-back cache starts empty. Access sequence: `Write Mem[100]; Write Mem[100]; Read Mem[200]; Write Mem[200]; Write Mem[100]`.

**No-write-allocate:**
- `Write 100` → miss (not brought into cache)
- `Write 100` → miss (still not in cache)
- `Read 200` → miss (loaded into cache — reads always allocate)
- `Write 200` → hit! (200 is now in the cache from the read)
- `Write 100` → miss (100 was never brought in)
- **Result: 4 misses, 1 hit.**

**Write-allocate:**
- `Write 100` → miss, block loaded
- `Write 100` → hit
- `Read 200` → miss, block loaded
- `Write 200` → hit
- `Write 100` → hit
- **Result: 2 misses, 3 hits.**

**Lesson:** whether write-allocate "wins" depends entirely on whether writes to the same recently-written address recur — here, with repeated writes to address 100, write-allocate captures much more benefit (5 hits vs 1 across the same access sequence... concretely, 3 hits vs 1 hit here).

### 6.6 Diagrams
```
Write-through:                          Write-back:
CPU write ──► Cache (updated)           CPU write ──► Cache (updated, dirty bit SET)
              │                                        │
              └──► Lower level (updated immediately)   └──► Lower level updated
                                                             ONLY on eventual eviction,
                                                             and only if dirty bit is set
```

### 6.7 Connections to Other Topics
- The dirty bit directly interacts with replacement policy (Section 5): evicting a dirty, write-back block requires an extra write-back step (real hardware, e.g. the Opteron example, buffers this via a small **victim buffer** so the eviction's write-back doesn't stall the incoming miss).
- Write-through's "lower level is always current" property is precisely why it's favored for **cache coherency** in multiprocessor systems (a preview of Part 4's multiprocessing basics) — other cores/caches can trust that main memory reflects the latest write.
- Multilevel cache hierarchies (Section 2.4) often use write-through for *upper* (closer to CPU, e.g., L1) levels specifically because the write only needs to propagate to the *next* cache level (e.g., L2), not all the way to main memory — making write-through's bandwidth cost far more tolerable than it would be for a single-level cache talking directly to DRAM.

### 6.8 How It Appears in Modern Processors
The AMD Opteron example (Section 4.5) is real: **write-back, write-allocate**, with a **dirty bit per block** and a dedicated **victim buffer** (8 entries in that design) to hold evicted dirty blocks so their write-back can proceed in parallel with servicing the new miss, rather than stalling it. This write-back + write-allocate + victim-buffer combination remains the standard pattern in modern L1/L2 data caches; L1 write-through to L2 remains common specifically because L2 is "close enough" that the bandwidth cost is acceptable, exactly per Section 6.7.

### 6.9 Common Pitfalls
- Assuming write-back is strictly "better" — it complicates multiprocessor cache coherency (Part 4) precisely because other cores/observers cannot assume main memory is current.
- Forgetting the dirty bit's role — without it, write-back would have to write back *every* evicted block, whether modified or not, erasing much of its bandwidth advantage.
- Mixing up which write-miss policy "naturally" pairs with which write-hit policy — while any combination is technically legal (Section 6.4), the write-back+write-allocate and write-through+no-write-allocate pairings are overwhelmingly the common real-world choices, for the reasons given in Section 6.4.

### 6.10 Key Terms & Definitions
- **Write-through:** every write immediately updates both cache and the next lower level.
- **Write-back:** writes update only the cache; the lower level is updated only on eviction of a modified (dirty) block.
- **Dirty bit:** a per-block status bit marking whether a write-back block has been modified since being loaded.
- **Write allocate:** a write miss first loads the block into the cache, then proceeds as a write hit.
- **No-write allocate:** a write miss bypasses the cache and writes directly to the lower level.
- **Write buffer:** a small FIFO structure letting the processor continue past a write-through write before it actually completes at the lower level.
- **Write stall:** a processor stall caused by waiting for a write-through write (or a full write buffer) to complete.
- **Victim buffer:** a small structure holding evicted dirty blocks so their write-back can proceed without stalling the incoming cache-fill.

---

## Bridge to Part 4
Part 3 covered how the *fastest* level of the memory hierarchy — cache — is organized, populated, and kept consistent with the levels below it. But Section 2's hierarchy table had two more levels below cache: **main memory** and **disk/secondary storage** — and the same four-question framework (placement, identification, replacement, write) reappears there too, just managed by the *operating system* instead of hardware, and at a vastly larger miss penalty (a page fault, not a cache miss). Part 4 covers exactly this: main memory and secondary storage, virtual memory, memory bandwidth/latency, and — since multiple cores now all share this same memory hierarchy — a first look at multiprocessing.

*(End of Part 3. Say "next" for Part 4: Main Memory & Secondary Storage, Virtual Memory Basics, Memory Bandwidth & Latency, and Multiprocessing Basics.)*
