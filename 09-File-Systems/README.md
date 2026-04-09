# File Systems

File systems organize data on storage devices into files and directories -- the logical layer that sits above raw disk blocks and below application code. They answer a deceptively complex question: how do you take a flat array of disk blocks and present it as a hierarchical, named, permission-controlled collection of files that multiple processes can safely share?

Every `open()`, `read()`, `write()`, and `close()` call you make passes through the file system layer. Understanding how file systems work internally helps you reason about performance bottlenecks, debug data corruption issues, choose the right FS for your workload, and answer a surprisingly common class of interview questions.

## Why This Matters

- **Performance**: Sequential vs random I/O patterns, extent-based allocation, and journaling modes can mean 10x throughput differences on the same hardware.
- **Data integrity**: Knowing how journaling and copy-on-write work tells you what happens when power fails mid-write -- and what guarantees your application actually has.
- **Debugging**: Understanding inodes, file descriptors, and path resolution demystifies errors like "too many open files," "no space left on device" (when `df` shows space), and broken symlinks.
- **System design**: Cloud storage, container images, and database engines all build on file system concepts. You need the mental model to evaluate trade-offs.

## Prerequisites

| Section | Why You Need It |
|---------|----------------|
| [08-Storage-Management](../08-Storage-Management/README.md) | Disk structure, I/O scheduling, block devices -- the layer beneath file systems |
| [07-Memory-Management](../07-Memory-Management/README.md) | Page cache, mmap, and how file data gets cached in RAM |

## Reading Order

| # | Topic | What You'll Learn |
|---|-------|-------------------|
| 1 | [File System Concepts and Interfaces](01-file-system-concepts-and-interfaces.md) | Files, file descriptors, the three-level lookup, file types and access methods |
| 2 | [Directory Structure and Path Resolution](02-directory-structure-and-path-resolution.md) | Directories, dentries, path resolution walk-through, hard vs soft links, mounting |
| 3 | [File Allocation Methods](03-file-allocation-methods.md) | Contiguous, linked, and indexed allocation -- how file data maps to disk blocks |
| 4 | [Free Space Management](04-free-space-management.md) | Bitmaps, linked lists, extents -- how the FS tracks available blocks |
| 5 | [Inodes and the Unix File System](05-inodes-and-unix-file-system.md) | The inode data structure, direct/indirect blocks, max file size calculation |
| 6 | [Journaling and Log-Structured File Systems](06-journaling-and-log-structured-file-systems.md) | Crash consistency, journal modes, WAL, LFS, copy-on-write |
| 7 | [VFS and FUSE](07-vfs-and-fuse.md) | Virtual File System abstraction, FUSE user-space filesystems, procfs/sysfs |
| 8 | [Modern File Systems: ext4, XFS, Btrfs, ZFS](08-modern-file-systems-ext4-xfs-btrfs-zfs.md) | Comparing production file systems, features, trade-offs, how to choose |

## Key Interview Questions

1. **Explain the three-level file descriptor structure in Unix.** How does a file descriptor in a process map to actual file data on disk? What happens when two processes open the same file?

2. **What is an inode? What does it store and what does it NOT store?** Why is `mv` within the same filesystem O(1) while `cp` is O(n)?

3. **Compare journaling modes (journal, ordered, writeback).** What trade-offs does each make between safety and performance? What does ext4 use by default and why?

4. **How does path resolution work for `/home/user/file.txt`?** Walk through every step from root inode to the final file's data blocks.

5. **When would you choose XFS over ext4? Btrfs over XFS? ZFS over all of them?** What are the practical trade-offs in production environments?
