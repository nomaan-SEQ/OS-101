# Classic Synchronization Problems

These three problems are the canonical interview questions for concurrency. They appear in every OS textbook because they model real-world patterns that show up everywhere in systems programming. Know them cold.

## 1. Producer-Consumer (Bounded Buffer)

### The Problem

Producers generate data and place it in a fixed-size buffer. Consumers remove data from the buffer. You need to handle:

- **Buffer full** — producers must wait
- **Buffer empty** — consumers must wait
- **Simultaneous access** — buffer must be protected from concurrent modification

```
  Producer                  Bounded Buffer (size N)                Consumer
  +------+              +---+---+---+---+---+---+               +--------+
  |      | --produce--> | 1 | 2 | 3 |   |   |   | --consume--> |        |
  +------+              +---+---+---+---+---+---+               +--------+
  +------+                    ^                                  +--------+
  |      | --produce-->       |                   --consume--> |        |
  +------+              Must synchronize:                       +--------+
                        - mutual exclusion on buffer
                        - block when full/empty
```

### Solution with Semaphores

```
semaphore mutex = 1;     // Protects buffer access
semaphore empty = N;     // Counts empty slots (init to buffer size)
semaphore full  = 0;     // Counts full slots (init to 0)

Producer:                           Consumer:
    produce item                        sem_wait(full);    // Wait for item
    sem_wait(empty);   // Wait for      sem_wait(mutex);   // Lock buffer
                       // empty slot    
    sem_wait(mutex);   // Lock buffer   item = remove_from_buffer();
    add_to_buffer(item);                
    sem_post(mutex);   // Unlock        sem_post(mutex);   // Unlock
    sem_post(full);    // Signal item   sem_post(empty);   // Signal slot
                       // available                        // available
```

**Why three semaphores?**
- `mutex`: mutual exclusion on the buffer data structure
- `empty`: blocks producers when buffer is full (value hits 0)
- `full`: blocks consumers when buffer is empty (value hits 0)

**Order matters!** Always wait on the counting semaphore BEFORE the mutex. If you reverse them:

```
// DEADLOCK scenario (wrong order):
Producer:                    Consumer:
    sem_wait(mutex);  // got     sem_wait(mutex);  // BLOCKED
    sem_wait(empty);  // buffer  //  (mutex held by producer)
    // full! BLOCKS   //
    // holding mutex! //         // Can never enter to consume
    // DEADLOCK       //         // DEADLOCK
```

### Solution with Condition Variables

```
mutex_t mutex;
cond_t  not_full;
cond_t  not_empty;
int count = 0;

Producer:                               Consumer:
    produce item                            mutex_lock(&mutex);
    mutex_lock(&mutex);                     while (count == 0)
    while (count == N)                          cond_wait(&not_empty, &mutex);
        cond_wait(&not_full, &mutex);       item = remove_from_buffer();
    add_to_buffer(item);                    count--;
    count++;                                cond_signal(&not_full);
    cond_signal(&not_empty);                mutex_unlock(&mutex);
    mutex_unlock(&mutex);                   consume item
```

### Real-World Producer-Consumer

| System | Buffer | Producer | Consumer |
|--------|--------|----------|----------|
| **Kafka** | Topic partitions | Message publishers | Consumer groups |
| **RabbitMQ** | Message queues | Publishers | Subscribers |
| **Unix pipes** | Kernel pipe buffer (64KB) | Writer process | Reader process |
| **Go channels** | Buffered channel | Goroutine sending | Goroutine receiving |
| **Thread pools** | Task queue | Code submitting tasks | Worker threads |
| **Logging** | Log buffer | Application threads | Log writer thread |

---

## 2. Readers-Writers

### The Problem

Multiple threads access a shared resource (e.g., a database). Readers only read the data. Writers modify it.

**Rules:**
- Multiple readers can read simultaneously (no conflict)
- A writer needs **exclusive access** (no other readers or writers)
- You must decide how to handle the case when both readers and writers are waiting

```
Readers can overlap:          Writer needs exclusion:
  R1: ----[read]----            W1: ----[write]----
  R2:   ----[read]----          R1:                 ---[read]---
  R3:     ----[read]----        R2:                 ---[read]---
  (All OK simultaneously)       (W1 blocks until readers done,
                                 readers wait for W1 to finish)
```

### First Readers-Writers (Reader Preference)

Readers get priority. If readers are active, new readers can join even if a writer is waiting. **Risk: writer starvation.**

```
semaphore rw_mutex = 1;     // Controls access to the resource
semaphore mutex   = 1;      // Protects read_count
int read_count = 0;         // Number of active readers

Writer:                         Reader:
    sem_wait(rw_mutex);             sem_wait(mutex);
    // >>> WRITE <<<                read_count++;
    sem_post(rw_mutex);             if (read_count == 1)
                                        sem_wait(rw_mutex); // First reader locks
                                    sem_post(mutex);
                                    
                                    // >>> READ <<<
                                    
                                    sem_wait(mutex);
                                    read_count--;
                                    if (read_count == 0)
                                        sem_post(rw_mutex); // Last reader unlocks
                                    sem_post(mutex);
```

**How it works:** The first reader locks out writers. Subsequent readers just increment the count and proceed. The last reader to leave releases the lock for writers.

### Second Readers-Writers (Writer Preference)

Once a writer is waiting, no new readers can start. Active readers finish, then the writer proceeds. **Risk: reader starvation.**

This requires additional semaphores to block new readers when a writer is waiting. The solution is more complex but the concept is: a waiting writer sets a flag that prevents new readers from entering.

### Comparison

| Policy | Readers | Writers | Risk |
|--------|---------|---------|------|
| **Reader preference** | New readers always enter if readers active | Wait for all readers to finish | Writer starvation |
| **Writer preference** | Wait if any writer is waiting | Priority over new readers | Reader starvation |
| **Fair (FIFO)** | Requests served in arrival order | Requests served in arrival order | Neither starves, less concurrency |

### Real-World Readers-Writers

| System | Implementation |
|--------|---------------|
| **Go** | `sync.RWMutex` — `RLock()`/`RUnlock()` for readers, `Lock()`/`Unlock()` for writers |
| **Java** | `ReentrantReadWriteLock` — supports fair and unfair modes |
| **PostgreSQL** | `ACCESS SHARE` (read) vs `ACCESS EXCLUSIVE` (write) lock modes |
| **Linux kernel** | `rw_semaphore` — used for VFS, memory management |
| **Caching systems** | Read from cache (many readers), invalidate/update cache (writer) |

---

## 3. Dining Philosophers

### The Problem

Five philosophers sit around a circular table. Each has a plate of food. Between each pair of philosophers is a single fork (5 forks total). A philosopher needs **two forks** (left and right) to eat.

```
            P0
        f4      f0
      P4          P1
        f3      f1
          P3--P2
             f2

  P = Philosopher
  f = Fork
  Each philosopher needs both adjacent forks to eat
```

**Philosopher lifecycle:**

```
while (true) {
    think();
    pick_up_left_fork();
    pick_up_right_fork();
    eat();
    put_down_right_fork();
    put_down_left_fork();
}
```

### The Deadlock

If all five philosophers pick up their left fork simultaneously:

```
P0: picks up f0, waits for f4
P1: picks up f1, waits for f0  (held by P0)
P2: picks up f2, waits for f1  (held by P1)
P3: picks up f3, waits for f2  (held by P2)
P4: picks up f4, waits for f3  (held by P3)

Circular wait! DEADLOCK — nobody can eat, everyone starves.
```

This is the classic demonstration of how acquiring multiple resources can lead to deadlock.

### Solutions

**Solution 1: Resource Ordering (break circular wait)**

Number the forks 0-4. Always pick up the lower-numbered fork first.

```
P0: picks up min(f0,f4)=f0, then f4
P1: picks up min(f1,f0)=f0, then f1
P2: picks up min(f2,f1)=f1, then f2
P3: picks up min(f3,f2)=f2, then f3
P4: picks up min(f4,f3)=f3, then f4

Now P0 and P1 both try f0 first — one waits.
No circular dependency possible!
```

**Solution 2: Limit Diners (break hold-and-wait)**

Use a semaphore initialized to 4. At most 4 philosophers can attempt to eat at once. Since 5 forks serve 4 diners, at least one always gets both forks.

```
semaphore seats = 4;  // Only 4 can try at once

philosopher(i):
    sem_wait(seats);           // Get a seat
    pick_up_fork(left);
    pick_up_fork(right);
    eat();
    put_down_fork(right);
    put_down_fork(left);
    sem_post(seats);           // Leave the seat
```

**Solution 3: Asymmetric (break circular wait)**

Even-numbered philosophers pick up left first, odd-numbered pick up right first.

```
if (i % 2 == 0):
    pick_up(left);  then pick_up(right);
else:
    pick_up(right); then pick_up(left);
```

This breaks the circular wait because adjacent philosophers have different acquisition orders.

### Why This Problem Matters

The Dining Philosophers isn't about dinner — it's about **any system that needs multiple locks simultaneously:**

| Real System | "Philosophers" | "Forks" |
|-------------|---------------|---------|
| **Database transactions** | Transactions | Row/table locks |
| **Resource allocation** | Processes | Devices, memory, files |
| **Network protocols** | Nodes | Communication channels |
| **Microservices** | Services | Shared resources (DB, cache) |

The solutions map directly:
- Resource ordering = always acquire locks in a consistent global order
- Limit diners = acquire all locks at once or use a higher-level coordinator
- Asymmetric = break symmetry to prevent circular dependencies

## Real-World Connection

These three problems aren't academic exercises — they're the patterns you'll implement every week:

- **Producer-Consumer** is the backbone of every message queue, event system, and data pipeline
- **Readers-Writers** is how every database, cache, and configuration system handles concurrent access
- **Dining Philosophers** models any situation where you need multiple shared resources simultaneously

Understanding these patterns means you can recognize them in production code and apply proven solutions instead of reinventing (and getting wrong) synchronization from scratch.

## Interview Angle

**Q: Solve the Producer-Consumer problem using semaphores.**

A: Three semaphores: `mutex` (binary, init 1) protects the buffer, `empty` (counting, init N) tracks empty slots, `full` (counting, init 0) tracks full slots. Producer: wait(empty), wait(mutex), add item, signal(mutex), signal(full). Consumer: wait(full), wait(mutex), remove item, signal(mutex), signal(empty). Key point: wait on the counting semaphore before the mutex to avoid deadlock.

**Q: How would you prevent deadlock in the Dining Philosophers problem?**

A: The simplest and most practical solution is resource ordering: number the forks and always pick up the lower-numbered fork first. This breaks the circular wait condition. In real systems, this translates to: always acquire locks in a consistent global order (e.g., by memory address or by resource ID). Other approaches include limiting the number of concurrent diners (like a semaphore) or making one philosopher pick up forks in reverse order (asymmetric).

**Q: In the Readers-Writers problem, what's the tradeoff between reader-preference and writer-preference?**

A: Reader-preference allows new readers to join even when a writer is waiting, maximizing read concurrency but risking writer starvation — the writer might wait indefinitely if readers keep arriving. Writer-preference blocks new readers once a writer is waiting, ensuring writers get timely access but risking reader starvation. A fair FIFO approach avoids starvation but reduces concurrency. The choice depends on the workload: heavily read-skewed systems often prefer readers, while systems where write freshness matters prefer writers.

---

**Next:** [Deadlocks: Detection, Prevention, Avoidance](06-deadlocks-detection-prevention-avoidance.md)
