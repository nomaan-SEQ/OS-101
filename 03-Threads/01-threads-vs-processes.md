# Threads vs Processes

A thread is an independent flow of execution within a process. If a process is a factory building, threads are the workers inside it. They all share the same floor space, tools, and materials (address space, code, open files), but each worker has their own task list and personal notepad (stack, registers, program counter).

The key insight: **processes are about resource ownership, threads are about execution**.

## What Threads Share vs What They Don't

This distinction is fundamental. Get it wrong and you'll write bugs that are nearly impossible to reproduce.

```
┌─────────────────────────────────────────────────────┐
│                    PROCESS                           │
│                                                      │
│  ┌─────────────────────────────────────────────┐     │
│  │           SHARED RESOURCES                   │     │
│  │                                              │     │
│  │  Code (text segment)                         │     │
│  │  Data segment (global variables)             │     │
│  │  Heap (dynamically allocated memory)         │     │
│  │  Open file descriptors                       │     │
│  │  Signal handlers                             │     │
│  │  Working directory                           │     │
│  │  User/Group ID                               │     │
│  └─────────────────────────────────────────────┘     │
│                                                      │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐          │
│  │ Thread 1│    │ Thread 2│    │ Thread 3│          │
│  │         │    │         │    │         │          │
│  │ Stack   │    │ Stack   │    │ Stack   │          │
│  │ Regs    │    │ Regs    │    │ Regs    │          │
│  │ PC      │    │ PC      │    │ PC      │          │
│  │ TID     │    │ TID     │    │ TID     │          │
│  │ errno   │    │ errno   │    │ errno   │          │
│  │ Signal  │    │ Signal  │    │ Signal  │          │
│  │ mask    │    │ mask    │    │ mask    │          │
│  └─────────┘    └─────────┘    └─────────┘          │
│   PRIVATE         PRIVATE         PRIVATE            │
└─────────────────────────────────────────────────────┘
```

### Shared (all threads in a process see the same):

| Resource | Why Shared |
|----------|-----------|
| Code segment | All threads execute functions from the same program |
| Data/BSS segment | Global and static variables are shared state |
| Heap | `malloc`'d memory is accessible by any thread |
| Open file descriptors | Threads can read/write the same files and sockets |
| Signal handlers | Signal disposition is per-process |
| Address space | Same page table, same virtual-to-physical mappings |

### Private (each thread has its own):

| Resource | Why Private |
|----------|------------|
| Stack | Each thread has its own function call chain and local variables |
| Registers | Each thread is at a different point in execution |
| Program counter | Each thread executes different instructions at any given moment |
| Thread ID (TID) | Unique identifier per thread |
| `errno` | System call errors must not leak between threads |
| Signal mask | Each thread can block different signals |

## Why Use Threads Instead of Processes?

### 1. Lower Creation Overhead

Creating a thread is much cheaper than creating a process. A `fork()` must duplicate the entire address space (even with copy-on-write, the page table entries must be copied). A thread just needs a new stack and register set.

```
Typical creation cost (Linux, approximate):
  fork()          ~100-200 microseconds
  pthread_create  ~10-50 microseconds
  
  That's 5-10x faster for threads.
```

### 2. Faster Context Switching

Switching between threads in the same process doesn't require changing the page table or flushing the TLB (Translation Lookaside Buffer). The address space stays the same.

```
Context switch cost (approximate):
  Process switch:  ~3-5 microseconds (TLB flush, page table swap)
  Thread switch:   ~1-2 microseconds (just swap stack/registers)
```

### 3. Shared Memory Communication

Threads communicate by reading and writing shared variables -- no need for pipes, sockets, or shared memory segments. This is both the biggest advantage and the biggest source of bugs.

```
Process Communication:        Thread Communication:
┌─────────┐  pipe/  ┌──────┐  ┌─────────────────────┐
│Process A│──socket──│Proc B│  │  Thread A   Thread B │
│  data   │  shmem  │ data │  │    │  shared   │     │
└─────────┘         └──────┘  │    └── data ───┘     │
                               └─────────────────────┘
Requires IPC                   Direct memory access
mechanisms                     (but needs synchronization!)
```

### 4. Resource Efficiency

1000 threads in one process share one address space. 1000 processes each have their own address space. The memory savings are significant.

## When to Use Processes Instead

Threads aren't always the answer. Choose processes when you need:

| Need | Why Processes Win |
|------|------------------|
| **Crash isolation** | A segfault in one thread kills the entire process. A crashed process doesn't affect others. |
| **Security isolation** | Processes can run as different users with different privileges. Threads share the same UID. |
| **Memory isolation** | A memory corruption bug in one thread corrupts shared heap for all threads. Processes have separate address spaces. |
| **Different programs** | You can't `exec()` a different program in just one thread -- it replaces the whole process. |
| **Language constraints** | Python's GIL means threads can't achieve CPU parallelism. Use multiprocessing instead. |

## Process vs Thread: Full Comparison

| Dimension | Process | Thread |
|-----------|---------|--------|
| Address space | Own (isolated) | Shared with other threads |
| Creation cost | High (~100-200 us) | Low (~10-50 us) |
| Context switch | Slow (TLB flush needed) | Fast (same address space) |
| Communication | IPC required (pipes, sockets, shmem) | Direct shared memory |
| Memory overhead | High (separate page tables, data) | Low (shared everything except stack) |
| Crash impact | Only that process dies | Entire process (all threads) dies |
| Security | Can have different privileges | All share same privileges |
| CPU parallelism | Yes | Yes (with kernel threads) |
| Debugging | Easier (isolated state) | Harder (shared mutable state, race conditions) |
| Scalability | Limited by memory per process | Limited by stack space and kernel limits |

## Real-World Connection

### Chrome's Multi-Process Architecture

Chrome deliberately chose processes over threads for its tab architecture. Each tab (roughly) runs in its own process. Why?

```
Chrome Architecture:
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Tab 1      │  │   Tab 2      │  │   Tab 3      │
│  (Process)   │  │  (Process)   │  │  (Process)   │
│              │  │              │  │              │
│  Renderer    │  │  Renderer    │  │  Renderer    │
│  JS Engine   │  │  JS Engine   │  │  JS Engine   │
│  DOM         │  │  DOM         │  │  DOM         │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └────────┬────────┴────────┬────────┘
                │                 │
         ┌──────┴──────┐  ┌──────┴──────┐
         │   Browser   │  │    GPU      │
         │   Process   │  │   Process   │
         │  (Main UI)  │  │ (Rendering) │
         └─────────────┘  └─────────────┘
```

- **Crash isolation**: A misbehaving website crashes only its tab, not the whole browser
- **Security isolation**: Each renderer process runs in a sandbox with restricted permissions
- **Memory overhead**: Yes, this uses more memory than threads -- Chrome's well-known memory usage is the tradeoff for stability and security

This is a textbook example of choosing isolation over efficiency. For a browser running untrusted code from the internet, isolation wins.

### Linux Implementation Detail

On Linux, threads and processes are more similar than you might think. Both are created with the `clone()` system call. The difference is which resources are shared:

- `fork()` = `clone()` with separate everything
- `pthread_create()` = `clone()` with shared address space, file descriptors, signal handlers

Linux calls both "tasks" internally. The kernel doesn't have a fundamentally different concept of "thread" -- it's just a task that shares more with its parent.

## Interview Angle

**Q: If threads share the heap, how do you prevent data corruption?**

A: You need synchronization primitives -- mutexes, semaphores, condition variables. Without them, two threads writing to the same data structure simultaneously can corrupt it. This is a "race condition." The next chapter on synchronization covers this in depth. The fundamental tradeoff: shared memory makes communication fast but makes correctness hard.

**Q: Can you have a process with zero threads?**

A: No. Every process has at least one thread of execution (the main thread). When that thread exits, the process terminates. When people say "a single-threaded process," they mean a process with exactly one thread.

**Q: Why does Linux implement threads as processes internally?**

A: Linux uses `clone()` for both, treating everything as a "task" with configurable sharing. This is an engineering decision that simplifies the kernel -- instead of maintaining two separate concepts with different scheduling code, it has one unified task model. The flags passed to `clone()` determine what's shared. This means Linux threads are sometimes called "lightweight processes."

---

**Next:** [User Threads vs Kernel Threads](02-user-threads-vs-kernel-threads.md)
