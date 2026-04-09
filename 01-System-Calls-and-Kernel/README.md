# 01 - System Calls and the Kernel

System calls are **the** interface between your programs and the operating system kernel. Every time your code opens a file, allocates memory, creates a process, or sends a network packet, it ultimately funnels through a system call. Understanding this boundary is what separates engineers who _use_ an OS from engineers who _understand_ one.

This section takes you from "what is a system call?" all the way through the trap mechanism, the major POSIX calls you'll encounter in real systems, and how the kernel can be extended at runtime with modules and eBPF.

---

## Why This Matters

- **Performance debugging**: When your service is slow, `strace` and syscall-level analysis reveal whether the bottleneck is in your code or in kernel operations (disk I/O, network, scheduling).
- **Security**: Every privilege escalation exploit involves crossing the user-kernel boundary. Understanding how that boundary works is the first step to defending it.
- **Containers and cloud**: Docker, Kubernetes, and cloud VMs all depend on specific system calls (`clone`, `unshare`, `pivot_root`, `seccomp`). You can't reason about container isolation without understanding syscalls.
- **Interviews**: System call mechanics are a staple at systems-focused companies (Google, Meta, Amazon, any infra role).

---

## Prerequisites

| Section | Concepts You Need |
|---------|-------------------|
| [00-Introduction](../00-Introduction/README.md) | User mode vs kernel mode, what an OS does, basic hardware concepts |

Make sure you understand the **privilege ring model** (Ring 0 = kernel, Ring 3 = user) before diving in. Everything in this section builds on that foundation.

---

## Reading Order

| # | File | Topic | Time |
|---|------|-------|------|
| 1 | [01-what-are-system-calls.md](01-what-are-system-calls.md) | What system calls are and why they exist | 15 min |
| 2 | [02-kernel-structure-and-responsibilities.md](02-kernel-structure-and-responsibilities.md) | What the kernel does and how it's organized | 15 min |
| 3 | [03-system-call-mechanism-traps-and-interrupts.md](03-system-call-mechanism-traps-and-interrupts.md) | The trap mechanism, interrupts, and IDT | 20 min |
| 4 | [04-common-system-calls-posix.md](04-common-system-calls-posix.md) | Major POSIX syscalls categorized, plus strace | 20 min |
| 5 | [05-kernel-modules-and-extensions.md](05-kernel-modules-and-extensions.md) | Loadable kernel modules and eBPF | 15 min |

---

## Key Interview Questions

1. **What happens when a user program calls `read()` on a file?** Trace the path from the C library call through the trap into kernel mode, the VFS layer, the actual disk I/O, and the return to user space.

2. **What is the difference between a system call, a library call, and a function call?** Know which crosses the privilege boundary and which doesn't.

3. **Why can't user programs just call kernel functions directly?** This tests your understanding of memory protection, privilege rings, and why the controlled entry point (trap gate) exists.

4. **Explain the difference between hardware interrupts and software interrupts (traps).** Be ready to give examples of each and explain how the kernel handles them differently.

5. **What is a kernel module and when would you use one instead of compiling functionality into the kernel?** Discuss tradeoffs: flexibility vs security, boot-time vs runtime loading, and the role of `modprobe`.

---

**Next section**: [02-Processes](../02-Processes/README.md)
