# Segmentation

Paging divides memory into fixed-size blocks without regard for what's in them — code, data, and stack all get chopped into uniform 4 KB pages. Segmentation takes a different approach: divide the address space by **logical meaning**. Code goes in one segment, the stack in another, the heap in another. This matches how programmers actually think about memory and enables per-section protection (code is read+execute, data is read+write).

## The Concept

A segmented address space consists of a variable number of segments, each with its own base address and length:

```
  Process's Logical View:

  Segment 0: Code       [0x0000 - 0x1FFF]   8 KB, read + execute
  Segment 1: Data       [0x0000 - 0x0FFF]   4 KB, read + write
  Segment 2: Stack      [0x0000 - 0x07FF]   2 KB, read + write
  Segment 3: Heap       [0x0000 - 0x2FFF]  12 KB, read + write
  Segment 4: Shared Lib [0x0000 - 0x3FFF]  16 KB, read + execute
```

A virtual address in a segmented system consists of two parts: **(segment number, offset)**. The CPU uses the segment number to look up the segment's base and limit in the segment table, then adds the offset to the base to get the physical address.

## Address Translation

```
  Virtual Address:
  +------------------+------------------+
  | Segment Number   |     Offset       |
  +------------------+------------------+
         |                    |
         v                    |
  +---------+--------+-------+
  | Segment | Base   | Limit |     Segment Table
  +---------+--------+-------+
  |    0    | 0x4000 | 8192  |
  |    1    | 0x8000 | 4096  |
  |    2    | 0xA000 | 2048  |
  |    3    | 0xC000 | 12288 |
  +---------+--------+-------+
         |
         v
   base = 0x4000 (for segment 0)
   
   if (offset >= limit)
       TRAP: segmentation fault
   else
       physical_address = base + offset
```

### Worked Example

Virtual address: (segment=1, offset=0x0200)

1. Look up segment 1 in the segment table: base=0x8000, limit=4096
2. Check: 0x0200 (512) < 4096? Yes, valid.
3. Physical address = 0x8000 + 0x0200 = **0x8200**

If the offset were 0x2000 (8192), it would exceed the limit (4096), triggering a segmentation fault — this is where the name "segfault" originally comes from.

## Segment Table

Each process has its own segment table, managed by the OS and loaded into hardware registers (or a special register pointing to the table in memory) on context switch.

```
  +---------+--------+--------+-----+-----+-----+
  | Segment | Base   | Limit  | R   | W   | X   |
  +---------+--------+--------+-----+-----+-----+
  |    0    | 0x4000 |  8192  | yes | no  | yes |  <- Code
  |    1    | 0x8000 |  4096  | yes | yes | no  |  <- Data
  |    2    | 0xA000 |  2048  | yes | yes | no  |  <- Stack
  |    3    | 0xC000 | 12288  | yes | yes | no  |  <- Heap
  |    4    | 0x2000 | 16384  | yes | no  | yes |  <- Shared lib
  +---------+--------+--------+-----+-----+-----+
```

## Advantages of Segmentation

**1. Logical organization.** Segments correspond to natural program structures. The compiler and OS know that "this is code" and "this is stack" — enabling meaningful protection.

**2. Per-segment protection.** Code segments can be marked read+execute (no writes), data as read+write (no execute). This is a natural fit for security policies like W^X (write XOR execute) — memory should never be both writable and executable.

**3. Sharing is natural.** Two processes can share a code segment (e.g., a shared library) by pointing their segment table entries to the same physical base. Each process has its own data and stack segments, but the shared code exists only once in physical memory.

```
  Process A's table:           Process B's table:
  Seg 0 (code) -> 0x4000      Seg 0 (code) -> 0x4000   <- SAME physical base!
  Seg 1 (data) -> 0x8000      Seg 1 (data) -> 0xD000   <- different
  Seg 2 (stack)-> 0xA000      Seg 2 (stack)-> 0xF000   <- different
```

**4. Segment growth.** Segments can grow independently. The heap segment can expand without affecting the code segment. (Though this only works if there's free space adjacent to the segment's end.)

## Disadvantages of Segmentation

**1. External fragmentation.** This is the dealbreaker. Segments are variable-sized, so we're back to the same problem as contiguous allocation. As segments are allocated and freed, memory becomes a patchwork of holes.

```
  Physical Memory (fragmented):
  +------------------+
  |   OS             |
  +------------------+
  | Proc A Code (8K) |
  +------------------+
  |   FREE (3K)      |   <- too small for most segments
  +------------------+
  | Proc B Data (4K) |
  +------------------+
  |   FREE (6K)      |
  +------------------+
  | Proc A Stack (2K)|
  +------------------+
  |   FREE (1K)      |   <- useless
  +------------------+
```

**2. Complex management.** The OS must track variable-size free regions, handle allocation strategies (First Fit, Best Fit), and potentially compact memory.

**3. Maximum segment size.** A segment can't be larger than the largest contiguous free region, unless compaction is performed.

## Segmentation with Paging

The obvious thought: combine both. Use segmentation for logical structure and protection, and paging within each segment to eliminate external fragmentation.

Intel's x86 architecture (32-bit) actually did this:

```
  Logical Address
  +----------+----------+
  | Segment  |  Offset  |
  +----------+----------+
       |           |
       v           v
  Segment Table   (offset becomes a virtual address)
       |           |
       v           v
  +----------+----------+----------+
  | Linear Address                 |
  +----------+----------+----------+
  | Dir (10) | Table(10)| Off (12) |
  +----------+----------+----------+
       |           |          |
       v           v          v
      Page Table Lookup
       |
       v
  Physical Address
```

The segment table produces a **linear address**, which is then translated through the page table to a physical address. This gave the best of both worlds: logical structure from segmentation, no external fragmentation from paging.

## x86-64: Segmentation Mostly Gone

Intel's 64-bit mode (long mode) effectively disables segmentation:

- The CS, DS, SS, ES segment registers have their base forced to 0 and limit to the full address space
- This creates a **flat memory model** — the entire virtual address space is one segment
- FS and GS segments still work (Linux uses them for thread-local storage, Windows for the TEB)
- Paging does all the heavy lifting for protection and address translation

Why? Segmentation added complexity with minimal benefit once paging could handle protection (via page-level read/write/execute bits). Maintaining two translation steps was unnecessary overhead.

## Paging vs Segmentation: Comparison

| Feature | Paging | Segmentation |
|---------|--------|-------------|
| Unit size | Fixed (4 KB typically) | Variable (any size) |
| Address structure | Page number + offset | Segment number + offset |
| External fragmentation | None | Yes (variable sizes) |
| Internal fragmentation | Small (last page) | None |
| Logical structure | No — arbitrary page boundaries | Yes — matches program structure |
| Sharing | Possible but at page granularity | Natural — share a segment |
| Protection | Per-page bits | Per-segment (more meaningful) |
| Implementation complexity | Moderate (multi-level tables) | Moderate (segment tables + free lists) |
| Modern usage | Universal | Mostly abandoned |
| Hardware support | All modern CPUs | x86-64 has vestigial support only |

## Why Paging Won

1. **No external fragmentation.** This alone is nearly sufficient reason. Variable-size allocation is fundamentally harder to manage than fixed-size.

2. **Simpler for the OS.** Tracking free frames is easy (bitmap or free list of identical-size units). Tracking variable-size free regions requires complex algorithms.

3. **Virtual memory works naturally with paging.** Demand paging, page replacement, and swap all operate on fixed-size units. Swapping variable-size segments is much messier.

4. **Page-level protection is good enough.** With NX (No Execute) bits in page table entries, paging can enforce read/write/execute protection at 4 KB granularity. That's fine for most purposes.

5. **Hardware simplicity.** One translation mechanism (page tables + TLB) is simpler than two (segment table + page table).

Modern OSes use a flat segmentation model (effectively no segmentation) with paging for all memory management.

## Real-World Connection

- **The name "segmentation fault"**: even though modern systems primarily use paging, the term persists. A "segfault" now typically means accessing an unmapped or protected virtual page, not an actual segment violation. The name is a historical artifact from when segmentation was the primary protection mechanism.
- **Thread-local storage (TLS)**: Linux uses the FS register (a segment register) to point to thread-local data. Each thread has FS set to a different base address, so accessing `FS:0x10` gives each thread its own copy of a variable. This is one of the last active uses of segmentation on x86-64.
- **Intel MPX and SGX**: Intel's Memory Protection Extensions and Software Guard Extensions used segment-like concepts for fine-grained protection, though MPX has since been deprecated.

## Interview Angle

**Q: What is segmentation and how does it differ from paging?**

Segmentation divides a process's address space into logical sections (code, data, stack, heap) of variable size. Paging divides it into fixed-size pages regardless of content. Segmentation preserves logical structure and enables natural sharing (share the code segment), but suffers external fragmentation due to variable sizes. Paging eliminates external fragmentation, simplifies memory management, and works naturally with virtual memory, which is why all modern systems use paging over segmentation.

**Q: Why don't modern x86-64 systems use segmentation?**

x86-64's long mode forces most segment bases to 0 with unlimited range, creating a flat memory model. Segmentation's benefits (logical structure, per-section protection) can be achieved with page-level protections. The downsides (external fragmentation, complex two-step translation) aren't worth the cost. The only remaining use of segment registers is FS/GS for thread-local storage.

**Q: Can you combine segmentation and paging?**

Yes — 32-bit x86 did exactly this. A logical address was first translated through the segment table to a linear address, then through the page table to a physical address. This gave logical structure from segmentation and no external fragmentation from paging. But the complexity wasn't justified — paging alone proved sufficient, and x86-64 dropped segmentation for this reason.

---

Next: [05 - Virtual Memory and Demand Paging](05-virtual-memory-and-demand-paging.md)
