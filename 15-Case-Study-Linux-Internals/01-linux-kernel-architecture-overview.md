# Linux Kernel Architecture Overview

## The Core Idea

The Linux kernel is **monolithic but modular**. This sounds like a contradiction, but it is the key design insight that makes Linux both fast and flexible.

**Monolithic** means the entire kernel — process scheduler, memory manager, filesystem layer, networking stack, device drivers — runs as a single binary in a single address space in kernel mode. When the scheduler needs to check a process's memory mappings, it directly calls functions in the memory subsystem. No message passing, no context switches, no serialization overhead.

**Modular** means pieces of functionality (primarily device drivers and filesystem implementations) can be compiled as **loadable kernel modules** (`.ko` files) that are inserted into or removed from the running kernel at runtime, without rebooting.

Think of it this way: **the Linux kernel is like a mega-corporation where all departments (scheduling, memory, networking, filesystems) work in the same building (address space) and can walk to each other's desks (direct function calls). Loadable modules are like contractors who can badge in and out of the building as needed — they get full building access while they are inside, but the building does not need to be rebuilt every time a new contractor shows up.**

This stands in contrast to a **microkernel** (like Mach or MINIX 3) where subsystems run as separate user-space processes and communicate via message passing. Microkernels are theoretically cleaner and more fault-tolerant, but the IPC overhead is significant. Linus Torvalds famously debated this with Andrew Tanenbaum in 1992 — and the industry largely sided with the monolithic approach for performance reasons.

---

## Kernel Source Tree Layout

The Linux kernel source code is organized into a well-defined directory structure. Understanding this layout is your roadmap to understanding the kernel itself:

```
linux/
├── arch/           Architecture-specific code (x86, arm64, riscv, etc.)
│   ├── x86/        x86-specific: boot, memory management, syscall entry
│   └── arm64/      ARM64-specific code
├── kernel/         Core kernel: scheduling, signals, timers, tracing
│   ├── sched/      The scheduler (CFS, RT, deadline)
│   └── locking/    Mutexes, spinlocks, RCU
├── mm/             Memory management: page allocator, slab, page cache, OOM
├── fs/             Filesystems: VFS layer + specific implementations
│   ├── ext4/       ext4 filesystem
│   ├── xfs/        XFS filesystem
│   ├── proc/       procfs (/proc)
│   └── sysfs/      sysfs (/sys)
├── net/            Networking: TCP/IP stack, socket layer, netfilter
│   ├── ipv4/       IPv4 implementation
│   ├── ipv6/       IPv6 implementation
│   └── core/       Core networking (sk_buff, device handling)
├── drivers/        Device drivers (largest directory — ~70% of kernel code)
│   ├── gpu/        Graphics drivers
│   ├── net/        Network interface drivers
│   ├── block/      Block device drivers
│   └── usb/        USB drivers
├── include/        Header files for all subsystems
│   ├── linux/      Core kernel headers (sched.h, mm.h, fs.h)
│   └── uapi/       User-space API headers (syscall numbers, ioctl definitions)
├── init/           Kernel initialization (main.c contains start_kernel())
├── ipc/            System V IPC: shared memory, semaphores, message queues
├── security/       LSM framework, SELinux, AppArmor, seccomp
├── block/          Block I/O layer (request queues, I/O schedulers)
├── crypto/         Cryptographic API and algorithms
├── tools/          Userspace tools (perf, bpf, selftests)
└── Documentation/  Kernel documentation (increasingly in RST format)
```

The `drivers/` directory alone accounts for roughly 70% of the total kernel source. This makes sense — the kernel must support an enormous range of hardware, and each device needs its own driver. The core kernel logic (scheduling, memory, filesystems, networking) is comparatively compact.

---

## Subsystem Interconnection

The kernel subsystems are not isolated silos. They interact constantly:

```
                    ┌──────────────────────────────────────────────┐
                    │              System Call Interface            │
                    │   (read, write, open, fork, mmap, socket...) │
                    └──────┬──────┬──────┬──────┬──────┬──────────┘
                           │      │      │      │      │
                    ┌──────▼──┐ ┌─▼────┐ │  ┌───▼──┐ ┌─▼────────┐
                    │ Process │ │ VFS  │ │  │ Net  │ │ Security  │
                    │  Mgmt   │ │Layer │ │  │Stack │ │  (LSM)    │
                    │(sched/) │ │(fs/) │ │  │(net/)│ │(security/)│
                    └────┬────┘ └──┬───┘ │  └──┬───┘ └───────────┘
                         │         │     │     │
                    ┌────▼─────────▼─────▼─────▼────┐
                    │       Memory Management        │
                    │   (mm/ — page tables, slab,    │
                    │    page cache, mmap, OOM)       │
                    └──────────────┬─────────────────┘
                                   │
                    ┌──────────────▼─────────────────┐
                    │       Block I/O Layer           │
                    │   (block/ — request queues,     │
                    │    I/O schedulers, blk-mq)      │
                    └──────────────┬─────────────────┘
                                   │
                    ┌──────────────▼─────────────────┐
                    │       Device Drivers            │
                    │   (drivers/ — disk, NIC, GPU)   │
                    └──────────────┬─────────────────┘
                                   │
                    ┌──────────────▼─────────────────┐
                    │   Architecture-Specific Code    │
                    │ (arch/ — interrupt handling,     │
                    │  context switch, page table ops) │
                    └─────────────────────────────────┘
```

Notice how **memory management** sits at the center — almost every subsystem needs to allocate and free memory, manage page tables, or interact with the page cache. This is why `mm/` is one of the most critical (and complex) parts of the kernel.

---

## Loadable Kernel Modules

Modules let you extend the kernel without recompiling or rebooting. Most device drivers and some filesystems are shipped as modules:

```bash
# List currently loaded modules
lsmod

# Load a module
sudo modprobe vfat          # Load the FAT filesystem module

# Remove a module
sudo modprobe -r vfat

# See module info
modinfo ext4

# Module files live here:
ls /lib/modules/$(uname -r)/kernel/
```

When a module is loaded, it gets full access to the kernel address space — it can call any exported kernel function. This is both powerful and dangerous: a buggy module can crash the kernel just like a buggy built-in subsystem. This is why Linux marks the kernel as "tainted" when you load out-of-tree (non-mainline) modules, and bug reports from tainted kernels are often deprioritized.

---

## Kernel Versioning

Linux uses a simple versioning scheme:

| Component | Meaning | Example |
|-----------|---------|---------|
| **Major.Minor** | Release series | 6.8 |
| **Patch** | Stable fixes within a release | 6.8.3 |
| **-rc** | Release candidate (pre-release) | 6.9-rc4 |

New major releases happen roughly every 9-10 weeks. Certain releases are designated as **LTS (Long-Term Support)** and receive bug and security fixes for 2-6 years. Distros typically pick an LTS kernel:

| Distro | Typical Kernel | Why |
|--------|---------------|-----|
| Ubuntu 22.04 LTS | 5.15 LTS | Stability for enterprise |
| RHEL 9 | 5.14 (heavily backported) | Decades of support commitment |
| Arch Linux | Latest stable | Bleeding edge |
| Android | Modified LTS kernels | Stability + vendor patches |

---

## How to Read Kernel Source

Reading kernel source can feel overwhelming, but there is a consistent approach:

1. **Start from the syscall entry.** If you want to understand `read()`, find the `SYSCALL_DEFINE3(read, ...)` macro in `fs/read_write.c`.
2. **Follow the call chain.** `read()` calls `vfs_read()`, which calls the filesystem's `file_operations->read` or `read_iter`, which eventually reaches the device driver.
3. **Use cross-reference tools.** [Elixir (elixir.bootlin.com)](https://elixir.bootlin.com/linux/latest/source) provides hyperlinked kernel source — click on any function to jump to its definition.
4. **Read the header files.** Data structures are defined in `include/linux/`. Want to understand processes? Read `include/linux/sched.h` for `task_struct`.

---

## /proc and /sys: Windows Into Kernel State

Linux exposes its internal state through two virtual filesystems that are indispensable for understanding what the kernel is doing:

### /proc — Process and System Information

```
/proc/
├── 1/              PID 1 (init/systemd) info
│   ├── status      State, memory, UIDs, capabilities
│   ├── maps        Virtual memory regions
│   ├── fd/         Open file descriptors (symlinks to actual files)
│   ├── cmdline     Command that started this process
│   └── stat        Raw scheduling and timing stats
├── meminfo         System memory breakdown
├── cpuinfo         CPU details (model, cache, flags)
├── interrupts      Interrupt counts per CPU per IRQ
├── buddyinfo       Buddy allocator free page counts
├── slabinfo        Slab allocator object caches
└── sys/            Tunable kernel parameters (sysctl)
    ├── kernel/     Kernel tunables (pid_max, shmmax, ...)
    ├── vm/         VM tunables (swappiness, dirty ratios)
    └── net/        Network tunables (tcp_*, ip_*)
```

### /sys — Kernel Object Hierarchy

```
/sys/
├── class/          Device classes (net/, block/, tty/)
├── devices/        Physical device hierarchy
├── fs/             Filesystem-related (cgroup, ext4 params)
├── kernel/         Kernel state (debug, security)
└── module/         Loaded kernel modules and their parameters
```

Everything under `/proc` and `/sys` is generated on-the-fly by kernel code when you read it. There are no real files on disk. This is why `cat /proc/meminfo` always shows current values — the kernel computes them at read time.

---

## Key Kernel Configuration Options

When compiling a kernel, thousands of options can be toggled. Here are the ones that fundamentally change kernel behavior:

| Config Option | What It Controls |
|--------------|-----------------|
| `CONFIG_SMP` | Symmetric Multi-Processing support (multi-core) |
| `CONFIG_PREEMPT_NONE` | No kernel preemption (server workloads — maximum throughput) |
| `CONFIG_PREEMPT` | Full kernel preemption (desktop/low-latency — better responsiveness) |
| `CONFIG_PREEMPT_RT` | Real-time preemption patch (hard real-time guarantees) |
| `CONFIG_MODULES` | Enable loadable kernel modules |
| `CONFIG_CGROUPS` | Control group support (required for containers) |
| `CONFIG_NAMESPACES` | Namespace support (required for containers) |
| `CONFIG_BLK_CGROUP` | Block I/O cgroup controller |
| `CONFIG_SECURITY_SELINUX` | SELinux support |
| `CONFIG_DEBUG_INFO_BTF` | BPF Type Format (needed for modern eBPF tools) |

### Compilation Basics

```bash
# Get source
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git

# Configure (interactive menu)
make menuconfig

# Or start from your distro's config
cp /boot/config-$(uname -r) .config
make olddefconfig

# Build (using all CPU cores)
make -j$(nproc)

# Install modules and kernel
sudo make modules_install
sudo make install
```

Different distros ship very different kernel configs because they target different workloads. Ubuntu enables most modules for broad hardware support. RHEL strips features for stability and certification. Android heavily customizes for mobile power management and hardware abstraction.

---

## Real-World Connection

**Why distro kernel configs differ:** Ubuntu Desktop enables `CONFIG_PREEMPT` for snappy responsiveness. Ubuntu Server and RHEL use `CONFIG_PREEMPT_NONE` for maximum throughput on server workloads. Android kernels enable aggressive power management features (interactive governor, wakelocks) that mainline Linux does not have. This is the same kernel source code — the configuration makes it behave very differently.

**Kernel modules in practice:** When you plug in a USB Wi-Fi adapter, udev detects the device, looks up its vendor/product ID, and loads the appropriate kernel module automatically. If the driver is not included with your distro's kernel, you may need to compile it from source or use DKMS (Dynamic Kernel Module Support) to build it against your running kernel.

**Reading /proc in production:** When diagnosing memory pressure, `cat /proc/meminfo` tells you exactly how RAM is being used — how much is page cache, how much is slab, how much is truly free. When debugging I/O, `/proc/diskstats` shows per-device read/write counts and latencies. These are the same sources that monitoring tools like Prometheus node_exporter read.

---

## Interview Angle

**Q: What does it mean that Linux is a monolithic kernel, and what are the tradeoffs?**

A: All kernel subsystems (scheduling, memory management, filesystems, networking, drivers) run in a single address space in kernel mode. The advantage is performance — subsystems communicate via direct function calls with no overhead. The disadvantage is reliability — a bug in any subsystem (especially a driver) can crash the entire kernel. Linux mitigates this with loadable modules (so buggy drivers can be removed without reboot), code review processes, and extensive testing infrastructure. In contrast, a microkernel isolates subsystems in separate address spaces, gaining fault isolation but paying significant IPC overhead.

**Q: How would you explore what a specific system call does inside the kernel?**

A: Start by finding the syscall definition using the `SYSCALL_DEFINEn` macro (e.g., `SYSCALL_DEFINE3(read, ...)` in `fs/read_write.c`). Follow the call chain through the VFS layer to the specific filesystem or subsystem implementation. Use an online cross-reference tool like Elixir Bootlin to navigate the source interactively. For runtime tracing, tools like `strace` (syscall boundary), `ftrace` (kernel function tracing), and `bpftrace` (programmable tracing) let you observe the kernel in action on a running system.

**Q: What is the difference between /proc and /sys?**

A: `/proc` is the older interface, primarily focused on process information (`/proc/PID/`) and system-wide statistics (`/proc/meminfo`, `/proc/cpuinfo`). It also exposes tunable parameters via `/proc/sys/`. `/sys` (sysfs) is newer and more structured — it exposes the kernel's internal object model: devices, drivers, modules, and their attributes as a proper hierarchy. Both are virtual filesystems with no on-disk storage; the kernel generates content at read time.

---

**Next**: [02-linux-process-and-thread-implementation.md](02-linux-process-and-thread-implementation.md)
