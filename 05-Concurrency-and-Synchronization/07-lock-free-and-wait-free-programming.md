# Lock-Free and Wait-Free Programming

## Beyond Locks

Locks work. They're well-understood and correct when used properly. But they have costs:

- **Blocking:** A thread holding a lock can be preempted, and everyone else waits
- **Priority inversion:** A low-priority thread holds a lock needed by a high-priority thread
- **Deadlock potential:** Multiple locks can create circular dependencies
- **Contention:** Under high load, threads pile up waiting for the same lock

Lock-free programming eliminates these problems by using **atomic hardware instructions** instead of locks. The tradeoff: the code becomes much harder to write correctly.

**This is an advanced topic.** Understand the concepts and know when to use them, but don't prematurely optimize — locks are correct, simpler, and fast enough for the vast majority of code.

## Progress Guarantees

| Guarantee | Definition | What It Means |
|-----------|-----------|---------------|
| **Blocking** (locks) | A thread can prevent all others from making progress | Normal mutex/spinlock behavior |
| **Lock-free** | At least one thread makes progress at any time | No deadlock possible, but individual threads may starve |
| **Wait-free** | Every thread makes progress in a bounded number of steps | No deadlock AND no starvation — strongest guarantee |

```
Strength of guarantee:

  Blocking  <  Lock-free  <  Wait-free
  (weakest)                  (strongest)
  
  Ease of implementation:
  
  Blocking  >  Lock-free  >  Wait-free
  (easiest)                  (hardest)
```

Most "lock-free" data structures in practice are truly lock-free (not wait-free). Wait-free algorithms exist but are rarely used due to complexity and overhead.

## The Key Primitive: Compare-And-Swap (CAS)

CAS is the foundation of all lock-free programming. It's a single atomic hardware instruction.

```
// Pseudocode — this entire operation is ONE atomic instruction
bool CAS(int *addr, int expected, int new_value) {
    // Atomically:
    if (*addr == expected) {
        *addr = new_value;
        return true;    // Success: value was updated
    }
    return false;       // Failure: someone else changed it
}
```

**How to use it:** Read the current value, compute the new value, try to CAS. If CAS fails (someone else modified it), retry.

```
// Lock-free increment
void atomic_increment(int *counter) {
    int old_val, new_val;
    do {
        old_val = *counter;              // Read current value
        new_val = old_val + 1;           // Compute new value
    } while (!CAS(counter, old_val, new_val));  // Try to swap
    // If CAS fails, another thread changed it — retry
}
```

**Hardware instructions:**

| Architecture | Instruction |
|-------------|-------------|
| x86/x64 | `CMPXCHG` (compare and exchange) |
| ARM | `LDREX`/`STREX` (load-exclusive/store-exclusive) or `CAS` (ARMv8.1+) |
| RISC-V | `LR`/`SC` (load-reserved/store-conditional) |

## Lock-Free Stack Example

A classic lock-free data structure: a stack using CAS to update the head pointer.

```
struct Node {
    int data;
    Node *next;
};

Node *head = NULL;  // Top of stack (shared)

void push(int value) {
    Node *new_node = new Node(value);
    do {
        new_node->next = head;               // Point to current top
    } while (!CAS(&head, new_node->next, new_node));
    // If head changed between read and CAS, retry
}

int pop() {
    Node *old_head;
    do {
        old_head = head;
        if (old_head == NULL) return EMPTY;
    } while (!CAS(&head, old_head, old_head->next));
    // If head changed between read and CAS, retry
    int value = old_head->data;
    free(old_head);  // Careful! (see ABA problem below)
    return value;
}
```

```
Push operation:

Before:  head -> [B] -> [A] -> NULL

Thread 1 pushes C:
  new_node->next = head  (points to B)
  CAS(&head, B, C)       (success!)
  
After:   head -> [C] -> [B] -> [A] -> NULL
```

## The ABA Problem

The biggest pitfall of CAS-based programming.

```
Thread 1:                          Thread 2:
old_head = A                       
(gets preempted)                   
                                   pop() -> removes A
                                   pop() -> removes B
                                   push(A) -> A is back on top!
                                     (same pointer, maybe different data)
(resumes)
CAS(&head, A, A->next)
// CAS succeeds! head was A, is now A.
// But A->next is WRONG — B was removed!
// The stack is now corrupted.
```

**The problem:** CAS checks if the value is the same, but it can't tell if the value changed and changed back. A went to B went back to A — CAS sees A both times and thinks nothing happened.

### Solutions to ABA

| Solution | How It Works |
|----------|-------------|
| **Tagged pointers** | Pair each pointer with a version counter. CAS on the pair. Even if the pointer is the same, the version number differs |
| **Hazard pointers** | Before accessing a node, publish it as "hazard." Other threads check hazard lists before freeing memory |
| **Epoch-based reclamation** | Divide time into epochs. Only free memory when all threads have passed the epoch where the node was removed |
| **Garbage collection** | Let the GC handle memory reclamation (Java, Go). ABA is mostly a C/C++ problem |

```
Tagged pointer approach:

Instead of CAS(&head, old_ptr, new_ptr):

struct TaggedPtr {
    Node *ptr;
    int   tag;    // Incremented on every modification
};

CAS(&head, {A, 5}, {C, 6})

Even if ptr returns to A, tag is now 7 (not 5), so CAS fails correctly.
```

## Memory Ordering and Memory Barriers

Modern CPUs and compilers **reorder instructions** for performance. This is invisible to single-threaded code but breaks multi-threaded lock-free code.

```
// Thread 1                    // Thread 2
data = 42;                     while (!ready) ;
ready = true;                  print(data);

// You'd expect: prints 42
// But CPU might reorder Thread 1's writes:
//   ready = true BEFORE data = 42
// Thread 2 sees ready=true but data is still uninitialized!
```

**Memory barriers** (fences) prevent reordering:

| Barrier Type | Guarantee |
|-------------|-----------|
| **Acquire** | No reads/writes after this can be reordered before it. Used when acquiring a resource |
| **Release** | No reads/writes before this can be reordered after it. Used when releasing a resource |
| **Full fence** | No reordering across the barrier in either direction |

```
// Thread 1                    // Thread 2
data = 42;                     while (!load_acquire(&ready)) ;
store_release(&ready, true);   // Acquire barrier: guaranteed to
// Release barrier: data=42    // see data=42 because Thread 1's
// is visible before ready=true // release pairs with our acquire
                               print(data);  // Always prints 42
```

**In practice:** Use your language's atomic types, which handle memory ordering correctly:

| Language | Atomic Support |
|----------|---------------|
| **C++** | `std::atomic<T>` with `memory_order_acquire`, `memory_order_release`, `memory_order_seq_cst` |
| **Java** | `AtomicInteger`, `AtomicReference`, `volatile` keyword (provides acquire/release) |
| **Go** | `sync/atomic` package: `atomic.LoadInt64`, `atomic.StoreInt64`, `atomic.CompareAndSwapInt64` |
| **Rust** | `std::sync::atomic` with explicit `Ordering` (most restrictive: `SeqCst`) |
| **C** | `_Atomic` qualifier, `<stdatomic.h>` |

## Lock-Free Data Structures in Practice

| Language / Library | Lock-Free Structures |
|-------------------|---------------------|
| **Java** | `AtomicInteger`, `AtomicLong`, `AtomicReference`, `ConcurrentHashMap`, `ConcurrentLinkedQueue` |
| **C++** | `std::atomic`, Boost.Lockfree (queue, stack) |
| **Go** | `sync/atomic` primitives. Channels are lock-based internally |
| **Linux kernel** | RCU (Read-Copy-Update) — lock-free read access to shared data structures |
| **Intel TBB** | `concurrent_queue`, `concurrent_hash_map` |

### RCU: The Linux Kernel's Lock-Free Secret Weapon

Read-Copy-Update is the most successful lock-free technique in practice:

```
Readers:   Access shared data with NO locking at all (just read)
Writers:   1. Copy the data structure
           2. Modify the copy
           3. Atomically swap the pointer to the new version
           4. Wait for all current readers to finish (grace period)
           5. Free the old version

Result:    Readers are completely lock-free (zero overhead)
           Writers pay the cost of copying + waiting
           Perfect for read-heavy kernel data structures
```

RCU is used throughout the Linux kernel for routing tables, filesystem structures, and module lists — anywhere reads vastly outnumber writes.

## When to Use Lock-Free Programming

**Use it when:**
- You have a **proven hot path** with high lock contention
- You've profiled and locks are the bottleneck
- You need **progress guarantees** (real-time systems, interrupt handlers)
- You're building infrastructure (database engine, OS kernel, concurrent runtime)

**Don't use it when:**
- You haven't profiled (premature optimization)
- A simple mutex works fine (it usually does)
- Correctness matters more than performance (it almost always does)
- Your team doesn't have deep expertise in memory ordering

```
Decision tree:

Is there measurable lock contention?
  |
  No --> Use a lock. Stop here.
  |
  Yes --> Can you reduce contention (finer granularity, shorter CS)?
           |
           No --> Have you profiled to confirm locks are the bottleneck?
                   |
                   Yes --> Consider lock-free, but use proven libraries
                           (Java ConcurrentHashMap, not your own)
                   |
                   No --> Profile first. Really.
```

## Real-World Connection

| System | Lock-Free Usage |
|--------|----------------|
| **Linux kernel** | RCU for read-heavy data structures, atomic operations for counters and flags |
| **Java ConcurrentHashMap** | Lock-free reads using volatile reads and CAS. Locks only for writes (segment-level) |
| **LMAX Disruptor** | Lock-free ring buffer processing 6 million transactions per second |
| **Databases** | Lock-free skip lists in MemSQL/SingleStore, lock-free B-trees in research |
| **Network stacks** | DPDK uses lock-free ring buffers for zero-copy packet processing |
| **Garbage collectors** | Concurrent GC (Java G1, Go GC) uses CAS extensively for thread-safe marking |

## Interview Angle

**Q: What is the difference between lock-free and wait-free?**

A: Lock-free guarantees that at least one thread makes progress at any point — no deadlock is possible, but individual threads might retry indefinitely (starvation). Wait-free guarantees that every thread completes in a bounded number of steps — no deadlock and no starvation. Wait-free is strictly stronger but much harder to implement. Most practical "lock-free" data structures are lock-free, not wait-free.

**Q: Explain Compare-And-Swap (CAS) and how it enables lock-free programming.**

A: CAS atomically checks if a memory location holds an expected value and, if so, updates it to a new value. The pattern is: read the current state, compute the desired new state, CAS to apply it. If CAS fails (another thread modified the value), retry. This lets threads make progress without locks — at worst, one thread's CAS succeeds while others retry. It's implemented as a single hardware instruction (CMPXCHG on x86).

**Q: What is the ABA problem?**

A: In CAS-based algorithms, a value might change from A to B and back to A. CAS sees A both times and succeeds, but the state may have changed in between. For example, in a lock-free stack, a node might be popped and re-pushed — CAS on the head pointer succeeds but the linked structure is corrupted. Solutions include tagged pointers (pairing a version counter with the pointer), hazard pointers, or using garbage collection to prevent memory reuse.

**Q: When should you use lock-free programming versus regular locks?**

A: Use locks by default — they're simpler, well-understood, and fast enough for most cases. Only consider lock-free when you've profiled and identified lock contention as a bottleneck, and even then, prefer proven lock-free libraries (Java's ConcurrentHashMap, Go's atomic package) over writing your own. Lock-free code is extremely difficult to get right — subtle bugs around memory ordering, ABA problems, and memory reclamation can take weeks to debug.
