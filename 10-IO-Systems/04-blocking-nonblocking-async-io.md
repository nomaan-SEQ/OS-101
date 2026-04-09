# Blocking, Non-blocking, and Async I/O

So far we've looked at I/O from the hardware side -- how devices talk to the CPU. Now let's flip to the **application's perspective**: when your code calls `read()` or `write()`, what happens to your thread? The answer depends on the I/O model, and choosing the right one is the difference between a server that handles 100 connections and one that handles 100,000.

## Model 1: Blocking I/O (Synchronous)

The default behavior. When you call `read()`, your thread **sleeps until data arrives**.

```
Thread              Kernel              Device
  |                   |                   |
  |-- read(fd) ------>|                   |
  |                   |-- DMA setup ----->|
  |   [BLOCKED]       |                   |
  |   [sleeping]      |   (waiting for    |
  |   [can't run]     |    device...)     |
  |                   |                   |
  |                   |<-- DMA complete --|
  |                   |-- copy to user -->|
  |<-- data ---------|                   |
  |                   |                   |
  |  (continue)       |                   |
  v                   v                   v
```

```c
// Blocking read -- thread sleeps until data arrives
char buf[4096];
ssize_t n = read(fd, buf, sizeof(buf));  // blocks here
// n bytes of data are now in buf
process(buf, n);
```

### Thread-per-Connection Model

The natural pattern with blocking I/O: spawn one thread for each client connection.

```
Main Thread:
  while (true) {
      client = accept(server_socket);  // blocks until connection
      spawn_thread(handle_client, client);
  }

Worker Thread:
  while (true) {
      data = read(client_fd);  // blocks until data
      response = process(data);
      write(client_fd, response);  // blocks until sent
  }
```

**Pros:** Simple mental model. Easy to write, easy to debug. Each thread is a straightforward sequential program.

**Cons:** Each thread costs ~1-8 MB of stack memory. 10,000 connections = 10,000 threads = 10-80 GB of memory just for stacks. Context switching between thousands of threads destroys cache performance.

## Model 2: Non-blocking I/O

Set a file descriptor to non-blocking mode. `read()` returns **immediately** -- either with data or with an error saying "nothing available yet."

```
Thread              Kernel              Device
  |                   |                   |
  |-- read(fd) ------>|                   |
  |<-- EAGAIN --------|  (no data yet)    |
  |                   |                   |
  |  (do other work)  |                   |
  |                   |                   |
  |-- read(fd) ------>|                   |
  |<-- EAGAIN --------|  (still nothing)  |
  |                   |                   |
  |  (do other work)  |                   |
  |                   |<-- data ready ----|
  |-- read(fd) ------>|                   |
  |<-- data ---------|                   |
  v                   v                   v
```

```c
// Set non-blocking mode
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// Non-blocking read
ssize_t n = read(fd, buf, sizeof(buf));
if (n == -1 && errno == EAGAIN) {
    // No data available -- do something else, try again later
} else if (n > 0) {
    // Got data!
    process(buf, n);
}
```

**By itself, non-blocking I/O is useless** -- you'd just be polling in a loop, wasting CPU like hardware polling. The power comes when you combine it with **I/O multiplexing** (select/poll/epoll), which we cover in the next section.

## Model 3: Asynchronous I/O (AIO)

Submit an I/O request and **get notified when it's complete**. The thread never blocks and never polls -- it just receives a callback or event.

```
Thread              Kernel              Device
  |                   |                   |
  |-- aio_read(fd) -->|                   |
  |<-- "submitted" ---|                   |
  |                   |-- DMA setup ----->|
  |  (do other work!) |                   |
  |  (truly free)     |   (transfer...)   |
  |  (no polling)     |                   |
  |                   |<-- DMA complete --|
  |                   |-- copy to user    |
  |<== notification ==|  (signal/event)   |
  |                   |                   |
  |  (process data)   |                   |
  v                   v                   v
```

### Linux AIO Evolution

**POSIX AIO (aio_read/aio_write):** Old, clunky. Creates kernel threads internally -- not truly async for many operations.

**libaio (io_submit/io_getevents):** Better, but only works with O_DIRECT files (bypassing page cache). Doesn't work with network I/O.

**io_uring (Linux 5.1+, 2019):** The modern answer. Shared ring buffers between user and kernel space. Supports everything -- files, network, timers. Zero-copy possible. Batched submission (one syscall for many operations).

```
io_uring Architecture:

User Space              Kernel Space
+------------------+    +------------------+
| Submission Queue |    | Submission Queue |
| (SQ) - ring buf  |--->| processing       |
| [req1][req2][...]|    | [executes I/O]   |
+------------------+    +------------------+
                              |
+------------------+    +------------------+
| Completion Queue |    | Completion Queue |
| (CQ) - ring buf  |<---| results          |
| [res1][res2][...]|    | [done events]    |
+------------------+    +------------------+

- Shared memory: no data copying between user/kernel
- Batched: submit many requests with one syscall
- Poll mode: zero syscalls possible
```

### Windows IOCP (I/O Completion Ports)

Windows took a different approach with IOCP -- widely considered the best-designed async I/O model:

- Create a completion port (an OS-managed event queue)
- Associate file handles with the port
- Submit async I/O operations
- Worker threads call `GetQueuedCompletionStatus()` to receive completions
- The OS manages the thread pool for you -- if a thread blocks, it wakes another

IOCP is a **completion-based** model (you're told when I/O is done), unlike epoll which is **readiness-based** (you're told when I/O is possible).

## Comparison Table

| Property | Blocking | Non-blocking | Async (io_uring/IOCP) |
|----------|----------|-------------|----------------------|
| Thread during I/O | Sleeping | Running (must retry) | Running (truly free) |
| Data notification | Thread wakes up | EAGAIN then data | Event/callback |
| Threads needed | 1 per connection | 1 for many connections | 1 for many connections |
| Programming complexity | Simple | Moderate | High |
| CPU efficiency | Low (idle threads) | Depends on design | High |
| Memory per connection | ~1-8 MB (thread stack) | ~KB (event state) | ~KB (request state) |
| Max connections (practical) | ~1K-10K | ~100K+ | ~100K+ |
| Debugging | Easy (stack traces) | Hard (state machines) | Hard (callbacks/events) |

## Timeline Comparison

```
Blocking (3 connections, 3 threads):
Thread 1: |===read===|--process--|===read===|--process--|
Thread 2: |===read===|--process--|===read===|--process--|
Thread 3: |===read===|--process--|===read===|--process--|
           ^^^ idle CPU time (threads sleeping)

Non-blocking + event loop (3 connections, 1 thread):
Thread 1: |poll|read1|process1|poll|read2|process2|poll|read3|process3|
           ^^^ one thread handles all three, no idle time

Async (3 connections, 1 thread):
Thread 1: |submit1,2,3|--other work--|complete1|process1|complete2|process2|
           ^^^ submit all at once, process as they complete
```

## The C10K Problem

In 1999, Dan Kegel posed the question: **how do you handle 10,000 concurrent connections on a single server?**

With blocking I/O and thread-per-connection:
- 10,000 threads x 1 MB stack = 10 GB just for thread stacks
- Context switching 10,000 threads destroys CPU cache
- Most threads are sleeping (waiting on I/O) -- massive waste

The solution: **don't use one thread per connection.** Use non-blocking I/O with event-driven architecture:
- One thread monitors thousands of FDs (via epoll/kqueue)
- When data arrives on any FD, process it immediately
- No sleeping threads, no wasted stacks, no context-switch overhead

Today the problem has evolved to **C10M** (10 million connections) -- solved with kernel bypass (DPDK), io_uring, and hardware offload.

## Real-World Connection

**Node.js**: Uses non-blocking I/O with an event loop (libuv library). Your JavaScript code runs in a single thread. When you call `fs.readFile()`, Node submits the I/O, continues running other callbacks, and calls your callback when data arrives. This is why Node handles thousands of connections with one thread -- but CPU-heavy work blocks the whole event loop.

**Go**: Uses blocking I/O per goroutine, but goroutines are **green threads** (~2KB stack, not 1MB). The Go runtime multiplexes goroutines onto OS threads and uses epoll/kqueue underneath. You write simple blocking code, the runtime handles the async complexity.

**Java NIO / Netty**: Java's NIO package provides non-blocking channels and selectors (wrapping epoll). Netty is the popular framework that builds high-performance networking on top of NIO. Powers major systems like Cassandra, Elasticsearch, and gRPC.

**Redis**: Single-threaded event loop using epoll. One thread handles all client connections and all operations. This works because Redis operations are fast (in-memory) -- the event loop never blocks for long. Redis 6+ added I/O threads for network read/write, but command execution is still single-threaded.

**Python asyncio**: Python's `async/await` is built on non-blocking I/O + event loop. The `asyncio` event loop uses epoll on Linux. But Python's GIL means CPU work can't parallel -- async only helps with I/O-bound workloads.

**Database connections**: This is why connection pools exist. Opening a blocking database connection per request means one thread per request. A pool of 50 connections can serve thousands of requests by recycling connections -- the application queues requests until a connection is available.

## Interview Angle

**Q: Explain the difference between blocking, non-blocking, and asynchronous I/O.**

A: Blocking: the thread sleeps until I/O completes (simple but wastes threads). Non-blocking: the call returns immediately with EAGAIN if data isn't ready, and the app must retry -- usually combined with an event loop. Async: submit the I/O request and get notified on completion -- the thread is truly free to do other work without retrying. Key distinction: non-blocking still requires the app to check back; async delivers the result to you.

**Q: How does Node.js handle thousands of connections with a single thread?**

A: Node uses non-blocking I/O with an event loop (libuv). When an I/O operation is started, Node registers interest and continues processing other events. When the OS signals readiness (via epoll/kqueue), Node calls the callback. No thread is ever sleeping on I/O. The single thread is always doing useful work -- processing callbacks, running JavaScript, or waiting in epoll_wait for the next event.

**Q: What is the C10K problem?**

A: The challenge of handling 10,000 concurrent connections on a single server. With the traditional thread-per-connection model, 10K threads consume massive memory (10GB+ in stacks) and context switching kills performance. The solution is non-blocking I/O with event-driven architecture -- one thread monitors thousands of connections using epoll/kqueue, processing data as it arrives.

**Q: Why does Go use blocking I/O if non-blocking is more scalable?**

A: Go uses blocking I/O per goroutine for simplicity -- developers write straightforward sequential code. But goroutines are lightweight (~2KB stack) and multiplexed onto a small number of OS threads by the runtime. Under the hood, the Go runtime uses epoll/kqueue for non-blocking I/O. You get the programming simplicity of blocking I/O with the scalability of non-blocking -- the runtime absorbs the complexity.

---

**Next:** [I/O Multiplexing: select, poll, epoll](05-io-multiplexing-select-poll-epoll.md) -- how to monitor thousands of connections efficiently.
