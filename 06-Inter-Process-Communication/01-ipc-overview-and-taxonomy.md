# IPC Overview and Taxonomy

## Why IPC Exists

Operating systems enforce **process isolation** as a fundamental security and stability boundary. Each process gets its own virtual address space — process A cannot read or write process B's memory. If one process crashes, it doesn't take down others.

But isolation alone isn't useful. Real systems need cooperation:

- A shell pipeline connects multiple programs to transform data
- A web server communicates with a database to handle requests
- A logging daemon collects messages from dozens of services
- A container orchestrator coordinates hundreds of processes

IPC mechanisms are the controlled channels the kernel provides so processes can exchange data and coordinate without breaking isolation.

## Two Fundamental Approaches

Every IPC mechanism falls into one of two categories:

```
                        IPC Approaches
                             |
              +--------------+--------------+
              |                             |
       Shared Memory                 Message Passing
              |                             |
   Processes map the same          Processes send/receive
   physical memory region          messages through the kernel
              |                             |
   Fast: no copying after         Slower: data copied to/from
   setup, but requires            kernel, but simpler to
   explicit synchronization       reason about
```

### Shared Memory

Two (or more) processes map the same region of physical memory into their virtual address spaces. Once set up, they read and write directly — **no kernel involvement per operation**.

The catch: since both processes access the same memory, you need synchronization (mutexes, semaphores) to avoid race conditions. The kernel sets up the mapping but doesn't referee access.

### Message Passing

Processes send discrete messages through a kernel-managed channel. The kernel copies data from the sender's address space to the receiver's. This is inherently safer — the kernel mediates every transfer — but the copying adds overhead.

### Head-to-Head Comparison

| Aspect | Shared Memory | Message Passing |
|--------|--------------|-----------------|
| **Speed** | Fastest — no kernel copy after setup | Slower — kernel copies data per message |
| **Setup complexity** | More complex (create region, attach, sync) | Simpler (create channel, send/receive) |
| **Synchronization** | YOU must handle it (semaphores, mutexes) | Kernel handles ordering for you |
| **Data size** | Efficient for large data transfers | Better for small, structured messages |
| **Safety** | Risky — race conditions if sync is wrong | Safer — kernel mediates all access |
| **Debugging** | Harder — concurrent access issues | Easier — message flow is traceable |

## Taxonomy of IPC Mechanisms

```
                            IPC Mechanisms
                                 |
          +----------------------+----------------------+
          |                      |                      |
    Message Passing        Shared Memory            Signals
          |                      |               (notification
          |                      |                only, no data)
    +-----+------+        +-----+-----+
    |     |      |        |           |
  Pipes  Msg   Sockets  POSIX       mmap
  FIFOs  Queues          shm     (MAP_SHARED)
                  |
           +------+------+
           |             |
      Unix Domain    Network
      Sockets       Sockets
      (local)     (local or remote)
```

## Local IPC vs Network IPC

Most IPC mechanisms work only on a single machine. Sockets are the exception — they can work locally or across a network.

| Scope | Mechanisms | When to Use |
|-------|-----------|-------------|
| **Local only** | Pipes, FIFOs, message queues, shared memory, signals, Unix domain sockets | Processes on the same host, maximum performance |
| **Local or network** | TCP/UDP sockets | When you need to communicate across machines, or want the flexibility to scale out later |

This distinction matters for system design. If your processes will always be on the same host, use local IPC for performance. If they might someday run on different hosts (e.g., breaking a monolith into microservices), consider sockets from the start.

## Comparison of All IPC Mechanisms

| Mechanism | Speed | Complexity | Direction | Unrelated Processes? | Message Boundaries? | Best For |
|-----------|-------|-----------|-----------|---------------------|--------------------:|----------|
| **Anonymous Pipe** | Fast | Low | Unidirectional | No (parent-child only) | No (byte stream) | Shell pipelines, simple parent-child |
| **Named Pipe (FIFO)** | Fast | Low | Unidirectional | Yes | No (byte stream) | Simple communication between unrelated processes |
| **Message Queue** | Medium | Medium | Bidirectional possible | Yes | Yes | Typed, prioritized messages |
| **Shared Memory** | Fastest | High | Bidirectional | Yes | No (raw memory) | High-throughput data sharing |
| **Unix Domain Socket** | Fast | Medium | Bidirectional | Yes | Yes (datagrams) or No (stream) | General-purpose local IPC, modern default |
| **Network Socket** | Slowest (local) | Medium | Bidirectional | Yes (even cross-machine) | Depends on protocol | Cross-machine communication |
| **Signal** | N/A (notification) | Low | Unidirectional | Yes | N/A | Process notification, not data transfer |

## How to Choose the Right IPC Mechanism

Use this decision flow:

```
Need to communicate across machines?
  YES --> Network sockets (TCP/UDP)
  NO  --> How much data are you transferring?
            |
      Large, continuous data
      (e.g., shared buffers)
            --> Shared memory + synchronization
            |
      Structured messages?
            --> Need bidirectional?
                  YES --> Unix domain sockets
                  NO  --> Need message types/priorities?
                            YES --> Message queues
                            NO  --> Pipe or FIFO
            |
      Just need to notify a process?
            --> Signal
```

**Rules of thumb:**

- **Default to Unix domain sockets** for new designs — they're flexible, bidirectional, and well-understood.
- **Use pipes** for simple parent-child data flow or shell-style composition.
- **Use shared memory** only when you've profiled and message copying is actually a bottleneck.
- **Use signals** only for notifications, never for data transfer.
- **Use network sockets** when you need cross-machine communication or want to make that transition easy.

## Real-World Connection

- **Docker** uses Unix domain sockets (`/var/run/docker.sock`) for CLI-to-daemon communication.
- **PostgreSQL** uses shared memory for its shared buffer pool and Unix domain sockets for client connections.
- **Shell pipelines** (`ps aux | grep nginx | awk '{print $2}'`) create a chain of anonymous pipes.
- **Kubernetes** pods in the same pod share an IPC namespace, allowing System V shared memory between containers.
- **Nginx** uses signals for graceful reload (SIGHUP) and shared memory for caching and rate limiting across worker processes.

## Interview Angle

**Q: If you were designing a system where two processes need to exchange high-throughput data on the same machine, which IPC mechanism would you choose and why?**

A: Shared memory, because it eliminates kernel-mediated data copying after initial setup. Both processes access the same physical memory region directly, making it the fastest IPC option. The tradeoff is that I need to implement synchronization (typically with POSIX semaphores or mutexes in the shared region) to coordinate reads and writes. This is exactly what databases like PostgreSQL do for their buffer pool — shared memory with careful locking.

**Q: What's the difference between local IPC and network IPC? When would you choose one over the other?**

A: Local IPC mechanisms (pipes, shared memory, Unix domain sockets) only work between processes on the same machine but are faster because they avoid network stack overhead. Network sockets (TCP/UDP) work across machines but involve serialization, network stack processing, and potentially network latency. Choose local IPC when processes are co-located and performance matters. Choose network sockets when processes might run on different machines, or when you want the architectural flexibility to separate them later.

---

**Next:** [Pipes and FIFOs](02-pipes-and-fifos.md) — the simplest form of IPC, and the mechanism behind every shell pipeline.
