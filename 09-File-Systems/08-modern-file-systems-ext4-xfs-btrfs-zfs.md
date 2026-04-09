# Modern File Systems: ext4, XFS, Btrfs, ZFS

## Choosing a Filesystem in Practice

Every production system runs on a real filesystem, and the choice matters. This section covers the four most important Linux filesystems, their design philosophies, strengths, and when to use each.

---

## ext4: The Reliable Workhorse

**Lineage**: ext2 (1993) -> ext3 (2001, added journaling) -> ext4 (2008, major improvements)

ext4 is the **default filesystem on most Linux distributions** (Debian, Ubuntu, Arch, etc.). It evolved incrementally from ext2, prioritizing backward compatibility and stability.

### Key Features

| Feature | Details |
|---------|---------|
| Extents | Replaces indirect block pointers. Each extent = (logical block, physical start, length). One extent can map up to 128 MB of contiguous data. |
| Delayed allocation | Defers block allocation until data is flushed to disk. Better contiguity, fewer allocations for temp files. |
| Journal checksumming | Checksums journal entries to detect corruption in the journal itself. |
| Multi-block allocator | Allocates multiple blocks at once, improving layout for large writes. |
| Max file size | 16 TB |
| Max filesystem size | 1 EB (exabyte) |
| Inode allocation | Fixed at format time (configurable with `mkfs.ext4 -i`) |
| Online resize | Can grow (not shrink) a mounted filesystem |

### When to Use ext4

- General-purpose Linux servers and workstations
- When you need maximum stability and tooling maturity
- Boot partitions (widest bootloader support)
- When in doubt -- ext4 is rarely the wrong choice

### Limitations

- Fixed inode count (can run out with many small files)
- No built-in checksumming of data blocks (only journal checksums)
- No native snapshots, compression, or deduplication
- Single-threaded fsck can be slow on very large volumes

---

## XFS: High-Performance at Scale

**Origin**: Developed by SGI (Silicon Graphics) in 1993 for IRIX. Ported to Linux in 2001. **Default on RHEL, CentOS, Rocky Linux, and Amazon Linux.**

XFS was designed from the start for large files, large filesystems, and parallel I/O.

### Key Features

| Feature | Details |
|---------|---------|
| B+ tree everything | Metadata (inodes, directories, free space) stored in **B+ trees** -- self-balancing tree structures where all data sits in leaf nodes and internal nodes are just navigation keys, guaranteeing O(log n) lookups with minimal disk reads per level. |
| Allocation groups | Filesystem divided into independent allocation groups (AGs) that can be operated on in parallel. |
| Delayed allocation | Like ext4, defers allocation for better layout. |
| Online defragmentation | `xfs_fsr` reorganizes files while mounted. |
| Dynamic inode allocation | Inodes allocated on demand -- never runs out of inodes. |
| Max file size | 8 EB (exabytes) |
| Max filesystem size | 8 EB |
| Parallel I/O | Per-AG locking enables high concurrency for multi-threaded workloads. |
| Reflinks | Copy-on-write file clones (`cp --reflink`) -- instant file copy. |

### When to Use XFS

- Large files (media, databases, scientific data)
- High-throughput parallel I/O workloads
- Servers with many concurrent readers/writers
- Enterprise Linux environments (RHEL ecosystem)

### Limitations

- Cannot be shrunk (only grown)
- Historically weaker for workloads with many small files (improved significantly in recent kernels)
- Less community tooling compared to ext4

---

## Btrfs: The Feature-Rich COW Filesystem

**Origin**: Development started at Oracle in 2007. Included in Linux mainline since 2009. **Default on openSUSE and Fedora Workstation (since Fedora 33).**

Btrfs (pronounced "butter-FS" or "B-tree FS") uses copy-on-write throughout and aims to be the Linux answer to ZFS.

### Key Features

| Feature | Details |
|---------|---------|
| Copy-on-Write | Never overwrites data in place. Enables snapshots and crash consistency without a journal. |
| Snapshots | Instant, space-efficient snapshots of subvolumes. Only changed blocks consume additional space. |
| Built-in RAID | RAID 0, 1, 10, 5, 6 managed at the filesystem level (RAID 5/6 had stability issues, improving). |
| Checksums | Every data and metadata block is checksummed (CRC32C by default, also SHA-256, xxhash). |
| Transparent compression | zlib, LZO, or zstd compression per-file or per-mount. |
| Subvolumes | Lightweight, independent filesystem namespaces within a single Btrfs filesystem. |
| Send/receive | Incrementally replicate snapshots to another Btrfs filesystem. |
| Online resize | Grow and shrink while mounted. |
| Max file size | 16 EB |
| Max filesystem size | 16 EB |

### Snapshot Workflow Example

```bash
# Create a snapshot before a risky upgrade
$ btrfs subvolume snapshot / /snapshots/pre-upgrade

# Do the upgrade...
$ apt upgrade

# Something broke? Roll back:
$ btrfs subvolume set-default /snapshots/pre-upgrade
$ reboot
```

### When to Use Btrfs

- Systems that benefit from snapshots (workstations, development machines)
- Data integrity requirements (checksumming catches bit rot)
- Flexible storage management (add/remove disks, change RAID levels)
- Desktop Linux where compression saves SSD space

### Limitations

- RAID 5/6 implementation has had stability concerns (improving but not universally trusted)
- Fragmentation from COW can impact performance on HDDs with random-write workloads
- Less mature than ext4/XFS for enterprise database workloads
- Higher memory usage for metadata management

---

## ZFS: Maximum Data Integrity

**Origin**: Developed at Sun Microsystems (2005). Open-sourced as OpenZFS. Available on Linux via OpenZFS/DKMS (not in mainline kernel due to CDDL license vs GPL).

ZFS is the most feature-rich filesystem available, combining a filesystem and volume manager into one.

### Key Features

| Feature | Details |
|---------|---------|
| Combined FS + volume manager | ZFS manages pools of disks directly -- no need for separate LVM/mdadm. |
| 128-bit addressing | Theoretical max capacity: 256 quadrillion zettabytes. Future-proof. |
| End-to-end checksums | Every block (data and metadata) is checksummed. Parent blocks store children's checksums. |
| Self-healing | With redundancy (mirror/RAID-Z), ZFS detects and automatically repairs corrupted blocks using the good copy. |
| Copy-on-Write | Like Btrfs, never overwrites data in place. |
| Snapshots and clones | Instant snapshots; clones are writable snapshots. |
| Deduplication | Block-level dedup (memory-intensive: ~5GB RAM per TB of deduplicated data). |
| Compression | LZ4 (default, fast), zstd, gzip, lzjb. |
| RAID-Z (1/2/3) | Software RAID with 1, 2, or 3 parity disks. No write hole problem unlike traditional RAID 5. |
| Send/receive | Incremental snapshot replication (backup to remote). |
| ARC cache | Adaptive Replacement Cache -- sophisticated in-memory caching of frequently accessed blocks. |
| Max file size | 16 EB |
| Max filesystem size | 256 ZB (theoretical) |

### ZFS Architecture

```
+------------------------------------------+
|           ZFS Datasets (filesystems)     |
|  /tank/home   /tank/data   /tank/docker  |
+------------------------------------------+
|               ZFS Pool (zpool)           |
|  "tank": manages all disks as one pool   |
+------------------------------------------+
|  vdev: mirror     |  vdev: raidz1        |
|  [sda] [sdb]      |  [sdc] [sdd] [sde]  |
+------------------------------------------+
```

### When to Use ZFS

- NAS and storage servers (FreeNAS/TrueNAS is built on ZFS)
- Systems where data integrity is paramount
- Large storage pools with mixed disk sizes
- Environments that need robust snapshot and replication
- Production databases where silent corruption is unacceptable

### Limitations

- License incompatibility (CDDL vs GPL) -- cannot be shipped in the Linux kernel; requires DKMS
- Higher memory requirements (minimum 1GB RAM per TB of storage recommended, more for dedup)
- Cannot easily remove disks from a vdev (pool restructuring is limited)
- Steeper learning curve

---

## NTFS: Brief Overview

**NTFS** (New Technology File System) is the default Windows filesystem:

| Feature | Details |
|---------|---------|
| Journaling | Yes (metadata journaling via the $LogFile) |
| Max file size | 16 EB (theoretical, practical limit lower) |
| ACLs | Rich access control lists (more granular than Unix permissions) |
| Alternate data streams | Files can have multiple named data streams (used for metadata, also exploited by malware) |
| Compression | Per-file NTFS compression |
| Encryption | EFS (Encrypting File System) built in |
| Linux support | Read-write via NTFS-3G (FUSE) or the newer ntfs3 kernel driver |

---

## Comparison Table

| Feature | ext4 | XFS | Btrfs | ZFS |
|---------|------|-----|-------|-----|
| **Design** | Journaling | Journaling + B+ trees | Copy-on-Write | Copy-on-Write |
| **Max file size** | 16 TB | 8 EB | 16 EB | 16 EB |
| **Max FS size** | 1 EB | 8 EB | 16 EB | 256 ZB |
| **Data checksums** | No | No | Yes | Yes |
| **Snapshots** | No (use LVM) | No (use LVM) | Yes (native) | Yes (native) |
| **Compression** | No | No | Yes (zstd, lzo, zlib) | Yes (lz4, zstd, gzip) |
| **Deduplication** | No | No | No (planned) | Yes (RAM-heavy) |
| **RAID support** | No (use mdadm) | No (use mdadm) | Yes (built-in) | Yes (RAID-Z) |
| **Shrink filesystem** | Yes (offline) | No | Yes (online) | No |
| **Dynamic inodes** | No (fixed) | Yes | Yes | Yes |
| **Self-healing** | No | No | With redundancy | Yes |
| **Stability** | Excellent | Excellent | Good (improving) | Excellent |
| **License** | GPL | GPL | GPL | CDDL (not in kernel) |
| **Default on** | Debian, Ubuntu, Arch | RHEL, CentOS, Amazon Linux | openSUSE, Fedora WS | FreeBSD, TrueNAS |

## How to Choose

```
Decision tree:

Need maximum stability and simplicity?
  --> ext4 (proven, boring, works everywhere)

Working with large files or high-throughput parallel I/O?
  --> XFS (designed for this, excellent at scale)

Need snapshots, checksums, or compression on Linux?
  --> Btrfs (best native Linux option for these features)

Data integrity is absolutely critical? Have RAM to spare?
  --> ZFS (the gold standard, accept the license complexity)

Boot partition?
  --> ext4 (widest bootloader support)

USB drive or cross-platform sharing?
  --> FAT32 (< 4GB files) or exFAT (> 4GB files)
```

## Real-World Connection

- **Cloud providers**: AWS EBS volumes typically default to ext4 or XFS. GCP persistent disks default to ext4. The choice is driven by stability and broad compatibility.
- **Container storage**: Docker's overlay2 storage driver works on top of ext4 or XFS. The underlying filesystem affects performance: XFS with d_type support is required for overlay2 on CentOS/RHEL.
- **Database workloads**: PostgreSQL and MySQL perform well on both ext4 and XFS. XFS often wins for write-heavy workloads due to its allocation group parallelism. ZFS is popular for database servers that need snapshot-based backups.
- **NAS appliances**: TrueNAS (formerly FreeNAS), Synology, and many NAS solutions use ZFS or Btrfs for their snapshot and data integrity features.
- **Fedora and openSUSE defaults**: Fedora switched to Btrfs by default in 2020, giving it broader desktop testing. openSUSE has used Btrfs with snapper (snapshot manager) for automatic rollbacks of failed updates.
- **Kubernetes persistent volumes**: The CSI (Container Storage Interface) driver handles formatting volumes -- you configure the filesystem type. Most operators choose ext4 for simplicity or XFS for performance.

## Interview Angle

**Q: When would you choose XFS over ext4?**

A: XFS excels at large files and parallel I/O thanks to its allocation group architecture that allows concurrent operations across groups. It dynamically allocates inodes (never runs out), and its B+ tree metadata structures scale better than ext4 for very large directories. Choose XFS for: database servers, media storage, high-throughput workloads, or when the filesystem will be very large. Choose ext4 for: general purpose, smaller filesystems, when you need the ability to shrink, or boot partitions.

**Q: What makes ZFS special compared to other filesystems?**

A: ZFS combines the filesystem and volume manager, checksums every block (data and metadata), supports self-healing (automatically repairs corrupt blocks from redundant copies), and provides instant snapshots via COW. Its end-to-end checksum chain means silent data corruption (bit rot) is detected, something ext4 and XFS cannot do. The trade-offs are higher memory requirements, CDDL license incompatibility with the Linux kernel, and a steeper learning curve.

**Q: What is the difference between journaling (ext4) and copy-on-write (Btrfs/ZFS) for crash consistency?**

A: Journaling writes planned changes to a log first, then applies them in place. After a crash, the journal is replayed to complete or discard partial operations. COW never overwrites existing blocks -- it writes new data to a new location and atomically swaps the pointer. The old data remains valid until the swap completes. COW provides crash consistency without a separate journal and enables free snapshots (just keep old pointers), but causes fragmentation because updated blocks are written to new locations rather than in place.

**Q: A team is setting up a storage server for backups. They need snapshots, data integrity checks, and the ability to add disks over time. What do you recommend?**

A: ZFS or Btrfs. ZFS is the stronger choice: end-to-end checksums detect bit rot, RAID-Z avoids the write hole problem, snapshots are instant and space-efficient, and send/receive enables incremental replication to a remote backup. The trade-off is CDDL licensing (need DKMS on Linux) and higher RAM requirements. Btrfs is the alternative if you want GPL licensing and in-kernel support, but its RAID 5/6 is less mature. Either way, avoid ext4/XFS for this use case -- they lack native snapshots and checksumming.
