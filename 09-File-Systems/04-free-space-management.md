# Free Space Management

## The Problem

When a file system needs to allocate blocks for a new file or grow an existing one, it must quickly answer: **which blocks are free?** And when a file is deleted, those blocks must be returned to the free pool. The data structure used to track free space has a direct impact on allocation speed, fragmentation, and disk overhead.

## Bit Vector (Bitmap)

The simplest and most widely used approach. One **bit per block**: 0 = free, 1 = allocated (or vice versa).

```
Block number:  0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15
Bitmap:        1  1  1  0  0  1  1  1  0  0   0  1  1  0  0  0
               |used |  |free|  |  used  |  | free  | used|free|
```

**Finding free blocks**: Scan the bitmap for 0 bits. Hardware bit-manipulation instructions (`ffs` -- find first set/clear) make this fast.

**Finding N contiguous free blocks**: Scan for a run of N consecutive 0 bits. Efficient for extent-based allocation.

**Space overhead calculation:**
```
1 TB disk, 4 KB blocks:
  Total blocks = 1 TB / 4 KB = 256 million blocks
  Bitmap size  = 256 M bits = 32 MB
  Overhead     = 32 MB / 1 TB = 0.003%
```

32 MB is small enough to cache entirely in memory, making allocation decisions very fast.

**Advantages:**
- Simple to implement
- Efficient at finding contiguous free regions (important for extent-based FS)
- Small and cacheable
- O(1) to check if a specific block is free

**Disadvantages:**
- Must scan to find free blocks (though hardware acceleration helps)
- Bitmap itself can become a write bottleneck (many allocations update the same bitmap blocks)

**Used by**: ext2, ext3, ext4 (block group bitmaps), NTFS

## Linked List of Free Blocks

Chain all free blocks together using pointers stored in the free blocks themselves:

```
Free list head --> Block 3 --> Block 4 --> Block 8 --> Block 9 --> Block 10 --> NULL
                   [next:4]    [next:8]    [next:9]    [next:10]   [next:NULL]
```

**Advantages:**
- No extra space needed (pointers stored in the free blocks themselves)
- Allocating one block is O(1) -- take the head of the list

**Disadvantages:**
- Finding contiguous free blocks is expensive (must traverse list)
- To allocate N blocks, must do N list removals
- One disk I/O per traversal step (unless blocks are cached)
- Poor locality of the list itself

**Rarely used in practice** due to poor performance for contiguous allocation.

## Grouping

Store **N free block addresses** in the first free block. The last address in that block points to another block containing N more free addresses, and so on:

```
Block 3 (first group):          Block 15 (second group):
+------------------------+      +------------------------+
| free: 4, 8, 9, 10, 13 |      | free: 20, 21, 25, 30  |
| next group: block 15 --+----->| next group: block 40   |
+------------------------+      +------------------------+
```

This is essentially the linked list approach but with **batching** -- you get N free block addresses per disk read instead of 1.

**Advantages:**
- Much faster than simple linked list
- Allocating a batch of blocks is efficient

**Disadvantages:**
- Still not great for finding contiguous regions
- More complex bookkeeping

## Counting (Run-Length Encoding)

Instead of listing every free block individually, store **(start block, count)** pairs:

```
Free space table:
+-------+-------+
| Start | Count |
+-------+-------+
|   3   |   2   |   --> blocks 3, 4
|   8   |   3   |   --> blocks 8, 9, 10
|  13   |   1   |   --> block 13
|  20   |   2   |   --> blocks 20, 21
+-------+-------+
```

This is very efficient when free blocks tend to cluster together (common after sequential allocation and deletion patterns).

**Advantages:**
- Compact when free space is mostly contiguous
- Natural representation for extent-based allocation
- Easy to find contiguous runs

**Disadvantages:**
- Can be large if free space is highly fragmented (worst case: one entry per free block)
- Merging adjacent entries on deallocation requires bookkeeping

## Extent-Based Free Space Tracking

Modern file systems combine counting with tree-based data structures for O(log n) lookup:

```
                    B+ Tree of Free Extents
                    
                         [root]
                        /      \
              [start<100]      [start>=100]
               /     \            /      \
          (3,2)  (8,3)      (108,50)  (200,100)
          
    Each leaf: (start_block, length)
    Sorted by start block for fast merging of adjacent extents
```

**Advantages:**
- O(log n) allocation and deallocation
- Efficient at finding contiguous regions of any size
- Handles both fragmented and contiguous free space well
- Can also search by size (secondary index) to find "best fit" quickly

## What Modern File Systems Actually Use

| File System | Free Space Tracking | Details |
|-------------|-------------------|---------|
| ext4 | Block group bitmaps + extents | Each block group (128MB) has its own bitmap. Allocation uses extents (contiguous runs). Multi-block allocator tries to allocate contiguous chunks. |
| XFS | B+ trees | Two B+ trees per allocation group: one sorted by start block, one by size. Enables fast best-fit allocation. |
| Btrfs | Extent tree (B-tree) | Copy-on-write B-tree tracks both allocated and free extents. Block groups organize space by type (data, metadata, system). |
| ZFS | Space maps | Each metaslab has a space map -- a log of allocation/free operations. Loaded into an in-memory AVL tree for fast allocation. |
| NTFS | Bitmap | $Bitmap file tracks cluster allocation. Simple bitmap similar to ext4. |
| FAT32 | FAT table | The FAT itself doubles as both the allocation chain and free space tracker (entries with value 0 are free). |

### ext4's Multi-Block Allocator (mballoc)

ext4 deserves special mention because its allocator is particularly clever:

```
Disk layout:
+------------------+------------------+------------------+
| Block Group 0    | Block Group 1    | Block Group 2    |
| bitmap + inodes  | bitmap + inodes  | bitmap + inodes  |
| + data blocks    | + data blocks    | + data blocks    |
+------------------+------------------+------------------+

Each block group: ~128 MB
Each has: block bitmap, inode bitmap, inode table, data blocks
```

The multi-block allocator:
1. Tries to allocate in the same block group as the file's inode (locality)
2. Pre-allocates extra blocks (speculative, for files that will likely grow)
3. Uses buddy allocator logic to find contiguous regions efficiently
4. Delayed allocation: waits until data is flushed to disk before choosing blocks (better decisions with more information)

## Real-World Connection

- **`df` vs `du` discrepancy**: `df` reads the superblock's free block count (derived from bitmaps). `du` walks the directory tree. They can disagree when deleted files are still held open or when reserved blocks exist (ext4 reserves 5% for root by default).
- **Running out of space with `df` showing free space**: ext4's reserved blocks (tunable with `tune2fs -m`) mean non-root users hit "no space" before the disk is truly full. This reserves space so root can still log in and fix things.
- **SSD TRIM**: When blocks are freed, the FS can issue TRIM commands to the SSD, telling it those blocks are no longer in use. This helps the SSD's internal garbage collector. Controlled by the `discard` mount option or the `fstrim` command.
- **Fragmentation tools**: `e4defrag` (ext4), `xfs_fsr` (XFS) use free space information to defragment files. Less critical on SSDs but still relevant for reducing extent counts.

## Interview Angle

**Q: How does a file system track free disk blocks? Compare the main approaches.**

A: The most common approach is a bitmap (one bit per block) -- simple, compact (32MB for a 1TB disk), and good at finding contiguous regions. Linked lists of free blocks use no extra space but are slow for contiguous allocation. Modern FS use extent-based tracking with B+ trees (XFS) or hybrid approaches (ext4 uses per-block-group bitmaps with a multi-block allocator). The choice depends on the workload: bitmaps work well for general use, B+ trees excel for large volumes with heavy allocation/deallocation.

**Q: What is delayed allocation, and why does it improve performance?**

A: With delayed allocation (used by ext4 and XFS), the FS does not choose which disk blocks to use when `write()` is called. Instead, data sits in the page cache, and blocks are allocated only when the data is actually flushed to disk. This has two benefits: (1) the allocator sees the full size of the write and can find a contiguous region rather than allocating block-by-block, and (2) short-lived temporary files that are written and deleted before a flush never need block allocation at all.

**Q: A disk has plenty of free space according to `df`, but a user cannot create files. What happened?**

A: Most likely the filesystem ran out of inodes (check with `df -i`). On ext4, the number of inodes is fixed at format time. A workload that creates millions of tiny files (like a mail server) can exhaust inodes while barely using disk space. Another possibility: the ext4 reserved block percentage (default 5%) is preventing non-root users from allocating, even though root could still write.

---

Next: [Inodes and the Unix File System](05-inodes-and-unix-file-system.md)
