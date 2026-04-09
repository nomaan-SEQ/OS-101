# Linux Boot Process in Depth

## The Core Idea

When you press the power button on a Linux system, an intricate sequence of handoffs takes the machine from inert silicon to a running operating system with services, network, and a login prompt. Each stage has a specific job, and each passes control to the next in a carefully orchestrated chain.

Understanding this chain is essential for debugging boot failures, optimizing startup time, and grasping how the kernel initializes itself. This file traces the complete path: firmware to bootloader to kernel to initramfs to systemd.

---

## The Complete Boot Sequence

```
Power On
   │
   ▼
┌─────────────────────────┐
│  UEFI Firmware          │  Hardware initialization, POST,
│  (or Legacy BIOS)       │  find boot device, load bootloader
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  GRUB2 Bootloader       │  Display menu, load kernel + initramfs
│  (from EFI partition)   │  into memory, set kernel command line
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  Linux Kernel           │  Decompress, start_kernel(),
│  Early Initialization   │  initialize all subsystems
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  initramfs              │  Minimal root filesystem in RAM,
│  (Initial RAM FS)       │  load drivers, find real root, mount it
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  systemd (PID 1)        │  Build dependency graph, start
│  on Real Root FS        │  services in parallel, reach target
└───────────┬─────────────┘
            │
            ▼
       Login Prompt
```

---

## Stage 1: UEFI Firmware

Modern systems use UEFI (Unified Extensible Firmware Interface) rather than legacy BIOS. UEFI performs hardware initialization (POST — Power-On Self-Test), discovers boot devices, and loads the bootloader from the **EFI System Partition (ESP)** — a small FAT32 partition (typically 512 MB) at the beginning of the disk.

UEFI reads the ESP and finds the bootloader binary. For GRUB2, this is typically `/EFI/ubuntu/grubx64.efi` or `/EFI/fedora/grubx64.efi`. UEFI stores boot entries in NVRAM, so it knows which ESP path to load.

Secure Boot (if enabled) verifies the bootloader's digital signature before executing it. The chain of trust extends from the firmware's built-in keys to the signed bootloader to the signed kernel.

---

## Stage 2: GRUB2 Bootloader

GRUB2 (GRand Unified Bootloader 2) is the standard Linux bootloader. It reads its configuration from `grub.cfg` and presents a boot menu:

```
# Simplified grub.cfg structure:
menuentry 'Ubuntu 24.04' {
    set root='(hd0,gpt2)'
    linux   /boot/vmlinuz-6.8.0-40-generic \
            root=UUID=a1b2c3d4-... ro quiet splash
    initrd  /boot/initrd.img-6.8.0-40-generic
}

menuentry 'Ubuntu 24.04 (recovery mode)' {
    set root='(hd0,gpt2)'
    linux   /boot/vmlinuz-6.8.0-40-generic \
            root=UUID=a1b2c3d4-... ro recovery nomodeset
    initrd  /boot/initrd.img-6.8.0-40-generic
}
```

The `linux` line loads the kernel image and sets the **kernel command line** — parameters that control kernel behavior:

| Parameter | Purpose |
|-----------|---------|
| `root=UUID=...` | Which device/partition holds the real root filesystem |
| `ro` | Mount root read-only initially (fsck safety) |
| `quiet` | Suppress most boot messages |
| `splash` | Show graphical boot splash |
| `init=/bin/bash` | Override PID 1 (emergency shell, bypasses systemd) |
| `nomodeset` | Disable kernel mode setting (GPU debugging) |
| `mem=4G` | Limit usable memory (testing) |
| `isolcpus=2,3` | Isolate CPUs from the scheduler (real-time workloads) |
| `systemd.unit=rescue.target` | Boot into rescue mode |
| `rd.break` | Break into initramfs before pivot to real root |

GRUB loads both the kernel (`vmlinuz`) and the initramfs (`initrd.img`) into memory, then transfers control to the kernel's entry point.

---

## Stage 3: Kernel Early Boot

The kernel image (`vmlinuz`) is compressed. The first thing that runs is a small decompressor stub that extracts the real kernel into memory. Then execution reaches `start_kernel()` in `init/main.c` — the single most important function in the kernel.

**`start_kernel()` is like the opening ceremony of the kernel. It introduces all the departments one by one: memory management steps up first, then interrupts, then scheduling, then each subsystem in order. Each initialization function must complete before the next one runs — the order matters because later subsystems depend on earlier ones being ready.**

Here is the initialization sequence (simplified):

```
start_kernel()  [init/main.c]
    │
    ├── setup_arch()              Architecture-specific setup (memory layout,
    │                             CPU features, ACPI tables)
    │
    ├── mm_init()                 Memory management initialization
    │   ├── mem_init()            Free bootmem, init buddy allocator
    │   └── kmem_cache_init()     Initialize slab/SLUB allocator
    │
    ├── sched_init()              Initialize the scheduler (per-CPU run queues)
    │
    ├── init_IRQ()                Set up interrupt descriptor table
    │
    ├── time_init()               Initialize timekeeping (TSC, HPET, ACPI timer)
    │
    ├── console_init()            Early console for boot messages
    │
    ├── vfs_caches_init()         Initialize VFS (dentry cache, inode cache)
    │
    ├── page_cache_init()         Initialize the page cache
    │
    ├── signals_init()            Signal infrastructure
    │
    ├── proc_root_init()          Mount the initial /proc filesystem
    │
    ├── fork_init()               Initialize the task allocator (max threads)
    │
    ├── rest_init()               Final setup:
    │   ├── kernel_thread(kernel_init)   Create PID 1 kernel thread
    │   ├── kernel_thread(kthreadd)      Create PID 2 (kernel thread manager)
    │   └── cpu_idle()                   Become the idle task (PID 0)
    │
    └── kernel_init()             [runs as PID 1]
        ├── kernel_init_freeable()
        │   ├── do_basic_setup()         Run all __initcall functions
        │   │                            (module_init, device probing)
        │   └── prepare_namespace()      Mount initramfs
        │
        └── run_init_process()           exec() into /sbin/init (systemd)
                                         or /init in initramfs
```

After `start_kernel()` completes its initialization, the system has a working scheduler, memory manager, filesystem layer, and driver framework. But the real root filesystem is not yet mounted — that is where initramfs comes in.

---

## Stage 4: initramfs

The initramfs (initial RAM filesystem) is a compressed CPIO archive that GRUB loaded into memory. The kernel unpacks it into a temporary root filesystem (`rootfs`) and runs its init script.

### Why initramfs Exists

The kernel needs filesystem drivers and storage drivers to mount the real root filesystem. But those drivers might be compiled as modules (not built into the kernel). The initramfs contains exactly the modules and tools needed to find and mount the real root:

```
initramfs contents (typical):
├── init                 Main init script (or systemd in initramfs)
├── bin/
│   ├── busybox          Minimal shell and utilities
│   └── mount, switch_root, ...
├── lib/modules/6.8.0/
│   ├── ext4.ko          Filesystem driver
│   ├── nvme.ko          NVMe storage driver
│   ├── dm-crypt.ko      Encryption driver (for LUKS)
│   └── ahci.ko          SATA driver
├── etc/
│   └── lvm/             LVM configuration
├── sbin/
│   ├── udevd            Device manager
│   ├── lvm              Logical volume tools
│   └── cryptsetup       LUKS decryption
└── scripts/             Init hook scripts
```

### The initramfs Init Process

```
initramfs /init script runs:
    │
    ├── Start udevd (device manager)
    │   → Detect hardware, load appropriate kernel modules
    │
    ├── Assemble storage:
    │   ├── Activate LVM logical volumes (if using LVM)
    │   ├── Unlock LUKS encrypted volumes (prompt for passphrase)
    │   └── Assemble RAID arrays (if using mdraid)
    │
    ├── Find root filesystem:
    │   └── Match root= parameter (UUID, LABEL, or device path)
    │
    ├── fsck root filesystem (if needed)
    │
    ├── Mount real root filesystem at /sysroot (or /root)
    │
    └── switch_root /sysroot /sbin/init
        │
        ▼
    Execution transfers to the REAL /sbin/init (systemd)
    on the real root filesystem.
    initramfs is freed from memory.
```

`switch_root` (or `pivot_root`) performs the critical handoff: it makes the real root filesystem the new `/`, frees the initramfs memory, and `exec()`s the real init process. From this point, the initramfs no longer exists.

### Initramfs Generation Tools

| Tool | Used By | Notes |
|------|---------|-------|
| **dracut** | Fedora, RHEL, SUSE | Modular, policy-driven (includes only what is needed) |
| **mkinitramfs** / **update-initramfs** | Debian, Ubuntu | Includes most modules by default (larger but safer) |
| **mkinitcpio** | Arch Linux | Hook-based, manual configuration |

```bash
# Rebuild initramfs on Ubuntu:
sudo update-initramfs -u

# Rebuild on Fedora/RHEL:
sudo dracut --force

# List initramfs contents:
lsinitramfs /boot/initrd.img-$(uname -r)  # Debian/Ubuntu
lsinitrd /boot/initramfs-$(uname -r).img  # Fedora/RHEL
```

---

## Stage 5: systemd as PID 1

Once the real root filesystem is mounted and `switch_root` executes `/sbin/init` (which is typically a symlink to `/lib/systemd/systemd`), systemd takes over as PID 1.

**systemd is like a project manager with a dependency chart. It knows which services depend on what, starts independent ones in parallel, and ensures everything comes up in the right order. Instead of the old sequential init scripts (start A, wait, start B, wait, start C), systemd analyzes the dependency graph and launches A, B, and C simultaneously if they have no dependencies on each other.**

### Unit Types

systemd organizes everything into **units**:

| Unit Type | Extension | Purpose |
|-----------|-----------|---------|
| **service** | `.service` | Daemons and one-shot programs |
| **socket** | `.socket` | Socket activation (start service on connection) |
| **mount** | `.mount` | Filesystem mounts (replaces /etc/fstab lines) |
| **timer** | `.timer` | Scheduled execution (replaces cron) |
| **target** | `.target` | Groups of units (like runlevels) |
| **slice** | `.slice` | cgroup resource control grouping |
| **device** | `.device` | Kernel device (auto-created by udev) |
| **path** | `.path` | File/directory monitoring |

### Targets (Modern Runlevels)

| Target | Equivalent Runlevel | Purpose |
|--------|-------------------|---------|
| `poweroff.target` | 0 | System halt |
| `rescue.target` | 1 | Single-user rescue mode |
| `multi-user.target` | 3 | Multi-user, no GUI |
| `graphical.target` | 5 | Multi-user with GUI |
| `reboot.target` | 6 | System reboot |

### Parallel Startup

systemd builds a dependency graph from unit files and starts units as soon as their dependencies are met:

```
                    sysinit.target
                    /      |       \
                   /       |        \
           udev.service  lvm2.service  cryptsetup.target
                   \       |        /
                    \      |       /
                   basic.target
                   /       |      \
                  /        |       \
         network.target  docker.service  sshd.service
                  \        |       /
                   \       |      /
                multi-user.target
                       |
                  display-manager.service
                       |
                graphical.target
```

Services at the same level with no interdependencies start simultaneously. This is why systemd boots significantly faster than SysVinit, which ran init scripts sequentially.

### Key systemd Commands

```bash
# Service management:
systemctl start nginx.service
systemctl stop nginx.service
systemctl restart nginx.service
systemctl status nginx.service

# Enable/disable at boot:
systemctl enable nginx.service     # Create symlink in target's wants/
systemctl disable nginx.service

# Check boot time:
systemd-analyze                    # Total boot time
systemd-analyze blame              # Time per unit (sorted)
systemd-analyze critical-chain     # Critical path (longest dependency chain)

# Logs:
journalctl -u nginx.service        # Logs for specific unit
journalctl -b                      # All logs from current boot
journalctl -b -1                   # Logs from previous boot
journalctl --since "10 minutes ago"
journalctl -p err                  # Only errors and above
```

---

## Boot Optimization

```bash
# Find the slowest units:
$ systemd-analyze blame
  12.345s NetworkManager-wait-online.service
   4.567s docker.service
   3.210s snapd.service
   2.100s plymouth-quit-wait.service
   1.500s systemd-udevd.service
   ...

# Find the critical chain (longest path):
$ systemd-analyze critical-chain
graphical.target @25.000s
└─multi-user.target @24.998s
  └─docker.service @20.431s +4.567s
    └─network.target @20.400s
      └─NetworkManager.service @15.200s +5.200s
        └─basic.target @15.100s
          └─sockets.target @15.050s
            ...
```

Common optimizations:
- Disable `NetworkManager-wait-online.service` if network is not needed at boot
- Use socket activation to defer service startup until first connection
- Move slow services to `timers` that start after boot completes
- Mask unnecessary services: `systemctl mask snapd.service`

---

## Debugging Boot Failures

| Symptom | Debug Approach |
|---------|---------------|
| System hangs before GRUB | UEFI settings, boot order, Secure Boot keys |
| GRUB menu appears but kernel fails to load | Check `grub.cfg`, verify kernel image exists |
| Kernel panic during boot | Add `nomodeset`, remove `quiet splash`, read panic message |
| initramfs drops to shell | Storage driver missing, wrong root= parameter, fsck failure |
| systemd hangs on a service | Boot with `systemd.unit=rescue.target`, check `journalctl -b` |
| Login prompt never appears | Check display manager: `systemctl status gdm` or `sddm` |

### Emergency Access

```bash
# At GRUB menu, press 'e' to edit kernel command line:

# Drop to initramfs shell (before real root is mounted):
# Add: rd.break

# Drop to root shell (bypass systemd):
# Change: init=/bin/bash

# Boot to rescue mode (minimal systemd):
# Add: systemd.unit=rescue.target

# Boot to emergency mode (no mounts except root):
# Add: systemd.unit=emergency.target
```

---

## Real-World Connection

**Cloud instance boot:** When an AWS EC2 instance launches, the boot sequence is the same: UEFI/BIOS firmware in the hypervisor loads GRUB, which loads the kernel and initramfs from the EBS root volume. The initramfs detects the virtualized storage (NVMe or Xen block device), mounts the root filesystem, and systemd starts. cloud-init runs as a systemd service to configure the instance (SSH keys, network, hostname, user data scripts). The entire boot takes 10-30 seconds depending on instance type.

**Kernel panic analysis:** When a production server fails to boot, the kernel panic message is critical. It includes a call trace showing exactly which function crashed. Common causes: corrupted initramfs (rebuild it), bad kernel module (boot with previous kernel), hardware failure (memory or disk errors), root filesystem corruption (boot to rescue, run fsck).

**Fast boot environments:** Embedded Linux systems (routers, IoT devices) optimize for sub-second boot by: using a minimal kernel with only required drivers built-in (no modules, no initramfs needed), replacing systemd with a minimal init, and mounting a read-only root filesystem.

---

## Interview Angle

**Q: Walk through the Linux boot process from power-on to login prompt.**

A: UEFI firmware initializes hardware and loads GRUB2 from the EFI System Partition. GRUB reads its configuration, loads the kernel and initramfs into memory, and passes control to the kernel with command line parameters. The kernel decompresses, runs `start_kernel()` to initialize memory management, the scheduler, interrupts, VFS, and drivers. It then unpacks the initramfs, which loads storage drivers, assembles any LVM/RAID/LUKS volumes, finds the real root filesystem, and mounts it. `switch_root` transfers to the real root and execs systemd as PID 1. systemd builds a dependency graph of unit files and starts services in parallel until reaching the default target (usually multi-user or graphical), which includes the login prompt.

**Q: What is the purpose of initramfs and why cannot the kernel mount the root filesystem directly?**

A: The kernel needs storage drivers and filesystem drivers to mount the root filesystem, but those drivers might be compiled as loadable modules rather than built into the kernel. The initramfs is a small RAM-based filesystem that contains exactly the modules and tools needed to access the real root: storage drivers (NVMe, SCSI), filesystem drivers (ext4, XFS), and tools for LVM, LUKS decryption, and RAID assembly. Once the real root is accessible, initramfs hands off control and is freed from memory. Without initramfs, the kernel would need every possible storage and filesystem driver built-in, making it enormous.

**Q: How does systemd achieve faster boot times compared to SysVinit?**

A: SysVinit runs init scripts sequentially — service A starts, finishes, then service B starts. systemd builds a dependency graph from unit files and starts all units whose dependencies are already satisfied in parallel. Services with no interdependencies start simultaneously. Additionally, socket activation defers starting a service until something actually connects to its socket, and aggressive on-demand loading means services that are not needed immediately do not delay the boot path.

---

**Next**: [08-linux-cgroups-namespaces-deep-dive.md](08-linux-cgroups-namespaces-deep-dive.md)
