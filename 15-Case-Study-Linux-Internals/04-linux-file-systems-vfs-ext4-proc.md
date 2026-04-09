# Linux File Systems: VFS, ext4, and procfs

## The Core Idea

Linux supports dozens of different filesystems — ext4, XFS, Btrfs, NFS, FAT32, NTFS, procfs, sysfs, tmpfs — yet every user-space program uses the same `open()`, `read()`, `write()`, `close()` system calls regardless of which filesystem holds the data. This uniformity is not magic. It is the result of the **Virtual File System (VFS)** layer, one of the most elegant abstractions in the Linux kernel.

**VFS is like a universal translator. Your application speaks one language: "open this file, read these bytes, write this data." VFS translates that into whatever the underlying filesystem actually needs — ext4 talks about extents and journals, NFS talks about RPC and network packets, procfs talks about kernel data structures. The application never knows and never cares.**

---

## VFS: The Four Key Objects

The VFS layer is built around four fundamental data structures. Every filesystem must implement (or inherit defaults for) operations on each:

```
┌────────────────────────────────────────────────────────────┐
│                    VFS Layer                                │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────┐  ┌──────┐ │
│  │ superblock   │  │   inode     │  │ dentry  │  │ file │ │
│  │ (filesystem  │  │ (one per    │  │ (path   │  │ (open│ │
│  │  instance)   │  │  file/dir)  │  │  entry) │  │  fd) │ │
│  └──────┬──────┘  └──────┬──────┘  └────┬────┘  └──┬───┘ │
│         │                │              │           │      │
│  super_operations  inode_operations  dentry_ops  file_ops  │
└─────────┼────────────────┼──────────────┼───────────┼──────┘
          │                │              │           │
          ▼                ▼              ▼           ▼
     Filesystem-specific implementations
     (ext4, XFS, NFS, procfs, tmpfs, ...)
```

| VFS Object | What It Represents | Key Contents |
|------------|-------------------|--------------|
| **superblock** | A mounted filesystem instance | Block size, total/free blocks, root inode, filesystem type |
| **inode** | A specific file or directory (metadata) | Size, permissions, timestamps, block pointers, link count |
| **dentry** | A directory entry (path component) | Name, parent dentry, pointer to inode |
| **file** | An open file descriptor | Current position (offset), access mode, pointer to dentry and inode |

### Operations Tables

Each filesystem driver registers function tables that tell VFS how to perform operations:

```c
// Filesystem registers these at mount time:
struct super_operations ext4_sops = {
    .alloc_inode    = ext4_alloc_inode,
    .destroy_inode  = ext4_destroy_inode,
    .write_inode    = ext4_write_inode,
    .put_super      = ext4_put_super,
    .statfs         = ext4_statfs,
    ...
};

struct inode_operations ext4_dir_inode_operations = {
    .create     = ext4_create,      // Create a new file
    .lookup     = ext4_lookup,      // Find inode for a name
    .link       = ext4_link,        // Create hard link
    .unlink     = ext4_unlink,      // Remove directory entry
    .mkdir      = ext4_mkdir,
    .rmdir      = ext4_rmdir,
    .rename     = ext4_rename2,
    ...
};

struct file_operations ext4_file_operations = {
    .read_iter  = ext4_file_read_iter,
    .write_iter = ext4_file_write_iter,
    .mmap       = ext4_file_mmap,
    .fsync      = ext4_sync_file,
    .llseek     = ext4_llseek,
    ...
};
```

When you call `read()` on an ext4 file, VFS looks up the file's `file_operations` table and calls `ext4_file_read_iter()`. If the same code runs on an NFS file, VFS calls `nfs_file_read()` instead. The syscall interface is identical; only the backend differs.

---

## Dentry Cache (dcache)

Path lookup is one of the most frequent operations in the kernel. Every `open("/var/log/syslog")` requires resolving `/`, then `var`, then `log`, then `syslog` — four directory lookups. Hitting disk for each one would be painfully slow.

The **dentry cache** (dcache) keeps recently resolved directory entries in memory. It is a hash table indexed by (parent dentry, name) that maps to the target dentry:

```
Path: /var/log/syslog

Lookup chain:
  "/" → dentry for root (always cached)
   └─ "var" → dentry for /var (cached)
       └─ "log" → dentry for /var/log (cached)
           └─ "syslog" → dentry for /var/log/syslog (cached → inode)
```

An important optimization is **negative dentries** — the dcache caches "this name does NOT exist in this directory." Why? Because many programs probe for files that may or may not exist (e.g., searching `$PATH` for an executable). Without negative caching, each probe would hit disk. With it, the kernel instantly answers "no" from cache.

The dcache is one of the most performance-critical caches in the kernel. On a busy web server, path lookups happen millions of times per second, and the dcache hit rate is typically 99%+.

---

## Page Cache for File I/O

The page cache (covered in the memory section) is deeply intertwined with VFS. Here is how read and write operations actually flow:

### Read Path

```
Application: read(fd, buf, 4096)
    │
    ▼
VFS: vfs_read() → file->f_op->read_iter()
    │
    ▼
Filesystem (e.g., ext4): ext4_file_read_iter()
    │
    ▼
generic_file_read_iter()
    │
    ▼
Check page cache (find_get_page):
    ├── HIT: Copy data from cached page to user buffer → return
    │
    └── MISS: a]locate page, submit I/O to block layer
              Wait for I/O completion
              Add page to cache
              Copy data to user buffer → return
```

### Write Path

```
Application: write(fd, buf, 4096)
    │
    ▼
VFS: vfs_write() → file->f_op->write_iter()
    │
    ▼
generic_file_write_iter()
    │
    ▼
Find or allocate page in page cache
Copy data from user buffer into cached page
Mark page DIRTY
Return to application (fast — no disk I/O yet!)
    │
    ▼  (later, asynchronously)
Writeback daemon (kworker) wakes up
Finds dirty pages past the dirty threshold
Calls filesystem's writeback to flush to disk
Clears dirty flag
```

This is why power loss can lose recent writes — dirty pages in the page cache have not yet been flushed to disk. The `fsync()` syscall forces all dirty pages for a file to be written to disk and waits for completion. Databases call `fsync()` (or `fdatasync()`) after every transaction commit to guarantee durability.

**Dirty thresholds** control when writeback starts:

```bash
# Writeback starts when dirty pages exceed this % of RAM:
cat /proc/sys/vm/dirty_background_ratio    # Default: 10%

# Processes writing data are BLOCKED when dirty pages exceed this %:
cat /proc/sys/vm/dirty_ratio               # Default: 20%
```

---

## ext4 Internals

ext4 is the default filesystem on most Linux distributions. It evolved from ext3 (which added journaling to ext2) and is designed for reliability, performance, and backward compatibility.

### Extents

Older filesystems (ext2/ext3) used **indirect block pointers** — each inode contained pointers to data blocks, pointers to blocks of pointers, and so on. For a large file, this was slow and wasteful.

ext4 uses **extents** — each extent describes a contiguous range of disk blocks:

```
Indirect blocks (ext3):          Extents (ext4):
inode → block 100                inode → extent: start=100, len=500
inode → block 101                       (one entry covers 500 blocks!)
inode → block 102
... (hundreds of pointers)       Much more compact, much faster lookup
```

An extent is a triple: (logical block, physical block, length). A perfectly contiguous 1 GB file needs only ONE extent entry, versus thousands of indirect block pointers.

### Delayed Allocation

ext4 does not allocate disk blocks when a `write()` occurs. It waits until the data is actually flushed from the page cache to disk (writeback time). This gives the block allocator a much better picture of the file's total size, enabling it to allocate contiguous blocks and reduce fragmentation.

### Journal (JBD2)

ext4 uses a journal to protect filesystem metadata consistency across crashes. The default mode is **ordered**:

| Journal Mode | What Gets Journaled | Tradeoff |
|-------------|-------------------|---------|
| **journal** | Metadata AND data | Safest but slowest (data written twice) |
| **ordered** (default) | Only metadata, but data is flushed before metadata | Good balance of safety and speed |
| **writeback** | Only metadata, data can be written in any order | Fastest but data can be stale after crash |

### Directory Indexing (htree)

For directories with thousands of entries, linear search is too slow. ext4 uses a hash tree (htree) — a B-tree variant that hashes filenames for O(log n) lookups. This is why `ls` in a directory with 100,000 files is still fast on ext4.

### Multiblock Allocator (mballoc)

When allocating blocks for a file, ext4 tries to allocate many blocks at once rather than one at a time. Combined with delayed allocation, this results in much better contiguity and reduced fragmentation. mballoc uses buddy-system-like lists to find the best fit.

---

## procfs (/proc)

`/proc` is a **virtual filesystem** — no data lives on disk. Every read generates content on the fly from kernel data structures. It was originally designed for process information (`/proc/PID/`) but grew to expose system-wide information.

### Per-Process Information

```
/proc/PID/
├── status        Readable process state (Name, State, Pid, Uid, VmRSS, Threads)
├── stat          Raw process stats (state, ppid, priority, CPU times) — for tools
├── maps          Virtual memory areas with addresses, permissions, mapped files
├── cmdline       Command line arguments (null-separated)
├── environ       Environment variables (null-separated)
├── fd/           Directory of open file descriptors (symlinks to real files)
├── fdinfo/       Per-fd details (position, flags, mnt_id)
├── cgroup        Cgroup memberships
├── mountinfo     Mount table from this process's mount namespace
├── ns/           Namespace references (ipc, mnt, net, pid, user, uts)
├── io            I/O statistics (bytes read/written, syscalls)
├── oom_score     Current OOM score
├── oom_score_adj OOM score adjustment (-1000 to +1000)
└── limits        Resource limits (max open files, max processes, etc.)
```

### System-Wide Information

```bash
/proc/meminfo       # Memory statistics (MemTotal, MemAvailable, Cached, Buffers)
/proc/cpuinfo       # CPU model, features, cache sizes, per-core info
/proc/interrupts    # Interrupt counts per CPU per IRQ line
/proc/loadavg       # 1/5/15-minute load averages
/proc/stat          # System-wide CPU times, context switches, process counts
/proc/diskstats     # Per-disk I/O statistics
/proc/net/tcp       # Open TCP connections (used by netstat/ss)
/proc/sys/          # Tunable kernel parameters (writable via sysctl)
```

Internally, each `/proc` entry is backed by a kernel function that formats the current state. `cat /proc/meminfo` calls `meminfo_proc_show()` which reads the zone statistics and formats them as text. This is why `/proc` files have no meaningful file size — the content does not exist until you read it.

---

## sysfs (/sys)

`/sys` exposes the kernel's internal **object model** as a filesystem hierarchy. Where `/proc` is a grab-bag of information, `/sys` is structured around the kernel's kobject/kset hierarchy:

```
/sys/
├── block/          Block devices (sda, nvme0n1, dm-0)
├── bus/            Bus types (pci, usb, i2c)
├── class/          Device classes
│   ├── net/        Network interfaces (eth0, wlan0)
│   ├── block/      Block devices
│   └── thermal/    Thermal zones
├── devices/        Physical device tree
├── fs/             Filesystem parameters
│   ├── cgroup/     Cgroup filesystem
│   └── ext4/       Per-ext4-mount tuning
├── kernel/         Kernel objects (security, uevent_helper)
└── module/         Loaded modules and their parameters
```

---

## tmpfs: RAM-Backed Filesystem

tmpfs stores files in memory (page cache + swap). It is used for:
- `/tmp` — Temporary files (fast, auto-cleaned on reboot)
- `/run` — Runtime state (PID files, sockets)
- `/dev/shm` — POSIX shared memory
- initramfs (the initial root filesystem during boot)

tmpfs only uses memory for files that actually have data — an empty tmpfs uses no RAM. Files can be swapped to disk under memory pressure, unlike a ramdisk which holds pages permanently in RAM.

```bash
# Mount a tmpfs with a 512 MB limit:
mount -t tmpfs -o size=512M tmpfs /mnt/tmp

# Check current usage:
df -h /dev/shm
```

---

## Real-World Connection

**NFS and the VFS abstraction:** When an application on a web server reads `/mnt/shared/data.csv` and the mount is NFS, VFS transparently translates the `read()` into NFS RPC calls over the network to the NFS server. The application has no idea the file is remote. This same abstraction is what makes FUSE (Filesystem in Userspace) possible — you can implement a filesystem in Python that speaks the VFS interface, and every program on the system can use it transparently.

**The ext4 journal in practice:** When a Linux system crashes (power loss, kernel panic), ext4 replays its journal on the next mount. This takes seconds rather than the minutes that a full `fsck` scan would take. The journal ensures metadata consistency — you will not get corrupted directory entries or dangling inode references. Data consistency depends on the journal mode (ordered mode protects against stale data in most cases).

**Why /proc is a security surface:** `/proc/PID/` exposes detailed information about every process — command line arguments, environment variables (which may contain secrets), memory maps, open file descriptors. Container runtimes like Docker use procfs masking (`/proc/kcore`, `/proc/sysrq-trigger`) and read-only remounts to limit what containers can see. Hardened systems may restrict `/proc` access via `hidepid` mount options.

---

## Interview Angle

**Q: What is the VFS layer and why does Linux need it?**

A: VFS (Virtual File System) provides a uniform interface for all filesystem operations. Applications use the same `open()`, `read()`, `write()`, `close()` syscalls regardless of whether the file is on ext4, XFS, NFS, or even a virtual filesystem like procfs. Each filesystem driver registers operations tables (file_operations, inode_operations, super_operations) that VFS dispatches to. This means adding a new filesystem requires implementing these operations — user-space code never changes. Without VFS, every application would need filesystem-specific code paths.

**Q: How does the dentry cache improve performance?**

A: The dentry cache (dcache) keeps recently resolved path components in a hash table indexed by (parent, name). Since path lookup is one of the most frequent kernel operations (every `open()` requires it), the dcache turns multi-step disk lookups into fast in-memory hash table lookups with 99%+ hit rates on typical systems. It also caches negative entries — "this file does not exist in this directory" — to avoid repeated disk probes for nonexistent files.

**Q: Explain the difference between ext4 journal modes.**

A: ext4 has three journal modes. **Journal mode** writes both metadata and data to the journal before committing — safest but slowest because data is written twice. **Ordered mode** (default) only journals metadata, but guarantees data is flushed to its final location before the metadata journal entry is committed — good balance of safety and speed. **Writeback mode** only journals metadata with no ordering guarantee for data — fastest but can expose stale data after a crash if new metadata points to blocks that have not yet received their new data.

---

**Next**: [05-linux-io-and-networking-internals.md](05-linux-io-and-networking-internals.md)
