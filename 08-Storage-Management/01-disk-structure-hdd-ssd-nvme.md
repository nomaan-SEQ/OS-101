# Disk Structure: HDD, SSD, and NVMe

The OS needs to store and retrieve data on physical hardware. But the hardware technologies are radically different — spinning magnetic platters, silicon flash chips, and high-speed PCIe buses. The OS's job is to understand each technology's characteristics while presenting a uniform **block device** abstraction to everything above it.

---

## HDD (Hard Disk Drive)

HDDs store data magnetically on spinning platters. They've been the dominant storage technology for decades and still dominate in capacity-per-dollar for bulk storage.

### Physical Structure

```
              Spindle (motor)
                  |
    ============[=]============   <-- Platter (top surface)
    ============[=]============   <-- Platter (bottom surface)
    ============[=]============   <-- Platter (top surface)
    ============[=]============   <-- Platter (bottom surface)
                  |
                  
    Actuator arm with read/write heads
    =========>  [head]
    =========>  [head]
    =========>  [head]
    =========>  [head]
```

```
    Top view of a single platter:

    +-----------------------------------------+
    |    Track 0 (outermost)                  |
    |   +-----------------------------------+ |
    |   |  Track 1                          | |
    |   | +-------------------------------+ | |
    |   | |  Track 2                      | | |
    |   | | +---------------------------+ | | |
    |   | | |         ...               | | | |
    |   | | |     +-----------+         | | | |
    |   | | |     |  Spindle  |         | | | |
    |   | | |     +-----------+         | | | |
    |   | | |                           | | | |
    |   | | +---------------------------+ | | |
    |   | +-------------------------------+ | |
    |   +-----------------------------------+ |
    +-----------------------------------------+

    Each track is divided into sectors (typically 512B or 4KB).
    A cylinder = the same track number across all platters.
```

**Key terms:**
- **Platter** — circular magnetic disk, data on both surfaces
- **Track** — one concentric ring on a platter surface
- **Sector** — smallest addressable unit on a track (512 bytes or 4KB)
- **Cylinder** — all tracks at the same radius across all platters
- **Head** — read/write element, one per platter surface
- **Actuator arm** — moves all heads together to the target track

### HDD Access Time

```
Total Access Time = Seek Time + Rotational Latency + Transfer Time

Seek Time:         Time to move the arm to the correct track
                   ~5-10 ms (average), ~1 ms (adjacent track)

Rotational Latency: Time for the target sector to rotate under the head
                    Average = half a rotation
                    7200 RPM: 1 rotation = 60/7200 = 8.33 ms
                    Average rotational latency = ~4.17 ms

Transfer Time:     Time to read/write the data once positioned
                   Depends on data size and rotational speed
                   Typically ~0.01-0.1 ms for a single sector
```

**This is why sequential I/O crushes random I/O on HDDs.** Sequential reads hit consecutive sectors — no additional seek or rotation needed. Random reads require a new seek + rotation for every request. The difference can be 100x or more.

---

## SSD (Solid State Drive)

SSDs store data in NAND flash memory — no moving parts, no seek time, no rotational latency. But they come with their own set of constraints that the OS must understand.

### NAND Flash Architecture

```
    SSD Internal Architecture:

    +--------------------------------------------------+
    |  SSD Controller (FTL - Flash Translation Layer)   |
    +--------------------------------------------------+
          |          |          |          |
    +---------+ +---------+ +---------+ +---------+
    | Channel | | Channel | | Channel | | Channel |
    |    0    | |    1    | |    2    | |    3    |
    +---------+ +---------+ +---------+ +---------+
       |  |       |  |       |  |       |  |
     Chip Chip  Chip Chip  Chip Chip  Chip Chip
      |          |
    +--------+  Each chip contains:
    | Block  |  - Multiple blocks (128-512 per chip)
    |--------|
    | Page 0 |  - Each block contains pages (64-256 per block)
    | Page 1 |  - Page = smallest READ/WRITE unit (~4-16 KB)
    | Page 2 |  - Block = smallest ERASE unit (~256 KB - 4 MB)
    |  ...   |
    | Page N |
    +--------+
```

### NAND Cell Types

**P/E cycles** (Program/Erase cycles) measure how many times a NAND cell can be written and erased before it wears out and can no longer reliably hold data. Each erase degrades the cell's oxide layer slightly, so fewer P/E cycles means shorter drive lifespan.

| Type | Bits/Cell | Read Speed | Write Speed | Endurance (P/E Cycles) | Cost | Use Case |
|------|-----------|------------|-------------|----------------------|------|----------|
| SLC  | 1         | Fastest    | Fastest     | ~100,000             | $$$$  | Enterprise cache |
| MLC  | 2         | Fast       | Fast        | ~10,000              | $$$   | Enterprise storage |
| TLC  | 3         | Moderate   | Moderate    | ~3,000               | $$    | Consumer SSDs |
| QLC  | 4         | Slower     | Slower      | ~1,000               | $     | Read-heavy, bulk |

More bits per cell = cheaper per GB, but slower and wears out faster.

### The Erase-Before-Write Problem

Flash memory has an asymmetry that fundamentally shapes SSD design:

```
    Read:   Can read any page individually        (~25-100 microseconds)
    Write:  Can write to an EMPTY page             (~200-500 microseconds)
    Erase:  Must erase an ENTIRE BLOCK at once     (~1-5 milliseconds)
    
    You CANNOT overwrite a page in-place.
    To modify data, you must:
    
    1. Read the entire block into RAM
    2. Modify the target page(s) in RAM
    3. Erase the entire block on flash
    4. Write the modified block back
    
    This is expensive! So SSDs use a smarter approach...
```

### Flash Translation Layer (FTL)

The SSD controller maintains a **Flash Translation Layer (FTL)** — firmware inside the SSD controller that acts as a middleman, translating the logical block addresses the OS uses into the actual physical NAND page locations. This indirection is what lets the SSD write to new pages instead of overwriting in place. Instead of erasing and rewriting, it:

1. Writes modified data to a **new empty page**
2. Updates the mapping table
3. Marks the old page as **invalid** (stale)

This is why SSDs need **garbage collection** and **TRIM**.

### Garbage Collection and TRIM

```
    Before GC:
    +-------+-------+-------+-------+-------+-------+
    | Valid | Stale | Valid | Stale | Stale | Valid  |  Block X
    +-------+-------+-------+-------+-------+-------+

    Garbage Collection:
    1. Copy valid pages to a new block
    2. Erase the old block (now fully free)

    After GC:
    +-------+-------+-------+-------+-------+-------+
    | Valid | Valid | Valid | Empty | Empty | Empty  |  New Block
    +-------+-------+-------+-------+-------+-------+
    +-------+-------+-------+-------+-------+-------+
    | Free  | Free  | Free  | Free  | Free  | Free  |  Block X (erased)
    +-------+-------+-------+-------+-------+-------+
```

**TRIM** is a command the OS sends to the SSD: "I deleted these logical blocks — you can erase the underlying pages whenever convenient." Without TRIM, the SSD doesn't know which data the filesystem considers deleted, forcing it to preserve stale data longer and increasing write amplification.

### Write Amplification

**Write amplification** = (Amount physically written to NAND) / (Amount logically written by OS)

Ideally this ratio is 1.0, but garbage collection, wear leveling, and internal bookkeeping push it higher. A write amplification of 3x means for every 1 GB the OS writes, the SSD internally writes 3 GB — wearing out the flash 3x faster.

### Wear Leveling

NAND cells have limited program/erase cycles. Without intervention, frequently-written blocks would die while rarely-written blocks stay fresh. **Wear leveling** distributes writes evenly across all blocks to maximize drive lifespan.

---

## NVMe (Non-Volatile Memory Express)

NVMe isn't a storage medium — it's a **protocol** designed specifically for flash-based storage. It replaces the AHCI protocol (designed for spinning disks) with something that actually exploits SSD capabilities.

### Why NVMe Exists

```
    SATA SSD path:
    CPU <-> SATA Controller <-> AHCI Protocol <-> SSD
         PCIe -> SATA bridge      1 command queue
                                   32 commands deep

    NVMe SSD path:
    CPU <-> PCIe Bus <-> NVMe Protocol <-> SSD
         Direct connection       65,535 queues
                                 65,536 commands each
```

**AHCI was the bottleneck, not the SSD.** SATA's single command queue and protocol overhead limited SATA SSDs to ~550 MB/s — far below what the flash chips could deliver. NVMe removes this bottleneck.

### NVMe Advantages

| Feature | SATA/AHCI | NVMe |
|---------|-----------|------|
| Interface | SATA (6 Gbps max) | PCIe (Gen4: 64 Gbps, Gen5: 128 Gbps) |
| Command queues | 1 | Up to 65,535 |
| Queue depth | 32 | 65,536 per queue |
| Protocol overhead | Higher (designed for HDD) | Lower (designed for flash) |
| CPU cycles per I/O | ~6,000 | ~2,500 |

---

## Comparison: HDD vs SATA SSD vs NVMe SSD

| Metric | HDD (7200 RPM) | SATA SSD | NVMe SSD |
|--------|----------------|----------|----------|
| **Random Read IOPS** | 75-150 | 75,000-100,000 | 500,000-1,000,000+ |
| **Random Write IOPS** | 75-150 | 50,000-90,000 | 200,000-500,000+ |
| **Sequential Read** | 100-200 MB/s | 500-550 MB/s | 3,500-7,000+ MB/s |
| **Sequential Write** | 100-200 MB/s | 450-520 MB/s | 3,000-5,000+ MB/s |
| **Read Latency** | 5-15 ms | 25-100 us | 10-20 us |
| **Write Latency** | 5-15 ms | 200-500 us | 20-100 us |
| **Cost per GB** | $0.02-0.03 | $0.07-0.10 | $0.08-0.15 |
| **Endurance** | Mechanical wear | NAND P/E cycles | NAND P/E cycles |
| **Power (active)** | 6-10W | 2-4W | 5-8W |
| **Vibration sensitive** | Yes | No | No |

The gap between HDD and SSD random IOPS is **three to four orders of magnitude**. This single fact explains most storage architecture decisions.

---

## Block Device Abstraction

Regardless of whether the physical device is an HDD, SSD, or NVMe drive, the OS presents them all through a **block device** interface:

```
    Application
        |
    Filesystem (ext4, XFS, NTFS)
        |
    Block Layer (uniform interface)
        |
    +---+---+---+
    |   |   |   |
   HDD SSD NVMe
```

The block layer sees storage as a linear array of fixed-size blocks (typically 512 bytes or 4KB). It translates logical block addresses (LBAs) to physical locations using device-specific drivers. The filesystem doesn't need to know whether it's talking to spinning rust or flash silicon.

This abstraction is why you can swap an HDD for an SSD without reinstalling your OS — the block interface is the same.

---

## Real-World Connection

- **Databases love NVMe** because random read IOPS directly determine query throughput. PostgreSQL on NVMe can handle 10-50x more concurrent queries than on HDD.
- **Cloud storage tiers** map directly to this hardware spectrum. AWS EBS gp3 (general purpose SSD), io2 (provisioned IOPS SSD), and sc1 (cold HDD) offer different price-performance tradeoffs.
- **TRIM should be enabled** on any Linux system with SSDs (`fstrim` or `discard` mount option). Without it, SSD performance degrades over time as the drive runs out of pre-erased blocks.
- **Docker and containers** create heavy random write workloads (overlayfs metadata). Running Docker on HDD is painful — this is why SSD is the minimum recommendation for container hosts.
- **Write amplification** matters for database WAL (write-ahead log) workloads. High write amplification on QLC SSDs can cause unexpected performance drops and shorter drive life.

---

## Interview Angle

**Q: Why is random I/O so much slower than sequential I/O on an HDD?**

A: Random I/O requires a seek (moving the arm to a new track, ~5-10ms) and rotational latency (waiting for the sector, ~4ms) for every request. Sequential I/O pays this cost once, then reads consecutive sectors as the platter spins — the data streams in at the transfer rate. The ratio can be 100x or more. This is why defragmentation improves HDD performance: it makes logical sequences physically contiguous.

**Q: What is write amplification on an SSD, and why does it matter?**

A: Write amplification occurs because NAND flash can only erase at the block level (hundreds of pages) but writes at the page level. When the SSD needs to reclaim space, garbage collection must copy valid pages from a partially-stale block to a new location, then erase the old block. This means the SSD physically writes more data than the OS requested. It reduces performance and accelerates flash wear. TRIM helps by letting the SSD know about deleted data early, reducing unnecessary copying during GC.

**Q: Why can't you overwrite data in-place on an SSD?**

A: NAND flash physics: a cell can be programmed (written) from 1 to 0, but erasing (resetting to all 1s) can only happen at the block granularity. To change even one byte, you'd need to erase the entire block. The Flash Translation Layer avoids this by writing to a new page and updating a mapping table, treating the old page as stale.

---

Next: [02-disk-scheduling-algorithms.md](02-disk-scheduling-algorithms.md)
