# File Allocation Methods

## The Core Problem

A file is a logical sequence of bytes, but a disk is a flat array of fixed-size blocks (typically 4KB). The file system must decide **how to map file data to disk blocks**. This mapping strategy profoundly affects performance, fragmentation, and maximum file size.

## Contiguous Allocation

Each file occupies a **set of consecutive disk blocks**. The directory entry stores the starting block and the length.

```
Directory entry:  file.txt -> start: 5, length: 4

Disk blocks:
+---+---+---+---+---+---+---+---+---+---+---+---+
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |10 |11 |
+---+---+---+---+---+---+---+---+---+---+---+---+
                      |   file.txt    |
                      [===block 5-8===]
```

**Advantages:**
- Excellent sequential read performance (blocks are adjacent -- one seek, then stream)
- Fast random access: block i of file = start + i (O(1) calculation)
- Simple metadata: just (start, length) per file

**Disadvantages:**
- **External fragmentation**: after files are created and deleted, free space becomes scattered. A 10-block file may not fit even if 20 blocks are free (just not contiguous)
- **File size must be known at creation time** -- or use complex compaction/reallocation
- Growing a file may require moving it entirely

**Used by**: CD-ROM filesystems (ISO 9660), some embedded systems. Modern FS use extents (a refined version of this idea).

## Linked Allocation

Each block contains a pointer to the **next block** in the file. The directory entry stores the first block.

```
Directory entry: file.txt -> first block: 5

Disk blocks:
  Block 5          Block 9          Block 3          Block 7
+---------+      +---------+      +---------+      +---------+
| data    |      | data    |      | data    |      | data    |
| next: 9 |----->| next: 3 |----->| next: 7 |----->| next: -1|
+---------+      +---------+      +---------+      +---------+
```

**Advantages:**
- No external fragmentation -- any free block can be used
- Files can grow dynamically (just link in a new block)
- No need to know file size upfront

**Disadvantages:**
- **Terrible random access**: to read block N, must traverse N-1 pointers (O(n))
- Pointer space overhead: each block loses some bytes to the "next" pointer
- Reliability: a single corrupted pointer breaks the entire chain from that point on
- Sequential read performance is poor if blocks are scattered across disk

### FAT: Linked Allocation Done Better

The **File Allocation Table (FAT)** is a variant that moves all the "next" pointers out of the data blocks and into a **separate table in a known location**:

```
FAT (in memory):                 Disk blocks:
+-------+---------+             +---+---+---+---+---+---+---+---+---+---+
| Block | Next    |             | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
+-------+---------+             +---+---+---+---+---+---+---+---+---+---+
|   0   |  free   |                         |dat|       |dat|   |dat|   |dat|
|   1   |  free   |
|   2   |  free   |             file.txt chain: 5 -> 9 -> 3 -> 7 -> EOF
|   3   |    7    |
|   4   |  free   |
|   5   |    9    |  <- file.txt starts here
|   6   |  free   |
|   7   |   EOF   |
|   8   |  free   |
|   9   |    3    |
+-------+---------+
```

Because the FAT is small enough to cache entirely in RAM, random access becomes fast -- just walk the chain in memory instead of doing disk I/O for each block.

**Used by**: FAT12, FAT16, FAT32 (USB drives, SD cards, EFI system partitions). FAT32 is still the most universally compatible filesystem.

## Indexed Allocation

An **index block** (also called an "inode block" in this context) contains pointers to all of the file's data blocks:

```
Directory entry: file.txt -> index block: 20

Index block 20:              Data blocks:
+------+                     
| ptr0 | --> block 5         Block 5:  [data...]
| ptr1 | --> block 9         Block 9:  [data...]
| ptr2 | --> block 3         Block 3:  [data...]
| ptr3 | --> block 7         Block 7:  [data...]
| ptr4 | --> NULL            
| ...  |                     
+------+                     
```

**Advantages:**
- Fast random access: block i of file = index_block[i] -- O(1) with one extra read
- No external fragmentation -- blocks can be anywhere
- Files can grow (add entries to index block)

**Disadvantages:**
- Overhead for small files (need a whole index block even for a 1-block file)
- **Index block size limits file size**: a 4KB block with 4-byte pointers holds 1024 entries = max 4MB file
- Wasted space if file is small

### Multi-Level Index

To handle large files, use **indirect index blocks**:

```
Index block:
+---------+
| ptr0    | --> data block         (direct)
| ptr1    | --> data block         (direct)
| ...     |
| ptr_ind | --> index block 2     (single indirect)
+---------+      +---------+
                 | ptr0    | --> data block
                 | ptr1    | --> data block
                 | ...     |
                 | ptr_ind | --> index block 3  (double indirect)
                 +---------+      +---------+
                                  | ptr0    | --> data block
                                  | ...     |
                                  +---------+
```

This is exactly the approach Unix/ext filesystems use (covered in detail in the inodes section).

## The Unix/ext Approach: Combined Scheme

Unix filesystems use a **hybrid** of direct and multi-level indexed allocation within the inode:

```
Inode:
+------------------+
| 12 direct ptrs   | --> 12 data blocks (48KB with 4KB blocks)
| 1 single-indirect| --> 1 index block --> 1024 data blocks
| 1 double-indirect| --> 1 index block --> 1024 index blocks --> 1M data blocks
| 1 triple-indirect| --> ... (for very large files)
+------------------+
```

This optimizes for the common case: most files are small and only need the 12 direct pointers. Large files pay the extra indirection cost, but they are rare.

## Comparison Table

| Feature | Contiguous | Linked | FAT (Linked variant) | Indexed |
|---------|-----------|--------|---------------------|---------|
| Sequential read | Excellent | Poor (scattered) | Moderate | Good |
| Random access | O(1) | O(n) | O(n) in memory (fast) | O(1) + indirection |
| Fragmentation | External | None | None | None |
| File growth | Difficult | Easy | Easy | Easy (until index full) |
| Max file size | Limited by contiguous space | Unlimited | Limited by FAT size | Limited by index depth |
| Space overhead | None | Pointer per block | Separate FAT table | Index block(s) |
| Reliability | Good | Single point of failure | FAT can be duplicated | Index block is critical |
| Used in | CD-ROMs, embedded | Historical | USB, SD cards, EFI | Unix, ext2/3/4, NTFS |

## Real-World Connection

- **ext4 extents**: Modern ext4 does not use the traditional block pointer approach for new files. Instead, it uses **extents** -- (start block, length) pairs -- which is essentially a refined contiguous allocation. A single extent can describe up to 128MB of contiguous data, dramatically reducing metadata overhead for large sequential files.
- **NTFS**: Uses a similar concept called "runs" -- contiguous block ranges stored in the Master File Table (MFT).
- **SSD considerations**: On SSDs, the distinction between sequential and random block placement matters less for read performance (no seek time), but sequential allocation still helps with write amplification and garbage collection.
- **Database files**: Databases (PostgreSQL, MySQL) allocate large contiguous files and manage their own block layout internally. The FS allocation method still matters for the initial allocation and growth patterns.

## Interview Angle

**Q: Compare contiguous, linked, and indexed file allocation. When would you use each?**

A: Contiguous gives the best read performance but suffers from external fragmentation and requires knowing file size upfront -- good for read-only media like CD-ROMs. Linked allocation eliminates fragmentation and allows easy growth, but random access is O(n) because you must traverse the chain. Indexed allocation provides O(1) random access via an index block but wastes space for small files. Modern systems use a hybrid: ext4 uses direct pointers (like indexed) for small files and extents (like contiguous) for large files.

**Q: What is the FAT, and why is it still relevant?**

A: The File Allocation Table is a linked allocation scheme where all block pointers are stored in a separate table rather than in each data block. It remains relevant because FAT32 is the most universally compatible filesystem -- every OS can read it. It is the standard for USB drives, SD cards, and EFI system partitions. Its simplicity is its strength, though it lacks journaling, permissions, and has a 4GB max file size limit (FAT32).

**Q: What are extents, and why are they better than individual block pointers?**

A: An extent is a (start block, length) pair that describes a contiguous run of blocks. Instead of storing one pointer per block (12 bytes per 4KB block for a 1GB file = 3MB of pointers), a single extent can describe the entire 1GB in just 12 bytes if the blocks are contiguous. This dramatically reduces metadata overhead and improves performance. ext4, XFS, Btrfs, and NTFS all use extent-based allocation.

---

Next: [Free Space Management](04-free-space-management.md)
