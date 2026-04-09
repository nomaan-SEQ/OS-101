# 07 - Memory Management

Every process believes it has an entire machine's worth of memory to itself. It can address gigabytes of space, allocate and free at will, and never worry about what other processes are doing. This is an illusion — a carefully constructed one. **Memory management** is the set of OS mechanisms and hardware support that makes this illusion possible, while ensuring processes are isolated, memory is used efficiently, and the system doesn't grind to a halt.

Memory management is consistently a top-3 interview topic alongside concurrency and processes. It spans hardware concepts (caches, TLBs, MMU), OS algorithms (page replacement, allocation strategies), and real-world engineering decisions (why your container got OOM-killed, why your database wants huge pages, why adding RAM helped more than adding CPU).

## Why This Matters

- **Every program you write depends on virtual memory.** malloc, new, mmap — they all rely on the OS's memory management subsystem to map virtual addresses to physical frames.
- **Performance debugging requires memory knowledge.** TLB misses, page faults, thrashing, swap storms — you can't diagnose these without understanding how the OS manages memory.
- **Cloud and container engineering is memory engineering.** Kubernetes memory limits, cgroup OOM kills, and swap policies are all direct applications of the concepts in this section.
- **Databases are memory management systems.** PostgreSQL's shared buffers, MySQL's buffer pool, Redis's in-memory model — understanding OS memory management helps you tune these systems.
- **Interviews test this relentlessly.** From "explain virtual memory" to "walk me through a page fault" to "what is thrashing and how do you fix it?"

## Prerequisites

| Section | Why You Need It |
|---------|----------------|
| [02 - Processes](../02-Processes/README.md) | Every process has its own address space — you need to understand process isolation and what an address space is |
| [01 - System Calls and Kernel](../01-System-Calls-and-Kernel/README.md) | Memory management splits between hardware (MMU, TLB) and kernel (page tables, page fault handlers) — you need to know how user/kernel mode transitions work |

## Reading Order

| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 1 | Memory Hierarchy & Address Spaces | [01-memory-hierarchy-and-address-spaces.md](01-memory-hierarchy-and-address-spaces.md) | The memory pyramid, virtual vs physical addresses, and the MMU |
| 2 | Contiguous Allocation & Fragmentation | [02-contiguous-allocation-and-fragmentation.md](02-contiguous-allocation-and-fragmentation.md) | The simplest allocation scheme, why it breaks, and what fragmentation is |
| 3 | Paging | [03-paging.md](03-paging.md) | The breakthrough: fixed-size pages, page tables, and multi-level page tables |
| 4 | Segmentation | [04-segmentation.md](04-segmentation.md) | Logical division of memory, segment tables, and why paging won |
| 5 | Virtual Memory & Demand Paging | [05-virtual-memory-and-demand-paging.md](05-virtual-memory-and-demand-paging.md) | The big idea: using disk as an extension of RAM, page faults, swap space |
| 6 | Page Replacement Algorithms | [06-page-replacement-algorithms.md](06-page-replacement-algorithms.md) | FIFO, LRU, Clock, Belady's anomaly, and what Linux actually does |
| 7 | TLB & Page Table Structures | [07-tlb-and-page-table-structures.md](07-tlb-and-page-table-structures.md) | Making address translation fast with TLBs, multi-level tables, huge pages |
| 8 | Thrashing & Working Set Model | [08-thrashing-and-working-set-model.md](08-thrashing-and-working-set-model.md) | When the system falls off a cliff, why it happens, and how to fix it |
| 9 | Memory-Mapped Files & mmap | [09-memory-mapped-files-and-mmap.md](09-memory-mapped-files-and-mmap.md) | mmap(), shared mappings, anonymous memory, and how malloc works internally |

## Key Interview Questions

1. **What is virtual memory and why does it exist?**
   Virtual memory gives each process the illusion of a large, contiguous, private address space. The OS + hardware (MMU) map virtual addresses to physical addresses at runtime. This provides process isolation (one process can't corrupt another's memory), allows processes to use more memory than physically available (via swap), and simplifies programming (every process starts at address 0).

2. **Walk me through what happens on a page fault.**
   A process accesses a virtual address whose page is not in physical memory. The MMU triggers a trap to the kernel. The kernel checks if the access is valid (if not, it's a segfault). It finds a free frame (or evicts a page using a replacement algorithm). It reads the needed page from disk (swap) into the frame. It updates the page table to map the virtual page to the new frame. It restarts the instruction that faulted.

3. **What is thrashing and how do you detect/fix it?**
   Thrashing occurs when the system spends more time handling page faults than executing useful work. It happens when the combined working sets of all processes exceed available physical memory. You detect it by monitoring page fault rates, swap I/O, and CPU utilization (which drops dramatically). Fix it by reducing the number of processes, adding RAM, or setting memory limits (cgroups in containers).

4. **Compare paging and segmentation.**
   Paging divides memory into fixed-size blocks (pages/frames), eliminating external fragmentation but losing logical structure. Segmentation divides memory by logical sections (code, data, stack) with variable sizes, preserving logical structure but suffering external fragmentation. Modern systems (x86-64) use paging almost exclusively, with flat segmentation as a thin compatibility layer.

5. **Why do TLBs matter and what happens on a context switch?**
   Without a TLB, every memory access would require multiple memory accesses just to walk the page table (4 extra on x86-64). The TLB caches recent translations, achieving >99% hit rates. On a context switch, the TLB must be flushed (or tagged with ASIDs) because the new process has a different page table. This is a significant hidden cost of context switching.

6. **How does mmap() differ from read()/write() for file I/O?**
   mmap() maps a file directly into the process's virtual address space. Instead of explicit read/write syscalls that copy data between kernel and user buffers, you access the file through memory loads and stores. The OS handles loading pages on demand via page faults. It's great for random access, shared mappings between processes, and large files. But it provides less control over error handling and doesn't suit sequential-only or network-backed access patterns.
