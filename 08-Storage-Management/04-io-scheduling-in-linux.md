# I/O Scheduling in Linux

The Linux kernel sits between userspace applications and block devices. When processes issue read/write requests, the **I/O scheduler** (also called the elevator) decides the order those requests reach the disk. This is where the theory from disk scheduling algorithms meets real, running systems.

Linux I/O scheduling has gone through a major architectural shift: from a **single-queue** model to a **multi-queue** model. Understanding both eras helps you make sense of documentation, blog posts, and system configurations you'll encounter in the wild.

---

## The Block I/O Path

```
    Application
        |
    Filesystem (ext4, XFS, btrfs)
        |
    Page Cache (in RAM)
        |
    Block Layer
        |
    +-------------------+
    |   I/O Scheduler    |  <-- Reorders, merges, prioritizes requests
    +-------------------+
        |
    Device Driver
        |
    Hardware (HDD / SSD / NVMe)
```

The I/O scheduler operates after the filesystem and page cache but before the device driver. It sees a stream of block I/O requests and can:
- **Merge** adjacent requests into larger ones (fewer I/O operations)
- **Sort** requests by sector to minimize seek time
- **Prioritize** certain requests (e.g., reads over writes, real-time over background)

---

## Single-Queue Era (Legacy, pre-kernel 5.0)

Before kernel 3.13, Linux had a single request queue per block device. All CPUs funneled requests through one queue, protected by one lock. Three schedulers competed for this slot:

### CFQ (Completely Fair Queuing)

```
    CFQ Architecture:
    
    Process A ──> [Queue A] ──┐
    Process B ──> [Queue B] ──┤
    Process C ──> [Queue C] ──┼──> Time-sliced dispatch ──> Disk
    Process D ──> [Queue D] ──┤
    Process E ──> [Queue E] ──┘
    
    Each process gets its own queue.
    CFQ round-robins between queues, giving each a time slice.
```

- **How it works:** Creates a separate queue per process (based on I/O context). Allocates time slices to each queue in round-robin fashion. Within each queue, requests are sorted by sector.
- **Strengths:** Fair — no single process can monopolize the disk. Good for desktop/interactive workloads.
- **Weaknesses:** Fairness overhead hurts raw throughput. Not optimal for SSDs.
- **Was the default** Linux scheduler for many years (kernel 2.6.18 through ~4.x).

### Deadline

```
    Deadline Architecture:
    
    Incoming requests
         |
         v
    +------------------+     +------------------+
    | Sorted Queue     |     | FIFO Queue       |
    | (by sector)      |     | (by arrival time)|
    +------------------+     +------------------+
    | ...sector 100... |     | req1 (age: 3ms)  |
    | ...sector 200... |     | req2 (age: 15ms) |
    | ...sector 350... |     | req3 (age: 480ms)|  <-- Approaching deadline!
    +------------------+     +------------------+
            |                        |
            v                        v
    Normally dispatch from      If any request exceeds
    sorted queue (seek          its deadline, dispatch
    optimized)                  it immediately
    
    Default deadlines: Read = 500ms, Write = 5000ms
```

- **How it works:** Maintains both a sector-sorted queue (for seek optimization) and a FIFO queue with expiration times. Normally serves from the sorted queue, but if any request approaches its deadline, it gets priority.
- **Strengths:** Bounded worst-case latency. Excellent for databases where you cannot tolerate multi-second I/O stalls.
- **Weaknesses:** Less fair than CFQ — doesn't consider which process owns the request.
- **Best for:** Database servers, latency-sensitive workloads on HDD.

### Noop

```
    Noop Architecture:
    
    Incoming requests ──> [Simple FIFO + merge] ──> Disk
    
    No sorting. No priority. Just merge adjacent requests and send them out.
```

- **How it works:** Basic FIFO with request merging. No seek optimization.
- **Strengths:** Minimal CPU overhead. Best choice for SSDs (no seek to optimize) and virtual machines (the hypervisor handles scheduling).
- **Weaknesses:** Terrible for HDDs (random seek patterns).
- **Best for:** SSDs, VMs with paravirtualized I/O.

---

## Multi-Queue Era (blk-mq, Modern Linux)

Starting with kernel 3.13 (became default in 5.0+), Linux introduced the **multi-queue block layer** (blk-mq). This was driven by NVMe SSDs that could handle millions of IOPS — the single-queue lock became the bottleneck, not the disk.

```
    Single-queue (old):                Multi-queue (blk-mq):
    
    CPU 0 ──┐                          CPU 0 ──> [Software Queue 0] ──> [HW Queue 0]
    CPU 1 ──┤                          CPU 1 ──> [Software Queue 1] ──> [HW Queue 1]
    CPU 2 ──┼─> [Single Queue] ──>     CPU 2 ──> [Software Queue 2] ──> [HW Queue 0]
    CPU 3 ──┤        LOCK              CPU 3 ──> [Software Queue 3] ──> [HW Queue 1]
    CPU 4 ──┘                          
                                       Per-CPU software queues, no global lock.
                                       Multiple hardware dispatch queues.
```

### mq-deadline

The multi-queue successor to the deadline scheduler. Same algorithm (sorted + FIFO with deadlines), but designed for the blk-mq infrastructure.

- **When to use:** HDDs, SATA SSDs — anywhere you want bounded latency with some seek optimization.
- **Default on** many distributions for SATA devices.

### BFQ (Budget Fair Queuing)

```
    BFQ assigns I/O "budgets" (in sectors) to processes:
    
    Process A: budget = 8000 sectors  (interactive app, high priority)
    Process B: budget = 2000 sectors  (background backup, low priority)
    Process C: budget = 5000 sectors  (normal process)
    
    Processes with larger budgets get more disk time.
    Interactive processes get automatic budget boosts.
```

- **How it works:** Assigns sector budgets to processes/groups. Processes spend their budget, then wait. Interactive processes get priority boosts. Supports cgroups for hierarchical I/O control.
- **Strengths:** Best fairness among blk-mq schedulers. Excellent for desktop systems — a large background copy won't make your UI stutter.
- **Weaknesses:** Higher CPU overhead than mq-deadline. Throughput slightly lower on server workloads.
- **Best for:** Desktop Linux, systems where interactive responsiveness matters.

### kyber

- **How it works:** Latency-aware scheduler that auto-tunes. Maintains target latencies for reads and writes, adjusting queue depths to meet them. Minimal configuration needed.
- **Strengths:** Self-tuning, low overhead. Good for fast devices where you want latency control without manual tuning.
- **Weaknesses:** Less widely tested than mq-deadline. Not ideal for HDDs.
- **Best for:** Fast SSDs where you want latency guarantees with minimal configuration.

### none (No Scheduler)

- **How it works:** Requests go directly to hardware queues with no reordering. Only basic merging.
- **Best for:** NVMe SSDs. The drive has 65,535 hardware queues and its own internal scheduler. Adding a software scheduler on top just adds latency and CPU overhead.

---

## Checking and Changing the Scheduler

```bash
# See current scheduler for a device (bracketed = active)
cat /sys/block/sda/queue/scheduler
# Output: [mq-deadline] kyber bfq none

# Change scheduler at runtime
echo "bfq" | sudo tee /sys/block/sda/queue/scheduler

# Check for NVMe device
cat /sys/block/nvme0n1/queue/scheduler
# Output: [none]

# Make persistent (varies by distro):
# Method 1: udev rule
# /etc/udev/rules.d/60-ioscheduler.rules
# ACTION=="add|change", KERNEL=="sd*", ATTR{queue/scheduler}="mq-deadline"
# ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"

# Method 2: kernel parameter
# Add to GRUB: elevator=mq-deadline
```

---

## Which Scheduler for Which Device

| Device Type | Recommended Scheduler | Why |
|-------------|----------------------|-----|
| HDD | mq-deadline or BFQ | Seek optimization + bounded latency (mq-deadline) or fairness (BFQ) |
| SATA SSD | mq-deadline | Light reordering + deadline guarantees. No seek to optimize, but queue management helps. |
| NVMe SSD | none | NVMe hardware has its own multi-queue scheduler. Software scheduling adds overhead for zero benefit. |
| Virtual disk (VM) | mq-deadline or none | The hypervisor handles physical scheduling. Keep it simple on the guest. |
| Desktop (any device) | BFQ | Prevents background I/O from starving interactive applications. |

---

## I/O Priorities with ionice

Linux supports three I/O scheduling classes. These work with CFQ and BFQ (not with mq-deadline or none).

```bash
# Check current I/O priority
ionice -p <PID>

# Set real-time I/O class (highest priority, class 1)
ionice -c 1 -n 0 <command>

# Set best-effort (normal, class 2, priority 0-7)
ionice -c 2 -n 4 <command>

# Set idle (only gets I/O when no one else needs it, class 3)
ionice -c 3 <command>

# Practical example: run backup at idle I/O priority
ionice -c 3 rsync -a /data /backup
```

| Class | Name | Priority Levels | Behavior |
|-------|------|----------------|----------|
| 1 | Real-time | 0-7 (0 = highest) | Always served first. Can starve other classes. Use carefully. |
| 2 | Best-effort | 0-7 (0 = highest) | Default class. Fair scheduling within the class. |
| 3 | Idle | None | Only gets I/O time when no other class has pending requests. |

---

## cgroups and I/O Control

For container and multi-tenant environments, Linux cgroups (v2) provide I/O bandwidth control:

```bash
# Set I/O weight for a cgroup (relative priority, 1-10000)
echo "100" > /sys/fs/cgroup/myapp/io.weight

# Set absolute I/O limits (bytes/sec and IOPS)
# Format: MAJOR:MINOR rbps=BYTES wbps=BYTES riops=NUM wiops=NUM
echo "8:0 rbps=50000000 wbps=10000000" > /sys/fs/cgroup/myapp/io.max

# This is how Docker/Kubernetes implement --blkio-weight and I/O limits
```

---

## Real-World Connection

- **Database tuning:** For PostgreSQL or MySQL on SATA SSD, switch from the default scheduler to `mq-deadline`. On NVMe, use `none`. This alone can improve p99 latency significantly.
- **Cloud VM storage:** AWS EBS volumes are network-attached and already queued/scheduled on the hypervisor side. Using `none` or `mq-deadline` in the guest is typical. Don't use BFQ in a server VM — the fairness overhead isn't worth it.
- **Container I/O isolation:** In Kubernetes, you can set I/O limits per pod using cgroups v2. Without limits, a noisy-neighbor container running a backup can saturate shared storage and slow every other container on the node.
- **Monitoring I/O:** `iostat -x 1` shows per-device I/O stats including `await` (average I/O latency), `%util` (device saturation), and `r/s` / `w/s` (IOPS). If `await` is high and `%util` is 100%, your device is the bottleneck.
- **Desktop responsiveness:** If your Linux desktop feels sluggish during large file copies, switching to BFQ scheduler can dramatically improve interactive responsiveness by giving GUI applications higher I/O priority.

---

## Interview Angle

**Q: What Linux I/O scheduler would you use for a database on NVMe, and why?**

A: `none` (the no-op scheduler). NVMe drives connect via PCIe with up to 65,535 hardware submission queues, each 65,536 entries deep. The drive's internal firmware handles request scheduling across its flash channels far more efficiently than any software scheduler can. Adding a software scheduler on top just burns CPU cycles and adds latency. For NVMe, the best thing the OS can do is get out of the way.

**Q: Explain the difference between the deadline and CFQ schedulers.**

A: CFQ creates per-process I/O queues and allocates time slices to each, ensuring fairness — no single process can monopolize the disk. It's great for desktops but the fairness mechanism adds overhead. Deadline maintains a sector-sorted queue for throughput plus a FIFO queue with expiration times (500ms for reads, 5s for writes). It guarantees no request waits longer than its deadline, making it ideal for database workloads where a single slow I/O request can stall a transaction. CFQ optimizes for fairness, deadline optimizes for bounded latency.

**Q: How would you diagnose a storage bottleneck on a Linux server?**

A: Start with `iostat -x 1` to check per-device metrics. High `%util` (near 100%) means the device is saturated. High `await` (average I/O time) with low `%util` suggests queuing at a higher level. Check `avgqu-sz` for queue depth — SSDs need high queue depth for peak performance. Then check the I/O scheduler (`cat /sys/block/sdX/queue/scheduler`) to ensure it matches the device type. Use `iotop` to identify which processes are generating the most I/O. For cgroup-limited containers, check `io.stat` to see if they're hitting their I/O ceiling.

**Q: What is ionice and when would you use it?**

A: `ionice` sets the I/O scheduling class and priority for a process. The three classes are real-time (highest, can starve others), best-effort (default, fair scheduling), and idle (only runs when no one else needs I/O). A classic use case: running `ionice -c 3 rsync -a /data /backup` to ensure a backup job never interferes with production database I/O. It only works with schedulers that support I/O priorities (BFQ, legacy CFQ), not with mq-deadline or none.
