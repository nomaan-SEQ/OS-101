# Linux Memory Management Internals

## The Core Idea

Memory management is the beating heart of the Linux kernel. Almost every subsystem depends on it — processes need address spaces, filesystems need page cache, networking needs buffers, drivers need DMA-capable memory. The `mm/` directory is where some of the most intricate and performance-critical kernel code lives.

Linux memory management has three major layers: **virtual memory** (giving each process the illusion of its own address space), **physical page allocation** (the buddy system), and **object allocation** (the slab/SLUB allocator for kernel data structures). On top of all this sits the **page cache**, which turns free RAM into a disk cache, making Linux systems appear to use almost all their memory even when idle.

---

## Virtual Address Space Layout (x86-64)

On a 64-bit x86 system, the virtual address space is divided into two halves with a giant hole in the middle:

```
0xFFFF_FFFF_FFFF_FFFF  ┌──────────────────────────────┐
                        │                              │
                        │   Kernel Space (128 TB)      │
                        │   Direct physical mapping    │
                        │   vmalloc area               │
                        │   Module mapping space        │
                        │   Fixmap and early ioremap    │
                        │                              │
0xFFFF_8000_0000_0000  ├──────────────────────────────┤
                        │                              │
                        │   Canonical Hole             │
                        │   (non-addressable ~16M TB)  │
                        │   Any access = fault         │
                        │                              │
0x0000_7FFF_FFFF_FFFF  ├──────────────────────────────┤
                        │                              │
                        │   User Space (128 TB)        │
                        │   Stack (top, grows down)    │
                        │   mmap region                │
                        │   Heap (brk, grows up)       │
                        │   BSS, Data, Text segments   │
                        │                              │
0x0000_0000_0000_0000  └──────────────────────────────┘
```

The **canonical hole** exists because x86-64 only uses 48 bits of the 64-bit virtual address (bits 0-47). Bits 48-63 must be copies of bit 47 (sign extension). This creates two valid ranges and a massive gap in between. With 5-level page tables (enabled on some newer systems), the usable space expands to 57 bits (64 PB per half).

The kernel maps all physical memory starting at `0xFFFF_8888_0000_0000` (the direct map), so it can access any physical page by adding an offset. This is critical for the page allocator and page cache.

---

## 4-Level Page Tables

**Finding a physical address through page tables is like navigating a library: PGD is the building, PUD is the floor, PMD is the aisle, PTE is the shelf — and the offset within the page tells you which book.**

For a 48-bit virtual address, the translation works like this:

```
Virtual Address (48 bits used):
┌───────┬───────┬───────┬───────┬──────────────┐
│ PGD   │ PUD   │ PMD   │ PTE   │ Page Offset  │
│ 9 bits│ 9 bits│ 9 bits│ 9 bits│ 12 bits      │
└───┬───┴───┬───┴───┬───┴───┬───┴──────┬───────┘
    │       │       │       │          │
    ▼       ▼       ▼       ▼          │
 ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │
 │ PGD │→│ PUD │→│ PMD │→│ PTE │─────┘
 │Table│ │Table│ │Table│ │Table│  Contains physical
 │512  │ │512  │ │512  │ │512  │  frame number
 │entry│ │entry│ │entry│ │entry│
 └─────┘ └─────┘ └─────┘ └─────┘
   ↑
   CR3 register (per-process, loaded on context switch)
```

Each level contains 512 entries (9 bits of index), each entry is 8 bytes, so each table is exactly one 4 KB page. The PTE contains the physical page frame number plus permission bits (present, writable, user-accessible, no-execute, dirty, accessed).

| Level | Full Name | Indexes Into |
|-------|-----------|-------------|
| PGD | Page Global Directory | Top-level, pointed to by CR3 |
| PUD | Page Upper Directory | Second level |
| PMD | Page Middle Directory | Third level (can point to 2 MB huge page) |
| PTE | Page Table Entry | Fourth level (points to 4 KB page) |

With **5-level page tables** (CONFIG_X86_5LEVEL), a P4D level is added between PGD and PUD, extending addressable space to 128 PB per half.

---

## Page Allocator: The Buddy System

The buddy system is how Linux manages physical page frames. It maintains free lists of contiguous page blocks in powers of two:

**Think of it like a cash register that keeps bills in denominations: $1 (1 page), $2 (2 pages), $4, $8, $16, up to $1024 (1024 pages = 4 MB). When you need $3 worth of pages, you get a $4 bill and split it. When you return adjacent $2 bills, they merge back into a $4 bill.**

```
Order:  0     1     2     3     4  ...  10
Pages:  1     2     4     8     16      1024
Size:   4KB   8KB   16KB  32KB  64KB    4MB

Free lists (example):
Order 0: [page_A] → [page_B] → [page_C] → ...
Order 1: [pages_D-E] → [pages_F-G] → ...
Order 2: [pages_H-K] → ...
Order 3: (empty)
Order 4: [pages_L-AA] → ...
...
Order 10: [pages_X-...] → ...
```

### Allocation

When you request pages of order *n*:
1. Check the free list for order *n*. If available, return a block.
2. If empty, check order *n+1*. Split the larger block into two "buddies" of order *n*. Return one, put the other on the order *n* free list.
3. Keep going up until you find a free block or run out (allocation failure).

### Freeing (Coalescing)

When freeing a block of order *n*:
1. Check if the buddy block (the adjacent block of the same size) is also free.
2. If yes, merge them into a block of order *n+1* and repeat.
3. If no, just put the block on the order *n* free list.

The "buddy" of a block at physical address *A* of order *n* is at address `A XOR (1 << (n + PAGE_SHIFT))`. This XOR trick makes finding buddies O(1).

```bash
# Check buddy system state:
cat /proc/buddyinfo
# Node 0, zone   Normal   4321  2105  1032   516   241   120   55   28   14   7   3
#                         ord0  ord1  ord2  ord3  ...                          ord10
```

Each number shows how many free blocks of that order exist. If the high-order counts are near zero, the system is fragmented.

---

## SLAB/SLUB Allocator

The buddy system allocates whole pages (minimum 4 KB). But kernel objects like `task_struct` (6 KB), `inode` (1 KB), or `dentry` (192 bytes) need smaller allocations. Enter the slab allocator.

**The slab allocator is like a factory assembly line that pre-manufactures common parts. Need a `task_struct`? Do not start from raw materials (pages) — grab one from the pre-built shelf of task_structs. When you are done, return it to the shelf. The factory keeps shelves stocked for every commonly needed part: inodes, dentries, file descriptors, socket buffers.**

The slab layer sits on top of the buddy system:

```
Application / Kernel Subsystem
         │
         ▼
┌──────────────────────────────┐
│  kmalloc(size) / kmem_cache  │  Object-level allocation
│  SLUB Allocator              │  Caches of pre-built objects
└──────────┬───────────────────┘
           │ Requests whole pages when cache runs low
           ▼
┌──────────────────────────────┐
│  Buddy System (Page Alloc)   │  Page-level allocation
│  alloc_pages(order)          │  Powers-of-two page blocks
└──────────┬───────────────────┘
           │
           ▼
      Physical Memory
```

Linux has had three slab implementations:
- **SLAB**: The original (complex, designed for uniprocessor era).
- **SLUB**: The modern default since 2008. Simpler, better SMP scalability, lower memory overhead. Uses per-CPU freelists to minimize lock contention.
- **SLOB**: Minimal allocator for tiny embedded systems (removed in kernel 6.4).

```bash
# Inspect slab caches:
cat /proc/slabinfo
# name              <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
# task_struct            1250      1344       6080      5            8
# inode_cache            8534      8820       1024      4            1
# dentry                45280     45312       192       21           1
```

Each line shows a cache, how many objects exist, how many are active, and the object size. The `task_struct` cache keeps pre-allocated task_structs ready for `fork()` — no need to kmalloc 6 KB each time.

---

## kmalloc vs vmalloc

| Feature | kmalloc | vmalloc |
|---------|---------|---------|
| Physical contiguity | Yes — physically contiguous pages | No — only virtually contiguous |
| Speed | Fast (direct buddy allocation) | Slower (needs page table manipulation) |
| Max size | Limited (typically a few MB) | Large (limited by address space) |
| Use case | DMA buffers, small kernel objects | Large non-performance-critical allocations |
| Backed by | Buddy system directly | Buddy system + vmalloc page tables |

DMA hardware often requires physically contiguous memory (the device accesses physical addresses directly), which is why `kmalloc` is the workhorse for most kernel allocations.

---

## Page Cache

This is where Linux's memory management becomes brilliant. The page cache uses **all available free RAM** as a cache for file data.

**Page cache is like your browser cache — recently accessed disk data stays in RAM. That is why "free" RAM on Linux always looks low. It is being used as cache, and will be released the instant an application needs it. A Linux system with "only 200 MB free" out of 64 GB is not running out of memory — it has 64 GB of useful data cached, and the kernel will reclaim cache pages on demand.**

```
Application calls read(fd, buf, 4096)
         │
         ▼
    Is the page in the page cache?
        ╱          ╲
      Yes           No
       │             │
       ▼             ▼
  Copy from       Issue disk I/O
  cache to buf    Wait for completion
  (fast!)         Add page to cache
       │          Copy to buf
       ▼             │
    Return           ▼
                  Return
```

For writes, the kernel writes to the page cache and marks the page **dirty**. Background writeback threads (`kworker` running writeback work items) eventually flush dirty pages to disk. This is why:
- Writes appear fast (they only hit RAM)
- Power loss can lose recent writes (dirty pages not yet flushed)
- `fsync()` forces dirty pages to disk (critical for databases)

```bash
# Check memory usage:
cat /proc/meminfo
# MemTotal:       65536000 kB
# MemFree:          204800 kB    <- "Free" (not used for anything)
# MemAvailable:   51200000 kB    <- Actually available (free + reclaimable cache)
# Buffers:          102400 kB    <- Block device metadata cache
# Cached:         50000000 kB    <- Page cache (file data)
# SwapCached:            0 kB
# Active:         40000000 kB    <- Recently used pages
# Inactive:       20000000 kB    <- Candidates for reclaim
```

The critical number is **MemAvailable**, not MemFree. MemAvailable = MemFree + reclaimable page cache + reclaimable slab. This is what applications can actually use.

---

## OOM Killer

When the system is truly out of memory — free pages exhausted, page cache fully reclaimed, swap full or absent — the kernel invokes the OOM (Out-Of-Memory) killer. It must sacrifice a process to free memory, or the entire system hangs.

The OOM killer selects a victim using `oom_score` (visible at `/proc/PID/oom_score`), which considers:
- **Memory consumption**: More memory = higher score
- **Process age**: Older processes score slightly lower (they have been useful longer)
- **Nice value**: Lower-priority processes score higher
- **oom_score_adj**: Admin-tunable value from -1000 (never kill) to +1000 (kill first)

```bash
# Check a process's OOM score
cat /proc/$(pgrep nginx)/oom_score

# Protect a critical process from OOM killing:
echo -1000 > /proc/$(pgrep critical-app)/oom_score_adj

# Make a process more likely to be killed:
echo 500 > /proc/$(pgrep expendable-worker)/oom_score_adj
```

OOM kills are logged in `dmesg`:
```
Out of memory: Kill process 12345 (java) score 850 or sacrifice child
Killed process 12345 (java) total-vm:4096000kB, anon-rss:3500000kB
```

---

## Memory Zones

Physical memory is divided into zones based on hardware addressing limitations:

| Zone | Address Range | Purpose |
|------|--------------|---------|
| `ZONE_DMA` | 0 - 16 MB | Legacy ISA DMA devices (24-bit addresses) |
| `ZONE_DMA32` | 0 - 4 GB | 32-bit DMA devices |
| `ZONE_NORMAL` | Above 4 GB | General-purpose allocations |
| `ZONE_HIGHMEM` | Above ~896 MB | **32-bit only** — memory not permanently mapped into kernel space |

On 64-bit systems, `ZONE_HIGHMEM` does not exist because the kernel can directly map all physical memory. Most modern allocations come from `ZONE_NORMAL`.

The buddy system maintains separate free lists per zone. When allocating, the kernel prefers the zone the caller requested but can fall back to lower zones (a `ZONE_NORMAL` request can be satisfied from `ZONE_DMA32` if necessary, but not the reverse).

---

## Transparent Huge Pages (THP)

Standard pages are 4 KB. For workloads with large memory footprints, using millions of 4 KB pages creates significant TLB pressure. Transparent Huge Pages automatically promote 4 KB pages to **2 MB huge pages** when possible:

- The `khugepaged` kernel thread scans for opportunities to collapse contiguous 4 KB pages into a single 2 MB page.
- The PMD entry points directly to the 2 MB page, eliminating the PTE level for that region.
- One TLB entry covers 2 MB instead of 4 KB — a 512x improvement in TLB coverage.

However, THP has tradeoffs:
- **Memory waste**: If only 8 KB of a 2 MB huge page is actually used, the rest is wasted.
- **Allocation latency**: Finding a contiguous 2 MB block can trigger compaction, causing latency spikes.
- **Page faults are coarser**: A CoW fault on a huge page requires copying 2 MB instead of 4 KB.

```bash
# Check THP status:
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# Set THP to madvise-only (application must opt in):
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
```

---

## Real-World Connection

**Why databases disable THP:** Databases like PostgreSQL, MongoDB, and Redis recommend disabling THP (setting it to `never` or `madvise`). Database access patterns are often random and fine-grained, and the latency spikes from THP compaction during allocation can cause tail latency problems. Databases manage their own memory carefully and prefer predictable 4 KB page behavior.

**Why Java uses -XX:+AlwaysPreTouch:** The JVM allocates a large heap at startup (say, 32 GB) but does not actually fault in the pages until they are first accessed. With THP enabled, the first write to each page triggers huge page allocation, which can stall the application. `-XX:+AlwaysPreTouch` forces the JVM to touch every page at startup, paying the cost upfront rather than during request processing.

**The "my Linux server uses 98% memory" panic:** New Linux admins see `free` output showing almost no free memory and assume the server is overloaded. In reality, the page cache is doing its job — caching recently read file data. `MemAvailable` (or the `available` column in `free -h`) shows the true amount of memory available for new allocations. Educating teams about this is a rite of passage in Linux administration.

---

## Interview Angle

**Q: Explain the Linux page cache and why a Linux server appears to use nearly all its RAM.**

A: Linux uses free RAM as a page cache, storing recently accessed file data in memory. When an application reads a file, the data stays cached — subsequent reads hit RAM instead of disk. This is why `free` shows low "free" memory. The cached pages are reclaimable on demand; the real metric is `MemAvailable`, which includes free memory plus reclaimable cache. A server showing 95% memory "used" with 80% as page cache is healthy — it is using RAM efficiently.

**Q: What is the buddy system and how does it prevent external fragmentation?**

A: The buddy system maintains free lists of page blocks in powers of two (1, 2, 4, 8, ... up to 1024 pages). When allocating, it finds the smallest sufficient block, splitting larger blocks as needed. When freeing, it checks if the adjacent "buddy" block is also free and merges them. This coalescing prevents external fragmentation — free memory does not stay scattered as tiny unusable fragments. You can check fragmentation via `/proc/buddyinfo`.

**Q: What is the difference between SLAB/SLUB and the buddy system?**

A: The buddy system allocates physical page frames (minimum 4 KB). SLUB sits on top of the buddy system and allocates smaller kernel objects (like `task_struct` at 6 KB, `dentry` at 192 bytes). SLUB pre-allocates pages from the buddy system and carves them into fixed-size object caches. This avoids the waste of allocating a full page for a 192-byte dentry. SLUB also reuses recently freed objects, avoiding the overhead of reinitializing them from scratch.

---

**Next**: [04-linux-file-systems-vfs-ext4-proc.md](04-linux-file-systems-vfs-ext4-proc.md)
