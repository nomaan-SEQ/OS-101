# Windows File Systems: NTFS and ReFS

## NTFS: The Foundation of Windows Storage

**NTFS (New Technology File System)** has been the default Windows filesystem since NT 3.1 in 1993. It replaced FAT (File Allocation Table) for the same reasons ext4 replaced ext2 -- the need for journaling, permissions, large file support, and reliability. If you work with Windows at any level beyond casual use, you will encounter NTFS.

Where Linux offers a choice of filesystems (ext4, XFS, Btrfs, ZFS), Windows has essentially standardized on NTFS for three decades. This is partly by design (one filesystem to optimize) and partly by ecosystem lock-in (applications depend on NTFS-specific features like Alternate Data Streams and ACLs).

## The MFT: The Heart of NTFS

The **Master File Table (MFT)** is THE central data structure in NTFS. Every file and directory on an NTFS volume has at least one entry (record) in the MFT. This is fundamentally different from ext4's distributed inode tables.

The MFT is like a library card catalog where each card (MFT entry) contains not just the book's location, but for short books, the entire text is written on the card itself.

```
MFT Record (typically 1 KB)
+----------------------------------------------------------+
| Record Header                                             |
|   Sequence number, flags, reference count                 |
+----------------------------------------------------------+
| $STANDARD_INFORMATION attribute                           |
|   Creation time, modification time, access time           |
|   Flags (hidden, system, archive, etc.)                   |
+----------------------------------------------------------+
| $FILE_NAME attribute                                      |
|   Short name (8.3) and long name                          |
|   Parent directory reference                              |
+----------------------------------------------------------+
| $SECURITY_DESCRIPTOR attribute                            |
|   Owner SID, DACL, SACL (or index into $Secure)          |
+----------------------------------------------------------+
| $DATA attribute                                           |
|   For small files: actual file data (resident)            |
|   For large files: extent list (cluster runs)             |
+----------------------------------------------------------+
```

Key insight: **everything about a file is stored as attributes** in the MFT record. The file name is an attribute. The timestamps are an attribute. The security descriptor is an attribute. Even the file's data is an attribute called `$DATA`. This is an elegant, uniform design.

### Resident vs Non-Resident Attributes

If a file's data fits within the MFT record (roughly under 700 bytes after other attributes), the data is stored directly in the MFT record. This is called a **resident attribute**. Small configuration files, shortcuts, and tiny text files live entirely in the MFT with zero additional disk seeks.

For larger files, the `$DATA` attribute becomes **non-resident** and contains a list of **extent runs** (also called cluster runs) -- essentially "data starts at cluster X, runs for Y clusters." This is similar to ext4's extent-based allocation.

```
Small file (resident):          Large file (non-resident):
+------------------+            +------------------+
| MFT Record       |            | MFT Record       |
| [header]         |            | [header]         |
| [attributes]     |            | [attributes]     |
| [DATA: "Hello!"] |            | [DATA: run list] |
+------------------+            |  cluster 100, 5  |
  Everything in one place       |  cluster 250, 3  |
                                +---+-----------+--+
                                    |           |
                                    v           v
                              +--------+  +--------+
                              |clusters|  |clusters|
                              |100-104 |  |250-252 |
                              +--------+  +--------+
```

### Special MFT Files

The first 16 MFT records are reserved for NTFS metadata files:

| MFT Record | Name | Purpose |
|---|---|---|
| 0 | $MFT | The MFT itself (self-referential) |
| 1 | $MFTMirr | Mirror of first 4 MFT records (redundancy) |
| 2 | $LogFile | Transaction log (journaling) |
| 3 | $Volume | Volume name, NTFS version, flags |
| 4 | $AttrDef | Attribute definitions |
| 5 | . (root dir) | Root directory index |
| 6 | $Bitmap | Cluster allocation bitmap |
| 7 | $Boot | Boot sector and bootstrap code |
| 8 | $BadClus | Bad cluster tracking |
| 9 | $Secure | Security descriptor index |
| 10 | $UpCase | Unicode uppercase mapping table |
| 11 | $Extend | Extensions directory ($Quota, $ObjId, $Reparse, $UsnJrnl) |

## NTFS Features Deep Dive

### Journaling ($LogFile)

NTFS journals **metadata changes** (not file data by default) to `$LogFile`. Before any metadata operation (rename, permission change, directory update), the operation is logged. If the system crashes mid-operation, NTFS replays or undoes the log entries during the next mount.

This is the same concept as ext4's journal, with similar tradeoffs: metadata integrity is guaranteed, but file data written just before a crash may be lost unless the application uses flush/fsync explicitly.

### Alternate Data Streams (ADS)

Every NTFS file can have **multiple named data streams**. The default, unnamed stream is what you normally read/write. Named streams are hidden parallel data channels.

```
report.txt                     <-- default stream (your file content)
report.txt:summary             <-- alternate stream named "summary"
report.txt:Zone.Identifier     <-- stream added by the browser
```

You can write to alternate streams from the command line:
```
echo "hidden data" > file.txt:secret
more < file.txt:secret
```

The `Zone.Identifier` stream is how Windows tracks where a file was downloaded from (Internet, Intranet, Local). This is the "this file was downloaded from the Internet" warning dialog.

**Security concern**: Malware can store payloads in alternate data streams. The file looks normal in Explorer (file size only reflects the default stream), but an ADS can contain executable code. Modern antivirus scanners check ADS, but it remains a known hiding technique.

Linux has no direct equivalent. Extended attributes (`xattr`) provide similar metadata-attachment capabilities but are limited in size and are not arbitrary data streams.

### Links and Junctions

| Link Type | Description | Linux Equivalent |
|---|---|---|
| Hard link | Multiple names for the same MFT record | Hard link (same inode) |
| Symbolic link | Points to another path (file or directory) | Symlink |
| Junction point | Directory symbolic link (NTFS-specific, local only) | Symlink to directory |

Hard links in NTFS work the same way as Linux: multiple directory entries point to the same MFT record. Deleting one name does not delete the file until the last link is removed. The limitation is the same too -- hard links cannot cross volume boundaries.

### Compression

NTFS supports **per-file and per-directory transparent compression**. Compressed files are read and written normally by applications -- the filesystem handles compression/decompression transparently.

Compression works on 16-cluster units (usually 64 KB). If a compression unit compresses to fewer clusters than the original, the saved clusters become sparse. If it does not compress well, it is stored uncompressed.

The tradeoff: compressed files fragment more easily and have higher CPU overhead. This is why NTFS compression is rarely used for performance-sensitive workloads.

### Encryption (EFS)

**EFS (Encrypting File System)** provides per-file encryption integrated into NTFS. Each file is encrypted with a random FEK (File Encryption Key), and the FEK is encrypted with the user's public key. This means files are automatically decrypted when the owning user accesses them and remain encrypted for everyone else.

EFS is distinct from BitLocker, which encrypts entire volumes at the block level. EFS encrypts individual files at the filesystem level.

### Sparse Files

A sparse file has large regions of zeros that are not actually stored on disk. The MFT records which ranges are zero (sparse) and only stores non-zero data. This is useful for database files, virtual disk images, and log files that pre-allocate space.

Linux also supports sparse files -- `lseek` past the end and write creates a sparse region, and `fallocate` with `FALLOC_FL_PUNCH_HOLE` can create holes in existing files.

### Disk Quotas

NTFS supports per-user storage quotas, tracking how much disk space each user (by SID) consumes. Administrators can set warning levels and hard limits. This is equivalent to Linux's `quota` system for ext4/XFS.

## NTFS Permissions: Granular Access Control

NTFS permissions are significantly more granular than traditional Unix permissions. Where Unix has a 9-bit permission model (rwx for owner/group/others), NTFS uses full Access Control Lists (ACLs).

Each file or directory has a **security descriptor** containing a DACL (Discretionary Access Control List). The DACL is an ordered list of ACEs (Access Control Entries):

```
File: C:\Projects\report.docx

DACL:
+-------+----------+---------------------------+
| Type  | SID      | Permissions               |
+-------+----------+---------------------------+
| Allow | Alice    | Read, Write, Execute      |
| Allow | Dev Team | Read                      |
| Deny  | Bob      | Write                     |
| Allow | Everyone | Read                      |
+-------+----------+---------------------------+
```

NTFS permissions include: Full Control, Modify, Read & Execute, List Folder Contents, Read, Write, and many special permissions (Delete, Change Permissions, Take Ownership, Create Files, Create Folders, etc.).

**Inheritance**: Permissions set on a parent directory automatically propagate to children. An ACE can be marked as "this folder only," "this folder and subfolders," "files only," etc. This is far more flexible than Linux's umask-based default permission model.

**Effective permissions**: When multiple ACEs apply to a user (through group membership), the effective permission is the union of all Allow ACEs minus all Deny ACEs. Deny always wins over Allow when they conflict.

## ReFS: Resilient File System

**ReFS (Resilient File System)** was introduced in Windows Server 2012 as a next-generation filesystem for data-integrity-critical workloads. Think of it as Microsoft's answer to ZFS and Btrfs.

| Feature | NTFS | ReFS |
|---|---|---|
| Checksums | No | Metadata always, data optionally |
| Copy-on-write | No | Yes (integrity streams) |
| Compression | Yes | No |
| Encryption (EFS) | Yes | No |
| Disk quotas | Yes | No |
| Alternate Data Streams | Yes | Limited |
| Max volume size | 256 TB (practical) | 35 PB (practical) |
| Boot support | Yes | No (cannot be the OS volume) |
| Self-healing | No | Yes (with Storage Spaces) |
| Block cloning | No | Yes (fast VM copy) |

ReFS uses **copy-on-write (COW)** for metadata and optionally for data. When you modify a file, ReFS writes the new data to a new location and then atomically updates the metadata to point to the new location. This means the old data is never overwritten -- if a crash occurs mid-write, the old data is still intact.

When paired with **Storage Spaces** (Windows' software-defined storage / RAID), ReFS can detect and repair data corruption using checksums and redundant copies. This is conceptually identical to ZFS's scrub and repair capability.

ReFS is NOT a replacement for NTFS on desktops. It cannot be the boot volume, lacks many NTFS features, and is primarily used for:
- Storage Spaces Direct (hyper-converged infrastructure)
- Hyper-V virtual disk storage (fast block cloning for VM checkpoints)
- Backup targets (integrity protection for archived data)

## Filter Manager and Minifilters

One of the most architecturally interesting parts of Windows file I/O is the **Filter Manager**. It is like a stack of inspectors at a checkpoint -- every file operation passes through them, and each filter can inspect, modify, or block the operation.

```
Application
    |
    v  CreateFile("report.docx")
+------------------------------------+
|         Filter Manager              |
|  +------------------------------+  |
|  | Antivirus minifilter         |  |  <-- Scans file for malware
|  +------------------------------+  |
|  | Encryption minifilter (EFS)  |  |  <-- Decrypts on read, encrypts on write
|  +------------------------------+  |
|  | Backup minifilter (VSS)      |  |  <-- Tracks changes for snapshots
|  +------------------------------+  |
|  | HSM minifilter               |  |  <-- Recalls files from archive storage
|  +------------------------------+  |
+------------------------------------+
    |
    v
NTFS filesystem driver
    |
    v
Disk driver
```

Each minifilter has an **altitude** (a number) that determines its position in the stack. Microsoft assigns altitude ranges to ensure filters load in the correct order (antivirus above encryption, encryption above backup, etc.).

This is architecturally different from Linux, which uses the **VFS (Virtual Filesystem Switch)** layer with filesystem-specific operations. Linux does have `fanotify` for file access monitoring and LSM hooks for security modules, but there is no equivalent of the minifilter stack's generality.

## Comparison: NTFS vs ext4 vs XFS

| Feature | NTFS | ext4 | XFS |
|---|---|---|---|
| Central structure | MFT (one entry per file) | Inode table (per block group) | Inode (per allocation group) |
| Small file optimization | Resident data in MFT | Inline data in inode | Inline data in inode |
| Extent-based | Yes (cluster runs) | Yes (extent tree) | Yes (B+ tree extents) |
| Journaling | Metadata ($LogFile) | Metadata + optional data | Metadata (log) |
| Max file size | 16 EB (theoretical) | 16 TB | 8 EB |
| Max volume size | 256 TB practical | 1 EB | 8 EB |
| Permissions | Full ACL per file | POSIX ACL (optional) + mode bits | POSIX ACL + mode bits |
| Alternate streams | Yes (ADS) | No | No |
| Compression | Per-file/directory | No (ext2 had it, removed) | No |
| Encryption | EFS (per-file) | fscrypt (per-file/dir) | fscrypt |
| Checksums | No | Metadata only | Metadata only |
| Online defrag | Yes | Yes (e4defrag) | Yes (xfs_fsr) |

## Real-World Connection

When a Windows system administrator troubleshoots slow file access, they check NTFS fragmentation (especially MFT fragmentation, which can devastate performance), review alternate data streams for unexpected content, and verify that NTFS permissions are not overly complex (deeply nested inheritance chains slow down access checks).

When a security engineer investigates a compromised system, they scan for alternate data streams on executables (hiding malicious payloads), check the $UsnJrnl (Update Sequence Number Journal) for evidence of file creation/deletion/renaming, and examine the $LogFile for recent metadata operations. These NTFS artifacts are forensic gold.

When a storage architect designs a Hyper-V cluster, they choose ReFS for VM storage (fast block cloning for checkpoints, integrity streams for corruption detection) but keep NTFS for the OS volume (boot support required).

## Interview Angle

**Q: How does NTFS differ from ext4 in its core design?**

A: The fundamental difference is the MFT (Master File Table) vs distributed inode tables. NTFS stores everything about a file as attributes in a single MFT record -- name, timestamps, security descriptor, and even the data itself for small files (resident attributes). ext4 uses separate inode tables distributed across block groups, with data in separate blocks. NTFS supports features that ext4 lacks: Alternate Data Streams (multiple data streams per file), built-in per-file encryption (EFS), and transparent compression. ext4 has simpler permissions (mode bits + optional POSIX ACLs) while NTFS uses full Windows ACLs with fine-grained inheritance. Both use extent-based allocation and metadata journaling.

**Q: What are Alternate Data Streams and why do they matter for security?**

A: ADS allow a file to have multiple named data streams beyond the default unnamed stream. Any user with write access can attach an ADS to a file. The file's apparent size in Explorer only reflects the default stream, making ADS invisible to casual inspection. The Zone.Identifier ADS is used legitimately by browsers to mark downloaded files. However, attackers can store malicious payloads in ADS -- an innocent-looking text file can have an executable hidden in an alternate stream. Security tools must explicitly enumerate and scan ADS to detect this. The `dir /r` command and Sysinternals Streams tool can reveal ADS on files.

---

Next: [I/O Model and Driver Framework](05-windows-io-model-and-driver-framework.md)
