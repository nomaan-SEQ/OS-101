# Monitors and Condition Variables

## The Problem with Low-Level Primitives

Semaphores work, but they're error-prone. Swap the order of two `wait()` calls and you have a deadlock. Forget a `signal()` and threads block forever. The logic of synchronization is spread across your code with no structure.

**Monitors** solve this by packaging mutual exclusion and condition synchronization into a single, structured construct.

## Monitors

A **monitor** is a high-level synchronization construct — essentially a class or module where:

1. Only **one thread** can be active inside the monitor at any time (built-in mutual exclusion)
2. Threads can **wait** for specific conditions using condition variables
3. The compiler/runtime enforces the locking automatically

```
+------------------------------------------+
|              MONITOR                      |
|                                           |
|  +----------+  +-----------+              |
|  | shared   |  | shared    |   Only ONE   |
|  | data A   |  | data B    |   thread     |
|  +----------+  +-----------+   active at   |
|                                a time      |
|  +-------------------+                    |
|  | method1()         | <-- entry queue    |
|  | method2()         |     (threads wait  |
|  | method3()         |      to enter)     |
|  +-------------------+                    |
|                                           |
|  Condition Variables:                     |
|  [notFull]  [notEmpty]                    |
+------------------------------------------+
```

**You already use monitors if you use:**

| Language | Monitor Mechanism |
|----------|------------------|
| **Java** | `synchronized` keyword on methods/blocks |
| **C#** | `lock` statement, `Monitor.Enter/Exit` |
| **Python** | `with threading.Lock()` (manual), `Queue` (built-in monitor) |
| **Go** | `sync.Mutex` + `sync.Cond` (manual composition) |

### Java Example (Pseudocode)

```java
class BankAccount {
    private int balance;
    
    // synchronized = only one thread can execute this at a time
    public synchronized void withdraw(int amount) {
        balance -= amount;
    }
    
    public synchronized void deposit(int amount) {
        balance += amount;
    }
}
```

When Thread A calls `withdraw()`, the JVM acquires the object's intrinsic lock. If Thread B calls `deposit()` on the same object, it blocks until Thread A exits the synchronized method. No explicit lock/unlock calls needed — the monitor handles it.

## Condition Variables

Mutual exclusion alone isn't enough. Threads often need to **wait for a specific condition** before proceeding.

Example: a consumer thread needs to wait until the buffer has items. It can't just spin checking — that wastes CPU. And it can't just sleep — it needs to release the lock so producers can add items.

A **condition variable** provides exactly this: the ability to atomically **release a lock and sleep**, then **re-acquire the lock when woken up**.

### Three Operations

| Operation | What It Does |
|-----------|-------------|
| **wait(cond, mutex)** | Release the mutex, sleep until signaled, then re-acquire the mutex |
| **signal(cond)** / **notify()** | Wake up ONE thread waiting on this condition |
| **broadcast(cond)** / **notifyAll()** | Wake up ALL threads waiting on this condition |

### The Flow

```
Thread A (Consumer):                  Thread B (Producer):
                                      
mutex_lock(&m);                       
while (buffer_empty()) {              
    cond_wait(&not_empty, &m);        
    // 1. Releases mutex              
    // 2. Sleeps                      
    // ...sleeping...                 mutex_lock(&m);
    //                                add_item(buffer);
    //                                cond_signal(&not_empty);
    //                                mutex_unlock(&m);
    // 3. Woken up                    
    // 4. Re-acquires mutex           
    // 5. Re-checks condition (loop)
}
item = remove_item(buffer);
mutex_unlock(&m);
```

## Why Condition Variables Need a Mutex

You might wonder: why can't `wait()` just sleep without a mutex?

**Problem 1: Lost Signal**

```
WITHOUT mutex protection:

Consumer:                        Producer:
if (buffer_empty())              
                                 add_item(buffer);
                                 signal(not_empty);  // Signal sent
                                                     // but nobody is waiting yet!
    wait(not_empty);             // Consumer missed the signal
    // Sleeps FOREVER            // Producer already signaled
```

The mutex ensures the check-and-wait is atomic. No signal can arrive between checking the condition and going to sleep.

**Problem 2: Spurious Wakeups**

Operating systems may wake threads without an explicit signal (an implementation detail for performance). If you only check the condition once with `if`, a spurious wakeup lets you proceed when the condition isn't actually true.

## The While-Loop Pattern (Critical!)

**Always use `while`, never `if`:**

```c
// WRONG — breaks on spurious wakeup or stolen condition
mutex_lock(&m);
if (buffer_empty()) {
    cond_wait(&not_empty, &m);
}
// Another thread may have emptied the buffer
// between signal and this thread re-acquiring the lock!
item = remove_item(buffer);  // BUG: buffer might be empty!
mutex_unlock(&m);

// CORRECT — always re-check after waking up
mutex_lock(&m);
while (buffer_empty()) {
    cond_wait(&not_empty, &m);
}
item = remove_item(buffer);  // Guaranteed: buffer is not empty
mutex_unlock(&m);
```

**Two reasons for the while loop:**

1. **Spurious wakeups** — the OS might wake you for no reason
2. **Stolen condition** — between the signal and you re-acquiring the mutex, another thread might have consumed the item you were woken up for

This is one of the most common concurrency bugs. Interviewers specifically look for whether you use `while` or `if`.

## Monitor with Condition Variable: Complete Flow

```
+--------------------------------------------------+
|                    MONITOR                        |
|                                                   |
|     Entry Queue                                   |
|     [T3] [T4] [T5]  -- waiting to enter           |
|          |                                        |
|          v                                        |
|    +-----------+                                  |
|    |  Active   |     only ONE thread here          |
|    |  Thread   |                                  |
|    +-----------+                                  |
|      |       |                                    |
|   signal()  wait()                                |
|      |       |                                    |
|      |       +---> Condition Queue                |
|      |             [T1] [T2]  -- waiting for       |
|      |                         condition           |
|      +-----------> moves T1 back to entry queue    |
|                                                   |
+--------------------------------------------------+
```

**When a thread calls `wait()`:**
1. It releases the monitor lock
2. It moves to the condition's wait queue
3. Another thread from the entry queue enters

**When a thread calls `signal()`:**
1. One thread is moved from the condition queue to the entry queue
2. That thread will eventually re-enter and re-check its condition

## signal() vs broadcast()

| Use | When |
|-----|------|
| **signal() / notify()** | Only one waiter can make progress (e.g., one item added to buffer, wake one consumer) |
| **broadcast() / notifyAll()** | Multiple waiters might make progress, or you're not sure which waiter should proceed |

**When in doubt, use broadcast().** It's less efficient (wakes all waiters, most go back to sleep), but it's always correct. Using `signal()` incorrectly can cause threads to sleep forever.

## POSIX Condition Variable API

```c
#include <pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond  = PTHREAD_COND_INITIALIZER;

// Waiting side
pthread_mutex_lock(&mutex);
while (!condition_is_true()) {
    pthread_cond_wait(&cond, &mutex);  // Atomically: unlock + sleep + relock
}
// ... do work ...
pthread_mutex_unlock(&mutex);

// Signaling side
pthread_mutex_lock(&mutex);
make_condition_true();
pthread_cond_signal(&cond);    // Wake one waiter
// or: pthread_cond_broadcast(&cond);  // Wake all waiters
pthread_mutex_unlock(&mutex);
```

## Java synchronized + wait/notify (Pseudocode)

```java
class BoundedBuffer<T> {
    private Queue<T> buffer = new LinkedList<>();
    private int capacity;

    public synchronized void produce(T item) throws InterruptedException {
        while (buffer.size() == capacity) {
            wait();    // Release lock, sleep (built into Object)
        }
        buffer.add(item);
        notifyAll();   // Wake consumers
    }

    public synchronized T consume() throws InterruptedException {
        while (buffer.isEmpty()) {
            wait();    // Release lock, sleep
        }
        T item = buffer.remove();
        notifyAll();   // Wake producers
        return item;
    }
}
```

## Real-World Connection

| System | Usage |
|--------|-------|
| **Java** | Every object is a monitor. `synchronized` acquires the intrinsic lock. `wait()`/`notify()` are condition variable operations |
| **Python** | `threading.Condition` wraps a lock + condition variable. `Queue` class is a complete monitor |
| **Go** | `sync.Cond` with `Wait()`, `Signal()`, `Broadcast()` — always used with a `sync.Mutex` |
| **C++ (std)** | `std::condition_variable` with `std::unique_lock<std::mutex>` |
| **Linux kernel** | `wait_event()` / `wake_up()` macros — condition variable pattern for kernel threads |
| **Docker** | Container readiness checks use condition-variable-like patterns — wait until a service is healthy |

## Interview Angle

**Q: Why must you always use a while loop, not an if, with condition variables?**

A: Two reasons: (1) Spurious wakeups — the OS may wake your thread without anyone calling signal(), so you must re-check the condition. (2) Stolen conditions — between the signal and your thread re-acquiring the mutex, another thread might have already consumed the resource you were waiting for. The while loop guarantees you only proceed when the condition is actually true.

**Q: What is a monitor? How does Java implement it?**

A: A monitor is a synchronization construct where only one thread can be active at a time, with condition variables for waiting. In Java, every object has an intrinsic lock (monitor). The `synchronized` keyword acquires this lock automatically. `wait()` releases the lock and sleeps, `notify()`/`notifyAll()` wake waiting threads. This gives you mutual exclusion plus condition synchronization in one package.

**Q: What's the difference between signal/notify and broadcast/notifyAll?**

A: `signal()` wakes exactly one waiting thread. `broadcast()` wakes all waiting threads (they then compete to re-acquire the lock). Use `signal()` when you know only one thread can make progress (one item produced, one consumer needed). Use `broadcast()` when multiple threads might proceed or when different threads wait on different conditions using the same condition variable. When in doubt, `broadcast()` is always safe.

---

**Next:** [Classic Problems: Producer-Consumer, Readers-Writers, Dining Philosophers](05-classic-problems-producer-consumer-readers-writers-dining-philosophers.md)
