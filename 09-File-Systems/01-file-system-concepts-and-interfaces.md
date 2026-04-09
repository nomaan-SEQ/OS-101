# File System Concepts and Interfaces

## What Is a File?

A **file** is a named collection of related data stored on secondary storage (disk, SSD). It is the smallest unit of logical storage that users and applications interact with. The file system's job is to map this logical abstraction to physical blocks on a storage device.

Without files, every program would need to know the exact disk addresses of its data. Files give us:
- **Naming**: humans deal with names, not block numbers
- **Persistence**: data survives process termination and reboots
- **Sharing**: multiple processes can access the same data
- **Protection**: access control through permissions

## File Attributes (Metadata)

Every file carries metadata alongside its actual data:

| Attribute | Description | Linux Example |
|-----------|-------------|---------------|
| Name | Human-readable identifier | `report.pdf` |
| Type | Regular file, directory, symlink, etc. | `-` (regular), `d` (directory) |
| Size | Bytes of data content | `4096` |
| Permissions | Read/write/execute for owner/group/other | `rwxr-xr--` (754) |
| Owner | User ID (UID) and group ID (GID) | `uid=1000, gid=1000` |
| Timestamps | Created, modified, accessed | `ctime`, `mtime`, `atime` |
| Link count | Number of directory entries (hard links) pointing to this file's inode. When the link count drops to 0 and no process has the file open, the OS frees the file's data blocks. | `1` |
| Block pointers | Addresses of data blocks on disk | (stored in inode) |

**Timestamp details on Linux:**
- `mtime` -- last time file **content** was modified
- `atime` -- last time file was **accessed** (read). Often disabled (`noatime`) for performance
- `ctime` -- last time file **metadata** (inode) changed (permissions, owner, link count)
- Note: Linux has no "creation time" in traditional ext filesystems. ext4 added `crtime` (birth time) but it is not universally exposed

## File Operations

The basic system calls for file manipulation:

```
create(path, mode)       -- Create a new file
open(path, flags)        -- Open file, returns file descriptor (fd)
read(fd, buffer, count)  -- Read bytes from current position
write(fd, buffer, count) -- Write bytes at current position
lseek(fd, offset, whence)-- Move the read/write position
close(fd)                -- Release the file descriptor
unlink(path)             -- Remove directory entry (delete when link count = 0)
truncate(path, length)   -- Resize file to given length
stat(path, buf)          -- Get file metadata without opening
```

Key insight: `open()` returns an integer (file descriptor), and all subsequent operations use that integer -- not the filename. This is a layer of indirection that enables powerful features.

## File Descriptors and the Three-Level Structure

When a process calls `open("/home/user/data.txt", O_RDONLY)`, the kernel builds a **three-level lookup structure**:

```
  Process A                  Kernel (shared)                 Disk
 +-----------------+      +---------------------+      +-----------+
 | FD Table        |      | Open File Table     |      | Inode     |
 | (per-process)   |      | (system-wide)       |      | Table     |
 |                 |      |                     |      |           |
 | fd 0 -> stdin   |      | entry 0:            |      | inode 42: |
 | fd 1 -> stdout  |      |   offset: 0         |      |  size     |
 | fd 2 -> stderr  |      |   flags: O_RDONLY   |      |  perms    |
 | fd 3 -------+   |      |   inode ptr --------+----->|  blocks   |
 |             |   |      |                     |      |  ...      |
 +-------------|---+      | entry 1:            |      +-----------+
               |          |   offset: 1024      |
               +--------->|   flags: O_RDWR     |
                          |   inode ptr --------|----> inode 87
  Process B               |                     |
 +-----------------+      | entry 2:            |
 | FD Table        |      |   offset: 0         |
 |                 |      |   flags: O_RDONLY   |
 | fd 0 -> stdin   |      |   inode ptr --------+----> inode 42
 | fd 1 -> stdout  |      |                     |      (same file!)
 | fd 2 -> stderr  |      +---------------------+
 | fd 3 -------+   |
 |             |   |
 +-------------|---+
               |
               +--------> (entry 2 in open file table)
```

**Level 1 -- Per-Process File Descriptor Table:**
- Each process has its own array of file descriptors
- FDs 0, 1, 2 are conventionally stdin, stdout, stderr
- Each entry points to an entry in the system-wide open file table

**Level 2 -- System-Wide Open File Table:**
- Shared across all processes
- Each entry stores: current file offset (position), access mode (flags), pointer to inode
- Two processes opening the same file get **different** open file table entries (different offsets)
- `fork()` shares the same open file table entry (parent and child share offset)
- `dup()` creates a new FD pointing to the **same** open file table entry

**Level 3 -- Inode Table (in-memory copy):**
- Contains the actual file metadata and block pointers
- One in-memory inode per open file (regardless of how many processes have it open)
- Cached from disk; written back when modified

### Why Three Levels?

This design separates three concerns:
1. **Process-local naming** (FD table) -- each process has its own FD namespace
2. **Session state** (open file table) -- offset and flags for each open instance
3. **File identity** (inode) -- the actual file, shared across all opens

This is why `fork()` + `exec()` works for I/O redirection in shells: the shell opens a file, puts it on fd 1, and the child inherits that FD table entry.

## File Types

Unix systems recognize several file types:

| Type | Character | Description | Example |
|------|-----------|-------------|---------|
| Regular file | `-` | Ordinary data file | `report.txt`, `app.bin` |
| Directory | `d` | Maps names to inodes | `/home`, `/etc` |
| Symbolic link | `l` | Points to another path | `libfoo.so -> libfoo.so.3` |
| Block device | `b` | Buffered device access | `/dev/sda`, `/dev/nvme0n1` |
| Character device | `c` | Unbuffered device access | `/dev/tty`, `/dev/null` |
| Named pipe (FIFO) | `p` | IPC between processes | Created with `mkfifo` |
| Socket | `s` | IPC endpoint | `/var/run/docker.sock` |

You can see the type in `ls -l` output:

```
drwxr-xr-x  2 user user 4096 Jan 10 10:00 documents/
-rw-r--r--  1 user user  512 Jan 10 10:00 notes.txt
lrwxrwxrwx  1 user user    9 Jan 10 10:00 link -> notes.txt
crw-rw-rw-  1 root root 1, 3 Jan 10 10:00 /dev/null
```

## File Access Methods

**Sequential access**: Read bytes in order, from beginning to end. This is the most common pattern. The file offset advances automatically after each `read()` or `write()`.

```
read() -> bytes 0-99
read() -> bytes 100-199
read() -> bytes 200-299
```

**Random (direct) access**: Jump to any position and read/write. Enabled by `lseek()`.

```
lseek(fd, 5000, SEEK_SET)   -- jump to byte 5000
read()                        -- read from byte 5000
lseek(fd, -100, SEEK_CUR)   -- move back 100 bytes
read()                        -- read from byte 4900
```

Databases use random access heavily. Log files are typically sequential (append-only). The access pattern matters enormously for performance -- especially on spinning disks where sequential access avoids seek time.

## Real-World Connection

- **"Too many open files" error**: Each process has a limit on FDs (often 1024 by default, tunable via `ulimit -n`). A server leaking file descriptors will hit this. Check with `ls /proc/<pid>/fd | wc -l`.
- **Docker and file descriptors**: Containers inherit the host's FD limits unless overridden. A common production issue.
- **`/dev/null`**: A character device file that discards all writes and returns EOF on reads. Used everywhere in shell scripting (`command > /dev/null 2>&1`).
- **Everything is a file**: Unix philosophy -- devices, pipes, sockets are all accessed through the same `open/read/write/close` interface. This is why you can redirect command output to a file, a pipe, or a network socket with the same shell syntax.

## Interview Angle

**Q: What happens when two processes open the same file?**

A: Each process gets its own file descriptor (in its per-process FD table) and its own open file table entry (with its own offset and flags). But both entries point to the same in-memory inode. So they can independently read/write at different positions. However, concurrent writes without coordination can interleave -- which is why databases use file locking (`flock`, `fcntl`).

**Q: What is the difference between `dup()` and `open()` on the same file?**

A: `open()` creates a new open file table entry (new offset, new flags). `dup()` creates a new FD pointing to the *same* open file table entry -- so both FDs share the same offset. When one advances, the other sees it. This is how shell I/O redirection works: `2>&1` makes FD 2 a `dup()` of FD 1.

**Q: After `fork()`, do parent and child share the file offset?**

A: Yes. `fork()` duplicates the FD table, but the entries point to the same open file table entries. So if the child reads 100 bytes, the parent's offset advances too. This is by design -- it is how pipes work between parent and child.

---

Next: [Directory Structure and Path Resolution](02-directory-structure-and-path-resolution.md)
