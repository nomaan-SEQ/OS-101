# Scheduling Concepts and Criteria

Before diving into specific algorithms, you need to understand *when* scheduling decisions happen, *what* the scheduler is trying to optimize, and *how* processes actually use the CPU. These concepts form the foundation for evaluating every scheduling algorithm.

---

## When Does Scheduling Happen?

The scheduler is invoked at specific decision points during a process's lifecycle:

```
Process       +-----------+      +--------+
Created  ---->| Ready     |----->| Running |-----> Terminates
              | Queue     |<-----| on CPU  |
              +-----------+  (4) +--------+
                   ^                  |
                   |    +-----------+ |
                   +----| Waiting   |<+
                   (3)  | (I/O)     |
                        +-----------+

Scheduling decisions happen at:
(1) Process created     --> added to ready queue
(2) Process terminates  --> pick next from ready queue
(3) Process blocks (I/O)--> pick next from ready queue
(4) Timer interrupt     --> may preempt running process
```

| Trigger | What Happens | Preemptive? |
|---|---|---|
| New process created | Added to ready queue; scheduler may compare priority | Only if preemptive |
| Running process terminates | CPU is free; scheduler must pick next | Always (no choice) |
| Process blocks on I/O | CPU is free; scheduler must pick next | Always (no choice) |
| Timer interrupt fires | Scheduler checks if current process should be preempted | Only if preemptive |
| I/O completion interrupt | Waiting process moves to ready queue; may preempt current | Only if preemptive |

## Preemptive vs Non-Preemptive Scheduling

This is the most fundamental classification of scheduling approaches.

**Non-preemptive (cooperative) scheduling**: Once a process gets the CPU, it keeps it until it voluntarily gives it up — by terminating, blocking on I/O, or explicitly yielding. The scheduler only runs at decision points (2) and (3) above.

**Preemptive scheduling**: The OS can forcibly take the CPU from a running process, typically via a timer interrupt. The scheduler runs at all decision points, including (1) and (4).

```
Non-Preemptive:
  Process A runs until it decides to stop
  [  A runs...  |  A blocks  |  B runs...  |  B done  |  C runs...  ]

Preemptive:
  OS can interrupt processes mid-execution
  [  A  | timer! |  B  | timer! |  A  | A blocks |  C  | timer! |  B  ]
```

| Aspect | Non-Preemptive | Preemptive |
|---|---|---|
| CPU taken away? | No, process runs until yield/block/terminate | Yes, via timer interrupt or higher-priority arrival |
| Responsiveness | Poor — one long process delays everything | Good — no process can monopolize CPU |
| Implementation | Simpler — no need for timer interrupts | More complex — needs interrupt handling, safe context switching |
| Race conditions | Fewer — process won't be interrupted mid-operation | More — shared data can be accessed by interrupted process |
| Used by | Early systems, some embedded/RTOS | All modern general-purpose OSes (Linux, Windows, macOS) |

Modern OSes are all preemptive. Non-preemptive scheduling is a historical curiosity for general-purpose systems, but it still appears in specific contexts like cooperative multitasking in game engines or user-space coroutine schedulers.

---

## Scheduling Criteria (Metrics)

The scheduler is trying to optimize multiple metrics simultaneously, and they often conflict with each other.

### The Five Key Metrics

| Metric | Definition | Optimize | Who Cares? |
|---|---|---|---|
| **CPU Utilization** | Percentage of time the CPU is doing useful work | Maximize (target: 40-90%) | System administrators |
| **Throughput** | Number of processes completed per time unit | Maximize | Batch processing, servers |
| **Turnaround Time** | Total time from process submission to completion | Minimize | Batch jobs |
| **Waiting Time** | Total time spent sitting in the ready queue | Minimize | All processes |
| **Response Time** | Time from submission to first response produced | Minimize | Interactive users |

### How They Conflict

```
Optimizing for...        May hurt...
-------------------------------------------------
Throughput               Response time
  (batch big jobs)         (interactive jobs wait)

Response time            Throughput
  (frequent switching)     (overhead from context switches)

Fairness                 Throughput
  (give everyone a turn)   (could batch similar jobs)

CPU utilization          Response time
  (keep CPU always busy)   (may delay interactive tasks)
```

The trade-off between throughput and response time is the central tension in scheduler design. Batch-oriented systems optimize throughput; interactive systems optimize response time. Most real systems need both.

### Calculating the Metrics — An Example

```
Process  Arrival  Burst
  P1       0       8
  P2       1       4
  P3       2       2

Using FCFS (First-Come, First-Served):

Gantt Chart:
|   P1   |  P2 | P3|
0        8    12   14

                   Turnaround    Waiting     Response
Process  Arrival   (Finish-Arr)  (Turn-Burst) (Start-Arr)
  P1       0       8-0  = 8      8-8  = 0     0-0 = 0
  P2       1       12-1 = 11     11-4 = 7     8-1 = 7
  P3       2       14-2 = 12     12-2 = 10    12-2 = 10

Averages:          10.33          5.67         5.67
```

---

## CPU Bursts and I/O Bursts

Processes don't use the CPU continuously. They alternate between CPU computation and I/O operations in a characteristic pattern:

```
A typical process lifecycle:

  CPU burst   I/O wait   CPU burst   I/O wait   CPU burst
|-----------|__________|---------|__________|-----|
   12ms        50ms       5ms       30ms      3ms

Notice: CPU bursts tend to get shorter over time
```

Research shows CPU burst durations follow an exponential distribution — most bursts are very short, with a long tail of occasional long bursts. This matters because algorithms like SJF exploit this pattern.

### CPU-Bound vs I/O-Bound Processes

| Characteristic | CPU-Bound | I/O-Bound |
|---|---|---|
| CPU bursts | Long, infrequent | Short, frequent |
| I/O bursts | Short, infrequent | Long, frequent |
| Examples | Video encoding, compilation, ML training | Web server, text editor, database queries |
| Scheduling need | Needs long time slices to make progress | Needs high priority to quickly use CPU and get back to I/O |
| Bottleneck | CPU speed | I/O device speed, scheduling latency |

```
CPU-bound process:
[======CPU======][io][==========CPU==========][io][======CPU======]

I/O-bound process:
[CPU][====I/O====][CPU][======I/O======][CPU][====I/O====][CPU]
```

**Why this distinction matters for scheduling**: If you give I/O-bound processes higher priority, they'll quickly use their short CPU burst and go back to waiting on I/O, freeing the CPU for CPU-bound work. This improves both response time and CPU utilization. Most modern schedulers do exactly this — they automatically detect I/O-bound behavior and boost those processes.

---

## The Dispatcher

The scheduler *decides* which process runs next. The **dispatcher** is the component that actually performs the switch.

```
Scheduler: "Process B should run next"
            |
            v
Dispatcher: 1. Save state of Process A (registers, PC, etc.)
            2. Load state of Process B
            3. Switch to user mode
            4. Jump to the correct location in Process B
```

**Dispatch latency** is the time the dispatcher takes to stop one process and start another. It includes:
- Context switch time (saving/restoring registers)
- Switching to user mode
- Jumping to the proper program location

Dispatch latency is pure overhead — no useful work is done during a context switch. On modern hardware, a context switch typically takes 1-10 microseconds, but with cache pollution effects, the real cost can be much higher.

---

## Real-World Connection

- **Linux** uses preemptive scheduling with the CFS scheduler. You can observe scheduling behavior with `perf sched` and see context switch counts with `vmstat`.
- **Docker containers** share the host's scheduler. The `--cpu-shares` and `--cpus` flags in Docker map to scheduler parameters (CFS bandwidth control).
- **Cloud VMs** on AWS/GCP are scheduled by a hypervisor. "Noisy neighbor" problems happen when the hypervisor's scheduler gives your VM less CPU time because another VM on the same host is CPU-bound.
- **Node.js** is an interesting case: it uses cooperative scheduling in user space (event loop) on top of the OS's preemptive scheduler. A long-running synchronous function blocks the event loop because Node won't preempt it.

---

## Interview Angle

**Q: A user reports that their interactive application becomes unresponsive when a background batch job runs. What scheduling concept explains this, and how would you fix it?**

A: This is likely a priority/scheduling class issue. The batch job is CPU-bound and consuming the CPU, delaying the interactive application's short CPU bursts. Fixes include: (1) lowering the batch job's priority with `nice`/`renice`, (2) using `cgroups` to limit the batch job's CPU share, or (3) using `ionice` if it's also contending on I/O. Modern schedulers like CFS handle this reasonably well, but a truly CPU-hungry process with the same priority will still affect responsiveness.

**Q: What is dispatch latency and why does it matter?**

A: Dispatch latency is the time it takes to stop one process and start another. It's pure overhead — during a context switch, no useful work happens. While the raw CPU state save/restore takes 1-10 microseconds, the real cost is higher due to cache, TLB, and pipeline effects. This is why choosing the right time quantum in Round Robin matters — too small and you spend too much time context switching.

---

Next: [FCFS, SJF, and Priority Scheduling](02-fcfs-sjf-priority-scheduling.md)
