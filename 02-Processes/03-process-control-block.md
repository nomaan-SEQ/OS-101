# Process Control Block (PCB)

## The Core Idea

If a process is a running program, the **Process Control Block (PCB)** is the kernel's complete dossier on that process. It's the data structure that lets the OS manage, schedule, and context-switch processes. Without the PCB, the kernel would have no way to stop a process, run something else, and then resume the original process exactly where it left off.

Every process has exactly one PCB. The kernel creates it when the process is born and destroys it when the process is fully reaped.

---

## What's in a PCB

The PCB contains everything the kernel needs to know about a process:

```
┌─────────────────────────────────────────────────────────┐
│                 Process Control Block                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  IDENTIFICATION                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ PID: 1205          PPID: 500                    │    │
│  │ UID: 33 (www-data) GID: 33                      │    │
│  │ Session ID: 500    Process Group ID: 1205        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  CPU STATE (saved on context switch)                     │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Program Counter: 0x4005a0                       │    │
│  │ Stack Pointer:   0x7ffd2340                     │    │
│  │ General Registers: rax, rbx, rcx, rdx, ...     │    │
│  │ Flags Register:  0x202                          │    │
│  │ FPU/SSE/AVX state                               │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  PROCESS STATE                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │ State: RUNNING / READY / WAITING / etc.         │    │
│  │ Exit code: (if terminated)                      │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  SCHEDULING INFO                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Priority: 20 (nice 0)                           │    │
│  │ Scheduling policy: SCHED_OTHER (CFS)            │    │
│  │ CPU time used: 1.52 seconds                     │    │
│  │ Time slice remaining: 4ms                       │    │
│  │ vruntime: 234567890 (CFS virtual runtime)       │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  MEMORY MANAGEMENT                                       │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Page table base register (CR3 on x86)           │    │
│  │ Memory maps (vm_area_struct list):              │    │
│  │   Text:  0x400000 - 0x401000 (r-x)             │    │
│  │   Data:  0x601000 - 0x602000 (rw-)             │    │
│  │   Heap:  0x1000000 - 0x1004000 (rw-)           │    │
│  │   Stack: 0x7ffd2000 - 0x7ffd4000 (rw-)         │    │
│  │ RSS: 14 MB    VSZ: 120 MB                       │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  FILE DESCRIPTORS                                        │
│  ┌─────────────────────────────────────────────────┐    │
│  │ fd 0 → /dev/pts/1 (stdin)                      │    │
│  │ fd 1 → /dev/pts/1 (stdout)                     │    │
│  │ fd 2 → /dev/pts/1 (stderr)                     │    │
│  │ fd 3 → /var/log/app.log (O_WRONLY)             │    │
│  │ fd 4 → socket(TCP, 10.0.0.1:8080)             │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  SIGNAL INFO                                             │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Pending signals: (none)                         │    │
│  │ Blocked signals: {SIGPIPE}                      │    │
│  │ Signal handlers: SIGTERM→handler_func           │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ACCOUNTING                                              │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Start time: Apr 09 10:05:32                     │    │
│  │ User CPU time:   1.20s                          │    │
│  │ System CPU time: 0.32s                          │    │
│  │ Voluntary context switches: 450                 │    │
│  │ Involuntary context switches: 120               │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## PCB Fields Grouped by Purpose

| Category | Key Fields | Why It's Needed |
|----------|-----------|----------------|
| **Identity** | PID, PPID, UID, GID | Know which process this is and who owns it |
| **CPU state** | Program counter, registers, stack pointer, flags | Resume execution after a context switch |
| **Scheduling** | Priority, policy, time slice, vruntime | Decide when and how long this process runs |
| **Memory** | Page table pointer, memory maps, RSS/VSZ | Define what memory this process can access |
| **Files** | File descriptor table, cwd, umask | Track what files/sockets/pipes are open |
| **Signals** | Pending, blocked, handlers | Handle asynchronous events |
| **Accounting** | CPU time, start time, context switch counts | Resource tracking and billing |
| **State** | Current state (R/S/D/Z/T), exit code | Track where the process is in its lifecycle |

---

## Linux: task_struct

In the Linux kernel, the PCB is a C struct called `task_struct`. It's defined in `include/linux/sched.h` and it's massive — over 600 fields in modern kernels, spanning several kilobytes.

Some notable fields:

| task_struct field | Purpose |
|-------------------|---------|
| `pid` / `tgid` | Process ID / Thread Group ID |
| `state` | Current process state (TASK_RUNNING, etc.) |
| `stack` | Pointer to the kernel stack |
| `mm` | Pointer to the memory descriptor (`mm_struct`) |
| `files` | Pointer to open file table (`files_struct`) |
| `fs` | Filesystem info (cwd, root) |
| `signal` | Signal handling state |
| `parent` | Pointer to parent task_struct |
| `children` | List of child processes |
| `se` | Scheduling entity (for CFS) |
| `comm[TASK_COMM_LEN]` | Process name (16 chars max) |
| `cred` | Credentials (UID, GID, capabilities) |
| `nsproxy` | Namespace pointers (for containers) |

The `task_struct` is allocated from a dedicated slab cache for performance. When you `fork()`, the kernel allocates a new `task_struct` and copies (or shares) the relevant parts from the parent.

Importantly, in Linux, **threads and processes use the same `task_struct`**. A thread is a `task_struct` that shares `mm`, `files`, and `signal` with other `task_struct`s in the same thread group. This is why Linux calls them "tasks" rather than "processes" internally.

---

## The Process Table

The **process table** is the collection of all PCBs in the system. Conceptually:

```
Process Table
┌───────┬──────────┬─────────┬───────────┬─────┐
│  PID  │  State   │  PPID   │   Name    │ ... │
├───────┼──────────┼─────────┼───────────┼─────┤
│   1   │ Sleeping │   0     │  systemd  │     │
│ 500   │ Sleeping │   1     │  sshd     │     │
│ 1200  │ Sleeping │  500    │  sshd     │     │
│ 1201  │ Running  │  1200   │  bash     │     │
│ 1205  │ Sleeping │   1     │  nginx    │     │
│ 1350  │ Zombie   │  1201   │  gcc      │     │
└───────┴──────────┴─────────┴───────────┴─────┘
```

In Linux, you can inspect the process table through `/proc/`:
- `/proc/<PID>/status` — readable summary of the PCB
- `/proc/<PID>/stat` — machine-parseable PCB fields
- `/proc/<PID>/maps` — memory mappings
- `/proc/<PID>/fd/` — open file descriptors (symlinks to actual files/sockets)

The process table has a finite size bounded by `pid_max`. If the table fills up (every PID is taken), `fork()` fails with `EAGAIN`. This is one reason zombie processes are dangerous — they occupy PID slots.

---

## The File Descriptor Table

Each process has its own **file descriptor table**, which maps small integers (0, 1, 2, 3, ...) to open file descriptions in the kernel:

```
Process 1201 (bash)              Kernel File Table
┌─────┬──────────────┐          ┌───────────────────────┐
│ fd  │  Pointer     │          │ Open File Description  │
├─────┼──────────────┤     ┌───>│ /dev/pts/1             │
│  0  │  ──────────────────┤    │ offset=0, flags=O_RDWR │
│  1  │  ──────────────────┤    └───────────────────────┘
│  2  │  ──────────────────┘    ┌───────────────────────┐
│  3  │  ──────────────────────>│ /var/log/app.log       │
│  4  │  ──────────┐           │ offset=4096, O_WRONLY  │
└─────┴────────────┘           └───────────────────────┘
                     │          ┌───────────────────────┐
                     └─────────>│ socket(TCP)            │
                                │ 10.0.0.1:8080         │
                                └───────────────────────┘
```

Key points:
- **fd 0, 1, 2** are stdin, stdout, stderr by convention
- File descriptors are per-process — fd 3 in process A and fd 3 in process B are completely different
- When a process `fork()`s, the child gets a **copy** of the fd table (both point to the same kernel file descriptions, sharing the file offset)
- This is how shell I/O redirection works — between fork and exec, the shell closes/reopens fds to redirect the child's I/O

---

## Why the PCB Matters for Context Switching

When the OS switches from Process A to Process B, it must:

1. **Save** Process A's CPU registers, program counter, and stack pointer **into A's PCB**
2. **Update** A's state in its PCB (Running → Ready or Waiting)
3. **Load** Process B's CPU registers, program counter, and stack pointer **from B's PCB**
4. **Update** B's state (Ready → Running)
5. **Switch** the page table pointer to B's address space

The PCB is the mechanism that makes this possible. Without it, you'd lose A's execution state forever when you switched to B. The PCB is literally the save slot for a process.

---

## Real-World Connection

**`/proc` filesystem in Linux**: The `/proc` filesystem is the kernel exposing PCB data to user space. When monitoring tools like `top`, `htop`, `ps`, and Prometheus `node_exporter` report process metrics, they're reading from `/proc`. There's no magic — it's all PCB fields made readable.

**Container resource limits**: When you set `docker run --memory=512m --cpus=2`, Docker writes to cgroup files, and the kernel stores those limits alongside the process's scheduling info in the `task_struct`. The scheduler checks these limits on every scheduling decision.

**strace and ptrace**: When you `strace` a process, the kernel's `ptrace` mechanism uses the PCB to intercept and inspect system calls. The tracer can read the traced process's registers and memory maps directly from its `task_struct`.

---

## Interview Angle

**Q: What is a PCB and what does it contain?**

A: The Process Control Block is the kernel's data structure for tracking everything about a process. It contains the process's identity (PID, UID), CPU state (registers, program counter), scheduling info (priority, time slice), memory management info (page table pointer, memory maps), open file descriptors, signal state, and accounting data. In Linux, this is the `task_struct`.

**Q: Why is the PCB critical for context switching?**

A: A context switch requires saving the current process's entire CPU state and loading the next process's state. The PCB is where this state is saved and restored from. Without the PCB, the kernel couldn't resume a process after switching away from it — the register values, program counter, and stack pointer would be lost.

**Q: How can you inspect a process's PCB on a Linux system?**

A: Through the `/proc/<PID>/` filesystem. Key files: `status` (readable summary), `stat` (parseable fields), `maps` (memory layout), `fd/` (open files), `sched` (scheduling stats). Tools like `ps`, `top`, and `htop` read from these same files.

---

**Next**: [04-context-switching.md](04-context-switching.md)
