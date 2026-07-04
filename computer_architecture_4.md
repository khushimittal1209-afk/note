# Computer Architecture — Deep Study Notes
## Part 4 of 4: Main Memory, Secondary Storage, Virtual Memory, Bandwidth/Latency, Multiprocessing Basics

**Primary references used for this part:**
- Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*, 5th Ed. — Chapter 2 (Memory Hierarchy Design, Section 2.3 Memory Technology), Appendix B (Section B.4, Virtual Memory), and Chapter 5 (Thread-Level Parallelism, Sections 5.1–5.2).
- Supplementary framing in the style of Patterson & Hennessy's *Computer Organization and Design* and Neso Academy's COA lecture sequence.

**Series roadmap:**
- Part 1: Architecture fundamentals, performance metrics, ISA, CPU datapath, control unit, assembly-level view. *(done)*
- Part 2: Pipelining, pipeline hazards, hazard detection & forwarding, branch handling & prediction. *(done)*
- Part 3: Locality of reference, memory hierarchy, cache memory, mapping, replacement, write policies. *(done)*
- **Part 4 (this file):** Main memory & secondary storage → Virtual memory basics → Memory bandwidth & latency → Multiprocessing basics.

---

## 1. Main Memory and Secondary Storage

### 1.1 What It Is
Main memory (DRAM) is the memory-hierarchy level directly below cache — large enough to hold entire active programs and their data, but far slower than SRAM-based cache. Secondary storage (magnetic disk, or SSD/Flash in modern systems) sits below main memory — vastly larger and cheaper per bit, but far slower still, and is the backing store for virtual memory (Section 2).

### 1.2 How It Works — SRAM vs. DRAM
| Property | SRAM (used for cache) | DRAM (used for main memory) |
|---|---|---|
| Storage cell | ~6 transistors per bit | 1 transistor + 1 capacitor per bit |
| Needs refresh? | No — access time ≈ cycle time | Yes — reading destroys the stored charge, so data must be **restored after every read**, and every row must be periodically **refreshed** (typically within an ~8 ms window) or the charge leaks away and data is lost |
| Access time vs. cycle time | Nearly identical | Cycle time traditionally **longer** than access time, because of the mandatory restore/refresh overhead |
| Relative speed/cost | Faster, more expensive per bit | Slower, far cheaper per bit — enables much larger capacity |

**DRAM internal organization:** conceptually a **rectangular matrix** addressed by row and column. To reduce address pin count, DRAMs traditionally multiplex the address: half sent via **RAS (Row Access Strobe)**, the other half via **CAS (Column Access Strobe)**. Reading an entire row into an internal row buffer (needed anyway, since the whole row must be refreshed together) enabled a major bandwidth innovation: once a row is buffered, **multiple column accesses can be served from that same row** without paying the full row-access latency again — the ancestor of modern "burst mode" transfers.

### 1.3 DRAM Evolution — A Concrete Numerical Trend
| Year | Chip size | Type | Row access (slowest, ns) | Column access/transfer (ns) | Cycle time (ns) |
|---|---|---|---|---|---|
| 1980 | 64 Kbit | DRAM | 180 | 75 | 250 |
| 1996 | 64 Mbit | SDRAM | 70 | 12 | 110 |
| 2000 | 256 Mbit | DDR1 | 65 | 7 | 90 |
| 2006 | 2 Gbit | DDR2 | 50 | 2.5 | 60 |
| 2012 | 8 Gbit | DDR3 | 30 | 0.5 | 31 |

**Key quantitative insight worth remembering for exams:** row-access time (latency) improved only **~5% per year** historically, while column-access/transfer time (bandwidth) improved **more than twice as fast** — a direct, textbook-sourced illustration of the general architectural truism that **bandwidth is much easier to improve than latency**. This is exactly why so much DRAM innovation (wider buses, multiple banks, double-data-rate transfers, burst mode) targets bandwidth specifically, while raw latency has remained comparatively hard to shrink.

### 1.4 Key DRAM Innovations (Why Modern DDRx Names Exist)
1. **Row buffering / burst mode:** repeated column accesses to an already-open row, without re-paying row-access latency.
2. **Synchronous DRAM (SDRAM):** added a clock signal to the DRAM interface, eliminating per-transfer resynchronization overhead.
3. **Wider interfaces:** from 4-bit transfer width historically up to 16-bit buses in DDR2/DDR3.
4. **Double Data Rate (DDR):** transferring data on **both the rising and falling clock edge**, doubling peak bandwidth for the same clock frequency — the origin of the "DDR" name itself.
5. **Banking:** splitting a DRAM chip into multiple independent banks (2–8 in DDR3) so one bank's refresh/precharge can overlap with another bank actively transferring data — improving effective bandwidth and reducing idle time.

### 1.5 Secondary Storage — Disk and Flash
- **Magnetic disk:** rotating platters; access time dominated by mechanical seek + rotational latency, on the order of **milliseconds** — roughly 5–6 orders of magnitude slower than DRAM access time. Vastly cheaper per bit than DRAM (historically ~$0.09/GB vs. $20–40/GB for SDRAM in the textbook's reference year).
- **Flash memory (NAND):** a type of EEPROM; **non-volatile** (retains data without power); must be **erased in whole blocks** before being overwritten (unlike DRAM's byte/word-level writability) — meaning a partial-block write requires reading, merging, and rewriting the entire block; has a **limited write-cycle endurance** (historically ~100,000 writes per block), motivating **wear-leveling** to spread writes evenly; sits performance-wise **between DRAM and disk** — far faster than disk, but still meaningfully slower than DRAM, especially for writes.

### 1.6 Numerical / Solved Problem — Relative Access Times
Using the textbook's own comparative figures: a 256-byte transfer takes roughly **6.5 μs** from high-density Flash (burst mode), about **one-quarter** of that from DDR SDRAM (~1.6 μs), and roughly **1000× longer** from disk (~6.5 ms) for a comparable transfer. This single comparison is a powerful, concrete anchor for reasoning about *why* Flash has become the dominant "level 3.5" of the hierarchy in personal mobile devices and SSD-based systems — it fills the enormous latency gap between DRAM and rotating disk.

### 1.7 Dependability — Why Large Memories Need Error Correction
Larger caches and main memories increase exposure to **soft errors** (transient bit-flips, e.g., from cosmic ray strikes — a change in stored *data*, not a permanent circuit defect) and, less frequently, **hard errors** (permanent cell/circuit failures). Standard mitigations:
- **Parity:** 1 bit per ~8 data bits, detects (but cannot correct) a single-bit error — adequate for read-only instruction caches.
- **ECC (Error Correcting Code):** typically ~8 bits per 64 data bits, can detect two errors and correct one — standard for larger data caches and main memory.
- **Chipkill (IBM)/SDDC (Intel):** a RAID-like scheme distributing data and ECC information across multiple chips so that the **complete failure of an entire memory chip** can still be recovered from the survivors — essential at warehouse-scale (the textbook's own analysis: a 10,000-processor system might see one unrecoverable error every ~17 minutes with parity alone, every ~7.5 hours with plain ECC, but only about once every ~2 months with Chipkill).

### 1.8 Diagrams
```
DRAM internal structure (conceptual):
        ┌─────────────────────────────┐
Row addr│                             │
───────►│      Memory array (rows)     │──► row buffer (holds one full row
        │                             │      after RAS + activate)
        └─────────────────────────────┘
                    │
             Column addr (CAS) selects
             bytes within the buffered row
                    │
                    ▼
              Data out (burst transfer,
              possibly on both clock edges = DDR)
```

### 1.9 Connections to Other Topics
- Main memory is exactly the "next level down" from cache in the Part 3 hierarchy — every AMAT/miss-penalty formula from Part 3 has its concrete physical basis in the row/column access timing described here.
- Bandwidth-vs-latency asymmetry (Section 1.3) directly explains cache design choices from Part 3 — e.g., **larger cache block sizes** are specifically attractive because they amortize a DRAM's relatively fixed row-access latency over more bytes, exploiting the DRAM's comparatively strong bandwidth.
- Flash's erase-before-write, block-level write behavior is the direct hardware reason **write-back caching and OS-level wear-leveling/log-structured filesystems** matter so much for Flash-backed storage systems.

### 1.10 How It Appears in Modern Processors
Modern systems use DDR4/DDR5 DRAM (direct descendants of the DDR1/2/3 progression in Section 1.3, still bound by the same fundamental row/column/refresh physics), NVMe SSDs (Flash-based, following exactly the block-erase/wear-leveling constraints of Section 1.5), and near-universal ECC in server-class main memory. Graphics-specific GDDR memory (a DDR variant with much wider interfaces, e.g., 32-bit vs. 4/8/16-bit) exists specifically because GPUs demand far higher per-chip bandwidth than general-purpose CPUs.

### 1.11 Common Pitfalls
- Assuming "more DRAM capacity" is a free win — capacity, latency, and bandwidth improve at very different historical rates (Section 1.3), and simply adding capacity does nothing for the processor-memory latency gap that motivates caching in the first place.
- Confusing DRAM refresh (a routine, expected overhead) with an error condition — refresh is a normal, scheduled operation, not a fault.
- Treating Flash like DRAM for write patterns — Flash's block-erase requirement and limited write endurance make naive, DRAM-style random small writes a real performance and lifetime hazard.

### 1.12 Key Terms & Definitions
- **RAS/CAS (Row/Column Access Strobe):** the two halves of a multiplexed DRAM address.
- **Refresh:** the periodic re-read/rewrite of every DRAM row needed to prevent charge leakage from destroying stored data.
- **SDRAM / DDR:** Synchronous DRAM / Double Data Rate — DRAM interface innovations targeting bandwidth.
- **Burst mode:** transferring multiple sequential words from an already-open row without repeating full row-access latency.
- **Soft error / hard error:** a transient bit-flip vs. a permanent circuit defect.
- **ECC / Chipkill:** error-correcting mechanisms of increasing scope (bit-level vs. whole-chip failure).
- **Wear-leveling:** spreading Flash writes evenly across blocks to extend device lifetime.

---

## 2. Virtual Memory Basics

### 2.1 What It Is
Virtual memory is the mechanism, jointly implemented by hardware and the operating system, that automatically manages the two lowest levels of the memory hierarchy — **main memory and secondary (disk) storage** — giving every running process the illusion of its own large, contiguous, private address space, while transparently keeping only the actively-used portion in physical DRAM.

### 2.2 Why It Exists — Beyond Just "More Memory"
Contrary to a common assumption, virtual memory's *original* motivation was **not** sharing/multiprogramming — it was relieving programmers of the burden of manually managing **overlays** (manually swapping program pieces in and out of limited physical memory) when a program didn't fit. Today it serves three distinct, essential purposes simultaneously:
1. **Automatic hierarchy management** between DRAM and disk (the "big memory illusion").
2. **Protection** — restricting each process to only its own allocated blocks, essential for safely multiprogramming multiple processes.
3. **Relocation** — letting the same program run at any physical memory location, since the mapping (not the program's own addresses) determines where it actually lives.

### 2.3 How It Works — Same Four Questions, New Answers
Virtual memory reuses the exact same four-question cache framework from Part 3, but the answers differ because the miss penalty (a disk access) is **so** much larger than a cache miss:

| Question | Cache answer (Part 3) | Virtual memory answer |
|---|---|---|
| Q1: Where can a block go? | Direct/set-associative/fully associative | **Always fully associative** — a page can go anywhere in main memory, since the miss penalty (disk access) is so catastrophically high that minimizing miss rate always wins over placement simplicity |
| Q2: How is a block found? | Tag comparison | A **page table**, indexed by virtual page number, storing each page's physical location; a **TLB (Translation Lookaside Buffer)** caches recent translations to avoid a second memory access on every reference |
| Q3: Which block is replaced? | LRU/Random/FIFO | Essentially always an **LRU approximation**, using hardware **reference/use bits** the OS periodically samples — again because the miss penalty is so large that a smarter (if slower, software-driven) decision is worth the extra time |
| Q4: What happens on a write? | Write-through or write-back | **Always write-back** — no real system writes through all the way to disk on every store; a **dirty bit** tracks which pages actually need to be written back on eviction |

### 2.4 Paging vs. Segmentation
| Property | Paging | Segmentation |
|---|---|---|
| Block size | Fixed (typically 4–8 KB) | Variable |
| Address structure | Single number (page + offset) | Two parts (segment + offset) |
| Visible to programmer? | No | Sometimes |
| Replacing a block | Trivial (all blocks same size) | Hard (must find a contiguous free region of the right, variable size) |
| Fragmentation type | **Internal** (unused space within a page) | **External** (unused, scattered gaps between segments) |

Because of the *replacement* difficulty (finding contiguous variable-size free space is genuinely hard, and got harder as memory filled with fragments), **pure segmentation has fallen out of favor**; most modern systems use pure paging, or occasionally a hybrid ("paged segments," where a segment is composed of an integral number of pages, combining a segment's logical-unit convenience with a page's trivial-to-replace simplicity).

### 2.5 The Page Table and Address Translation
A virtual address splits into a **virtual page number** and a **page offset**. The virtual page number indexes a **page table**, which stores the corresponding **physical page frame number**; the offset is then simply concatenated (unchanged) onto that physical frame number to form the full physical address.

**Numerical example — page table size:** for a 32-bit virtual address, 4 KB (2^12-byte) pages, and 4-byte page table entries:
```
Number of virtual pages = 2^32 / 2^12 = 2^20
Page table size = 2^20 entries × 4 bytes/entry = 2^22 bytes = 4 MB
```
Since a full page table can itself be very large (and is normally stored in main memory, sometimes even paged itself!), some designs use an **inverted page table** — hashed by *physical* page number instead, sized to the number of physical pages rather than virtual pages, which can be dramatically smaller when physical memory is much smaller than the full virtual address space.

### 2.6 The TLB (Translation Lookaside Buffer)
Since every single memory reference would otherwise require *two* memory accesses (one to the page table for translation, one for the actual data) if the page table lived only in main memory, hardware exploits locality (Part 3, Section 1) by caching recent virtual-to-physical translations in a small, fast, typically **fully-associative** structure: the TLB. Each TLB entry behaves like a miniature cache entry — a tag (portion of the virtual address), plus data (physical page frame number, protection bits, valid bit, and usually a use/reference bit and dirty bit for that page). A TLB hit avoids the extra page-table memory access almost entirely; a TLB miss requires walking the page table (in hardware or software, depending on the architecture) before the reference can complete.

### 2.7 Numerical / Solved Comparison — Cache vs. Virtual Memory Parameters
| Parameter | First-level cache | Virtual memory |
|---|---|---|
| Block (page) size | 16–128 bytes | 4096–65,536 bytes |
| Hit time | 1–3 clock cycles | 100–200 clock cycles |
| Miss penalty | 8–200 clock cycles | 1,000,000–10,000,000 clock cycles |
| Miss rate | 0.1–10% | 0.00001–0.001% |

This table makes the core design-philosophy difference between the two levels completely concrete: **virtual memory miss penalties are 4–6 orders of magnitude larger than cache miss penalties** — exactly why virtual memory can afford (and indeed *requires*) fully-associative placement, OS-driven LRU approximation, and write-back-only policy, none of which would necessarily be the right economic trade-off for a fast SRAM cache with a much smaller miss-penalty gap (Part 3, Section 4.6's direct-mapped-vs-2-way example is a perfect contrast: there, hit-time cost dominated because the miss penalty gap was comparatively small).

### 2.8 Diagrams
```
Virtual address translation:
 [ Virtual Page Number | Page Offset ]
              │
              ▼
      ┌───────────────┐
      │  TLB (fast,    │──hit──► physical page frame ──► + offset ──► Physical address
      │  fully assoc.) │
      └───────────────┘
              │ miss
              ▼
      ┌───────────────┐
      │  Page table    │──► physical page frame (or: page fault, page is on disk)
      │  (in memory)   │
      └───────────────┘
```
```
Page fault handling (very different from a cache miss — handled in software, by the OS):
  reference to a page NOT in main memory
        │
        ▼
  OS traps, selects a victim page (LRU-ish, via reference bits),
  writes it back to disk if dirty, reads the needed page from disk
  (millions of cycles), updates page table + TLB, resumes the process
  -> processor typically switches to another task during this wait,
     since it is far too long to simply stall on (unlike a cache miss).
```

### 2.9 Connections to Other Topics
- The entire four-question framework here is a direct, deliberate re-application of Part 3's cache design questions — recognizing this parallel is one of the strongest ways to retain both topics together rather than as separate facts.
- Virtual memory's protection role is the software/hardware foundation that makes safe **multiprogramming** possible — the direct segue into Section 4's multiprocessing basics, where multiple independent processes/threads must be safely isolated (or deliberately share memory) across multiple cores.
- TLB behavior is governed by the exact same locality principles from Part 3, Section 1 — a TLB miss is expensive precisely because it needs a further memory access, so TLB design (size, associativity) follows the same cost/benefit logic as any other cache.

### 2.10 How It Appears in Modern Processors
Every general-purpose OS/CPU combination (Linux/Windows/macOS on x86, ARM, RISC-V) uses paged virtual memory with hardware TLBs, typically multi-level page tables (to avoid the Section 2.5 "4 MB single-level table" cost for large virtual address spaces) and hardware page-table-walking units. Modern virtualization (virtual machines) builds an *additional* layer of address translation (guest-virtual → guest-physical → host-physical) directly on top of this same paging mechanism, motivating hardware features like nested/extended page tables.

### 2.11 Common Pitfalls
- Assuming virtual memory replacement is hardware-controlled like a cache — it is **primarily OS-controlled**, precisely because the enormous miss penalty justifies spending more (software) time making a good decision.
- Forgetting that a TLB miss is *not* the same thing as a page fault — a TLB miss just means the translation isn't cached (the page table walk can still find the page in main memory quickly); a **page fault** means the page itself isn't in main memory at all and must be fetched from disk — a vastly more expensive event.
- Confusing internal fragmentation (paging's cost) with external fragmentation (segmentation's cost) — they're opposite failure modes with opposite mitigation strategies.

### 2.12 Key Terms & Definitions
- **Page / page fault:** virtual memory's equivalent of a cache's block / miss.
- **Page table:** the data structure mapping virtual page numbers to physical page frames.
- **TLB (Translation Lookaside Buffer):** a fast, usually fully-associative cache of recent page-table translations.
- **Inverted page table:** a hashed page table sized by physical (not virtual) page count.
- **Segmentation:** variable-size-block virtual memory, largely superseded by paging due to replacement difficulty.
- **Relocation:** the ability to run the same program at any physical memory location via address translation.

---

## 3. Memory Bandwidth and Latency

### 3.1 What It Is
**Latency** is the time for one memory request to complete (how long you wait); **bandwidth** is the rate at which data can be transferred once a transfer is underway (how much data flows per unit time). These are related but genuinely distinct performance metrics — directly echoing the response-time-vs-throughput distinction for CPU performance from Part 1, now applied specifically to the memory system.

### 3.2 How It Works — Access Time vs. Cycle Time
With burst-transfer memories (essentially all modern DRAM and Flash), latency itself splits into two measures:
- **Access time:** time between a read request and the arrival of the *first* requested word.
- **Cycle time:** the minimum time between two *unrelated* requests to memory (often longer than access time for DRAM, due to the internal restore/refresh requirement discussed in Section 1.2).

**Whose problem is which, traditionally:** main memory *latency* primarily affects **cache miss penalty** (a single core waiting on a single request) — this is squarely the cache designer's concern from Part 3. Main memory *bandwidth* primarily matters for **multiprocessors and I/O**, where many simultaneous requests compete for the same memory system — a direct preview of Section 4's multiprocessing concerns.

### 3.3 Why Bandwidth Is Easier to Improve Than Latency
As already quantified in Section 1.3, historical DRAM row-access latency improved only ~5%/year, while column-access/bandwidth-related timing improved more than twice as fast. The reason: bandwidth can be attacked structurally — **wider buses, multiple independent banks, burst transfers, double-data-rate signaling** — all of which add parallelism or overlap without needing to speed up the fundamental (physics-limited) DRAM cell access itself. Reducing raw latency, by contrast, generally requires attacking the DRAM cell's fundamental physical read/restore process directly, which is a much harder problem.

### 3.4 Techniques That Trade Latency for Bandwidth (or Vice Versa)
| Technique | Effect |
|---|---|
| Wider memory bus / DIMM organization | Higher bandwidth per request |
| Multiple DRAM banks | Overlap one bank's precharge/refresh with another's active transfer → higher effective bandwidth |
| Burst mode transfers | Amortize one row-access latency over many sequential column accesses → higher bandwidth per access |
| Double Data Rate (DDR) | Transfer on both clock edges → doubles peak bandwidth at the same clock frequency |
| Larger cache block size (Part 3) | Directly exploits available DRAM bandwidth, at the cost of a larger miss penalty for any one miss (more bytes to transfer) |

### 3.5 Numerical / Solved Example — Bandwidth Scaling Across DDR Generations
| Standard | Clock rate (MHz) | Transfers/sec (millions) | Peak bandwidth per DIMM (MB/s) |
|---|---|---|---|
| DDR-266 | 133 | 266 | 2,128 |
| DDR2-800 | 400 | 800 | 6,400 |
| DDR3-1600 | 800 | 1600 | 12,800 |

Notice the clean, direct relationship: peak bandwidth scales essentially linearly with transfers/second (itself double the clock rate for DDR, thanks to Section 3.4's double-data-rate signaling), multiplied by the fixed per-transfer width of the interface — a straightforward, exam-friendly numeric pattern (bandwidth roughly doubles as you move DDR → DDR2 → DDR3 generation to generation, tracking the doubling of transfer rate at each step).

### 3.6 Diagrams
```
Latency vs. Bandwidth (conceptual):

  Latency:  |----- wait -----| DATA
             (time to first byte — improves slowly, ~5%/yr historically)

  Bandwidth: DATA DATA DATA DATA DATA DATA ... (once started, sustained rate)
             (bytes/sec once flowing — improves much faster via width/DDR/banking)
```

### 3.7 Connections to Other Topics
- This is the memory-system-specific instance of the response-time-vs-throughput distinction introduced for CPU performance in Part 1 — the same conceptual split, applied one level of the system down.
- Cache block size decisions in Part 3 are fundamentally a bandwidth-vs-latency trade-off: bigger blocks exploit bandwidth (Section 3.4) but increase the miss penalty (latency-sensitive) for the specific access that caused the miss.
- Bandwidth (not latency) is the dominant concern in the multiprocessor memory systems of Section 4 — many cores' simultaneous requests make aggregate throughput, not any single request's latency, the primary system bottleneck, which is exactly why distributed memory architectures (Section 4.4) exist.

### 3.8 How It Appears in Modern Processors
Modern memory controllers actively exploit bank-level parallelism (issuing new requests to idle banks while other banks are busy refreshing/precharging), wide multi-channel memory buses (dual/quad-channel DDR configurations effectively widening the bus further), and prefetching (Part 3-adjacent technique) to hide latency behind bandwidth-rich, overlapped transfers. GDDR/HBM (High Bandwidth Memory) in GPUs represent the most extreme modern expression of "sacrifice some flexibility/cost for radically higher bandwidth," using very wide (up to 1024-bit+ for HBM stacks) interfaces.

### 3.9 Common Pitfalls
- Treating "faster memory" as a single number — always ask whether a claimed improvement is a latency improvement, a bandwidth improvement, or both; they have different causes and different consumers (single-thread cache-miss-bound code cares about latency; multi-core/streaming workloads care more about bandwidth).
- Assuming bandwidth improvements automatically help single-threaded, latency-bound code — a workload dominated by pointer-chasing (poor spatial locality, Part 3 Section 1) barely benefits from more bandwidth if it can't issue enough independent, overlapped requests to use it.

### 3.10 Key Terms & Definitions
- **Latency (access time):** time to the first word of a response.
- **Bandwidth:** sustained data transfer rate once a transfer is underway.
- **Cycle time:** minimum time between unrelated memory requests (can exceed access time for DRAM).
- **Bank-level parallelism:** overlapping one bank's overhead (refresh/precharge) with another bank's active data transfer.

---

## 4. Multiprocessing Basics

### 4.1 What It Is
Multiprocessing exploits **thread-level parallelism (TLP)** — parallelism identified at the software level (by a programmer, compiler, or OS) across multiple independent instruction streams (threads/processes), each with its own program counter — as opposed to **instruction-level parallelism (ILP)**, which pipelining, forwarding, and branch prediction (Part 2) extract *within* a single instruction stream. A multiprocessor is defined as a set of tightly-coupled processors, usually sharing a single address space, coordinated by a single operating system.

### 4.2 Why It Matters — the Historical Turning Point
Growth in single-core (uniprocessor) performance slowed markedly in the mid-2000s, as chasing additional ILP hit steeply diminishing returns in both silicon area and power efficiency (power/area costs grew faster than the resulting performance). Multiprocessing/multicore became, in the textbook's own framing, essentially **"the only scalable and general-purpose way... to increase performance faster than the basic [uniprocessor] technology allows"** — directly echoed by industry's own public pivot to multicore designs in the mid-2000s as the primary path to further performance gains.

### 4.3 Two Software Models of Thread-Level Parallelism
- **Parallel processing:** a tightly-coupled set of threads collaborating on a *single* task (e.g., splitting a matrix multiply across cores).
- **Request-level parallelism:** multiple, relatively independent processes/requests running concurrently (e.g., a database server answering many independent queries, or general multiprogramming of unrelated applications) — a smaller-scale relative of the cluster/warehouse-scale request-level parallelism used in cloud computing.

### 4.4 Two Hardware Organizations
| Organization | Structure | Also called |
|---|---|---|
| **Centralized shared memory (SMP)** | Small core counts (typically ≤8); all processors share one centralized physical memory with equal, uniform access latency | **UMA** (Uniform Memory Access) |
| **Distributed shared memory (DSM)** | Memory physically distributed among processors/nodes; each core's *local* memory is faster to access than *remote* memory across the interconnect | Sometimes informally contrasted as **NUMA** (Non-Uniform Memory Access) |

Modern multicore chips are internally SMPs (cores on one chip share memory, typically with a shared outer-level cache plus private per-core caches); connecting multiple multicore chips together typically creates a DSM system, since each chip's own attached memory becomes "local," and other chips' memory becomes comparatively "remote."

### 4.5 The Cache Coherence Problem
Caching **private** data (used by only one processor) is unproblematic — behavior is identical to a uniprocessor. Caching **shared** data (used by multiple processors) introduces a new problem: **different processors' caches can end up holding different, inconsistent values for the same memory location.**

**Concrete illustration (write-through cache assumed):**
| Time | Event | Cache A | Cache B | Memory |
|---|---|---|---|---|
| 0 | (initial state) | — | — | 1 |
| 1 | A reads X | 1 | — | 1 |
| 2 | B reads X | 1 | 1 | 1 |
| 3 | A writes 0 to X | 0 | **1 (stale!)** | 0 |

After step 3, if B reads X again from its own cache, it will incorrectly see the old value **1**, even though the true, current value is **0** — this is the cache coherence problem in its simplest possible form.

### 4.6 Formal Coherence Conditions
A memory system is considered coherent if:
1. A processor's own read of X, following its own write to X (with no intervening writes by others), always returns its own written value (ordinary program order — true even on a uniprocessor).
2. A read of X by one processor, following a write to X by *another* processor, eventually returns the new value, once enough time has passed and no further writes intervene.
3. **Write serialization:** all writes to the *same* location, by any processors, are seen in the **same order** by every processor — no processor can see a later write's value and then later see an earlier write's value for the same location.

**Coherence vs. consistency (an important, often-confused distinction):** *coherence* governs reads/writes to the **same** memory location; *consistency* governs the relative ordering of accesses to **different** memory locations, and defines exactly *when* a written value becomes visible to other processors — a subtler, related but distinct property.

### 4.7 Two Classes of Coherence Protocols
| Protocol type | Idea | Typical use |
|---|---|---|
| **Snooping** | Every cache controller "listens" (snoops) on a shared broadcast bus/interconnect and monitors all requests, invalidating or updating its own copy when it sees a conflicting access to a block it holds | Historically the classic choice for bus-connected, small-scale SMPs |
| **Directory-based** | A single, centralized (or distributed, for larger/DSM systems) directory tracks precisely which caches hold a copy of each memory block, and coherence traffic is routed only to the relevant caches rather than broadcast to all | Necessary for larger-scale/DSM systems, where a shared broadcast bus doesn't scale |

### 4.8 Numerical / Conceptual Problem — Why Coherence Matters for Correctness, Not Just Performance
Suppose two threads on two cores both increment a shared counter (`counter++`, i.e., read, add 1, write back) without any coherence protection or synchronization. If both cores read the same stale value of `counter` before either writes back, one increment can be silently lost — this is precisely why real multiprocessors *must* solve the coherence problem in hardware (via snooping or directories) as a baseline correctness requirement, entirely separate from any explicit software-level synchronization (locks, atomics) built on top of it.

### 4.9 Diagrams
```
Centralized shared memory (SMP / UMA):        Distributed shared memory (DSM):
 P1  P2  P3  P4                                [P1+Mem]--[P2+Mem]
  \  |   |  /   (private caches per core)          |   interconnect  |
   \ |   |  /                                  [P3+Mem]--[P4+Mem]
  [ shared cache ]                             (local memory access is fast;
        |                                       remote memory access is slower —
  [ main memory ]  <- uniform access time       non-uniform / NUMA-style access)
   from every core
```

### 4.10 Connections to Other Topics
- Multiprocessing directly reuses Part 3's write-policy vocabulary — the coherence example above explicitly used a **write-through** cache, and write-back caches raise the *same* underlying coherence problem "with some additional but similar complications," per the textbook's own framing, since a dirty, unwritten-back block is even more likely to be silently stale from another processor's point of view.
- Virtual memory's protection role (Section 2.2) is exactly what makes safe multiprogramming across cores possible in the first place — each process/thread's isolated address space, combined with explicit, controlled sharing, underlies the entire multiprocessing software model.
- Amdahl's Law (Part 1, Section 2.4) reappears here in its most famous modern application: the speedup available from adding more cores is fundamentally capped by whatever fraction of a program *cannot* be parallelized — a direct, practical multiprocessing consequence of a Part 1 concept.

### 4.11 How It Appears in Modern Processors
Every modern consumer/server CPU is a multicore SMP by default; large-scale servers/cloud infrastructure extend this to DSM/NUMA architectures connecting multiple multicore sockets. Real cache coherence protocols in shipping hardware (e.g., MESI and its variants — Modified/Exclusive/Shared/Invalid) are direct, concrete refinements of the simple invalidate-on-write coherence idea introduced here. GPUs and warehouse-scale/cloud systems extend the same underlying TLP and request-level-parallelism ideas to vastly larger scales (thousands of cores, tens of thousands of servers), which the source textbook treats as a distinct, larger topic beyond this appendix-level introduction.

### 4.12 Common Pitfalls
- Confusing thread-level parallelism (multiprocessing's domain) with instruction-level parallelism (pipelining/superscalar's domain, Part 2) — they are complementary, not competing, techniques, and modern high-performance cores use both simultaneously.
- Assuming more cores always yields proportionally more performance — Amdahl's Law (Section 4.10) puts a hard, quantifiable ceiling on this for any program with a non-parallelizable portion.
- Treating cache coherence and memory consistency as the same concept — coherence is about *same-location* read/write ordering; consistency is the broader, subtler question of ordering *across different* locations.

### 4.13 Key Terms & Definitions
- **TLP (Thread-Level Parallelism) vs. ILP (Instruction-Level Parallelism):** software-identified parallel threads vs. hardware-extracted parallelism within one instruction stream.
- **SMP / UMA:** Symmetric Multiprocessor / Uniform Memory Access — centralized shared memory with equal access latency from any core.
- **DSM / NUMA:** Distributed Shared Memory / Non-Uniform Memory Access — physically distributed memory with faster local, slower remote access.
- **Cache coherence:** the guarantee that all processors observe a single, consistent view of any given memory location's value and write order.
- **Write serialization:** the requirement that all writes to the same location be seen in the same order by every processor.
- **Snooping / directory-based protocols:** the two major hardware mechanisms for enforcing cache coherence.

---

## Series Wrap-Up
Across all four parts, one throughline should now be clear: computer architecture is a continuous negotiation between **speed, capacity, cost, and correctness**, fought at every level of the system —
- **Part 1** gave you the vocabulary (ISA, CPI, datapath, control unit) and the measurement tools (Amdahl's Law, the CPU performance equation) to reason about any design trade-off quantitatively.
- **Part 2** showed how pipelining turns that datapath into an overlapped assembly line, and how hazards — structural, data, control — are the direct, unavoidable cost of that overlap, mitigated by forwarding, stalling, and prediction.
- **Part 3** went to the memory system, showing how locality of reference justifies an entire hierarchy, and how cache mapping, replacement, and write policies are all different answers to the same four fundamental questions.
- **Part 4** pushed those same four questions one level further down (virtual memory) and one dimension sideways (multiple cores sharing that same hierarchy), closing the loop from single-instruction datapath design all the way to multiprocessor cache coherence.

Revisit the **"Connections to Other Topics"** subsections across all four parts as your best single study tool for exam/interview prep — they are deliberately written to show you the same handful of ideas (locality, the four hierarchy questions, Amdahl's Law, hazard-and-stall reasoning) recurring at every scale of the system.

*(End of Part 4 — series complete.)*
