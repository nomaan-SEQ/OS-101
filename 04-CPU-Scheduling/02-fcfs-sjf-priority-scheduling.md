# FCFS, SJF, and Priority Scheduling

These three algorithms represent the foundational scheduling strategies. Each one makes a different bet about what matters most: arrival order, burst length, or assigned importance. Understanding their trade-offs is essential — pieces of all three show up in every real scheduler.

---

## FCFS (First-Come, First-Served)

The simplest scheduling algorithm: processes are executed in the order they arrive in the ready queue. It's a plain FIFO queue.

**Properties:**
- Non-preemptive — once a process starts, it runs to completion (or until it blocks)
- Easy to implement — just a queue
- No starvation — every process eventually runs

### Example

```
Process  Arrival  Burst
  P1       0       24
  P2       1        3
  P3       2        3

Gantt Chart:
| P1                       | P2  | P3  |
0                         24    27    30

             Turnaround    Waiting     Response
Process      (Finish-Arr)  (Turn-Burst) (Start-Arr)
  P1         24-0 = 24     24-24 = 0    0-0  = 0
  P2         27-1 = 26     26-3  = 23   24-1 = 23
  P3         30-2 = 28     28-3  = 25   27-2 = 25

Averages:     26.0           16.0        16.0
```

### The Convoy Effect

The big problem with FCFS: one long process blocks everything behind it.

```
Scenario: P1 (CPU-bound, burst=100), P2-P10 (I/O-bound, burst=1 each)

With FCFS:
|       P1 (100ms)          | P2|P3|P4|P5|P6|P7|P8|P9|P10|
0                          100                             109

P2-P10 each waited ~100ms for their 1ms CPU burst!
The I/O devices sit idle while these processes wait.
This is the convoy effect.
```

Think of it like being stuck behind a slow truck on a one-lane road. Nine cars with a 10-second trip each have to wait for the truck's 10-minute trip.

---

## SJF (Shortest Job First)

Schedule the process with the shortest next CPU burst. This is provably optimal for minimizing average waiting time.

**Intuition**: Short jobs finish fast and leave the queue, reducing the total accumulated wait for everyone else.

### Non-Preemptive SJF Example

```
Process  Arrival  Burst
  P1       0       7
  P2       2       4
  P3       4       1
  P4       5       4

Gantt Chart (non-preemptive, decisions at completion):
| P1          | P3| P2       | P4       |
0             7   8         12         16

             Turnaround    Waiting
Process      (Finish-Arr)  (Turn-Burst)
  P1         7-0  = 7      7-7  = 0
  P2         12-2 = 10     10-4 = 6
  P3         8-4  = 4      4-1  = 3
  P4         16-5 = 11     11-4 = 7

Averages:     8.0           4.0
```

### Preemptive SJF = SRTF (Shortest Remaining Time First)

When a new process arrives, compare its burst with the *remaining* time of the currently running process. If the new one is shorter, preempt.

```
Process  Arrival  Burst
  P1       0       7
  P2       2       4
  P3       4       1
  P4       5       4

Gantt Chart (SRTF):
|P1 |P2 |P3|P2  |P4       |P1              |
0   2   4  5    7         11               16

At t=0: P1 starts (only process, remaining=7)
At t=2: P2 arrives (burst=4 < P1 remaining=5), preempt P1
At t=4: P3 arrives (burst=1 < P2 remaining=2), preempt P2
At t=5: P3 done. P2 remaining=2, P4 burst=4, P1 remaining=5 -> P2 runs
At t=7: P2 done. P4 remaining=4, P1 remaining=5 -> P4 runs
At t=11: P4 done. P1 runs (remaining=5)

             Turnaround    Waiting
Process      (Finish-Arr)  (Turn-Burst)
  P1         16-0 = 16     16-7 = 9
  P2         7-2  = 5      5-4  = 1
  P3         5-4  = 1      1-1  = 0
  P4         11-5 = 6      6-4  = 2

Averages:     7.0           3.0  (better than non-preemptive!)
```

### The Prediction Problem

SJF is optimal but has one fatal flaw: **you can't know the next CPU burst length in advance**.

The common workaround is exponential averaging — predict the next burst based on previous bursts:

```
tau(n+1) = alpha * t(n) + (1 - alpha) * tau(n)

Where:
  tau(n+1) = predicted next burst
  t(n)     = actual length of last burst
  tau(n)   = previous prediction
  alpha    = weight (typically 0.5)

Example with alpha = 0.5:
  Actual bursts:  6,  4,  6,  4,  13, 13, 13
  Predictions:   10,  8,  6,  6,   5,  9, 11, 12
                      ^
                  (10*0.5 + 6*0.5 = 8)
```

This makes SJF an approximation in practice, not an exact algorithm. But even approximate SJF outperforms FCFS significantly.

---

## Priority Scheduling

Each process is assigned a priority number. The highest-priority process runs first. (Convention varies: some systems use low numbers for high priority, others the opposite.)

**Properties:**
- Can be preemptive (preempt when higher-priority process arrives) or non-preemptive
- SJF is actually a special case of priority scheduling where priority = predicted burst length
- Priorities can be internal (set by OS based on behavior) or external (set by administrator)

### Example (Lower Number = Higher Priority)

```
Process  Arrival  Burst  Priority
  P1       0       10      3
  P2       0        1      1  (highest)
  P3       0        2      4
  P4       0        1      5
  P5       0        5      2

Non-preemptive (all arrive at t=0, pick highest priority first):
| P2| P5        | P1              | P3  | P4|
0   1           6                16    18   19

             Turnaround    Waiting
Process      (Finish-Arr)  (Turn-Burst)
  P1         16            16-10 = 6
  P2          1             1-1  = 0
  P3         18            18-2  = 16
  P4         19            19-1  = 18
  P5          6             6-5  = 1

Averages:    12.0           8.2
```

### The Starvation Problem

**Starvation** (also called indefinite blocking): A low-priority process may wait forever if higher-priority processes keep arriving.

```
Time ->
  High-priority jobs keep arriving:
  [H1][H2][H3][H4][H5][H6]...  (low-priority P never runs!)

  P (priority=99) has been in the ready queue for 3 hours...
```

This is a real problem. The famous story: when MIT shut down their IBM 7094 in 1973, they found a low-priority process submitted in 1967 that had never run.

### The Solution: Aging

**Aging** gradually increases the priority of processes that have been waiting a long time.

```
Process P starts with priority = 50 (low)

Every 1 second in ready queue: priority increases by 1

After 30 seconds waiting: priority = 50 - 30 = 20
After 45 seconds waiting: priority = 50 - 45 = 5 (now high!)

Eventually, every process reaches high enough priority to run.
```

Aging guarantees that no process starves, while still respecting priority assignments for processes that haven't waited long.

---

## Comparison Table

| Aspect | FCFS | SJF / SRTF | Priority |
|---|---|---|---|
| Selection rule | Arrival order | Shortest burst | Highest priority |
| Preemptive? | No | SJF: No, SRTF: Yes | Either |
| Optimal avg wait? | No | Yes (provably) | No |
| Starvation? | No | Yes (long jobs) | Yes (low priority) |
| Needs burst prediction? | No | Yes | No |
| Convoy effect? | Yes | No | Possible |
| Implementation | Simple FIFO | Need burst estimates | Priority queue |
| Best for | Simple batch systems | Minimizing wait time | Differentiated service levels |
| Real-world usage | Rarely alone | Approximated in SRTF-like schedulers | Basis for most real schedulers |

---

## Real-World Connection

- **FCFS** is still used inside other schedulers — within a single priority level, processes are often scheduled FCFS. It's also the default for I/O request queues in many systems.
- **SJF/SRTF** principles appear in web server request scheduling. Nginx and HAProxy can prioritize shorter requests to avoid head-of-line blocking — this is the SJF idea applied to network requests.
- **Priority scheduling** is everywhere. Linux has 140 priority levels (0-99 for real-time, 100-139 for normal). `nice` values map to priorities in the normal range. Kubernetes pod priorities determine which pods get evicted under resource pressure.
- **Starvation** in real systems: database query schedulers can starve long-running analytical queries in favor of short OLTP queries. This is why databases separate OLTP and OLAP workloads.

---

## Interview Angle

**Q: If SJF is optimal, why don't real operating systems use it?**

A: Because you can't know the length of the next CPU burst in advance. You can only estimate it using past behavior (exponential averaging). Real schedulers use approximations — for example, multilevel feedback queues implicitly favor short-burst processes by demoting long-running ones to lower queues. CFS achieves something similar through virtual runtime accounting.

**Q: Walk me through the convoy effect. How would you detect it in a production system?**

A: The convoy effect happens in FCFS when a long CPU-bound process holds the CPU while short I/O-bound processes wait, resulting in low I/O utilization and high average waiting time. In production, you'd see symptoms like: high CPU utilization but low I/O throughput, high `runq` latency in monitoring tools, and interactive processes with unexpectedly high scheduling latency. Tools like `perf sched latency` on Linux show per-process scheduling delays.

**Q: A system uses priority scheduling. Users report that their background backup job never completes. What's happening and how do you fix it?**

A: This is likely starvation — the backup job has low priority and keeps getting preempted by higher-priority foreground work. Solutions: (1) implement aging so the backup job's priority gradually increases, (2) use `nice 19` to set it to lowest priority but with a scheduler (like CFS) that guarantees some CPU share, (3) use cgroups to reserve a minimum CPU percentage for the backup job, or (4) schedule the backup during off-peak hours.

---

Next: [Round Robin and Multilevel Queues](03-round-robin-and-multilevel-queues.md)
