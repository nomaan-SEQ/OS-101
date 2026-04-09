# Contiguous Allocation and Fragmentation

The simplest way to manage memory is the most intuitive: give each process a single contiguous block of physical memory. Process A gets addresses 0-99, Process B gets 100-249, Process C gets 250-349. Simple, fast, and it works — until it doesn't. Understanding why this approach breaks down explains why paging was invented.

## Contiguous Allocation: The Basic Idea

Each process gets one continuous region of physical memory. The OS maintains a table of which regions are allocated and which are free.

```
  Physical Memory
  +------------------+  0
  |   OS Kernel      |
  +------------------+  100KB
  |   Process A      |
  |   (200 KB)       |
  +------------------+  300KB
  |   Process B      |
  |   (150 KB)       |
  +------------------+  450KB
  |     FREE         |
  |   (100 KB)       |
  +------------------+  550KB
  |   Process C      |
  |   (250 KB)       |
  +------------------+  800KB
  |     FREE         |
  |   (200 KB)       |
  +------------------+  1000KB
```

Protection is straightforward: each process has a **base register** (start of its region) and a **limit register** (size of its region). On every memory access, the hardware checks:

```
if (virtual_address >= limit)
    trap: segmentation fault
physical_address = base + virtual_address
```

This is fast and simple. But the problems appear quickly when processes come and go.

## Fixed Partitioning

The simplest variant: divide memory into fixed-size partitions at boot time.

```
  +------------------+
  |   OS             |
  +------------------+
  |  Partition 1     |   256 KB
  |  (Process A)     |
  +------------------+
  |  Partition 2     |   256 KB
  |  (empty)         |
  +------------------+
  |  Partition 3     |   256 KB
  |  (Process B)     |
  +------------------+
  |  Partition 4     |   256 KB
  |  (Process C)     |
  +------------------+
```

**Problems:**
- If a process needs 300 KB but partitions are 256 KB, it can't run.
- If a process needs only 50 KB, the remaining 206 KB in its partition is wasted (**internal fragmentation**).
- The number of concurrent processes is limited to the number of partitions.

Fixed partitioning was used in IBM OS/360 (1960s). It's mostly a historical concept now, but the ideas resurface in embedded systems with static memory budgets.

## Variable Partitioning

Better idea: allocate exactly the amount of memory each process needs.

```
  Initial state:                  After A exits:

  +------------------+            +------------------+
  |   OS             |            |   OS             |
  +------------------+            +------------------+
  |   Process A      |            |     FREE         |
  |   (200 KB)       |            |   (200 KB)       |
  +------------------+            +------------------+
  |   Process B      |            |   Process B      |
  |   (150 KB)       |            |   (150 KB)       |
  +------------------+            +------------------+
  |   Process C      |            |   Process C      |
  |   (250 KB)       |            |   (250 KB)       |
  +------------------+            +------------------+
  |     FREE         |            |     FREE         |
  |   (400 KB)       |            |   (400 KB)       |
  +------------------+            +------------------+
```

No internal fragmentation — each process gets exactly what it needs. But now we have a different problem.

## Allocation Strategies

When a new process arrives and needs memory, the OS scans the free list for a hole large enough to fit it. But which hole should it pick?

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **First Fit** | Use the first hole that's large enough | Fast — stops searching early | Tends to fragment the beginning of memory |
| **Best Fit** | Use the smallest hole that's large enough | Minimizes wasted space in the chosen hole | Slow (scans entire list), leaves many tiny useless holes |
| **Worst Fit** | Use the largest hole | Leaves larger remaining holes (potentially more usable) | Slow (scans entire list), fragments the largest blocks |

In practice, **First Fit** is generally the best choice. It's fast, and simulations show it performs about as well as Best Fit. Worst Fit is almost always the worst performer.

### Example

Free holes: [100KB, 500KB, 200KB, 300KB, 50KB]
New process needs: 180 KB

- **First Fit**: picks the 500KB hole (first one >= 180KB), leaves 320KB remainder
- **Best Fit**: picks the 200KB hole, leaves 20KB remainder (probably too small to be useful)
- **Worst Fit**: picks the 500KB hole (largest), leaves 320KB remainder

## External Fragmentation

This is the killer problem with contiguous allocation. Over time, as processes start and exit, memory becomes a checkerboard of allocated and free regions.

```
  After several processes come and go:

  +------------------+
  |   OS             |
  +------------------+
  |   FREE (80 KB)   |     <-- too small for new process
  +------------------+
  |   Process B      |
  +------------------+
  |   FREE (40 KB)   |     <-- too small
  +------------------+
  |   Process D      |
  +------------------+
  |   FREE (120 KB)  |     <-- too small
  +------------------+
  |   Process E      |
  +------------------+
  |   FREE (60 KB)   |     <-- too small
  +------------------+

  Total free: 300 KB
  New process needs: 250 KB
  CANNOT ALLOCATE — even though enough total memory exists!
```

**External fragmentation** means free memory exists but is scattered in non-contiguous chunks. None of the individual holes is large enough, even though the total free space is sufficient.

Statistical analysis shows that with First Fit, about **one-third of memory is lost to fragmentation** (the "50% rule" — if N blocks are allocated, approximately 0.5N blocks are lost to fragmentation).

## Internal Fragmentation

Even with variable partitioning, memory is typically allocated in multiples of some unit (e.g., 4-byte or 8-byte aligned blocks). If a process needs 1001 bytes and the OS allocates in 8-byte multiples, it gets 1008 bytes — wasting 7 bytes.

```
  Allocated block: 1008 bytes
  +------------------------------------------+
  | Process data (1001 bytes)    | Wasted (7) |
  +------------------------------------------+
                                   ^ internal
                                     fragmentation
```

Internal fragmentation is minor compared to external fragmentation. It becomes more relevant with fixed partitioning or paging (where the last page may be partially empty).

## Compaction

One fix for external fragmentation: **move all processes to one end of memory**, consolidating free space into one large block.

```
  Before compaction:               After compaction:

  +------------------+             +------------------+
  |   OS             |             |   OS             |
  +------------------+             +------------------+
  |   FREE (80 KB)   |             |   Process B      |
  +------------------+             +------------------+
  |   Process B      |             |   Process D      |
  +------------------+             +------------------+
  |   FREE (40 KB)   |             |   Process E      |
  +------------------+             +------------------+
  |   Process D      |             |                  |
  +------------------+             |   FREE (300 KB)  |
  |   FREE (120 KB)  |             |                  |
  +------------------+             |                  |
  |   Process E      |             +------------------+
  +------------------+
  |   FREE (60 KB)   |
  +------------------+
```

Compaction is like defragmenting a hard drive. It works, but it's **extremely expensive**:

- All affected processes must be paused while their memory is copied
- All addresses/pointers within those processes must be updated (only possible with run-time address binding)
- Copying large amounts of memory takes significant time

Compaction is a band-aid, not a solution. This is why the computing world moved to paging.

## Why Contiguous Allocation Doesn't Scale

| Problem | Impact |
|---------|--------|
| External fragmentation | Up to 1/3 of memory wasted |
| Compaction is expensive | Requires pausing processes and copying memory |
| Process growth is hard | If a process's heap needs to grow, the adjacent memory might be occupied |
| No sharing | Can't efficiently share code (like shared libraries) between processes |
| Physical limit | A process can't be larger than the largest contiguous free region |

These limitations drove the development of **paging** — a scheme that breaks memory into small, fixed-size chunks that can be scattered anywhere in physical memory. No more contiguous requirement, no more external fragmentation.

## Real-World Connection

- **malloc internals**: user-space memory allocators like glibc's malloc face the same fragmentation problems within a process's heap. They use strategies like bins of different sizes, coalescing adjacent free blocks, and switching to mmap for large allocations — all techniques evolved from dealing with fragmentation.
- **Disk allocation**: file systems face identical fragmentation issues. ext4 uses extents (contiguous runs of blocks) but must handle fragmentation. This is why `defrag` exists for NTFS and why SSDs don't need defragmentation (random access is fast on flash).
- **Memory pools**: game engines and real-time systems often pre-allocate pools of fixed-size objects to avoid fragmentation entirely — a return to fixed partitioning at the application level, where predictability matters more than flexibility.

## Interview Angle

**Q: What's the difference between internal and external fragmentation?**

External fragmentation is when total free memory is sufficient but no single contiguous block is large enough for a request. It happens with variable-size allocation. Internal fragmentation is when allocated blocks are slightly larger than what was requested (due to alignment or fixed block sizes), wasting space within the block. Paging eliminates external fragmentation but introduces small amounts of internal fragmentation (the last page of a process is partially empty on average).

**Q: Why is First Fit generally preferred over Best Fit?**

First Fit is faster because it stops at the first suitable hole instead of scanning the entire list. Counterintuitively, it also tends to produce similar fragmentation to Best Fit in practice. Best Fit leaves behind many tiny holes that are too small to be useful, which can actually make fragmentation worse.

**Q: Why did operating systems move from contiguous allocation to paging?**

Contiguous allocation suffers from external fragmentation (up to 1/3 of memory wasted), makes process growth difficult (can't extend into occupied neighbor), and requires expensive compaction. Paging solves all these problems by breaking memory into small fixed-size pages that can be placed in any available frame — the pages don't need to be contiguous in physical memory. This also enables virtual memory and demand paging.

---

Next: [03 - Paging](03-paging.md)
