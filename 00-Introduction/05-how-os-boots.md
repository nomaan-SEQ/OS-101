# How an OS Boots

## Why This Matters

Understanding the boot process answers the question: **"How does an OS go from 'no power' to 'running processes'?"** It ties together hardware initialization, firmware, bootloaders, and the kernel — and it's a surprisingly common interview question.

## The Big Picture

```
Power On
   │
   ▼
┌──────────┐    ┌────────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Firmware │───►│ Bootloader │───►│  Kernel  │───►│  Init    │───►│  Login   │
│(BIOS/UEFI)│   │(GRUB/Boot  │    │  Loads & │    │ System   │    │  Screen  │
│           │    │ Manager)   │    │  Inits   │    │(systemd) │    │          │
└──────────┘    └────────────┘    └──────────┘    └──────────┘    └──────────┘
  Hardware        Find & load       Init hardware   Start system    Ready for
  self-test       the kernel        subsystems       services        user
```

## Step-by-Step: The Boot Sequence

### Step 1: Power On and CPU Reset

When you press the power button:

**1. The power supply stabilizes and sends a "power good" signal to the motherboard.**

The power supply unit (PSU) doesn't instantly produce clean electricity. It needs ~100–500ms to stabilize the voltages your components need (3.3V, 5V, 12V). If the CPU started executing before voltages were stable, it would malfunction. So the PSU sends an electrical signal called **"power good"** to the motherboard, meaning: *"Voltages are stable. Safe to start."*

> **Analogy:** Think of it like a safety inspector at a construction site. Workers (the CPU) can't start until the inspector confirms the scaffolding (power) is solid.

**2. The CPU resets to a known state and begins executing instructions at a hardcoded address (the reset vector).**

When a CPU is first powered on, its internal registers (tiny storage slots that hold data the CPU is actively working with) could contain random garbage values from electrical noise. **"Reset to a known state"** means the CPU forces all its registers to specific, documented default values — the instruction pointer is set to the **reset vector** address (`0xFFFFFFF0` on x86), and other registers are zeroed or set to fixed defaults. This gives the CPU a clean, predictable starting point.

> **Analogy:** Like clearing a calculator before starting a new calculation. You can't trust whatever number was left on the display — you hit "C" first, then start.

The reset vector address is hardwired to point at the firmware (BIOS/UEFI) stored on a flash chip on the motherboard. So the very first thing the CPU does is start running firmware code.

**3. At this point: RAM is empty, no OS is loaded, and the CPU is running in real mode.**

#### What Is Real Mode?

Real mode is the CPU's simplest and most primitive operating mode. It's inherited from the original Intel 8086 processor (1978), and **every x86 CPU still starts in real mode** for backward compatibility — even a modern 64-core server CPU.

| Property | Real Mode |
|----------|-----------|
| **Address bus** | 20-bit — can only access **1 MB** of memory |
| **Memory protection** | **None** — any code can read/write any memory address |
| **Virtual memory** | **None** — all addresses are physical |
| **Privilege levels** | **None** — no distinction between kernel and user code |
| **Instruction set** | 16-bit operations |

> **Analogy:** Real mode is like a house with no locks on any doors and no walls between rooms. Anyone (any program) can walk anywhere and touch anything. The first thing the OS does during boot is build those walls and install those locks (by switching to protected or long mode).

The firmware runs in real mode initially. Later, the kernel will switch the CPU out of real mode into a modern mode with memory protection — we'll see that in Step 4.

### Step 2: Firmware (BIOS or UEFI)

The firmware is software stored on a flash chip on the motherboard. But there are two very different kinds of firmware — **BIOS** and **UEFI** — and understanding the difference matters.

#### What Are BIOS and UEFI?

**BIOS (Basic Input/Output System)** is the original firmware standard, dating back to 1981 (the IBM PC). It's extremely simple — it runs in 16-bit real mode, has a text-based setup screen, and was designed when hard drives were measured in megabytes.

**UEFI (Unified Extensible Firmware Interface)** is the modern replacement, developed in the 2000s. It runs in 32-bit or 64-bit mode, supports a graphical interface, can load drivers, and includes features like Secure Boot.

| Feature | BIOS (Legacy) | UEFI (Modern) |
|---------|--------------|---------------|
| **Year introduced** | 1981 | ~2005 (widespread by 2012+) |
| **CPU mode** | 16-bit real mode | 32-bit or 64-bit |
| **Interface** | Text-based, keyboard only | Graphical, mouse support |
| **Partition scheme** | MBR (max 4 partitions, 2 TB disks) | GPT (128 partitions, 9.4 ZB disks) |
| **Boot process** | Loads 512-byte MBR bootloader | Loads `.efi` program from ESP partition |
| **Secure Boot** | No | Yes — verifies bootloader signatures |
| **Network boot** | Basic PXE | Full network stack |
| **Max disk support** | 2 TB | Virtually unlimited |

> **Analogy:** BIOS is like an old flip phone — it makes calls and that's about it. UEFI is a smartphone — same basic job, but with a touchscreen, apps, security features, and no arbitrary limitations.

Most computers made after 2012 use UEFI. Many still offer a "Legacy BIOS" compatibility mode (CSM) for older operating systems.

#### a) POST (Power-On Self-Test)

Regardless of whether the firmware is BIOS or UEFI, the first thing it does is POST:

- Tests that critical hardware works: CPU, RAM, video, keyboard
- If something fails, you hear beep codes or see no display
- This is why you sometimes hear a single beep at startup — that's POST saying "all good"

#### b) Find and Load the Bootloader

Now the firmware needs to find the OS. This is where BIOS and UEFI paths diverge:

**Legacy BIOS Path:**
1. BIOS looks at the **boot order** (configured in BIOS settings)
2. Reads the **first sector** (512 bytes) of the boot device — this is the **Master Boot Record (MBR)**
3. MBR contains a tiny bootloader program and a **partition table**
4. BIOS loads that bootloader into memory and jumps to it

**What is a partition table?** A hard drive is one big block of storage, but you usually want to divide it into separate sections — one for the OS, one for your data, maybe one for swap space. These sections are called **partitions**. The partition table is a small data structure that records where each partition starts and ends on the disk.

> **Analogy:** Think of a partition table like a table of contents in a book. The book (disk) has chapters (partitions), and the table of contents tells you where each chapter begins and ends so you can jump to the right one.

| Partition Scheme | Max Partitions | Max Disk Size | Used With |
|-----------------|---------------|---------------|-----------|
| **MBR** | 4 primary (or 3 + extended) | 2 TB | BIOS |
| **GPT** | 128 | 9.4 ZB (effectively unlimited) | UEFI |

**Modern UEFI Path:**
1. UEFI reads the **EFI System Partition** (a FAT32 partition on the disk)
2. Finds bootloader files (e.g., `\EFI\BOOT\BOOTX64.EFI`)
3. Loads and executes the bootloader directly
4. No 512-byte size limit — much more capable than BIOS/MBR

```
BIOS Path:                          UEFI Path:
─────────                           ─────────
ROM → POST → MBR (512B)            Flash → POST → ESP Partition
     → Bootloader (limited)              → Bootloader (.efi file)
     → Kernel                            → Kernel
```

### Step 3: Bootloader

The bootloader's job: **find and load the OS kernel into memory**.

**Common bootloaders:**
- **GRUB** (Linux) — Shows a menu if multiple OSes are installed
- **Windows Boot Manager** (`bootmgr`) — Loads `winload.efi` → NT kernel
- **systemd-boot** — Simpler UEFI-only bootloader for Linux

**What GRUB does (Linux example):**
1. Displays boot menu (if configured)
2. Reads its config to find the kernel image path (e.g., `/boot/vmlinuz-6.1.0`)
3. Loads the **kernel image** into memory
4. Loads the **initial RAM filesystem** (`initramfs` / `initrd`) — a temporary root filesystem with essential drivers
5. Passes **kernel command-line parameters** (e.g., `root=/dev/sda1 quiet`)
6. Jumps to the kernel's entry point

### Step 4: Kernel Initialization

The kernel takes over. This is where the real OS starts.

#### What Are Protected Mode and Long Mode?

Remember, the CPU started in **real mode** (16-bit, no protection, 1 MB limit). The kernel's first job is to switch the CPU into a modern mode:

**Protected Mode (32-bit)** — introduced with the Intel 80386 (1985):
- Addressable memory jumps from 1 MB to **4 GB**
- **Memory protection** is enabled — processes can't access each other's memory
- **Privilege rings** activate — Ring 0 (kernel) and Ring 3 (user programs)
- **Virtual memory** with paging becomes possible

**Long Mode (64-bit)** — introduced with AMD64 (2003):
- Addressable memory jumps to **256 TB** (48-bit virtual addresses, extensible to 57-bit)
- Required for modern OSes and for using more than 4 GB of RAM
- All modern desktop and server OSes run in long mode

> **Analogy:** Real mode is a dirt road with no traffic rules — one lane, no speed limits, anyone can go anywhere. Protected mode adds lanes, traffic lights, and speed limits — a proper road system. Long mode widens that road into a 16-lane highway that can handle massive traffic.

The transition happens during early kernel boot: **real mode → protected mode → long mode**. Once in long mode, the CPU never goes back (until the next reboot).

```
Kernel Entry Point
       │
       ▼
1. Switch to protected mode (32-bit) then long mode (64-bit)
       │
       ▼
2. Set up memory management
   - Initialize page tables (data structures that map virtual addresses to physical RAM locations — covered in depth in the Memory Management section)
   - Enable virtual memory (paging)
       │
       ▼
3. Initialize core subsystems
   - Interrupt handlers (IDT — the **Interrupt Descriptor Table**, which maps each interrupt/exception number to the kernel function that should handle it)
   - Process scheduler
   - Memory allocator
       │
       ▼
4. Detect and initialize hardware
   - Using initramfs drivers
   - PCI bus enumeration
   - Disk controllers, network cards
       │
       ▼
5. Mount the real root filesystem
   - Switch from initramfs to actual disk
       │
       ▼
6. Start the first user-space process
   - PID 1: the init system
```

**Key moment:** When the kernel starts the first user-space process, the system transitions from "booting" to "running." Everything after this is user-space.

### Step 5: Init System (PID 1)

The **first process** the kernel creates is the init system. It's the ancestor of all other processes.

**Modern Linux: systemd**
- Reads **unit files** that describe services and their dependencies
- Starts services in **parallel** (faster boot than sequential init)
- Manages service lifecycle (restart on crash, logging, etc.)

**Boot target sequence (systemd):**
```
sysinit.target          → Mount filesystems, set hostname, load modules
    │
    ▼
basic.target            → Start basic system services (logging, D-Bus)
    │
    ▼
multi-user.target       → Start networking, SSH, databases, web servers
    │
    ▼
graphical.target        → Start display manager (login screen)
(if desktop)
```

**Windows equivalent:**
- `smss.exe` (Session Manager) → `csrss.exe` (Client/Server Runtime) → `wininit.exe` → `services.exe` (Service Control Manager) → Login screen

### Step 6: Login and User Session

- **Linux:** Display manager (GDM, SDDM) or text login prompt
- **Windows:** `winlogon.exe` shows the lock screen / login prompt

After authentication, a **user session** starts — your shell, desktop environment, and applications.

## The Complete Timeline (Linux)

```
Time ─────────────────────────────────────────────────────►

│ Firmware │  Bootloader  │     Kernel Init      │  Userspace  │
│ (UEFI)  │   (GRUB)     │                      │  (systemd)  │
│          │              │                      │             │
│  POST    │  Load kernel │  Memory mgmt         │  Mount FS   │
│  Find    │  Load initrd │  Init scheduler      │  Networking │
│  boot    │  Set params  │  Detect hardware     │  Services   │
│  device  │  Jump to     │  Mount root FS       │  Login      │
│          │  kernel      │  Start PID 1         │  Desktop    │
│          │              │                      │             │
│ ~2-5s    │   ~1-3s      │     ~2-5s            │   ~3-10s    │
```

## What Is initramfs?

**initramfs** (initial RAM filesystem) is a small, compressed archive (typically a gzip-compressed cpio archive) that contains a **minimal filesystem** — just enough to get the real OS accessible.

It includes:
- **Essential device drivers** — for disk controllers, filesystems, RAID, LVM, encryption
- **Filesystem tools** — `mount`, `fsck`, `modprobe`
- **An init script** — a small program that finds and mounts the real root filesystem

The bootloader loads initramfs into **RAM** alongside the kernel image. It's not a real disk filesystem — it exists purely in memory and is discarded once its job is done.

> **Analogy:** initramfs is like a pocket toolkit you carry when you're locked out of your workshop. It has just enough tools (screwdriver, lock pick) to open the door. Once inside, you have access to all your real tools and you can put the pocket toolkit away.

You might wonder: why do we need it at all?

## Why initramfs Exists

A chicken-and-egg problem:
- The kernel needs **drivers** to read the disk where the root filesystem is stored
- But those drivers are **on the root filesystem**

Solution: The bootloader loads `initramfs` — a small compressed filesystem loaded entirely into RAM. It contains just enough drivers and tools to mount the real root filesystem. Once the real root is mounted, initramfs is discarded.

```
Bootloader loads into RAM:
┌─────────────┐  ┌──────────────┐
│   Kernel    │  │  initramfs   │
│   Image     │  │  (drivers,   │
│             │  │   mount      │
│             │  │   tools)     │
└─────────────┘  └──────────────┘
                       │
                       ▼
                 Mount real root
                 filesystem on disk
                       │
                       ▼
                 Switch to real root,
                 discard initramfs
```

## Real-World Connection

- **"My Linux server won't boot"** → Usually a GRUB misconfiguration or missing initramfs. Understanding the boot stages tells you where to look.
- **Secure Boot (UEFI)** → Firmware verifies bootloader signature → bootloader verifies kernel signature. Prevents bootkits from loading malicious kernels.
- **Fast boot on modern systems** → UEFI is faster than BIOS. systemd's parallel service startup dramatically reduced boot times compared to SysV init's sequential approach.
- **Cloud VM boot** → Cloud providers optimize boot by pre-building images with drivers baked in, using UEFI direct boot, and minimizing initramfs.

## Interview Angle

**Q: What happens when you press the power button on a computer?**
> The power supply stabilizes and the CPU starts executing firmware (BIOS/UEFI) from a hardcoded address. The firmware runs POST to test hardware, then finds and loads a bootloader from the configured boot device. The bootloader loads the OS kernel and an initial RAM filesystem into memory. The kernel initializes memory management, the scheduler, and hardware drivers, mounts the root filesystem, and starts the first user-space process (PID 1, typically systemd on Linux). That init process then starts all system services, eventually presenting a login prompt.

**Q: What is the role of GRUB in the boot process?**
> GRUB is a bootloader — it bridges firmware and the kernel. It presents a boot menu (if multiple OSes exist), locates the kernel image and initramfs on disk, loads them into memory, passes kernel parameters, and transfers control to the kernel. Without a bootloader, the firmware wouldn't know how to find or load the OS kernel.

**Q: What is initramfs and why is it needed?**
> initramfs is a temporary root filesystem loaded into RAM by the bootloader alongside the kernel. It contains the drivers and tools needed to access the real root filesystem on disk. It solves a bootstrap problem: the kernel needs disk drivers to read the root filesystem, but those drivers are stored on the root filesystem. Once the real root is mounted, initramfs is discarded.

---

**You've completed the Introduction section!** You now have the foundation: what an OS is, how it evolved, how it's structured, the user/kernel boundary, and how it boots.

**Next section:** [01-System-Calls-and-Kernel](../01-System-Calls-and-Kernel/) — Now that you know the kernel exists and runs in privileged mode, let's explore how your programs actually talk to it.
