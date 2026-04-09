# Virtual Memory and Demand Paging

Virtual memory is arguably the most important concept in all of memory management. Here's the big idea: **a process can use more memory than physically exists**. The OS uses disk storage as an extension of RAM, transparently moving data back and forth. The process never knows the difference — it sees a vast virtual address space and accesses it freely. The OS and hardware conspire to make this work.

## The Core Idea

Without virtual memory, every page a process uses must be in physical RAM at all times. But consider: does a process use all its memory simultaneously? Almost never. A text editor doesn't access its spell-check code while you're scrolling. A web server doesn't touch its admin module while handling regular requests. Most pages sit idle most of the time.

Virtual memory exploits this observation:

- Keep **actively used pages** in physical RAM (fast)
- Store **inactive pages** on disk in swap space (slow but large)
- When a process accesses an inactive page, transparently load it from disk

```
                     Virtual Address Space (large)
                    +---------------------------+
                    |  Page 0  [in RAM]         |
                    |  Page 1  [on disk]        |
                    |  Page 2  [in RAM]         |
                    |  Page 3  [on disk]        |
                    |  Page 4  [in RAM]         |
                    |  ...                      |
                    |  Page N  [on disk]        |
                    +---------------------------+
                              |
              +---------------+----------------+
              |                                |
    Physical RAM (small)              Swap Space on Disk (large)
    +------------------+              +------------------+
    | Frame 0: Page 0  |              | Page 1           |
    | Frame 1: Page 2  |              | Page 3           |
    | Frame 2: Page 4  |              | Page N           |
    | Frame 3: (free)  |              | ...              |
    +------------------+              +------------------+
```

The process sees pages 0 through N as if they're all in memory. Only some are actually in RAM. The rest are on disk, loaded on demand.

## Demand Paging

**Demand paging** is the strategy of loading pages into memory only when they're actually accessed — not before. When a process starts, the OS doesn't load its entire executable into memory. It sets up the page table with all entries marked as "not present" and lets the process begin executing. The very first instruction triggers a page fault, which loads the first page of code. Other pages are loaded as needed.

This is lazy loading applied to memory management:

- **Pure demand paging**: no page is loaded until accessed. Start with zero pages in memory.
- **Pre-paging**: load a few pages up front (e.g., the entry point and likely first-access pages) to reduce initial page faults.

Most real systems do some pre-paging for performance, but the core mechanism is demand-driven.

## Page Faults: The Critical Path

A **page fault** occurs when a process accesses a virtual address whose page is not currently in a physical frame. The valid bit in the page table entry is 0 (not present). Here's exactly what happens:

```
  +---------+     +------+     +----------+     +--------+     +---------+
  | Process |     | MMU  |     |  Kernel  |     |  Disk  |     | Process |
  | accesses| --> |checks| --> | handles  | --> | reads  | --> | resumes |
  | address |     | PTE  |     | fault    |     | page   |     |         |
  +---------+     +------+     +----------+     +--------+     +---------+
```

### Detailed Steps

```
  1. CPU executes instruction that accesses virtual address 0x7F3A_2000
     |
  2. MMU checks page table entry for page 0x7F3A2
     |
     +---> Valid bit = 1?  YES --> Normal access, no fault
     |
     +---> Valid bit = 0?  PAGE FAULT (trap to kernel)
           |
  3. Kernel's page fault handler runs:
     |
     a. Is this a valid address (in a mapped region)?
     |   NO  --> Send SIGSEGV to process (segfault) --> Process terminates
     |   YES --> Continue
     |
  4. Find a free physical frame
     |   No free frames? --> Run page replacement algorithm to evict a page
     |   (If evicted page is dirty, write it to swap first)
     |
  5. Issue disk I/O: read the faulted page from swap (or file) into the free frame
     |   This takes ~10ms -- the process is BLOCKED during this time
     |   The OS can schedule other processes while waiting
     |
  6. Disk I/O completes. Update the page table entry:
     |   - Set frame number to the newly loaded frame
     |   - Set valid bit = 1
     |   - Set referenced bit = 1
     |
  7. Restart the instruction that caused the fault
     |   The instruction re-executes, MMU finds valid PTE, access succeeds
```

### Why "Restart the Instruction"?

The faulting instruction never completed — it was interrupted mid-execution when the MMU couldn't translate the address. After the page is loaded, the CPU replays the exact same instruction from the beginning. This requires hardware support: the CPU must be able to save enough state to perfectly restart any instruction, even complex ones like x86's `rep movsb` (which copies a string byte-by-byte and could fault in the middle).

## The Performance Reality

Page faults are **catastrophically expensive** compared to normal memory access:

| Operation | Time | Relative |
|-----------|------|----------|
| Memory access (TLB hit) | ~10 ns | 1x |
| Memory access (TLB miss) | ~50 ns | 5x |
| Page fault (SSD swap) | ~100 us | 10,000x |
| Page fault (HDD swap) | ~10 ms | 1,000,000x |

A single page fault to HDD takes a million times longer than a normal memory access. Even with SSD swap, it's 10,000 times slower.

### Effective Access Time (EAT)

```
EAT = (1 - p) * memory_access_time  +  p * page_fault_time

Where p = page fault rate (probability of a fault on any access)
```

Example with HDD swap:
- Memory access time = 100 ns
- Page fault time = 10 ms = 10,000,000 ns
- If p = 0.001 (one fault per 1000 accesses):

```
EAT = 0.999 * 100 + 0.001 * 10,000,000
    = 99.9 + 10,000
    = 10,099.9 ns
    ≈ 10 us
```

A page fault rate of just **0.1% makes memory access 100x slower**. To keep performance within 10% of optimal, you need a page fault rate below 1 in 10 million (p < 0.0000001). This is why page replacement algorithms and working set management matter so much.

## Swap Space

Pages evicted from RAM need somewhere to go. That's **swap space** — a dedicated area on disk.

**Swap partition**: a raw disk partition used exclusively for swapping. Faster than a swap file because it avoids filesystem overhead. Common on Linux servers.

**Swap file**: a regular file on a filesystem used as swap. Easier to resize, works on any filesystem. Modern Linux supports swap files on most filesystems.

```
  $ swapon --show
  NAME      TYPE  SIZE  USED PRIO
  /dev/sda2 partition  8G  1.2G   -2
  /swapfile file       4G  0B     -3

  $ free -h
                total   used   free   shared  buff/cache  available
  Mem:          16Gi    8.2Gi  1.1Gi  512Mi   6.7Gi       7.0Gi
  Swap:         12Gi    1.2Gi  10.8Gi
```

### How Much Swap?

The old rule of "2x RAM" is outdated. Modern guidelines:

| Scenario | Swap Size | Rationale |
|----------|-----------|-----------|
| Desktop (16-32 GB RAM) | 4-8 GB | Enough for hibernation and occasional overflow |
| Server (64+ GB RAM) | 4-16 GB or less | Servers should rarely swap; if they do, something is wrong |
| Containers / cloud | Often 0 | Many container environments disable swap entirely |

In container environments, swap is often disabled because:
- Kubernetes doesn't account for swap in resource scheduling
- Swapping causes unpredictable latency spikes
- Memory limits (cgroups) are the primary control mechanism

## Copy-on-Write (COW)

Virtual memory enables an elegant optimization for `fork()`. When a process forks:

1. The child gets a copy of the parent's page table (not a copy of the memory)
2. All shared pages are marked **read-only** in both parent and child page tables
3. When either process writes to a page, a page fault occurs
4. The kernel copies just that one page, gives each process its own copy, and marks both as writable
5. Pages that are never written are never copied

```
  After fork() (before any writes):

  Parent page table        Physical Memory       Child page table
  +------+-------+       +------------+        +------+-------+
  | Pg 0 | Fr 5  |------>| Frame 5    |<-------| Pg 0 | Fr 5  |
  +------+-------+  RO   +------------+   RO   +------+-------+
  | Pg 1 | Fr 8  |------>| Frame 8    |<-------| Pg 1 | Fr 8  |
  +------+-------+  RO   +------------+   RO   +------+-------+
  | Pg 2 | Fr 3  |------>| Frame 3    |<-------| Pg 2 | Fr 3  |
  +------+-------+  RO   +------------+   RO   +------+-------+

  After child writes to Page 1:

  Parent page table        Physical Memory       Child page table
  +------+-------+       +------------+        +------+-------+
  | Pg 0 | Fr 5  |------>| Frame 5    |<-------| Pg 0 | Fr 5  |
  +------+-------+  RO   +------------+   RO   +------+-------+
  | Pg 1 | Fr 8  |------>| Frame 8    |        | Pg 1 | Fr 12 |--+
  +------+-------+  RW   +------------+        +------+-------+  |
  | Pg 2 | Fr 3  |------>| Frame 3    |<-------| Pg 2 | Fr 3  |  |
  +------+-------+  RO   +------------+   RO   +------+-------+  |
                          +------------+                     RW   |
                          | Frame 12   |<-------------------------+
                          | (copy of 8)|
                          +------------+
```

This makes `fork()` nearly instant, even for processes with gigabytes of memory. The typical `fork() + exec()` pattern benefits enormously — most of the parent's pages are never copied because `exec()` replaces the child's address space immediately.

## The OOM Killer

What happens when both RAM and swap are exhausted? On Linux, the **OOM (Out of Memory) killer** activates. It selects a process to terminate, freeing its memory so the system can continue.

The OOM killer scores processes based on:
- How much memory they use (higher = more likely to be killed)
- How long they've been running (longer = less likely)
- Process priority and other factors

You can adjust the score: write to `/proc/<pid>/oom_score_adj` (-1000 to 1000). Critical processes like `sshd` are often protected with a negative score.

```
  # Check OOM scores
  $ cat /proc/$(pidof mysqld)/oom_score
  150

  # Protect a process from OOM killer
  $ echo -1000 > /proc/$(pidof sshd)/oom_score_adj
```

The OOM killer is a last resort. If it's firing, your system is fundamentally under-provisioned.

## Real-World Connection

- **Why servers monitor swap usage**: any significant swap usage on a server means your working set exceeds RAM. Even a small amount of swapping can cause latency spikes. Monitoring tools (Datadog, Prometheus) alert on swap usage for this reason.
- **Kubernetes memory limits**: when you set `resources.limits.memory: 512Mi` on a container, Kubernetes uses cgroups to enforce this limit. If the container exceeds it, the kernel's OOM killer terminates it — you see `OOMKilled` in `kubectl describe pod`.
- **Overcommit**: Linux defaults to overcommitting memory — malloc succeeds even if there isn't enough physical RAM + swap. The assumption is that most allocated memory is never actually used. The OOM killer handles the case when this assumption fails. You can change this behavior via `vm.overcommit_memory` sysctl.
- **Why Redis says "disable swap"**: databases and caches that need predictable latency must never swap. Redis documentation explicitly recommends disabling swap or setting `vm.swappiness=0` to minimize the chance of page-out.

## Interview Angle

**Q: What is virtual memory and how does it work?**

Virtual memory creates the illusion that each process has a large, contiguous private address space, potentially larger than physical RAM. The OS uses disk (swap space) as an extension of RAM. Pages that are actively used stay in physical memory; inactive pages can be stored on disk. When a process accesses a page that's on disk, a page fault occurs — the OS transparently loads the page into a free frame (possibly evicting another page) and the process continues, unaware of the delay.

**Q: Walk through what happens on a page fault.**

The MMU detects that the page table entry's valid bit is 0 and triggers a trap. The kernel's page fault handler checks if the address is valid (in a mapped region). If not, it sends SIGSEGV (segfault). If valid, it finds a free frame (running page replacement if needed), issues disk I/O to read the page from swap or its backing file, blocks the process during I/O, updates the page table entry with the new frame number and sets valid=1, then restarts the faulting instruction.

**Q: What is copy-on-write and why does it matter for fork()?**

When fork() creates a child process, the kernel doesn't copy the parent's memory. Instead, both processes share the same physical pages, marked read-only. When either process writes to a page, a page fault triggers the kernel to copy just that page, giving each process its own writable copy. This makes fork() nearly instantaneous regardless of process size, and most pages are never copied (especially with the common fork+exec pattern).

**Q: Why is the page fault rate so critical for performance?**

A page fault to disk takes about 10ms (HDD) or 100us (SSD), compared to 100ns for a normal memory access — a penalty of 10,000x to 1,000,000x. Even a tiny page fault rate (0.1%) can make effective memory access 100x slower. This is why working set management, proper memory sizing, and page replacement algorithms are so important.

---

Next: [06 - Page Replacement Algorithms](06-page-replacement-algorithms.md)
