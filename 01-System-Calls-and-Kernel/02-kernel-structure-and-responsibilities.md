# Kernel Structure and Responsibilities

## What the Kernel Does

The kernel is the core of the operating system. It's the one piece of software that runs in privileged mode (Ring 0) with full access to hardware. Everything else -- your applications, your shell, even your GUI -- runs in user mode and must ask the kernel for anything that requires privilege.

The kernel has six major responsibilities:

```
KERNEL RESPONSIBILITIES
========================

  +----------------------------------------------------+
  |                    THE KERNEL                       |
  |                                                    |
  |  +------------+  +-------------+  +------------+  |
  |  |  Process   |  |   Memory    |  |    File    |  |
  |  | Management |  | Management  |  |  Systems   |  |
  |  +------------+  +-------------+  +------------+  |
  |                                                    |
  |  +------------+  +-------------+  +------------+  |
  |  |  Device    |  |  Networking |  |  Security  |  |
  |  |  Drivers   |  |   Stack     |  |  & Access  |  |
  |  +------------+  +-------------+  +------------+  |
  +----------------------------------------------------+
         |                |                |
    +---------+     +---------+     +---------+
    |   CPU   |     | Memory  |     |  Disk/  |
    |         |     |  (RAM)  |     | Network |
    +---------+     +---------+     +---------+
```

| Responsibility | What It Does | Key Syscalls |
|---------------|-------------|-------------|
| **Process Management** | Create, schedule, and terminate processes; manage CPU time | `fork`, `exec`, `wait`, `exit`, `kill`, `sched_setscheduler` |
| **Memory Management** | Virtual memory, page tables, allocation, swapping | `mmap`, `munmap`, `brk`, `mprotect`, `madvise` |
| **File Systems** | VFS layer, file operations, directory management, mount | `open`, `read`, `write`, `close`, `stat`, `mount` |
| **Device Drivers** | Interface between hardware and kernel subsystems | `ioctl`, `read`, `write` (on device files) |
| **Networking** | TCP/IP stack, socket management, packet routing | `socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv` |
| **Security & Access** | User/group permissions, capabilities, namespaces, SELinux | `setuid`, `setgid`, `capset`, `seccomp`, `unshare` |

> **VFS (Virtual File System)** is an abstraction layer that lets the kernel support multiple file system types (ext4, NTFS, FAT32, etc.) through a single uniform interface. Your program calls the same `read()`/`write()` syscalls regardless of the underlying filesystem — VFS routes the call to the correct driver.

## Kernel Address Space vs User Address Space

On a 64-bit Linux system, each process sees a virtual address space. The kernel reserves the upper portion for itself, and the lower portion is available to the user process:

```
VIRTUAL ADDRESS SPACE LAYOUT (x86-64 Linux)
=============================================

  0xFFFFFFFFFFFFFFFF  +-------------------------+
                      |                         |
                      |    KERNEL SPACE          |
                      |    (shared across all    |
                      |     processes)           |
                      |                         |
                      |  - kernel code & data   |
                      |  - kernel stacks        |
                      |  - page tables          |
                      |  - device memory maps   |
                      |                         |
  0xFFFF800000000000  +-------------------------+  <- Kernel boundary
                      |                         |
                      |    NON-CANONICAL HOLE    |  <- Addresses here are
                      |    (unmapped, invalid)   |     invalid on x86-64
                      |                         |
  0x00007FFFFFFFFFFF  +-------------------------+  <- Top of user space
                      |        Stack            |  <- grows downward
                      |          |              |
                      |          v              |
                      |                         |
                      |    (unmapped gap)       |
                      |                         |
                      |          ^              |
                      |          |              |
                      |        Heap             |  <- grows upward (brk/mmap)
                      +-------------------------+
                      |   BSS (uninitialized)   |
                      |   Data (initialized)    |
                      |   Text (code)           |
  0x0000000000400000  +-------------------------+  <- Program start
                      |   (unmapped, null page) |
  0x0000000000000000  +-------------------------+
```

Key points:
- **Kernel space is mapped into every process's address space** but is only accessible in Ring 0. If user code tries to read a kernel address, the MMU triggers a page fault, and the kernel kills the process with a segfault.
- **The kernel address space is shared** -- there's only one kernel, and it appears at the same virtual addresses in every process.
- **KPTI (Kernel Page Table Isolation)**, introduced as a Meltdown mitigation, actually unmaps most kernel pages when running in user mode, adding overhead but closing a hardware vulnerability.

## Kernel Threads vs User Processes

Not everything running on your system is a user-space process. The kernel itself creates **kernel threads** to handle background work:

| Aspect | User Process | Kernel Thread |
|--------|-------------|--------------|
| **Runs in** | User mode (Ring 3), traps into kernel as needed | Kernel mode (Ring 0) permanently |
| **Address space** | Has its own virtual address space | Shares the kernel address space |
| **Created by** | `fork()` / `clone()` | `kthread_create()` inside the kernel |
| **Visible in `ps`?** | Yes | Yes -- shown in brackets like `[kworker/0:1]` |
| **Examples** | `nginx`, `bash`, `python` | `[ksoftirqd]`, `[kworker]`, `[migration]`, `[kswapd]` |
| **Can access user memory?** | Yes (its own) | Not directly -- must use `copy_from_user()` |

Run `ps aux` on any Linux system and you'll see kernel threads in square brackets. Common ones:

- **`[kswapd0]`** -- handles memory page swapping/reclaim
- **`[ksoftirqd/N]`** -- processes deferred soft interrupts on CPU N
- **`[kworker/N:M]`** -- general-purpose kernel work queues
- **`[migration/N]`** -- handles moving processes between CPUs

## The Kernel Is Event-Driven

The kernel doesn't "run" in the traditional sense of having a main loop. Instead, it **reacts to events**:

```
KERNEL EVENT SOURCES
=====================

  +------------------+     +------------------+     +------------------+
  |   SYSTEM CALLS   |     |    INTERRUPTS    |     |   EXCEPTIONS     |
  |                  |     |                  |     |                  |
  | User program     |     | Hardware device  |     | CPU detects an   |
  | explicitly       |     | signals the CPU  |     | error condition  |
  | requests service |     | (timer, disk,    |     | (div by zero,    |
  |                  |     |  network, kbd)   |     |  page fault,     |
  |                  |     |                  |     |  invalid opcode) |
  +--------+---------+     +--------+---------+     +--------+---------+
           |                        |                        |
           v                        v                        v
  +-----------------------------------------------------------+
  |                      KERNEL ENTRY POINT                    |
  |   Determine event type -> dispatch to appropriate handler  |
  +-----------------------------------------------------------+
           |
           v
  +-----------------------------------------------------------+
  |                   KERNEL HANDLER EXECUTES                  |
  |   Process the event, update data structures, schedule work |
  +-----------------------------------------------------------+
           |
           v
  +-----------------------------------------------------------+
  |                     RETURN / SCHEDULE                       |
  |   Return to user process, or context-switch to another     |
  +-----------------------------------------------------------+
```

This means the kernel runs in three contexts:
1. **Process context**: executing a syscall on behalf of a user process (has a "current" process)
2. **Interrupt context**: handling a hardware interrupt (no "current" process, cannot sleep)
3. **Kernel thread context**: running a kernel thread (can sleep, has its own scheduling)

## Key Kernel Data Structures

Understanding these structures helps you reason about what the kernel tracks:

### `task_struct` (the process descriptor)

Every process/thread in Linux has a `task_struct`. This is a large struct (~6KB) containing everything the kernel needs to know about a task:

```
task_struct (simplified)
========================
  - pid, tgid              // process ID, thread group ID
  - state                  // RUNNING, SLEEPING, STOPPED, ZOMBIE
  - mm (memory descriptor) // points to page tables, VMAs
  - files                  // file descriptor table
  - signal handlers        // what to do on SIGTERM, etc.
  - scheduling info        // priority, time slice, CPU affinity
  - parent / children      // process tree links
  - cgroups, namespaces    // container isolation info
```

### File Descriptor Table

Each process has a table mapping **file descriptor numbers** (0, 1, 2, ...) to kernel **file objects**:

```
  Process A's FD Table          Kernel File Objects
  =====================         ===================
  
  fd 0 --------------------->  { /dev/tty, read mode, offset=0 }     (stdin)
  fd 1 --------------------->  { /dev/tty, write mode, offset=0 }    (stdout)
  fd 2 --------------------->  { /dev/tty, write mode, offset=0 }    (stderr)
  fd 3 --------------------->  { /home/user/data.txt, rw, offset=42 }
  fd 4 --------------------->  { socket: 10.0.0.1:8080, TCP }
```

This is why `dup2()` works -- it just copies a pointer in this table. And why `fork()` is interesting -- the child gets a **copy** of the parent's FD table (both pointing to the same kernel file objects, sharing the file offset).

### Page Tables

**Page tables** are data structures the kernel uses to map virtual addresses to physical addresses — we cover them in depth in section 07 (Memory Management). They're a tree structure (4 levels on x86-64) that the MMU hardware walks on every memory access:

```
  Virtual Address --> [PML4] -> [PDPT] -> [PD] -> [PT] -> Physical Page
                      Level 4   Level 3   Level 2  Level 1
```

We'll cover page tables in depth in the Memory Management section. For now, know that each process has its own page table tree (pointed to by `CR3` register), and the kernel has its own mappings within it.

---

## Real-World Connection

- **Container isolation** (Docker, Kubernetes) relies on kernel data structures. Linux namespaces modify what a `task_struct` can see (PID namespace gives the process its own PID 1; network namespace gives it isolated interfaces). Cgroups limit how much CPU/memory the task can use. None of this requires a separate kernel -- it's all bookkeeping within these structures.

- **`/proc` filesystem**: When you read `/proc/1234/status`, you're actually reading formatted output from that process's `task_struct`. The `/proc` filesystem is a window into kernel data structures, rendered as files.

- **OOM Killer**: When the system runs out of memory, the kernel walks the list of `task_struct`s, scores each process based on memory usage and other factors, and kills the highest-scoring one. Understanding `task_struct` helps you understand why your process got OOM-killed (check `/proc/<pid>/oom_score`).

---

## Interview Angle

**Q: What is the kernel responsible for, and why can't these things be done in user space?**

A: The kernel manages hardware resources (CPU scheduling, memory allocation, device I/O, networking) and enforces protection between processes. These require privileged CPU instructions (writing page tables, programming interrupt controllers, accessing I/O ports) that are only available in Ring 0. User-space programs can't directly control hardware because the CPU itself prevents it -- the MMU blocks access to kernel memory, and privileged instructions trigger a general protection fault if executed in Ring 3. The kernel provides controlled access through the syscall interface.

**Q: What's the difference between kernel space and user space in the virtual address layout?**

A: On x86-64 Linux, the upper half of the virtual address space (above `0xFFFF800000000000`) is reserved for the kernel and is mapped identically in every process's page tables. The lower half (below `0x00007FFFFFFFFFFF`) is per-process user space containing the program's code, heap, stack, and shared libraries. The kernel pages are present in every process's page table but marked as supervisor-only -- the MMU hardware prevents Ring 3 code from accessing them. With KPTI enabled, most kernel pages are actually unmapped when running in user mode, requiring a page table switch on every syscall entry/exit.

---

**Next**: [03-system-call-mechanism-traps-and-interrupts.md](03-system-call-mechanism-traps-and-interrupts.md)
