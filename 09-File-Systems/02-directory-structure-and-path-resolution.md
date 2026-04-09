# Directory Structure and Path Resolution

## What Is a Directory?

A **directory** is a special file that contains a list of mappings from names to file metadata. In Unix systems, each mapping is called a **directory entry (dentry)** and maps a filename to an **inode number**.

```
Directory "/home/user" (inode 500):
+------------------+--------+
| Name             | Inode  |
+------------------+--------+
| .                |  500   |   <-- this directory itself
| ..               |  300   |   <-- parent directory
| documents        |  501   |
| notes.txt        |  502   |
| .bashrc          |  503   |
+------------------+--------+
```

Key insight: **the filename is NOT stored in the inode** -- it is stored in the parent directory's data. An inode can have multiple names (hard links), or even no name at all (an open but deleted file).

## Directory Structures

Operating systems have evolved through several directory organization schemes:

### Single-Level Directory

```
+------------------------------------------+
| file1.txt  file2.txt  report.pdf  app.c  |
+------------------------------------------+
```

All files in one flat directory. Simple but impractical -- name collisions are inevitable, and there is no way to organize files logically.

### Two-Level Directory

```
Root
 +-- user1/
 |     file1.txt  file2.txt
 +-- user2/
       file1.txt  report.pdf
```

Each user gets their own directory. Solves naming collisions between users but still flat within each user's space.

### Tree-Structured (Hierarchical)

```
/
+-- home/
|   +-- user/
|       +-- documents/
|       |   +-- report.pdf
|       +-- .bashrc
+-- etc/
|   +-- nginx/
|       +-- nginx.conf
+-- var/
    +-- log/
        +-- syslog
```

The standard model used by all modern operating systems. Directories can contain subdirectories, forming an arbitrary-depth tree.

### Acyclic Graph

```
/home/user/project/shared.h  <---+
                                  |  (hard link)
/home/user/backup/shared.h   <---+
```

Extends the tree by allowing a file to appear in multiple directories (via hard links). The graph must remain acyclic -- no directory loops -- which is why most systems prohibit hard links to directories.

## Path Resolution: Step by Step

When you access `/home/user/file.txt`, the kernel walks the directory tree:

```
Step 1: Start at root inode (always inode 2 on ext4)
        Read root directory data blocks
        
Step 2: Search root directory entries for "home"
        Found: "home" -> inode 100
        Check: is inode 100 a directory? Yes.
        Check: do we have execute (search) permission? Yes.
        
Step 3: Read inode 100's data blocks (the /home directory)
        Search for "user"
        Found: "user" -> inode 500
        Check: directory? Yes. Permission? Yes.
        
Step 4: Read inode 500's data blocks (the /home/user directory)
        Search for "file.txt"
        Found: "file.txt" -> inode 502
        
Step 5: Read inode 502 metadata
        Check permissions for requested operation (read/write/etc.)
        Return inode 502 to the caller
```

```
 /            home/          user/          file.txt
 inode 2  --> inode 100  --> inode 500  --> inode 502
 [read dir]   [read dir]    [read dir]     [done!]
 find "home"  find "user"   find "file.txt"
```

**This is why deep paths are slightly slower**: each component requires reading a directory from disk (though the dentry cache eliminates most of this cost in practice).

### Absolute vs Relative Paths

- **Absolute path** (`/home/user/file.txt`): resolution starts at root inode (inode 2)
- **Relative path** (`./documents/file.txt`): resolution starts at the process's **current working directory** (stored in the process's task struct)

The `chdir()` system call changes the current working directory. Each process tracks its own cwd.

### The Dentry Cache

Path resolution would be painfully slow if every component required a disk read. Linux maintains a **dentry cache (dcache)** in memory -- a hash table mapping (parent inode, name) to child inode. Most path lookups hit the cache and never touch disk.

## Hard Links vs Symbolic Links

### Hard Links

A hard link is an additional directory entry pointing to the same inode:

```
Directory A:               Directory B:
+------------+------+      +------------+------+
| report.txt | 1001 |      | backup.txt | 1001 |  <-- same inode!
+------------+------+      +------------+------+

Inode 1001:
  type: regular file
  link count: 2          <-- tracks number of hard links
  size: 4096
  blocks: [50, 51]
```

**Properties:**
- Same inode, same data -- changes through one name are visible through the other
- Cannot cross filesystem boundaries (inodes are per-filesystem)
- Cannot link to directories (to prevent cycles)
- File is only deleted when link count reaches 0 AND no process has it open
- `ls -l` shows the link count (second column)

**Created with**: `ln existing_file new_name`

### Symbolic (Soft) Links

A symbolic link is a separate file whose content is a path string:

```
Directory:
+------------+------+
| link.txt   | 2000 |   <-- different inode
| target.txt | 1001 |
+------------+------+

Inode 2000 (symlink):        Inode 1001 (regular file):
  type: symbolic link          type: regular file
  data: "target.txt"           size: 4096
  size: 10 (path length)       blocks: [50, 51]
```

**Properties:**
- Separate inode, separate file -- just stores a path string
- CAN cross filesystem boundaries
- CAN point to directories
- CAN dangle (target can be deleted, symlink still exists but broken)
- Has its own permissions (though most systems ignore them -- target permissions matter)

**Created with**: `ln -s target_path link_name`

### Comparison Table

| Feature | Hard Link | Symbolic Link |
|---------|-----------|---------------|
| Same inode as target? | Yes | No (has its own inode) |
| Cross filesystem? | No | Yes |
| Link to directories? | No (usually) | Yes |
| Dangling possible? | No | Yes (target can be deleted) |
| Performance | Direct (no extra lookup) | Extra step to read link content |
| Disk space | Just a directory entry | A small file (stores path) |
| Target deleted? | File persists (link count > 0) | Link becomes broken |

## Mounting

**Mounting** attaches a filesystem to a directory (the **mount point**) in the existing hierarchy:

```
Before mount:                    After: mount /dev/sdb1 /mnt/usb
                                 
/                                /
+-- home/                        +-- home/
+-- mnt/                         +-- mnt/
    +-- usb/  (empty dir)            +-- usb/       <-- now shows contents
                                         +-- photos/    of /dev/sdb1
                                         +-- music/
```

During path resolution, when the kernel encounters a mount point, it switches to the root inode of the mounted filesystem. The original contents of the mount point directory are hidden (but not deleted) until unmount.

**Key commands:**
- `mount /dev/sdb1 /mnt/usb` -- attach filesystem
- `umount /mnt/usb` -- detach filesystem
- `mount` (no args) or `findmnt` -- show current mounts
- `/etc/fstab` -- persistent mount configuration

## Real-World Connection

- **Linux Filesystem Hierarchy Standard (FHS)**: `/bin`, `/etc`, `/home`, `/var`, `/tmp` all have defined purposes. Knowing FHS helps you navigate any Linux system.
- **Container layered filesystems**: Docker uses OverlayFS, which stacks a read-only base layer with a writable upper layer. Path resolution walks the layers top-down -- upper layer files "override" lower layer ones. This is how container images share base layers efficiently.
- **`chroot` and mount namespaces**: `chroot` changes a process's root directory (used for sandboxing). Containers use mount namespaces to give each container its own filesystem view with its own mount table.
- **Broken symlinks in production**: A deployment that replaces a directory via symlink swap (`ln -sfn new_release current`) can briefly create dangling references. Tools like `readlink -f` resolve the full chain.
- **The `/proc` filesystem**: A virtual directory structure where path resolution is handled by kernel code, not disk data. `/proc/1234/fd/` shows all file descriptors for PID 1234 -- each entry is a symlink to the actual file.

## Interview Angle

**Q: Walk through what happens when the kernel resolves `/home/user/file.txt`.**

A: Start at root inode (inode 2). Read the root directory's data blocks and search for "home" -- find its inode number. Read that inode, confirm it is a directory and we have search permission. Read its data blocks, search for "user." Repeat: read the "user" inode, read its directory data, search for "file.txt." Read the final inode and check permissions for the requested operation. In practice, the dentry cache short-circuits most of these disk reads.

**Q: Why can't hard links cross filesystem boundaries?**

A: A hard link is just a directory entry containing an inode number. Inode numbers are local to a single filesystem -- inode 500 on `/dev/sda1` is a completely different file from inode 500 on `/dev/sdb1`. A directory entry on one filesystem cannot meaningfully reference an inode on another. Symbolic links solve this by storing a path string instead.

**Q: A file is deleted with `rm`, but disk space is not reclaimed. Why?**

A: `rm` calls `unlink()`, which removes the directory entry and decrements the inode's link count. But the file's blocks are only freed when the link count reaches 0 AND no process has the file open. If a process still has the file open (common with log files), the data persists until that process closes the FD. Check with `lsof +L1` to find deleted-but-open files.

---

Next: [File Allocation Methods](03-file-allocation-methods.md)
