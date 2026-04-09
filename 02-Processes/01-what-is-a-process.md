# What Is a Process?

## The Core Idea

A **process** is a program in execution. That single sentence is the most important definition in operating systems.

A **program** is a passive entity — a file sitting on disk containing instructions (compiled code) and maybe some initial data. It does nothing on its own.

A **process** is an active entity — an instance of that program running on the CPU, with its own memory, registers, open files, and identity. It's alive.

| Aspect | Program | Process |
|--------|---------|---------|
| Nature | Static (file on disk) | Dynamic (executing) |
| Lifetime | Exists until deleted | Exists from creation to termination |
| State | No state — just bytes | Has state: registers, memory, files |
| Multiplicity | One program file | Multiple processes can run it |
| Contains | Code + initial data | Code + data + heap + stack + OS state |

A single program can have many processes. When 50 users SSH into a server and each runs `bash`, there are 50 bash processes — all running the same `/bin/bash` program. Each process has its own memory, its own variables, its own open files.

---

## What a Process Consists Of

A process is more than just the code it's executing. The OS must track a significant amount of state for each process:

**User-space state (the process's own memory):**
- **Code (text segment)** — The compiled machine instructions. Read-only and shared if multiple processes run the same program.
- **Data segment** — Global and static variables that are initialized.
- **BSS segment** — Global and static variables that are uninitialized (zero-filled).
- **Heap** — Dynamically allocated memory (`malloc`, `new`). Grows upward.
- **Stack** — Function call frames, local variables, return addresses. Grows downward.

**Kernel-managed state (tracked in the PCB — covered in detail later):**
- CPU register values (including the program counter)
- Open file descriptors
- Current working directory
- Signal handlers and pending signals
- Memory mappings (page tables)
- Process identity (PID, parent PID, user ID, group ID)
- Scheduling priority and CPU time accounting

---

## Process Memory Layout

Every process sees a flat virtual address space. Here's the classic layout on a typical system:

```
High Address ┌─────────────────────────────┐
             │         Kernel Space        │  <- Process can't touch this
             │    (mapped but protected)   │
             ├─────────────────────────────┤  0xC0000000 (32-bit) or
             │                             │  0x7FFF... (64-bit)
             │          Stack              │  <- Grows DOWNWARD
             │    (local vars, frames)     │
             │            │                │
             │            v                │
             │                             │
             │         ... gap ...         │  <- Unused virtual space
             │                             │
             │            ^                │
             │            │                │
             │          Heap               │  <- Grows UPWARD
             │     (malloc, new)           │
             ├─────────────────────────────┤
             │     BSS (uninitialized)     │  <- Zero-filled globals
             ├─────────────────────────────┤
             │     Data (initialized)      │  <- Global/static vars
             ├─────────────────────────────┤
             │     Text (code)             │  <- Read-only, shareable
Low Address  └─────────────────────────────┘  0x00000000 (approx)
```

The key insight: the heap and stack grow **toward each other**. If they ever collide, you get a stack overflow or out-of-memory error (in practice, the OS and hardware catch this via guard pages before actual collision).

On 64-bit systems, the address space is so vast (128 TB on x86-64 Linux) that collision is essentially impossible. The gap between heap and stack is enormous.

---

## Process vs Program: The Full Picture

Think of it like cooking:

- The **program** is the recipe (a static document with instructions).
- The **process** is the act of cooking — the cook (CPU), the ingredients on the counter (data), the current step you're on (program counter), and the oven temperature (state).

Multiple cooks can follow the same recipe simultaneously, each at different steps, each with their own bowls and ingredients. That's multiple processes running the same program.

```
Program (on disk)          Processes (in memory)
┌──────────────┐
│  /bin/bash   │ ───────>  Process 1001 (user alice, PID 1001)
│              │ ───────>  Process 1042 (user bob,   PID 1042)
│  (one file)  │ ───────>  Process 1087 (user carol, PID 1087)
└──────────────┘
                           Each has its own:
                           - Stack, heap, data
                           - Open files
                           - Register values
                           - PID and identity
                           
                           They SHARE:
                           - Text (code) segment (read-only)
```

---

## Process Identity

Every process has a unique identity within the system:

| Identifier | What It Is | How to See It |
|------------|-----------|---------------|
| **PID** | Process ID — unique integer assigned at creation | `getpid()`, `ps -ef` |
| **PPID** | Parent Process ID — who created this process | `getppid()`, `ps -ef` |
| **UID** | User ID — which user owns this process | `getuid()`, `ps -eo pid,uid,comm` |
| **GID** | Group ID — which group this process belongs to | `getgid()` |
| **SID** | Session ID — for terminal/job control | `getsid()` |
| **PGID** | Process Group ID — for signal delivery to a group | `getpgid()` |

PIDs are assigned sequentially by the kernel and wrap around after reaching a maximum (typically 32768 by default on Linux, configurable up to ~4 million via `/proc/sys/kernel/pid_max`).

The **PPID** creates a tree structure — every process (except PID 1, `init`/`systemd`) has a parent:

```
systemd (PID 1)
├── sshd (PID 500)
│   ├── sshd (PID 1200) ── session for alice
│   │   └── bash (PID 1201)
│   │       ├── vim (PID 1350)
│   │       └── grep (PID 1351)
│   └── sshd (PID 1400) ── session for bob
│       └── bash (PID 1401)
├── nginx (PID 600)
│   ├── nginx-worker (PID 601)
│   └── nginx-worker (PID 602)
└── postgres (PID 700)
    ├── postgres-writer (PID 701)
    └── postgres-wal (PID 702)
```

You can see this tree with `pstree` or `ps auxf` on Linux.

---

## Every Process Has an Address Space

This is a critical concept that we'll explore deeply in the Virtual Memory section later: **every process thinks it has the entire memory to itself**.

Process A and Process B both have memory at address `0x400000`, but they're different physical memory locations. The hardware (**MMU** — Memory Management Unit, the CPU component that translates virtual addresses to physical addresses) and OS (**page tables** — data structures that define the virtual-to-physical address mapping for each process) collaborate to create this illusion. This is called **virtual memory**, and it provides:

- **Isolation** — Process A can't read or corrupt Process B's memory
- **Simplicity** — Each process sees a clean, linear address space starting from 0
- **Security** — A buggy or malicious process can't damage other processes

For now, just remember: each process has its own virtual address space, and the hardware enforces the boundaries.

---

## Real-World Connection

**Linux `ps` and `/proc`**: Each process has a directory under `/proc/<PID>/` containing its full state. Try `ls /proc/self/` — it shows the current process. Key files:
- `/proc/<PID>/maps` — memory layout (text, data, heap, stack, shared libs)
- `/proc/<PID>/status` — PID, PPID, state, memory usage
- `/proc/<PID>/fd/` — open file descriptors

**Docker containers** are processes. When you run `docker run nginx`, Docker calls `fork()` + `exec()` to create a process, then applies Linux namespaces (so the process sees a different PID space, filesystem, network) and cgroups (to limit CPU/memory). But underneath, it's still just a process on the host. You can see it with `ps` on the host machine.

**Chrome's multi-process architecture**: Chrome runs each tab as a separate process. If one tab crashes, the others survive. This is a direct application of process isolation — each tab process has its own address space and can't corrupt others.

---

## Interview Angle

**Q: What is the difference between a process and a program?**

A: A program is a passive file on disk containing code and data. A process is an active instance of that program in execution. A process includes the program's code, but also its current state: registers, program counter, stack, heap, open files, and kernel-tracked metadata like PID and scheduling priority. The same program can have multiple processes running simultaneously, each independent and isolated.

**Q: If I run `cat /etc/passwd` in two terminals, how many programs and processes are involved?**

A: One program (`/bin/cat`), two processes. Each process has its own PID, its own stack and heap, its own file descriptors (one has `/etc/passwd` open as fd 3, the other also has it open independently as fd 3). They share the same read-only text segment (the compiled instructions of `cat`), but everything else is separate.

**Q: What is the process memory layout and why does the stack grow downward?**

A: From low to high: text (code), data (initialized globals), BSS (uninitialized globals), heap (grows up), and stack (grows down from high addresses). The stack grows downward by convention, which allows the heap and stack to grow toward each other from opposite ends of the address space, maximizing the usable virtual memory without pre-committing a fixed size for either.

---

**Next**: [02-process-lifecycle-and-states.md](02-process-lifecycle-and-states.md)
