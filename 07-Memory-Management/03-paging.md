# Paging

Paging is the single most important memory management concept in modern operating systems. It solves the external fragmentation problem by dividing both virtual and physical memory into small, fixed-size blocks. A process's pages can be scattered anywhere in physical memory — they don't need to be contiguous. This one idea enables virtual memory, demand paging, memory-mapped files, and efficient process isolation.

## The Big Idea

Instead of giving a process one contiguous block of memory, divide everything into fixed-size pieces:

- **Page**: a fixed-size block of virtual memory (what the process sees)
- **Frame**: a fixed-size block of physical memory (same size as a page)
- **Page table**: a per-process data structure that maps pages to frames

```
  Process's Virtual            Page Table         Physical Memory
  Address Space                                   (Frames)
  +-----------+               +----+----+         +-----------+
  | Page 0    | ------------> | 0  |  5 | ------> | Frame 0   |  (OS)
  +-----------+               +----+----+         +-----------+
  | Page 1    | ------------> | 1  |  2 | --+     | Frame 1   |  (other)
  +-----------+               +----+----+   |     +-----------+
  | Page 2    | ------------> | 2  |  8 | -+|+--> | Frame 2   |  Page 1
  +-----------+               +----+----+  |||    +-----------+
  | Page 3    | ------------> | 3  |  1 | +|||    | Frame 3   |  (other)
  +-----------+               +----+----+ ||||    +-----------+
                                          ||||    | ...       |
                                          |||+--> | Frame 5   |  Page 0
                                          ||+---> | Frame 8   |  Page 2
                                          |+----> | Frame 1   |  Page 3
```

The pages don't need to be in order in physical memory. Page 0 maps to Frame 5, Page 1 maps to Frame 2, etc. The process sees a contiguous address space (pages 0, 1, 2, 3), but the physical frames can be anywhere.

## Page Size

The page size is a fixed hardware parameter. The most common size is **4 KB (4096 bytes)**, used by default on x86, ARM, and most other architectures.

| Page Size | Used By | Notes |
|-----------|---------|-------|
| 4 KB | Default on x86, ARM, most systems | Good balance of granularity and overhead |
| 2 MB | x86-64 "huge pages" | Reduces TLB misses for large memory workloads |
| 1 GB | x86-64 "gigantic pages" | For very large memory applications (databases, VMs) |
| 16 KB | ARM64 (optional) | Apple Silicon uses 16 KB pages |

Why 4 KB? It's a tradeoff:
- **Smaller pages** = less internal fragmentation, finer-grained memory management, but more page table entries (more overhead)
- **Larger pages** = fewer page table entries, better TLB coverage, but more internal fragmentation and less flexibility

## Address Translation

A virtual address is split into two parts:

```
  Virtual Address (32-bit example, 4 KB pages):
  +-------------------+------------------+
  | Page Number (20b) | Offset (12b)     |
  +-------------------+------------------+
       |                      |
       v                      | (copied directly)
  Page Table Lookup           |
       |                      |
       v                      v
  +-------------------+------------------+
  | Frame Number      | Offset (12b)     |
  +-------------------+------------------+
       Physical Address
```

- **Offset** (12 bits for 4 KB pages, since 2^12 = 4096): the byte position within the page. This part is copied directly to the physical address — it doesn't change.
- **Page number** (remaining bits): used as an index into the page table to find the corresponding frame number.

### Worked Example

System: 32-bit virtual addresses, 4 KB pages, 32-bit physical addresses.

Virtual address: `0x00003A7F`
- Binary: `0000 0000 0000 0000 0011 1010 0111 1111`
- Page number (upper 20 bits): `0x00003` = page 3
- Offset (lower 12 bits): `0xA7F` = byte 2687 within the page

Look up page 3 in the page table → frame 7 (for example).
- Physical address: frame 7 base + offset = `0x00007A7F`

## Page Table Entries

Each entry in the page table stores more than just the frame number:

```
  Page Table Entry (typical):
  +-------+---+---+---+---+---+---+--------------------+
  | Frame | V | R | M | P | U | X | (other flags)      |
  +-------+---+---+---+---+---+---+--------------------+
     |      |   |   |   |   |   |
     |      |   |   |   |   |   +-- Execute permission
     |      |   |   |   |   +------ User/Kernel mode
     |      |   |   |   +---------- Protection (read/write)
     |      |   |   +-------------- Modified (dirty) bit
     |      |   +------------------ Referenced (accessed) bit
     |      +---------------------- Valid bit (is this page in memory?)
     +---------------------------- Physical frame number
```

- **Valid bit**: is this page currently in a physical frame? If not, accessing it triggers a page fault.
- **Referenced bit**: has this page been accessed recently? Used by page replacement algorithms.
- **Modified (dirty) bit**: has this page been written to? If so, it must be written back to disk before being evicted.
- **Protection bits**: read, write, execute permissions. Enforced by hardware.

## No External Fragmentation

Because every allocation is in fixed-size pages, there's no external fragmentation. Any free frame can hold any page. If you have 100 free frames scattered across physical memory, you can allocate 100 pages to a process — they don't need to be adjacent.

**Internal fragmentation** still exists but is minimal: on average, half a page is wasted per process (the last page is partially empty). With 4 KB pages, that's an average waste of 2 KB per process — negligible.

## The Problem with Flat Page Tables

A simple, flat page table has one entry per virtual page. Let's calculate the size:

**32-bit address space, 4 KB pages:**
- Number of pages: 2^32 / 2^12 = 2^20 = ~1 million entries
- Each entry: 4 bytes
- Page table size: **4 MB per process**

That's already significant. Now consider:

**64-bit address space, 4 KB pages:**
- Number of pages: 2^64 / 2^12 = 2^52 = ~4.5 quadrillion entries
- Each entry: 8 bytes
- Page table size: **32 PB (petabytes) per process**

Obviously impossible. A flat page table for 64-bit address spaces cannot exist. We need a better structure.

## Multi-Level Page Tables

The solution: **don't allocate page table entries for address ranges the process doesn't use**. Most of the 64-bit address space is empty — a process might use a few MB of code, a few MB of heap, and a few MB of stack, leaving vast regions untouched.

### Two-Level Page Table (32-bit example)

Split the page number into two parts. The first part indexes a **page directory** (level 1). Each directory entry points to a **page table** (level 2). Only allocate level-2 tables for regions that are actually in use.

```
  Virtual Address (32-bit, 4 KB pages):
  +----------+----------+----------+
  | Dir (10) | Table(10)| Off (12) |
  +----------+----------+----------+
       |           |          |
       v           |          |
  +--------+      |          |
  | Page   |      v          |
  | Dir    | -> +--------+   |
  | Entry  |    | Page   |   |
  +--------+    | Table  |   |
                | Entry  |---+-----> Physical Address
                +--------+
```

- Level 1 (page directory): 2^10 = 1024 entries, 4 KB
- Each level-2 table: 1024 entries, 4 KB
- Only allocate level-2 tables for address ranges in use

If a process uses only 12 MB of its 4 GB address space, you might only need the directory (4 KB) + 3 page tables (12 KB) = **16 KB** instead of 4 MB. Massive savings.

### Four-Level Page Table (x86-64)

Modern x86-64 processors use **4 levels** of page tables. Only 48 bits of the 64-bit virtual address are used (256 TB address space):

```
  Virtual Address (x86-64, 48-bit used, 4 KB pages):
  +-------+-------+-------+-------+-----------+
  | PML4  |  PDP  |   PD  |   PT  |  Offset   |
  |  (9)  |  (9)  |  (9)  |  (9)  |   (12)    |
  +-------+-------+-------+-------+-----------+
     |        |       |       |         |
     v        v       v       v         |
   Level 4  Level 3 Level 2 Level 1     |
   (PGD)    (PUD)   (PMD)   (PTE)      |
                                        v
                              Physical Address
```

Each level has 2^9 = 512 entries. Each table is exactly one 4 KB page (512 entries x 8 bytes = 4096 bytes). The levels are named:

| Level | x86-64 Name | Linux Name | Entries |
|-------|-------------|------------|---------|
| 4 | PML4 (Page Map Level 4) | PGD (Page Global Directory) | 512 |
| 3 | PDPT (Page Directory Pointer Table) | PUD (Page Upper Directory) | 512 |
| 2 | PD (Page Directory) | PMD (Page Middle Directory) | 512 |
| 1 | PT (Page Table) | PTE (Page Table Entry) | 512 |

**Intel recently extended this to 5 levels** (LA57) for 57-bit addresses (128 PB virtual space), used in servers that need to address very large memory.

## Inverted Page Table

An alternative approach: instead of one entry per virtual page (enormous), have **one entry per physical frame** (bounded by actual RAM size).

```
  Virtual Address: (process_id, page_number)
       |
       v
  +----------------------------------+
  | Inverted Page Table              |
  | (one entry per physical frame)   |
  +----+--------+--------+----------+
  | 0  | pid=3  | page=7 | next     |
  +----+--------+--------+----------+
  | 1  | pid=1  | page=0 | next     |
  +----+--------+--------+----------+
  | 2  | pid=3  | page=2 | next     |
  +----+--------+--------+----------+
  | ...                              |
  +----------------------------------+
```

- Only one table for the entire system (not per-process)
- Size proportional to physical memory, not virtual address space
- Used by PowerPC, IA-64 (Itanium)
- Downside: lookup requires searching the table (use hash table for speed), and sharing pages between processes is more complex

## Real-World Connection

- **Linux page tables**: Linux uses a 4-level (or 5-level with LA57) page table structure. You can see a process's page mappings in `/proc/<pid>/pagemap`. The kernel allocates page table pages on demand — creating a process doesn't immediately allocate page tables for the entire address space.
- **fork() and copy-on-write**: when you fork a process, the kernel doesn't copy all pages. It marks both parent's and child's page tables as read-only pointing to the same frames. Only when one process writes does the kernel copy the page (copy-on-write). This is a direct application of page table manipulation.
- **Shared libraries**: libc.so is loaded once in physical memory but mapped into every process's virtual address space. Different processes' page tables point to the same physical frames for the shared library code.

## Interview Angle

**Q: How does paging eliminate external fragmentation?**

External fragmentation happens when free memory is scattered in non-contiguous chunks. Paging eliminates this by allocating memory in fixed-size units (pages/frames). Any free frame can satisfy any page allocation — the pages don't need to be physically contiguous. The OS just needs to track which frames are free, and any free frame works for any page.

**Q: Why do modern systems use multi-level page tables instead of flat ones?**

A flat page table for a 64-bit address space would require petabytes of memory — obviously impossible. Multi-level page tables are sparse: they only allocate table pages for address ranges that are actually in use. Since most of the virtual address space is empty, this saves enormous amounts of memory. x86-64 uses 4 levels, and each level-N table is only allocated if some address in that range is mapped.

**Q: What happens during address translation with a 4-level page table?**

The virtual address is split into 4 index fields (9 bits each) plus a 12-bit offset. The CPU starts at the PML4 table (pointed to by the **CR3 register** -- a special CPU register that holds the physical base address of the current process's top-level page table; the OS updates CR3 on every context switch), uses the first 9 bits to index into it, follows the pointer to the next-level table, repeats for all 4 levels, and finally gets the physical frame number from the PTE. It combines this with the offset to form the physical address. Without a TLB hit, this requires 4 extra memory accesses — which is why the TLB is critical.

---

Next: [04 - Segmentation](04-segmentation.md)
