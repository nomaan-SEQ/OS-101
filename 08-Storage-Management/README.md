# Storage Management

Storage management is the physical layer of the I/O stack — the interface between the operating system and persistent storage hardware like hard drives, SSDs, and NVMe devices. While memory management deals with volatile RAM, storage management handles the slower, larger, non-volatile tier where your data actually lives. The OS must abstract away wildly different hardware technologies (spinning platters vs. flash cells) behind a uniform block device interface, schedule I/O requests efficiently, and work with redundancy schemes that keep data safe when hardware inevitably fails.

## Why This Matters

- **Database performance** lives and dies by storage I/O. Understanding disk structure, scheduling, and RAID directly explains why your queries are slow or fast.
- **Cloud infrastructure decisions** — choosing between gp3, io2, and local NVMe on AWS requires understanding IOPS, latency, and throughput tradeoffs.
- **System administration** — picking the right Linux I/O scheduler, configuring RAID arrays, and enabling TRIM are daily ops tasks.
- **Interview favorite** — disk scheduling algorithms, RAID levels, and HDD vs SSD tradeoffs appear constantly in systems design and OS interviews.

## Prerequisites

| Topic | Why You Need It |
|-------|----------------|
| [07-Memory-Management](../07-Memory-Management/README.md) | Swap space lives on storage devices. Demand paging triggers disk I/O. Understanding virtual memory's relationship to disk is essential context. |

## Reading Order

| # | File | Topic | Key Concept |
|---|------|-------|-------------|
| 1 | [01-disk-structure-hdd-ssd-nvme.md](01-disk-structure-hdd-ssd-nvme.md) | Disk Structure: HDD, SSD, NVMe | How storage hardware actually works and why the OS abstracts it all as blocks |
| 2 | [02-disk-scheduling-algorithms.md](02-disk-scheduling-algorithms.md) | Disk Scheduling Algorithms | FCFS, SSTF, SCAN, C-SCAN, LOOK — minimizing seek time on HDDs |
| 3 | [03-raid-levels.md](03-raid-levels.md) | RAID Levels | Combining disks for redundancy and performance — RAID 0/1/5/6/10 |
| 4 | [04-io-scheduling-in-linux.md](04-io-scheduling-in-linux.md) | I/O Scheduling in Linux | Real kernel schedulers — CFQ, Deadline, BFQ, mq-deadline, kyber |

## Key Interview Questions

1. **"Walk me through what happens when a process reads a file from an HDD vs an SSD."**
   Tests understanding of physical access mechanics — seek time and rotational latency for HDD vs page-level reads on NAND flash.

2. **"Why does SSTF disk scheduling cause starvation, and how does SCAN fix it?"**
   Classic algorithm comparison question. Shows you understand the fairness-throughput tradeoff.

3. **"Explain RAID 5. What happens when a disk fails? What about during rebuild?"**
   Tests depth on parity-based redundancy, XOR reconstruction, and the dangerous "degraded mode" rebuild window.

4. **"Which Linux I/O scheduler would you choose for a database server on NVMe? Why?"**
   Practical question bridging theory to real systems. Answer: `none` (or `mq-deadline`) — NVMe has no seek time, and its hardware queues handle scheduling internally.

5. **"Why do SSDs need garbage collection and TRIM? How does write amplification affect performance?"**
   Tests understanding of NAND flash constraints — erase-before-write at block granularity, and why the OS must cooperate with the drive's firmware.
