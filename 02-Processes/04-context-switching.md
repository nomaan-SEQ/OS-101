# Context Switching

## The Core Idea

A **context switch** is the act of saving one process's state and loading another's so the CPU can run a different process. It's how the OS creates the illusion that many processes run simultaneously on a limited number of CPU cores.

Here's the uncomfortable truth: **a context switch does zero useful work**. No user computation happens during the switch. It's pure overhead — the price you pay for multitasking. Understanding this cost is what helps you make informed architecture decisions.

> **Analogy:** A context switch is like a chef cooking multiple dishes. When the timer rings for dish B, the chef saves progress on dish A (notes where they left off, sets ingredients aside), then picks up dish B's recipe and ingredients. The "saving and restoring" takes time — that's the context switch overhead.

---

## What Triggers a Context Switch

Context switches don't happen randomly. Something must cause the CPU to stop running the current process:

| Trigger | Type | Example |
|---------|------|---------|
| **Timer interrupt** | Involuntary (preemption) | Process has used its time slice (e.g., 4ms in Linux CFS) |
| **Blocking system call** | Voluntary | Process calls `read()` on a slow disk, `recv()` on a socket, `wait()` for a child |
| **Higher-priority process arrives** | Involuntary (preemption) | A real-time process becomes ready while a normal process is running |
| **Process terminates** | Voluntary | Process calls `exit()` or returns from `main()` |
| **I/O interrupt** | Involuntary | Disk read completes, waking a blocked process that has higher priority |
| **Explicit yield** | Voluntary | Process calls `sched_yield()` (rare in practice) |

**Voluntary** context switches happen when a process gives up the CPU on its own (usually by blocking). **Involuntary** context switches happen when the OS forces the process off the CPU. You can see both counts in `/proc/<PID>/status`:

```
voluntary_ctxt_switches:    450
nonvoluntary_ctxt_switches: 120
```

A high ratio of involuntary switches means the process is CPU-bound and keeps getting preempted. A high ratio of voluntary switches means the process frequently blocks on I/O.

---

## Step-by-Step: What Happens During a Context Switch

Switching from Process A to Process B involves these steps:

```
Process A (Running)                    Process B (Ready)
     │                                       │
     │  1. Interrupt/trap fires              │
     │     (timer, syscall, etc.)            │
     v                                       │
┌─────────────────────┐                      │
│  2. Save A's state  │                      │
│     to A's PCB:     │                      │
│     - All CPU regs  │                      │
│     - Program ctr   │                      │
│     - Stack pointer │                      │
│     - FPU/SSE state │                      │
│     - Flags         │                      │
└─────────┬───────────┘                      │
          │                                  │
          v                                  │
┌─────────────────────┐                      │
│  3. Update A's PCB  │                      │
│     state:          │                      │
│     Running → Ready │                      │
│     (or Waiting)    │                      │
└─────────┬───────────┘                      │
          │                                  │
          v                                  │
┌─────────────────────┐                      │
│  4. Run scheduler   │                      │
│     Pick Process B  │                      │
└─────────┬───────────┘                      │
          │                                  │
          v                                  │
┌─────────────────────┐                      │
│  5. Switch address  │                      │
│     space:          │                      │
│     Load B's page   │──────────────────────┘
│     table (CR3)     │
│     Flush TLB       │
└─────────┬───────────┘
          │
          v
┌─────────────────────┐
│  6. Load B's state  │
│     from B's PCB:   │
│     - All CPU regs  │
│     - Program ctr   │
│     - Stack pointer │
│     - FPU/SSE state │
│     - Flags         │
└─────────┬───────────┘
          │
          v
┌─────────────────────┐
│  7. Update B's PCB  │
│     state:          │
│     Ready → Running │
└─────────┬───────────┘
          │
          v
   Process B resumes
   execution exactly
   where it left off
```

---

## Timeline View

Here's a time-based view of what the CPU is doing:

```
Time ──────────────────────────────────────────────────────>

Process A    ██████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
             [executing A's code]

Context                        ▓▓▓▓▓▓
Switch                         [save A]
                               [schedule]
                               [load B]

Process B    ░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████████████
                                          [executing B's code]

             ─────────────────│──────│─────────────────────
                              ^      ^
                              │      │
                         A preempted  B starts running
                         (e.g., timer interrupt)

Legend:  ██ = useful work    ▓▓ = overhead    ░░ = not running
```

The gray `▓▓` region is pure overhead. No user-visible work happens there. The goal of OS design is to keep this overhead as small as possible.

---

## The Cost of a Context Switch

### Direct Costs

The time to save/restore registers and switch page tables:

| Operation | Approximate Time |
|-----------|-----------------|
| Save/restore general-purpose registers | ~100 ns |
| Save/restore FPU/SSE/AVX state | ~200 ns |
| Switch page tables (write CR3) | ~100 ns |
| Kernel scheduling logic | ~200 ns |
| **Total direct cost** | **~1-5 microseconds** |

On modern hardware, the direct cost is roughly 1-10 microseconds depending on the CPU and how much state needs saving.

### Indirect Costs (The Real Killer)

The indirect costs are much worse than the direct costs:

**CPU Cache Pollution**: Process A's data is hot in the L1/L2/L3 caches. When Process B starts running, it accesses completely different memory, evicting A's data. When A runs again later, it faces a wall of cache misses. This can cost hundreds of microseconds to milliseconds.

```
Before switch:  Cache full of Process A's data ████████
After switch:   Process B evicts A's cache lines
                B starts running: cold cache ░░░░░░░░
                B gradually warms cache: ░░██░░██████

When A returns: Cache full of B's data ████████ (A's data gone)
                A faces cache misses: slow ░░░░░░░░
```

**TLB Flush**: The Translation Lookaside Buffer caches virtual-to-physical address translations. When you switch page tables, the TLB entries for the old process become invalid. The new process starts with an empty (or mostly empty) TLB, meaning every memory access initially requires a full page table walk.

Modern CPUs mitigate this with:
- **ASID (Address Space ID)** / **PCID (Process Context ID)**: Tags TLB entries with a process identifier so entries from different processes can coexist
- This avoids a full TLB flush on every switch, reducing indirect cost significantly

**Branch Predictor Pollution**: Modern CPUs learn branch patterns. A context switch resets this learning for the new process.

### Cost Summary

| Cost Type | Magnitude | Can Be Mitigated? |
|-----------|----------|-------------------|
| Direct (register save/load) | 1-10 microseconds | Hardware improvements |
| Cache pollution | 10-1000+ microseconds | Larger caches, CPU affinity |
| TLB flush | 10-100 microseconds | PCID/ASID support |
| Branch predictor pollution | Varies | Modern CPUs learn fast |

---

## When Context Switches Become a Problem

### Thrashing

If the OS switches processes too frequently, it spends more time switching than doing useful work. This is called **thrashing** (though the term is more commonly used for memory thrashing with excessive paging).

```
Extreme case: switch every 10 microseconds, switch takes 5 microseconds

Time: ██▓▓▓██▓▓▓██▓▓▓██▓▓▓██▓▓▓██▓▓▓
      33% useful work, 67% overhead  <-- terrible
      
Healthy case: switch every 4 milliseconds, switch takes 5 microseconds

Time: ████████████████████████████████▓██████████████...
      99.9% useful work, 0.1% overhead  <-- acceptable
```

### How to Monitor Context Switches

```bash
# System-wide context switch rate
vmstat 1
# Look at the "cs" (context switches) column

# Per-process context switches
cat /proc/<PID>/status | grep ctxt

# Detailed with perf
perf stat -e context-switches ./my_program
```

A typical server might see 10,000-100,000 context switches per second. If you're seeing millions, something is wrong (too many threads, spinlock contention, etc.).

---

## Reducing Context Switch Overhead

| Strategy | How It Helps |
|----------|-------------|
| **CPU affinity** (`taskset`, `isolcpus`) | Pin a process to specific cores so its cache stays warm |
| **Fewer processes/threads** | Fewer things competing for CPU = fewer switches |
| **Event-driven I/O** (epoll, kqueue) | One thread handles many connections without blocking |
| **Batch processing** | Do more work per time slice before yielding |
| **PCID on x86** | Avoid TLB flush on context switch |
| **Isolate CPU cores** | Reserve cores for specific processes (real-time, latency-sensitive) |

This is why nginx (event-driven, few worker processes) handles concurrent connections more efficiently than Apache in prefork mode (one process per connection, constant context switching).

---

## Real-World Connection

**nginx vs Apache (prefork)**: Apache prefork spawns a process per connection. With 10,000 concurrent connections, that's 10,000 processes competing for maybe 8 CPU cores — thousands of context switches per second, massive cache pollution. nginx uses an event loop with a handful of worker processes, minimizing context switches. This is a key reason nginx handles higher concurrency.

**CPU pinning in databases**: High-performance databases like Redis pin their main thread to a specific CPU core using `taskset` or the `cpu_affinity` config. This keeps the CPU cache warm and avoids involuntary context switches with other processes. Same pattern appears in high-frequency trading systems.

**Container density**: When you pack many containers onto a host (common in Kubernetes), each container's processes add to the context switch load. This is why oversubscribing CPU in Kubernetes (request < limit, many pods per node) can degrade performance nonlinearly — the context switch overhead compounds.

---

## Interview Angle

**Q: What is a context switch and why is it expensive?**

A: A context switch saves the current process's CPU state (registers, program counter, stack pointer) to its PCB, runs the scheduler to pick the next process, switches page tables, and loads the new process's CPU state from its PCB. The direct cost is 1-10 microseconds. But the real cost is indirect: the new process finds CPU caches cold (filled with the previous process's data), the TLB needs repopulating, and branch prediction must relearn. These indirect costs can dwarf the direct cost by 10-100x.

**Q: What's the difference between voluntary and involuntary context switches?**

A: A voluntary context switch happens when a process gives up the CPU by making a blocking system call (read, recv, wait). An involuntary context switch happens when the OS forces the process off the CPU — typically because its time slice expired (timer interrupt) or a higher-priority process became ready. High voluntary counts indicate I/O-bound work; high involuntary counts indicate CPU-bound work competing for cores.

**Q: How would you reduce context switching overhead in a high-performance server?**

A: Use an event-driven architecture (epoll/kqueue) instead of thread-per-connection to minimize the number of threads. Pin performance-critical threads to dedicated CPU cores with CPU affinity. Use `isolcpus` to keep other processes off those cores. Monitor with `vmstat` and `perf stat` to measure the actual switch rate. Consider using DPDK or io_uring for truly latency-sensitive paths that bypass the kernel entirely.

---

**Next**: [05-process-creation-fork-exec.md](05-process-creation-fork-exec.md)
