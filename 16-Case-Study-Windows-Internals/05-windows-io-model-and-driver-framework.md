# Windows I/O Model and Driver Framework

## The IRP: The Universal I/O Currency

Every I/O operation in Windows -- file reads, network sends, device control commands, power state changes -- flows through an **IRP (I/O Request Packet)**. The IRP is the single most important data structure in the Windows I/O subsystem. Nothing moves without one.

An IRP is like a work order in a factory. It starts at the top (your read request), gets stamped and processed by each station (driver) on the way down to the hardware, and the results flow back up. Each station adds its own notes and passes the work order along.

```
Application calls ReadFile()
        |
        v
+------------------+
|   I/O Manager    |  Creates an IRP
+--------+---------+
         |
         v  IRP flows DOWN the device stack
+------------------+
| Filter Driver    |  (e.g., antivirus scans)
| (upper filter)   |
+--------+---------+
         |
         v
+------------------+
| Function Driver  |  (main driver for the device)
| (e.g., NTFS)     |
+--------+---------+
         |
         v
+------------------+
| Filter Driver    |  (e.g., disk encryption)
| (lower filter)   |
+--------+---------+
         |
         v
+------------------+
| Port/Miniport    |  (hardware-specific driver)
| Driver           |
+--------+---------+
         |
         v
      HARDWARE      (disk controller, NIC, etc.)
         |
         v  Results flow BACK UP the stack
   Completion routines called at each layer
         |
         v
   I/O Manager completes the IRP
         |
         v
   Application gets its data
```

### IRP Structure

An IRP contains:
- **Header**: the common fields (status, buffer pointers, completion info)
- **I/O Stack Locations**: one per driver in the device stack, each containing the parameters specific to that driver's role

```
IRP
+------------------------------------------+
| Header                                    |
|   IoStatus (status + bytes transferred)   |
|   UserBuffer / SystemBuffer / MdlAddress  |
|   RequestorMode (UserMode/KernelMode)     |
|   Cancel flag                             |
+------------------------------------------+
| I/O Stack Location [0] - Filter Driver    |
|   MajorFunction: IRP_MJ_READ             |
|   Parameters.Read.Length = 4096           |
+------------------------------------------+
| I/O Stack Location [1] - NTFS            |
|   MajorFunction: IRP_MJ_READ             |
|   Parameters specific to filesystem       |
+------------------------------------------+
| I/O Stack Location [2] - Disk Driver      |
|   MajorFunction: IRP_MJ_READ             |
|   Parameters specific to disk             |
+------------------------------------------+
```

The `MajorFunction` field determines what kind of operation this is:

| Major Function | Operation | Linux Equivalent |
|---|---|---|
| IRP_MJ_CREATE | Open a file/device | open() |
| IRP_MJ_READ | Read data | read() |
| IRP_MJ_WRITE | Write data | write() |
| IRP_MJ_CLOSE | Close a handle | close() |
| IRP_MJ_DEVICE_CONTROL | Device-specific command | ioctl() |
| IRP_MJ_CLEANUP | All handles closed | flush() |
| IRP_MJ_PNP | Plug and Play event | uevent |
| IRP_MJ_POWER | Power state change | PM callbacks |

In Linux, the equivalent concept is scattered across different subsystems: the VFS provides `file_operations` (read, write, ioctl), the block layer uses `struct bio` for block I/O, and the network stack uses `struct sk_buff` for packets. Windows unifies ALL of these under the IRP model. One structure, one dispatch mechanism, one completion model.

## The Device Stack: Layered Drivers

Windows drivers are organized into **device stacks**. When a device is detected, the Plug and Play Manager builds a stack of driver layers:

```
                    Device Stack for a Disk Volume
                    
+----------------------------------------------------------+
|  Upper Filter Driver Object                               |
|  (e.g., antivirus filesystem filter)                      |
+----------------------------------------------------------+
|  Function Driver Object (FDO)                             |
|  (e.g., NTFS - the main filesystem driver)                |
+----------------------------------------------------------+
|  Lower Filter Driver Object                               |
|  (e.g., BitLocker encryption driver)                      |
+----------------------------------------------------------+
|  Physical Device Object (PDO)                             |
|  (e.g., disk.sys - the partition driver)                  |
+----------------------------------------------------------+
|  Port/Miniport Driver                                     |
|  (e.g., storport.sys + vendor miniport)                   |
+----------------------------------------------------------+
           |
           v
        Hardware (SATA controller, NVMe, etc.)
```

Each layer in the stack has a **device object** and processes IRPs for its specific responsibility. Filter drivers (both upper and lower) can inspect, modify, or block IRPs as they flow through. This layered architecture is what makes it possible for antivirus software, encryption, and backup tools to transparently intercept I/O without modifying the filesystem or disk driver.

Linux achieves similar layering through different mechanisms: device mapper for block-level filtering, FUSE for user-space filesystems, and netfilter for network packet filtering. The approach is more ad-hoc -- each subsystem has its own interception mechanism rather than a universal IRP stack.

## Asynchronous I/O: The Default Design

Windows was designed from the ground up with asynchronous I/O as the primary model. This is arguably the most important architectural difference between Windows and Linux I/O.

In Linux, I/O calls are **synchronous by default**. `read()` blocks until data is available. You opt into async behavior with `select()`/`poll()`/`epoll()` (readiness notification) or `io_uring` (true async I/O).

In Windows, **every I/O can be async**. When you open a file with `FILE_FLAG_OVERLAPPED`, read and write operations return immediately. You are notified when they complete.

### Overlapped I/O

The basic async model is **overlapped I/O**: pass an `OVERLAPPED` structure to `ReadFile()`/`WriteFile()`, and the call returns immediately. You later check for completion via:
- **GetOverlappedResult**: poll for completion
- **Event signaling**: the OVERLAPPED contains an event that gets set on completion
- **APC (Asynchronous Procedure Call)**: a callback runs in the calling thread when I/O completes
- **I/O Completion Ports**: the high-performance production model

### I/O Completion Ports (IOCP)

**IOCP is THE high-performance async I/O mechanism in Windows.** It is the foundation of every serious Windows server: IIS, SQL Server, .NET Kestrel, and game servers all use IOCP.

IOCP is like a restaurant kitchen's order-up window. Waiters (threads) drop off orders (I/O requests) and go serve other tables. When food is ready (I/O completes), the bell rings and ANY available waiter picks it up. No waiter stands idle waiting for one specific order.

```
                    I/O COMPLETION PORT
                    
Thread Pool                              I/O Operations
+---------+                              +-----------+
|Thread 1 |---+                     +--->| File Read |
+---------+   |   +-----------+    |    +-----------+
              +-->|           |----+
+---------+   |   | Completion|         +-----------+
|Thread 2 |---+-->|   Port    |-------->| Socket Recv|
+---------+   |   |  (IOCP)  |         +-----------+
              |   |           |----+
+---------+   |   +-----------+    |    +-----------+
|Thread 3 |---+                    +--->| Pipe Read |
+---------+                             +-----------+

Threads call GetQueuedCompletionStatus() to wait.
When ANY I/O completes, ONE thread wakes up and handles it.
```

The critical design decisions in IOCP:

1. **Completion-based, not readiness-based**: IOCP tells you "this I/O is done, here is your data." Linux's epoll tells you "this socket is ready to read" -- you still have to call `read()`. IOCP eliminates that second syscall.

2. **Built-in thread pool management**: IOCP limits the number of concurrently-running threads to the number of CPUs. If a thread blocks (e.g., on a lock), IOCP releases another thread to keep all CPUs busy. This prevents both thread starvation and thread oversubscription.

3. **LIFO thread wake-up**: When a completion arrives, IOCP wakes the most recently sleeping thread. This improves cache locality because that thread's data is most likely still in the CPU cache.

4. **Scalability**: A single IOCP can handle hundreds of thousands of concurrent I/O operations with a thread pool sized to the number of CPUs. This is why Windows servers historically scaled well for high-connection-count workloads.

### IOCP vs epoll: The Fundamental Difference

| Aspect | Windows IOCP | Linux epoll |
|---|---|---|
| Model | Completion-based ("I/O done, here's data") | Readiness-based ("socket ready, call read()") |
| Buffer management | OS fills your buffer, returns it complete | You provide buffer after readiness notification |
| Thread model | Built-in concurrency limiting | You manage your own thread pool |
| Syscalls per I/O | 1 (GetQueuedCompletionStatus returns data) | 2 (epoll_wait + read) |
| File I/O | Works for files, sockets, pipes, devices | Sockets and pipes only (no regular files) |
| Equivalent latency | Similar for network I/O | Similar for network I/O |

Linux's **io_uring** (kernel 5.1+) is moving toward completion-based I/O, and in many ways is inspired by IOCP. io_uring supports true async file I/O (which epoll cannot do) and batches submissions/completions through shared ring buffers, potentially outperforming IOCP for many workloads.

## WDM and WDF: Driver Development Frameworks

### WDM (Windows Driver Model)

**WDM (Windows Driver Model)** is the legacy driver framework, introduced with Windows 2000. WDM drivers directly handle IRPs, manage device objects, and deal with power management and Plug and Play events manually. Writing a WDM driver is complex and error-prone -- you must handle dozens of IRP types, manage reference counts, and avoid deadlocks in a preemptible kernel.

Think of WDM as writing assembly language -- you have full control but also full responsibility.

### WDF (Windows Driver Framework)

**WDF (Windows Driver Framework)** is the modern replacement, providing a higher-level abstraction over WDM:

```
+---------------------------------------------------+
|  UMDF (User-Mode Driver Framework)                 |
|  Runs in user space, crashes don't BSOD            |
|  Good for: USB devices, sensors, simple HID         |
+---------------------------------------------------+
|  KMDF (Kernel-Mode Driver Framework)               |
|  Runs in kernel, but WDF handles boilerplate        |
|  Good for: storage, network, complex devices         |
+---------------------------------------------------+
|  WDM (Windows Driver Model)                        |
|  Raw kernel driver, full control                    |
|  Good for: filesystem filters, virtualization        |
+---------------------------------------------------+
|  Kernel (ntoskrnl.exe)                              |
+---------------------------------------------------+
```

**KMDF (Kernel-Mode Driver Framework)**: Handles most boilerplate -- PnP state machine, power management, IRP queuing and dispatching. The driver author writes callbacks for device-specific operations. This is similar to how Linux's driver core and subsystem frameworks (USB core, PCI subsystem) simplify driver development.

**UMDF (User-Mode Driver Framework)**: The driver runs as a user-mode process. If it crashes, the process restarts -- no BSOD (Blue Screen of Death). This is conceptually similar to running drivers in user space with FUSE (for filesystems) or DPDK (for network drivers), though UMDF is more general. UMDF is used for devices where performance is less critical: printers, USB accessories, sensors.

| Framework | Ring Level | Crash Impact | Complexity | Use Cases |
|---|---|---|---|---|
| WDM | Kernel (ring 0) | BSOD | Very high | FS filters, virtualization |
| KMDF | Kernel (ring 0) | BSOD | Medium | Storage, network, complex HW |
| UMDF | User (ring 3) | Process restart | Low | USB, sensors, printers |

## Plug and Play Manager

The **PnP Manager** handles device detection, driver loading, and device removal. When you plug in a USB device:

1. The bus driver (e.g., usbhub.sys) detects the new device and reports it to the PnP Manager.
2. The PnP Manager queries the device for its hardware IDs.
3. The PnP Manager searches the driver store for a matching driver (using INF files).
4. The driver is loaded and a device stack is built.
5. The PnP Manager sends IRP_MJ_PNP with `IRP_MN_START_DEVICE` to initialize the device.

In Linux, the equivalent flow is: the bus driver detects the device, creates a `struct device`, the kernel's driver matching logic finds a matching driver (via `MODULE_DEVICE_TABLE`), the driver's `probe()` function is called, and udev creates the `/dev` node in user space.

## Power Manager

The **Power Manager** coordinates system and device power states:

**System power states**:
| State | Description | Linux Equivalent |
|---|---|---|
| S0 | Fully running | Running |
| S1 | CPU stops, RAM powered | (rarely used) |
| S2 | CPU off, RAM powered | (rarely used) |
| S3 | Suspend to RAM (Sleep) | Suspend (mem) |
| S4 | Suspend to Disk (Hibernate) | Hibernate (disk) |
| S5 | Soft Off | Shutdown |

**Device power states** (D0 fully on through D3 fully off) allow individual devices to enter low-power states independently. The Power Manager sends IRP_MJ_POWER requests down the device stack, and each driver handles its hardware's power transitions.

Modern Windows uses **Connected Standby** (S0 low-power idle), where the system stays in S0 but puts devices to sleep and wakes them on demand. This is how a Windows laptop can receive email while the lid is closed, similar to how phones work.

## Comparison: Linux I/O vs Windows I/O

| Feature | Linux | Windows |
|---|---|---|
| I/O dispatch structure | file_operations, bio, sk_buff (per subsystem) | IRP (universal for all I/O) |
| Driver layering | Device mapper, FUSE, netfilter (per subsystem) | Device stack with filter drivers (universal) |
| Async I/O (network) | epoll (readiness-based) | IOCP (completion-based) |
| Async I/O (file) | io_uring (completion-based, modern) | Overlapped I/O / IOCP (always supported) |
| Driver frameworks | Subsystem-specific (USB core, PCI, etc.) | WDM / KMDF / UMDF |
| User-mode drivers | FUSE, DPDK, VFIO | UMDF |
| Device detection | udev + driver core | PnP Manager |
| Power management | ACPI subsystem, pm_runtime | Power Manager with IRP_MJ_POWER |

## Real-World Connection

When a .NET web server handles 10,000 concurrent connections, Kestrel (the ASP.NET Core web server) uses IOCP under the hood. Each incoming request is an async socket read posted to the completion port. The thread pool (sized to the number of CPUs) processes completions as they arrive. No thread-per-connection, no busy polling. This is why a single Windows server can handle tens of thousands of concurrent HTTP connections efficiently.

When an antivirus product scans files in real-time, it registers a minifilter driver that intercepts IRP_MJ_CREATE (file open) and IRP_MJ_READ. Before the application gets its data, the AV filter scans the buffer for malware signatures. If the file is malicious, the filter completes the IRP with STATUS_ACCESS_DENIED, and the application sees "Access Denied" without knowing a filter intervened.

When a game developer uses DirectStorage on Windows 11, they bypass the traditional I/O stack entirely. DirectStorage sends NVMe commands directly from the GPU to the storage device, with the data decompressed by the GPU rather than the CPU. This is possible because the Windows I/O model's layered architecture allows the DirectStorage driver to replace the normal file I/O path for game asset loading.

## Interview Angle

**Q: What is an IRP and how does Windows I/O differ from Linux?**

A: An IRP (I/O Request Packet) is the universal data structure for all I/O in Windows -- file, network, device, power, and PnP operations all use IRPs. An IRP flows down a device stack (layers of filter and function drivers), each processing or modifying it, then results flow back up via completion routines. This is a unified model, while Linux uses different structures for different subsystems: `file_operations` for VFS, `struct bio` for block I/O, and `struct sk_buff` for networking. The Windows approach provides consistent layering (filter drivers work the same way for files, network, and devices), while Linux's approach is more specialized per subsystem.

**Q: How do I/O Completion Ports (IOCP) compare to Linux's epoll?**

A: The fundamental difference is completion-based vs readiness-based. IOCP notifies your thread when I/O is completely done and the data is already in your buffer. epoll notifies you when a file descriptor is ready for an operation -- you still need a separate read/write call. IOCP also includes built-in thread pool management, limiting concurrency to the number of CPUs and using LIFO wake-up for cache efficiency. epoll requires you to manage your own thread pool. IOCP works for all I/O types (files, sockets, pipes, devices), while epoll works only for sockets and pipes -- not regular files. Linux's io_uring is the modern evolution that bridges this gap, offering completion-based async I/O with shared ring buffers.

---

Next: [Security Model: Tokens and ACLs](06-windows-security-model-tokens-acls.md)
