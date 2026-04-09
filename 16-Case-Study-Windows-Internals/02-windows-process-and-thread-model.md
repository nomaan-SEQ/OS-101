# Windows Process and Thread Model

## EPROCESS: The Master Process Record

Every running process in Windows is represented by an **EPROCESS** (Executive Process) structure in kernel memory. EPROCESS is Windows' version of Linux's `task_struct` -- the master record that contains everything the kernel needs to know about a process.

But here is the key difference: Linux uses one structure (`task_struct`) for both processes and threads. Windows has separate structures -- **EPROCESS** for the process container and **ETHREAD** for each thread within it. This is not just a naming difference. It reflects a fundamental design choice: Windows treats processes and threads as genuinely different things, while Linux treats threads as processes that share resources.

Think of EPROCESS like a company's corporate registration. It describes the company (process) itself: its name, its assets (address space), its employees (threads), its security badge (access token), and its resource allocations. Individual employees (threads) have their own records (ETHREAD) with their own schedules and work assignments.

### Key EPROCESS Fields

| Field | Purpose | Linux Equivalent |
|---|---|---|
| PEB pointer | Points to user-mode Process Environment Block | mm_struct + environment |
| Handle table | Maps handles to kernel objects | File descriptor table (files_struct) |
| Token | Security identity of the process | cred struct (uid, gid, capabilities) |
| VAD root | Root of Virtual Address Descriptor tree | mm_struct -> mmap (VMA list) |
| Thread list | Linked list of all ETHREAD structures | thread_group list |
| Image filename | Name of the executable | comm field in task_struct |
| Exit status | Process exit code | exit_code in task_struct |
| Session ID | Terminal Services / logon session | Session concept via cgroups/namespaces |
| Job pointer | Job object this process belongs to | cgroup pointer |

## PEB: The User-Mode Window into Process State

The **PEB (Process Environment Block)** is a structure in user-mode memory that contains process information accessible without a syscall. This is a clever performance optimization -- certain frequently-accessed data lives where user-mode code can read it directly.

```
+---------------------------+
|        EPROCESS           |  <-- Kernel mode (ring 0)
|  (kernel-only fields)     |
|  Token, Handle table,     |
|  VAD tree, Thread list    |
|         |                 |
|    PEB pointer --------+  |
+------------------------|--+
                         |
=========================|======= User/Kernel boundary
                         |
                         v
+---------------------------+
|          PEB              |  <-- User mode (ring 3)
|  Image base address       |
|  Loader data (DLL list)   |
|  Process heap pointer     |
|  Environment variables    |
|  Process parameters       |
|  API set schema           |
+---------------------------+
```

The PEB contains the loaded DLL list, the process heap, environment variables, and the image base address. Debuggers and profilers read the PEB constantly. Malware also reads and modifies the PEB to hide loaded DLLs -- a technique called PEB unlinking.

In Linux, some of this information lives in `/proc/[pid]/maps`, `/proc/[pid]/environ`, and the auxiliary vector. There is no single user-accessible structure equivalent to the PEB, though the VDSO (Virtual Dynamic Shared Object) serves a similar purpose for certain system data like the current time.

## ETHREAD and KTHREAD: Thread Representation

Each thread is represented by two nested structures:

- **ETHREAD (Executive Thread)**: the full thread structure visible to the Executive layer. Contains the thread's start address, impersonation token, IPC info, and a pointer to the owning EPROCESS.
- **KTHREAD (Kernel Thread)**: embedded inside ETHREAD. Contains the scheduling-critical fields: priority, quantum, wait state, kernel stack pointer, and the thread context (saved registers).

The separation exists because the Kernel layer (which does scheduling) only needs KTHREAD fields, while the Executive layer needs the full ETHREAD for things like security and I/O.

### TEB: The Thread Environment Block

Just as EPROCESS has a PEB, each ETHREAD has a **TEB (Thread Environment Block)** in user mode:

| TEB Field | Purpose |
|---|---|
| Stack base and limit | Stack boundaries for overflow detection |
| TLS (Thread Local Storage) array | Per-thread data slots |
| Last error value | GetLastError() reads this directly -- no syscall needed |
| Exception list | Structured Exception Handling (SEH) chain |
| Locale information | Per-thread locale settings |

The `GetLastError()` function is so fast precisely because it just reads a field from the TEB -- no kernel transition required. This is similar to how Linux uses the VDSO to avoid syscalls for `gettimeofday()`.

## Thread Priorities: A 32-Level System

Windows uses a **priority-based preemptive scheduler** with 32 priority levels (0-31):

```
Priority Level    Category
  31            +-- Real-time class (16-31)
  ...           |   Fixed priorities, no dynamic boosting
  16            +--
  15            +-- Dynamic class (1-15)
  ...           |   Subject to priority boosts
  1             +--
  0             +-- Zero Page Thread only (system idle task)
```

The effective priority is calculated from two inputs:

1. **Priority class** (set on the process): Idle, Below Normal, Normal, Above Normal, High, Realtime
2. **Priority level** (set on the thread): Idle, Lowest, Below Normal, Normal, Above Normal, Highest, Time Critical

These combine into a **base priority** number between 0 and 31. The scheduler always runs the highest-priority ready thread.

### Priority Boosts

Windows temporarily boosts thread priority in specific situations:

- **I/O completion**: A thread that just finished waiting for disk I/O gets a boost (encourages I/O-bound threads to run promptly and release resources).
- **Window input**: The thread that owns the foreground window gets a priority boost (makes the UI feel responsive).
- **Starvation prevention**: If a ready thread has not run for about 4 seconds, Windows boosts it to 15 temporarily (prevents priority inversion).

After a boost, the priority decays back to the base over successive quantum expirations.

In Linux, the CFS (Completely Fair Scheduler) uses a different approach: virtual runtime and a red-black tree rather than fixed priority levels. Linux does have real-time scheduling classes (SCHED_FIFO, SCHED_RR) with priorities 1-99, which are conceptually similar to Windows' real-time range.

## Jobs: Resource Limits for Process Groups

A **Job object** is like a corporate department budget. All processes in the job share limits on CPU time, memory, and handles -- like cgroups in Linux.

```
+-------------------------------------------+
|               JOB OBJECT                  |
|  CPU time limit: 60 seconds               |
|  Memory limit: 512 MB                     |
|  Max processes: 10                         |
|                                            |
|  +----------+  +----------+  +----------+ |
|  | Process A|  | Process B|  | Process C| |
|  | (and its |  | (and its |  | (and its | |
|  | threads) |  | threads) |  | threads) | |
|  +----------+  +----------+  +----------+ |
+-------------------------------------------+
```

Job objects can limit:
- Total and per-process CPU time
- Working set size (memory)
- Number of active processes
- UI restrictions (clipboard, display settings)
- Network bandwidth (Windows 10+)

| Feature | Windows Jobs | Linux cgroups |
|---|---|---|
| Scope | Group of processes | Group of processes |
| CPU limits | CPU time limit, CPU rate control | cpu.max, cpu.shares |
| Memory limits | Working set limits | memory.max, memory.high |
| Nesting | Nested jobs (Windows 8+) | Hierarchical cgroups (v2) |
| Accounting | Per-job CPU/memory accounting | Per-cgroup accounting |
| I/O limits | Limited (bandwidth only) | io.max, io.weight |
| Network limits | Net rate control (Win10+) | Via eBPF / tc |

Modern Windows uses Job objects extensively. Every UWP (Universal Windows Platform) app runs inside a Job object, and Windows containers (Server Containers, Hyper-V Containers) use Jobs with silos for isolation.

## Fibers: User-Mode Cooperative Threads

**Fibers** are cooperative, user-mode scheduling units. Think of them as "threads within a thread" that you schedule yourself, without kernel involvement.

```
        Thread A
    +-------------------+
    |  Fiber 1 (active) |  <-- Currently running
    |  Fiber 2 (suspended)|
    |  Fiber 3 (suspended)|
    +-------------------+
```

Key APIs:
- `ConvertThreadToFiber()` -- turns the calling thread into a fiber
- `CreateFiber()` -- creates a new fiber with its own stack
- `SwitchToFiber()` -- cooperatively switches to another fiber

Fibers are like coroutines or green threads. The kernel sees only the underlying thread; fiber switching is just a user-mode context save/restore. This makes switching extremely fast (no syscall) but means fibers cannot run in parallel on different CPUs -- they share one kernel thread.

Use cases are niche: porting legacy cooperative-multitasking code, implementing coroutine libraries, or database engines that want fine-grained control over scheduling. Most modern Windows code uses the thread pool or async/await instead.

In Linux, the equivalent concept would be user-space threading libraries (like `makecontext`/`swapcontext`) or language-level coroutines (Go goroutines, Rust async tasks).

## UMS: User-Mode Scheduling (Deprecated)

**User-Mode Scheduling (UMS)** was introduced in Windows 7 to allow applications to schedule their own threads without kernel transitions. It was primarily used by SQL Server to manage thousands of concurrent queries efficiently. UMS has been deprecated in favor of other mechanisms, but it is historically interesting as an example of the OS providing scheduling flexibility to applications.

## CreateProcess Internals

Creating a process in Windows is significantly more complex than Linux's `fork() + exec()`. There is no `fork()` equivalent -- Windows always creates a fresh process. Here is the simplified flow:

```
CreateProcess("notepad.exe")
    |
    v
1. Open the executable image file
    |
    v
2. Create a Section object (memory-mapped image)
    |
    v
3. Create the EPROCESS and initial address space
    |
    v
4. Map ntdll.dll into the new process
    |
    v
5. Create the initial ETHREAD (primary thread)
    |
    v
6. Notify csrss.exe (Client/Server Runtime) of new process
    |
    v
7. Start the initial thread
    |
    v
8. In the new process: ntdll!LdrInitializeThunk
   - Process PEB initialization
   - Load required DLLs (kernel32, etc.)
   - Call DLL entry points (DllMain with DLL_PROCESS_ATTACH)
   - Jump to application entry point (mainCRTStartup -> main)
```

This is more steps than Linux's fork+exec because Windows does not copy an existing address space. It builds everything from scratch. The upside: no copy-on-write complexity. The downside: process creation is slower than Linux's fork (though modern Linux clone+exec is not that different in total cost).

## Comparison: Linux vs Windows Processes and Threads

| Aspect | Linux | Windows |
|---|---|---|
| Process structure | task_struct | EPROCESS |
| Thread structure | task_struct (shared mm) | ETHREAD / KTHREAD |
| Process creation | fork() + exec() or clone() | CreateProcess() (no fork) |
| Thread creation | clone(CLONE_VM \| ...) or pthread_create | CreateThread() |
| Process ID | PID (integer, global) | Process ID + Handle (handle is per-process) |
| Thread ID | TID (integer, global) | Thread ID + Handle |
| User-mode process info | /proc/[pid]/, auxv | PEB |
| User-mode thread info | TLS via pthread, VDSO | TEB |
| Resource groups | cgroups | Job objects |
| Cooperative threads | makecontext/swapcontext, coroutines | Fibers |
| Priority levels | Nice (-20 to 19) + RT (1-99) | 0-31 (dynamic 1-15, real-time 16-31) |
| Priority boosting | CFS vruntime adjustments | Explicit boosts for I/O, UI, starvation |

## Real-World Connection

When a .NET developer calls `Task.Run()`, the Task Parallel Library uses the Windows thread pool, which creates threads with `CreateThread()`, which creates ETHREAD structures in the kernel. Each thread gets a TEB, a kernel stack, and a priority level. The CLR's thread pool manager adjusts the number of threads based on work queue depth, and the Windows scheduler uses KTHREAD priorities to decide which threads run. Understanding this chain -- from `Task.Run()` down to KTHREAD scheduling -- is what separates a developer who can diagnose thread pool starvation from one who cannot.

In game development, threads servicing the render loop are often set to High priority class to ensure smooth frame rates. Audio threads may use real-time priorities. The Job object might be used to constrain background asset-loading threads so they do not starve the render thread of memory.

## Interview Angle

**Q: How do Windows processes and threads differ from Linux?**

A: The fundamental difference is structural. Linux uses `task_struct` for both processes and threads -- a thread is just a process that shares its parent's address space via `clone(CLONE_VM)`. Windows uses separate structures: EPROCESS for the process container and ETHREAD/KTHREAD for threads. Process creation also differs: Linux uses fork+exec (copy-on-write then replace), while Windows uses CreateProcess (build from scratch, no fork). For resource limiting, Windows uses Job objects while Linux uses cgroups. Both are hierarchical and control CPU, memory, and process counts, but cgroups are more granular for I/O and network limits.

**Q: What is the PEB and why is it important?**

A: The Process Environment Block (PEB) is a user-mode structure containing process state that can be read without a syscall: loaded DLL list, process heap pointer, environment variables, and image base address. It exists for performance -- frequently-accessed data should not require a kernel transition. Debuggers read the PEB to enumerate loaded modules. Malware modifies the PEB's DLL list to hide injected DLLs from detection tools. Security tools monitor PEB integrity for this reason.

---

Next: [Memory Management Internals](03-windows-memory-management-internals.md)
