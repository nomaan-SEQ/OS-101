# OS 101 — Operating Systems from the Ground Up

A structured, incrementally built learning repository for understanding Operating Systems. Each topic builds on the previous one, creating a mental map that connects theory to real-world systems.

## Who Is This For?

- Software engineers who want to **level up their OS knowledge** for daily work
- Engineers preparing for **system design and OS interviews**
- Anyone who wants to understand **what happens beneath your code** — from process scheduling to how Linux actually manages memory

## How to Use This Repo

1. **Go in order.** Folders are numbered `00` through `16`. Each one builds on concepts from earlier folders.
2. **Start with the folder README.** Every folder has a `README.md` that introduces the topic, explains why it matters, and tells you the order to read the subtopic files.
3. **Subtopic files are numbered.** Inside each folder, read `01-xxx.md` before `02-xxx.md`, and so on.
4. **Don't skip the case studies.** Folders `15` and `16` tie everything together by showing how Linux and Windows actually implement the concepts.

## Fast Track (Interview Prep)

Short on time? These five folders cover the highest-frequency OS interview topics:

| Priority | Folder | Why |
|----------|--------|-----|
| 1 | [02-Processes](./02-Processes/) | Foundation of everything — process lifecycle, context switching, fork/exec |
| 2 | [05-Concurrency-and-Synchronization](./05-Concurrency-and-Synchronization/) | Locks, deadlocks, race conditions — the #1 OS interview topic |
| 3 | [07-Memory-Management](./07-Memory-Management/) | Virtual memory, paging, page replacement — asked in every systems interview |
| 4 | [03-Threads](./03-Threads/) | Threads vs processes, threading models — fundamental for any backend role |
| 5 | [04-CPU-Scheduling](./04-CPU-Scheduling/) | Scheduling algorithms, CFS — frequently asked, easy to score points |

---

## Table of Contents

### Part I — Foundations

#### [00-Introduction](./00-Introduction/)
> What is an OS? Why should you care? The big picture before we dive in.

- `01-what-is-an-os.md` — Definition, role, and responsibilities of an operating system
- `02-os-history-and-evolution.md` — From batch systems to modern OSes
- `03-os-architecture-monolithic-micro-hybrid.md` — Monolithic vs microkernel vs hybrid designs
- `04-user-mode-vs-kernel-mode.md` — The fundamental protection boundary
- `05-how-os-boots.md` — What happens from power-on to login screen

#### [01-System-Calls-and-Kernel](./01-System-Calls-and-Kernel/)
> The interface between your programs and the OS. Everything else goes through here.

- `01-what-are-system-calls.md` — How user programs request OS services
- `02-kernel-structure-and-responsibilities.md` — What the kernel does and how it's organized
- `03-system-call-mechanism-traps-and-interrupts.md` — The trap instruction, interrupt handling, mode switching
- `04-common-system-calls-posix.md` — open, read, write, fork, exec, wait, and friends
- `05-kernel-modules-and-extensions.md` — Loadable modules, extending the kernel at runtime

---

### Part II — Core Abstractions

#### [02-Processes](./02-Processes/)
> The most fundamental OS abstraction — a running program and everything the OS tracks about it.

- `01-what-is-a-process.md` — Process definition, address space, and resources
- `02-process-lifecycle-and-states.md` — New, ready, running, waiting, terminated
- `03-process-control-block.md` — The data structure that represents a process
- `04-context-switching.md` — How the OS switches between processes (and the cost)
- `05-process-creation-fork-exec.md` — fork(), exec(), and the Unix process model
- `06-orphan-and-zombie-processes.md` — What happens when processes misbehave

#### [03-Threads](./03-Threads/)
> Lightweight execution within a process. Shared memory, shared problems.

- `01-threads-vs-processes.md` — What threads share, what they don't, and when to use which
- `02-user-threads-vs-kernel-threads.md` — Who manages the thread — the OS or a library?
- `03-threading-models.md` — One-to-one, many-to-one, many-to-many
- `04-thread-pools-and-practical-patterns.md` — How real applications use threads
- `05-green-threads-goroutines-fibers.md` — Modern lightweight concurrency models

#### [04-CPU-Scheduling](./04-CPU-Scheduling/)
> How the OS decides which process/thread gets the CPU and for how long.

- `01-scheduling-concepts-and-criteria.md` — Throughput, turnaround time, fairness, starvation
- `02-fcfs-sjf-priority-scheduling.md` — Classic scheduling algorithms
- `03-round-robin-and-multilevel-queues.md` — Time-slicing and priority-based approaches
- `04-multiprocessor-and-multicore-scheduling.md` — Scheduling across multiple CPUs
- `05-real-time-scheduling.md` — Hard and soft real-time constraints
- `06-linux-cfs-and-real-world-schedulers.md` — How Linux actually schedules (CFS, EEVDF)

---

### Part III — Concurrency and Communication

#### [05-Concurrency-and-Synchronization](./05-Concurrency-and-Synchronization/)
> When threads share resources, chaos follows. This is how you tame it.

- `01-race-conditions-and-critical-sections.md` — The fundamental problem of shared state
- `02-mutexes-and-spinlocks.md` — Mutual exclusion mechanisms
- `03-semaphores.md` — Counting and binary semaphores
- `04-monitors-and-condition-variables.md` — Higher-level synchronization
- `05-classic-problems-producer-consumer-readers-writers-dining-philosophers.md` — The problems every OS course teaches (and interviews ask)
- `06-deadlocks-detection-prevention-avoidance.md` — When everyone waits for everyone else
- `07-lock-free-and-wait-free-programming.md` — Advanced: avoiding locks entirely

#### [06-Inter-Process-Communication](./06-Inter-Process-Communication/)
> How processes that don't share memory talk to each other.

- `01-ipc-overview-and-taxonomy.md` — The landscape of IPC mechanisms
- `02-pipes-and-fifos.md` — The simplest form of IPC (and how shell pipes work)
- `03-message-queues.md` — Structured message passing
- `04-shared-memory.md` — Fast IPC through shared address space regions
- `05-sockets-for-ipc.md` — Unix domain sockets and network sockets for local communication
- `06-signals.md` — Asynchronous notifications between processes

---

### Part IV — Memory

#### [07-Memory-Management](./07-Memory-Management/)
> How the OS gives every process the illusion of having all the memory to itself.

- `01-memory-hierarchy-and-address-spaces.md` — Registers, cache, RAM, disk — and logical vs physical addresses
- `02-contiguous-allocation-and-fragmentation.md` — The simplest approach and why it breaks down
- `03-paging.md` — Dividing memory into fixed-size pages
- `04-segmentation.md` — Dividing memory by logical segments
- `05-virtual-memory-and-demand-paging.md` — The big idea: more memory than you physically have
- `06-page-replacement-algorithms.md` — FIFO, LRU, Clock — what to evict when memory is full
- `07-tlb-and-page-table-structures.md` — Making address translation fast (TLB, multi-level page tables)
- `08-thrashing-and-working-set-model.md` — When paging goes wrong
- `09-memory-mapped-files-and-mmap.md` — Mapping files into memory for efficient I/O

---

### Part V — Storage, Files, and I/O

#### [08-Storage-Management](./08-Storage-Management/)
> The physical layer: disks, SSDs, and how the OS talks to them.

- `01-disk-structure-hdd-ssd-nvme.md` — How storage hardware works (sectors, blocks, flash)
- `02-disk-scheduling-algorithms.md` — FCFS, SSTF, SCAN, C-SCAN
- `03-raid-levels.md` — Redundancy and performance through disk arrays
- `04-io-scheduling-in-linux.md` — CFQ, deadline, noop/none, BFQ

#### [09-File-Systems](./09-File-Systems/)
> The logical layer that organizes data on storage into files and directories.

- `01-file-system-concepts-and-interfaces.md` — Files, directories, metadata, permissions
- `02-directory-structure-and-path-resolution.md` — How `/home/user/file.txt` gets resolved
- `03-file-allocation-methods.md` — Contiguous, linked, indexed allocation
- `04-free-space-management.md` — Bitmaps, free lists, and space tracking
- `05-inodes-and-unix-file-system.md` — The inode structure and classic Unix FS design
- `06-journaling-and-log-structured-file-systems.md` — Crash consistency and recovery
- `07-vfs-and-fuse.md` — Virtual file system layer and user-space file systems
- `08-modern-file-systems-ext4-xfs-btrfs-zfs.md` — What's actually used today and why

#### [10-IO-Systems](./10-IO-Systems/)
> How the OS manages all input/output — not just disks, but every device.

- `01-io-hardware-overview.md` — Buses, controllers, ports, and device registers
- `02-io-techniques-polling-interrupts-dma.md` — Three ways to do I/O (and their tradeoffs)
- `03-device-drivers.md` — The software layer between OS and hardware
- `04-blocking-nonblocking-async-io.md` — Synchronous vs asynchronous I/O models
- `05-io-multiplexing-select-poll-epoll.md` — Handling thousands of connections efficiently

---

### Part VI — Security

#### [11-Security-and-Protection](./11-Security-and-Protection/)
> How the OS protects processes from each other and the system from malicious code.

- `01-protection-domains-and-access-control.md` — Rings, domains, and the principle of least privilege
- `02-authentication-and-authorization.md` — Who are you, and what can you do?
- `03-access-control-lists-and-capabilities.md` — Two models for managing permissions
- `04-common-attacks-buffer-overflow-privilege-escalation.md` — How attackers exploit OS-level vulnerabilities
- `05-sandboxing-and-mandatory-access-control.md` — Containing damage: sandboxes, SELinux, AppArmor
- `06-secure-boot-and-trusted-computing.md` — Ensuring the system boots into a trusted state

---

### Part VII — Virtualization and Networking

#### [12-Virtualization-and-Containers](./12-Virtualization-and-Containers/)
> Running multiple OSes on one machine, and the lighter-weight alternative: containers.

- `01-virtualization-concepts-and-hypervisors.md` — Type 1 vs Type 2 hypervisors, full vs para-virtualization
- `02-hardware-assisted-virtualization.md` — VT-x, AMD-V, and how CPUs support virtualization
- `03-os-level-virtualization-namespaces-cgroups.md` — The kernel features that make containers possible
- `04-containers-docker-containerd.md` — How Docker and container runtimes actually work
- `05-vm-vs-container-tradeoffs.md` — When to use VMs vs containers (and when to combine them)
- `06-orchestration-kubernetes-basics.md` — Managing containers at scale

#### [13-Networking-from-OS-Perspective](./13-Networking-from-OS-Perspective/)
> How the OS implements networking — from the kernel network stack to zero-copy I/O.

- `01-network-stack-in-the-kernel.md` — The path of a packet through the kernel
- `02-sockets-api-tcp-udp.md` — The programming interface for network communication
- `03-network-namespaces-and-virtual-networking.md` — Network isolation for containers
- `04-zero-copy-io-and-kernel-bypass.md` — High-performance networking: sendfile, DPDK, io_uring

---

### Part VIII — Advanced and Modern Topics

#### [14-Modern-OS-Concepts](./14-Modern-OS-Concepts/)
> Where OS design is heading — new abstractions, new boundaries, new challenges.

- `01-microservices-and-os-implications.md` — How microservice architecture changes what we need from the OS
- `02-ebpf-and-kernel-observability.md` — Running sandboxed programs in the kernel for tracing and security
- `03-unikernels-and-library-os.md` — Single-purpose VMs with no traditional OS
- `04-wasm-runtimes-and-os-boundary.md` — WebAssembly as a new OS-like abstraction
- `05-os-for-distributed-systems.md` — Scheduling, memory, and storage across machines

---

### Part IX — Case Studies

#### [15-Case-Study-Linux-Internals](./15-Case-Study-Linux-Internals/)
> How Linux actually implements everything we've learned. Theory meets reality.

- `01-linux-kernel-architecture-overview.md` — Monolithic kernel, source tree layout, subsystem overview
- `02-linux-process-and-thread-implementation.md` — task_struct, clone(), NPTL threads
- `03-linux-memory-management-internals.md` — Page tables, slab allocator, OOM killer
- `04-linux-file-systems-vfs-ext4-proc.md` — VFS layer, ext4 internals, /proc and /sys
- `05-linux-io-and-networking-internals.md` — Block I/O layer, netfilter, socket buffers
- `06-linux-security-selinux-apparmor.md` — LSM framework, SELinux, AppArmor, seccomp
- `07-linux-boot-process-in-depth.md` — BIOS/UEFI → GRUB → kernel → systemd
- `08-linux-cgroups-namespaces-deep-dive.md` — The building blocks of containers, in detail

#### [16-Case-Study-Windows-Internals](./16-Case-Study-Windows-Internals/)
> How Windows does it differently. Contrast with Linux to deepen understanding.

- `01-windows-architecture-overview-nt-kernel.md` — NT kernel, HAL, Executive, subsystem architecture
- `02-windows-process-and-thread-model.md` — EPROCESS, ETHREAD, job objects, fibers
- `03-windows-memory-management-internals.md` — VADs, working sets, page file, AWE
- `04-windows-file-systems-ntfs-refs.md` — MFT, alternate data streams, ReFS
- `05-windows-io-model-and-driver-framework.md` — IRP-based I/O, WDM, WDF, KMDF/UMDF
- `06-windows-security-model-tokens-acls.md` — Access tokens, SIDs, DACLs, SACLs, integrity levels
- `07-windows-boot-process-uefi-bootmgr.md` — UEFI → bootmgr → winload → ntoskrnl
- `08-windows-registry-and-system-services.md` — Registry hives, SCM, services architecture

---

## Recommended References

- **OSTEP** (Operating Systems: Three Easy Pieces) — Free online, excellent for building intuition
- **Silberschatz — Operating System Concepts** — The classic textbook
- **Tanenbaum — Modern Operating Systems** — Great for design perspective
- **Love — Linux Kernel Development** — For the Linux case study
- **Russinovich — Windows Internals** — For the Windows case study
- **man7.org** — Linux man pages and system call documentation

---

## Progress Tracker

| # | Folder | Status |
|---|--------|--------|
| 00 | Introduction | Done |
| 01 | System Calls and Kernel | Done |
| 02 | Processes | Done |
| 03 | Threads | Done |
| 04 | CPU Scheduling | Done |
| 05 | Concurrency and Synchronization | Done |
| 06 | Inter-Process Communication | Done |
| 07 | Memory Management | Done |
| 08 | Storage Management | Done |
| 09 | File Systems | Done |
| 10 | I/O Systems | Done |
| 11 | Security and Protection | Done |
| 12 | Virtualization and Containers | Done |
| 13 | Networking from OS Perspective | Done |
| 14 | Modern OS Concepts | Done |
| 15 | Case Study: Linux Internals | Done |
| 16 | Case Study: Windows Internals | Done |
