# RAID Levels

RAID stands for **Redundant Array of Independent Disks** (originally "Inexpensive" Disks). The idea: combine multiple physical disks into a single logical unit to achieve better **performance**, **reliability**, or both. The OS (or a hardware RAID controller) presents this array as one disk to the filesystem.

RAID exists because individual disks fail. Given enough disks and enough time, failure is guaranteed. The question is: when a disk dies, do you lose everything?

---

## RAID 0 — Striping

Data is split ("striped") across multiple disks in chunks. No redundancy at all.

```
    Write: "ABCDEFGH" (8 chunks)

    Disk 0        Disk 1        Disk 2        Disk 3
    +------+      +------+      +------+      +------+
    |  A   |      |  B   |      |  C   |      |  D   |
    +------+      +------+      +------+      +------+
    |  E   |      |  F   |      |  G   |      |  H   |
    +------+      +------+      +------+      +------+
```

| Property | Value |
|----------|-------|
| Min disks | 2 |
| Usable capacity | 100% (N disks) |
| Read speed | N x single disk |
| Write speed | N x single disk |
| Fault tolerance | **None** — any single disk failure destroys ALL data |

**Why it exists:** Raw speed. Striping lets you read/write in parallel across all disks. Great for scratch space, temporary data, or anything you can afford to lose.

**The danger:** With N disks, your probability of failure is N times higher than a single disk. RAID 0 with 4 disks is 4x more likely to suffer total data loss than 1 disk.

---

## RAID 1 — Mirroring

Every write goes to two (or more) identical copies. Maximum redundancy, simplest recovery.

```
    Write: "ABCD" (4 chunks)

    Disk 0 (primary)    Disk 1 (mirror)
    +------+            +------+
    |  A   |            |  A   |
    +------+            +------+
    |  B   |            |  B   |
    +------+            +------+
    |  C   |            |  C   |
    +------+            +------+
    |  D   |            |  D   |
    +------+            +------+
    
    Exact copies — either disk can serve any read.
```

| Property | Value |
|----------|-------|
| Min disks | 2 |
| Usable capacity | 50% (N/2) |
| Read speed | Up to 2x (reads from either disk) |
| Write speed | 1x (must write to both) |
| Fault tolerance | Survives 1 disk failure (in a 2-disk mirror) |

**Why it exists:** Simplicity and fast recovery. When a disk fails, the mirror is already a complete copy — no computation needed. Rebuild just copies data to the replacement disk.

---

## RAID 5 — Striping with Distributed Parity

Data is striped across 3+ disks with **parity** distributed across all disks. Parity allows reconstructing any single lost disk's data using XOR math.

```
    Write: "ABCDEFGHI" across 4 disks (3 data + 1 parity per stripe)

    Disk 0        Disk 1        Disk 2        Disk 3
    +------+      +------+      +------+      +------+
    |  A   |      |  B   |      |  C   |      | P(ABC)|  Stripe 0
    +------+      +------+      +------+      +------+
    |  D   |      |  E   |      | P(DEF)|     |  F   |  Stripe 1
    +------+      +------+      +------+      +------+
    |  G   |      | P(GHI)|     |  H   |      |  I   |  Stripe 2
    +------+      +------+      +------+      +------+
    
    P = parity block (XOR of data blocks in that stripe)
    Parity ROTATES across disks to avoid one disk being a bottleneck.
```

### How XOR Parity Works

```
    A = 1010
    B = 1100
    C = 0110
    
    P = A XOR B XOR C = 0000
    
    If disk with B dies:
    B = A XOR C XOR P = 1010 XOR 0110 XOR 0000 = 1100  (recovered!)
    
    XOR property: any one value can be recovered from the others.
```

| Property | Value |
|----------|-------|
| Min disks | 3 |
| Usable capacity | (N-1)/N (e.g., 75% with 4 disks) |
| Read speed | (N-1) x single disk |
| Write speed | Slower — every write requires read-modify-write of parity |
| Fault tolerance | Survives 1 disk failure |

### RAID 5 Write Penalty

Every small random write requires **4 operations**:

```
    To update block A on stripe 0:
    
    1. Read old A
    2. Read old parity P
    3. Compute new P = old_P XOR old_A XOR new_A
    4. Write new A + write new P
    
    4 I/O operations for 1 logical write = "write penalty"
```

### The RAID 5 Rebuild Danger

When a disk fails, RAID 5 enters **degraded mode** — it can still serve reads by computing the missing disk's data from the remaining disks + parity. But:

- Performance drops severely (every read now requires reconstruction)
- A **second disk failure during rebuild = total data loss**
- With modern large disks (10+ TB), rebuild can take **12-24 hours**
- During those hours, the remaining disks are under heavy stress

This is why RAID 5 has fallen out of favor for large disks.

---

## RAID 6 — Double Distributed Parity

Like RAID 5, but with **two independent parity blocks** per stripe. Survives any two simultaneous disk failures.

```
    Disk 0        Disk 1        Disk 2        Disk 3        Disk 4
    +------+      +------+      +------+      +------+      +------+
    |  A   |      |  B   |      |  C   |      | P1   |      | P2   |
    +------+      +------+      +------+      +------+      +------+
    |  D   |      |  E   |      | P1   |      | P2   |      |  F   |
    +------+      +------+      +------+      +------+      +------+
    
    P1 = XOR parity, P2 = Reed-Solomon or other independent parity
    Both parity blocks rotate across disks.
```

| Property | Value |
|----------|-------|
| Min disks | 4 |
| Usable capacity | (N-2)/N (e.g., 60% with 5 disks) |
| Read speed | (N-2) x single disk |
| Write speed | Slower than RAID 5 (two parity blocks to update) |
| Fault tolerance | Survives **2 disk failures** |

RAID 6 is the safe choice for large arrays where rebuild time is measured in hours and a second failure during rebuild is a real risk.

---

## RAID 10 (1+0) — Mirror, Then Stripe

Combine RAID 1 (mirroring) and RAID 0 (striping). First mirror pairs of disks, then stripe across the mirrored pairs.

```
    RAID 10: 4 disks

           Stripe (RAID 0)
        /                    \
    Mirror (RAID 1)      Mirror (RAID 1)
    Disk 0   Disk 1      Disk 2   Disk 3
    +------+ +------+    +------+ +------+
    |  A   | |  A   |    |  B   | |  B   |
    +------+ +------+    +------+ +------+
    |  C   | |  C   |    |  D   | |  D   |
    +------+ +------+    +------+ +------+
    |  E   | |  E   |    |  F   | |  F   |
    +------+ +------+    +------+ +------+
    
    Data is striped (A,B across pairs), each stripe is mirrored.
```

| Property | Value |
|----------|-------|
| Min disks | 4 (must be even) |
| Usable capacity | 50% (N/2) |
| Read speed | N x single disk (reads from any mirror) |
| Write speed | (N/2) x single disk (write to both mirrors in each pair) |
| Fault tolerance | Survives 1 failure per mirror pair (up to N/2 failures if lucky) |

**Why databases love RAID 10:** Best combination of performance and reliability. No parity computation overhead on writes (just mirror). Fast rebuilds (only need to copy from the surviving mirror). The 50% capacity cost is worth it for production database servers.

---

## Master Comparison Table

| RAID Level | Min Disks | Usable Capacity | Read Speed | Write Speed | Fault Tolerance | Best For |
|------------|-----------|-----------------|------------|-------------|-----------------|----------|
| **0** | 2 | 100% | Excellent | Excellent | None | Scratch/temp data |
| **1** | 2 | 50% | Good | OK | 1 disk | Boot drives, OS |
| **5** | 3 | (N-1)/N | Good | Moderate | 1 disk | General file servers |
| **6** | 4 | (N-2)/N | Good | Slower | 2 disks | Large archives |
| **10** | 4 | 50% | Excellent | Good | 1 per mirror | Databases, high I/O |

---

## Hardware RAID vs Software RAID

| Aspect | Hardware RAID | Software RAID |
|--------|--------------|---------------|
| **Implementation** | Dedicated controller card with its own CPU/RAM | OS kernel handles RAID logic |
| **CPU overhead** | None (offloaded to controller) | Uses host CPU (minimal on modern hardware) |
| **Cost** | $200-2000+ for enterprise controllers | Free (included in OS) |
| **Battery backup** | Often includes BBU for write cache | No (but filesystem journaling helps) |
| **OS independence** | Works before OS boots | Requires OS driver |
| **Flexibility** | Locked to vendor/model | Portable, open source |
| **Examples** | Dell PERC, HP Smart Array | Linux md, ZFS, Windows Storage Spaces |

**Modern trend:** Software RAID has won for most use cases. Linux `mdadm` handles RAID reliably, and ZFS provides RAID-like functionality (RAIDZ, RAIDZ2) with integrated filesystem features. Hardware RAID is still used in enterprise SANs and legacy environments.

---

## Real-World Connection

- **Database servers** almost universally use RAID 10 for data disks. The write performance (no parity penalty) and fast rebuild make it the standard choice for MySQL, PostgreSQL, and Oracle production systems.
- **NAS appliances** (Synology, QNAP) default to RAID 5 or RAID 6 for home/office file storage — good capacity efficiency with protection against drive failures.
- **Cloud providers** handle redundancy internally. AWS EBS replicates data across an Availability Zone. You typically don't configure RAID in the cloud — the provider abstracts it away. Exception: instance store NVMe drives on EC2 have no redundancy, so you might RAID 1 them.
- **ZFS** combines RAID, filesystem, and volume management. RAIDZ1 = RAID 5 equivalent, RAIDZ2 = RAID 6 equivalent, but with additional integrity features like checksumming.
- **Never use RAID as a backup.** RAID protects against hardware failure, not against accidental deletion, ransomware, or corruption. You need proper backups regardless of RAID level.

---

## Interview Angle

**Q: Compare RAID 5 and RAID 10. When would you choose each?**

A: RAID 5 uses distributed parity to survive one disk failure with only one disk's worth of capacity overhead — making it capacity-efficient (75% usable with 4 disks vs 50% for RAID 10). However, RAID 5 has a write penalty (4 I/O operations per logical write due to parity computation) and a dangerous rebuild window where a second failure means total loss. RAID 10 mirrors then stripes — no parity overhead on writes, fast rebuilds (just copy from the mirror), and can survive multiple failures if they're in different mirror pairs. I'd choose RAID 10 for write-heavy workloads like databases, and RAID 5 (or RAID 6) for read-heavy storage where capacity matters more than write performance.

**Q: What happens when a disk fails in RAID 5?**

A: The array enters degraded mode. It can still serve all reads and writes — the controller reconstructs the missing disk's data on-the-fly using XOR across the surviving disks and parity. Performance drops because every read now involves computation. A hot spare (if configured) kicks in and the rebuild process starts, copying reconstructed data to the replacement disk. The dangerous period is during rebuild — if a second disk fails before rebuild completes, all data is lost. With large modern disks, rebuild can take 12-24+ hours.

**Q: Why do people say "RAID is not a backup"?**

A: RAID protects against a specific failure mode: disk hardware failure. It does not protect against accidental deletion (all copies are deleted simultaneously), filesystem corruption (corruption is mirrored/striped like normal data), ransomware (encrypted data replaces good data on all disks), fire/theft (all disks are in the same machine), or software bugs. A proper backup strategy requires separate copies on separate media in separate locations, with versioning to recover from logical errors.

---

Next: [04-io-scheduling-in-linux.md](04-io-scheduling-in-linux.md)
