# 06 - Inter-Process Communication

Processes are isolated by design — each lives in its own address space, unable to read or write another process's memory. But useful systems require cooperation. A web server needs to talk to a database. A shell pipeline connects a chain of programs. A container orchestrator must coordinate dozens of services. **Inter-Process Communication (IPC)** is the set of mechanisms that make this cooperation possible.

IPC is one of those topics that bridges OS theory and real-world systems engineering. Every shell pipe you write, every Docker container that connects to a database socket, every microservice that sends a message — they all rely on IPC primitives built into the kernel.

## Why This Matters

- **Shell pipelines are IPC.** Every time you write `cat file | grep pattern | sort`, you're creating anonymous pipes between forked processes.
- **Microservices are IPC at scale.** Whether processes communicate over Unix sockets, shared memory, or network sockets, the underlying principles are the same.
- **Containers depend on IPC.** Docker uses Unix domain sockets for daemon communication. Kubernetes pods share IPC namespaces. Understanding these mechanisms helps you debug and architect containerized systems.
- **Performance-critical systems choose IPC carefully.** Databases use shared memory for buffer pools. Chrome uses shared memory between renderer and browser processes. The choice of IPC mechanism directly impacts throughput and latency.
- **Interviews test IPC knowledge across levels.** From "how do shell pipes work?" to "design a producer-consumer system" to "how would you share state between microservices?"

## Prerequisites

| Section | Why You Need It |
|---------|----------------|
| [02 - Processes](../02-Processes/README.md) | You need to understand process isolation, fork/exec, and file descriptors |
| [05 - Concurrency and Synchronization](../05-Concurrency-and-Synchronization/README.md) | Shared memory IPC requires synchronization — mutexes, semaphores, and an understanding of race conditions |

## Reading Order

| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 1 | IPC Overview & Taxonomy | [01-ipc-overview-and-taxonomy.md](01-ipc-overview-and-taxonomy.md) | The big picture: why IPC exists, shared memory vs message passing, how to choose |
| 2 | Pipes & FIFOs | [02-pipes-and-fifos.md](02-pipes-and-fifos.md) | The simplest IPC — how shell pipelines actually work under the hood |
| 3 | Message Queues | [03-message-queues.md](03-message-queues.md) | Kernel-managed queues with message boundaries and priorities |
| 4 | Shared Memory | [04-shared-memory.md](04-shared-memory.md) | The fastest IPC mechanism and the synchronization challenges it brings |
| 5 | Sockets for IPC | [05-sockets-for-ipc.md](05-sockets-for-ipc.md) | Unix domain sockets, the workhorse of modern local IPC |
| 6 | Signals | [06-signals.md](06-signals.md) | Asynchronous process notifications — simple but surprisingly tricky |

## Key Interview Questions

1. **What are the two fundamental approaches to IPC, and what are the tradeoffs?**
   Shared memory and message passing. Shared memory is faster (no kernel involvement after setup) but requires explicit synchronization. Message passing is simpler to reason about but involves kernel overhead for each transfer.

2. **How does `ls | grep foo` work at the system call level?**
   The shell creates a pipe (two file descriptors), forks twice. The first child redirects stdout to the pipe's write end and execs `ls`. The second child redirects stdin to the pipe's read end and execs `grep foo`. The shell closes its copies of the pipe ends and waits for both children.

3. **Why would you choose Unix domain sockets over pipes or shared memory?**
   Unix domain sockets are bidirectional, support both stream and datagram modes, work between unrelated processes, and use the familiar socket API. They're faster than TCP loopback and simpler than shared memory. If you might later need network communication, sockets let you swap AF_UNIX for AF_INET with minimal code changes.

4. **What signals cannot be caught or ignored, and why?**
   SIGKILL (9) and SIGSTOP cannot be caught, blocked, or ignored. The kernel enforces this so that there is always a way to terminate or stop a runaway process. Without this guarantee, a buggy or malicious process could make itself unkillable.

5. **When would you use shared memory over message passing?**
   When throughput matters and you're transferring large amounts of data between processes on the same machine. Shared memory avoids copying data through the kernel entirely. The tradeoff is that you must handle synchronization yourself (semaphores, mutexes, or atomic operations). Databases like PostgreSQL use shared memory for their buffer pools for exactly this reason.
