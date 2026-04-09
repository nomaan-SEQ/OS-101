# 05 - Concurrency and Synchronization

When multiple threads or processes share resources — memory, files, data structures — you need synchronization to prevent chaos. Without it, your program produces different results on different runs, corrupts data silently, and creates bugs that are nearly impossible to reproduce.

This is arguably **the most important OS topic for interviews**. Expect multiple questions on locks, deadlocks, and classic synchronization problems. Companies love this topic because it tests whether you truly understand concurrent systems — not just theory, but the traps that catch even experienced engineers.

## Why This Matters

- **Every backend system is concurrent.** Web servers handle thousands of requests simultaneously. Databases manage concurrent transactions. Even your phone runs dozens of threads.
- **Concurrency bugs are the hardest bugs.** They're non-deterministic, hard to reproduce, and often only manifest under load in production.
- **This knowledge separates junior from senior engineers.** Understanding synchronization deeply means you can design systems that are both correct and performant.
- **Cloud-native systems amplify these problems.** Microservices, distributed databases, and container orchestration all deal with concurrency at massive scale.

## Prerequisites

| Section | Why You Need It |
|---------|----------------|
| [02 - Processes](../02-Processes/README.md) | Processes are independent execution units that can share resources |
| [03 - Threads](../03-Threads/README.md) | Threads share memory within a process — the primary source of concurrency issues |
| [04 - CPU Scheduling](../04-CPU-Scheduling/README.md) | The scheduler decides when threads run, which directly causes race conditions |

## Reading Order

| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 1 | Race Conditions & Critical Sections | [01-race-conditions-and-critical-sections.md](01-race-conditions-and-critical-sections.md) | Why concurrent access breaks things, and the rules for fixing it |
| 2 | Mutexes & Spinlocks | [02-mutexes-and-spinlocks.md](02-mutexes-and-spinlocks.md) | The two fundamental lock types and when to use each |
| 3 | Semaphores | [03-semaphores.md](03-semaphores.md) | Counting-based synchronization for resource pools |
| 4 | Monitors & Condition Variables | [04-monitors-and-condition-variables.md](04-monitors-and-condition-variables.md) | High-level synchronization and waiting for conditions |
| 5 | Classic Problems | [05-classic-problems-producer-consumer-readers-writers-dining-philosophers.md](05-classic-problems-producer-consumer-readers-writers-dining-philosophers.md) | The three canonical problems you'll be asked about |
| 6 | Deadlocks | [06-deadlocks-detection-prevention-avoidance.md](06-deadlocks-detection-prevention-avoidance.md) | When threads wait forever, and what to do about it |
| 7 | Lock-Free Programming | [07-lock-free-and-wait-free-programming.md](07-lock-free-and-wait-free-programming.md) | Advanced: synchronization without locks |

## Key Interview Questions

1. **What is a race condition and how do you prevent it?**
   A race condition occurs when the correctness of a program depends on the relative timing of thread execution. Prevent it with mutual exclusion (locks, semaphores) or by avoiding shared mutable state.

2. **Mutex vs Spinlock — when would you use each?**
   Mutex puts the thread to sleep (good for long waits, single-core). Spinlock busy-waits (good for very short critical sections on multi-core). The Linux kernel uses spinlocks extensively for short, interrupt-context locks.

3. **What are the four conditions for deadlock?**
   Mutual exclusion, hold and wait, no preemption, and circular wait. All four must hold simultaneously. Break any one to prevent deadlock.

4. **Solve the Producer-Consumer problem with semaphores.**
   Use three semaphores: `mutex` (binary, protects buffer), `empty` (counts empty slots, init N), `full` (counts full slots, init 0). Producer waits on empty, locks mutex, adds item, unlocks mutex, signals full. Consumer does the reverse.

5. **Why do you need a while loop (not if) when using condition variables?**
   Because of spurious wakeups and the fact that another thread may have changed the condition between signal and the woken thread re-acquiring the lock. Always re-check the condition.

6. **What is the difference between a semaphore and a mutex?**
   A mutex has ownership — only the thread that locked it can unlock it. A semaphore has no ownership — any thread can call signal(). Mutexes are for mutual exclusion; semaphores are for signaling and resource counting.

7. **How does Compare-And-Swap (CAS) enable lock-free programming?**
   CAS atomically checks if a memory location holds an expected value and, if so, updates it. This lets threads make progress without locks. If CAS fails (another thread changed the value), the thread retries.
