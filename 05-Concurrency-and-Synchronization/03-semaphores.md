# Semaphores

## What Is a Semaphore?

A **semaphore** is an integer variable accessed through two atomic operations:

- **wait()** (also called **P()** or **down()**) — decrement the value. If it goes negative, block the calling thread.
- **signal()** (also called **V()** or **up()**) — increment the value. If threads are waiting, wake one up.

The names P and V come from the Dutch words *proberen* (to try) and *verhogen* (to increment), coined by Dijkstra in 1965.

Think of a semaphore as a **permit counter**. Each `wait()` takes a permit. Each `signal()` returns a permit. If no permits are available, you wait.

## Two Types of Semaphores

### Binary Semaphore (Value: 0 or 1)

Acts like a mutex — only one thread can proceed at a time.

```
sem_t binary_sem;
sem_init(&binary_sem, 0, 1);  // initial value = 1

// Thread usage
sem_wait(&binary_sem);    // value: 1 -> 0 (acquired)
// >>> CRITICAL SECTION <<<
sem_post(&binary_sem);    // value: 0 -> 1 (released)
```

### Counting Semaphore (Value: 0 to N)

Controls access to N identical resources. Up to N threads can proceed simultaneously.

```
sem_t pool_sem;
sem_init(&pool_sem, 0, 5);  // 5 database connections available

// Each thread:
sem_wait(&pool_sem);         // Take a connection (value decrements)
// ... use the connection ...
sem_post(&pool_sem);         // Return the connection (value increments)
```

## How It Works Internally

```
wait(S):                         signal(S):
    S.value--                        S.value++
    if (S.value < 0)                 if (S.value <= 0)
        add this thread                  remove a thread
        to S.wait_queue                  from S.wait_queue
        block()                          wake it up
```

**Important:** The decrement/increment and the check are performed **atomically** (typically using hardware CAS or disabling interrupts inside the kernel). No race condition on the semaphore itself.

## State Transition Diagram

```
Counting Semaphore (initialized to 3):

 Value:  3     2     1     0    -1    -2
         |     |     |     |     |     |
         v     v     v     v     v     v
 State: [3]-->[2]-->[1]-->[0]-->[-1]-->[-2]
        wait  wait  wait  wait  wait   wait
              <--   <--   <--   <--    <--
             signal signal signal signal signal

 value > 0  : that many threads can still enter without blocking
 value = 0  : next wait() will block
 value < 0  : |value| threads are currently blocked in the queue
```

## Classic Use: Connection Pool

```
Scenario: Web server with 10 database connections

                 +---+---+---+---+---+---+---+---+---+---+
Connection Pool: | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |10 |
                 +---+---+---+---+---+---+---+---+---+---+
                 
Semaphore value = 10 (initially)

Thread A: sem_wait()  -> value = 9, gets connection
Thread B: sem_wait()  -> value = 8, gets connection
...
Thread J: sem_wait()  -> value = 0, gets last connection
Thread K: sem_wait()  -> value = -1, BLOCKS (no connections)

Thread A: sem_post()  -> value = 0, returns connection
                         Thread K wakes up, gets the connection
```

## Semaphore vs Mutex

| Property | Mutex | Semaphore |
|----------|-------|-----------|
| **Ownership** | Yes — only the locker can unlock | No — any thread can signal |
| **Value range** | Locked / Unlocked | 0 to N (or negative when blocking) |
| **Purpose** | Mutual exclusion | Signaling + resource counting |
| **Can signal without prior wait?** | No (must lock before unlock) | Yes (common pattern for signaling) |
| **Priority inversion handling** | Often has priority inheritance | Typically does not |
| **Typical use** | Protect a critical section | Control access to N resources, thread signaling |

### The Ownership Distinction Matters

```
Mutex — ownership enforced:
    Thread A: lock()
    Thread B: unlock()   // ERROR! B doesn't own the mutex

Semaphore — no ownership:
    Thread A: wait()     // Decrement (maybe block)
    Thread B: signal()   // Increment (this is fine and common!)
```

This "no ownership" property makes semaphores useful for **signaling between threads** — one thread waits for an event, another thread signals that the event occurred.

## Semaphore for Signaling (Not Mutual Exclusion)

```
sem_t data_ready;
sem_init(&data_ready, 0, 0);  // Start at 0 — nothing ready yet

// Producer thread
produce_data();
sem_post(&data_ready);     // Signal: "data is ready"

// Consumer thread
sem_wait(&data_ready);     // Block until data is ready
consume_data();
```

This is a pure signaling pattern. The semaphore isn't protecting a critical section — it's coordinating execution order. A mutex can't do this (you'd deadlock trying to unlock something you never locked).

## When to Use What

| Situation | Use |
|-----------|-----|
| Protect shared data from concurrent access | **Mutex** |
| Limit concurrent access to N identical resources | **Counting semaphore** |
| Signal between threads (event notification) | **Binary semaphore** (or condition variable) |
| Need priority inheritance | **Mutex** |
| Need to signal from interrupt handler | **Semaphore** (can't use mutex — no ownership in ISR) |

## POSIX Semaphore API

```c
#include <semaphore.h>

sem_t sem;

sem_init(&sem, 0, initial_value);  // 0 = shared between threads (not processes)
sem_wait(&sem);                     // P() — decrement, block if < 0
sem_post(&sem);                     // V() — increment, wake one waiter
sem_trywait(&sem);                  // Non-blocking wait (returns error if would block)
sem_getvalue(&sem, &val);           // Read current value
sem_destroy(&sem);                  // Clean up
```

**Named semaphores** (for inter-process synchronization):

```c
sem_t *sem = sem_open("/my_semaphore", O_CREAT, 0644, initial_value);
// ... use sem_wait/sem_post ...
sem_close(sem);
sem_unlink("/my_semaphore");
```

## Real-World Connection

| System | Semaphore Usage |
|--------|----------------|
| **Database connection pools** | Counting semaphore limits concurrent connections (HikariCP, pgbouncer) |
| **Thread pools** | Semaphore limits how many tasks can execute simultaneously |
| **Rate limiting** | Semaphore with timed waits controls request rate |
| **Linux kernel** | `struct semaphore` used for process-level synchronization; `down()` and `up()` |
| **Java** | `java.util.concurrent.Semaphore` — used for resource pools and throttling |
| **Kubernetes** | Resource quotas act like counting semaphores at the cluster level |

## Interview Angle

**Q: What is the difference between a binary semaphore and a mutex?**

A: The key difference is ownership. A mutex must be unlocked by the same thread that locked it. A binary semaphore can be signaled by any thread. This makes semaphores suitable for signaling patterns (one thread waits, another signals) while mutexes are strictly for mutual exclusion. Mutexes also typically support priority inheritance to prevent priority inversion, while semaphores don't.

**Q: How would you implement a connection pool using semaphores?**

A: Initialize a counting semaphore to N (the pool size). Each thread calls `sem_wait()` before using a connection, which decrements the count. When done, it calls `sem_post()` to return the connection. If all N connections are in use, the next `sem_wait()` blocks until one is returned. You'll also need a mutex to protect the actual pool data structure (the list of available connections).

**Q: Can you use a semaphore to enforce execution order between threads?**

A: Yes. Initialize the semaphore to 0. The thread that must go second calls `sem_wait()` (blocks because value is 0). The thread that must go first does its work, then calls `sem_post()`. This guarantees ordering. This is a signaling pattern that you can't do with a regular mutex.

---

**Next:** [Monitors and Condition Variables](04-monitors-and-condition-variables.md)
