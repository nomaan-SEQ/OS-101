# Journaling and Log-Structured File Systems

## The Crash Consistency Problem

File system operations often require **multiple disk writes** that must all succeed for the FS to remain consistent. Consider creating a new file:

1. Allocate an inode (write inode bitmap)
2. Initialize the inode (write inode table)
3. Add directory entry (write directory data block)
4. Update parent directory's mtime (write parent inode)
5. Mark data blocks as used (write block bitmap)

If the system crashes between steps 2 and 3, you have an allocated inode that no directory entry points to -- an orphan inode that wastes space forever. Other crash scenarios can be worse: a directory entry pointing to an uninitialized inode, or data blocks marked as both free and in use.

```
CRASH SCENARIOS:

Write 1 done, Write 2 pending:
  +- Inode bitmap says "allocated"
  +- Inode data is garbage         <-- INCONSISTENT!
  +- No directory entry

Write 1+2 done, Write 3 pending:
  +- Inode allocated and valid
  +- No directory entry            <-- Orphan inode (space leak)

Write 3 done, Write 1+2 pending:
  +- Directory entry points to uninitialized inode  <-- DANGEROUS!
```

## The Old Way: fsck

Before journaling, the solution was **fsck** (file system check): on boot after a crash, scan the entire filesystem to find and repair inconsistencies.

**Problems with fsck:**
- Scans every inode, every block, every directory -- takes minutes to hours on large disks
- System is unavailable during the check
- Some inconsistencies cannot be resolved automatically
- On a multi-terabyte filesystem, fsck can take longer than the typical interval between crashes

## Journaling: Write-Ahead Logging for File Systems

The solution: before making changes to the actual filesystem structures, write a **log entry** (called a **journal entry** or **transaction**) describing what you are about to do.

```
Normal write flow (without journal):

   Application write
         |
         v
   Update data blocks
   Update inode
   Update bitmap
         |
   [CRASH HERE = inconsistent FS]


Journaled write flow:

   Application write
         |
         v
   1. Write to JOURNAL:          <-- "I'm about to do these changes"
      "Update inode 42, block 100,
       bitmap entry 100"
   2. COMMIT record in journal   <-- "All changes are logged"
         |
         v
   3. Write actual changes       <-- "Checkpoint" the journal
      to filesystem structures
         |
         v
   4. Mark journal entry         <-- "Done, can reuse this
      as complete                     journal space"
```

**Recovery after crash:**
- Crash during step 1 (incomplete journal write): discard the partial entry. Filesystem is unchanged.
- Crash during step 3 (journal complete, checkpoint incomplete): **replay the journal** -- apply all logged changes. This is safe because the journal has the complete set of changes.
- Crash during step 4: replay detects the transaction is already applied. No harm done.

Recovery takes seconds (just replay the journal) instead of minutes/hours (full fsck scan).

## Journal Modes

Not all data needs to be journaled. The three modes trade safety for performance:

### Full Journal Mode (`data=journal`)

```
Journal contains: metadata changes AND file data

Write "hello" to file:
  Journal: [inode update] [block bitmap] [data: "hello"]
  Then: checkpoint all to filesystem
```

- **Safest**: after a crash, both metadata and data are consistent
- **Slowest**: every byte of data is written twice (journal + final location)
- **Use case**: critical systems where no data loss is acceptable

### Ordered Mode (`data=ordered`) -- ext4 Default

```
Write order guaranteed:
  1. Write file DATA to final location on disk
  2. Write METADATA changes to journal
  3. Checkpoint metadata to filesystem

Key: data is written BEFORE the metadata that references it
```

- **Good balance**: metadata is journaled, data is not -- but ordering guarantees prevent stale data exposure
- **After crash**: metadata is consistent (replayed from journal), and any data it references was already written
- **Cannot expose old/garbage data** in place of new data
- **Default for ext4** because it is safe enough for most workloads with reasonable performance

### Writeback Mode (`data=writeback`)

```
No ordering guarantee:
  - Metadata changes journaled
  - Data written whenever (before or after metadata)
```

- **Fastest**: no ordering constraints on data writes
- **Risk**: after a crash, a newly allocated file might contain stale data from a previously deleted file (the metadata says "file exists" but the data blocks were not yet overwritten)
- **Use case**: performance-critical systems where stale data exposure is acceptable (e.g., scratch/temp storage)

### Mode Comparison

| Feature | journal | ordered | writeback |
|---------|---------|---------|-----------|
| Metadata safety | Yes | Yes | Yes |
| Data safety | Yes | Ordering guarantee | No |
| Performance | Slowest (2x write) | Moderate | Fastest |
| Stale data risk | None | None | Yes |
| Use case | Critical data | General purpose | Performance-first |
| ext4 mount option | `data=journal` | `data=ordered` | `data=writeback` |

## Write-Ahead Logging (WAL)

Journaling in file systems is an application of the general **Write-Ahead Logging** principle used throughout computer science:

> Before modifying any data structure, write the intended change to a sequential log. Only then apply the change.

The same technique appears in:
- **Databases**: PostgreSQL WAL, MySQL redo log, SQLite WAL mode
- **Distributed systems**: Raft/Paxos commit logs
- **Key-value stores**: LSM trees (LevelDB, RocksDB) write to a WAL before memtable

The principle is universal because sequential writes are fast and atomic (a log entry either fully exists or does not), making them ideal for crash recovery.

## Log-Structured File Systems (LFS)

LFS takes journaling to its logical extreme: **the entire filesystem IS a log**. All writes -- data and metadata -- are appended sequentially.

```
Traditional FS:                    Log-Structured FS:
                                   
Writes go to fixed locations:      ALL writes are sequential appends:
                                   
  inode table  [update in place]   +--+--+--+--+--+--+--+--+---->
  data blocks  [update in place]   |d1|i1|d2|d3|i2|d4|d5|i3| ...
  bitmap       [update in place]   +--+--+--+--+--+--+--+--+---->
                                   ^                            ^
                                   old data                     new data
                                   (may be stale)               (current)
```

**Advantages:**
- All writes are sequential -- ideal for flash storage (SSDs)
- Very fast writes (no seeking)
- Automatic versioning (old data remains until overwritten)

**Disadvantages:**
- **Garbage collection**: overwritten data (old versions) creates "holes" in the log that must be reclaimed
- Random reads require an inode map to find current locations
- GC overhead can be significant under heavy random-write workloads

**Implementations**: Sprite LFS (research), F2FS (Flash-Friendly FS, used on Android), NILFS2 (Linux)

## Copy-on-Write (COW)

An alternative to journaling that **never overwrites existing data**:

```
Write to block 5:

Traditional (in-place update):
  Block 5: [old data] --> [new data]   (overwritten)

COW:
  Block 5:  [old data]  (unchanged)
  Block 99: [new data]  (written to new location)
  Update parent pointer: block 5 -> block 99
  Update grandparent... (propagates up the tree)
```

```
Before write:                After COW write:
                             
    Root                         Root'  (new)
   /    \                       /    \
  A      B                    A      B' (new)
 / \    / \                  / \    / \
D   E  F   G               D   E  F   G' (new data)

Old tree still exists!      Only the path from root
(snapshot = keep old root)  to changed leaf is rewritten
```

**Advantages:**
- **Atomic updates**: the old tree is valid until the new root pointer is committed
- **Free snapshots**: just keep the old root pointer
- **No journal needed**: consistency comes from never overwriting
- **Data integrity**: checksums can be verified at any time

**Disadvantages:**
- **Write amplification**: changing one block requires rewriting every block up to the root
- **Fragmentation**: sequential files become fragmented after updates (new blocks allocated elsewhere)
- More complex space management

**Implementations**: Btrfs, ZFS, WAFL (NetApp)

## Real-World Connection

- **ext4 default**: uses ordered journaling. When you hear "ext4 is journaled," it means metadata is journaled with ordered data writes. You can change the mode at mount time: `mount -o data=journal /dev/sda1 /mnt`.
- **Database double-write**: databases like MySQL/InnoDB have their own WAL *on top of* the filesystem's journal. This is because the FS journal only guarantees FS-level consistency, not application-level consistency.
- **ZFS and Btrfs use COW**: neither uses a traditional journal. Instead, atomic COW updates provide crash consistency. ZFS additionally checksums every block, catching silent data corruption (bit rot).
- **F2FS on Android**: many Android devices use F2FS (a log-structured FS) because it is optimized for flash storage -- sequential writes reduce write amplification and improve SSD longevity.
- **Snapshots for backups**: Btrfs/ZFS snapshots are a COW feature -- creating a snapshot is instant (just save the current root pointer). This enables consistent backup of a live filesystem without unmounting.

## Interview Angle

**Q: What is the crash consistency problem, and how does journaling solve it?**

A: File system operations require multiple disk writes (inode, directory, bitmap). A crash between writes leaves the FS in an inconsistent state. Without journaling, `fsck` must scan the entire disk to find and fix inconsistencies -- this is slow and the system is unavailable. Journaling solves this by writing a complete description of the pending changes to a log before applying them. After a crash, recovery just replays the journal (seconds) instead of scanning the whole disk (minutes to hours).

**Q: Compare ext4's journaling modes: journal, ordered, and writeback.**

A: All three modes journal metadata. The difference is how they handle file data. "Journal" mode writes data to the journal too -- safest but slowest (2x writes). "Ordered" mode (ext4 default) guarantees data is written to its final location before the metadata journal entry is committed -- this prevents stale data exposure without the overhead of journaling data. "Writeback" mode journals only metadata with no data ordering -- fastest, but after a crash a new file might contain stale data from a previously deleted file.

**Q: What is copy-on-write, and how does it differ from journaling?**

A: Journaling writes changes to a log, then applies them in place. COW never overwrites existing data -- it writes modified data to a new location and atomically updates the pointer. The old data remains valid until the new pointer is committed, providing crash consistency without a journal. COW enables free snapshots (keep old pointers) and data integrity verification. The trade-off is write amplification (pointer updates propagate up the tree) and potential fragmentation. Btrfs and ZFS use COW; ext4 uses journaling.

---

Next: [VFS and FUSE](07-vfs-and-fuse.md)
