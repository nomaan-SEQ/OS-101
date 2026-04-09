# Linux CFS and Real-World Schedulers

Theory is useful, but what do real operating systems actually do? This section covers the schedulers running on billions of devices right now. Linux's CFS is the most important one to understand — it takes a fundamentally different approach from the textbook algorithms and is the scheduler you'll interact with most as an engineer.

---

## Linux CFS (Completely Fair Scheduler)

CFS, introduced in Linux 2.6.23 (2007), was designed by Ingo Molnar. Its core idea is deceptively simple: **give every runnable process a fair share of the CPU, and always run the process that has received the least CPU time so far**.

### Virtual Runtime (vruntime)

Instead of priority queues or time quanta, CFS tracks a single metric per process: **virtual runtime (vruntime)** — a weighted measure of how much CPU time the process has consumed.

```
vruntime increases as a process runs:
  
  Normal process (nice 0):  vruntime increases at 1x rate
  Low priority (nice 19):   vruntime increases FASTER (gets less real time)
  High priority (nice -20):  vruntime increases SLOWER (gets more real time)

The process with the LOWEST vruntime runs next.
Always. That's the entire algorithm.
```

How weighting works:

```
Process A: nice 0  (weight = 1024)
Process B: nice 5  (weight = 335)

Both runnable. Total weight = 1024 + 335 = 1359

A gets: 1024/1359 = 75.3% of CPU
B gets:  335/1359 = 24.7% of CPU

A's vruntime grows at:  real_time * (1024 / 1024) = normal rate
B's vruntime grows at:  real_time * (1024 / 335)  = ~3x faster

So B accumulates vruntime faster, gets scheduled less often,
and receives less CPU time. But it DOES still run — no starvation.
```

### The Red-Black Tree

CFS stores all runnable processes in a **red-black tree**, sorted by vruntime. The leftmost node (lowest vruntime) is always the next process to run.

```
Red-Black Tree (sorted by vruntime):

              [P3: vrt=50]
             /             \
       [P1: vrt=30]    [P5: vrt=80]
       /          \           \
  [P2: vrt=10]  [P4: vrt=45]  [P6: vrt=100]
  
  Leftmost = P2 (vruntime=10) --> runs next!
  
  Operations:
  - Find minimum (next to run): O(1) -- cached leftmost pointer
  - Insert (wake up / new process): O(log n)
  - Remove (process blocks/terminates): O(log n)
```

Why a red-black tree instead of a heap? The tree allows efficient removal of any node (not just the minimum), which is needed when a process blocks or changes state. It also supports efficient iteration for debugging and accounting.

### No Fixed Time Quantum

Unlike Round Robin, CFS doesn't have a fixed time slice. Instead, it computes a **dynamic time slice** based on the number of runnable processes:

```
Target latency = 6ms (default, for <= 8 runnable processes)
Minimum granularity = 0.75ms (floor -- never less than this)

If 4 processes are runnable:
  Each gets: 6ms / 4 = 1.5ms per scheduling period

If 100 processes are runnable:
  Target would be 6ms / 100 = 0.06ms (too small!)
  Minimum granularity kicks in: each gets 0.75ms
  Scheduling period becomes: 100 * 0.75ms = 75ms
  
With nice values, slices are weighted:
  Process A (nice 0, weight=1024): gets proportionally more
  Process B (nice 5, weight=335):  gets proportionally less
```

### CFS Handling of New and Waking Processes

```
New process created (fork):
  vruntime = max(parent's vruntime, min_vruntime of the queue)
  This prevents gaming: can't fork repeatedly to get fresh low vruntime.

Process wakes from sleep (I/O completion):
  vruntime = max(saved vruntime, min_vruntime - threshold)
  
  The threshold ensures sleeping processes get a small boost
  (they "catch up" slightly) but can't dominate the CPU by
  sleeping for a long time and waking with an extremely low vruntime.
```

### Niceness Values

```
Nice value range: -20 (highest priority) to +19 (lowest priority)
Default: 0

Each nice level represents roughly a 10% difference in CPU share:
  nice 0  vs nice 1:  process at nice 0 gets ~10% more CPU
  nice 0  vs nice 5:  process at nice 0 gets ~3x more CPU
  nice 0  vs nice 19: process at nice 0 gets ~100x more CPU

Nice value to weight mapping (some key values):
  nice -20: weight = 88761  (most CPU)
  nice -10: weight = 9548
  nice   0: weight = 1024   (default)
  nice  10: weight = 110
  nice  19: weight = 15     (least CPU)
```

---

## EEVDF (Earliest Eligible Virtual Deadline First)

Starting with Linux 6.6 (late 2023), EEVDF is replacing the CFS pick-next logic. CFS had a subtle problem: it was fair on average but could have poor **latency** characteristics. A process that just became runnable might wait a long time if other processes also had low vruntimes.

EEVDF adds the concept of a **virtual deadline** — each process gets a deadline based on when it should next receive its share of CPU.

```
CFS pick-next:
  Always pick the process with lowest vruntime.
  Problem: a just-woken I/O-bound process may have similar vruntime
  to CPU-bound processes, and must wait its turn.

EEVDF pick-next:
  Among ELIGIBLE processes (those whose virtual time has arrived),
  pick the one with the EARLIEST virtual deadline.
  
  This means: a process that just woke up and needs a small slice
  gets a near virtual deadline and runs sooner.
  
  Result: better latency for interactive/I/O-bound workloads
  while maintaining overall fairness.
```

EEVDF is still based on the same vruntime accounting and red-black tree structure. The change is primarily in how the next process is selected, adding deadline awareness to the fairness model.

---

## Windows Scheduler

Windows uses a **priority-based preemptive** scheduler with 32 priority levels.

```
Windows Priority Levels:

  31 |===================| Real-time (16-31)
  30 |===================|   Fixed priorities
  ... |                   |   No dynamic boosting
  16 |===================|
  ---|-------------------|---
  15 |===================| Variable (1-15)
  14 |===================|   Subject to priority boosting
  ... |                   |   and decay
   1 |===================|
   0 |===================| Idle (0) -- system idle thread
```

**Key Windows scheduling features:**

| Feature | Description |
|---|---|
| Priority boosting | Interactive threads get temporary priority boost (e.g., +2 when receiving window input) |
| Priority decay | Boosted priority decays back to base after using quantum |
| Foreground boost | The foreground application gets 3x longer quantum |
| Processor groups | For systems with > 64 logical processors |
| UMS (User-Mode Scheduling) | Allows user-space thread scheduling (for SQL Server, etc.) |

Windows doesn't try to be "fair" in the CFS sense. Its philosophy: **interactive responsiveness matters most**. The foreground window gets more CPU, and I/O-completing threads get priority bumps. This is why Windows historically felt snappier for desktop use, while Linux was better for server throughput.

---

## macOS Scheduler

macOS uses a hybrid approach inherited from Mach/BSD:

- **Mach layer**: Thread scheduling with 128 priority levels across multiple scheduling bands (real-time, system, interactive, background)
- **BSD layer**: POSIX nice values and signals
- **Grand Central Dispatch (GCD)**: User-space work queue system that manages thread pools and maps work items to quality-of-service (QoS) classes

```
macOS QoS Classes:

  User Interactive  --> highest priority, immediate response expected
  User Initiated    --> user waiting for result
  Default           --> general work
  Utility           --> long-running, progress bar shown
  Background        --> user doesn't know it's happening
```

macOS aggressively manages scheduling for battery life. Background QoS tasks are deferred and batched to minimize CPU wake-ups. The App Nap feature throttles background applications by deprioritizing their scheduled work.

Apple Silicon (M-series) adds another dimension: **efficiency cores vs performance cores**. The scheduler maps QoS classes to core types — background work goes to efficiency cores, interactive work goes to performance cores.

---

## Comparison: Linux CFS vs Windows Scheduler

| Aspect | Linux CFS | Windows |
|---|---|---|
| Core philosophy | Fairness (equal CPU share) | Responsiveness (interactive priority) |
| Data structure | Red-black tree (vruntime) | Multi-level priority queues |
| Priority levels | 140 (100-139 for normal, nice-based) | 32 (0-31) |
| Time quantum | Dynamic (based on # runnable processes) | Fixed per priority class (foreground 3x) |
| I/O boost | Via vruntime (sleepers catch up) | Explicit priority boost (+1 to +2) |
| Fairness guarantee | Yes (by design — vruntime ensures it) | No (higher priority always preempts) |
| Starvation | Impossible (vruntime converges) | Possible (mitigated by priority decay) |
| Best for | Servers, mixed workloads | Desktop interactivity |

---

## Practical Commands

```bash
# --- Viewing scheduling info ---

# See scheduler statistics
cat /proc/sched_debug              # Detailed scheduler state
cat /proc/<PID>/sched              # Per-process scheduling info

# --- Controlling nice values ---

# Run a process with lower priority (nice 10)
nice -n 10 ./my_background_job

# Change priority of running process
renice -n 5 -p <PID>

# --- Real-time scheduling ---

# Run with SCHED_FIFO, priority 50
sudo chrt -f 50 ./my_realtime_app

# Run with SCHED_RR, priority 30
sudo chrt -r 30 ./my_app

# Check scheduling policy of a process
chrt -p <PID>

# --- CPU affinity ---

# Pin process to cores 0,1
taskset -c 0,1 ./my_app

# --- Monitoring ---

# Watch context switches and scheduling
vmstat 1                           # cs column = context switches/sec
pidstat -w 1                       # Context switches per process
perf sched record                  # Record scheduling events
perf sched latency                 # Show per-process scheduling latency

# --- cgroups CPU control ---

# Limit a group to 50% of one CPU
# (using cgroups v2)
echo "50000 100000" > /sys/fs/cgroup/my_group/cpu.max
```

---

## Real-World Connection

- **Docker** uses CFS bandwidth control under the hood. `--cpus=0.5` translates to limiting the container's cgroup to 50000 us of CPU per 100000 us period. Understanding CFS helps you set Docker CPU limits correctly — setting them too low causes CFS throttling, which shows up as mysterious latency spikes.
- **Kubernetes** CPU requests and limits map to CFS parameters. A request of `500m` (500 millicores) sets the CFS shares (relative weight), while a limit of `1000m` sets the CFS bandwidth cap. Over-provisioning CPU limits is a common source of performance issues in K8s clusters.
- **Database tuning**: PostgreSQL and MySQL documentation recommend setting the database process to `nice -5` or using cgroup CPU shares to ensure the database gets priority over other workloads on the same host. For latency-sensitive queries, some operators pin database threads to dedicated cores.
- **Performance debugging**: When users report "the system is slow," the first tool is often `perf sched latency` or `pidstat -w` to see if scheduling latency is the bottleneck. High context switch rates (thousands per second per process) indicate contention or a too-small time slice.

---

## Interview Angle

**Q: How does Linux CFS ensure fairness? Walk me through the data structures and algorithm.**

A: CFS tracks each process's virtual runtime (vruntime), which measures weighted CPU consumption. The process with the lowest vruntime always runs next. Processes are stored in a red-black tree sorted by vruntime — finding the minimum is O(1) via a cached leftmost pointer, insertions and deletions are O(log n). Higher nice values make vruntime increase faster (so the process gets less CPU), lower nice values make it increase slower (more CPU). There's no fixed quantum — time slices are computed dynamically from the target latency divided by the number of runnable processes. This ensures every process eventually runs (vruntime catches up), making starvation impossible by design.

**Q: A container in Kubernetes is experiencing CPU throttling despite low average utilization. What's happening?**

A: CFS bandwidth control enforces CPU limits using a quota/period model (typically 100ms period). If a container has a 200m limit (20ms quota per 100ms), it can only use 20ms of CPU per 100ms window. If the application is bursty — idle for 80ms then needs 30ms of CPU — it gets throttled even though average utilization is low. This is because CFS doesn't bank unused quota across periods. Solutions: (1) increase the CPU limit, (2) remove the limit entirely and rely on requests for scheduling weight, (3) if available, enable the CFS burst feature (since Linux 5.14) which allows limited borrowing from future periods.

**Q: What changed from CFS to EEVDF in Linux 6.6, and why?**

A: CFS picked the process with the lowest vruntime, which ensured fairness but could delay newly-woken processes because their vruntime might be close to CPU-bound processes. EEVDF adds a virtual deadline concept: among eligible processes (whose virtual start time has passed), it picks the one with the earliest virtual deadline. This gives better latency to short-burst and I/O-bound processes without sacrificing overall fairness. The underlying accounting (vruntime, red-black tree) remains the same — the change is in the pick-next-task logic.

---

Next: [05-Process-Synchronization](../05-Process-Synchronization/README.md)
