# VFS and FUSE

## The Problem: Too Many File Systems

Linux supports dozens of filesystem types: ext4, XFS, Btrfs, ZFS, FAT, NTFS, NFS, CIFS, procfs, sysfs, tmpfs, overlayfs, and more. Without an abstraction layer, every program would need filesystem-specific code to read a file from ext4 vs XFS vs NFS.

The **Virtual File System (VFS)** solves this by providing a single, uniform interface that all filesystems implement.

## VFS: The Abstraction Layer

VFS sits between user-space system calls and the individual filesystem drivers:

```
  User Space
+--------------------------------------------------+
|  Application: open("/home/user/file.txt", O_RDWR)|
+--------------------------------------------------+
         |  system call
         v
  Kernel Space
+--------------------------------------------------+
|                    VFS Layer                      |
|  - Resolves path                                 |
|  - Manages file descriptors                      |
|  - Calls the right FS driver                     |
|  - Common caching (page cache, dentry cache)     |
+--------------------------------------------------+
    |           |           |           |
    v           v           v           v
+-------+  +-------+  +-------+  +-------+
| ext4  |  |  XFS  |  |  NFS  |  | tmpfs |
| driver|  | driver|  | driver|  | driver|
+-------+  +-------+  +-------+  +-------+
    |           |           |           |
    v           v           v           v
+-------+  +-------+  +-------+  +-------+
| Block |  | Block |  |Network|  |Memory |
| device|  | device|  | (RPC) |  |(RAM)  |
+-------+  +-------+  +-------+  +-------+
```

Every filesystem driver registers a set of **operations** (function pointers) with VFS. When a user calls `read()`, VFS dispatches to the correct driver's `read` implementation.

## Key VFS Objects

VFS defines four core data structures that every filesystem must implement:

### 1. Superblock (`struct super_block`)

Represents a **mounted filesystem instance**. Contains:
- Filesystem type (ext4, XFS, etc.)
- Block size
- Maximum file size
- Pointer to the root inode
- Filesystem-specific private data
- Operations: `alloc_inode()`, `write_super()`, `sync_fs()`

One superblock per mounted filesystem. Created when `mount()` is called.

### 2. Inode (`struct inode`)

Represents a **file** (in the broad sense: regular file, directory, symlink, device, etc.). Contains:
- File type and permissions
- Owner, size, timestamps
- Operations: `lookup()`, `create()`, `link()`, `unlink()`, `mkdir()`

The VFS inode is an in-memory representation. Each FS driver maps it to/from its on-disk format.

### 3. Dentry (`struct dentry`)

Represents a **directory entry** -- one component of a path. The dentry cache is critical for performance:

```
Path: /home/user/file.txt

Dentry cache entries:
  "/" (root)
    -> "home" (child dentry)
        -> "user" (child dentry)
            -> "file.txt" (child dentry -> inode)
```

Dentries are cached even for negative lookups (file does not exist) to avoid repeated disk reads for non-existent paths.

### 4. File (`struct file`)

Represents an **open file** -- the instance created by `open()`. Contains:
- Current file position (offset)
- Access mode (read, write, append)
- Pointer to the dentry (and thus the inode)
- Operations: `read()`, `write()`, `llseek()`, `mmap()`, `poll()`

Multiple `struct file` objects can point to the same inode (multiple opens of the same file).

### How They Connect

```
Process FD Table
  fd 3 --> struct file
              |
              +-- f_pos = 1024 (current offset)
              +-- f_flags = O_RDWR
              +-- f_dentry --> struct dentry ("file.txt")
                                |
                                +-- d_parent --> dentry ("user")
                                +-- d_inode --> struct inode
                                                  |
                                                  +-- i_sb --> struct super_block
                                                  +-- i_op --> ext4_inode_operations
                                                  +-- i_fop --> ext4_file_operations
```

## How a New Filesystem Registers with VFS

A filesystem driver provides operation tables:

```
// Simplified -- what ext4 does at registration

static struct file_operations ext4_file_ops = {
    .read       = ext4_file_read,
    .write      = ext4_file_write,
    .llseek     = ext4_llseek,
    .mmap       = ext4_mmap,
    .open       = ext4_file_open,
    .release    = ext4_release_file,
    .fsync      = ext4_sync_file,
};

static struct inode_operations ext4_dir_inode_ops = {
    .create     = ext4_create,
    .lookup     = ext4_lookup,
    .link       = ext4_link,
    .unlink     = ext4_unlink,
    .mkdir      = ext4_mkdir,
    .rmdir      = ext4_rmdir,
    .rename     = ext4_rename,
};
```

This is a textbook example of the **strategy pattern**: VFS defines the interface, each FS provides the implementation.

## FUSE: Filesystem in Userspace

**FUSE** allows you to implement a filesystem as a **regular user-space program** instead of a kernel module.

```
Application:  open("/mnt/myfs/data.txt")
    |
    v
+-------------------+
|     VFS Layer     |
+-------------------+
    |
    v
+-------------------+
| FUSE kernel module|  <-- Generic kernel module (part of Linux)
+-------------------+
    |  (via /dev/fuse)
    v
+-------------------+
| Your FUSE daemon  |  <-- YOUR code, running in user space
| (user-space)      |
+-------------------+
    |
    v
  Whatever backend: SSH server, S3 bucket, 
  database, ZIP file, Google Drive...
```

**How it works:**
1. Your FUSE program registers mount point and handlers
2. When any process accesses a file under that mount point, VFS routes to the FUSE kernel module
3. The FUSE module forwards the request to your user-space daemon via `/dev/fuse`
4. Your daemon processes the request (e.g., fetches data from a remote server)
5. Response flows back: daemon -> FUSE module -> VFS -> application

### FUSE Examples

| FUSE Filesystem | What It Does |
|----------------|-------------|
| **SSHFS** | Mount a remote directory over SSH. `sshfs user@host:/remote /local` |
| **s3fs / goofys** | Mount an S3 bucket as a local directory |
| **gcsfuse** | Mount a Google Cloud Storage bucket |
| **NTFS-3G** | Read/write NTFS (used on Linux to access Windows drives) |
| **rclone mount** | Mount any cloud storage (Drive, Dropbox, OneDrive, etc.) |
| **archivemount** | Mount a tar/zip file as a directory |
| **EncFS** | Encrypted filesystem overlay |

### FUSE Trade-offs

**Advantages:**
- Develop in user space (any language: C, Python, Go, Rust)
- No kernel module compilation; no risk of kernel panics from bugs
- Easy to iterate and debug
- Can implement exotic data sources as filesystems

**Disadvantages:**
- **Performance overhead**: each operation crosses the kernel-user boundary twice (down to daemon, back up to kernel)
- Higher latency compared to in-kernel filesystems
- Memory copies between kernel and user space

## Virtual Filesystems: procfs and sysfs

Not all filesystems store data on disk. **Virtual filesystems** generate data dynamically from kernel state.

### procfs (`/proc`)

Exposes process and kernel information as a file hierarchy:

```
/proc/
  +-- 1/                  # Process PID 1 (init/systemd)
  |   +-- cmdline         # Command line arguments
  |   +-- status          # Process status (memory, state, threads)
  |   +-- fd/             # Open file descriptors (symlinks)
  |   +-- maps            # Memory mappings
  |   +-- cgroup          # Cgroup membership
  +-- cpuinfo             # CPU information
  +-- meminfo             # Memory statistics
  +-- loadavg             # System load averages
  +-- sys/                # Tunable kernel parameters
      +-- vm/
      |   +-- swappiness  # Can read AND write to tune behavior
      +-- net/
          +-- ipv4/
```

Reading `/proc/meminfo` does not read a file from disk -- the kernel generates the content on the fly from internal data structures.

### sysfs (`/sys`)

Exposes the **device model** and kernel object hierarchy:

```
/sys/
  +-- block/              # Block devices
  |   +-- sda/
  |       +-- queue/      # I/O scheduler settings
  |       +-- stat        # I/O statistics
  +-- class/              # Device classes
  |   +-- net/
  |       +-- eth0/
  +-- devices/            # Physical device tree
  +-- fs/                 # Filesystem parameters
```

Both procfs and sysfs are implemented as VFS drivers -- they register the same operation tables (superblock ops, inode ops, file ops) but generate content from kernel data structures instead of reading disk blocks.

## Real-World Connection

- **Docker and overlay filesystems**: Docker uses OverlayFS (a VFS-level filesystem) to layer container images. The lower layers are read-only base images; the upper layer captures writes. VFS path resolution walks the layers, checking the upper layer first. This is why container startup is fast -- no data copying.

- **Cloud storage FUSE mounts**: Tools like `gcsfuse` and `s3fs` let applications treat cloud storage as local files. This is powerful for compatibility (any program that reads files works with cloud storage) but has pitfalls: high latency, no atomic rename, eventual consistency. Serious workloads typically use native SDKs instead.

- **`/proc/PID` for debugging**: `cat /proc/1234/status` shows memory usage, thread count, and state of process 1234. `ls -l /proc/1234/fd` shows all open file descriptors. `cat /proc/1234/maps` shows the memory layout. These are indispensable for production debugging.

- **Kernel tuning via procfs/sysfs**: `echo 10 > /proc/sys/vm/swappiness` changes the kernel's swappiness at runtime. `echo deadline > /sys/block/sda/queue/scheduler` changes the I/O scheduler. These are virtual files backed by kernel variables.

- **Everything is a file (really)**: VFS is what makes the Unix philosophy concrete. Regular files, directories, devices (`/dev`), process info (`/proc`), kernel objects (`/sys`), network sockets -- all accessible through the same `open/read/write/close` interface because they all implement VFS operations.

## Interview Angle

**Q: What is VFS, and why does Linux need it?**

A: VFS (Virtual File System) is a kernel abstraction layer that provides a uniform file API (`open`, `read`, `write`, `close`) regardless of the underlying filesystem type. Without VFS, every application would need code for each filesystem. VFS defines four core objects (superblock, inode, dentry, file) that each filesystem driver implements. When you call `read()`, VFS dispatches to the correct driver's read function. This is the strategy pattern applied at the OS level.

**Q: What is FUSE? When would you use it, and what are the trade-offs?**

A: FUSE (Filesystem in Userspace) lets you implement a filesystem as a user-space program instead of a kernel module. The FUSE kernel module intercepts VFS calls and forwards them to your daemon via `/dev/fuse`. Use cases: mounting cloud storage (S3, GCS), accessing remote servers (SSHFS), or any custom data source. The trade-off is performance -- each operation crosses the kernel-user boundary twice, adding latency. For performance-critical workloads, in-kernel filesystems or native APIs are better.

**Q: How does `/proc` work? Is it a real filesystem?**

A: `/proc` is a virtual filesystem -- it has no backing storage on disk. It is implemented as a VFS driver where read operations generate content dynamically from kernel data structures. For example, reading `/proc/meminfo` triggers kernel code that formats current memory statistics into text. It is "real" in the sense that it is a fully functional VFS filesystem with inodes and dentries, but the data is synthesized on each read rather than stored persistently.

---

Next: [Modern File Systems: ext4, XFS, Btrfs, ZFS](08-modern-file-systems-ext4-xfs-btrfs-zfs.md)
