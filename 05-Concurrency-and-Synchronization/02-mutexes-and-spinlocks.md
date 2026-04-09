# Mutexes and Spinlocks

## Two Philosophies for Waiting

When a thread finds a lock already held, it has two choices:

1. **Go to sleep** (mutex) — give up the CPU, let other threads run, wake up when the lock is available
2. **Keep checking** (spinlock) — burn CPU cycles in a tight loop until the lock becomes free

Neither is universally better. The right choice depends on how long you expect to wait and how many CPU cores you have.

> **Analogy:** A mutex is like a doctor's waiting room -- you take a seat, close your eyes, and a nurse wakes you when the doctor is free. A spinlock is like standing at the door, repeatedly peeking in to ask "are you done yet?" The waiting room is efficient if the appointment is long, but if the doctor only needs 10 seconds, sitting down and getting called back costs more than just standing there.

## Mutex (Mutual Exclusion Lock)

A mutex is the workhorse of synchronization. Lock it before entering a critical section, unlock it when you're done.

```
mutex_lock(&m);
// >>> CRITICAL SECTION <<<
// Only one thread executes this at a time
counter++;
mutex_unlock(&m);
```

**What happens when a thread can't get the lock:**

```
Thread A                         Thread B
--------                         --------
mutex_lock(&m)  -> acquired
                                 mutex_lock(&m)  -> BLOCKED
// doing work...                 // moved to wait queue
                                 // context switch to other work
mutex_unlock(&m)
                                 // woken up, acquires lock
                                 // context switch back
                                 mutex_lock(&m)  -> acquired
```

**Key behavior:**
- Thread is **put to sleep** (moved from running to waiting state)
- OS performs a **context switch** so other threads can use the CPU
- When the lock is released, a waiting thread is **woken up**
- Involves **two context switches** minimum (sleep + wake)

### Cost of a Mutex

```
Context switch cost:  ~1-10 microseconds (depends on OS/hardware)
Lock/unlock (uncontended): ~25 nanoseconds on modern x86

If critical section < context switch time:
    Mutex overhead dominates -> consider spinlock
If critical section > context switch time:
    Mutex is efficient -> sleeping saves CPU
```

## Spinlock

A spinlock doesn't put the thread to sleep. Instead, it loops continuously, checking if the lock is free.

```
// Simplified spinlock using atomic test-and-set
void spin_lock(spinlock_t *lock) {
    while (test_and_set(&lock->locked))
        ;  // spin — burn CPU cycles
}

void spin_unlock(spinlock_t *lock) {
    lock->locked = 0;
}
```

**What happens when a thread can't get the lock:**

```
Thread A (Core 0)                Thread B (Core 1)
-----------------                -----------------
spin_lock(&s) -> acquired
                                 spin_lock(&s) -> spinning...
// doing work (short!)           // while(test_and_set(&locked))
                                 //     ; // CPU is 100% busy here
spin_unlock(&s)
                                 spin_lock(&s) -> acquired!
                                 // No context switch needed!
```

**Key behavior:**
- Thread **stays on the CPU**, burning cycles checking the lock
- **No context switch** — when the lock is released, the spinning thread acquires it immediately
- Wastes CPU time but avoids sleep/wake overhead
- **Only makes sense on multiprocessor systems** — on single-core, the spinning thread prevents the lock holder from running!

### The Single-Core Disaster

```
Single-core system with spinlock:

Thread A: holds the lock, gets preempted by timer interrupt
Thread B: starts spinning, waiting for lock
          ...spinning...
          ...spinning...  (Thread A can't run to release the lock!)
          ...spinning...  (until B's time quantum expires)
Thread A: finally runs again, releases lock

Result: wasted an entire time quantum spinning for nothing
```

This is why spinlocks should only be used on multi-core systems.

## Mutex vs Spinlock Comparison

| Property | Mutex | Spinlock |
|----------|-------|----------|
| **Waiting behavior** | Sleep (block) | Busy-wait (spin) |
| **Context switch** | Yes (2 per contention) | No |
| **CPU usage while waiting** | None (thread sleeps) | 100% on one core |
| **Best for** | Long critical sections | Very short critical sections |
| **Core requirement** | Works on single-core | Needs multi-core |
| **Can sleep inside CS?** | Yes | No (never sleep holding a spinlock) |
| **Used in interrupt context?** | No (can't sleep) | Yes |
| **Overhead (uncontended)** | ~25ns | ~10ns |
| **Overhead (contended)** | Context switch (~1-10us) | Wasted CPU cycles |
| **Linux kernel usage** | `mutex_lock()` | `spin_lock()` |
| **User-space usage** | `pthread_mutex_lock()` | Rarely used directly |

**Rule of thumb:** If the critical section is shorter than a context switch (~1-10us), spinlock wins. If it's longer, mutex wins.

## Recursive (Reentrant) Mutex

A normal mutex deadlocks if the same thread tries to lock it twice:

```
mutex_lock(&m);
// ... some code calls a function that also does:
mutex_lock(&m);  // DEADLOCK! This thread already holds m
```

A **recursive mutex** allows the same thread to lock it multiple times. It keeps a count and only truly unlocks when the count reaches zero.

```
recursive_mutex_lock(&m);   // count = 1
recursive_mutex_lock(&m);   // count = 2 (same thread, allowed)
recursive_mutex_unlock(&m); // count = 1
recursive_mutex_unlock(&m); // count = 0, lock released
```

**Use with caution:** Recursive mutexes often indicate poor design. If you need one, consider refactoring so that the inner function doesn't need to re-acquire the lock.

## Read-Write Locks (rwlock)

Many workloads are read-heavy. If multiple threads only read shared data, there's no conflict — they can all proceed simultaneously. Conflict only arises when someone writes.

```
+---------+----------+---------+
|         | Reader   | Writer  |
|         | wants in | wants in|
+---------+----------+---------+
| Reader  | ALLOWED  | BLOCKED |
| inside  | (shared) |         |
+---------+----------+---------+
| Writer  | BLOCKED  | BLOCKED |
| inside  |          |         |
+---------+----------+---------+
```

**Three policies:**

| Policy | Behavior | Risk |
|--------|----------|------|
| **Reader-preference** | New readers can enter even while writers wait | Writers may starve |
| **Writer-preference** | Once a writer is waiting, no new readers enter | Readers may starve |
| **Fair (FIFO)** | Requests served in order | Balanced but less concurrent |

```
// Usage pattern
rwlock_rdlock(&rw);
// ... read shared data (multiple readers OK) ...
rwlock_rdunlock(&rw);

rwlock_wrlock(&rw);
// ... modify shared data (exclusive access) ...
rwlock_wrunlock(&rw);
```

**Real-world examples:**
- Go's `sync.RWMutex`
- Java's `ReentrantReadWriteLock`
- Linux kernel's `rw_semaphore`
- Database row-level read/write locks

## POSIX Mutex Example (Pseudocode)

```c
#include <pthread.h>

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int shared_counter = 0;

void *thread_func(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&lock);
        shared_counter++;          // Protected by mutex
        pthread_mutex_unlock(&lock);
    }
    return NULL;
}

// With 2 threads: shared_counter will correctly be 2000000
// Without the mutex: could be anything from ~1000000 to 2000000
```

## Common Bugs

### 1. Forgetting to Unlock

```c
void transfer(Account *from, Account *to, int amount) {
    pthread_mutex_lock(&from->lock);
    if (from->balance < amount) {
        return;  // BUG: lock is never released!
    }
    from->balance -= amount;
    to->balance += amount;
    pthread_mutex_unlock(&from->lock);
}
```

**Fix:** Use **RAII** (Resource Acquisition Is Initialization) in C++ (`lock_guard`), try-finally in Java, or `defer` in Go. RAII is a C++ pattern where acquiring a resource (like a lock) is tied to an object's lifetime -- the lock is released automatically when the object goes out of scope, even if an exception is thrown.

### 2. Double-Locking (Deadlock with Self)

```c
pthread_mutex_lock(&m);
do_something();              // This function also calls:
                             //   pthread_mutex_lock(&m);  DEADLOCK!
pthread_mutex_unlock(&m);
```

### 3. Wrong Lock Granularity

```
Too coarse (one lock for everything):
    +---------------------------------+
    | Lock A protects ALL shared data |
    +---------------------------------+
    Threads serialize even when accessing different data
    -> Low concurrency, poor performance

Too fine (one lock per field):
    +-------+ +-------+ +-------+ +-------+
    | Lock1 | | Lock2 | | Lock3 | | Lock4 |
    +-------+ +-------+ +-------+ +-------+
    More concurrency, but complex code and deadlock risk
    -> Hard to maintain, easy to forget a lock

Right granularity: one lock per logical data group
```

### 4. Lock Ordering Violation

```c
// Thread 1                    // Thread 2
lock(A);                       lock(B);
lock(B);  // waits for B       lock(A);  // waits for A
// DEADLOCK!
```

**Fix:** Always acquire locks in the same global order (e.g., by address or by ID).

## Real-World Connection

| System | Lock Usage |
|--------|-----------|
| **Linux kernel** | ~30,000+ lock instances. Uses spinlocks for short critical sections in interrupt context, mutexes for longer sections in process context |
| **Java** | `synchronized` keyword uses an internal monitor (like a mutex). `ReentrantLock` gives more control |
| **Go** | `sync.Mutex` for mutual exclusion, `sync.RWMutex` for read-heavy workloads |
| **PostgreSQL** | Lightweight locks (LWLocks) for internal data structures, heavyweight locks for user-visible table/row locking |
| **Rust** | `Mutex<T>` wraps the data it protects — you literally can't access the data without holding the lock (compile-time guarantee) |

## Interview Angle

**Q: When would you choose a spinlock over a mutex?**

A: Spinlock when: (1) the critical section is very short (shorter than a context switch), (2) you're on a multi-core system, and (3) you can't sleep (like in an interrupt handler). Mutex when: the critical section might take a while, you're okay with context switch overhead, or you might need to sleep while holding the lock. In user-space code, mutexes are almost always the right default. Spinlocks shine inside OS kernels.

**Q: What is a read-write lock and when would you use it?**

A: A read-write lock allows multiple concurrent readers but only one writer (with no readers). Use it when reads vastly outnumber writes — like a configuration cache that's read by every request but updated rarely. Be aware of starvation: reader-preference can starve writers, writer-preference can starve readers.

**Q: What happens if you forget to release a mutex?**

A: Any thread that subsequently tries to acquire the mutex will block forever — it's essentially a deadlock. In languages with RAII (C++ `lock_guard`, Rust `MutexGuard`) or deferred execution (Go `defer`), the lock is automatically released when the scope exits, even on error paths. This is why these patterns exist and why you should always use them.

---

**Next:** [Semaphores](03-semaphores.md)
