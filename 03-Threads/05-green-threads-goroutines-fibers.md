# Green Threads, Goroutines, and Fibers

The One-to-One threading model won at the OS level, but it has a hard ceiling: you can't create millions of OS threads. Modern applications -- chat servers with millions of connections, microservices handling massive fan-out -- need concurrency beyond what OS threads can offer. This section covers the solutions that emerged.

## Green Threads

Green threads are threads managed entirely by a language runtime or virtual machine, not by the OS kernel. The name comes from the original "Green Team" at Sun Microsystems that implemented them in early Java.

```
┌──────────────────────────────────────┐
│           Language Runtime            │
│                                      │
│   ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐   │
│   │G1│ │G2│ │G3│ │G4│ │G5│ │G6│   │   Green threads
│   └──┘ └──┘ └──┘ └──┘ └──┘ └──┘   │   (many, lightweight)
│     \    |    /     \    |    /     │
│      \   |   /       \   |   /     │
│     ┌─────────┐     ┌─────────┐    │
│     │ OS Thr 1│     │ OS Thr 2│    │   OS threads
│     └─────────┘     └─────────┘    │   (few, heavyweight)
└──────────────────────────────────────┘
              │              │
           ┌──┴──┐        ┌──┴──┐
           │CPU 0│        │CPU 1│
           └─────┘        └─────┘
```

**Key characteristics:**
- Created and scheduled by the runtime, not the OS
- Much smaller stacks (KBs instead of MBs)
- The runtime decides when to switch between green threads
- The runtime wraps I/O calls to make them non-blocking under the hood

Early green threads (Java pre-1.2) were Many-to-One and couldn't use multiple CPUs. Modern implementations (goroutines, virtual threads) are Many-to-Many and do use multiple CPUs. The concept matured significantly.

## Goroutines (Go)

Goroutines are Go's implementation of lightweight, runtime-managed threads. They're the most successful modern example of the M:N model -- and arguably the reason Go became popular for server-side programming.

### The Go Scheduler: G, M, P

Go's scheduler has three key components:

```
G = Goroutine (the unit of work)
M = Machine  (an OS thread)
P = Processor (a logical CPU / scheduling context)

┌──────────────────────────────────────────────────┐
│                  Go Runtime                       │
│                                                   │
│  ┌─────┐  ┌─────┐                               │
│  │  P0  │  │  P1  │     P = GOMAXPROCS (default: │
│  │      │  │      │         number of CPU cores)  │
│  │ Local│  │ Local│                               │
│  │Queue:│  │Queue:│     Each P has a local run    │
│  │G1,G2 │  │G4,G5 │     queue of goroutines       │
│  │G3    │  │G6    │                               │
│  └──┬───┘  └──┬───┘                               │
│     │         │                                   │
│     │         │         Global Run Queue           │
│     │         │         ┌─────────────────┐        │
│     │         │         │ G7, G8, G9 ...  │        │
│     │         │         └─────────────────┘        │
│     │         │                                   │
│  ┌──┴──┐  ┌──┴──┐                               │
│  │  M0  │  │  M1  │     M = OS threads             │
│  │(thr) │  │(thr) │     (matched to Ps)            │
│  └──┬───┘  └──┬───┘                               │
└─────┼─────────┼──────────────────────────────────┘
      │         │
   ┌──┴──┐  ┌──┴──┐
   │CPU 0│  │CPU 1│
   └─────┘  └─────┘
```

### How It Works

1. **`go func()`** creates a goroutine (G) and places it on a P's local run queue
2. Each P is bound to one M (OS thread) at a time
3. The P pulls goroutines from its local queue and executes them on the M
4. If the local queue is empty, the P steals goroutines from another P's queue (work stealing)
5. When a goroutine makes a blocking syscall, the M detaches from the P, and the P gets a new M to keep running other goroutines

### Why Goroutines Are Cheap

| Property | OS Thread | Goroutine |
|----------|-----------|-----------|
| Initial stack size | ~1-8 MB (fixed) | ~2 KB (grows dynamically) |
| Creation time | ~50 microseconds | ~0.3 microseconds |
| Context switch | ~1-5 microseconds | ~0.2 microseconds |
| Memory per 10K | ~10-80 GB | ~20 MB |
| Practical limit | ~10,000 | ~1,000,000+ |

The dynamic stack is key. A goroutine starts with a 2KB stack that grows and shrinks as needed. An OS thread gets a fixed 1-8MB stack allocated upfront (mostly virtual memory, but still consumes address space and page table entries).

### Scheduling: Cooperative with Preemption

Goroutines are cooperatively scheduled at certain points:
- Channel operations (send/receive)
- I/O operations
- `runtime.Gosched()` calls
- Function calls (the compiler inserts preemption checks)

Since Go 1.14, there's also **asynchronous preemption**: the runtime can interrupt a goroutine that's been running too long (using OS signals). This prevents a CPU-bound goroutine from starving others.

### Handling Blocking

```
Goroutine G1 makes a blocking syscall (e.g., file read):

Before:                         After:
┌────┐                          ┌────┐
│ P0 │──── M0                   │ P0 │──── M2 (new OS thread)
│    │     executing G1         │    │     executing G2
│G2  │                          │G3  │
│G3  │                          └────┘
└────┘                          
                                M0 is blocked in kernel with G1
                                When G1's syscall returns, M0
                                returns G1 to a P's queue
```

The P detaches from the blocked M and grabs a free M (or creates a new one). This way, blocking syscalls don't block the scheduler.

## Java Virtual Threads (Project Loom)

Java 21 introduced virtual threads -- Java's answer to goroutines. Same core idea: lightweight, runtime-managed threads multiplexed onto a small number of OS threads (called "carrier threads").

```
┌─────────────────────────────────────┐
│          JVM / Project Loom          │
│                                      │
│  Virtual Threads (millions):         │
│  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ...     │
│  │V1│ │V2│ │V3│ │V4│ │V5│         │
│  └──┘ └──┘ └──┘ └──┘ └──┘         │
│    \    |   /      \   /            │
│     Mounted/unmounted on:           │
│  ┌──────────┐  ┌──────────┐        │
│  │ Carrier 1│  │ Carrier 2│        │
│  │ (OS thr) │  │ (OS thr) │        │
│  └──────────┘  └──────────┘        │
└─────────────────────────────────────┘
```

**Key features:**
- Create with `Thread.ofVirtual().start(runnable)` or use `ExecutorService` with virtual threads
- Blocking I/O automatically unmounts the virtual thread from its carrier (the carrier can run other virtual threads)
- Existing synchronous Java code works as-is -- no need to rewrite to async
- Stack traces work normally (unlike reactive/callback-based code)

```java
// Create 1 million virtual threads -- no problem
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        executor.submit(() -> {
            // This blocking call doesn't waste an OS thread
            String result = httpClient.send(request);
            return process(result);
        });
    }
}
```

The major advantage over reactive frameworks (Project Reactor, RxJava): you write normal sequential code instead of complex callback chains, and the runtime makes it efficient.

## Fibers

Fibers are cooperatively scheduled threads -- they explicitly yield control rather than being preempted by a scheduler.

```
Fiber 1 runs → yields → Fiber 2 runs → yields → Fiber 1 resumes

  Fiber 1         Fiber 2         Fiber 3
  ┌─────┐         ┌─────┐         ┌─────┐
  │ run │         │     │         │     │
  │     │         │     │         │     │
  │yield├────────>│ run │         │     │
  │     │         │     │         │     │
  │     │         │yield├────────>│ run │
  │     │         │     │         │     │
  │     │         │     │         │yield├──> ...
  └─────┘         └─────┘         └─────┘
```

**Key distinction from threads:** Fibers choose when to give up the CPU. Threads are preempted by the scheduler whether they like it or not.

**Advantage:** No race conditions from unexpected preemption -- the fiber controls exactly when context switches happen.

**Disadvantage:** A fiber that forgets to yield (or runs a long computation) blocks all other fibers on that thread.

**Examples:**
- Windows Fibers (`ConvertThreadToFiber`, `SwitchToFiber`)
- Ruby Fibers
- Boost.Fiber (C++)

## Coroutines

Coroutines are functions that can suspend their execution and resume later. They're the language-level primitive that powers async/await.

```
Python asyncio:                    Kotlin coroutines:

async def fetch_data():            suspend fun fetchData(): Data {
    response = await http.get()        val response = httpClient.get()
    data = await response.json()       val data = response.parse()
    return data                        return data
                                   }
```

**Important distinction:** Coroutines are typically single-threaded. They achieve concurrency (interleaving) but not parallelism (simultaneous execution). When one coroutine awaits I/O, another coroutine runs on the same thread.

```
Single thread running coroutines:

Time ──────────────────────────────────────>

Coroutine A: [compute]  [wait I/O]     [compute]
Coroutine B:            [compute] [wait]         [compute]
Coroutine C:                            [compute]

All on ONE thread. No parallelism, but concurrent I/O.
```

**Examples:**
- Python `asyncio` (async/await)
- Kotlin coroutines
- JavaScript Promises / async-await
- C# async/await
- C++20 coroutines

## The Full Comparison

| Dimension | OS Threads | Green Threads | Goroutines | Virtual Threads | Coroutines |
|-----------|-----------|---------------|------------|-----------------|------------|
| Managed by | OS kernel | Language runtime | Go runtime | JVM | Language runtime |
| Scheduling | Preemptive | Depends on impl | Cooperative + preemption | Cooperative (with carrier) | Cooperative |
| Stack size | 1-8 MB | KBs | ~2 KB (dynamic) | Dynamic | No stack (state machine) |
| Creation cost | ~50 us | ~1 us | ~0.3 us | ~1 us | Negligible |
| Parallelism | Yes | Depends | Yes (M:N) | Yes (M:N) | No (single-threaded) |
| Max practical count | ~10K | ~100K+ | ~1M+ | ~1M+ | ~1M+ |
| Blocking I/O | Blocks that thread | May block all | Runtime handles it | Runtime handles it | Must use async I/O |
| Code style | Synchronous | Synchronous | Synchronous | Synchronous | async/await |
| Debugging | Straightforward | Moderate | Good (goroutine dumps) | Good (stack traces) | Hard (async stack traces) |

## When to Use What

```
Decision tree:

Is your work CPU-bound or I/O-bound?
│
├── CPU-bound (computation, processing):
│   └── Use OS threads (one per core)
│       Limited parallelism = limited threads needed
│       Goroutines and virtual threads also work fine
│
└── I/O-bound (network, disk, database):
    │
    ├── How many concurrent tasks?
    │   │
    │   ├── Hundreds: OS thread pool works fine
    │   │
    │   ├── Thousands: Thread pool possible but watch memory
    │   │   Consider goroutines or virtual threads
    │   │
    │   └── Millions: Must use lightweight concurrency
    │       ├── Go? → Goroutines (built-in)
    │       ├── Java 21+? → Virtual threads
    │       ├── Python? → asyncio
    │       ├── JavaScript? → async/await (only option)
    │       ├── Kotlin? → coroutines
    │       └── C/C++? → Event loop (epoll) + thread pool
    │
    └── Must existing synchronous code work unchanged?
        ├── Yes → Goroutines or virtual threads
        │         (no code rewrite needed)
        └── No → Coroutines / async-await are fine
                  (requires async rewrite)
```

## The Trend: Lightweight Concurrency Is Winning

```
Evolution of concurrency models:

1990s: Processes (fork per request)
  └── Too heavy, too slow to create
  
2000s: OS Thread Pools (Apache, Tomcat)
  └── Works but limited to ~10K concurrent connections
  
2010s: Event Loops (Node.js, Nginx)
  └── Scales to millions, but callback hell / async complexity
  
2020s: Lightweight threads (Goroutines, Virtual Threads)
  └── Scales to millions AND keeps synchronous code style
      Best of both worlds
```

The trend is clear: modern languages are building lightweight, runtime-managed concurrency into their core. Go had goroutines from day one. Java added virtual threads. Kotlin has coroutines. Python has asyncio. Even C++ added coroutines in C++20.

The OS thread isn't going away -- these lightweight models run on top of OS threads. But application developers increasingly interact with the lightweight abstraction rather than OS threads directly.

## Real-World Connection

### Go in Production: Why Goroutines Matter

A typical Go web server (using `net/http`) creates one goroutine per incoming connection. With goroutines costing ~2KB each, a server with 100K concurrent connections uses ~200MB of goroutine stacks. The same with OS threads would require ~100-800GB.

This is why Go dominates infrastructure tooling: Docker, Kubernetes, Terraform, Prometheus, and most cloud-native tools are written in Go. The ability to handle massive concurrency with simple synchronous code is a huge productivity advantage.

### Java's Shift: From Reactive to Virtual Threads

Before Project Loom, Java developers who needed high concurrency had two options:
1. **Reactive frameworks** (Project Reactor, RxJava): High performance but complex, unreadable code and broken stack traces
2. **Large thread pools**: Simple code but limited scalability

Virtual threads let Java developers write the same simple blocking code they always wrote, but now it scales to millions of concurrent tasks. This is deprecating much of the reactive ecosystem -- there's less reason to suffer callback complexity when virtual threads provide the same performance with straightforward code.

### Python asyncio: The Tradeoff

Python's `asyncio` gives you coroutine-based concurrency, but with a major caveat: the entire call chain must be async. One blocking call in an async function blocks the entire event loop. This "function coloring" problem (async functions can only be called from async functions) fragments the Python ecosystem into sync and async halves.

Go and Java virtual threads avoid this problem because their runtimes intercept blocking calls transparently.

## Interview Angle

**Q: How are goroutines different from OS threads?**

A: Goroutines are multiplexed onto a small number of OS threads by Go's runtime scheduler (M:N model). Key differences: goroutines start with ~2KB stacks that grow dynamically (vs 1-8MB fixed for OS threads), creation takes ~0.3 microseconds (vs ~50 for OS threads), and you can practically run millions of them. The Go runtime handles blocking syscalls by detaching the OS thread from the scheduler and assigning a new one, so blocking doesn't stall other goroutines.

**Q: What is the difference between concurrency and parallelism in the context of coroutines vs threads?**

A: Concurrency is about dealing with multiple things at once (structure). Parallelism is about doing multiple things at once (execution). Coroutines achieve concurrency on a single thread -- when one coroutine awaits I/O, another runs, but only one executes at any instant. OS threads (and goroutines) achieve both concurrency and parallelism -- multiple threads execute simultaneously on different CPU cores. For I/O-bound work, concurrency alone is often sufficient. For CPU-bound work, you need parallelism.

**Q: Why can't you just use async/await everywhere instead of threads?**

A: Three reasons. First, async/await requires rewriting your entire call chain to be async ("function coloring problem") -- you can't call a blocking function from an async context without blocking the event loop. Second, coroutines are typically single-threaded, so CPU-bound work blocks everything. Third, not all libraries support async I/O. Goroutines and virtual threads solve these problems by making blocking calls non-blocking transparently at the runtime level, without requiring code changes.
