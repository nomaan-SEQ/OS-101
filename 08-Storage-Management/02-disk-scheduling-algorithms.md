# Disk Scheduling Algorithms

When multiple processes issue I/O requests to a disk, the OS must decide **what order to serve them in**. On an HDD, the order matters enormously — a bad ordering means the disk arm ping-pongs across the platter, wasting milliseconds on seeks. A good ordering can improve throughput by 5-10x.

The goal: **minimize total seek time** by intelligently ordering requests in the disk queue.

---

## Setup: Example Request Queue

For all examples below, we'll use the same scenario:

```
    Disk has 200 tracks (0-199)
    Current head position: track 53
    Head moving toward higher track numbers
    
    Request queue (arrival order): 98, 183, 37, 122, 14, 124, 65, 67
```

---

## FCFS (First-Come, First-Served)

Serve requests in the order they arrive. Simple, fair, but terrible for seek optimization.

```
    Head movement:  53 -> 98 -> 183 -> 37 -> 122 -> 14 -> 124 -> 65 -> 67
    
    Track: 0    14  37  53  65 67  98    122 124         183      199
           |    |   |   |   | |   |      |   |           |        |
           |    |   |   *===|=|===*      |   |           |        |
           |    |   |       | |   |======*===*           |        |
           |    |   |       | |   |      |   |===========*        |
           |    *===|=======*=*   |      |   |           |        |
           |    |   |       | |   |      |   |           |        |
           *====*   |       | |   |      |   |           |        |
           |    |   |       *=*   |      |   |           |        |
    
    Seek distances: |53-98| + |98-183| + |183-37| + |37-122| + |122-14|
                    + |14-124| + |124-65| + |65-67|
                  = 45 + 85 + 146 + 85 + 108 + 110 + 59 + 2
    
    Total seek distance: 640 tracks
```

The arm zigzags wildly. FCFS makes no effort to optimize — it's purely the arrival order.

---

## SSTF (Shortest Seek Time First)

Always go to the **nearest** pending request. Greedy algorithm — good average performance, but can **starve** requests far from the current cluster of activity.

```
    Head at 53. Nearest request: 65
    Head at 65. Nearest: 67
    Head at 67. Nearest: 37
    Head at 37. Nearest: 14
    Head at 14. Nearest: 98
    Head at 98. Nearest: 122
    Head at 122. Nearest: 124
    Head at 124. Nearest: 183

    Movement: 53 -> 65 -> 67 -> 37 -> 14 -> 98 -> 122 -> 124 -> 183

    Seek distances: 12 + 2 + 30 + 23 + 84 + 24 + 2 + 59
    
    Total seek distance: 236 tracks
```

Much better than FCFS! But notice: if new requests keep arriving near the head's current position, request at track 14 (or 183) might wait forever. This is **starvation**.

---

## SCAN (Elevator Algorithm)

The head moves in one direction, serving all requests along the way, then **reverses** direction.

> **Analogy:** SCAN works exactly like an **elevator** in a building. The elevator picks a direction (up), stops at every requested floor along the way, rides to the top, then reverses and serves requests going down. C-SCAN is like an elevator that only picks people up while going up -- when it reaches the top floor, it drops express back to the ground floor without stopping, then starts upward again. This gives everyone a more predictable wait time.

```
    Head at 53, moving toward 199 (higher tracks).
    Serve all requests in the upward direction, go to end, then reverse.

    Upward:   53 -> 65 -> 67 -> 98 -> 122 -> 124 -> 183 -> [199]
    Downward: [199] -> 37 -> 14

    Track:  0    14  37  53  65 67  98    122 124         183      199
            |    |   |   |   | |   |      |   |           |        |
    UP:     |    |   |   *-->*-*-->*----->*-->*---------->*------->*
    DOWN:   |    |   |   |   | |   |      |   |           |        |
            *<---*<--|---|---|-|---|-------|---|-----------|--------*
    
    Seek distances: 12 + 2 + 31 + 24 + 2 + 59 + 16 + 162 + 23
    
    Total seek distance: 331 tracks
```

No starvation — every request is guaranteed to be served within two sweeps. The tradeoff: requests just behind the head's current direction must wait for a full sweep and return.

---

## C-SCAN (Circular SCAN)

Like SCAN, but when the head reaches one end, it **jumps back to the beginning** without serving requests on the return trip. This provides more **uniform wait times** — no request benefits from being "behind" the head during a reverse sweep.

```
    Head at 53, moving toward 199.
    Serve requests going up, jump to 0, continue upward again.

    Upward:   53 -> 65 -> 67 -> 98 -> 122 -> 124 -> 183 -> [199]
    Jump:     [199] ~~~jump to~~~ [0]
    Upward:   [0] -> 14 -> 37

    Track:  0    14  37  53  65 67  98    122 124         183      199
            |    |   |   |   | |   |      |   |           |        |
    UP:     |    |   |   *-->*-*-->*----->*-->*---------->*------->*
    JUMP:   *<~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~*
    UP:     *--->*-->*
    
    Total seek distance: 12 + 2 + 31 + 24 + 2 + 59 + 16 + (jump) + 14 + 23
```

The jump from 199 to 0 is often counted as free (the head moves fast when not reading), making C-SCAN more predictable for request response times.

---

## LOOK and C-LOOK

LOOK and C-LOOK are practical improvements: instead of going all the way to the disk edges (0 and 199), the head only goes as far as **the last actual request** in each direction.

```
    C-LOOK example:
    
    Head at 53, moving up.
    Go only as far as 183 (last request upward), then jump to 14 (lowest request).

    Upward:   53 -> 65 -> 67 -> 98 -> 122 -> 124 -> 183
    Jump:     183 ~~~jump to~~~ 14
    Upward:   14 -> 37

    Track:  0    14  37  53  65 67  98    122 124         183      199
            |    |   |   |   | |   |      |   |           |        |
            |    |   |   *-->*-*-->*----->*-->*---------->*        |
            |    *-->*   |   | |   |      |   |     jump  |        |
            |    *<~~|~~~|~~~|~|~~~|~~~~~~|~~~|~~~~~~~~~~~*        |
    
    Seek distances: 12 + 2 + 31 + 24 + 2 + 59 + (jump) + 23
    
    Total seek distance: 153 tracks (excluding jump)
```

**LOOK and C-LOOK are what real systems actually implement.** Going to the physical disk edges wastes time when there are no requests there.

---

## Algorithm Comparison

| Algorithm | Total Seek (our example) | Fairness | Starvation? | Throughput | Notes |
|-----------|-------------------------|----------|-------------|------------|-------|
| FCFS | 640 | Excellent | No | Poor | Baseline, no optimization |
| SSTF | 236 | Poor | Yes | Good | Greedy, favors clustered requests |
| SCAN | 331 | Good | No | Good | Guaranteed bounded wait |
| C-SCAN | ~322 | Very good | No | Good | Uniform wait distribution |
| LOOK | Better than SCAN | Good | No | Very good | Practical SCAN |
| C-LOOK | ~153 | Very good | No | Very good | What real systems use |

---

## Why Scheduling Matters Less for SSDs

SSDs have **no seek time and no rotational latency**. A random read on an SSD costs ~25-100 microseconds regardless of the logical block address. Reordering requests to minimize "seek distance" gains nothing.

But scheduling isn't completely irrelevant for SSDs:

- **Queue depth** matters — SSDs achieve peak IOPS only with multiple outstanding requests (the controller processes them in parallel across channels)
- **Write combining** — grouping small writes into larger sequential ones reduces write amplification
- **Priority and fairness** — even without seek optimization, you still want fair access between processes
- **NVMe hardware queues** — NVMe drives handle their own internal scheduling across 65,535 queues, so the OS scheduler should mostly get out of the way

This is why modern Linux uses `none` (no-op) scheduler for NVMe — let the hardware do the work.

---

## Real-World Connection

- **Linux I/O schedulers** (covered in detail in file 04) implement variants of these algorithms. The legacy `cfq` used a modified SCAN approach per process, while `deadline` adds time-based guarantees on top of a SCAN-like ordering.
- **Database query planners** consider sequential vs random I/O costs. PostgreSQL's `random_page_cost` parameter (default 4.0 on HDD, often set to 1.1 on SSD) reflects the HDD seek time penalty.
- **Defragmentation** improves HDD performance by making logical sequences physically contiguous — reducing the need for seeks. On SSDs, defragmentation is useless and actually harmful (extra writes for zero benefit).
- **Log-structured merge trees** (LSM trees, used in RocksDB/Cassandra/LevelDB) convert random writes to sequential writes. This design exists specifically because of the random I/O penalty on HDDs, and remains useful on SSDs to reduce write amplification.

---

## Interview Angle

**Q: Explain the SCAN disk scheduling algorithm and its advantage over SSTF.**

A: SCAN (the elevator algorithm) moves the disk head in one direction, serving all pending requests along the way, then reverses direction. Its key advantage over SSTF is **no starvation** — SSTF always picks the nearest request, which means requests far from the head's current position can wait indefinitely if new nearby requests keep arriving. SCAN guarantees every request is served within at most two sweeps of the disk. The tradeoff is slightly higher average seek time compared to SSTF.

**Q: Why is C-SCAN preferred over SCAN in practice?**

A: C-SCAN provides more uniform response times. With regular SCAN, requests at the midpoint of the disk get served twice as often as requests at the edges (once on the way up, once on the way down). C-SCAN only serves in one direction and jumps back to the start, ensuring every position has a similar expected wait time. This predictability matters for systems that need consistent latency.

**Q: Do disk scheduling algorithms matter for SSDs?**

A: Traditional seek-optimizing algorithms (SCAN, SSTF, etc.) provide no benefit on SSDs because there's no physical head movement — all addresses have equal access time. However, I/O scheduling still matters for SSDs in terms of queue depth management (SSDs need multiple outstanding requests to saturate their internal parallelism), write combining, and fairness between processes. For NVMe SSDs, the best approach is usually a no-op scheduler that lets the drive's hardware queues handle ordering internally.

---

Next: [03-raid-levels.md](03-raid-levels.md)
