# Windows Memory Management Internals

## Virtual Address Space Layout

On x86-64 Windows, every process gets a virtual address space split between user mode and kernel mode. The hardware supports 48-bit virtual addresses (256 TB), and Windows divides this:

```
+--------------------------------------------------+  0xFFFF FFFF FFFF FFFF
|                                                  |
|              KERNEL SPACE                        |
|   (shared across all processes)                  |
|   - Kernel code (ntoskrnl.exe)                   |
|   - HAL, drivers                                 |
|   - PFN database                                 |
|   - Paged and nonpaged pools                     |
|   - System cache                                 |
|   - Process page tables                          |
|                                                  |
+--------------------------------------------------+  0xFFFF 8000 0000 0000
|          (canonical address gap)                 |
+--------------------------------------------------+  0x0000 7FFF FFFF FFFF
|                                                  |
|              USER SPACE                          |
|   (private per process)                          |
|   Default: 8 TB (0 - 0x7FF FFFF FFFF)           |
|   With large address aware: 128 TB               |
|                                                  |
|   - EXE image                                    |
|   - DLLs (ntdll, kernel32, etc.)                 |
|   - Heaps                                        |
|   - Thread stacks                                |
|   - Memory-mapped files                          |
|   - PEB, TEBs                                    |
|                                                  |
+--------------------------------------------------+  0x0000 0000 0000 0000
```

This is structurally identical to Linux's user/kernel split. The canonical address gap (non-addressable region in the middle) exists because current CPUs only implement 48 bits of the 64-bit address space. Both OSes handle this the same way because it is a hardware constraint.

The default user-space limit is 8 TB, expandable to 128 TB if the executable is linked with the large-address-aware flag. Linux typically provides 128 TB of user space by default.

## VAD Tree: Tracking Virtual Address Ranges

Windows tracks which parts of the virtual address space are in use with a **VAD (Virtual Address Descriptor) tree** -- a self-balancing binary tree (AVL tree) where each node describes a contiguous range of virtual addresses.

VADs are like a property registry -- each entry describes a "parcel" of virtual memory: where it starts, how big it is, what permissions it has, and what backs it (file, page file, nothing).

```
                    VAD Root
                   /        \
              VAD (heap)    VAD (ntdll.dll)
             /     \              \
     VAD (stack)  VAD (.exe)    VAD (kernel32.dll)
```

Each VAD node contains:
- Start and end virtual page numbers
- Protection flags (read, write, execute, guard, etc.)
- Backing: file-mapped, page-file-backed, or demand-zero
- Commit charge (how much of the range is actually committed)

**Linux comparison**: Linux uses `vm_area_struct` (VMA) organized in a maple tree (previously a red-black tree + linked list before kernel 6.1). The concept is identical -- both track ranges of virtual addresses with their properties. The data structure choice differs: Windows uses an AVL tree, Linux uses a maple tree.

| Feature | Windows VAD | Linux VMA |
|---|---|---|
| Data structure | AVL tree | Maple tree (was rb-tree) |
| Per-node info | Range, protection, backing | Range, flags, backing (vm_file) |
| Query tool | !vad in WinDbg | /proc/[pid]/maps |
| Merge behavior | Adjacent compatible VADs can merge | Adjacent compatible VMAs merge |
| Lazy allocation | Reserve then commit model | mmap with MAP_NORESERVE |

A distinctive Windows feature is the **reserve vs commit** model. You can **reserve** a range of virtual addresses (creates a VAD, but no physical pages) and later **commit** portions of it (backs them with page file or physical memory). Linux's `mmap` with `MAP_NORESERVE` is similar but less explicit.

## Page Tables: Same Hardware, Different Management

Both Windows and Linux use the same x86-64 four-level page table structure because it is defined by the hardware:

```
Virtual Address
    |
    v
  PML4 (Page Map Level 4)     -- 512 entries, each covers 512 GB
    |
    v
  PDPT (Page Directory Pointer Table) -- 512 entries, each covers 1 GB
    |
    v
  PD (Page Directory)          -- 512 entries, each covers 2 MB
    |
    v
  PT (Page Table)              -- 512 entries, each covers 4 KB
    |
    v
  Physical Page (4 KB)
```

The hardware is identical. The difference is in how the OS manages these structures, fills them on page faults, and reclaims pages under memory pressure. That is where the Working Set Manager and PFN Database come in.

## Working Sets: Your Desk Analogy

The **Working Set** of a process is the set of virtual pages currently resident in physical RAM. Think of it like a desk -- you can only have so many papers (pages) on your desk (RAM) at once. When your desk is full and you need a new paper, the manager takes the least-used paper and files it away (pages it to disk).

```
Process Working Set
+-------+-------+-------+-------+-------+
| Page  | Page  | Page  | Page  | Page  |  <-- Pages in RAM
| 0x401 | 0x402 | 0x7FF | 0x100 | 0x200 |
+-------+-------+-------+-------+-------+
  code    code    stack   heap    DLL

Working Set Limit: 1024 pages (4 MB default min)
```

The **Working Set Manager** (a kernel thread called `KeBalanceSetManager`) monitors memory pressure and trims working sets when physical memory runs low. It does this by:

1. Aging pages in each working set (tracking access bits)
2. Removing old, unaccessed pages from working sets
3. Moving trimmed pages to the **standby list** (still in RAM, but reclaimable)

This is similar to Linux's `kswapd` daemon and the page reclaim mechanism using LRU lists (active/inactive). The key philosophical difference: Windows tracks working sets per-process explicitly, while Linux uses global LRU lists and per-cgroup memory limits.

## The PFN Database: Tracking Every Physical Page

The **PFN (Page Frame Number) Database** is a massive array with one entry per physical page frame. If you have 16 GB of RAM (about 4 million 4 KB pages), the PFN database has 4 million entries. Each entry tracks:

- Page state: Free, Zeroed, Standby, Modified, Active, Bad, Transition
- Owning process (if active)
- Reference count
- Page table entry that maps this page

```
Physical Page States and Transitions:

                    +--------+
         +--------->| Active |<---------+
         |          +---+----+          |
    fault|              | trim          | fault
    (demand)            v               | (soft)
         |          +--------+          |
         |    +---->|Standby |----------+
         |    |     +---+----+
         |    |         | repurpose
         |    |         v
    +----+----+-+   +--------+
    |  Zeroed   |<--| Free   |
    +-----------+   +--------+
         ^              ^
         |              |
    zero thread    modified writer
         |              |
         |          +--------+
         +----------+Modified|
                    +--------+
```

**State meanings**:

| State | Description | What Happens Next |
|---|---|---|
| Active | In a process working set, being used | Stays until trimmed |
| Standby | Removed from working set, still in RAM, clean | Reused on soft fault or repurposed for another process |
| Modified | Removed from working set, dirty | Written to disk by Modified Page Writer, then becomes Standby |
| Free | Not associated with anything | Zeroed by the Zero Page Thread |
| Zeroed | Free and zero-filled | Ready for immediate use (security: no data leaks) |
| Bad | Hardware error detected | Never used |

The standby list is crucial for performance. When a process faults on a page that is in the standby list, the page is simply moved back to the Active state -- no disk I/O needed. This is a **soft page fault** and it is fast. Linux has an equivalent concept in its inactive list -- pages can be rescued before being reclaimed.

**Linux comparison**: Linux uses the buddy allocator for physical page management and maintains active/inactive LRU lists. There is no single "PFN database" structure, but `struct page` (one per physical page frame) serves a similar role. The `meminfo` categories (Active, Inactive, Cached, Buffers) roughly correspond to Windows' page states.

## Section Objects: Windows' mmap

**Section objects** are Windows' mechanism for memory-mapped files and shared memory. They are the equivalent of Linux's `mmap()`.

```
Process A                  Kernel                    Process B
+---------+           +-------------+           +---------+
| Virtual |           |  Section    |           | Virtual |
| Address |---------->|  Object     |<----------| Address |
| 0x1000  |   map     | (backed by  |   map     | 0x5000  |
+---------+           |  file.dat)  |           +---------+
                      +------+------+
                             |
                             v
                      +-------------+
                      | file.dat    |
                      | on disk     |
                      +-------------+
```

Types of section objects:
- **File-backed**: Map a file into memory (like `mmap(fd, ...)` in Linux)
- **Page-file-backed**: Anonymous memory that can be shared between processes (like `mmap(MAP_SHARED | MAP_ANONYMOUS)` or POSIX `shm_open`)
- **Image sections**: Special type for loading executables and DLLs (copy-on-write)

Named section objects (placed in `\BaseNamedObjects`) allow unrelated processes to share memory by name, similar to POSIX shared memory objects in `/dev/shm`.

## Page File: Windows' Swap

The page file (`pagefile.sys`, typically on `C:\`) serves the same role as Linux swap space: it stores pages that have been evicted from physical RAM.

Key differences from Linux swap:
- Windows can dynamically grow the page file. Linux swap partitions have a fixed size (swap files can grow, but this is less common).
- Windows uses the page file for committed private memory that has been modified and trimmed. It does not page out file-backed pages to the page file -- those go back to the original file (same as Linux).
- The commit limit = physical RAM + page file size. If total committed memory exceeds this, allocations fail.

## Large Pages

Windows supports **large pages** (2 MB on x86-64) for performance-critical applications. Large pages reduce TLB misses because one TLB entry covers 2 MB instead of 4 KB.

Requirements:
- The process needs the `SeLockMemoryPrivilege` privilege
- Pages are allocated from contiguous physical memory
- Large pages are not pageable (always resident in RAM)

SQL Server and JVMs commonly use large pages for buffer pools. In Linux, large pages are managed through Transparent Huge Pages (THP) or explicit `hugetlbfs`.

## Memory Compression (Windows 10+)

Starting with Windows 10, the Memory Manager can **compress** pages instead of writing them to the page file. Instead of putting overflow pages in slow storage (disk), Windows tries vacuum-packing them (compression) to keep them in RAM. Decompressing from RAM is much faster than reading from disk.

```
Normal paging:                Compression:
Page trimmed                  Page trimmed
    |                             |
    v                             v
Write to pagefile.sys         Compress in RAM
(~5-10 ms disk I/O)          (~0.01 ms CPU)
    |                             |
    v                             v
Page fault: read from disk    Page fault: decompress from RAM
(~5-10 ms disk I/O)          (~0.01 ms CPU)
```

Compressed pages are stored in a special "compression store" managed by the System process's working set. This is visible in Task Manager as "Memory Compression" under the System process.

Linux has an equivalent: **zswap** (compresses pages before writing to swap, keeping a compressed cache in RAM) and **zram** (creates a compressed block device in RAM used as swap). The concept is identical -- trade CPU cycles for reduced disk I/O.

## Kernel Memory Pools

The kernel allocates its own memory from two pools:

| Pool | Description | Linux Equivalent |
|---|---|---|
| Nonpaged pool | Always in physical RAM, never paged out. Used for structures accessed at high IRQL (interrupt level). | kmalloc with GFP_ATOMIC / GFP_KERNEL |
| Paged pool | Can be paged to disk. Used for less time-critical kernel allocations. | vmalloc or kmalloc with GFP_KERNEL |

Pool tags (4-byte identifiers) mark each allocation, making it possible to track which driver or component is using kernel memory. The `poolmon.exe` tool or `!poolused` WinDbg command shows pool usage by tag. This is similar to Linux's `/proc/slabinfo` for slab allocator statistics.

## Comparison: Linux MM vs Windows MM

| Feature | Linux | Windows |
|---|---|---|
| Address space tracking | vm_area_struct (maple tree) | VAD (AVL tree) |
| Physical page tracking | struct page, buddy allocator | PFN Database |
| Page replacement | LRU lists (active/inactive), kswapd | Working Set Manager, standby/modified lists |
| File cache | Page cache (unified) | System cache (Cache Manager) |
| Memory-mapped files | mmap() | Section objects (CreateFileMapping) |
| Swap | Swap partition/file | Page file (pagefile.sys) |
| Huge pages | THP, hugetlbfs | Large pages (SeLockMemoryPrivilege) |
| Memory compression | zswap, zram | Memory Compression (Win10+) |
| Kernel allocator | Slab/SLUB allocator | Pool allocator (paged/nonpaged) |
| OOM handling | OOM killer (kills a process) | Hard commit limit (allocation fails) |
| Query tools | /proc/meminfo, vmstat | Task Manager, Resource Monitor, !pfn in WinDbg |

One important philosophical difference: Linux uses an **OOM killer** that terminates a process when memory is exhausted. Windows uses a **commit limit** -- when committed memory reaches physical RAM + page file size, new allocations fail with an out-of-memory error, but no process is killed. This makes Windows more predictable but means applications must handle allocation failures gracefully.

## Real-World Connection

When a game on Windows stutters every few seconds, the Memory Manager is often involved. The Working Set Manager might be trimming the game's pages because a background process is consuming too much memory. The trimmed pages go to the standby list, and the game takes soft faults to get them back -- but those faults still cost time. A performance engineer uses Resource Monitor to check the working set size, standby list depth, and page fault rate, then adjusts the game's working set minimum (using `SetProcessWorkingSetSizeEx`) or kills background processes.

Database administrators tune SQL Server's memory by granting `SeLockMemoryPrivilege` for large pages, setting a maximum server memory to control the working set, and monitoring the PFN database state for signs of memory pressure (high modified list = disk bottleneck, low standby list = no buffer for soft faults).

## Interview Angle

**Q: How does Windows manage virtual memory differently from Linux?**

A: Both use hardware page tables (PML4/PDPT/PD/PT) and demand paging, but the management layer differs. Windows tracks virtual ranges with VADs (AVL tree) and physical pages with the PFN Database -- a per-page-frame array with states like Active, Standby, Modified, Free, and Zeroed. Page replacement uses per-process Working Sets trimmed by the Balance Set Manager. Linux uses VMAs (maple tree) for virtual ranges, struct page for physical pages, and global LRU lists (active/inactive) reclaimed by kswapd. Windows has a reserve-then-commit model where virtual space can be reserved without backing, which is more explicit than Linux's overcommit approach. Under extreme pressure, Linux kills a process (OOM killer) while Windows fails allocations (commit limit).

**Q: What is the standby list and why does it matter for performance?**

A: The standby list holds clean pages that have been trimmed from working sets but remain in physical RAM. When a process faults on a standby page, it is a soft fault -- the page is simply moved back to Active state with no disk I/O. This acts as a second-chance cache. A large standby list means the system has memory headroom. A small or empty standby list means memory pressure is high and faults will require expensive disk reads. This is equivalent to Linux's inactive list, where pages get a second chance before being reclaimed.

---

Next: [File Systems: NTFS and ReFS](04-windows-file-systems-ntfs-refs.md)
