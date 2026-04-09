# Race Conditions and Critical Sections

## The Core Problem

A **race condition** occurs when the outcome of a program depends on the unpredictable timing of thread execution. Two threads access shared data, and the result changes depending on who runs first — or worse, who gets preempted in the middle of an operation.

This isn't a theoretical concern. Race conditions cause real-world disasters: corrupted databases, double-charged bank accounts, lost inventory updates, and security vulnerabilities.

## The Classic Example: Shared Counter

Two threads both try to increment a shared counter from 0 to 1:

```
Thread A                    Thread B
--------                    --------
read counter (0)
                            read counter (0)
increment (0 -> 1)
                            increment (0 -> 1)
write counter (1)
                            write counter (1)

Final value: 1  (should be 2!)
```

The problem: `counter++` looks like one operation, but it's actually three:
1. **Read** the current value from memory
2. **Increment** the value in a register
3. **Write** the new value back to memory

The scheduler can preempt a thread between any of these steps. When it does, another thread reads stale data.

```
What "counter++" actually does (x86):

    mov  eax, [counter]    ; 1. Load counter into register
    add  eax, 1            ; 2. Add 1
    mov  [counter], eax    ; 3. Store result back

Any interrupt between these instructions = potential race condition
```

## Critical Section

A **critical section** is any piece of code that accesses shared resources (variables, files, data structures, hardware). When one thread is in its critical section, no other thread should be in its critical section for the same resource.

```
+---------------------------+
|   Entry Section           |  <-- Acquire permission to enter
+---------------------------+
|   CRITICAL SECTION        |  <-- Access shared resource
|   (shared data access)    |
+---------------------------+
|   Exit Section            |  <-- Release permission
+---------------------------+
|   Remainder Section       |  <-- Non-shared code
+---------------------------+
```

## Requirements for a Correct Solution

Any solution to the critical section problem must satisfy three requirements:

| Requirement | What It Means | Why It Matters |
|-------------|---------------|----------------|
| **Mutual Exclusion** | Only one thread in the critical section at a time | Prevents data corruption |
| **Progress** | If no thread is in the CS, a waiting thread can enter without delay | Prevents unnecessary blocking |
| **Bounded Waiting** | A thread cannot wait forever to enter | Prevents starvation |

## Peterson's Solution (Historical)

Peterson's algorithm solves the critical section problem for two threads using only shared memory — no special hardware support. It's not used in practice, but it demonstrates the concepts beautifully.

```
// Shared variables
bool flag[2] = {false, false};  // flag[i] = true means thread i wants to enter
int turn;                        // whose turn it is

// Thread i (where i = 0 or 1, j = 1 - i)
flag[i] = true;        // I want to enter
turn = j;              // But I'll let you go first
while (flag[j] && turn == j)
    ;                  // Wait if you want in AND it's your turn
// >>> CRITICAL SECTION <<<
flag[i] = false;       // I'm done
```

**Why it works:**
- If both threads want to enter, the last one to set `turn` yields to the other
- Satisfies mutual exclusion, progress, and bounded waiting

**Why it's not used in practice:**
- Only works for 2 threads
- Modern CPUs reorder instructions — the reads/writes to `flag` and `turn` might execute out of order without memory barriers
- Hardware atomic instructions are simpler and faster

## Hardware Support for Synchronization

Modern processors provide atomic instructions that execute as a single, uninterruptible operation.

### Test-And-Set (TAS)

```
// Atomically:
bool test_and_set(bool *target) {
    bool old = *target;
    *target = true;
    return old;
}

// Usage: spin lock
while (test_and_set(&lock))
    ;  // spin until we get the lock
// >>> CRITICAL SECTION <<<
lock = false;
```

**Atomically** means the CPU guarantees no other core can read or write the memory location between the read and the write. The hardware bus is locked for that operation.

### Compare-And-Swap (CAS)

```
// Atomically:
bool compare_and_swap(int *value, int expected, int new_value) {
    if (*value == expected) {
        *value = new_value;
        return true;
    }
    return false;
}

// Usage: lock acquisition
while (!compare_and_swap(&lock, 0, 1))
    ;  // spin until lock is 0 and we set it to 1
// >>> CRITICAL SECTION <<<
lock = 0;
```

CAS is the foundation of all modern lock-free programming. It's available on x86 (`CMPXCHG`), ARM (`LDREX/STREX`), and every major architecture.

## Why Disabling Interrupts Doesn't Work

On a **single-core** system, you could prevent race conditions by disabling interrupts:

```
disable_interrupts();
// Critical section — no preemption possible
counter++;
enable_interrupts();
```

This works because on a single core, the only way another thread runs is via a timer interrupt that triggers the scheduler. No interrupt = no preemption.

**But on multiprocessors, this fails completely:**

```
Core 0                      Core 1
--------                    --------
disable_interrupts()
read counter (5)
                            (interrupts still enabled!)
                            read counter (5)
                            counter = 6
                            write counter (6)
counter = 6
write counter (6)
enable_interrupts()

Result: counter = 6 (should be 7)
```

Disabling interrupts on Core 0 does nothing to Core 1. And you can't disable interrupts on all cores — that would freeze the entire system. This is why we need atomic instructions and proper locks.

## Real-World Connection

| Scenario | The Race Condition |
|----------|--------------------|
| **Bank account** | Two ATM withdrawals read the same balance, both succeed, account goes negative |
| **Database lost update** | Two transactions read a row, both modify it, one update overwrites the other |
| **E-commerce inventory** | Two customers buy the last item simultaneously, both orders confirmed |
| **File system** | Two processes write to the same file, data gets interleaved/corrupted |
| **Docker container IDs** | Concurrent container creation could generate duplicate IDs without atomic counters |

Linux kernel developers deal with race conditions constantly. The kernel has thousands of spinlocks and mutexes protecting shared data structures. Tools like `lockdep` (lock dependency checker) and `KCSAN` (kernel concurrency sanitizer) help catch races at development time.

## Interview Angle

**Q: What is a race condition? Give an example.**

A: A race condition is when program correctness depends on the relative timing of thread execution. Classic example: two threads incrementing a shared counter. Each does read-modify-write. If Thread A reads the counter, then Thread B reads the same value before A writes back, one increment is lost. The fix is to make the read-modify-write atomic — either with an atomic instruction (like CAS) or by protecting the code with a lock.

**Q: What are the three requirements for a critical section solution?**

A: (1) Mutual exclusion — only one thread in the critical section at a time. (2) Progress — if no thread is in the CS, any thread that wants to enter can do so without waiting for others. (3) Bounded waiting — there's a limit on how many times other threads can enter before a waiting thread gets its turn. Missing any of these leads to either data corruption, unnecessary blocking, or starvation.

**Q: Why can't you just disable interrupts to prevent race conditions?**

A: Disabling interrupts only prevents preemption on the current CPU core. On a multiprocessor system, other cores continue executing and can access shared data concurrently. You'd have to disable interrupts on all cores, which would freeze the system. Instead, we use atomic hardware instructions (test-and-set, CAS) that work correctly across all cores.

---

**Next:** [Mutexes and Spinlocks](02-mutexes-and-spinlocks.md)
