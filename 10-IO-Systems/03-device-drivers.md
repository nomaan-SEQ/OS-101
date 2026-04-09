# Device Drivers

Your OS needs to talk to thousands of different devices -- each with its own protocol, registers, quirks, and timing requirements. The OS can't have built-in knowledge of every device ever made. The solution: **device drivers** -- modular pieces of software that know how to talk to a specific device, presenting a standard interface to the rest of the OS.

## What is a Device Driver?

A driver is a **translator**. The OS speaks generic commands ("read 4KB from offset 1000") and the driver translates those into the specific register writes, DMA setups, and timing sequences that a particular device understands.

```
+-----------------+
|  Application    |   read(fd, buf, 4096)
+-----------------+
        |
+-----------------+
|  VFS / Kernel   |   "Read 4KB from inode 42, offset 1000"
|  I/O Subsystem  |   (generic, device-agnostic)
+-----------------+
        |
+-----------------+
|  Device Driver  |   "Write 0x25 to register 0x1F, set DMA
|  (e.g., NVMe)  |    source=LBA 200, dest=0xABCD0000, len=8"
+-----------------+   (device-specific commands)
        |
+-----------------+
|  Device HW      |   (executes the commands)
+-----------------+
```

Without drivers, the kernel would need to know the intimate details of every piece of hardware. With drivers, the kernel defines a **contract** (interface), and each driver implements it for its device.

## Driver Interfaces

The OS defines standard interfaces that drivers must implement. This is what makes the "everything is a file" abstraction work in Unix -- the kernel doesn't care what's behind the interface.

### Block Driver Interface

For devices that store data in fixed-size blocks (disks, SSDs):

```c
struct block_device_operations {
    int  (*open)(struct block_device *bdev);
    void (*release)(struct block_device *bdev);
    int  (*read_block)(sector_t sector, void *buffer, size_t count);
    int  (*write_block)(sector_t sector, void *buffer, size_t count);
    int  (*ioctl)(unsigned int cmd, unsigned long arg);
    // ... more operations
};
```

### Character Driver Interface

For devices that produce/consume byte streams (serial ports, terminals):

```c
struct file_operations {
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    int     (*open)(struct inode *, struct file *);
    int     (*release)(struct inode *, struct file *);
    long    (*ioctl)(struct file *, unsigned int, unsigned long);
    // ... more operations
};
```

### Network Driver Interface

For network devices (Ethernet, WiFi):

```c
struct net_device_ops {
    int  (*ndo_open)(struct net_device *dev);
    int  (*ndo_stop)(struct net_device *dev);
    int  (*ndo_start_xmit)(struct sk_buff *skb, struct net_device *dev);
    void (*ndo_set_rx_mode)(struct net_device *dev);
    int  (*ndo_set_mac_address)(struct net_device *dev, void *addr);
    // ... more operations
};
```

The beauty: **the kernel calls the same function signature regardless of whether it's an Intel NIC, a Broadcom NIC, or a virtual network device**. The driver handles the device-specific details.

## Driver Lifecycle

```
  +--------+     +------+     +---------+     +--------+
  | Probe  +---->| Init +---->| Operate +---->| Remove |
  +--------+     +------+     +---------+     +--------+

  "Is my       "Set up      "Handle I/O    "Clean up,
   device       resources,    requests,      free
   present?"    register      interrupts"    resources"
                with kernel"
```

| Phase | What Happens |
|-------|-------------|
| **Probe** | Kernel asks driver: "Can you handle this device?" Driver checks vendor/device IDs |
| **Init** | Allocate memory, map device registers, register interrupt handler, create device node |
| **Operate** | Handle read/write/ioctl calls, process interrupts, manage DMA transfers |
| **Remove** | Device unplugged or driver unloaded -- free memory, unregister handlers, release hardware |

## Why Drivers Run in Kernel Space

In monolithic kernels (Linux, Windows), drivers run inside the kernel with full privileges:

```
+------------------------------------------+
|              User Space                  |
|  +----------+  +----------+  +--------+ |
|  |  App 1   |  |  App 2   |  | App 3  | |
|  +----------+  +----------+  +--------+ |
+------------------------------------------+
|              Kernel Space                |
|  +--------+ +--------+ +--------+       |
|  | FS     | | Net    | | NVMe   |       |
|  | Driver | | Driver | | Driver |  <-- Drivers here
|  +--------+ +--------+ +--------+       |
|  +----------------------------------+   |
|  |     Core Kernel                  |   |
|  +----------------------------------+   |
+------------------------------------------+
|              Hardware                    |
+------------------------------------------+
```

**Why kernel space?**
- **Direct hardware access**: Drivers need to read/write device registers, program DMA, handle interrupts -- all require privileged CPU instructions
- **No mode-switch overhead**: Every user-kernel transition costs ~1000+ cycles. A driver in user space would need two transitions per I/O operation
- **Shared kernel memory**: Drivers can directly access kernel data structures (buffer cache, page tables) without copying

**The trade-off**: A buggy driver can crash the entire system. There's no memory protection between a driver and the kernel.

## Linux Driver Model: Automatic Device Detection

Modern Linux doesn't require manual driver loading. The system automatically detects hardware and loads the right driver:

```
Hardware plugged in
       |
       v
+----------------+
| Device Tree /  |  Hardware enumeration (PCIe, USB, ACPI)
| Bus Enumeration|  discovers vendor:device IDs
+----------------+
       |
       v
+----------------+
| udev           |  Userspace daemon receives hotplug events
| (device mgr)   |  Looks up matching driver module
+----------------+
       |
       v
+----------------+
| modprobe       |  Loads the kernel module (.ko file)
| (module loader)|  from /lib/modules/$(uname -r)/
+----------------+
       |
       v
+----------------+
| sysfs          |  Driver registers in /sys/
| (/sys/)        |  Device info exposed to userspace
+----------------+
```

```bash
# See loaded kernel modules (drivers)
$ lsmod
Module                  Size  Used by
nvme                   45056  3
nvme_core             110592  5 nvme
e1000e                282624  0
snd_hda_intel          57344  2

# See device info in sysfs
$ cat /sys/class/block/nvme0n1/device/model
Samsung SSD 980 PRO 1TB

# Manually load/unload a driver
$ sudo modprobe usb_storage   # load
$ sudo rmmod usb_storage      # unload
```

## User-Space Drivers

Sometimes you want drivers **outside** the kernel. Two frameworks enable this:

### UIO (Userspace I/O)

- Kernel provides a minimal driver that maps device memory to user space
- The real driver logic runs as a regular user process
- Used by: DPDK (high-speed networking), some industrial control systems

### VFIO (Virtual Function I/O)

- Safely exposes device access to user space with **IOMMU** (I/O Memory Management Unit) protection. The IOMMU is hardware that does for device DMA what the MMU does for CPU memory access -- it translates device-visible addresses to physical addresses and restricts which memory regions a device can access, preventing a misbehaving device from corrupting arbitrary memory.
- Used by: GPU passthrough in VMs, SR-IOV network cards
- Essential for giving a VM direct access to hardware

```
Traditional:                    User-space driver (DPDK):
App -> Kernel -> NIC Driver     App -> DPDK (user-space) -> NIC
     2 mode switches                 0 mode switches
     Kernel network stack            Bypass kernel entirely
     ~1M packets/sec                 ~100M packets/sec
```

**Why user-space drivers exist:**
- **Performance**: Skip the kernel network stack entirely (DPDK achieves 100M+ packets/sec)
- **Safety**: A crash in a user-space driver doesn't take down the kernel
- **Development speed**: Easier to debug, can use standard libraries, no kernel recompilation
- **Isolation**: Used for VM device passthrough (VFIO + IOMMU)

## Why Drivers Are the #1 Source of Kernel Bugs

Studies have consistently found that **device drivers account for 70%+ of kernel bugs**. Why?

| Problem | Explanation |
|---------|-------------|
| Vendor code quality | Hardware vendors aren't kernel developers -- driver code is often lower quality |
| No isolation | A null pointer dereference in a driver crashes the whole kernel |
| Hardware complexity | Devices have undocumented quirks, race conditions, errata |
| Concurrency bugs | Drivers handle interrupts (asynchronous) while also serving kernel requests |
| Testing difficulty | Need actual hardware to test properly, hard to cover all edge cases |
| Huge codebase | Linux has ~25 million lines of code; ~15 million are drivers |

This is the strongest argument for **microkernels** (like MINIX, Fuchsia): move drivers to user space where a crash just restarts the driver, not the whole OS. The counter-argument: the performance overhead of user-kernel transitions for every I/O operation.

## Real-World Connection

**NVIDIA drivers on Linux**: The most famous driver saga. NVIDIA provides a proprietary, closed-source kernel driver (can't be reviewed by kernel developers, doesn't follow kernel coding standards). It regularly breaks on kernel updates. The open-source alternative (nouveau) has limited performance. This is why Linus Torvalds famously expressed strong frustration with NVIDIA.

**Windows driver signing**: Windows requires drivers to be digitally signed by Microsoft. This prevents loading malicious drivers that could compromise the kernel. "Driver signing enforcement" is a security feature, not a restriction.

**Docker and device access**: By default, containers can't access host devices. You explicitly grant access with `--device /dev/nvidia0` or use the NVIDIA Container Toolkit. This is driver interaction crossing the container boundary.

**Cloud SR-IOV**: In AWS, the Elastic Network Adapter (ENA) uses SR-IOV to give VMs near-bare-metal network performance. The NIC creates "virtual functions" -- each VM gets its own virtual NIC with its own driver, bypassing the hypervisor for data plane operations.

**Why your printer doesn't work**: Consumer printers are notorious for bad drivers. The device is a $50 printer with a $0.50 controller, and the driver was written by the lowest bidder. On Linux, CUPS and the printer driver ecosystem have improved dramatically, but you still occasionally hit devices with no Linux driver at all.

## Interview Angle

**Q: Why do device drivers run in kernel space in Linux?**

A: Drivers need direct hardware access (device registers, DMA, interrupts), which requires kernel privilege level. Running in kernel space also avoids the overhead of user-kernel mode switches on every I/O operation. The downside is that a buggy driver can crash the entire system, since there's no memory protection between drivers and the kernel.

**Q: What's the advantage of a user-space driver like DPDK?**

A: DPDK bypasses the kernel network stack entirely, eliminating mode switches and kernel overhead. This lets it achieve 100M+ packets per second vs ~1M through the kernel stack. The trade-off is that you lose kernel networking features (firewall, routing, TCP stack) and must implement them yourself. User-space drivers are also safer -- a crash doesn't bring down the kernel.

**Q: How does Linux automatically load the right driver for a new device?**

A: When a device is detected (via PCIe enumeration, USB hotplug, etc.), the kernel identifies it by vendor:device ID. The udev daemon receives the hotplug event, looks up the matching kernel module, and loads it via modprobe. The driver's probe function confirms it can handle the device, then initializes it. All this happens automatically -- no manual intervention needed for supported hardware.

**Q: Why do drivers cause most kernel bugs?**

A: Drivers are often written by hardware vendors (not kernel developers), they run with full kernel privileges but no isolation, they handle complex asynchronous events (interrupts + requests), and they interact with hardware that has undocumented quirks. About 60% of Linux kernel code is drivers, and they account for 70%+ of bugs.

---

**Next:** [Blocking, Non-blocking, and Async I/O](04-blocking-nonblocking-async-io.md) -- I/O models from the application's perspective.
