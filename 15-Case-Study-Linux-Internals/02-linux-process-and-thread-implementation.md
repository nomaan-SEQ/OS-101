# Linux Process and Thread Implementation

## The Core Idea

In theory, processes and threads are distinct concepts — processes have separate address spaces, threads share one. In Linux, the implementation unifies both: **processes and threads are both represented by the same data structure (`task_struct`) and created by the same system call (`clone()`)**. The only difference is which resources they share.

This is one of the most elegant design decisions in Linux. There is no separate "thread struct." There is no special thread scheduler. Every schedulable entity is a `task_struct`, and the flags passed to `clone()` determine what gets shared and what gets copied.

---

## task_struct: The Central Data Structure

`task_struct` is THE most important data structure in the Linux kernel. Defined in `include/linux/sched.h`, it contains over 600 fields and weighs in at roughly 6 KB per instance. Every process, every thread, every kernel worker has one.

**`task_struct` is like a person's complete government file — ID number, home address, job history, bank accounts, medical records, criminal record, everything the government needs to know about you. Every process and thread has one of these files, and the kernel consults it constantly.**

Here are the key groups of fields:

```
task_struct (simplified)
├── Identity
│   ├── pid              Kernel-internal task ID (unique per task)
│   ├── tgid             Thread Group ID (what userspace calls "PID")
│   ├── comm[16]         Executable name (truncated to 16 chars)
│   └── *cred            Credentials: uid, gid, capabilities
│
├── Scheduling
│   ├── state            TASK_RUNNING, TASK_INTERRUPTIBLE, etc.
│   ├── prio / static_prio / normal_prio   Priority values
│   ├── policy           SCHED_NORMAL, SCHED_FIFO, SCHED_RR, SCHED_DEADLINE
│   ├── se               Sched entity for CFS (vruntime, load weight)
│   └── *sched_class     Pointer to scheduler class (fair, rt, deadline)
│
├── Memory
│   ├── *mm              Pointer to memory descriptor (page tables, VMAs)
│   └── *active_mm       Used by kernel threads (which have no mm of their own)
│
├── Files
│   ├── *files           Open file descriptor table
│   ├── *fs              Filesystem context (root dir, current dir, umask)
│   └── *nsproxy         Namespace references (pid_ns, net_ns, mnt_ns, ...)
│
├── Signals
│   ├── *sighand         Signal handler table
│   ├── pending          Pending signal queue
│   └── blocked          Signal mask
│
├── Relationships
│   ├── *parent          Pointer to parent task
│   ├── children         List of child tasks
│   ├── sibling          List linkage among siblings
│   └── *group_leader    Thread group leader
│
└── Accounting
    ├── utime / stime    User and system CPU time consumed
    ├── start_time       When this task was created
    └── *io_context      I/O scheduling context
```

The `mm` pointer is the key differentiator between processes and threads. If two `task_struct`s point to the **same** `mm_struct`, they share an address space — they are threads. If they each have their own `mm_struct`, they are separate processes.

---

## Threads ARE Processes: The clone() Unification

In Linux, `fork()`, `vfork()`, and `pthread_create()` all end up calling the same internal path. The difference is entirely in the flags:

| Operation | What Happens | clone() Flags |
|-----------|-------------|---------------|
| `fork()` | New process, copy everything | No sharing flags (SIGCHLD only) |
| `vfork()` | New process, share address space temporarily | CLONE_VM \| CLONE_VFORK |
| `pthread_create()` | New thread, share (almost) everything | CLONE_VM \| CLONE_FS \| CLONE_FILES \| CLONE_SIGHAND \| CLONE_THREAD |

Key `clone()` flags and what they share:

| Flag | What Gets Shared |
|------|-----------------|
| `CLONE_VM` | Address space (mm_struct) |
| `CLONE_FS` | Filesystem info (root, cwd, umask) |
| `CLONE_FILES` | File descriptor table |
| `CLONE_SIGHAND` | Signal handler table |
| `CLONE_THREAD` | Thread group (same TGID, shared signals) |
| `CLONE_NEWPID` | New PID namespace (for containers) |
| `CLONE_NEWNET` | New network namespace |
| `CLONE_NEWNS` | New mount namespace |

This is a spectrum, not a binary. You could theoretically create a `clone()` that shares the file descriptor table but not the address space — something that is neither a traditional process nor a traditional thread. Linux does not force you into rigid categories.

---

## Thread Groups: TGID vs PID

This is a common source of confusion, so let us be precise:

```
Process (Thread Group)
TGID = 1000
├── Thread 1 (group leader): PID = 1000, TGID = 1000
├── Thread 2:                PID = 1001, TGID = 1000
└── Thread 3:                PID = 1002, TGID = 1000
```

- **PID** (kernel perspective): Every `task_struct` has a unique PID. This is the kernel's per-task identifier.
- **TGID** (Thread Group ID): The PID of the thread group leader. All threads in a group share the same TGID.
- **What userspace calls "PID"**: This is actually the TGID. When you call `getpid()` in your program, you get the TGID. When `ps` shows a PID, it is showing the TGID.

This means `kill(1000, SIGTERM)` sends the signal to the entire thread group, not just one thread. The `gettid()` syscall returns the actual kernel PID (the per-thread ID), which is what `htop` shows in its TID column.

---

## Process Creation Internals

When you call `fork()`, here is what the kernel actually does:

```
User calls fork()
    │
    ▼
sys_fork()  [arch-specific syscall entry]
    │
    ▼
kernel_clone()  [the central implementation]
    │
    ▼
copy_process()
    ├── dup_task_struct()       Allocate new task_struct + kernel stack
    ├── copy_creds()            Copy credentials (uid, gid, capabilities)
    ├── copy_semundo()          Copy semaphore undo state
    ├── copy_files()            Copy or share file descriptor table
    ├── copy_fs()               Copy or share filesystem context
    ├── copy_sighand()          Copy or share signal handlers
    ├── copy_signal()           Copy signal state
    ├── copy_mm()               Copy or share address space  ← KEY STEP
    ├── copy_namespaces()       Copy or create new namespaces
    ├── copy_thread()           Copy CPU registers, set child's return value to 0
    └── alloc_pid()             Allocate new PID
    │
    ▼
wake_up_new_task()             Put the new task on the run queue
```

Each `copy_*` function checks the `clone()` flags. If `CLONE_VM` is set, `copy_mm()` does not actually copy — it increments the reference count on the parent's `mm_struct` and points the child to the same one. If `CLONE_VM` is not set, it performs a full copy using **Copy-on-Write**.

---

## Copy-on-Write Implementation

When `fork()` copies an address space, it does not actually duplicate the physical memory. That would be enormously wasteful (imagine forking a 2 GB process just to exec a tiny program). Instead, Linux uses Copy-on-Write (CoW):

1. **Mark all writable pages in both parent and child as read-only** in their page table entries (PTEs).
2. **Both processes share the same physical pages.** Memory usage barely increases.
3. **When either process writes to a shared page**, the CPU triggers a page fault (the page is marked read-only).
4. **The page fault handler** checks whether this is a CoW page. If so, it allocates a new physical page, copies the content, and updates the faulting process's PTE to point to the new page with write permission.
5. **Only the modified pages get copied.** Pages that are never written remain shared forever.

```
After fork() — before any writes:

Parent PTE ──┐
              ├──► Physical Page A  (marked read-only in both PTEs)
Child PTE  ──┘

After child writes to the page:

Parent PTE ──────► Physical Page A  (now writable for parent)

Child PTE  ──────► Physical Page B  (new copy, writable for child)
                   (contains copy of A's data + child's modification)
```

This is why `fork()` followed by `exec()` is cheap — the child barely copies any pages before replacing its entire address space with the new program. Only kernel structures (task_struct, page table skeleton) need to be allocated.

---

## NPTL: Native POSIX Threads Library

Linux uses a **1:1 threading model** through NPTL (Native POSIX Threads Library), which replaced the older LinuxThreads library around 2003.

| Aspect | NPTL (Current) | LinuxThreads (Legacy) |
|--------|---------------|----------------------|
| Model | 1:1 (one pthread = one kernel task) | 1:1 but with a manager thread |
| `getpid()` | Returns same PID for all threads (correct POSIX behavior) | Each thread had different PID (broken) |
| Signal delivery | Proper per-process signal semantics | Signals sent to individual threads (broken) |
| Scalability | Excellent (tested with 100K+ threads) | Poor (manager thread bottleneck) |
| Synchronization | Uses futex (fast user-space mutex) | Used signals for synchronization (slow) |

With NPTL, creating a thread calls `clone()` with sharing flags, producing a new `task_struct` that shares the parent's address space. The kernel scheduler treats it identically to any other task — there is no user-level scheduling involved.

---

## Kernel Threads

Not all `task_struct`s correspond to user-space processes. The kernel creates its own threads for background work:

| Kernel Thread | Purpose |
|--------------|---------|
| `kthreadd` (PID 2) | Parent of all kernel threads |
| `kworker/*` | Worker threads for the workqueue subsystem (deferred work) |
| `ksoftirqd/*` | Per-CPU threads that process software interrupts when load is high |
| `kswapd*` | Page reclaim daemon — frees memory by writing dirty pages to swap |
| `khugepaged` | Scans for opportunities to collapse small pages into huge pages |
| `kcompactd` | Memory compaction daemon (defragments physical memory) |
| `jbd2/*` | Journal commit threads for ext4 filesystems |
| `rcu_gp` / `rcu_par_gp` | RCU grace period threads (Read-Copy-Update synchronization) |

Kernel threads have no user-space address space — their `mm` pointer is NULL. They run entirely in kernel context. You can see them in `ps` output — they appear in square brackets: `[kworker/0:1]`, `[ksoftirqd/0]`.

---

## Special Processes: PID 0 and PID 1

**PID 0 — The Idle Task (swapper):** This is not a real process. Each CPU has an idle task that runs when there is nothing else to schedule. It puts the CPU into a low-power state (halt instruction) and waits for an interrupt. You will not see PID 0 in `ps` because it is part of the scheduler itself.

**PID 1 — init (systemd):** The first user-space process, created by the kernel at the end of boot. It has a special responsibility: **orphan reaping**. When a process's parent dies, the orphaned children are reparented to PID 1, which must call `wait()` to clean up their zombie entries. If PID 1 dies, the kernel panics — the system cannot function without it.

---

## Inspecting Processes via /proc

Every process has a directory under `/proc/<PID>/` with rich information:

```bash
# Process status (state, memory, UIDs, threads)
cat /proc/1234/status

# Memory map (all virtual memory areas)
cat /proc/1234/maps
# Output: address range, permissions, offset, device, inode, path
# 00400000-0040c000 r-xp 00000000 08:01 131077  /usr/bin/myapp

# Open file descriptors
ls -la /proc/1234/fd/
# 0 -> /dev/pts/0 (stdin)
# 1 -> /dev/pts/0 (stdout)
# 3 -> /var/log/app.log
# 4 -> socket:[12345]

# Command line
cat /proc/1234/cmdline | tr '\0' ' '

# Scheduling stats
cat /proc/1234/sched

# Namespace membership
ls -la /proc/1234/ns/
```

---

## Process Credentials

Security information is stored in `task_struct->cred`, a `struct cred`:

```
struct cred {
    uid_t uid;          Real user ID
    uid_t euid;         Effective user ID (for privilege checks)
    uid_t suid;         Saved user ID (for setuid programs)
    uid_t fsuid;        Filesystem user ID (for file access checks)
    gid_t gid, egid, sgid, fsgid;   Same for group IDs
    kernel_cap_t cap_inheritable;    Capabilities inherited across exec
    kernel_cap_t cap_permitted;      Maximum capabilities available
    kernel_cap_t cap_effective;      Currently active capabilities
    ...
};
```

The `cred` structure is immutable once assigned — to change credentials, the kernel creates a new `cred` structure and atomically swaps the pointer. This avoids race conditions in credential checks.

---

## Real-World Connection

**Container init processes:** When you run a Docker container, the entry process becomes PID 1 inside the container's PID namespace. But most application processes are not designed to be PID 1 — they do not reap zombies. This is why tools like `tini` and `dumb-init` exist: they act as a minimal PID 1, forwarding signals and reaping zombie children.

**Thread explosion in Java:** A Java application using thread-per-connection can create thousands of threads. Each one is a `task_struct` in the kernel, consuming ~6 KB of kernel memory plus its stack. With 10,000 threads, that is 60+ MB of kernel memory for task_structs alone, plus scheduling overhead. This is why modern Java servers use virtual threads or NIO — to reduce the number of kernel tasks.

**strace follows threads:** When you run `strace -f`, the `-f` flag follows child tasks created by `clone()`. Without it, you only see the original thread. This is essential for debugging multithreaded applications — each thread makes its own syscalls.

---

## Interview Angle

**Q: In Linux, what is the difference between a process and a thread at the kernel level?**

A: There is no structural difference. Both are represented by `task_struct` and created by `clone()`. The difference is in what they share: a process (created by `fork()`) gets its own copy of the address space, file descriptor table, and signal handlers. A thread (created by `pthread_create()`, which calls `clone()` with `CLONE_VM | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD`) shares all of these with the parent. The kernel scheduler treats them identically — both are schedulable entities.

**Q: What is Copy-on-Write and why is it essential for fork()?**

A: When `fork()` creates a child, it does not duplicate physical memory. Instead, both parent and child share the same physical pages, with page table entries marked read-only. When either process writes to a page, a page fault occurs, and the kernel copies just that page. This makes `fork()` fast (O(page table size) rather than O(memory size)) and makes the common `fork() + exec()` pattern cheap — the child replaces its address space before most pages are ever written.

**Q: What is the TGID and why does it exist?**

A: TGID (Thread Group ID) is the PID of the thread group leader. All threads in a multithreaded process share the same TGID. This is what userspace `getpid()` returns, maintaining the POSIX illusion that all threads share one PID. The kernel internally assigns each thread its own unique PID (accessible via `gettid()`), but hides this behind the TGID abstraction. Signals sent to a TGID are delivered to the thread group, not an individual thread.

---

**Next**: [03-linux-memory-management-internals.md](03-linux-memory-management-internals.md)
