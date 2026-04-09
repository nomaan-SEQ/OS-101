# 02 - Processes

A process is **the fundamental unit of execution** in an operating system — a running program with its own address space, registers, open files, and identity. When you type `./myapp` in a terminal, the OS doesn't just load bytes from disk — it creates a living, breathing process with its own isolated world of memory and state.

This is THE most important concept in operating systems. Threads, scheduling, synchronization, memory management, IPC — every single topic that follows is built on top of processes. If you don't deeply understand what a process is, how it's created, how it transitions between states, and how the kernel tracks it, everything else will feel like floating abstractions.

---

## Why This Matters

- **Debugging production systems**: When you run `ps`, `top`, `htop`, or `strace`, you're looking at processes. Understanding what those states (R, S, D, Z) mean is the difference between guessing and diagnosing.
- **Containers and isolation**: A Docker container is just a process (or group of processes) with extra namespace and cgroup restrictions. There's no VM, no separate kernel — just processes.
- **Performance**: Context switching between processes has real cost. Knowing that cost helps you reason about architecture decisions (threads vs processes, connection-per-process vs event loop).
- **Every system call operates on a process**: `fork()`, `exec()`, `wait()`, `kill()`, `nice()` — the kernel's process API is the foundation of Unix.
- **Interviews**: Process lifecycle, fork/exec, zombies, and context switching are among the most frequently asked OS questions.

---

## Prerequisites

| Section | Concepts You Need |
|---------|-------------------|
| [00-Introduction](../00-Introduction/README.md) | What an OS does, user mode vs kernel mode, basic hardware concepts |
| [01-System-Calls-and-Kernel](../01-System-Calls-and-Kernel/README.md) | System call mechanism, traps, how user programs talk to the kernel |

You should be comfortable with the idea that **user programs can't directly access hardware or kernel memory** — they must go through system calls. Everything in this section happens on both sides of that boundary.

---

## Reading Order

| # | File | Topic | Time |
|---|------|-------|------|
| 1 | [01-what-is-a-process.md](01-what-is-a-process.md) | Process definition, memory layout, identity | 15 min |
| 2 | [02-process-lifecycle-and-states.md](02-process-lifecycle-and-states.md) | The 5-state model, transitions, and Linux ps states | 15 min |
| 3 | [03-process-control-block.md](03-process-control-block.md) | PCB, task_struct, process table | 15 min |
| 4 | [04-context-switching.md](04-context-switching.md) | How the CPU switches between processes | 15 min |
| 5 | [05-process-creation-fork-exec.md](05-process-creation-fork-exec.md) | fork(), exec(), wait(), Copy-on-Write | 20 min |
| 6 | [06-orphan-and-zombie-processes.md](06-orphan-and-zombie-processes.md) | Zombies, orphans, and how to handle them | 15 min |

---

## Key Interview Questions

1. **What is the difference between a process and a program?** A program is a static file on disk. A process is that program in execution — with its own address space, registers, and kernel state. Multiple processes can run the same program simultaneously.

2. **Walk through the lifecycle of a process from creation to termination.** Cover fork, the ready queue, being scheduled onto a CPU, potentially blocking on I/O, and eventually calling exit. Mention what happens to the PCB at each stage.

3. **What is a context switch and why is it expensive?** It involves saving all CPU state (registers, program counter, stack pointer) for the current process, switching page tables, and loading the state for the next process. The direct cost is microseconds, but the indirect cost (cache/TLB pollution) can be much worse.

4. **Explain fork() and exec(). Why are they separate calls?** Separating them gives the parent a window between fork and exec to set up the child's environment — redirect file descriptors, change the working directory, drop privileges. This is how shells implement pipes and redirection.

5. **What is a zombie process and how do you fix it?** A zombie is a process that has exited but whose parent hasn't called wait() to collect its exit status. The fix is to make the parent handle SIGCHLD or call wait(). If the parent is buggy, killing the parent makes init adopt and reap the zombie.

---

**Next section**: [03-Threads](../03-Threads/README.md)
