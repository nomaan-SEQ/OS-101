# Windows Boot Process: UEFI to Desktop

## Overview: The Complete Boot Chain

Booting Windows is a carefully orchestrated sequence of handoffs, each stage loading and verifying the next. Understanding this chain is critical for troubleshooting boot failures, analyzing rootkits, and understanding how Secure Boot protects the system.

The full chain looks like this:

```
UEFI Firmware
    |
    v
bootmgfw.efi (Windows Boot Manager)
    |   Reads BCD (Boot Configuration Data)
    v
winload.efi (Windows OS Loader)
    |   Loads ntoskrnl.exe, hal.dll, boot-start drivers
    |   Initializes SYSTEM registry hive
    v
ntoskrnl.exe (NT Kernel + Executive)
    |   Phase 0: single-threaded init (memory, objects, security)
    |   Phase 1: multi-threaded init (PnP, I/O, drivers, registry)
    v
smss.exe (Session Manager)
    |   Creates session 0 and session 1
    |   Launches csrss.exe and wininit.exe / winlogon.exe
    v
wininit.exe (Session 0)          winlogon.exe (Session 1)
    |                                 |
    v                                 v
services.exe (SCM)               LogonUI.exe
lsass.exe (Security)                 |
                                     v
                              User authenticates
                                     |
                                     v
                              userinit.exe -> Explorer.exe
                              (Your desktop)
```

Let us walk through each stage.

## Stage 1: UEFI Firmware

The UEFI firmware initializes hardware (CPU, memory, storage controllers, display) and looks for a boot loader. On a UEFI system, the firmware reads the **EFI System Partition (ESP)** -- a FAT32 partition containing boot loaders.

The firmware's boot order determines which `.efi` file to execute. For Windows, this is typically `\EFI\Microsoft\Boot\bootmgfw.efi`.

**Secure Boot** is enforced here: the firmware checks that `bootmgfw.efi` is signed by a trusted certificate (Microsoft's key is pre-loaded in the firmware). If the signature is invalid, the firmware refuses to load it. This prevents rootkits from replacing the boot manager.

In Linux, the equivalent is the firmware loading `grubx64.efi` (GRUB) or `systemd-bootx64.efi`. Secure Boot requires a signed boot loader -- distributions either use Microsoft's signing key (via shim) or their own key enrolled in the firmware.

## Stage 2: Windows Boot Manager (bootmgfw.efi)

The Boot Manager reads the **BCD (Boot Configuration Data)** store -- a registry-like binary database that contains boot entries. BCD is Windows' version of GRUB's grub.cfg -- it stores which OS to boot, where the kernel is, and boot parameters. But instead of a text file, it is a registry-like binary database.

```
BCD Store Contents (simplified):
+------------------------------------------+
| {bootmgr}                                |
|   displayorder = {entry1}, {entry2}       |
|   timeout = 30                            |
+------------------------------------------+
| {entry1} - Windows 11                     |
|   device = partition=C:                   |
|   path = \Windows\system32\winload.efi    |
|   systemroot = \Windows                   |
|   nx = OptIn                              |
+------------------------------------------+
| {entry2} - Windows 10 (dual boot)         |
|   device = partition=D:                   |
|   path = \Windows\system32\winload.efi    |
+------------------------------------------+
```

You manage BCD with `bcdedit.exe` (command line) or `msconfig.exe` (GUI). Common operations:
- `bcdedit /set {current} nx AlwaysOn` -- enable DEP (NX bit)
- `bcdedit /set {current} bootlog Yes` -- enable boot logging
- `bcdedit /set {current} safeboot minimal` -- boot into Safe Mode

If there are multiple boot entries, the Boot Manager displays a menu (or auto-selects after the timeout). Once a selection is made, it hands off to the OS loader.

## Stage 3: winload.efi (Windows OS Loader)

`winload.efi` is the workhorse of the boot process. It runs in UEFI mode (not yet in the Windows kernel) and performs critical loading tasks:

1. **Loads ntoskrnl.exe** (the kernel) into memory
2. **Loads hal.dll** (Hardware Abstraction Layer)
3. **Loads the SYSTEM registry hive** from `C:\Windows\System32\config\SYSTEM` -- this is needed because boot-start driver configuration is in the registry
4. **Loads boot-start drivers** -- drivers marked as `Start=0` in the registry (storage drivers, filesystem drivers, early security drivers)
5. **Verifies Secure Boot signatures** on all loaded binaries
6. **Sets up the page tables** for the kernel address space
7. **Transfers control** to `ntoskrnl.exe`'s entry point

This is analogous to GRUB loading the Linux kernel (`vmlinuz`) and the initial RAM filesystem (`initramfs`). The key difference: Linux's initramfs is a temporary root filesystem that contains drivers and tools needed to mount the real root. Windows loads those drivers directly via winload.efi using the registry as a manifest, with no equivalent of initramfs.

## Stage 4: Kernel Initialization (ntoskrnl.exe)

Kernel initialization happens in two phases, like building a house. Phase 0 lays the foundation (memory, objects, security) with one worker. Phase 1 brings in the full crew (device detection, driver loading, services) working in parallel.

### Phase 0: Single-Threaded Foundation

Phase 0 runs on the boot processor (BSP) only, with interrupts disabled:

1. **Memory Manager** initialization: set up page tables, PFN database, memory pools
2. **Object Manager** initialization: create the root object directory, basic types
3. **Security Reference Monitor** initialization: set up security infrastructure
4. **Process Manager** initialization: create the System process (PID 4) and the initial system thread
5. **Plug and Play Manager** basic initialization
6. **Executive** initialization of remaining components

At the end of Phase 0, the system has one process (System), one thread, basic memory management, and the object namespace. No drivers are running yet (beyond those loaded by winload.efi).

### Phase 1: Multi-Threaded Startup

Phase 1 enables interrupts, starts additional processors (if multiprocessor), and initializes the full I/O subsystem:

1. **Start additional CPUs** (Application Processors) and their schedulers
2. **I/O Manager** initialization: load remaining boot-start and system-start drivers
3. **PnP Manager** full initialization: enumerate devices on all buses, load drivers
4. **Registry** fully loaded and operational
5. **Cache Manager** initialization
6. **Power Manager** initialization
7. Create the **smss.exe** (Session Manager) process -- the first user-mode process

```
Phase 0 (single-threaded)         Phase 1 (multi-threaded)
+----------------------------+    +----------------------------+
| Memory Manager init        |    | Start additional CPUs      |
| Object Manager init        |    | I/O Manager init           |
| Security RM init           |    | PnP device enumeration     |
| Process Manager init       |    | Driver loading             |
| Create System process      |    | Registry fully loaded      |
|                            |    | Cache Manager init         |
|                            |    | Create smss.exe            |
+----------------------------+    +----------------------------+
    Foundation laid                   Full house built
```

## Stage 5: Session Manager (smss.exe)

**smss.exe** is the first user-mode process, and it acts as the construction foreman for the user-mode environment:

1. **Creates the Win32 subsystem** by launching csrss.exe (Client/Server Runtime Subsystem)
2. **Creates Session 0** (services session) and Session 1 (first interactive user session)
3. **Launches wininit.exe** (in Session 0) -- starts system services
4. **Launches winlogon.exe** (in Session 1) -- handles user logon
5. Processes the **BootExecute** registry value (runs chkdsk if needed)
6. Runs **pending file rename operations** (updates that require a reboot)

Session separation is important for security. Session 0 runs services in an isolated session that cannot directly interact with user desktops. This prevents "shatter attacks" where a service's window messages could be exploited by a malicious user. Linux achieves similar isolation through separate cgroups, namespaces, and D-Bus for service communication.

## Stage 6: Services and Logon

### wininit.exe (Session 0)

wininit.exe starts two critical processes:

- **services.exe (Service Control Manager / SCM)**: The master process for Windows services. It reads the registry (`HKLM\SYSTEM\CurrentControlSet\Services`) and starts services marked for automatic startup. This is the Windows equivalent of systemd's service management.

- **lsass.exe (Local Security Authority Subsystem Service)**: Handles authentication. When you type your password, lsass.exe validates it against the SAM database (local accounts) or Active Directory (domain accounts). It also generates access tokens for logged-in users.

### winlogon.exe (Session 1)

winlogon.exe manages the interactive logon:

1. Displays **LogonUI.exe** (the lock screen / login screen)
2. User enters credentials
3. Credentials sent to **lsass.exe** for validation
4. lsass.exe creates an access token for the user
5. winlogon.exe launches **userinit.exe** with the user's token
6. userinit.exe processes logon scripts and launches **Explorer.exe** (the desktop shell)
7. userinit.exe exits -- Explorer.exe is now the user's shell process

```
winlogon.exe
    |
    v
LogonUI.exe (shows login screen)
    |  User types credentials
    v
lsass.exe (validates credentials)
    |  Creates access token
    v
userinit.exe (logon scripts, profile load)
    |
    v
Explorer.exe (desktop shell, taskbar, Start menu)
```

## Complete Boot Chain Diagram

```
Time ------>

[UEFI]  [bootmgr]  [winload]  [ntoskrnl]       [smss]      [services]  [logon]
  |        |          |         |    |             |            |           |
  +--HW--->+--BCD---->+--load-->+    |             |            |           |
  init     read       kernel   Ph0  Ph1           |            |           |
                      hal      init init          |            |           |
                      drivers       |             |            |           |
                      SYSTEM        +--create---->+            |           |
                      hive                        |--csrss---->|           |
                                                  |--wininit-->+--svc.exe  |
                                                  |            +--lsass    |
                                                  |--winlogon------------>+
                                                                          |
                                                                    LogonUI
                                                                    lsass auth
                                                                    userinit
                                                                    Explorer.exe
                                                                         |
                                                                      DESKTOP
```

## Boot Timing

On a modern NVMe-equipped system:

| Phase | Approximate Time | What Happens |
|---|---|---|
| UEFI firmware | 1-5 seconds | Hardware init, POST |
| bootmgfw.efi | < 1 second | Read BCD, select OS |
| winload.efi | 1-3 seconds | Load kernel, drivers, registry |
| Kernel Phase 0 | < 1 second | Foundation init |
| Kernel Phase 1 | 2-5 seconds | PnP enumeration, driver init |
| smss to services | 1-3 seconds | Session creation, service startup |
| Logon to desktop | 2-10 seconds | User profile, startup apps |
| **Total** | **~10-25 seconds** | UEFI to usable desktop |

## Safe Mode

Safe Mode loads a minimal set of drivers and services, defined by registry entries under `HKLM\SYSTEM\CurrentControlSet\Control\SafeBoot`. There are two variants:

- **Safe Mode (Minimal)**: Basic drivers (VGA, keyboard, mouse, disk), no networking
- **Safe Mode with Networking**: Adds network drivers and services

Safe Mode is set via BCD: `bcdedit /set {current} safeboot minimal`. During boot, the kernel checks this flag and only loads drivers and services listed under the SafeBoot registry key.

In Linux, the equivalent is booting to `rescue.target` or `emergency.target` with systemd, or adding `single` to the kernel command line for single-user mode.

## Boot Logging

Enable boot logging with `bcdedit /set {current} bootlog yes`. This creates `C:\Windows\ntbtlog.txt`, which records every driver loaded (or not loaded) during boot. This is invaluable for troubleshooting driver-related boot failures.

Equivalent Linux tools: `dmesg` for kernel messages, `systemd-analyze blame` for service startup times, and `journalctl -b` for boot logs.

## Comparison: Linux Boot vs Windows Boot

| Stage | Linux | Windows |
|---|---|---|
| Firmware | UEFI | UEFI |
| Boot loader | GRUB / systemd-boot | bootmgfw.efi (Boot Manager) |
| Boot config | grub.cfg (text file) | BCD (binary registry) |
| Kernel loader | GRUB loads vmlinuz | winload.efi loads ntoskrnl.exe |
| Initial drivers | initramfs (cpio archive) | Boot-start drivers from registry |
| Kernel init | start_kernel() -> rest_init() | Phase 0 (single-threaded) + Phase 1 (multi-threaded) |
| First user process | init / systemd (PID 1) | smss.exe |
| Service manager | systemd | services.exe (SCM) |
| Authentication | PAM + login | lsass.exe + LogonUI |
| Shell | bash / desktop environment | Explorer.exe |
| Safe mode | rescue.target / single | Safe Mode (via BCD safeboot) |
| Boot log | dmesg, journalctl -b | ntbtlog.txt |

A key architectural difference: Linux uses initramfs as a temporary root filesystem containing just enough to mount the real root. Windows has no equivalent -- winload.efi loads boot-start drivers directly using the registry as a manifest. This means Windows needs to know more about the hardware at boot time (the registry must be correct), while Linux can be more flexible (initramfs can probe and adapt).

## Real-World Connection

When a Windows system fails to boot, a support engineer follows the boot chain to identify where it breaks:
- Black screen after UEFI? Boot Manager not found (EFI partition issue, Secure Boot problem).
- Blue screen during startup? Driver failure during kernel Phase 1 (check ntbtlog.txt).
- Hangs at "Starting Windows"? Service dependency deadlock (boot into Safe Mode, disable services).
- Logon screen appears but profile fails to load? User profile corruption (check NTUSER.DAT).

When a security researcher analyzes a bootkit (malware that infects the boot process), they trace the boot chain: Does the malware replace bootmgfw.efi? Does it patch winload.efi? Does it inject a malicious boot-start driver? Secure Boot and Measured Boot (TPM attestation) were designed specifically to make each stage verifiable, creating a "chain of trust" from firmware to kernel.

When a system administrator deploys Windows at enterprise scale, they use Windows PE (Preinstallation Environment) -- a minimal Windows boot environment that runs from RAM. Windows PE boots through the same chain (bootmgr -> winload -> minimal kernel) but loads a temporary image instead of a full Windows installation. It is used for imaging, recovery, and deployment tasks.

## Interview Angle

**Q: Walk through the Windows boot process from firmware to desktop.**

A: The UEFI firmware initializes hardware and loads bootmgfw.efi (Windows Boot Manager) from the EFI System Partition, verifying its Secure Boot signature. The Boot Manager reads the BCD store (a binary registry of boot entries) and launches winload.efi. winload.efi loads ntoskrnl.exe, hal.dll, the SYSTEM registry hive, and boot-start drivers into memory. The kernel initializes in two phases: Phase 0 is single-threaded and sets up the memory manager, object manager, and security reference monitor. Phase 1 enables multiprocessing, enumerates PnP devices, and loads remaining drivers. The kernel creates smss.exe (Session Manager), the first user-mode process, which creates sessions and launches csrss.exe, wininit.exe, and winlogon.exe. wininit.exe starts services.exe (the Service Control Manager) and lsass.exe (authentication). winlogon.exe presents the login screen, and after authentication, launches Explorer.exe as the user's shell.

**Q: How does the Windows boot process differ from Linux?**

A: The main structural differences are: (1) Windows uses a binary BCD store for boot configuration while Linux uses text-based grub.cfg. (2) Windows loads boot-start drivers directly via winload.efi using registry information, while Linux uses initramfs as a temporary filesystem to discover and load drivers dynamically. (3) Windows kernel initialization has explicit Phase 0 (single-threaded) and Phase 1 (multi-threaded) stages. (4) Windows uses separate processes for different boot roles (smss.exe, csrss.exe, services.exe, lsass.exe, winlogon.exe) while Linux consolidates much of this in systemd (PID 1). (5) Windows Secure Boot is integrated into the Microsoft signing infrastructure, while Linux distributions typically use a shim signed by Microsoft to chain-load their own signed bootloader.

---

Next: [Registry and System Services](08-windows-registry-and-system-services.md)
