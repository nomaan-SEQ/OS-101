# Thrashing and Working Set Model

Everything we've discussed so far works well when there's enough physical memory to hold the pages that processes are actively using. But what happens when there isn't? The system enters a state called **thrashing** — where it spends almost all its time swapping pages in and out of disk, and almost no time doing useful work. Understanding thrashing and the working set model is essential for diagnosing real-world performance problems in servers, containers, and cloud deployments.

## What Is Thrashing?

Thrashing occurs when a process (or the entire system) spends more time handling page faults than executing instructions. Each page fault takes milliseconds (disk I/O), while a useful instruction takes nanoseconds. When page faults dominate, the system effectively stalls.

### How Thrashing Develops

```
  1. System has many active processes
  2. Combined memory needs exceed physical RAM
  3. Page faults increase (pages being stolen from each process)
  4. Processes block waiting for page I/O
  5. CPU utilization drops (all processes are waiting for disk)
  6. OS scheduler sees low CPU utilization, thinks "not enough work"
  7. OS admits MORE processes (trying to increase utilization)
  8. MORE processes = MORE memory pressure = MORE page faults
  9. System collapses into a page fault spiral
```

### The Cliff

CPU utilization vs degree of multiprogramming has a characteristic cliff shape:

```
  CPU
  Utilization
  100% |
       |         .....
       |       .       .
       |     .           .
       |    .              .
       |   .                 \
       |  .                    \
       | .                       \
       |.                          \
       |                             \___________
   0%  +---------------------------------------------->
       Low            Optimal           Too Many
              Number of Processes (Multiprogramming)

                                  ^
                                  |
                            THRASHING CLIFF
                      (adding processes makes
                       everything SLOWER)
```

The system performs well as you add processes — up to a point. Beyond that point, performance doesn't degrade gradually. It falls off a cliff. CPU utilization can drop from 90% to under 10% with just a few more processes than the system can handle.

## Why Thrashing Happens

The root cause is simple: **the total working sets of all processes exceed available physical memory**.

Consider: Process A needs pages {1, 2, 3, 4, 5} to make progress. Process B needs pages {6, 7, 8, 9, 10}. If there are only 6 frames available, at most one process can have its full working set in memory. The other process will constantly fault, bringing in pages that evict the first process's pages, which then faults in turn.

```
  Available frames: 6
  Process A needs:  5 pages
  Process B needs:  5 pages
  Total needed:     10 pages

  Round 1: A has {1,2,3,4,5}, B has {6}
  B faults on 7 --> evicts A's page 1
  B faults on 8 --> evicts A's page 2
  B faults on 9 --> evicts A's page 3
  B faults on 10 --> evicts A's page 4
  Now A faults on 1 --> evicts B's page 6
  A faults on 2 --> evicts B's page 7
  ...endless cycle of stealing each other's pages
```

## The Working Set Model

The **working set** of a process is the set of pages it has accessed within a recent time window (delta). It represents the memory the process is actively using right now.

```
  W(t, delta) = set of pages accessed in the time interval [t - delta, t]
```

```
  Time --->
  Page accesses: 1 2 1 3 2 4 5 4 5 3 6 7 6 7 6

  Working set at time 15 with delta=5:
  Pages accessed in last 5 accesses: {3, 6, 7, 6, 7, 6}
  W = {3, 6, 7}  (3 unique pages)

  Working set at time 15 with delta=10:
  Pages accessed in last 10 accesses: {4, 5, 4, 5, 3, 6, 7, 6, 7, 6}
  W = {3, 4, 5, 6, 7}  (5 unique pages)
```

The **working set size** (WSS) is the number of pages in the working set. It varies over time as the process moves between different phases of execution (initialization, main processing, cleanup).

### The Working Set Principle

The key insight:

```
  D = sum of all processes' working set sizes
  M = total physical frames available

  If D > M --> THRASHING
  If D <= M --> system operates normally
```

**The solution to thrashing: only run processes whose working sets collectively fit in memory.** If the total demand D exceeds available memory M, the OS should suspend some processes (reduce the degree of multiprogramming) until the working sets fit.

## Detecting Thrashing

### Page Fault Frequency (PFF)

Monitor the page fault rate per process:

```
  Page Fault
  Rate
       |
  High |  ............  <-- Upper threshold: too many faults
       |                     (allocate more frames to this process)
       |
       |  ~~~~~~~~~~~~  <-- Acceptable range
       |
  Low  |  ............  <-- Lower threshold: too few faults
       |                     (process has more frames than it needs,
       |                      reclaim some)
       +---------------------------------------------->
               Time
```

- If a process's fault rate exceeds the upper threshold, give it more frames
- If a process's fault rate drops below the lower threshold, take frames away (it has more than it needs)
- If no frames are available to give, suspend a process

### System-Level Monitoring

On Linux, monitor these indicators:

```
  $ vmstat 1
  procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
   r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
   1  0    0   2145932  31284 4712344    0    0     0     0  156  289  1  0 99  0  0
   1  0    0   2145808  31284 4712344    0    0     0     0  148  276  1  0 99  0  0
   3  5  524288  12340  1024  102400  8420  9200  8420  9200 5200 12400  2 15  3 80  0
                                        ^     ^                              ^  ^
                                        |     |                              |  |
                                  swap in  swap out                     sys  wait
                                  (pages/s) (pages/s)                  time  time
```

Warning signs of thrashing:
- **si/so (swap in/out)**: high sustained values mean constant paging to/from disk
- **wa (I/O wait)**: high values mean CPUs are idle waiting for disk I/O
- **free**: near zero available memory
- **b (blocked)**: many processes blocked on I/O

Other useful tools:

```
  # Page fault statistics per process
  $ ps -eo pid,min_flt,maj_flt,cmd | sort -k3 -rn | head
  # min_flt = minor faults (page in memory, just not mapped)
  # maj_flt = major faults (page on disk, requires I/O)

  # Memory pressure (cgroups v2)
  $ cat /proc/pressure/memory
  some avg10=0.50 avg60=1.20 avg300=0.80 total=12345678
  full avg10=0.10 avg60=0.30 avg300=0.15 total=2345678

  # System-wide memory info
  $ sar -B 1        # page statistics
  $ sar -W 1        # swap statistics
```

## Solutions to Thrashing

| Solution | How It Works | When to Use |
|----------|-------------|-------------|
| **Add RAM** | More physical memory = larger working sets fit | The most effective solution for memory-bound workloads |
| **Reduce processes** | Fewer processes = lower total working set demand | When you're running more processes than the system can handle |
| **Memory limits (cgroups)** | Cap each process/container's memory usage | Container and cloud environments |
| **Better algorithms** | Applications that use less memory or have better locality | Application-level optimization |
| **Swap to SSD** | SSD swap is ~100x faster than HDD swap | Reduces the pain of page faults but doesn't eliminate the problem |
| **Disable swap** | Force OOM kills instead of thrashing | When predictable latency matters more than availability |

## Thrashing in Containers and Cloud

In modern deployments, thrashing manifests differently because of containerization and orchestration:

**Memory limits with cgroups**: containers have hard memory limits. When a container exceeds its limit, the OOM killer terminates it (you see `OOMKilled` in Kubernetes). This is actually _better_ than thrashing — a quick death is preferable to a slow, system-wide degradation.

```
  # Kubernetes pod spec
  resources:
    requests:
      memory: "256Mi"     # Scheduler uses this for placement
    limits:
      memory: "512Mi"     # Hard limit -- OOM kill if exceeded
```

**Why Kubernetes disabled swap by default**: Kubernetes historically required swap to be disabled on nodes. The reason is exactly thrashing — if a pod starts swapping, its performance becomes unpredictable, health checks fail, and the cascading effects can destabilize the entire node. It's better to OOM-kill the pod and reschedule it elsewhere.

(Kubernetes 1.22+ added alpha support for swap, but it's still not recommended for production workloads.)

**Memory overcommit in VMs**: cloud providers overcommit physical RAM across VMs (sell more memory than physically exists). This works because most VMs don't use their full allocation simultaneously. But when too many VMs become active at once — you get the "noisy neighbor" problem, which is thrashing at the hypervisor level.

## Adding RAM vs Adding CPU

A common interview question: if your system is slow, should you add more RAM or more CPU?

```
  Memory-bound system (thrashing):
  - High swap I/O, high page faults
  - CPU wait time is high, CPU utilization is LOW
  - Adding CPU: NO HELP (CPUs are idle, waiting for I/O)
  - Adding RAM: HELPS (fits more working sets, reduces faults)

  CPU-bound system:
  - CPU utilization near 100%, low swap/page faults
  - Adding RAM: NO HELP (memory isn't the bottleneck)
  - Adding CPU: HELPS (more compute capacity)
```

The key metric is **what the CPUs are waiting for**. If they're waiting for disk I/O (memory swapping), more CPUs won't help — you need more RAM.

## Real-World Connection

- **Database memory tuning**: PostgreSQL's `shared_buffers` should be large enough to hold the working set of active queries. If it's too small, the database constantly reads from disk (its own form of "page faults"). MySQL's `innodb_buffer_pool_size` is the same concept — undersized buffer pools thrash.
- **JVM heap sizing**: Java applications have a working set that includes the live object set plus GC overhead. If the heap is too small, the GC runs constantly (GC thrashing), which is the application-level equivalent of OS thrashing.
- **Browser tabs**: ever noticed your browser slowing to a crawl with too many tabs? Each tab has a working set. When total tab memory exceeds RAM, the system thrashes. Modern browsers mitigate this by suspending inactive tabs (reducing the effective working set).
- **Monitoring and alerting**: production systems should alert on high swap usage or high page fault rates, not just high CPU. A system that's thrashing might show low CPU utilization, misleading you into thinking it's under-utilized when it's actually drowning.

## Interview Angle

**Q: What is thrashing and what causes it?**

Thrashing is when a system spends more time paging (swapping pages between RAM and disk) than executing useful instructions. It happens when the combined working sets of all active processes exceed physical memory. Each process constantly steals pages from other processes, triggering a cascade of page faults. CPU utilization drops because all processes are waiting for disk I/O, and the system becomes effectively unresponsive.

**Q: What is the working set of a process?**

The working set is the set of pages a process actively uses within a recent time window. It represents the memory footprint needed for the process to make progress without excessive page faults. If the system can't keep a process's working set in physical memory, that process will thrash. The total of all processes' working sets must fit in physical memory to avoid system-wide thrashing.

**Q: How would you diagnose and fix thrashing on a Linux server?**

Diagnose with `vmstat` (look for high si/so swap traffic, high wa I/O wait, low CPU utilization), `free` (check available memory), and `sar -B` (page fault rates). The fix depends on the cause: add RAM if the workload legitimately needs more memory, reduce the number of processes if overloaded, set memory limits via cgroups to prevent individual processes from consuming too much, or tune application memory usage (database buffer pools, JVM heap sizes). In containers, prefer OOM kills over thrashing — set memory limits and let Kubernetes reschedule failed pods.

**Q: Why does adding RAM help more than adding CPU when a system is thrashing?**

During thrashing, CPUs are idle waiting for disk I/O (page-in/page-out operations). Adding more CPUs doesn't help because the bottleneck is memory, not compute. Adding RAM allows more working sets to fit in physical memory, reducing page faults and the associated disk I/O. Once page faults drop, the existing CPUs can return to full utilization.

---

Next: [09 - Memory-Mapped Files and mmap](09-memory-mapped-files-and-mmap.md)
