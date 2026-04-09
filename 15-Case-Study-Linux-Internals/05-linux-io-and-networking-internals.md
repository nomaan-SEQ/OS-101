# Linux I/O and Networking Internals

## The Core Idea

I/O is where the kernel meets the physical world — disks, network interfaces, GPUs, USB devices. Linux has built sophisticated layers between user-space applications and hardware to maximize throughput, minimize latency, and support thousands of different devices.

This section covers two major I/O domains: **block I/O** (disk and storage) and **networking** (packets and connections). Both have evolved dramatically in recent years with multi-queue architectures and zero-copy interfaces like `io_uring` that fundamentally change how high-performance applications interact with the kernel.

---

## Block I/O Layer

When an application reads or writes a file, the request passes through VFS, through the filesystem, and eventually reaches the **block I/O layer** — the kernel subsystem that manages I/O requests to block devices (disks, SSDs, NVMe drives).

**The block I/O layer is like a mail sorting facility. Individual letters (bios) arrive from different senders (filesystems, swap, direct I/O). They get grouped into bags (requests) by destination (disk location), sorted for efficient delivery, then handed to the postal truck (device driver) for the final trip to the disk.**

### The bio Structure

The fundamental unit of block I/O is the `bio` (block I/O) structure. A `bio` represents a single I/O operation:

```
struct bio (simplified)
├── bi_bdev          Target block device
├── bi_iter          Starting sector, remaining size
├── bi_opf           Operation (READ, WRITE, FLUSH, DISCARD)
├── bi_vcnt          Number of page segments
└── bi_io_vec[]      Array of (page, offset, length) segments
    ├── [0]: page_A, offset=0,    len=4096
    ├── [1]: page_B, offset=0,    len=4096
    └── [2]: page_C, offset=2048, len=2048
```

A single `bio` can reference multiple non-contiguous pages in memory (gathered I/O), but the disk sectors it targets must be contiguous. Multiple `bio`s can be merged into a single **request** if they target adjacent disk sectors.

### The Request Path

```
Filesystem / Page Cache / Direct I/O
         │
         ▼
    submit_bio()          Submit a bio to the block layer
         │
         ▼
    Block Layer (blk-mq)
    ├── Merge with adjacent requests if possible
    ├── Apply I/O scheduler (mq-deadline, bfq, kyber, none)
    └── Place on hardware dispatch queue
         │
         ▼
    Device Driver          Convert request to hardware commands
    (NVMe, SCSI, virtio)  (NVMe submission queue entry, SCSI CDB, etc.)
         │
         ▼
    Hardware               Execute I/O, raise interrupt on completion
         │
         ▼
    Completion path        irq → softirq → complete bio → wake waiting process
```

### blk-mq: Multi-Queue Block Layer

Traditional Linux had a single request queue per block device, protected by a single lock. On modern systems with NVMe drives (which have 64+ hardware queues) and multi-core CPUs, this single lock became a catastrophic bottleneck.

blk-mq (multi-queue block layer, default since kernel 3.16) provides:
- **Per-CPU software staging queues** — each CPU submits I/O to its own queue, eliminating cross-CPU lock contention
- **Hardware dispatch queues** — mapped to the device's actual hardware queues (NVMe drives may have one per CPU)
- **No I/O scheduler overhead when unnecessary** — NVMe drives do their own internal scheduling, so Linux can use `none` (passthrough)

```
CPU 0  ──► SW Queue 0 ──┐
CPU 1  ──► SW Queue 1 ──┤──► HW Queue 0 ──► NVMe SQ 0
CPU 2  ──► SW Queue 2 ──┤──► HW Queue 1 ──► NVMe SQ 1
CPU 3  ──► SW Queue 3 ──┘──► HW Queue 2 ──► NVMe SQ 2
                         ...
```

I/O schedulers in the blk-mq era:

| Scheduler | Behavior | Best For |
|-----------|----------|----------|
| **none** | Passthrough, no reordering | NVMe (device handles scheduling) |
| **mq-deadline** | Deadline-based, prevents starvation | Rotating disks, mixed workloads |
| **bfq** (Budget Fair Queueing) | Fair bandwidth + latency guarantees | Desktop, interactive workloads |
| **kyber** | Lightweight, targets latency percentiles | Fast SSDs with latency requirements |

---

## Device Mapper (dm)

The device mapper is a layer between block devices and filesystems that enables powerful storage transformations:

```
Filesystem (ext4, XFS)
       │
       ▼
┌──────────────────────┐
│   Device Mapper (dm)  │   Transforms block I/O
│                       │
│  dm-linear           │   Concatenate devices
│  dm-striped          │   Stripe across devices (RAID0-like)
│  dm-mirror           │   Mirror writes to multiple devices
│  dm-crypt (LUKS)     │   Encrypt all I/O (full disk encryption)
│  dm-thin             │   Thin provisioning (overcommit storage)
│  dm-cache            │   Use SSD as cache for HDD
│  dm-snapshot         │   Point-in-time snapshots
└──────────┬───────────┘
           │
           ▼
    Physical Block Devices (sda, nvme0n1, ...)
```

**LVM (Logical Volume Manager)** is built on device mapper. When you create an LVM logical volume, LVM configures dm-linear (or dm-striped) tables that map logical blocks to physical blocks across one or more physical volumes. This is how you can resize filesystems, add disks to a volume group, or create snapshots on a running system.

**LUKS (Linux Unified Key Setup)** uses dm-crypt to encrypt entire block devices. Every read decrypts blocks on the fly; every write encrypts before hitting disk. The encryption key is derived from your passphrase and stored (encrypted) in the LUKS header. Performance overhead on modern CPUs with AES-NI is typically under 5%.

---

## Networking Internals

Linux networking is a complex subsystem that handles everything from raw Ethernet frames to TCP connection management to packet filtering and traffic shaping.

### NAPI: Efficient Packet Processing

Early Linux networking was purely interrupt-driven: every incoming packet triggered an interrupt, which triggered packet processing. At 10 Gbps with minimum-size packets, that is over 14 million interrupts per second — the CPU spends all its time handling interrupts and gets no real work done (interrupt livelock).

**NAPI (New API) is like a doorbell. The first ring (interrupt) tells you someone is at the door. Then you open the door and let everyone in the line come in (polling mode) without ringing the bell again. Once the line is empty, you go back inside and re-enable the doorbell.**

```
Normal operation (low traffic):
  Packet arrives → Interrupt → Process packet → Wait for next interrupt

NAPI (high traffic):
  Packet arrives → Interrupt → Disable interrupts for this device
       │
       ▼
  Switch to polling mode:
  Poll device: process batch of packets (up to budget, e.g., 64)
       │
       ├── More packets? → Poll again (no interrupt needed)
       │
       └── No more packets? → Re-enable interrupts, wait
```

NAPI is critical for high-speed networking. At 10/25/100 Gbps, the interrupt-per-packet model is completely unworkable. NAPI bounds the interrupt rate regardless of packet rate.

### Socket Buffer (sk_buff)

The `sk_buff` (socket buffer) is THE packet data structure in Linux networking. Every packet passing through the network stack is wrapped in an sk_buff:

```
sk_buff (simplified)
├── head           Start of allocated buffer space
├── data           Start of current packet data (moves as headers are added/removed)
├── tail           End of current packet data
├── end            End of allocated buffer space
├── protocol       Packet protocol (ETH_P_IP, ETH_P_ARP, ...)
├── *dev           Network device this packet is associated with
├── transport_header   Offset to TCP/UDP header
├── network_header     Offset to IP header
├── mac_header         Offset to Ethernet header
└── *sk            Socket this packet belongs to (if any)

Buffer layout:
┌─────┬────────────┬──────────────┬──────────┬──────┐
│head │  headroom  │  packet data │ tailroom │  end │
│room │ (for adding│ (Eth+IP+TCP+ │(for adding│      │
│     │  headers)  │   payload)   │  trailers)│      │
└─────┴────────────┴──────────────┴──────────┴──────┘
      ↑            ↑              ↑
     head         data           tail
```

The headroom/tailroom design is clever: when a packet moves up the stack, the `data` pointer is advanced past each header (Ethernet, IP, TCP) without copying. When building an outgoing packet, headers are prepended by moving the `data` pointer backward into the headroom. Zero copies for header manipulation.

### Netfilter and iptables/nftables

Netfilter provides hook points throughout the networking stack where packet filtering and manipulation modules can be registered:

```
Incoming packet
    │
    ▼
┌────────────┐
│ PREROUTING │ ← Hook: NAT, mangle, raw
└─────┬──────┘
      │
      ▼
  Routing decision: for this host or forward?
      │                    │
      ▼                    ▼
┌───────────┐        ┌───────────┐
│   INPUT   │        │  FORWARD  │
│  (local   │        │ (routed   │
│ delivery) │        │  traffic) │
└─────┬─────┘        └─────┬─────┘
      │                    │
      ▼                    ▼
  Local process      ┌─────────────┐
      │               │ POSTROUTING │ ← Hook: SNAT, masquerade
      ▼               └──────┬──────┘
┌───────────┐                │
│  OUTPUT   │                ▼
│ (locally  │           Out to NIC
│ generated)│
└─────┬─────┘
      ▼
┌─────────────┐
│ POSTROUTING │
└──────┬──────┘
       ▼
  Out to NIC
```

**iptables** (legacy) and **nftables** (modern replacement) both use the Netfilter hooks. nftables replaces the separate iptables/ip6tables/arptables/ebtables tools with a single framework and uses a virtual machine (nf_tables) to evaluate rules more efficiently.

### Traffic Control (tc)

The `tc` subsystem provides queuing disciplines (qdiscs) that control how packets are scheduled for transmission:

| Qdisc | Purpose |
|-------|---------|
| **pfifo_fast** | Simple priority FIFO (legacy default) |
| **fq_codel** | Fair queuing + controlled delay (modern default, fights bufferbloat) |
| **htb** | Hierarchical Token Bucket (bandwidth allocation with borrowing) |
| **tbf** | Token Bucket Filter (simple rate limiting) |
| **netem** | Network emulator (add latency, loss, jitter for testing) |

---

## io_uring Deep Dive

`io_uring` (introduced in kernel 5.1) is the most significant Linux I/O interface innovation in decades. It provides truly asynchronous I/O with minimal syscall overhead.

**io_uring is like a shared inbox/outbox between your application and the kernel. You drop request slips into the submission box (SQ), the kernel picks them up, performs the I/O, and drops result slips into the completion box (CQ). You never need to walk to the kernel's office (make a syscall) for each request — you just check your boxes.**

### Architecture

```
User Space                          Kernel Space
┌─────────────────────┐            ┌──────────────────────┐
│  Application        │            │  io_uring core       │
│                     │            │                      │
│  SQ (Submission     │ ◄──────── │  Reads SQ entries    │
│   Queue Ring)       │  shared    │  Performs I/O ops    │
│  [SQE][SQE][SQE]   │  memory    │  Writes CQE to CQ   │
│                     │            │                      │
│  CQ (Completion     │ ──────►   │                      │
│   Queue Ring)       │  shared    │                      │
│  [CQE][CQE][CQE]   │  memory    │                      │
└─────────────────────┘            └──────────────────────┘

SQE = Submission Queue Entry (describes one I/O operation)
CQE = Completion Queue Entry (result of one I/O operation)
```

### How It Works

1. **Setup** (once): `io_uring_setup()` syscall creates the SQ and CQ rings in shared memory between user and kernel space.
2. **Submit** (no syscall needed): Application writes SQE entries directly into the shared SQ ring. If the kernel is polling (SQPOLL mode), it picks them up automatically.
3. **Complete** (no syscall needed): Application reads CQE entries from the shared CQ ring. The kernel writes them there upon I/O completion.
4. **Enter** (optional): `io_uring_enter()` syscall can be used to submit and/or wait, but in SQPOLL mode, even this is avoided.

### Performance Impact

| Aspect | Traditional (read/write) | AIO (legacy) | io_uring |
|--------|------------------------|-------------|----------|
| Syscalls per I/O | 1 (minimum) | 2 (submit + reap) | 0 (in SQPOLL mode) |
| Copy overhead | Buffer copy to/from user | Some | Zero-copy possible |
| Batching | No (one op per syscall) | Limited | Natural (fill SQ, submit batch) |
| Operation types | File I/O only | File I/O only | File I/O, networking, timers, polling, and more |
| Memory model | Kernel buffers | Complex | Shared ring buffers |

io_uring supports not just file I/O but also: `accept()`, `connect()`, `recv()`, `send()`, `timeout`, `poll`, `fallocate`, `openat`, `close`, `statx`, and many more. It is evolving toward being a universal async syscall interface.

### Linked Operations

io_uring supports chaining operations: "read this file, THEN send the data on this socket." The kernel executes the chain without returning to user space between steps. This is powerful for operations like splice-like file-to-socket transfers.

---

## Real-World Connection

**NVMe and blk-mq:** Modern NVMe SSDs expose 64 or more hardware command queues, each capable of 64K outstanding commands. The old single-queue block layer was a terrible match for this hardware. blk-mq maps software queues to hardware queues, enabling full parallelism. On a high-end NVMe drive, this can mean the difference between 500K IOPS (single queue) and 5M+ IOPS (multi-queue).

**LUKS encryption overhead:** Full-disk encryption with dm-crypt uses AES-XTS. On CPUs with AES-NI hardware acceleration (virtually all modern x86 CPUs), the overhead is typically 2-5% for sequential I/O. Without AES-NI, it can be 20-40%. This is why server CPU feature flags matter for encrypted workloads. Check with `grep aes /proc/cpuinfo`.

**io_uring in production:** High-performance databases and web servers are adopting io_uring. Seastar (the framework behind ScyllaDB) and io_uring-based networking in modern web servers can handle millions of operations per second from a single thread. Tokio (Rust async runtime) has experimental io_uring backends that show significant throughput improvements for I/O-heavy workloads.

---

## Interview Angle

**Q: What is blk-mq and why did Linux move to multi-queue block I/O?**

A: The traditional block layer had a single request queue per device, protected by one lock. With modern NVMe SSDs that have dozens of hardware queues and multi-core CPUs, this lock caused severe contention. blk-mq provides per-CPU software queues that feed into hardware dispatch queues, eliminating cross-CPU contention. This enables Linux to fully utilize the parallelism of modern storage hardware, achieving millions of IOPS on NVMe devices.

**Q: How does NAPI improve network performance compared to pure interrupt-driven I/O?**

A: Pure interrupt-driven networking triggers one interrupt per received packet. At high packet rates (millions per second on 10+ Gbps links), the CPU is overwhelmed by interrupt handling — a condition called livelock. NAPI uses a hybrid approach: the first packet triggers an interrupt, which switches the driver to polling mode. In polling mode, the driver processes packets in batches without further interrupts. When the backlog is drained, interrupts are re-enabled. This bounds interrupt overhead regardless of packet rate, preventing livelock.

**Q: Explain io_uring and why it is faster than traditional system calls for I/O.**

A: io_uring uses shared memory ring buffers between user space and kernel space. The application writes I/O requests (SQEs) into the submission ring and reads results (CQEs) from the completion ring. Because both rings are in shared memory, the kernel can pick up requests and deliver results without a syscall per operation. In SQPOLL mode, a kernel thread polls the submission ring continuously, meaning the application never makes a syscall at all for steady-state I/O. This eliminates the overhead of syscall entry/exit (user-to-kernel transitions, register saving, security checks) that accumulates to significant cost at millions of operations per second.

---

**Next**: [06-linux-security-selinux-apparmor.md](06-linux-security-selinux-apparmor.md)
