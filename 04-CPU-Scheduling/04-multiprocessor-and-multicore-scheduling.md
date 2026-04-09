# Multiprocessor and Multicore Scheduling

Everything so far assumed a single CPU. Real servers have 8, 32, 64, or even 128 cores. Scheduling on multiple processors introduces new challenges: which core should a process run on? How do you balance load? What happens when memory is closer to some cores than others? Getting this wrong means your 64-core machine performs like a 16-core machine.

---

## Asymmetric vs Symmetric Multiprocessing

Two fundamental approaches to multiprocessor scheduling:

```
Asymmetric Multiprocessing (AMP):
+--------+     +--------+     +--------+
| Core 0 |     | Core 1 |     | Core 2 |
| MASTER |     | WORKER |     | WORKER |
| (sched)|---->| (runs  |     | (runs  |
| (I/O)  |     |  tasks)|     |  tasks)|
+--------+     +--------+     +--------+
  Only Core 0 makes scheduling decisions.
  Simpler, but Core 0 becomes a bottleneck.

Symmetric Multiprocessing (SMP):
+--------+     +--------+     +--------+
| Core 0 |     | Core 1 |     | Core 2 |
| (sched)|     | (sched)|     | (sched)|
| (runs  |     | (runs  |     | (runs  |
|  tasks)|     |  tasks)|     |  tasks)|
+--------+     +--------+     +--------+
  Each core runs its own scheduler.
  More complex, but scales better.
```

| Aspect | AMP | SMP |
|---|---|---|
| Scheduling decisions | One master core | Each core independently |
| Bottleneck | Master core | Shared data structures |
| Scalability | Poor (master saturates) | Good |
| Implementation | Simpler | More complex (locking needed) |
| Used by | Some embedded systems | All modern general-purpose OSes |

**All modern OSes use SMP.** Linux, Windows, and macOS all allow each core to make independent scheduling decisions.

---

## Processor Affinity

When a process runs on a core, it loads data into that core's cache. If the scheduler moves it to a different core on its next run, the old cache is useless and the new core has a cold cache — a **cache miss penalty** that can be significant.

```
Process P runs on Core 0:
  Core 0 Cache: [P's data: hot]     Core 1 Cache: [other data]
  
Scheduler migrates P to Core 1:
  Core 0 Cache: [P's data: wasted]  Core 1 Cache: [cold -- must reload]
  
  Cost: hundreds of cycles for cache misses!
```

**Processor affinity** = the tendency to keep a process on the same core.

| Type | Description | Enforcement |
|---|---|---|
| **Soft affinity** | OS tries to keep process on same core, but may migrate under load | Default in most OSes |
| **Hard affinity** | Process is pinned to specific core(s), never migrated | Set by user/admin |

```bash
# Linux: set hard affinity (pin process to cores 0 and 1)
taskset -c 0,1 ./my_process

# Linux: set affinity for running process
taskset -p -c 2,3 <PID>

# Inside a program: sched_setaffinity() system call
```

**When to use hard affinity:**
- Real-time processes that need predictable latency
- NUMA-aware applications (pin to cores near their memory)
- Performance-critical applications where cache warmth matters
- Isolating noisy neighbors in multi-tenant environments

---

## Load Balancing

With per-core run queues (the SMP approach), some cores may end up overloaded while others sit idle. Load balancing redistributes work.

```
Before load balancing:
  Core 0: [P1][P2][P3][P4][P5]     (overloaded!)
  Core 1: [P6]                       (mostly idle)
  Core 2: []                         (completely idle!)

After load balancing:
  Core 0: [P1][P2]
  Core 1: [P3][P4]
  Core 2: [P5][P6]
```

Two approaches:

| Approach | How It Works | Triggered By |
|---|---|---|
| **Push migration** | Periodic task checks core loads; moves processes from busy to idle | Timer (e.g., every 200ms in Linux) |
| **Pull migration** | Idle core looks at other cores' queues and steals work | Core becoming idle |

Linux uses **both** simultaneously:
- A periodic rebalancer runs every few milliseconds (push)
- When a core's run queue empties, it performs **work stealing** from the busiest core (pull)

```
Work Stealing (Pull Migration):
  Core 2 finishes all its work.
  Core 2: "I'm idle. Let me check other queues..."
  Core 2 sees Core 0 has 5 tasks.
  Core 2 steals P4 and P5 from Core 0's queue.
  
  This happens without any centralized coordinator.
```

### The Balancing Trade-Off

Load balancing conflicts with processor affinity:
- **Balancing says**: move P to an idle core for parallelism
- **Affinity says**: keep P on its current core for cache warmth

The scheduler must weigh the cost of cache misses against the cost of imbalanced load. The general rule: migrate only when the imbalance is significant enough to justify the cache penalty.

---

## NUMA-Aware Scheduling

> **Analogy:** NUMA is like an office building where each floor has its own supply closet. Workers on floor 2 can grab supplies from their own closet instantly, but getting supplies from floor 5's closet means taking the elevator -- it works, but it's slower. NUMA-aware scheduling keeps processes on the "floor" closest to their data.

On large multi-socket systems, memory access time depends on which CPU is accessing it. This is **NUMA (Non-Uniform Memory Access)**.

```
NUMA Topology:

+------------------+          +------------------+
|   Socket 0       |          |   Socket 1       |
|  +------+------+ |  Inter-  | +------+------+  |
|  |Core 0|Core 1| | connect  | |Core 2|Core 3|  |
|  +------+------+ |<-------->| +------+------+  |
|  | L3 Cache     | |  (slow)  | | L3 Cache     |  |
|  +--------------+ |          | +--------------+  |
|  | Local Memory | |          | | Local Memory |  |
|  | (fast: ~80ns)| |          | | (fast: ~80ns)|  |
|  +--------------+ |          | +--------------+  |
+------------------+          +------------------+

Core 0 accessing Socket 0's memory: ~80ns  (local)
Core 0 accessing Socket 1's memory: ~150ns (remote -- nearly 2x slower!)
```

### Why NUMA Matters for Scheduling

If a process allocated its memory on Socket 0, but the scheduler moves it to Core 2 (Socket 1), every memory access crosses the **interconnect** -- the high-speed bus connecting CPU sockets. Crossing it to access another socket's memory adds latency (typically 1.5-2x slower than local access) -- and takes nearly twice as long.

```
Good NUMA scheduling:
  Process P allocated memory on Socket 0
  Scheduler runs P on Core 0 or Core 1 (same socket)
  Memory access: fast (local)

Bad NUMA scheduling:
  Process P allocated memory on Socket 0
  Scheduler migrates P to Core 3 (Socket 1)
  Memory access: slow (remote, crosses interconnect)
  
  Performance can drop 30-50% for memory-intensive workloads!
```

**NUMA-aware scheduling rules:**
1. Try to schedule a process on a core in the same NUMA node as its memory
2. When allocating memory, prefer the local node
3. When load balancing, prefer to migrate within the same NUMA node
4. Only cross NUMA boundaries when the load imbalance is severe

```bash
# Linux: check NUMA topology
numactl --hardware

# Run process on specific NUMA node
numactl --cpunodebind=0 --membind=0 ./my_process

# Check NUMA statistics
numastat
```

---

## Per-CPU Run Queues vs Global Run Queue

```
Global Run Queue:
+---+---+---+---+---+---+---+
| P1| P2| P3| P4| P5| P6| P7|  <-- Single queue
+---+---+---+---+---+---+---+
       |          |         |
    Core 0     Core 1    Core 2

  Pros: Automatic load balancing
  Cons: Lock contention! All cores compete
        for the same lock to dequeue tasks.
        Becomes a bottleneck at 8+ cores.

Per-CPU Run Queues:
  Core 0        Core 1        Core 2
+---+---+    +---+---+    +---+---+
| P1| P2|    | P3| P4|    | P5| P6|
+---+---+    +---+---+    +---+---+

  Pros: No lock contention (each core has its own lock)
  Cons: Need explicit load balancing between queues
```

| Approach | Lock Contention | Load Balance | Scalability | Used By |
|---|---|---|---|---|
| Global queue | High (one lock) | Automatic | Poor (> 8 cores) | Early Linux (2.4) |
| Per-CPU queues | Low (per-core locks) | Needs rebalancer | Good | Linux 2.6+, Windows, macOS |

Linux switched from a global run queue to per-CPU run queues in the 2.6 kernel (the O(1) scheduler). This was essential for scaling to the multi-core era.

---

## Real-World Connection

- **Linux** uses per-CPU run queues with CFS. The `schedutil` governor ties CPU frequency scaling to scheduler load. Work stealing ensures idle cores pull tasks from busy ones. `perf stat` shows cache miss rates that indicate poor affinity.
- **Docker/containers**: By default, containers share all cores. Use `--cpuset-cpus` to pin containers to specific cores and `--cpuset-mems` for NUMA node binding. This is critical for latency-sensitive workloads.
- **Cloud VMs**: AWS c5.metal instances expose the NUMA topology to guests. Running a database on such instances requires NUMA-aware configuration — binding the database processes and memory to the same NUMA node can improve query latency by 20-40%.
- **JVM/Go runtime**: These have their own user-space schedulers that sit on top of OS scheduling. Go's runtime uses work-stealing across its P (processor) abstraction. The JVM can pin threads to cores for low-latency trading systems with `-XX:+UseNUMA`.

---

## Interview Angle

**Q: Why do modern OSes use per-CPU run queues instead of a single global queue?**

A: A global run queue creates a serialization bottleneck — every core contending for the same lock to pick the next task. At 8+ cores, lock contention dominates and scaling collapses. Per-CPU run queues let each core make scheduling decisions independently with its own lock. The trade-off is that you need a load balancer (push/pull migration) to prevent imbalance, but this runs infrequently and is much cheaper than constant lock contention.

**Q: When would you pin a process to a specific CPU core?**

A: Pin processes when cache warmth or latency predictability matters more than flexibility. Common cases: (1) real-time or latency-sensitive workloads where migration causes jitter, (2) NUMA-aware applications where you want memory locality, (3) isolating workloads in multi-tenant systems (pin the noisy neighbor to dedicated cores), (4) high-frequency trading where microseconds matter. Use `taskset` or `cgroups` cpuset on Linux.

**Q: A Java application on a 2-socket NUMA server performs well at low load but degrades badly at high load. What could be wrong?**

A: At high load, the OS likely migrates threads across NUMA nodes to balance load, causing them to access remote memory (2x slower). Diagnosis: check `numastat` for high `numa_miss` and `numa_foreign` counts. Fixes: (1) bind the JVM to a single NUMA node with `numactl --cpunodebind=0 --membind=0`, (2) use the JVM's `-XX:+UseNUMA` flag to make the garbage collector NUMA-aware, (3) if the application needs both sockets, ensure threads are pinned to the same socket as their data.

---

Next: [Real-Time Scheduling](05-real-time-scheduling.md)
