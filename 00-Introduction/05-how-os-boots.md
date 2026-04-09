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
1. The **power supply** stabilizes and sends a "power good" signal to the motherboard.
2. The CPU **resets** to a known state and begins executing instructions at a hardcoded address (the **reset vector**).
   - On x86: `0xFFFFFFF0` (mapped to firmware ROM/flash)

At this point, RAM is empty, no OS is loaded, and the CPU is running in **real mode** (16-bit, no memory protection — a legacy from the 1970s).

### Step 2: Firmware (BIOS or UEFI)

The firmware is software stored on a flash chip on the motherboard. It does two things:

#### a) POST (Power-On Self-Test)
- Tests that critical hardware works: CPU, RAM, video, keyboard
- If something fails, you hear beep codes or see no display
- This is why you sometimes hear a single beep at startup — that's POST saying "all good"

#### b) Find and Load the Bootloader

**Legacy BIOS Path:**
1. BIOS looks at the **boot order** (configured in BIOS settings)
2. Reads the **first sector** (512 bytes) of the boot device — this is the **Master Boot Record (MBR)**
3. MBR contains a tiny bootloader program and partition table
4. BIOS loads that bootloader into memory and jumps to it

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

```
Kernel Entry Point
       │
       ▼
1. Switch to protected mode (32-bit) or long mode (64-bit)
       │
       ▼
2. Set up memory management
   - Initialize page tables
   - Enable virtual memory (paging)
       │
       ▼
3. Initialize core subsystems
   - Interrupt handlers (IDT)
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
