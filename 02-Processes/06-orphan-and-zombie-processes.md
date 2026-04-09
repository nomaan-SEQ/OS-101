# Orphan and Zombie Processes

## The Core Idea

When a process exits, its story isn't quite over. The kernel keeps a small record of the dead process (its exit status) until the parent collects it. If the parent doesn't collect it — or if the parent is already gone — you get two special situations: **zombies** and **orphans**.

These aren't edge cases. They come up constantly in real production systems, and misunderstanding them leads to resource leaks, PID exhaustion, and mysterious failures.

---

## Zombie Processes

A **zombie process** is a process that has finished executing but whose parent hasn't called `wait()` to read its exit status.

### Why Zombies Exist

When a child process exits (calls `exit()`, returns from main, or is killed), the kernel:
1. Releases the process's memory, open files, and most resources
2. **Keeps the PCB entry** with the process's exit status, PID, and resource usage stats
3. Sends `SIGCHLD` to the parent to notify it

The kernel keeps this minimal record because the parent might need the child's exit status. Until the parent calls `wait()` or `waitpid()`, the dead process remains in the process table as a zombie.

```
Timeline:

Parent (PID 1000)                    Child (PID 1001)
     │                                     │
     │  fork() ──────────────────>         │  (created)
     │                                     │
     │  (doing other work,                 │  (running)
     │   not calling wait)                 │
     │                                     │  exit(0)
     │                                     │
     │                                     ╳  Process exits
     │                                     │
     │  SIGCHLD received                   ░  ZOMBIE STATE
     │  (but ignored!)                     ░  - No CPU used
     │                                     ░  - No memory used
     │  (still not calling wait...)        ░  - Just a PCB entry
     │                                     ░  - Shows as Z in ps
     │                                     ░
     │                                     ░  (waiting forever...)
```

### What a Zombie Consumes

| Resource | Consumed? |
|----------|-----------|
| CPU | No |
| RAM (user space) | No — freed on exit |
| Kernel memory (PCB) | Yes — a small `task_struct` entry |
| PID slot | **Yes** — this is the real problem |
| File descriptors | No — closed on exit |
| Network sockets | No — closed on exit |

A single zombie costs almost nothing. But if a parent keeps spawning children without calling `wait()`, zombies accumulate. Since each zombie holds a PID slot, you can eventually hit `pid_max` and `fork()` starts failing with `EAGAIN` — no new processes can be created.

### Spotting Zombies

```bash
# Zombies show up with state Z
$ ps aux | awk '$8 == "Z"'
alice     1560  0.0  0.0      0     0 ?        Z    10:33   0:00 [myapp] <defunct>

# Or use ps with the state filter
$ ps -eo pid,ppid,stat,comm | grep Z
 1560  1000 Z    myapp

# Count zombies
$ ps -eo stat | grep -c Z
3
```

The `<defunct>` label is a clear indicator. The process name in brackets means the process image is gone — only the PCB remains.

---

## Orphan Processes

An **orphan process** is a process whose parent has exited before the child. The child is still running, but its parent is gone.

### What Happens to Orphans

When a parent exits, the kernel doesn't kill its children. Instead, orphaned children are **re-parented** to `init` (PID 1, or `systemd` on modern Linux):

```
Before parent exits:                After parent exits:

systemd (PID 1)                     systemd (PID 1)
└── Parent (PID 1000)               └── Child (PID 1001)   <- adopted!
    └── Child (PID 1001)            
                                    Parent (PID 1000) is gone.
                                    Child's PPID is now 1.
```

### Why Orphans Are Usually Fine

`init`/`systemd` is designed to be a responsible parent. It periodically calls `wait()` to reap any adopted children that have exited. So orphan processes:

- Continue running normally
- Get a new parent (PID 1)
- Will be properly cleaned up when they exit (no zombies)

Orphans are **not inherently a problem**. Background daemons are intentionally created as orphans — the process forks, the parent exits, and the child continues as a daemon adopted by init.

### Orphan vs Zombie Comparison

| Aspect | Zombie | Orphan |
|--------|--------|--------|
| State | Dead (exited) | Alive (still running) |
| Parent | Parent is alive but not calling wait() | Parent is dead |
| New parent | Still the original parent | Adopted by init (PID 1) |
| Problem? | Yes — accumulates PID entries | Usually no — init reaps them |
| Shows in ps as | Z (zombie/defunct) | Normal state (R, S, etc.) |
| Consumes resources | Only a PID slot | Normal process resources |
| Fix | Make parent call wait() or kill parent | No fix needed |

---

## Fixing Zombie Processes

### Option 1: Fix the Parent (Best Solution)

The parent should handle child termination. Three common patterns:

**Pattern A: Explicitly wait**
```
// After fork, parent calls wait/waitpid
pid = fork()
if (pid > 0) {
    waitpid(pid, &status, 0)   // blocks until child exits
}
```

**Pattern B: Handle SIGCHLD**
```
// Install a signal handler that reaps children
void sigchld_handler(int sig) {
    // Reap ALL finished children (non-blocking)
    while (waitpid(-1, NULL, WNOHANG) > 0) {
        // child reaped
    }
}

signal(SIGCHLD, sigchld_handler)
```

**Pattern C: Ignore SIGCHLD**
```
// Tell the kernel to auto-reap children
signal(SIGCHLD, SIG_IGN)
```

Setting `SIGCHLD` to `SIG_IGN` tells the kernel that the parent doesn't care about child exit status. The kernel won't create zombies at all — children are reaped immediately on exit.

### Option 2: Kill the Parent

If you can't fix the parent's code, killing the parent makes all its zombie children become orphans of `init`, which will reap them:

```bash
# Find the parent of the zombies
$ ps -eo pid,ppid,stat,comm | grep Z
 1560  1000 Z    myapp
 1561  1000 Z    myapp
 1562  1000 Z    myapp

# Kill the parent (PID 1000)
$ kill 1000        # try SIGTERM first
$ kill -9 1000     # SIGKILL if needed

# Zombies become orphans of init, init reaps them
$ ps -eo stat | grep -c Z
0
```

### Option 3: The Double-Fork Trick

A clever technique to avoid zombies entirely: fork twice, have the middle process exit immediately, so the grandchild is orphaned to init:

```
Parent (PID 1000)
│
├── fork() ──> Intermediate (PID 1001)
│                    │
│                    ├── fork() ──> Grandchild (PID 1002)
│                    │                    │
│                    │                    │ (does actual work)
│                    │                    │
│                    exit(0)  <── immediately exits
│
│   wait() ──> reaps PID 1001 (instant, it already exited)
│
│   (Parent continues, no zombie)

Meanwhile:
Grandchild (PID 1002) is orphaned → adopted by init
When grandchild exits → init reaps it → no zombie
```

The parent only waits for the intermediate process (which exits immediately, so it's fast). The grandchild is adopted by init, so no zombie accumulates. This is the classic Unix daemon creation technique.

---

## SIGCHLD: The Signal That Ties It Together

When a child exits, the kernel sends `SIGCHLD` to the parent. How the parent handles this signal determines whether zombies accumulate:

| Parent's SIGCHLD handling | Result |
|--------------------------|--------|
| Default (SIG_DFL) | Signal delivered but parent must still call wait() manually. Zombies accumulate if it doesn't. |
| Ignored (SIG_IGN) | Kernel auto-reaps children. No zombies ever. No exit status available. |
| Custom handler that calls waitpid() | Best practice for servers. Reaps children as they exit. |
| Parent never handles it | Zombies accumulate indefinitely. |

---

## Why Long-Running Servers Must Handle SIGCHLD

Any server that spawns child processes (Apache pre-fork, PostgreSQL per-connection backends, any process that calls `fork()`) **must** handle `SIGCHLD` or call `wait()`. Otherwise:

```
Day 1:    5 zombies     (no one notices)
Day 7:    200 zombies   (still fine)
Day 30:   5000 zombies  (getting close to pid_max)
Day 45:   fork() fails! (PID space exhausted)
          New connections rejected.
          Service degraded or down.
```

This is a real production failure mode. It's slow-building and insidious because everything works fine until you run out of PIDs.

---

## Practical Commands Reference

```bash
# Find all zombie processes
ps aux | awk '$8 ~ /^Z/'

# Find zombie count
ps -eo stat | grep -c '^Z'

# Find which parent is creating zombies
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/ {print $2}' | sort | uniq -c | sort -rn

# Check PID limits
cat /proc/sys/kernel/pid_max

# Check current PID usage
ls /proc | grep -c '^[0-9]'

# Monitor in real-time
watch -n 1 'ps -eo stat | grep -c Z'
```

---

## Real-World Connection

**Kubernetes and zombie reaping**: Container entrypoints that fork child processes can accumulate zombies because PID 1 inside the container is the application, not init. If the app doesn't reap children, zombies pile up inside the container. This is why Kubernetes added the `shareProcessNamespace` option and why tools like `tini` or `dumb-init` exist — they act as PID 1 inside containers and properly reap zombies.

**systemd and PID 1 responsibilities**: `systemd` (PID 1 on most modern Linux distributions) is designed to handle orphan processes. It has an internal loop that calls `waitpid(-1, ...)` to collect exit statuses of any adopted orphans. Before systemd, SysV `init` did the same thing. This is why PID 1 is special — it's the universal parent that cleans up after everyone.

**CI/CD pipelines**: Build systems that spawn many short-lived child processes (compilers, test runners, linters) are prone to zombie accumulation if the orchestrator doesn't properly wait for each child. Jenkins, GitLab Runner, and similar tools must handle SIGCHLD carefully.

---

## Interview Angle

**Q: What is a zombie process and why does it exist?**

A: A zombie is a process that has exited but whose exit status hasn't been collected by its parent via wait(). It exists because Unix guarantees that a parent can always retrieve a child's exit status after the child terminates. The kernel keeps a minimal PCB entry (PID, exit status, resource usage) until the parent reads it. Zombies consume no CPU or memory, but they occupy a PID slot. Too many zombies can exhaust the PID space and prevent new processes from being created.

**Q: How do you fix zombie processes?**

A: The proper fix is to make the parent call wait() or waitpid(). In a server, install a SIGCHLD handler that calls waitpid(-1, NULL, WNOHANG) in a loop to reap all finished children. Alternatively, set SIGCHLD to SIG_IGN to have the kernel auto-reap. If you can't fix the parent, killing it makes init adopt and reap the zombies. For new designs, the double-fork trick avoids the problem entirely.

**Q: What's the difference between a zombie and an orphan?**

A: A zombie is dead — the process has exited but its parent hasn't collected the exit status, so a PCB entry lingers. An orphan is alive — the process is still running but its parent has exited. Orphans are adopted by init (PID 1), which properly reaps them when they eventually exit. Zombies are a problem (PID leak); orphans are usually not.

**Q: Why is PID 1 special?**

A: PID 1 (init/systemd) is the ancestor of all processes. When any process's parent dies, the orphaned children are re-parented to PID 1. PID 1 is responsible for calling wait() on these adopted children to prevent zombie accumulation. PID 1 also cannot be killed by SIGKILL — the kernel protects it because if PID 1 dies, the system panics. In containers, this responsibility matters because the container's PID 1 must handle reaping, which is why tools like tini exist.

---

**Next section**: [03-Threads](../03-Threads/README.md)
