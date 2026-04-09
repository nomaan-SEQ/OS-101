# Threading Models

Threading models describe how user threads map to kernel threads. This mapping determines whether your threads can run in parallel, how they handle blocking, and what overhead you pay. Three models have been tried historically -- only one dominates today.

## Many-to-One Model

Many user threads map to a single kernel thread. This is essentially the user-level threading model from the previous section.

```
     User Space              Kernel Space

  ┌──┐ ┌──┐ ┌──┐ ┌──┐
  │U1│ │U2│ │U3│ │U4│       User threads
  └──┘ └──┘ └──┘ └──┘
    \    \   /   /
     \    \ /   /
      \    |   /
       \   |  /
        \  | /
         \ |/
        ┌────┐
        │ K1 │               Single kernel thread
        └────┘
           │
        ┌──┴──┐
        │ CPU │              Can only use one CPU
        └─────┘
```

### How It Works

- A user-space thread library manages all thread creation, scheduling, and context switching
- The kernel sees one process with one schedulable entity
- The library multiplexes user threads onto that single kernel thread

### Characteristics

| Property | Value |
|----------|-------|
| Parallelism | **None** -- only one CPU can execute threads |
| Blocking behavior | One thread blocks on I/O, all block |
| Thread creation | Very fast (no syscall) |
| Context switch | Very fast (user space only) |
| Complexity | Simple |

### Who Used This

- **GNU Pth** (Portable Threads)
- **Early Java green threads** (pre-JDK 1.2)
- **Early Ruby** (before native thread support)

### Why It Failed

The inability to use multiple CPUs became unacceptable as multicore hardware became standard. A program with 100 user threads on a 16-core machine still only uses one core.

## One-to-One Model

Each user thread maps to exactly one kernel thread. When you create a thread, the OS creates a corresponding kernel thread.

```
     User Space              Kernel Space

  ┌──┐ ┌──┐ ┌──┐ ┌──┐
  │U1│ │U2│ │U3│ │U4│       User threads
  └──┘ └──┘ └──┘ └──┘
   │    │    │    │
   │    │    │    │          1:1 mapping
   │    │    │    │
  ┌┴─┐ ┌┴─┐ ┌┴─┐ ┌┴─┐
  │K1│ │K2│ │K3│ │K4│       Kernel threads
  └──┘ └──┘ └──┘ └──┘
   │    │    │    │
  ┌┴──┐┌┴──┐┌┴──┐┌┴──┐
  │CPU││CPU││CPU││CPU│     Can use all CPUs
  │ 0 ││ 1 ││ 2 ││ 3 │
  └───┘└───┘└───┘└───┘
```

### How It Works

- Each `pthread_create()` call results in a `clone()` system call on Linux
- The kernel maintains a thread control block for each thread
- The kernel scheduler independently schedules each thread on any available CPU
- Blocking one thread has no effect on others

### Characteristics

| Property | Value |
|----------|-------|
| Parallelism | **Full** -- threads run on different CPUs simultaneously |
| Blocking behavior | Only the blocked thread waits |
| Thread creation | Moderate (syscall required) |
| Context switch | Moderate (kernel involvement) |
| Complexity | Moderate |
| Thread limit | Constrained by kernel resources (~32K-100K typical) |

### Who Uses This

- **Linux NPTL** (since 2003) -- the standard on modern Linux
- **Windows** -- `CreateThread()` / `_beginthreadex()`
- **macOS / iOS** -- Mach-based kernel threads
- **FreeBSD** -- 1:1 threading model

### Why It Won

Modern hardware made the syscall overhead negligible. NPTL can create a thread in ~50 microseconds. The simplicity of a 1:1 mapping (no complex multiplexing logic, kernel handles all scheduling) combined with true multicore parallelism made this the clear winner.

The only real limitation: you can't create millions of threads. Each kernel thread needs a kernel stack (~8KB) plus a user stack (~1-8MB default). At a million threads, that's 8GB+ of kernel stack alone. This limitation drove the creation of hybrid models (goroutines, virtual threads).

## Many-to-Many Model

M user threads map to N kernel threads, where M >= N. A user-space scheduler multiplexes user threads onto a smaller pool of kernel threads.

```
     User Space              Kernel Space

  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
  │U1│ │U2│ │U3│ │U4│ │U5│ │U6│   6 user threads
  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘
    \    |   /     \    |   /
     \   |  /       \   |  /       M:N multiplexing
      \  | /         \  | /
       \ |/           \ |/
      ┌────┐        ┌────┐
      │ K1 │        │ K2 │         2 kernel threads
      └────┘        └────┘
        │              │
      ┌─┴──┐        ┌──┴─┐
      │CPU │        │CPU │         Can use multiple CPUs
      │ 0  │        │ 1  │
      └────┘        └────┘
```

### How It Works

1. The application creates M user threads
2. A runtime or library maps them onto N kernel threads (N <= M, typically N = number of CPUs)
3. When a user thread blocks, the runtime can switch another user thread onto that kernel thread
4. The runtime scheduler decides which user threads run on which kernel threads

### Characteristics

| Property | Value |
|----------|-------|
| Parallelism | Yes -- up to N kernel threads worth |
| Blocking behavior | Runtime can reschedule around blocks (if done right) |
| Thread creation | Fast for user threads (no syscall per thread) |
| Context switch | Fast for user thread switches, kernel speed for kernel switches |
| Complexity | **Very high** -- two schedulers interacting |
| Thread limit | Very high (user threads are cheap) |

### Who Used This

- **Solaris LWP** (Lightweight Processes) -- the classic example
- **HP-UX** -- historical implementation
- **IRIX** -- SGI's Unix variant

### Why It Didn't Win (in its original form)

The complexity was brutal. Two schedulers (user-space and kernel) had to coordinate:

- What happens when a kernel thread blocks on a page fault? The user-space scheduler doesn't know.
- Solaris solved this with "scheduler activations" (upcalls from kernel to user space), but the implementation was fragile.
- Debugging became a nightmare -- which user thread is on which kernel thread right now?

The 1:1 model was simpler and "good enough" once kernel thread creation became fast. However, the M:N idea didn't die -- it evolved into goroutines and virtual threads.

## Model Comparison

| Dimension | Many-to-One | One-to-One | Many-to-Many |
|-----------|-------------|------------|--------------|
| Mapping | M user : 1 kernel | 1 user : 1 kernel | M user : N kernel |
| Parallelism | None | Full | Partial to full |
| Blocking impact | All threads block | Only one blocks | Can work around blocks |
| Creation overhead | Very low | Moderate | Low for user threads |
| Max threads | Very high | Limited (~100K) | Very high |
| Implementation | Simple | Simple | Very complex |
| Modern examples | (Historical) | Linux, Windows, macOS | Go runtime, Java Loom |
| Status | **Dead** | **Dominant** | **Revived in new forms** |

## Which Model Won?

**One-to-One dominates modern operating systems.** Linux, Windows, macOS, and FreeBSD all use 1:1 threading.

```
The Threading Model Timeline:

1990s:  Many-to-One dominates
        (single core, speed matters, parallelism irrelevant)
            │
2000s:  One-to-One takes over
        (multicore arrives, NPTL makes kernel threads fast)
            │
2010s:  Many-to-Many revived at language runtime level
        (Go goroutines, Erlang processes, Java virtual threads)
            │
        Key insight: M:N failed at the OS level because of
        complexity, but succeeds at the language runtime level
        where the runtime has full knowledge of its threads.
```

The modern twist: Go's goroutines and Java's virtual threads are essentially M:N models, but implemented in the language runtime rather than the OS. This works better because:

1. The runtime knows when a goroutine is about to block (it controls the I/O library)
2. The runtime can grow/shrink goroutine stacks dynamically
3. There's only one scheduler to reason about (the runtime's)

We'll cover this in detail in the goroutines and fibers section.

## Real-World Connection

### Why Linux Chose 1:1

When Linux needed a new threading library (replacing the buggy LinuxThreads), there were two competing proposals:

- **NGPT** (Next Generation POSIX Threads): M:N model, backed by IBM
- **NPTL** (Native POSIX Threads Library): 1:1 model, backed by Red Hat

NPTL won because:
1. Simpler implementation = fewer bugs = faster development
2. Kernel improvements (futexes, `clone()` optimizations) made 1:1 fast enough
3. M:N complexity wasn't worth the marginal performance gains
4. Debugging M:N threading was significantly harder

The lesson: **simpler models that are "fast enough" beat complex models that are theoretically optimal**. This is a recurring theme in systems engineering.

### Docker and Threading Models

Inside a Docker container, you're using the host kernel's threading model (1:1 on Linux). Container isolation doesn't change how threads work -- containers share the host kernel. This means thread limits, CPU scheduling, and kernel thread overhead are the same inside and outside containers. Docker's `--cpus` flag limits how many CPUs your container's kernel threads can use, but doesn't change the threading model.

## Interview Angle

**Q: What threading model does Linux use and why?**

A: Linux uses the One-to-One model (NPTL). Each user thread maps to one kernel thread via the `clone()` system call. This was chosen over M:N because it's simpler to implement, easier to debug, and fast enough on modern hardware. The kernel can schedule threads independently across cores, and blocking one thread doesn't affect others.

**Q: If One-to-One won, why do Go goroutines use M:N?**

A: Goroutines use M:N at the language runtime level, not the OS level. The Go runtime multiplexes potentially millions of goroutines onto a small number of OS threads. This works because the Go runtime controls everything -- it knows when a goroutine is about to block, it can dynamically resize stacks, and it manages its own scheduler. This avoids the coordination problems that plagued OS-level M:N implementations like Solaris LWP.

**Q: What's the maximum number of threads a Linux system can handle?**

A: The practical limit depends on available memory. Each kernel thread needs ~8KB of kernel stack plus whatever user stack size you configure (default often 8MB, but this is virtual memory and only physical pages actually used count). The system-wide limit is in `/proc/sys/kernel/threads-max` (often ~30,000-60,000 by default). Per-process limits are controlled by `ulimit -u`. For applications needing millions of concurrent tasks, this is why goroutines and virtual threads exist.

---

**Next:** [Thread Pools and Practical Patterns](04-thread-pools-and-practical-patterns.md)
