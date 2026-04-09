# Deadlocks: Detection, Prevention, and Avoidance

## What Is a Deadlock?

A **deadlock** occurs when two or more threads are waiting for each other to release resources, and none can proceed. The system is stuck — permanently.

```
Thread A                    Thread B
--------                    --------
holds Lock 1                holds Lock 2
wants Lock 2                wants Lock 1
  |                           |
  +--- waiting for B ---------+
  |                           |
  +--------- waiting for A ---+
  
  Neither can proceed. DEADLOCK.
```

This isn't a temporary delay — without intervention, these threads will wait forever.

## The Four Necessary Conditions

A deadlock can occur **if and only if** all four of these conditions hold simultaneously:

| Condition | Meaning | Example |
|-----------|---------|---------|
| **1. Mutual Exclusion** | At least one resource is held in non-sharable mode | A lock can only be held by one thread |
| **2. Hold and Wait** | A thread holds at least one resource while waiting for another | Thread holds Lock A, waits for Lock B |
| **3. No Preemption** | Resources cannot be forcibly taken away | You can't rip a lock from a thread |
| **4. Circular Wait** | A circular chain of threads, each waiting for a resource held by the next | A waits for B, B waits for C, C waits for A |

**Break any one of these, and deadlock is impossible.**

## Resource Allocation Graph (RAG)

A RAG visualizes the relationships between threads and resources:

- **Circle (O)** = Thread/Process
- **Square ([])** = Resource (dots inside = instances)
- **Arrow from thread to resource** = Thread is requesting that resource
- **Arrow from resource to thread** = Resource is assigned to that thread

```
No deadlock:                    Deadlock:
                                
  T1 ---> [R1] ---> T2          T1 ---> [R1] ---> T2
                |                 ^                |
                v                 |                v
               [R2]              [R2] <-----------+
                                  
  T1 wants R1, T2 holds R1       Circular wait!
  T2 holds R2 (no cycle)         T1->R1->T2->R2->T1
```

**Rule:** If the RAG has a cycle AND each resource has only one instance, there is a deadlock. With multiple instances per resource, a cycle is necessary but not sufficient.

## Strategy 1: Deadlock Prevention

Prevent deadlock by ensuring at least one of the four conditions cannot hold. This is a **design-time** strategy.

### Break Mutual Exclusion

Make resources sharable. For example, read-only files can be shared. But many resources (printers, write locks) are inherently non-sharable.

**Practicality:** Limited — many resources genuinely need exclusive access.

### Break Hold and Wait

Require threads to request **all** resources at once, before starting execution.

```
// Instead of:
lock(A);
// ... do work ...
lock(B);     // Might deadlock if another thread holds B and wants A

// Do:
lock_all(A, B);   // Atomically acquire both or neither
// ... do work ...
unlock_all(A, B);
```

**Practicality:** Works but reduces concurrency. Thread holds all resources even when it only needs some. Also requires knowing all needed resources upfront, which isn't always possible.

### Break No Preemption

If a thread can't get a resource, it releases all resources it currently holds and retries.

```
lock(A);
if (!try_lock(B)) {
    unlock(A);      // Release what you have
    // Retry later (maybe after backoff)
}
```

**Practicality:** Works for resources whose state can be saved and restored (like CPU registers, memory pages). Doesn't work for resources like printers or database locks mid-transaction.

### Break Circular Wait (Most Practical)

Impose a **total ordering** on all resources. Every thread must acquire resources in ascending order.

```
Resources numbered: Lock_1 < Lock_2 < Lock_3 < ... < Lock_N

Rule: Always acquire lower-numbered locks before higher-numbered locks.

Thread A:            Thread B:
lock(1);             lock(1);    // Both try 1 first
lock(2);             lock(2);    // Then 2
                     // No circular dependency possible!
```

**Why it works:** If every thread acquires locks in order 1, 2, 3, ..., then a circular wait is impossible. The thread holding the highest-numbered lock never waits for a lower-numbered lock.

**This is the most commonly used prevention technique in practice.** The Linux kernel uses lock ordering extensively, and tools like `lockdep` verify that the ordering is never violated.

## Strategy 2: Deadlock Avoidance (Banker's Algorithm)

Instead of preventing deadlock structurally, **avoid** it dynamically: before granting a resource request, check if it would lead to an **unsafe state**.

### Safe State vs Unsafe State

```
Safe state:                     Unsafe state:
There exists at least one       No guaranteed sequence of
sequence of thread executions   completions exists.
that allows ALL threads to      Deadlock MAY occur.
complete.

Safe --> might lead to         Unsafe --> does NOT guarantee
         unsafe if careless              deadlock, but no safe
Safe --> will NOT deadlock               sequence exists
```

```
              +---------------------------+
              |      All states           |
              |  +---------------------+  |
              |  |    Safe states      |  |
              |  |                     |  |
              |  +---------------------+  |
              |  |    Unsafe states    |  |
              |  |  +--------------+  |  |
              |  |  |  Deadlock    |  |  |
              |  |  +--------------+  |  |
              |  +---------------------+  |
              +---------------------------+

Deadlock is a SUBSET of unsafe states.
Avoidance keeps us in the safe region.
```

### Banker's Algorithm

Named after a banker who needs to ensure they can always satisfy all customers' maximum needs with limited cash.

**Given:**
- `Available[j]`: instances of resource j currently available
- `Max[i][j]`: max demand of thread i for resource j
- `Allocation[i][j]`: currently allocated to thread i
- `Need[i][j]`: Max[i][j] - Allocation[i][j] (remaining need)

**Safety check:** Can we find a sequence where each thread can get its remaining needs from available + resources released by completed threads?

```
Example: 3 threads, 1 resource type, 12 total instances

         Max    Allocated    Need    Available: 3
T0:       10        5         5
T1:        4        2         2
T2:        9        2         7

Is this safe?
1. T1 can finish (need 2 <= available 3)
   After T1: available = 3 + 2 = 5
2. T0 can finish (need 5 <= available 5)
   After T0: available = 5 + 5 = 10
3. T2 can finish (need 7 <= available 10)
   Safe sequence: <T1, T0, T2>  --> SAFE STATE

Now if T0 requests 1 more:
         Max    Allocated    Need    Available: 2
T0:       10        6         4
T1:        4        2         2
T2:        9        2         7

1. T1 can finish (need 2 <= available 2)
   After T1: available = 2 + 2 = 4
2. T0 can finish (need 4 <= available 4)
   After T0: available = 4 + 6 = 10
3. T2 can finish (need 7 <= available 10)
   Still safe! Grant the request.
```

### Why the Banker's Algorithm Is Rarely Used in Practice

| Limitation | Why It Matters |
|-----------|----------------|
| Must know max resource needs in advance | Threads rarely declare this |
| Number of threads must be fixed | Systems have dynamic thread creation |
| O(n^2 * m) per request (n threads, m resources) | Too slow for high-frequency allocation |
| Too conservative | Rejects safe requests that happen to lead to unsafe states |
| Doesn't handle dynamic systems | Real systems create/destroy threads and resources |

It's important for theory and interviews, but no general-purpose OS uses it.

## Strategy 3: Deadlock Detection and Recovery

**Let deadlocks happen, then detect and fix them.** This is pragmatic when deadlocks are rare.

### Wait-For Graph

Simplify the RAG by removing resource nodes. An edge from T1 to T2 means T1 is waiting for a resource held by T2.

```
Resource Allocation Graph:        Wait-For Graph:

T1 ---> [R1] ---> T2              T1 ---> T2
T2 ---> [R2] ---> T3              T2 ---> T3
T3 ---> [R3] ---> T1              T3 ---> T1
                                   
                                   CYCLE! Deadlock detected.
```

**Detection:** Run cycle detection periodically (DFS on the wait-for graph). If a cycle exists, deadlock is present.

### Recovery Options

| Method | How It Works | Downside |
|--------|-------------|----------|
| **Kill a process** | Terminate one (or all) deadlocked threads | Lost work |
| **Rollback** | Restore a thread to a previous checkpoint | Requires checkpointing overhead |
| **Preempt resources** | Take a resource from one thread, give to another | May cause inconsistency |

**Which thread to kill?** Consider: priority, how much work it's done, how many resources it holds, how many more it needs, how many threads would be affected.

## Strategy 4: Deadlock Ignorance (Ostrich Algorithm)

Just... ignore the problem. Pretend deadlocks don't happen.

**This is what most operating systems actually do.**

| OS | Strategy |
|----|----------|
| **Linux** | Ostrich algorithm for user-space. Kernel uses lock ordering + lockdep |
| **Windows** | Ostrich algorithm for user-space. Kernel has some detection |
| **macOS** | Same approach |

**Why?**
- Deadlocks in user-space are rare (well-written code avoids them)
- Prevention/avoidance overhead costs more than the occasional deadlock
- When a deadlock happens, the user just kills the frozen program
- The engineering cost of full deadlock prevention doesn't pay off

This is a valid engineering tradeoff: **the cost of the solution exceeds the cost of the problem.**

## Practical Deadlock Strategies by Domain

| Domain | Strategy Used |
|--------|--------------|
| **Application code** | Lock ordering + try-lock with backoff |
| **Database (MySQL, PostgreSQL)** | Detection: wait-for graph, timeout-based. Recovery: rollback one transaction |
| **Linux kernel** | Prevention: strict lock ordering verified by lockdep at development time |
| **Distributed systems** | Timeout-based detection. If a distributed transaction hangs too long, abort it |
| **Java** | `jstack` shows thread dump with deadlock detection. JVM can detect monitor deadlocks |

## Real-World Connection

| Scenario | What Happens |
|----------|-------------|
| **MySQL deadlock** | Two transactions lock rows in opposite order. InnoDB detects the cycle, rolls back one transaction, and returns an error to the client |
| **PostgreSQL** | Similar detection with a configurable `deadlock_timeout` (default 1 second). Checks for cycles in the wait graph |
| **Kubernetes** | Pod scheduling can deadlock when resources are fragmented. The scheduler uses backoff and retry strategies |
| **Distributed microservices** | Service A calls B, B calls C, C calls A — distributed deadlock. Usually handled by timeouts and circuit breakers |
| **Linux lockdep** | Runtime lock dependency validator. Detects potential deadlocks during development by tracking lock acquisition order |

## Interview Angle

**Q: What are the four necessary conditions for deadlock?**

A: Mutual exclusion (resources can't be shared), hold and wait (threads hold resources while waiting for others), no preemption (resources can't be forcibly taken), and circular wait (circular chain of waiting). All four must hold simultaneously. Break any one to prevent deadlock. In practice, breaking circular wait (via lock ordering) is the most common and practical approach.

**Q: Explain the Banker's Algorithm.**

A: It's a deadlock avoidance algorithm. Before granting a resource request, it checks if the system would remain in a safe state — meaning there exists at least one sequence of thread completions that allows all threads to finish. If the request would lead to an unsafe state, it's denied. The algorithm needs to know each thread's maximum resource demands upfront. It's O(n^2 * m) per check and is rarely used in practice because of its overhead and requirement to know max needs in advance, but it's important conceptually.

**Q: How does a database handle deadlocks?**

A: Databases use detection and recovery. They maintain a wait-for graph of transactions. Periodically (or on timeout), they check for cycles. When a deadlock is detected, the database picks a "victim" transaction (usually the one that has done the least work) and rolls it back, releasing its locks. The application receives an error and should retry the transaction. MySQL's InnoDB and PostgreSQL both use this approach.

**Q: What is the Ostrich Algorithm and why do real operating systems use it?**

A: The Ostrich Algorithm means ignoring deadlocks entirely. Linux and Windows use this for user-space because deadlocks are rare in practice, the overhead of prevention or detection isn't worth it, and users can just kill frozen programs. It's a pragmatic engineering tradeoff — the cost of the occasional deadlock is less than the cost of preventing all possible deadlocks. However, the kernel itself uses strict lock ordering because kernel deadlocks crash the entire system.

---

**Next:** [Lock-Free and Wait-Free Programming](07-lock-free-and-wait-free-programming.md)
