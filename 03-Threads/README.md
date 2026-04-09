# 03 - Threads

Threads are lightweight execution units within a process that share the same address space. While a process is a heavyweight container with its own memory, file descriptors, and security context, threads are the actual workers inside that container. Every modern program you interact with -- web servers, databases, IDEs, browsers -- uses threads to do multiple things concurrently.

Understanding threads is where OS theory starts connecting directly to the code you write every day.

## Why This Matters

- **Web servers** handle thousands of concurrent requests using thread pools
- **Databases** use threads to execute queries in parallel across CPU cores
- **Cloud computing** bills you for CPU time -- threading determines how efficiently you use those cores
- **Interview staple** -- threading, concurrency, and parallelism questions appear in almost every systems or backend interview
- **Debugging production issues** like deadlocks, race conditions, and thread starvation requires understanding how threads actually work under the hood

## Prerequisites

- [02-Processes](../02-Processes/README.md) -- especially process creation (`fork`/`exec`) and context switching
- Understanding of virtual memory basics (address spaces, stack vs heap)

## Reading Order

| # | Topic | What You'll Learn |
|---|-------|-------------------|
| 1 | [Threads vs Processes](01-threads-vs-processes.md) | What threads share, what they don't, and when to pick one over the other |
| 2 | [User Threads vs Kernel Threads](02-user-threads-vs-kernel-threads.md) | Where thread management happens and why it matters |
| 3 | [Threading Models](03-threading-models.md) | Many-to-One, One-to-One, Many-to-Many -- and which one won |
| 4 | [Thread Pools and Practical Patterns](04-thread-pools-and-practical-patterns.md) | How real systems use threads: pools, sizing, and common pitfalls |
| 5 | [Green Threads, Goroutines, and Fibers](05-green-threads-goroutines-fibers.md) | Modern lightweight concurrency: Go, Java Virtual Threads, coroutines |

## Key Interview Questions

1. **What is the difference between a process and a thread?** Threads share the same address space (code, data, heap, open files) within a process, while each process has its own isolated address space. Threads are cheaper to create and switch between, but lack the isolation that separate processes provide.

2. **Why do threads share the heap but not the stack?** The heap holds dynamically allocated data that multiple threads need to access (shared state). Each thread needs its own stack because it has its own execution flow -- its own function calls, local variables, and return addresses. Sharing a stack would corrupt execution state.

3. **What is the difference between user-level threads and kernel-level threads?** User-level threads are managed entirely in user space (fast to create/switch, but one blocking syscall blocks all threads). Kernel-level threads are managed by the OS (can run on multiple CPUs, one blocking doesn't block others, but creation/switching requires syscalls).

4. **What is a thread pool and why would you use one?** A thread pool pre-creates a fixed number of threads and assigns incoming tasks from a queue. This avoids the overhead of creating and destroying threads for each request. Web servers like Tomcat and Nginx use this pattern to handle concurrent requests efficiently.

5. **How are goroutines different from OS threads?** Goroutines are multiplexed onto a small number of OS threads by Go's runtime scheduler (M:N model). They start with ~2KB stacks (vs ~1MB for OS threads), making it practical to run millions concurrently. The Go scheduler handles preemption and load balancing across OS threads transparently.
