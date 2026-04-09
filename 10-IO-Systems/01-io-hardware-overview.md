# I/O Hardware Overview

Every device attached to your computer -- disk, keyboard, network card, GPU, USB stick -- connects through a layered hardware architecture. Understanding this architecture explains why some devices are fast and others are slow, why your GPU sits on PCIe while your mouse uses USB, and how the CPU actually talks to hardware.

## The Big Picture: CPU to Device

```
+-------+       +-----------+       +------------+       +--------+
|       |       |           |       |            |       |        |
|  CPU  +-------> System    +-------> Device     +-------> Device |
|       |       | Bus (PCIe)|       | Controller |       |        |
+-------+       +-----------+       +------------+       +--------+
                     |
              +------+------+
              |             |
         +---------+   +---------+
         | Memory  |   | Other   |
         | (RAM)   |   | Buses   |
         +---------+   | (USB,   |
                       |  SATA)  |
                       +---------+
```

The CPU never talks directly to a device. It talks to the **device controller** -- a piece of hardware (often a chip on the device or on the motherboard) that provides a standardized interface. The controller handles the messy details of the physical device.

## Device Controllers and Registers

Every device controller exposes a small set of **registers** that the CPU can read and write:

| Register | Purpose | Direction |
|----------|---------|-----------|
| **Status** | Is the device busy? Is there data ready? Any errors? | Device -> CPU (read) |
| **Command** | Tell the device what to do (read sector, send packet) | CPU -> Device (write) |
| **Data-in** | Data transferred from device to CPU | Device -> CPU (read) |
| **Data-out** | Data transferred from CPU to device | CPU -> Device (write) |

A typical I/O operation:
1. CPU reads **status register** -- is device ready?
2. CPU writes command to **command register** -- "read sector 42"
3. Device performs the work
4. CPU reads result from **data-in register** (or data arrives via DMA)

## Port-Mapped I/O vs Memory-Mapped I/O

The CPU needs a way to access those device registers. There are two approaches:

### Port-Mapped I/O (PMIO)

The CPU has **special I/O instructions** (`in` and `out` on x86) and a separate **I/O address space**.

```
CPU Instruction          What Happens
---------------------------------------------------
out 0x3F8, 'A'          Write 'A' to serial port
in  al, 0x60            Read from keyboard controller
```

- Separate address space from memory (16-bit, 0x0000 - 0xFFFF on x86)
- Requires privileged CPU instructions (kernel only)
- Legacy approach, still used for some devices (keyboard, serial ports)

### Memory-Mapped I/O (MMIO)

Device registers are **mapped into the regular memory address space**. The CPU accesses them with normal load/store instructions.

```
Memory Map (simplified):
+------------------+ 0xFFFFFFFF
|  Device Registers|  <-- GPU registers mapped here
|  (MMIO region)   |
+------------------+ 0xF0000000
|                  |
|  Regular RAM     |
|                  |
+------------------+ 0x00000000
```

- No special instructions needed -- just read/write memory addresses
- Works with any CPU architecture (not just x86)
- Can be accessed from C code with pointers: `*(volatile uint32_t*)0xF0000010 = cmd;`
- The dominant approach in modern hardware

| Feature | Port-Mapped I/O | Memory-Mapped I/O |
|---------|-----------------|-------------------|
| CPU instructions | Special (in/out) | Normal (load/store) |
| Address space | Separate I/O space | Shared with memory |
| Protection | Privileged instructions | Page-table based |
| Language support | Needs inline assembly | Works with pointers |
| Modern usage | Legacy devices | Almost everything |

## Bus Architecture and Bandwidth

Not all buses are equal. Devices are connected through a hierarchy based on speed requirements:

```
                    +-------+
                    |  CPU  |
                    +---+---+
                        |
            +-----------+-----------+
            |     System Bus        |  (Front Side Bus / Infinity Fabric)
            +-----------+-----------+
                        |
                  +-----+-----+
                  | PCH / I/O |
                  |   Hub      |
                  +-----+-----+
                   /    |    \
                  /     |     \
           +-----+ +---+---+ +------+
           | PCIe| |  SATA | |  USB |
           |slots| |  ports| | ports|
           +-----+ +-------+ +------+
           |       |         |
          GPU    SSD/HDD   Mouse
          NIC              Keyboard
          NVMe             Webcam
```

### Bandwidth Hierarchy

| Bus / Interface | Bandwidth | Typical Devices |
|-----------------|-----------|-----------------|
| PCIe 4.0 x16 | ~32 GB/s | GPUs, high-end NICs |
| PCIe 4.0 x4 | ~8 GB/s | NVMe SSDs |

**PCIe lanes** (the "x4", "x16" notation) refer to the number of parallel data links between the device and the CPU. Each lane is an independent send/receive pair. More lanes means proportionally more bandwidth: x4 has 4 lanes (8 GB/s on Gen 4), x16 has 16 lanes (32 GB/s on Gen 4). A device negotiates the number of lanes it needs based on its bandwidth requirements.
| PCIe 5.0 x4 | ~16 GB/s | Next-gen NVMe SSDs |
| SATA III | ~600 MB/s | SATA SSDs, HDDs |
| USB 3.2 Gen 2 | ~1.25 GB/s | External SSDs, peripherals |
| USB 2.0 | ~60 MB/s | Keyboards, mice, webcams |
| Gigabit Ethernet | ~125 MB/s | Network |
| 100GbE | ~12.5 GB/s | Data center networking |

This explains why:
- Your **GPU** sits on PCIe x16 -- it needs 32 GB/s to feed those shader cores
- An **NVMe SSD** uses PCIe x4 -- 8 GB/s is 13x faster than SATA
- Your **mouse** uses USB 2.0 -- 60 MB/s is overkill for click events
- A **USB thumb drive** feels slow -- USB bottlenecks the otherwise fast flash memory

## Character Devices vs Block Devices

The OS categorizes devices into two fundamental types:

| Property | Block Device | Character Device |
|----------|-------------|------------------|
| Data unit | Fixed-size blocks (512B, 4KB) | Byte stream |
| Random access | Yes (seek to any block) | No (sequential only) |
| Buffering | Kernel buffer cache | Minimal or none |
| Examples | Disks, SSDs, USB drives | Keyboard, mouse, serial port, terminal |
| Linux path | /dev/sda, /dev/nvme0n1 | /dev/tty, /dev/input/mice |

```bash
# List block devices
$ lsblk
NAME   MAJ:MIN SIZE TYPE MOUNTPOINT
sda      8:0  500G disk
+-sda1   8:1  499G part /
nvme0n1 259:0  1TB disk
+-nvme0n1p1    512M part /boot

# Character devices in /dev
$ ls -la /dev/tty0
crw--w---- 1 root tty 4, 0 Apr  1 10:00 /dev/tty0
#^ 'c' = character device

$ ls -la /dev/sda
brw-rw---- 1 root disk 8, 0 Apr  1 10:00 /dev/sda
#^ 'b' = block device
```

## Modern I/O: NVMe Queue Model

Traditional storage (SATA) used a single command queue with 32 entries. NVMe changed everything:

```
Traditional SATA:                    NVMe:
+------------------+                 +------------------+
| Single Queue     |                 | Queue Pair 1     | (CPU core 0)
| (32 commands max)|                 +------------------+
+------------------+                 | Queue Pair 2     | (CPU core 1)
        |                            +------------------+
   One at a time                     | Queue Pair 3     | (CPU core 2)
                                     +------------------+
                                     |  ... up to 64K   |
                                     | queues, 64K cmds |
                                     | each             |
                                     +------------------+
                                           |
                                      Parallel, per-core
```

- **64K queues** with **64K commands each** -- massive parallelism
- One queue pair per CPU core -- no locking between cores
- Submission queue + completion queue (ring buffers in memory)
- Commands go directly over PCIe -- no AHCI translation layer

This is why NVMe SSDs can hit millions of IOPS while SATA SSDs top out around 100K.

## Real-World Connection

**Why GPU needs PCIe x16**: Training a neural network means shuffling gigabytes of weight matrices and activations between CPU memory and GPU memory every second. PCIe x16 Gen 4 provides 32 GB/s -- and even that's a bottleneck (hence NVLink for multi-GPU setups).

**Why USB is terrible for storage**: A USB 2.0 flash drive has the same NAND flash as an internal SSD, but USB 2.0 caps it at 60 MB/s. Plug the same flash chips into an NVMe controller on PCIe and you get 7 GB/s. The bus is the bottleneck, not the storage medium.

**Cloud instance I/O**: When you pick an AWS EC2 instance type, you're choosing I/O characteristics. An `i3` instance has local NVMe SSDs (millions of IOPS). A `t3` gets network-attached EBS (limited IOPS). The hardware architecture directly maps to cloud pricing.

**Docker and block devices**: Containers use overlay filesystems (overlayfs) on top of block devices. Understanding block vs character devices matters when you need to pass devices into containers (`--device /dev/sda`).

## Interview Angle

**Q: What's the difference between port-mapped I/O and memory-mapped I/O?**

A: Port-mapped I/O uses special CPU instructions (in/out on x86) and a separate address space. Memory-mapped I/O maps device registers into the regular memory address space -- the CPU accesses them with normal load/store instructions. MMIO is the dominant modern approach because it works with any CPU architecture, doesn't need special instructions, and can be protected using normal page tables.

**Q: Why does NVMe outperform SATA SSDs so dramatically?**

A: Three reasons: (1) NVMe uses PCIe directly (8 GB/s for x4) vs SATA III (600 MB/s). (2) NVMe has up to 64K queues with 64K commands each vs SATA's single queue of 32 commands. (3) NVMe assigns one queue pair per CPU core, eliminating lock contention. The protocol was designed from scratch for flash storage, while SATA was designed for spinning disks.

**Q: What's the difference between a block device and a character device?**

A: Block devices transfer data in fixed-size blocks and support random access (seek to any position) -- think disks and SSDs. Character devices transfer data as a byte stream, sequentially -- think keyboards and serial ports. In Linux, block devices get a kernel buffer cache for performance; character devices typically don't.

---

**Next:** [Polling, Interrupts, and DMA](02-io-techniques-polling-interrupts-dma.md) -- the three fundamental ways the CPU communicates with devices.
