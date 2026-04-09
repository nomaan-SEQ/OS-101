# Thread Pools and Practical Patterns

Theory matters, but this is where threads meet production code. Thread pools are the most common threading pattern in server-side software -- understanding them is essential for building and debugging real systems.

## The Thread Pool Pattern

A thread pool pre-creates a fixed number of worker threads that sit idle, waiting for tasks from a shared queue. When a task arrives, an available thread picks it up and executes it. When the task completes, the thread returns to the pool to wait for the next one.

```
                    Task Queue
                 ┌──────────────┐
  Incoming       │ T7 T6 T5 T4  │      Thread Pool
  Tasks ──────>  │  ──────────> │      ┌────────────────────┐
                 └──────┬───────┘      │  Worker 1: [T1]    │
                        │              │  Worker 2: [T2]    │
                        ├─────────────>│  Worker 3: [T3]    │
                        │              │  Worker 4: (idle)   │
                        │              └────────────────────┘
                        │
                   Tasks dequeued
                   by idle workers
```

### Why Not Create a Thread Per Task?

```
Thread-per-task:                Thread Pool:
                                
Request 1 → create thread       Request 1 → assign to Worker 1
Request 2 → create thread       Request 2 → assign to Worker 2
Request 3 → create thread       Request 3 → assign to Worker 3
   ...                          Request 4 → queued (wait for free worker)
Request 10000 → create thread      ...
                                Request 10000 → queued
                                
Problems:                       Benefits:
- 10K thread creation overhead  - N threads created once upfront
- 10K thread stacks (~80GB)     - Bounded memory (N stacks)
- OS scheduler overwhelmed      - Scheduler handles N threads easily
- OOM crash risk                - Graceful degradation (queue grows)
```

Thread creation isn't free (~50 microseconds on Linux). For a web server handling 10,000 requests per second, that's 500 milliseconds of CPU time per second just creating and destroying threads. Thread pools amortize this cost to zero.

### How Web Servers Use Thread Pools

**Apache Tomcat (Java):**
```
<Connector port="8080"
           maxThreads="200"        ← pool size
           minSpareThreads="25"    ← minimum idle threads
           acceptCount="100"       ← queue size before rejecting
           connectionTimeout="20000" />
```

**Nginx worker model:**
```
Nginx uses a different but related approach:
- Fixed number of worker PROCESSES (not threads)
- Each worker uses an event loop (epoll/kqueue)
- Small thread pool for blocking operations (file I/O)

worker_processes auto;   # typically = number of CPU cores
thread_pool default threads=32 max_queue=65536;
```

## Thread Pool Sizing

Getting the pool size right is critical. Too few threads = underutilized CPUs and high latency. Too many = excessive context switching and memory waste.

### CPU-Bound Tasks

If threads spend most of their time computing (not waiting for I/O):

```
Optimal pool size = Number of CPU cores

Why: Each thread can saturate one core. More threads than cores
means context switching with no benefit.

Example: Image processing, encryption, compression
- 8-core machine → 8 threads
- Maybe N+1 to keep CPUs busy during occasional pauses
```

### I/O-Bound Tasks

If threads spend significant time waiting for I/O (network, disk, database):

```
Optimal pool size = N_cores * (1 + Wait_time / Compute_time)

Example: Web server making database calls
- 8 cores
- Average request: 20ms total, 2ms compute, 18ms waiting for DB
- Wait/Compute ratio = 18/2 = 9
- Pool size = 8 * (1 + 9) = 80 threads

Why: While one thread waits for I/O, others can use the CPU.
The more time spent waiting, the more threads you need to keep
CPUs busy.
```

### Sizing Summary

| Workload Type | Formula | Example (8 cores) |
|--------------|---------|-------------------|
| Pure CPU-bound | N_cores | 8 threads |
| Mixed (50/50) | N_cores * 2 | 16 threads |
| I/O-heavy (90% wait) | N_cores * 10 | 80 threads |
| Very I/O-heavy (99% wait) | N_cores * 100 | 800 threads |

In practice, you benchmark and tune. These formulas give you a starting point, not a final answer.

## Producer-Consumer Pattern

The thread pool is actually an instance of the broader producer-consumer pattern, which you'll see constantly in concurrent systems.

```
  Producers                  Bounded Queue              Consumers
  (create tasks)             (buffer)                   (process tasks)

  ┌──────────┐              ┌───────────┐              ┌──────────┐
  │ Request  │──┐           │           │        ┌────>│ Worker 1 │
  │ Handler  │  │    ┌─────>│  T  T  T  │────────┤     └──────────┘
  └──────────┘  │    │      │           │        │     ┌──────────┐
  ┌──────────┐  ├────┘      └───────────┘        ├────>│ Worker 2 │
  │ Request  │──┤                                │     └──────────┘
  │ Handler  │  │     Blocks when full       Blocks    ┌──────────┐
  └──────────┘  │     (backpressure)        when empty>│ Worker 3 │
  ┌──────────┐  │                                      └──────────┘
  │ Request  │──┘
  │ Handler  │
  └──────────┘
```

Key properties:
- **Bounded queue**: Limits memory usage and provides backpressure
- **Decoupling**: Producers and consumers work at different speeds
- **Synchronization needed**: Access to the shared queue must be thread-safe (covered in the synchronization chapter)

This pattern is a preview of synchronization problems. The queue needs mutexes or lock-free data structures to prevent corruption when multiple threads access it simultaneously.

## Thread-Per-Request vs Thread Pool vs Event Loop

Three major approaches to handling concurrent requests:

| Dimension | Thread-Per-Request | Thread Pool | Event Loop |
|-----------|-------------------|-------------|------------|
| Model | New thread per request | Fixed threads, task queue | Single thread, non-blocking I/O |
| Resource usage | Unbounded (dangerous) | Bounded (predictable) | Minimal (one thread) |
| Max concurrency | Limited by OS thread limit | Limited by pool size | Limited by file descriptors |
| Latency under load | Degrades (thread creation) | Queue wait time | Can spike if event handler blocks |
| Best for | Simple apps, low traffic | Web servers, APIs | High-concurrency I/O (chat, streaming) |
| Used by | Early servlet containers | Tomcat, most app servers | Node.js, Nginx, Redis |
| Complexity | Low | Medium | High (callback/async) |

```
Thread-per-request:          Thread Pool:             Event Loop:
                                                      
Request → new thread         Request → queue →        Request → event queue
Request → new thread           worker picks up        Request → event queue
Request → new thread         Request → queue →        Request → event queue
  (thousands of threads)       worker picks up          (one thread, epoll)
                             Request → waits...       
                               (bounded workers)      ┌─────────────┐
                                                      │  while(1) { │
                                                      │    events =  │
                                                      │     poll()   │
                                                      │    handle()  │
                                                      │  }           │
                                                      └─────────────┘
```

Modern systems often combine these: Nginx uses an event loop for connections but a thread pool for file I/O. Node.js uses an event loop but offloads CPU-heavy work to worker threads.

## Common Pitfalls

### 1. Thread Leaks

A thread that never terminates consumes resources forever. This typically happens when exception handling is missing.

```
// BAD: Exception kills the thread, pool shrinks over time
void worker() {
    while (true) {
        Task task = queue.take();
        task.run();  // If this throws, thread dies silently
    }
}

// GOOD: Catch exceptions, keep the thread alive
void worker() {
    while (true) {
        Task task = queue.take();
        try {
            task.run();
        } catch (Exception e) {
            log.error("Task failed", e);
            // Thread continues to next task
        }
    }
}
```

### 2. Unhandled Exceptions Killing Threads

In Java, an uncaught exception in a thread terminates that thread. If your pool starts with 50 threads and tasks occasionally throw exceptions, the pool silently shrinks to zero.

Solution: Always set an `UncaughtExceptionHandler` and wrap task execution in try-catch.

### 3. Deadlocks in Thread Pools

If tasks submitted to a pool themselves submit tasks to the same pool and wait for results, you can deadlock:

```
Pool size: 2 threads

Thread 1: running Task A
  Task A submits Task C to pool, waits for result
Thread 2: running Task B  
  Task B submits Task D to pool, waits for result

Task C: waiting in queue (no free threads)
Task D: waiting in queue (no free threads)

DEADLOCK: All threads are waiting for tasks that need threads to run.
```

Solution: Use separate pools for parent and child tasks, or use non-blocking patterns.

### 4. Queue Unbounded Growth

An unbounded task queue can consume all available memory if producers outpace consumers:

```
Producer rate:  1000 tasks/sec
Consumer rate:  500 tasks/sec
Queue growth:   500 tasks/sec * 1KB/task = 500KB/sec = 1.7GB/hour

After a few hours: OutOfMemoryError
```

Solution: Use bounded queues with a rejection policy (throw error, block producer, or drop oldest task).

## Language-Specific Examples

### Java: ThreadPoolExecutor

```java
ExecutorService pool = new ThreadPoolExecutor(
    4,              // core pool size (min threads)
    16,             // max pool size
    60, TimeUnit.SECONDS,  // idle thread timeout
    new ArrayBlockingQueue<>(1000),  // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
);

// Submit tasks
Future<Result> future = pool.submit(() -> {
    return processRequest(data);
});
Result result = future.get();  // blocks until done
```

### Python: concurrent.futures

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=8) as pool:
    # Submit individual tasks
    future = pool.submit(process_request, data)
    result = future.result()  # blocks until done
    
    # Map over many items
    results = list(pool.map(process_item, items))
```

### Go: Worker Pool Pattern

```go
func workerPool(numWorkers int, tasks <-chan Task, results chan<- Result) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range tasks {
                results <- process(task)
            }
        }()
    }
    wg.Wait()
    close(results)
}
```

Go's goroutines are so cheap that explicit thread pools are less common. But the pattern still appears when you need to limit concurrency (e.g., limiting database connections to 20 concurrent queries).

## Real-World Connection

### Database Connection Pools

Database connection pools follow the exact same pattern as thread pools -- they pre-create N connections and reuse them:

```
Application Thread Pool          DB Connection Pool
┌────────────────────┐           ┌────────────────────┐
│  Worker 1 ─────────┼──────────>│  Connection 1      │
│  Worker 2 ─────────┼──────────>│  Connection 2      │
│  Worker 3 (waiting) │          │  Connection 3 (idle)│
│  Worker 4 ─────────┼──────────>│  Connection 4      │
└────────────────────┘           └────────────────────┘

Common config: HikariCP (Java)
  maximumPoolSize=10
  minimumIdle=5
  connectionTimeout=30000
```

The interaction between thread pool size and connection pool size matters. If your thread pool has 200 threads but your connection pool has 10 connections, 190 threads will be waiting for database connections.

### Kubernetes and Thread Pool Tuning

In containerized environments, thread pool sizing gets tricky. A container might be on a 64-core machine but limited to 2 CPUs. If your thread pool is sized based on `Runtime.getRuntime().availableProcessors()`, Java might return 64 (the host's core count) instead of 2 (the container's limit). Modern JVMs (Java 10+) are container-aware, but this has caused many production issues.

## Interview Angle

**Q: How would you size a thread pool for a web service that makes external API calls?**

A: Start with the formula: `N_cores * (1 + wait_time / compute_time)`. If each request spends 100ms waiting for external APIs and 5ms on CPU work, on an 8-core machine: `8 * (1 + 100/5) = 168 threads`. But this is a starting point -- benchmark under realistic load. Also consider: your downstream services might not handle 168 concurrent connections well. Thread pool sizing is a system-level decision, not just a local optimization.

**Q: What happens when all threads in a pool are busy?**

A: New tasks queue up. If the queue is bounded and full, the rejection policy kicks in. Common strategies: (1) throw an exception (fail fast), (2) block the caller until space is available (backpressure), (3) caller runs the task itself (CallerRunsPolicy in Java -- natural throttling), (4) discard the oldest queued task. The right choice depends on your use case -- most web servers use fail-fast (return HTTP 503) to avoid cascading failures.

**Q: Why would you use an event loop (like Node.js) instead of a thread pool?**

A: Event loops excel at I/O-bound workloads with many concurrent connections (10K+) where each connection does little CPU work. A thread pool with 10K threads wastes memory on stacks and CPU time on context switches. An event loop handles all connections with one thread and non-blocking I/O. The tradeoff: event loops are terrible for CPU-bound work (one heavy computation blocks all connections) and require async programming discipline.

---

**Next:** [Green Threads, Goroutines, and Fibers](05-green-threads-goroutines-fibers.md)
