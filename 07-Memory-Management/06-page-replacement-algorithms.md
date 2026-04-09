# Page Replacement Algorithms

When memory is full and a page fault occurs, the OS must evict a page to make room. The choice of which page to evict is critical — evict the wrong page and you'll need it again immediately, triggering another fault. Page replacement algorithms are one of the most studied topics in OS theory, and they have direct analogs in every caching system you'll encounter in practice.

## The Problem

```
  Physical Memory (full):
  +--------+--------+--------+--------+
  | Pg A   | Pg B   | Pg C   | Pg D   |   4 frames, all occupied
  +--------+--------+--------+--------+

  Process needs Page E --> PAGE FAULT

  Which page do we evict to make room for E?
  Bad choice: evict a page we'll need on the next access (another fault)
  Good choice: evict a page we won't need for a long time (no fault)
```

If the evicted page has been modified (dirty bit = 1), it must be written to disk (swap) before the frame can be reused. If it's clean (not modified since loading), the frame can be reused immediately — the disk copy is already up to date.

## FIFO (First In, First Out)

The simplest algorithm: evict the **oldest** page — the one that has been in memory the longest.

Implementation: maintain a queue. When a page is loaded, add it to the tail. When eviction is needed, remove from the head.

### Example

Reference string: `1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5`
Frames: 3

```
  Access: 1  2  3  4  1  2  5  1  2  3  4  5
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fr 0:  |1 |1 |1 |4 |4 |4 |5 |5 |5 |5 |5 |5 |
  Fr 1:  |  |2 |2 |2 |1 |1 |1 |1 |1 |3 |3 |3 |
  Fr 2:  |  |  |3 |3 |3 |2 |2 |2 |2 |2 |4 |4 |
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fault?  F  F  F  F  F  F  F        F  F

  Total page faults: 9
```

FIFO is simple but can perform poorly. An old page might still be heavily used (think of a frequently accessed library page loaded at startup). FIFO doesn't care about usage patterns — it only knows arrival order.

## Optimal (OPT / Belady's Algorithm)

Evict the page that **won't be used for the longest time in the future**. This is provably optimal — it produces the minimum number of page faults for any reference string.

### Example (same reference string, 3 frames)

```
  Access: 1  2  3  4  1  2  5  1  2  3  4  5
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fr 0:  |1 |1 |1 |1 |1 |1 |1 |1 |1 |1 |4 |4 |
  Fr 1:  |  |2 |2 |2 |2 |2 |2 |2 |2 |2 |2 |2 |
  Fr 2:  |  |  |3 |4 |4 |4 |5 |5 |5 |3 |3 |5 |
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fault?  F  F  F  F        F        F  F  F

  Total page faults: 7 (optimal -- no algorithm can do better)
```

When evicting to load page 4, OPT looks ahead: page 1 is next used at position 5, page 2 at position 6, page 3 at position 10. Evict page 3 — it won't be needed for the longest time.

**The catch**: OPT requires knowing the future. It's impossible to implement in practice. Its value is as a **benchmark** — you can compare any real algorithm against OPT to see how close it gets.

## LRU (Least Recently Used)

If we can't know the future, we approximate it using the past. **LRU evicts the page that hasn't been used for the longest time**, betting that pages unused recently won't be needed soon (temporal locality).

### Example (same reference string, 3 frames)

```
  Access: 1  2  3  4  1  2  5  1  2  3  4  5
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fr 0:  |1 |1 |1 |4 |4 |4 |5 |5 |5 |5 |4 |4 |
  Fr 1:  |  |2 |2 |2 |1 |1 |1 |1 |1 |3 |3 |5 |
  Fr 2:  |  |  |3 |3 |3 |2 |2 |2 |2 |2 |2 |2 |
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fault?  F  F  F  F  F  F  F        F  F  F

  Total page faults: 10
```

LRU is a good approximation of OPT in practice (not always — it can be worse on certain access patterns). It's the conceptual basis for most real page replacement algorithms.

### The Implementation Problem

True LRU requires knowing the exact order of all page accesses. Two approaches:

**Counter-based**: give each page a timestamp on every access. On eviction, scan for the smallest timestamp. Problem: updating a counter on every memory access (billions per second) is prohibitively expensive.

**Stack-based**: maintain a stack of page numbers. On every access, move that page to the top. The bottom of the stack is the LRU page. Problem: maintaining a stack on every memory access is also too expensive.

Neither is practical for a page replacement algorithm that runs on every memory access. We need approximations.

## Clock Algorithm (Second Chance)

The **Clock algorithm** is a practical approximation of LRU that uses only the **reference bit** (already maintained by hardware in each page table entry).

> **Analogy:** Imagine a security guard walking around a circular hallway of offices. Each office has a sign that flips to "visited" when someone enters. The guard sweeps around: if an office shows "visited," they flip the sign back to "not visited" and move on. If an office shows "not visited," that office has been idle since the last sweep -- time to reclaim it. The guard's steady sweep is the clock hand, and the signs are the reference bits.

### How It Works

Arrange all frames in a circular list with a "clock hand" pointer. Each frame has a reference bit (R):

1. When a page is accessed, hardware sets R = 1
2. When we need to evict a page:
   - Look at the page under the clock hand
   - If R = 1: give it a "second chance" — set R = 0, advance the hand
   - If R = 0: this page hasn't been accessed since its last chance — **evict it**
   - Keep sweeping until you find R = 0

```
  Clock hand
      |
      v
  +-------+     +-------+     +-------+     +-------+
  | Pg A  | --> | Pg B  | --> | Pg C  | --> | Pg D  | --+
  | R = 1 |     | R = 0 |     | R = 1 |     | R = 0 |   |
  +-------+     +-------+     +-------+     +-------+   |
      ^                                                   |
      +---------------------------------------------------+

  Need to evict:
  1. Hand at A: R=1 --> set R=0, advance
  2. Hand at B: R=0 --> EVICT B
```

The Clock algorithm is essentially FIFO with a mercy rule: if a page has been accessed since the last sweep, it gets to stay. This captures most of the benefit of LRU with minimal overhead — the reference bit is set by hardware automatically.

### Enhanced Clock (NRU - Not Recently Used)

Consider both the reference bit (R) and the dirty bit (M) to create priority classes:

| Class | R | M | Description | Eviction Priority |
|-------|---|---|-------------|-------------------|
| 0 | 0 | 0 | Not recently used, not modified | Best candidate (evict first) |
| 1 | 0 | 1 | Not recently used, but modified | Need disk write if evicted |
| 2 | 1 | 0 | Recently used, not modified | Likely needed again |
| 3 | 1 | 1 | Recently used and modified | Worst candidate (evict last) |

Prefer evicting class 0 pages (clean and unused) over class 1 (dirty but unused) over class 2, etc. This avoids unnecessary disk writes.

## Algorithm Comparison

| Algorithm | Faults (our example) | Speed | Accuracy | Practical? |
|-----------|---------------------|-------|----------|------------|
| FIFO | 9 | Very fast | Poor | Yes (but not great) |
| OPT | 7 | N/A | Perfect | No (needs future knowledge) |
| LRU | 10 | Slow (exact) | Very good | Not exactly (approximated) |
| Clock | ~8-9 | Fast | Good | Yes (widely used) |

## Belady's Anomaly

Here's a counterintuitive result: with FIFO, **adding more frames can increase page faults**.

Reference string: `1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5`

```
  With 3 frames (FIFO): 9 page faults
  With 4 frames (FIFO): Let's trace it:

  Access: 1  2  3  4  1  2  5  1  2  3  4  5
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fr 0:  |1 |1 |1 |1 |1 |1 |5 |5 |5 |5 |4 |4 |
  Fr 1:  |  |2 |2 |2 |2 |2 |2 |1 |1 |1 |1 |5 |
  Fr 2:  |  |  |3 |3 |3 |3 |3 |3 |2 |2 |2 |2 |
  Fr 3:  |  |  |  |4 |4 |4 |4 |4 |4 |3 |3 |3 |
         +--+--+--+--+--+--+--+--+--+--+--+--+
  Fault?  F  F  F  F        F  F  F  F  F  F

  Total page faults: 10 -- WORSE than 3 frames!
```

This is **Belady's anomaly**: adding more memory made performance worse. It only affects FIFO and some other algorithms. LRU and OPT are **stack algorithms** — they never exhibit Belady's anomaly (more frames always means equal or fewer faults).

This is a classic interview gotcha and a reason FIFO is not used in practice for page replacement.

## What Linux Actually Does

Linux doesn't use any textbook algorithm directly. It uses a **two-list strategy**:

```
  Active List                    Inactive List
  (recently accessed)            (candidates for eviction)
  +---+---+---+---+---+         +---+---+---+---+---+
  | A | B | C | D | E |  --->   | F | G | H | I | J |  ---> evict
  +---+---+---+---+---+         +---+---+---+---+---+
  (hot pages)                    (cold pages)
```

- **Active list**: pages that have been accessed recently. Protected from eviction.
- **Inactive list**: pages that haven't been accessed recently. Candidates for eviction.

Pages move between lists based on access patterns:
1. New pages start on the inactive list
2. If accessed while on the inactive list, they're promoted to the active list
3. When the active list gets too large, pages are demoted to the inactive list
4. Eviction happens from the tail of the inactive list

The **kswapd** daemon runs in the background, monitoring free memory and proactively moving pages from active to inactive lists when memory gets low. This avoids doing all the work at page fault time.

Linux also distinguishes between:
- **File-backed pages** (from mmap'd files): can be evicted without writing to swap (just re-read from the file)
- **Anonymous pages** (heap, stack): must be written to swap if dirty

The `vm.swappiness` parameter (0-200, default 60) controls the balance between evicting file-backed pages vs anonymous pages. Lower values prefer keeping anonymous pages in RAM.

## Real-World Connection

- **Database buffer pools**: MySQL's InnoDB buffer pool uses a modified LRU with "young" and "old" sublists — conceptually similar to Linux's active/inactive lists. PostgreSQL's shared buffers use a clock-sweep algorithm. Understanding OS page replacement helps you understand these systems.
- **Redis eviction policies**: Redis offers `allkeys-lru`, `allkeys-lfu`, and other eviction policies when memory is full — the same concept at the application level.
- **CDN caching**: CDNs like Cloudflare and Fastly must decide which content to keep in edge cache and which to evict. They use algorithms similar to LRU and LFU (Least Frequently Used).
- **CPU caches**: L1/L2/L3 caches use similar replacement policies implemented in hardware — often pseudo-LRU for speed.

## Interview Angle

**Q: Compare FIFO, LRU, and Clock page replacement algorithms.**

FIFO evicts the oldest page — simple but can evict heavily-used pages and exhibits Belady's anomaly. LRU evicts the least recently used page — a good approximation of optimal, but expensive to implement exactly (needs per-access tracking). Clock approximates LRU using the hardware reference bit: it sweeps a circular list, giving each referenced page a second chance before evicting. Clock is what real systems use because it's fast and reasonably accurate.

**Q: What is Belady's anomaly and which algorithms suffer from it?**

Belady's anomaly is the counterintuitive situation where adding more page frames increases page faults. It occurs with FIFO because eviction order (arrival time) doesn't correlate with future usage. LRU and OPT never exhibit this anomaly — they are "stack algorithms" where the set of pages in memory with N+1 frames is always a superset of the set with N frames.

**Q: Why can't we implement true LRU in an OS?**

True LRU requires knowing the exact order of every page access. This means updating a timestamp or stack on every memory access — billions of times per second. The overhead would be enormous. Instead, real systems approximate LRU using the hardware-maintained reference bit (which the MMU sets on each access automatically). The Clock algorithm and Linux's active/inactive lists are practical approximations that capture most of LRU's benefit at a fraction of the cost.

**Q: How does Linux actually do page replacement?**

Linux maintains two lists: an active list (hot pages, recently accessed) and an inactive list (cold pages, eviction candidates). Pages promoted between lists based on access. The kswapd daemon proactively manages these lists. Linux also distinguishes file-backed pages (can be re-read from file, cheaper to evict) from anonymous pages (must write to swap). The vm.swappiness parameter controls this balance.

---

Next: [07 - TLB and Page Table Structures](07-tlb-and-page-table-structures.md)
