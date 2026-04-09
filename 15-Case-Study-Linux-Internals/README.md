# 15 - Case Study: Linux Internals

You have spent 14 sections learning operating system theory — processes, threads, scheduling, memory management, filesystems, I/O, security, virtualization, networking. Now it is time to see how all of that actually works in the most important operating system on the planet: **Linux**.

This is not a rehash of theory. This section takes the concepts you already know and shows you how Linux implements them in practice. When we talked about "the page table" in the memory section, Linux uses a 4-level radix tree with specific data structures. When we talked about "the process control block," Linux calls it `task_struct` and it is one of the most complex structures in the kernel at over 600 fields. When we talked about "the filesystem," Linux has a brilliant abstraction layer (VFS) that lets dozens of different filesystems coexist behind a single API.

Every major topic from earlier sections appears here, but now with real kernel code paths, real data structures, real `/proc` and `/sys` entries you can inspect on a running system. This is where theory becomes engineering.

---

## Why Linux?

Linux is not just one operating system among many. It is THE operating system to understand deeply:

| Domain | Linux Presence |
|--------|---------------|
| **Servers** | 96%+ of the top 1 million web servers run Linux |
| **Cloud** | AWS, GCP, Azure — the vast majority of cloud instances are Linux |
| **Mobile** | Android runs on a modified Linux kernel (~3 billion devices) |
| **Embedded** | Routers, TVs, cars, IoT devices — Linux everywhere |
| **Supercomputers** | 100% of the TOP500 supercomputers run Linux |
| **Containers** | Docker, Kubernetes — built entirely on Linux kernel features |

If you are going to deeply study one OS kernel, it should be Linux. It is open source (you can read every line), extraordinarily well-documented by the community, and the skills transfer directly to production engineering work.

### A Brief History

In 1991, Linus Torvalds, a 21-year-old Finnish student, posted a message to the comp.os.minix newsgroup announcing a free operating system kernel he had been working on. That kernel — originally targeting i386 PCs — became Linux.

Key design decisions that shaped Linux:
- **Monolithic kernel**: All core subsystems (scheduling, memory, filesystems, networking, drivers) run in a single address space in kernel mode. This is the opposite of a microkernel where subsystems run as separate user-space processes.
- **GPL license**: The GNU General Public License ensures that anyone who modifies and distributes Linux must share their changes. This created a massive feedback loop of contributions.
- **Community-driven development**: Thousands of developers from hundreds of companies contribute to Linux. No single company controls it (though the Linux Foundation coordinates).
- **POSIX-compatible**: Linux implements the POSIX API, making it compatible with the Unix ecosystem of tools and applications.

Today, the Linux kernel has over 30 million lines of code, with thousands of commits per release cycle, making it one of the largest collaborative software projects in human history.

---

## Prerequisites

This is an advanced case study section. You should have completed **all previous sections**:

| Section | Key Concepts You Need |
|---------|----------------------|
| [00-Introduction](../00-Introduction/README.md) | OS role, user/kernel mode, hardware basics |
| [01-System-Calls-and-Kernel](../01-System-Calls-and-Kernel/README.md) | Syscall mechanism, traps, kernel entry/exit |
| [02-Processes](../02-Processes/README.md) | Process model, PCB, fork/exec, lifecycle |
| [03-Threads](../03-Threads/README.md) | Thread models, user vs kernel threads, pthreads |
| [04-CPU-Scheduling](../04-CPU-Scheduling/README.md) | Scheduling algorithms, preemption, priorities |
| [05-Concurrency-and-Synchronization](../05-Concurrency-and-Synchronization/README.md) | Locks, semaphores, deadlock, race conditions |
| [06-Inter-Process-Communication](../06-Inter-Process-Communication/README.md) | Pipes, shared memory, message queues, signals |
| [07-Memory-Management](../07-Memory-Management/README.md) | Virtual memory, paging, page tables, TLB |
| [08-Storage-Management](../08-Storage-Management/README.md) | Disk scheduling, RAID, block I/O |
| [09-File-Systems](../09-File-Systems/README.md) | Inodes, directories, journaling, VFS concept |
| [10-IO-Systems](../10-IO-Systems/README.md) | I/O models, DMA, interrupt handling |
| [11-Security-and-Protection](../11-Security-and-Protection/README.md) | Access control, capabilities, MAC vs DAC |
| [12-Virtualization-and-Containers](../12-Virtualization-and-Containers/README.md) | Namespaces, cgroups, containers, hypervisors |
| [13-Networking-from-OS-Perspective](../13-Networking-from-OS-Perspective/README.md) | Sockets, TCP/IP stack, network I/O |
| [14-Modern-OS-Concepts](../14-Modern-OS-Concepts/README.md) | io_uring, eBPF, modern kernel features |

If any of those feel shaky, revisit them first. This section assumes you know the theory and want to see how the real machine works.

---

## Reading Order

| # | File | Topic | Time |
|---|------|-------|------|
| 1 | [01-linux-kernel-architecture-overview.md](01-linux-kernel-architecture-overview.md) | Monolithic design, source tree, modules, /proc and /sys | 20 min |
| 2 | [02-linux-process-and-thread-implementation.md](02-linux-process-and-thread-implementation.md) | task_struct, clone(), threads-are-processes, NPTL | 25 min |
| 3 | [03-linux-memory-management-internals.md](03-linux-memory-management-internals.md) | Page tables, buddy system, slab allocator, page cache, OOM | 25 min |
| 4 | [04-linux-file-systems-vfs-ext4-proc.md](04-linux-file-systems-vfs-ext4-proc.md) | VFS layer, dentry cache, ext4 internals, procfs, sysfs | 25 min |
| 5 | [05-linux-io-and-networking-internals.md](05-linux-io-and-networking-internals.md) | Block I/O, device mapper, NAPI, sk_buff, io_uring | 25 min |
| 6 | [06-linux-security-selinux-apparmor.md](06-linux-security-selinux-apparmor.md) | LSM framework, SELinux, AppArmor, seccomp, capabilities | 25 min |
| 7 | [07-linux-boot-process-in-depth.md](07-linux-boot-process-in-depth.md) | UEFI, GRUB2, start_kernel(), initramfs, systemd | 20 min |
| 8 | [08-linux-cgroups-namespaces-deep-dive.md](08-linux-cgroups-namespaces-deep-dive.md) | Namespace types, cgroups v2, PSI, building a container | 25 min |

Start at the top and work through in order. Each file builds on the ones before it, and together they form a complete picture of how a modern Linux system operates from boot to shutdown, from process creation to packet delivery.

---

## Key Interview Questions

1. **Linux uses a monolithic kernel. What does that mean, and what are the tradeoffs compared to a microkernel?** All kernel subsystems run in a single address space, enabling direct function calls between them (fast) but meaning a bug in any subsystem can crash the entire kernel. Microkernels isolate subsystems but pay IPC overhead. Linux mitigates the monolithic risk with loadable modules and extensive code review.

2. **In Linux, what is the relationship between processes and threads?** They are both represented by `task_struct`. Threads are created via `clone()` with flags that specify shared resources (address space, file descriptors, etc.). A process is just a `clone()` with nothing shared. There is no separate "thread struct" — the kernel treats them uniformly.

3. **Explain the Linux virtual memory layout on x86-64 and how address translation works.** User space occupies the lower 128 TB, kernel space the upper 128 TB, with a canonical hole in between. Translation goes through 4 levels of page tables (PGD, PUD, PMD, PTE) to reach the physical frame. The TLB caches recent translations to avoid walking the full page table on every access.

4. **What is the VFS layer and why does Linux need it?** VFS provides a uniform interface (open, read, write, close) that works across all filesystems. Each filesystem registers its own operations tables. This lets user programs work identically whether the underlying storage is ext4, XFS, NFS, procfs, or tmpfs — the application code never changes.

5. **How do namespaces and cgroups work together to create containers?** Namespaces provide isolation (what a process can see — its own PID space, network, filesystem view), while cgroups provide resource control (how much a process can use — CPU, memory, I/O). Together, they create the container abstraction without requiring a hypervisor or separate kernel.

6. **Walk through what happens when a Linux system boots, from power-on to a login prompt.** UEFI firmware loads GRUB2, which loads the kernel and initramfs into memory. The kernel decompresses, runs `start_kernel()` to initialize all subsystems, then mounts initramfs and runs its init script to find and mount the real root filesystem. PID 1 (systemd) takes over, builds a dependency graph, starts services in parallel, and eventually launches the login prompt.

7. **What is the OOM killer and how does Linux decide which process to kill?** When the system is truly out of memory and cannot reclaim pages, the OOM killer selects a victim based on `oom_score` — a heuristic that considers memory usage, process age, and the `oom_score_adj` tunable. It kills the chosen process to free memory. Critical services can set `oom_score_adj` to -1000 to avoid being killed.

---

**Next section**: [16-Case-Study-Windows-Internals](../16-Case-Study-Windows-Internals/README.md)
