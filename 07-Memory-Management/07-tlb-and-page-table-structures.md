# TLB and Page Table Structures

There's a fundamental performance problem with paging: the page table is stored in memory, so every memory access requires an additional memory access just to translate the address. On x86-64 with 4-level page tables, a single load instruction could trigger **four** extra memory accesses to walk the page table. This would make paging unusably slow — unless we cache the translations. That cache is the TLB.

## The Problem

Without any optimization, reading a single byte from memory works like this:

```
  CPU wants to access virtual address 0x7FFF_1234

  Step 1: Read PML4 entry from memory    (~100 ns)
  Step 2: Read PDP entry from memory     (~100 ns)
  Step 3: Read PD entry from memory      (~100 ns)
  Step 4: Read PT entry from memory      (~100 ns)
  Step 5: Read actual data from memory   (~100 ns)
  ------------------------------------------------
  Total: 5 memory accesses = ~500 ns

  Without TLB, every memory access is 5x slower!
```

This is clearly unacceptable. We need to short-circuit the page table walk for most accesses.

## TLB: Translation Lookaside Buffer

The **TLB** is a small, fast cache inside the CPU that stores recent virtual-to-physical page translations. It's typically part of the MMU.

> **Analogy:** Think of the TLB as a **speed dial** on a phone. The full page table is like a thick phone book -- it has every number, but looking someone up takes time. The TLB stores the numbers you call most often so you can reach them instantly. You only open the phone book (walk the page table) when the number isn't in your speed dial.

```
  CPU generates virtual address
         |
         v
  +------------------+
  |       TLB        |     Fully associative cache
  | (64-1024 entries)|     of recent translations
  +------------------+
         |
    +----+----+
    |         |
  HIT       MISS
    |         |
    v         v
  Use cached    Walk the page table
  frame number  (4 memory accesses on x86-64)
  (~1 ns)       (~400 ns)
                    |
                    v
                Store result in TLB
                for next time
```

### TLB Characteristics

| Property | Typical Value |
|----------|--------------|
| Size | 64-1536 entries (varies by level) |
| Access time | ~1 ns (integrated into pipeline) |
| Associativity | Fully associative (any entry can hold any translation) |
| Hit rate | >99% for most workloads |
| Levels | Often 2: L1 ITLB + L1 DTLB (small, fast) + L2 STLB (larger, slower) |

Modern x86 CPUs typically have:
- **L1 ITLB** (instruction): 64-128 entries for 4 KB pages, 8 entries for 2 MB pages
- **L1 DTLB** (data): 64-128 entries for 4 KB pages, 32 entries for 2 MB pages
- **L2 STLB** (shared/unified): 1024-1536 entries

## Effective Access Time with TLB

```
EAT = hit_rate * (TLB_time + memory_time)
    + (1 - hit_rate) * (TLB_time + page_walk_time + memory_time)
```

Example (4-level page table):
- TLB lookup: 1 ns (overlapped with pipeline, effectively 0)
- Memory access: 100 ns
- Page table walk: 4 * 100 = 400 ns (4 levels)
- TLB hit rate: 99%

```
EAT = 0.99 * (100 ns)  +  0.01 * (400 + 100 ns)
    = 99 + 5
    = 104 ns

Only 4% overhead!  (vs 500 ns = 400% overhead without TLB)
```

With a 99.9% hit rate: EAT = 99.9 + 0.5 = 100.4 ns — virtually no overhead.

This is why TLB hit rate is one of the most important performance metrics for memory-intensive workloads.

## TLB and Context Switches

Each process has its own page table, so TLB entries from one process are invalid for another. When the OS switches processes:

**Option 1: Flush the entire TLB.** Simple but expensive — the new process starts with a cold TLB and suffers misses on every first access. This is the "TLB shootdown" cost of context switching.

**Option 2: Use ASIDs (Address Space Identifiers).** Tag each TLB entry with the process's ASID. The TLB can hold entries from multiple processes simultaneously. On context switch, just change the active ASID — no flush needed.

```
  TLB with ASIDs:
  +------+--------+-----------+-------+
  | ASID | V.Page | Frame     | Flags |
  +------+--------+-----------+-------+
  |  3   | 0x7F00 | 0x1A00   | RW    |   Process 3's mapping
  |  5   | 0x7F00 | 0x2B00   | RW    |   Process 5's mapping (same virtual page!)
  |  3   | 0x4000 | 0x0C00   | RX    |   Process 3's code page
  |  5   | 0x4000 | 0x0C00   | RX    |   Shared library (same frame!)
  +------+--------+-----------+-------+

  Context switch from process 3 to process 5:
  Just change active ASID from 3 to 5 -- no TLB flush needed
```

ASIDs are limited in size (typically 8-16 bits on ARM, variable on x86 with PCID). When you run out of ASIDs, some processes must share and their TLB entries must be flushed on switch. This is managed by the OS.

On x86-64, the equivalent feature is **PCID (Process Context ID)** — 12-bit identifiers (4096 unique IDs). Enabled in Linux since kernel 4.14. Before PCID, every context switch flushed the TLB.

## Multi-Level Page Tables (Detailed)

We introduced multi-level tables in file 03. Here's a deeper look at the x86-64 4-level structure:

```
  CR3 Register
  (holds the physical address of the current process's PML4 table;
   the OS loads a new value into CR3 on every context switch)
       |
       v
  +----------+         +----------+         +----------+         +----------+
  | PML4     |         | PDP      |         | PD       |         | PT       |
  | (512     | ------> | (512     | ------> | (512     | ------> | (512     |
  |  entries) |         |  entries) |         |  entries) |         |  entries) |
  +----------+         +----------+         +----------+         +----------+
  9 bits                9 bits               9 bits               9 bits
  (index 0-511)        (index 0-511)        (index 0-511)        (index 0-511)
                                                                       |
                                                                       v
                                                                  Frame number
                                                                  + 12-bit offset
                                                                  = Physical addr
```

### Why Sparse Address Spaces Work

A typical process uses:
- ~1-10 MB of code (lower virtual addresses)
- ~1-100 MB of heap (grows upward from code)
- ~1-8 MB of stack (top of address space, grows downward)
- Several MB of shared libraries (middle of address space)

That's maybe 200 MB out of 128 TB of usable virtual address space. The vast majority of the address space is unmapped.

With a 4-level page table, **only the table pages that point to actual mappings are allocated**:

```
  PML4 (always allocated):            1 page  = 4 KB
  PDP pages (few entries used):       ~2 pages = 8 KB
  PD pages (sparse):                  ~5 pages = 20 KB
  PT pages (only for mapped ranges):  ~50 pages = 200 KB
  -----------------------------------------------
  Total page table overhead:          ~232 KB

  vs flat page table for 48-bit address space:
  2^36 entries * 8 bytes = 512 GB  (impossible!)
```

## Huge Pages

With 4 KB pages and 64-entry TLB, you can cover only 256 KB of memory before TLB misses start. For a database with 100 GB of data, that's woefully inadequate. **Huge pages** solve this by increasing the page size:

| Page Size | TLB Entries Needed for 1 GB | Coverage with 64 TLB Entries |
|-----------|---------------------------|------------------------------|
| 4 KB | 262,144 | 256 KB |
| 2 MB | 512 | 128 MB |
| 1 GB | 1 | 64 GB |

With 2 MB huge pages, each TLB entry covers 512x more memory. A 64-entry TLB covers 128 MB instead of 256 KB.

### How Huge Pages Work

x86-64 implements huge pages by stopping the page walk early:

```
  2 MB huge page:
  PML4 --> PDP --> PD entry (with "huge page" flag set)
                      |
                      +---> 2 MB physical block directly
                            (skip PT level, use 21-bit offset)

  1 GB huge page:
  PML4 --> PDP entry (with "huge page" flag set)
              |
              +---> 1 GB physical block directly
                    (skip PD and PT levels, use 30-bit offset)
```

The PD entry (for 2 MB) or PDP entry (for 1 GB) directly contains the base address of a large physical block, with a flag indicating it's a huge page.

### Transparent Huge Pages (THP) in Linux

Linux offers two ways to use huge pages:

**1. Explicit huge pages (hugetlbfs):**
- Pre-allocate huge pages at boot time or via sysctl
- Applications must use `mmap()` with `MAP_HUGETLB` or mount hugetlbfs
- Pages are reserved and never swapped
- Used by databases (MySQL, PostgreSQL, Oracle) and VMs (QEMU/KVM)

```
  # Reserve 1024 huge pages (2 MB each = 2 GB)
  $ echo 1024 > /proc/sys/vm/nr_hugepages

  # Check allocation
  $ cat /proc/meminfo | grep Huge
  HugePages_Total:    1024
  HugePages_Free:      800
  HugePages_Rsvd:      200
  Hugepagesize:       2048 kB
```

**2. Transparent Huge Pages (THP):**
- The kernel automatically merges adjacent 4 KB pages into 2 MB huge pages
- No application changes needed — it happens transparently
- The `khugepaged` daemon runs in the background, scanning for merge opportunities
- Can cause latency spikes during compaction (finding contiguous 2 MB regions)

```
  # Check THP status
  $ cat /sys/kernel/mm/transparent_hugepage/enabled
  [always] madvise never

  # Disable THP (recommended for latency-sensitive workloads)
  $ echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

### When to Use Huge Pages

| Workload | Recommendation | Why |
|----------|---------------|-----|
| Databases (PostgreSQL, MySQL) | Explicit huge pages | Large buffer pools, predictable performance |
| Virtual machines (KVM/QEMU) | Explicit or THP | VMs have large contiguous memory |
| Redis, Memcached | Disable THP | THP compaction causes latency spikes |
| Scientific computing | THP or explicit | Large arrays benefit from TLB coverage |
| General-purpose server | THP=madvise | Let apps opt in via madvise() |

Redis and many latency-sensitive applications explicitly recommend **disabling THP** because the background compaction can cause multi-millisecond pauses.

## TLB Lookup Flow (Complete)

```
  CPU generates virtual address
         |
  +------v-------+
  | Split into   |
  | page + offset|
  +------+-------+
         |
  +------v-------+
  | Check TLB    |
  +------+-------+
         |
    +----+------+
    |           |
   HIT        MISS
    |           |
    |    +------v----------+
    |    | Walk page table  |
    |    | (4 memory reads) |
    |    +------+----------+
    |           |
    |    +------v-------+
    |    | Valid entry?  |
    |    +------+-------+
    |      |         |
    |     YES       NO
    |      |         |
    |      |    +----v-------+
    |      |    | Page fault |
    |      |    | (trap to   |
    |      |    |  kernel)   |
    |      |    +------------+
    |      |
    |    +------v-------+
    |    | Load TLB with |
    |    | new entry      |
    |    +------+--------+
    |           |
    +-----+-----+
          |
   +------v-------+
   | Combine frame|
   | + offset     |
   +------+-------+
          |
   +------v-------+
   | Access        |
   | physical mem  |
   +--------------+
```

## Real-World Connection

- **`perf stat` and TLB metrics**: you can measure TLB miss rates with `perf stat -e dTLB-load-misses,dTLB-loads ./your_program`. A high TLB miss rate (>1%) on a data-intensive workload suggests you'd benefit from huge pages.
- **PCID and Meltdown**: the Meltdown CPU vulnerability (2018) required kernel page table isolation (KPTI), which separates user and kernel page tables. Without PCID, this would double the TLB flush cost. PCID made the performance impact of KPTI much smaller.
- **NUMA and TLB**: on multi-socket systems, accessing memory on a remote NUMA node is slower. TLB entries don't distinguish which NUMA node the frame is on — the latency difference only shows up on the actual memory access after translation.
- **Database huge pages**: PostgreSQL recommends configuring `huge_pages = try` in postgresql.conf. Oracle Database's SGA (System Global Area) is typically backed by huge pages. These aren't just optimizations — they can improve throughput by 10-20% for memory-intensive queries.

## Interview Angle

**Q: What is the TLB and why is it necessary?**

The TLB (Translation Lookaside Buffer) is a small, fast cache inside the CPU that stores recent virtual-to-physical address translations. Without it, every memory access on x86-64 would require 4 extra memory accesses to walk the 4-level page table — making every access 5x slower. With a typical hit rate above 99%, the TLB reduces the average translation overhead to near zero.

**Q: What happens to the TLB on a context switch?**

Each process has its own page table, so TLB entries from one process are invalid for another. Without ASIDs/PCIDs, the entire TLB must be flushed on every context switch, causing a burst of TLB misses as the new process warms the cache. With ASIDs/PCIDs, each TLB entry is tagged with the process ID, allowing entries from multiple processes to coexist. This significantly reduces the cost of context switching.

**Q: What are huge pages and when should you use them?**

Huge pages (2 MB or 1 GB instead of 4 KB) allow each TLB entry to cover more memory, dramatically improving TLB hit rates for large-memory workloads. A 64-entry TLB covers 128 MB with 2 MB pages vs only 256 KB with 4 KB pages. They're most beneficial for databases, VMs, and scientific computing with large working sets. However, Transparent Huge Pages (THP) can cause latency spikes during background compaction, so latency-sensitive applications like Redis recommend disabling THP.

**Q: Why do x86-64 systems use 4-level page tables?**

A flat page table for a 48-bit address space would require 512 GB — impossibly large. Multi-level tables are sparse: each level is only 4 KB (512 entries), and table pages are only allocated for address ranges that are actually mapped. A typical process might use only 200 KB of page table memory instead of 512 GB. The 4 levels come from the math: 48 usable address bits minus 12-bit offset = 36 bits of page number, divided into four 9-bit indices.

---

Next: [08 - Thrashing and Working Set Model](08-thrashing-and-working-set-model.md)
