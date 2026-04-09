# CPU Scheduling

The CPU scheduler is one of the most critical components of an operating system. It decides which process or thread gets the CPU and for how long. Every time you switch tabs, stream a video while downloading a file, or run a build while browsing docs, the scheduler is making thousands of decisions per second to keep everything running smoothly. A good scheduler makes a single-core machine feel responsive; a bad one makes a 64-core server feel sluggish.

## Why This Matters

- **Responsiveness**: When you press a key, scheduling determines whether the response feels instant or laggy. Interactive applications need fast turnaround.
- **Throughput**: Servers processing thousands of requests per second depend on scheduling to maximize the amount of work completed in a given time window.
- **Fairness**: Without careful scheduling, one runaway process can starve everything else. The scheduler is the referee that keeps the system balanced.
- **Real systems**: Linux's CFS, Windows' priority scheduler, and real-time schedulers in embedded systems all make different trade-offs. Understanding scheduling helps you diagnose performance problems, tune systems, and answer "why is my application slow?"

## Prerequisites

| Prerequisite | Why You Need It |
|---|---|
| [02 - Processes](../02-Processes/README.md) | Scheduling operates on processes — you need to understand process states and context switching |
| [03 - Threads](../03-Threads/README.md) | Modern schedulers schedule threads, not processes. Thread models affect scheduling behavior |

## Reading Order

| # | Topic | What You'll Learn |
|---|---|---|
| 1 | [Scheduling Concepts and Criteria](01-scheduling-concepts-and-criteria.md) | When scheduling happens, key metrics (turnaround, waiting, response time), CPU vs I/O bursts |
| 2 | [FCFS, SJF, and Priority Scheduling](02-fcfs-sjf-priority-scheduling.md) | Three fundamental algorithms, convoy effect, starvation, aging |
| 3 | [Round Robin and Multilevel Queues](03-round-robin-and-multilevel-queues.md) | Time-sharing scheduling, multilevel queues, feedback queues that adapt to process behavior |
| 4 | [Multiprocessor and Multicore Scheduling](04-multiprocessor-and-multicore-scheduling.md) | SMP, processor affinity, load balancing, NUMA-aware scheduling |
| 5 | [Real-Time Scheduling](05-real-time-scheduling.md) | Hard vs soft real-time, RMS, EDF, priority inversion, the Mars Pathfinder bug |
| 6 | [Linux CFS and Real-World Schedulers](06-linux-cfs-and-real-world-schedulers.md) | How Linux CFS works, vruntime, red-black trees, EEVDF, Windows and macOS schedulers |

## Key Interview Questions

1. **What is the difference between preemptive and non-preemptive scheduling?** In preemptive scheduling, the OS can forcibly take the CPU from a running process (e.g., when a timer interrupt fires or a higher-priority process arrives). In non-preemptive scheduling, a process keeps the CPU until it voluntarily yields, blocks on I/O, or terminates.

2. **Why is SJF optimal for average waiting time but impractical?** SJF minimizes average waiting time because short jobs finish first, reducing the cumulative wait for all subsequent jobs. It's impractical because you cannot know the length of the next CPU burst in advance — you can only estimate it.

3. **What is starvation and how does aging solve it?** Starvation occurs when a low-priority process never gets CPU time because higher-priority processes keep arriving. Aging solves this by gradually increasing the priority of waiting processes, ensuring they eventually reach high enough priority to run.

4. **How does Linux CFS achieve fairness without a fixed time quantum?** CFS tracks each process's virtual runtime (vruntime) — a measure of how much CPU time it has received weighted by priority. The process with the lowest vruntime always runs next, stored in a red-black tree for O(log n) lookup. Time slices are dynamic, calculated from the number of runnable processes.

5. **What is the convoy effect and which scheduling algorithm suffers from it?** The convoy effect occurs in FCFS scheduling when a long-running CPU-bound process holds the CPU while many shorter processes wait behind it. This dramatically inflates average waiting time and wastes I/O device utilization because I/O-bound processes cannot run their quick CPU bursts and get back to I/O.

---

Next: [Scheduling Concepts and Criteria](01-scheduling-concepts-and-criteria.md)
