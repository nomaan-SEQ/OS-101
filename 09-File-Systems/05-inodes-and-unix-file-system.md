# Inodes and the Unix File System

## The Inode: Central Data Structure of Unix

The **inode** (index node) is the on-disk data structure that stores everything the kernel needs to know about a file -- except its name. Every file, directory, symlink, and special file has exactly one inode.

Think of an inode as a file's "identity card": it carries all metadata and the map to find the file's data on disk.

## What Is Inside an Inode?

```
+------------------------------------------+
|              Inode (256 bytes on ext4)    |
+------------------------------------------+
| File type      | regular, dir, symlink...|
| Permissions    | rwxr-xr-- (octal 754)   |
| Owner UID      | 1000                     |
| Group GID      | 1000                     |
| Size           | 16384 bytes              |
| Link count     | 2                        |
| Timestamps:                               |
|   atime        | last access              |
|   mtime        | last content modify      |
|   ctime        | last inode change        |
|   crtime       | creation (ext4 only)     |
| Flags          | immutable, append-only...|
| Extended attrs | SELinux context, ACLs    |
+------------------------------------------+
| Block Pointers (the data map):           |
|   12 direct pointers                     |
|   1 single-indirect pointer              |
|   1 double-indirect pointer              |
|   1 triple-indirect pointer              |
+------------------------------------------+
```

## What Is NOT in an Inode?

**The filename.** The name is stored in the parent directory's data (as a directory entry mapping name to inode number). This is why:
- A file can have multiple names (hard links)
- Renaming a file does not touch the inode at all
- You can delete a file by name while another name still points to it

## Block Pointers: How the Inode Finds Data

The inode uses a **multi-level indexing scheme** to map logical file offsets to physical disk blocks.

> **Analogy:** Think of it like a **treasure hunt with clues**. For a small treasure (small file), the map gives you the location directly (direct pointers). For a bigger treasure, the map sends you to an intermediate clue sheet that lists more locations (single indirect). For a truly massive treasure, the map leads to a clue sheet that leads to more clue sheets (double/triple indirect). Small treasures are found instantly; larger ones take more steps.

```
Inode
+------------------+
| direct[0]  ------+--> Data Block 0        (bytes 0-4095)
| direct[1]  ------+--> Data Block 1        (bytes 4096-8191)
| direct[2]  ------+--> Data Block 2        (bytes 8192-12287)
| ...              |
| direct[11] ------+--> Data Block 11       (bytes 45056-49151)
|                  |
| single-indirect -+--> Index Block
|                  |    +----------+
|                  |    | ptr[0] --+--> Data Block 12
|                  |    | ptr[1] --+--> Data Block 13
|                  |    | ...      |
|                  |    | ptr[1023]+--> Data Block 1035
|                  |    +----------+
|                  |
| double-indirect -+--> Index Block (level 1)
|                  |    +----------+
|                  |    | ptr[0] --+--> Index Block (level 2)
|                  |    |          |    +----------+
|                  |    |          |    | ptr[0] --+--> Data Block
|                  |    |          |    | ptr[1] --+--> Data Block
|                  |    |          |    | ...      |
|                  |    |          |    +----------+
|                  |    | ptr[1] --+--> Index Block (level 2) ...
|                  |    | ...      |
|                  |    +----------+
|                  |
| triple-indirect -+--> (three levels of index blocks)
+------------------+
```

### Maximum File Size Calculation (4KB blocks, 4-byte pointers)

Each 4KB block holds 4096 / 4 = **1024 pointers**.

| Level | Blocks Addressable | Data Size |
|-------|-------------------|-----------|
| 12 direct | 12 | 48 KB |
| 1 single-indirect | 1024 | 4 MB |
| 1 double-indirect | 1024 x 1024 = 1,048,576 | 4 GB |
| 1 triple-indirect | 1024 x 1024 x 1024 = 1,073,741,824 | 4 TB |
| **Total** | **~1,074,791,436** | **~4 TB** |

In practice, ext4 uses 48-bit block addresses (not 32-bit), and extent-based addressing rather than traditional block pointers, supporting files up to 16 TB.

### Why This Hybrid Works Well

The design optimizes for the common case:
- **Most files are small**: 12 direct pointers handle files up to 48KB with zero indirection overhead
- **Medium files**: single-indirect handles up to ~4MB with one extra block read
- **Large files are rare** but supported: double and triple indirection scale to terabytes

Reading byte N of a file:
1. Calculate block number: `block = N / 4096`
2. If block < 12: use direct pointer (1 disk read for the data)
3. If block < 12 + 1024: use single-indirect (2 disk reads: index + data)
4. If block < 12 + 1024 + 1024^2: use double-indirect (3 disk reads)
5. Else: triple-indirect (4 disk reads)

## The Inode Table

Inodes are stored in a **fixed-size table** on disk, allocated when the filesystem is formatted (`mkfs`):

```
Disk layout (ext4):
+-------+-------+--------+-----------+-----------+
| Super | Group | Block  | Inode     | Data      |
| block | Desc  | Bitmap | Table     | Blocks    |
+-------+-------+--------+-----------+-----------+
                          |           |
                          | inode 0   |
                          | inode 1   |
                          | inode 2 (root dir)
                          | inode 3   |
                          | ...       |
                          | inode N   |
```

**Key facts:**
- Inode numbers are array indices (inode 42 = the 42nd entry in the table)
- Inode 2 is always the root directory (`/`)
- The total number of inodes is fixed at format time (ext4 default: 1 inode per 16KB of disk)
- Each inode is typically 256 bytes (ext4)

## Running Out of Inodes

A filesystem can run out of inodes before running out of data blocks. This happens with workloads that create millions of tiny files:

```bash
$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   20G   80G  20% /          # Plenty of space!

$ df -i /
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda1       6553600 6553600     0  100% /     # But ZERO free inodes!
```

**Common scenarios:**
- Mail servers storing each email as a separate file
- Build systems with millions of tiny object files
- Container images with many layers of small files

**Solutions:**
- Format with more inodes: `mkfs.ext4 -N <count>` or `-i <bytes-per-inode>`
- Use a filesystem that allocates inodes dynamically (XFS, Btrfs, ZFS)
- Clean up unnecessary files

## Reading Inode Information

The `stat` command shows inode details:

```bash
$ stat /home/user/notes.txt
  File: /home/user/notes.txt
  Size: 4096        Blocks: 8          IO Block: 4096   regular file
Device: 801h/2049d  Inode: 131074      Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/user)   Gid: ( 1000/user)
Access: 2025-01-15 10:30:00.000000000 -0800
Modify: 2025-01-14 09:15:00.000000000 -0800
Change: 2025-01-14 09:15:00.000000000 -0800
 Birth: 2025-01-10 08:00:00.000000000 -0800
```

Other useful commands:
```bash
$ ls -i file.txt           # Show inode number
131074 file.txt

$ df -i                    # Show inode usage per filesystem

$ debugfs -R "stat <131074>" /dev/sda1   # Low-level inode dump (ext4)
```

## Real-World Connection

- **Why `mv` within the same FS is instant**: `mv a.txt b.txt` (same directory) just renames the directory entry. `mv dir1/a.txt dir2/a.txt` (same FS) removes the old directory entry and creates a new one, both pointing to the same inode. No data is copied. But `mv` across filesystems must copy all data (like `cp` + `rm`).

- **Why hard links share the inode**: `ln file.txt hardlink.txt` creates a new directory entry pointing to the same inode. Both names are equally "real" -- there is no "original." The inode's link count tracks how many names point to it. The file is only deleted when the link count drops to 0 and no process has it open.

- **Deleted files consuming space**: If a process has a file open when it is deleted (`rm`), the directory entry is removed and the link count drops to 0, but the inode and data blocks persist until the process closes the file. This is common with log rotation -- a log process keeps writing to a "deleted" file. Use `lsof +L1` to find these.

- **Docker and inodes**: Container overlay filesystems create many small metadata files. Dense container hosts can run out of inodes. Monitor with `df -i`.

- **ext4 extents vs block pointers**: Modern ext4 defaults to extents (a more efficient scheme) rather than the traditional 12+1+1+1 block pointer structure. Each extent stores (logical block, physical start, length), reducing metadata for large contiguous files from thousands of pointers to a handful of extents. The traditional block pointer scheme is still used by ext2 and is the classic exam/interview model.

## Interview Angle

**Q: What is an inode? What does it store, and what does it NOT store?**

A: An inode is the on-disk data structure representing a file in Unix filesystems. It stores: file type, permissions, owner (UID/GID), size, timestamps, link count, and block pointers (the map to data on disk). It does NOT store the filename -- that is in the directory entry. This separation is what enables hard links (multiple names for one inode) and makes rename operations O(1).

**Q: Why is `mv` within the same filesystem O(1), while `cp` is O(n)?**

A: `mv` on the same filesystem only modifies directory entries (remove old name, add new name, same inode number). No data blocks are read or written regardless of file size. `cp` must read every data block and write a copy to new blocks under a new inode, making it proportional to file size. If `mv` crosses filesystem boundaries, it degrades to cp + rm because inode numbers are per-filesystem.

**Q: Calculate the maximum file size for a traditional Unix inode with 4KB blocks and 4-byte block pointers.**

A: Direct: 12 blocks = 48KB. Single-indirect: 1024 blocks = 4MB. Double-indirect: 1024 x 1024 = ~4GB. Triple-indirect: 1024^3 = ~4TB. Total: approximately 4TB + 4GB + 4MB + 48KB, roughly 4TB. The key insight is that each level of indirection multiplies capacity by 1024 (the number of pointers per block).

**Q: A server shows disk space available but cannot create new files. Diagnosis?**

A: Most likely out of inodes. Check with `df -i`. On ext4, inodes are allocated at format time and cannot be added later (without reformatting). Common with workloads creating millions of tiny files. Solutions: reformat with more inodes (`mkfs.ext4 -i bytes_per_inode`), clean up small files, or switch to XFS/Btrfs which allocate inodes dynamically.

---

Next: [Journaling and Log-Structured File Systems](06-journaling-and-log-structured-file-systems.md)
