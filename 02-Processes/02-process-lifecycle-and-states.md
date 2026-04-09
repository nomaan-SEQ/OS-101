# Process Lifecycle and States

## The Core Idea

A process doesn't just run from start to finish. It goes through a series of **states** as the operating system manages CPU time among many competing processes. Understanding these states and the transitions between them is how you interpret what `ps`, `top`, and `htop` are actually telling you.

---

## The 5-State Model

Every OS textbook presents some version of this model. The names vary, but the concept is universal:

```
                    ┌──────────┐
     Process        │          │
     Created ──────>│   New    │
                    │          │
                    └────┬─────┘
                         │ admitted to
                         │ ready queue
                         v
                    ┌──────────┐   scheduled    ┌──────────┐
                    │          │ ─────────────> │          │
                    │  Ready   │                │ Running  │
                    │          │ <───────────── │          │
                    └──────────┘   preempted    └────┬─────┘
                         ^                          │  │
                         │                          │  │
                    I/O  │                          │  │ calls exit()
                 complete│    ┌──────────┐          │  │ or killed
                         │    │          │ <────────┘  │
                         └────│ Waiting  │  I/O req    │
                              │(Blocked) │  or event   │
                              │          │  wait       v
                              └──────────┘      ┌──────────┐
                                                │          │
                                                │Terminated│
                                                │          │
                                                └──────────┘
```

### State Descriptions

| State | What's Happening | CPU? | Memory? |
|-------|-----------------|------|---------|
| **New** | Process is being created (PCB allocated, memory set up) | No | Being allocated |
| **Ready** | Process is loaded and waiting for CPU time | No — waiting in queue | Yes |
| **Running** | Process instructions are being executed on a CPU core | **Yes** | Yes |
| **Waiting** (Blocked) | Process is waiting for an event (I/O, signal, lock) | No | Yes |
| **Terminated** | Process has finished; kernel is cleaning up | No | Being freed |

On a single-core system, **exactly one** process is Running at any time. On an N-core system, up to N processes can be Running simultaneously.

> **Analogy:** Think of process states like students in a university. **New** = enrolled but not yet attending. **Ready** = in the classroom waiting to be called on. **Running** = presenting at the board. **Waiting** = in the library doing research. **Terminated** = graduated.

---

## State Transitions

Each arrow in the diagram has a specific trigger:

| Transition | From → To | Trigger |
|-----------|-----------|---------|
| **Created** | New → Ready | OS finishes setting up the process (PCB, memory) |
| **Scheduled** (dispatched) | Ready → Running | Scheduler picks this process and loads it onto CPU |
| **Preempted** | Running → Ready | Timer interrupt fires (time slice expired), or higher-priority process arrives |
| **I/O or Event Wait** | Running → Waiting | Process makes a blocking syscall (read from disk, wait for network, acquire a lock) |
| **I/O or Event Complete** | Waiting → Ready | The I/O operation finishes, signal arrives, lock becomes available |
| **Exit** | Running → Terminated | Process calls `exit()`, returns from `main()`, or is killed by a signal |

The critical insight: **a process never goes directly from Waiting to Running**. When its I/O completes, it goes back to the Ready queue and waits for the scheduler to pick it up again. The scheduler decides who runs next — not the I/O subsystem.

---

## The Ready Queue and Wait Queues

The kernel maintains queues to organize processes by state:

```
                    Ready Queue (one per priority level, or one sorted queue)
                    ┌──────┬──────┬──────┬──────┐
  Scheduler picks   │ P3   │ P7   │ P12  │ P19  │ ──> next to run
  from here ───────>└──────┴──────┴──────┴──────┘

                    Wait Queues (one per event/resource)
                    
  Disk I/O queue:   ┌──────┬──────┐
                    │ P5   │ P8   │
                    └──────┴──────┘
                    
  Network queue:    ┌──────┬──────┐
                    │ P2   │ P14  │
                    └──────┴──────┘
                    
  Pipe read queue:  ┌──────┐
                    │ P11  │
                    └──────┘
```

When an I/O operation completes, the interrupt handler finds the corresponding wait queue, moves the process to the Ready queue, and eventually the scheduler notices it.

---

## Additional Process States

Real operating systems have more states than the basic 5. Three important ones:

### Zombie (Terminated but not reaped)

A process has exited, but its parent hasn't called `wait()` to read the exit status. The kernel keeps the PCB entry (so the exit code is available) but the process is not running and not consuming resources beyond that entry.

We cover zombies in depth in [06-orphan-and-zombie-processes.md](06-orphan-and-zombie-processes.md).

### Orphan (Parent has exited)

A process whose parent has terminated. Orphans are adopted by `init`/`systemd` (PID 1), which becomes their new parent and will eventually reap them.

### Stopped (Suspended)

A process that has been paused, usually by a signal:
- `SIGSTOP` or `SIGTSTP` (Ctrl+Z in a terminal) suspends the process
- `SIGCONT` resumes it

Stopped processes remain in memory but are not in the Ready queue — they won't be scheduled until resumed.

---

## Linux Process States (What You See in `ps`)

Linux maps these concepts to specific state codes. When you run `ps aux`, the `STAT` column shows:

| Code | State | Description |
|------|-------|-------------|
| **R** | Running/Runnable | On CPU or in the ready queue |
| **S** | Sleeping (Interruptible) | Waiting for an event, can be woken by signals |
| **D** | Sleeping (Uninterruptible) | Waiting for I/O, cannot be interrupted (e.g., disk read) |
| **Z** | Zombie | Terminated, waiting for parent to call wait() |
| **T** | Stopped | Suspended by signal (SIGSTOP/SIGTSTP) |
| **t** | Tracing stop | Stopped by debugger (ptrace) |
| **X** | Dead | Should never be seen (process fully cleaned up) |

Additional modifiers in the STAT column:

| Modifier | Meaning |
|----------|---------|
| **s** | Session leader |
| **l** | Multi-threaded |
| **+** | Foreground process group |
| **<** | High priority (nice < 0) |
| **N** | Low priority (nice > 0) |

So `Ss` means "sleeping, session leader" and `R+` means "running, foreground process."

### The D State is Special

The **uninterruptible sleep (D)** state is important in practice. A process in D state:
- Is waiting for I/O (typically disk)
- **Cannot be killed**, not even by `kill -9` (SIGKILL)
- If you see many D-state processes, it usually means your disk or NFS mount is having problems

This is why you sometimes can't kill a process — it's stuck in D state waiting for I/O that will never complete (common with hung NFS mounts).

---

## Seeing States in Practice

```bash
# Show all processes with their states
ps aux

# Show only zombies
ps aux | awk '$8 ~ /Z/'

# Show process states in real-time
top     # press 'H' to show threads

# Show the full process tree with states
ps auxf

# Count processes by state
ps -eo stat | sort | uniq -c | sort -rn
```

Example output of `ps aux`:
```
USER       PID %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
root         1  0.0  0.1 168936 13300 ?    Ss   Apr01   0:12 /sbin/init
root         2  0.0  0.0      0     0 ?    S    Apr01   0:00 [kthreadd]
www-data  1205  2.1  1.4 327000 57600 ?    S    10:05   1:30 nginx: worker
postgres  1400  0.3  2.1 420000 85000 ?    Ss   Apr01   5:42 postgres
alice     1550  0.0  0.0  12345  2340 pts/0 R+   10:32   0:00 ps aux
root      1100  0.0  0.0      0     0 ?    D    10:30   0:00 [nfs_write]
alice     1560  0.0  0.0      0     0 ?    Z    10:33   0:00 [defunct]
```

From this, you can read: init is sleeping as a session leader, the nginx worker is in interruptible sleep, postgres is sleeping as a session leader, `ps` itself is running in the foreground, something is stuck in uninterruptible sleep on NFS, and there's a zombie process.

---

## Real-World Connection

**Load average and process states**: **Load average** represents the average number of processes in the Running or uninterruptible sleep (D) state over 1, 5, and 15 minute intervals. A load average of 1.0 on a single-core system means the CPU is fully utilized; on a 4-core system, it means 25% utilized. So when you see a load average of 5.0 on a 4-core machine, it means on average 5 processes are either Running (R) or in D state. This is why high load average with low CPU usage usually means I/O bottleneck — processes are in D state waiting for disk.

**Container health checks**: Kubernetes liveness probes check if a container's main process is still running and responsive. Understanding process states helps you diagnose why a pod gets restarted — the process might be stuck in D state (I/O hang), zombie-accumulated, or legitimately crashed.

**Systemd and process supervision**: `systemd` monitors process states. If a service's main process enters the Terminated state unexpectedly, systemd can restart it based on the `Restart=` directive. It uses `waitpid()` to detect child termination and `SIGCHLD` signals.

---

## Interview Angle

**Q: Walk through the complete lifecycle of a process.**

A: A process starts in the New state when the OS allocates a PCB and sets up memory. Once initialized, it moves to Ready and enters the ready queue. The scheduler eventually dispatches it to Running. While running, it might get preempted back to Ready (time slice expired) or block on I/O and move to Waiting. When I/O completes, it returns to Ready (not directly to Running). Eventually the process calls exit() or is killed, entering the Terminated state. The parent calls wait() to read the exit status, and the kernel frees the PCB.

**Q: What's the difference between the S and D states in Linux?**

A: Both are sleeping/waiting, but S (interruptible sleep) means the process can be woken by a signal — you can kill it. D (uninterruptible sleep) means the process is waiting for I/O and cannot be interrupted, not even by SIGKILL. D state is typically brief (a disk read), but if the underlying I/O never completes (hung NFS mount, failing disk), the process gets stuck and becomes unkillable.

**Q: Can a process go directly from Waiting to Running?**

A: No. When a waiting process's I/O completes, it moves to the Ready queue. The scheduler then decides when to dispatch it to Running. This separation is fundamental — the scheduler maintains control over CPU allocation, not the I/O subsystem.

---

**Next**: [03-process-control-block.md](03-process-control-block.md)
