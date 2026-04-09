# Windows Architecture Overview and the NT Kernel

## The NT Hybrid Architecture

Windows NT was designed like a government with clear separation of powers. The kernel handles low-level CPU and interrupt management, the Executive provides higher-level services (like government agencies), and subsystems translate for different application types. This layered design came from Dave Cutler's experience building VMS at DEC, and it has survived three decades because the boundaries are well-defined.

Technically, NT is a **hybrid kernel**. The original design philosophy was microkernel-like: separate components with clean interfaces, environment subsystems running in user mode, and a small kernel core. In practice, performance requirements pulled more code into kernel mode over the years (notably, the graphics subsystem moved into the kernel in NT 4.0). The result is a kernel that has microkernel structure but monolithic performance characteristics.

If you know Linux, here is the key mental shift: Linux is openly monolithic -- the entire kernel is one big binary (vmlinuz) with loadable modules. Windows NT pretends to be layered -- the kernel (ntoskrnl.exe) contains both the small "Kernel" and the large "Executive," but they run in the same address space. The layering is architectural, not a hard protection boundary.

## The Layers of Windows

```
+------------------------------------------------------------------+
|                        USER MODE                                  |
|                                                                  |
|  +------------------+  +------------------+  +-----------------+ |
|  | Win32 Application|  | .NET Application |  | WSL / POSIX App | |
|  +--------+---------+  +--------+---------+  +--------+--------+ |
|           |                     |                     |          |
|  +--------v---------+  +-------v----------+  +-------v--------+ |
|  | Windows API       |  | CLR / CoreCLR    |  | Pico Provider  | |
|  | (kernel32.dll,    |  |                  |  | (lxss.sys)     | |
|  |  user32.dll,      |  +-------+----------+  +-------+--------+ |
|  |  advapi32.dll)    |          |                     |          |
|  +--------+----------+          |                     |          |
|           |                     |                     |          |
|  +--------v---------------------v---------------------v--------+ |
|  |                     ntdll.dll                                | |
|  |            (Native API / Syscall Stubs)                      | |
|  +-----------------------------+--------------------------------+ |
|                                |                                  |
|================================|==================================|
|                                | syscall instruction              |
|                        KERNEL MODE                                |
|                                |                                  |
|  +-----------------------------v--------------------------------+ |
|  |                  EXECUTIVE (ntoskrnl.exe)                    | |
|  |                                                              | |
|  |  +-----------+ +-----------+ +---------+ +----------------+  | |
|  |  | Process   | | Memory    | | I/O     | | Security Ref.  |  | |
|  |  | Manager   | | Manager   | | Manager | | Monitor        |  | |
|  |  +-----------+ +-----------+ +---------+ +----------------+  | |
|  |  +-----------+ +-----------+ +---------+ +----------------+  | |
|  |  | Object    | | Cache     | | PnP     | | Power          |  | |
|  |  | Manager   | | Manager   | | Manager | | Manager        |  | |
|  |  +-----------+ +-----------+ +---------+ +----------------+  | |
|  +--------------------------------------------------------------+ |
|  +--------------------------------------------------------------+ |
|  |                    KERNEL (ke)                                | |
|  |    Thread scheduling, interrupt dispatch, synchronization     | |
|  +--------------------------------------------------------------+ |
|  +--------------------------------------------------------------+ |
|  |              HAL (Hardware Abstraction Layer)                 | |
|  |      hal.dll -- interrupt controllers, timers, DMA            | |
|  +--------------------------------------------------------------+ |
|                                |                                  |
+--------------------------------|----------------------------------+
                                 v
                            HARDWARE
```

### User Mode: From API Call to Syscall

When a Win32 application calls `CreateFile()`, the journey is:

1. **kernel32.dll** -- the Win32 API DLL. `CreateFile()` lives here. It validates parameters and translates the call.
2. **ntdll.dll** -- the *real* syscall interface. kernel32.dll calls `NtCreateFile()` in ntdll.dll.
3. **ntdll.dll** puts the syscall number in a register and executes the `syscall` instruction.
4. The CPU switches to kernel mode and jumps to **KiSystemCall64** in ntoskrnl.exe.
5. The kernel uses the **SSDT (System Service Descriptor Table)** to look up the function pointer and calls the kernel's `NtCreateFile()`.

This is a critical distinction: **ntdll.dll is the real syscall layer, not kernel32.dll**. kernel32.dll is a convenience wrapper that provides the documented Win32 API. Many advanced tools and malware call ntdll.dll directly, bypassing kernel32.dll entirely.

In Linux, the equivalent is: `glibc` wraps the raw `syscall` instruction. The `open()` function in glibc issues `syscall(__NR_openat, ...)`. The structural difference is that Windows has an extra layer (kernel32 -> ntdll -> kernel) while Linux typically has one (glibc -> kernel).

### Kernel Mode: Executive, Kernel, and HAL

**The Executive** is the upper layer of kernel mode. It contains the major OS subsystems:

| Component | Responsibility | Linux Equivalent |
|---|---|---|
| Process Manager | Create/terminate processes and threads | Part of kernel core (fork, exec, exit) |
| Memory Manager | Virtual memory, paging, working sets | mm subsystem |
| I/O Manager | Driver loading, IRP dispatch | Block/char device layer, VFS |
| Security Reference Monitor | Access checks on all object operations | LSM (Linux Security Modules) |
| Object Manager | Create/manage kernel objects, namespace | No direct equivalent (everything is a file) |
| Cache Manager | Unified file cache | Page cache |
| PnP Manager | Device detection, driver matching | udev + driver core |
| Power Manager | Sleep states, device power management | ACPI subsystem, pm_runtime |

**The Kernel** (lowercase k) is the small core that handles thread scheduling, interrupt dispatching, exception handling, and multiprocessor synchronization. It is the equivalent of the CPU scheduler and interrupt handling code in Linux.

**The HAL (Hardware Abstraction Layer)** is like a universal power adapter -- it lets the same kernel run on different hardware without knowing the specifics. The HAL abstracts interrupt controllers, timers, DMA, and other hardware-specific details. In Linux, this role is scattered across architecture-specific code in `arch/`.

## The Object Manager: Everything Is an Object

In Linux, the philosophy is "everything is a file." In Windows, the philosophy is "everything is an **object**." Processes, threads, files, registry keys, mutexes, semaphores, events, timers, and even the desktop itself are kernel objects managed by the Object Manager.

Each object has:
- A **type** (Process, Thread, File, Mutant, Section, etc.)
- A **security descriptor** (who can access it)
- A **reference count** (garbage collection)
- An optional **name** in the object namespace

The **object namespace** is a hierarchical directory structure inside the kernel:

```
\                           (root)
+-- Device                  (device objects: \Device\HarddiskVolume1)
+-- BaseNamedObjects        (named mutexes, events, sections)
+-- Sessions                (per-session objects)
+-- ObjectTypes             (object type definitions)
+-- GLOBAL??                (symbolic links: C: -> \Device\HarddiskVolume2)
```

You can explore this namespace with **WinObj** (Sysinternals tool). It is the Windows equivalent of browsing `/dev`, `/proc`, and `/sys` in Linux.

**Handles** are how user-mode code references kernel objects. When you call `CreateFile()` and get back a `HANDLE`, that handle is an index into a per-process **handle table**. The handle table maps handle values to pointers to kernel objects. This is exactly like file descriptors in Linux -- `fd 3` is an index into the process's file descriptor table, which points to a kernel `struct file`.

| Concept | Windows | Linux |
|---|---|---|
| Kernel object reference | Handle (HANDLE) | File descriptor (int fd) |
| Object table | Per-process handle table | Per-process fd table |
| Object identity | Object pointer + type | Inode (for files), task_struct (for processes) |
| Object naming | Object namespace (\BaseNamedObjects) | Filesystem (/dev, /proc, /tmp) |
| Close | CloseHandle() | close() |

## System Call Mechanism

The syscall path on x86-64 Windows:

1. User code calls ntdll.dll function (e.g., `NtCreateFile`)
2. ntdll.dll loads the syscall number into `EAX` (e.g., `0x0055` for NtCreateFile)
3. `syscall` instruction executes
4. CPU switches to ring 0, jumps to `KiSystemCall64` (address stored in MSR)
5. `KiSystemCall64` saves user-mode registers, looks up function pointer in the **SSDT**
6. Dispatches to the appropriate kernel function
7. Returns result via `sysret` instruction

The **SSDT (System Service Descriptor Table)** is Windows' version of the Linux syscall table (`sys_call_table`). One key difference: Windows syscall numbers **change between OS versions**. Microsoft does not document them or guarantee stability. This is why you must call through ntdll.dll, never issue raw syscalls directly. Linux, by contrast, guarantees stable syscall numbers across versions.

## Executable Formats: PE vs ELF

| Feature | Windows PE (Portable Executable) | Linux ELF (Executable and Linkable Format) |
|---|---|---|
| Extension | .exe, .dll, .sys | No extension convention, .so for shared libs |
| Sections | .text, .rdata, .data, .rsrc | .text, .rodata, .data, .bss |
| Dynamic linking | Import Address Table (IAT) | GOT/PLT (Global Offset Table / Procedure Linkage Table) |
| Metadata | PE headers, Rich header | ELF headers, section headers |
| Debug info | PDB files (separate) | DWARF (embedded or separate) |
| Loader | ntdll.dll (LdrLoadDll) | ld-linux.so (dynamic linker) |

## Exploration Tools

| Tool | What It Shows | Linux Equivalent |
|---|---|---|
| Process Explorer (Sysinternals) | Processes, threads, handles, DLLs | ps, top, /proc/[pid] |
| WinObj (Sysinternals) | Object Manager namespace | /dev, /proc, /sys |
| WinDbg | Kernel debugger | gdb + kgdb |
| Process Monitor | Real-time file/registry/network activity | strace + inotifywait |
| WMI / PowerShell | System information queries | /proc, /sys, sysfs |

## Real-World Connection

When a performance engineer at a game studio sees frame drops, they reach for **Process Explorer** to check thread priorities, **Process Monitor** to watch I/O patterns, and **WinDbg** to analyze kernel stacks. Understanding the architecture -- that the call path goes Win32 API -> ntdll -> SSDT -> Executive component -> driver stack -- tells them exactly where to look for bottlenecks.

When a security researcher reverse-engineers malware, they trace ntdll.dll calls because that is where the real syscall interface lives. Malware that calls ntdll directly (bypassing kernel32) is trying to evade API-level hooks. Understanding the architecture reveals the evasion technique.

## Interview Angle

**Q: How does the Windows NT kernel architecture differ from Linux?**

A: Windows NT uses a hybrid architecture with three distinct kernel-mode layers: the HAL (hardware abstraction), the Kernel (scheduling, interrupts), and the Executive (process management, memory management, I/O, security). Linux is monolithic -- all kernel code runs in one layer with loadable modules. NT's Object Manager treats everything as an object (processes, files, mutexes), while Linux uses the "everything is a file" philosophy. The syscall path also differs: Windows goes through ntdll.dll -> SSDT -> Executive, while Linux goes through glibc -> sys_call_table -> kernel functions directly.

**Q: What is ntdll.dll and why is it significant?**

A: ntdll.dll is the lowest user-mode layer in Windows. It contains the actual syscall stubs that transition into kernel mode. The documented Win32 API (kernel32.dll, user32.dll) calls ntdll.dll internally. Unlike Linux syscall numbers, Windows syscall numbers are not stable across OS versions, so all user-mode code must go through ntdll.dll rather than issuing raw syscalls. This is also why ntdll.dll is a key monitoring point for security tools -- any operation that reaches the kernel must pass through it.

---

Next: [Process and Thread Model](02-windows-process-and-thread-model.md)
