# User Threads vs Kernel Threads

Where threads are managed -- in user space or in the kernel -- fundamentally determines what they can and can't do. This distinction explains why early threading implementations had crippling limitations and why modern systems work the way they do.

## User-Level Threads

User-level threads are managed entirely by a library in user space. The kernel has no idea they exist. As far as the kernel is concerned, there's just one process (with one schedulable entity).

> **Analogy:** Think of user-level threads like employees in an office that has only one phone line to the outside world (the kernel). A floor manager (the thread library) juggles who gets to work when, but to anyone calling in, it looks like one office. If one employee ties up the phone line with a long call (a blocking syscall), nobody else can make calls until it's free.

```
USER SPACE                          KERNEL SPACE
┌─────────────────────────┐         ┌──────────────────┐
│  Application            │         │                  │
│                         │         │  Kernel sees     │
│  ┌───┐ ┌───┐ ┌───┐     │         │  ONE process     │
│  │ T1│ │ T2│ │ T3│     │         │                  │
│  └─┬─┘ └─┬─┘ └─┬─┘     │         │  ┌────────────┐  │
│    │     │     │        │         │  │  Process    │  │
│  ┌─┴─────┴─────┴──┐    │         │  │  (single   │  │
│  │  Thread Library │    │ ──────> │  │   kernel   │  │
│  │  (scheduler,    │    │         │  │   thread)  │  │
│  │   context switch│    │         │  └────────────┘  │
│  │   in user space)│    │         │                  │
│  └─────────────────┘    │         │                  │
└─────────────────────────┘         └──────────────────┘
```

### How They Work

1. A user-space library (e.g., GNU Pth, old Java green threads) maintains its own thread table
2. The library implements its own scheduler and context-switching code
3. Switching between threads happens entirely in user space -- no system call needed
4. The library saves/restores registers and swaps stacks, just like the kernel would

### Advantages

| Advantage | Explanation |
|-----------|-------------|
| Fast creation | No syscall needed -- just allocate a stack and add an entry to the thread table |
| Fast context switch | User-space register swap is ~10x faster than a kernel context switch |
| Portable | Works on any OS, even one without thread support |
| Custom scheduling | Application can implement its own scheduling policy |

### The Fatal Flaw

**If any user thread makes a blocking system call, ALL threads in that process block.**

```
User threads T1, T2, T3 (one kernel thread):

  T1: read(socket)   <-- blocks in kernel
  T2: (wants to run)  <-- STUCK, kernel sees process as blocked
  T3: (wants to run)  <-- STUCK, same reason

  Kernel view: "Process is waiting for I/O. Nothing to schedule."
```

The kernel doesn't know about T2 and T3. It just sees one process blocked on I/O. There's no way to run the other user threads while T1 waits.

> **Analogy:** User-level threads sharing one kernel thread are like multiple people sharing a single phone line. If one person is on a long call (blocked), nobody else can make calls -- even if they're ready to talk.

### Other Limitations

- **No multicore parallelism**: The kernel schedules one kernel thread, so only one CPU can run the process's threads at a time
- **Page faults block everything**: If one thread triggers a page fault, the kernel blocks the whole process while it fetches the page from disk
- **Priority inversions**: The kernel can't prioritize individual user threads -- it only sees the process as a whole

### Historical Examples

- **GNU Pth** (Portable Threads): User-level threading library for C
- **Early Java (pre-1.2) green threads**: JVM managed threads in user space
- **Ruby MRI before native threads**: Used green threads internally

## Kernel-Level Threads

Kernel-level threads are managed directly by the OS kernel. The kernel maintains a thread table and schedules each thread independently.

```
USER SPACE                          KERNEL SPACE
┌─────────────────────────┐         ┌──────────────────┐
│  Application            │         │                  │
│                         │         │  Kernel sees     │
│  ┌───┐ ┌───┐ ┌───┐     │         │  THREE threads   │
│  │ T1│ │ T2│ │ T3│     │         │                  │
│  └─┬─┘ └─┬─┘ └─┬─┘     │         │  ┌──┐ ┌──┐ ┌──┐ │
│    │     │     │        │         │  │KT│ │KT│ │KT│ │
│    │     │     │        │         │  │ 1│ │ 2│ │ 3│ │
│    └─────┼─────┘        │ ──────> │  └──┘ └──┘ └──┘ │
│          │              │         │   │     │     │  │
│          │              │         │  CPU0  CPU1  CPU2 │
└─────────────────────────┘         └──────────────────┘
```

### How They Work

1. Thread creation goes through a system call (e.g., `clone()` on Linux)
2. Kernel maintains thread state, stacks, and scheduling information
3. Kernel scheduler treats each thread as an independent schedulable entity
4. Threads can be distributed across multiple CPU cores

### Advantages

| Advantage | Explanation |
|-----------|-------------|
| True parallelism | Different threads can run simultaneously on different CPUs |
| Independent blocking | One thread blocking on I/O doesn't affect others |
| Kernel-aware scheduling | Kernel can make informed priority and CPU affinity decisions |
| Page fault handling | Only the faulting thread blocks, others continue |

### Disadvantages

| Disadvantage | Explanation |
|-------------|-------------|
| Slower creation | Requires a system call, kernel allocates resources |
| Slower context switch | Must trap into kernel, switch kernel stacks, potentially flush TLB |
| Higher memory overhead | Each thread needs a kernel stack (~8KB on Linux) in addition to user stack |
| Limited count | Kernel has finite resources; creating 100,000 threads may exhaust memory |

### Modern Examples

- **Linux NPTL** (Native POSIX Threads Library): 1:1 mapping, each `pthread` is a kernel thread
- **Windows threads**: `CreateThread()` creates a kernel-scheduled thread
- **macOS/iOS threads**: Based on Mach threads, kernel-managed

## Side-by-Side Comparison

| Dimension | User-Level Threads | Kernel-Level Threads |
|-----------|-------------------|---------------------|
| Managed by | User-space library | OS kernel |
| Kernel awareness | Invisible to kernel | Fully visible |
| Creation speed | Very fast (no syscall) | Slower (syscall required) |
| Context switch | Fast (user space only) | Slower (kernel involvement) |
| Blocking I/O | Blocks all threads | Blocks only that thread |
| Multicore use | No (single kernel thread) | Yes (scheduled on any core) |
| Page fault impact | Blocks all threads | Blocks only faulting thread |
| Memory overhead | Low | Higher (kernel stack per thread) |
| Max thread count | Very high (limited by user memory) | Lower (kernel resource limits) |
| Scheduling control | Application decides | Kernel decides |
| Portability | High (runs on any OS) | OS-specific |

## Why Kernel Threads Won

The advantages of user-level threads (speed, portability) turned out to matter less than the disadvantages (no parallelism, blocking problems). As multicore CPUs became standard in the 2000s, the inability to use multiple cores was a dealbreaker.

**The progression:**
```
1990s: User-level threads popular (single-core CPUs, speed matters most)
  │
  ├── Problem: blocking I/O kills everything
  ├── Problem: can't use multiple cores
  │
2000s: Kernel threads become standard (multi-core CPUs arrive)
  │
  ├── Linux NPTL (2003): fast 1:1 kernel threads
  ├── Kernel thread overhead drops as hardware improves
  │
2010s+: Hybrid approaches emerge (goroutines, virtual threads)
  │
  └── Get user-thread speed with kernel-thread capabilities
      (More on this in the threading models and goroutines sections)
```

Modern systems almost universally use kernel threads. The overhead of a system call for thread creation is negligible on modern hardware, and the ability to use multiple cores is essential. The few remaining use cases for user-level threading have evolved into more sophisticated hybrid models (goroutines, Java virtual threads) covered later in this chapter.

## Real-World Connection

### Linux's NPTL Revolution

Before NPTL (2003), Linux used LinuxThreads, which had serious problems: each thread had a different PID, signals were broken, and performance was poor. NPTL provided true 1:1 kernel threading with:

- Thread creation in ~50 microseconds (fast enough for most use cases)
- Proper POSIX compliance
- Efficient futex-based synchronization

This is why you can confidently use `pthread_create()` on modern Linux without worrying about whether to use user or kernel threads -- you're getting kernel threads, and they're fast enough.

### The Python GIL Connection

Python's Global Interpreter Lock (GIL) creates an interesting situation: Python threads are real kernel threads (via pthreads), but the GIL ensures only one executes Python bytecode at a time. This effectively gives you the worst of both worlds for CPU-bound work -- kernel thread overhead without multicore parallelism. That's why CPU-bound Python uses `multiprocessing` instead of `threading`.

## Interview Angle

**Q: Why can't user-level threads take advantage of multiple CPUs?**

A: The kernel schedules kernel threads, not user threads. With user-level threading, the kernel sees one process with one kernel thread. It can only schedule that one entity on one CPU at a time. The user-space thread library can switch between user threads, but only one executes at any moment. You need kernel threads (one per CPU minimum) to achieve true parallelism.

**Q: If user-level threads are faster to create and switch, why don't we use them?**

A: The speed advantage is irrelevant if your threads can't run in parallel and any blocking I/O freezes everything. On a modern 16-core server, user-level threads can only use 1/16th of the available compute. Kernel threads are fast enough on modern hardware (NPTL creates threads in microseconds), and they work correctly with blocking I/O and multicore systems.

**Q: What's the relationship between Python's GIL and this user vs kernel thread distinction?**

A: Python threads are kernel threads (they can be scheduled on different CPUs). The GIL is an application-level lock that serializes Python bytecode execution. So the kernel is doing its job (scheduling threads independently), but the Python interpreter adds its own constraint. This is different from user-level threads where the kernel fundamentally can't schedule threads independently. The fix for Python is multiprocessing (separate processes, separate GILs) or using C extensions that release the GIL.

---

**Next:** [Threading Models](03-threading-models.md)
