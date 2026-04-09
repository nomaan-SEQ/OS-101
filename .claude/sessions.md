---
objective: Track progress across sessions for the OS 101 learning repository.
purpose: This file logs what was accomplished in each session so that any future session
  can quickly understand the current state, what's done, and what's remaining.
repo-goal: Build a comprehensive, incrementally structured OS learning repo with 17 topic
  folders covering fundamentals through Linux/Windows internals case studies.
total-folders: 17 (00-Introduction through 16-Case-Study-Windows-Internals)
approach: One folder at a time — root README.md first, then content folder-by-folder.
---

# Session Log

## Session 1 — 2026-04-08

### Completed
- Created root `README.md` with full table of contents, roadmap, and fast track
- Created `.claude/sessions.md` (this file) for session tracking

### Notes
- Only the root README.md was created in this session
- All 17 topic folders and their subtopic .md files are pending for future sessions
- Next session should start with `00-Introduction/`

## Session 2 — 2026-04-08

### Completed
- Created `00-Introduction/` folder with all 6 files:
  - `README.md` — Section overview, reading order, prerequisites, key interview questions
  - `01-what-is-an-os.md` — OS definition, two roles (resource manager + abstraction provider), kernel vs OS distinction
  - `02-os-history-and-evolution.md` — Batch → multiprogramming → time-sharing → personal → distributed → cloud/containers
  - `03-os-architecture-monolithic-micro-hybrid.md` — Three kernel architectures with tradeoffs, comparison table
  - `04-user-mode-vs-kernel-mode.md` — Protection boundary, mode switching (syscall/interrupt/exception), x86 rings, real-world connections
  - `05-how-os-boots.md` — Full boot sequence: firmware → bootloader → kernel → init → login, initramfs explanation
- Updated progress tracker in root README.md

### Notes
- Each file includes: concept explanation, ASCII diagrams, real-world connections, and interview Q&A
- Next session should start with `01-System-Calls-and-Kernel/`

## Session 3 — 2026-04-09

### Completed
- Created 5 sections (34 files): `01-System-Calls-and-Kernel/` (6), `02-Processes/` (7), `03-Threads/` (6), `04-CPU-Scheduling/` (7), `05-Concurrency-and-Synchronization/` (8)
- Sections 00-05 complete (6 of 17 folders done)

## Session 4 — 2026-04-09

### Completed
- Created 5 sections (37 files): `06-Inter-Process-Communication/` (7), `07-Memory-Management/` (10), `08-Storage-Management/` (5), `09-File-Systems/` (9), `10-IO-Systems/` (6)
- Sections 00-10 complete (11 of 17 folders done)

## Session 5 — 2026-04-09

### Completed
- Created 4 sections (25 files):

#### `11-Security-and-Protection/` (7 files)
- README.md, 01-protection-domains-and-access-control.md, 02-authentication-and-authorization.md, 03-access-control-lists-and-capabilities.md, 04-common-attacks-buffer-overflow-privilege-escalation.md, 05-sandboxing-and-mandatory-access-control.md, 06-secure-boot-and-trusted-computing.md

#### `12-Virtualization-and-Containers/` (7 files)
- README.md, 01-virtualization-concepts-and-hypervisors.md, 02-hardware-assisted-virtualization.md, 03-os-level-virtualization-namespaces-cgroups.md, 04-containers-docker-containerd.md, 05-vm-vs-container-tradeoffs.md, 06-orchestration-kubernetes-basics.md

#### `13-Networking-from-OS-Perspective/` (5 files)
- README.md, 01-network-stack-in-the-kernel.md, 02-sockets-api-tcp-udp.md, 03-network-namespaces-and-virtual-networking.md, 04-zero-copy-io-and-kernel-bypass.md

#### `14-Modern-OS-Concepts/` (6 files)
- README.md, 01-microservices-and-os-implications.md, 02-ebpf-and-kernel-observability.md, 03-unikernels-and-library-os.md, 04-wasm-runtimes-and-os-boundary.md, 05-os-for-distributed-systems.md

- Updated progress tracker in root README.md (sections 11-14 marked Done)

### Notes
- Sections 00-14 are now complete (15 of 17 folders done)
- Only 2 case study sections remain: `15-Case-Study-Linux-Internals/` and `16-Case-Study-Windows-Internals/`
- Next session should start with `15-Case-Study-Linux-Internals/`

## Session 6 — 2026-04-09

### Completed
- Created the final 2 case study sections (18 files total):

#### `15-Case-Study-Linux-Internals/` (9 files, ~150KB)
- `README.md` — Linux history, prerequisites, 7 key interview questions
- `01-linux-kernel-architecture-overview.md` — Monolithic+modular design, source tree layout, /proc and /sys, kernel config
- `02-linux-process-and-thread-implementation.md` — task_struct, clone() unification, TGID vs PID, fork/COW internals, NPTL
- `03-linux-memory-management-internals.md` — x86-64 virtual layout, buddy system, SLUB allocator, page cache, OOM killer, zones, THP
- `04-linux-file-systems-vfs-ext4-proc.md` — VFS objects + operations tables, dentry cache, ext4 internals, procfs, sysfs, tmpfs
- `05-linux-io-and-networking-internals.md` — Block I/O + bio + blk-mq, device mapper/LVM, NAPI, sk_buff, netfilter, io_uring
- `06-linux-security-selinux-apparmor.md` — LSM framework, SELinux type enforcement, AppArmor, seccomp-bpf, capabilities, Landlock
- `07-linux-boot-process-in-depth.md` — UEFI, GRUB2, start_kernel() walkthrough, initramfs, systemd, boot optimization
- `08-linux-cgroups-namespaces-deep-dive.md` — All namespace types, cgroups v2 controllers, PSI, building a container from scratch

#### `16-Case-Study-Windows-Internals/` (9 files)
- `README.md` — NT history, prerequisites, 6 key interview questions
- `01-windows-architecture-overview-nt-kernel.md` — NT hybrid architecture, Executive components, Object Manager, SSDT, ntdll.dll
- `02-windows-process-and-thread-model.md` — EPROCESS/ETHREAD, PEB/TEB, 32-level priority, Jobs (≈cgroups), Fibers, CreateProcess
- `03-windows-memory-management-internals.md` — VAD tree, Working Set Manager, PFN Database, Section objects, memory compression
- `04-windows-file-systems-ntfs-refs.md` — MFT, Alternate Data Streams, NTFS ACLs, ReFS, Filter Manager/minifilters
- `05-windows-io-model-and-driver-framework.md` — IRP model, device stacks, IOCP (vs epoll), WDM/KMDF/UMDF, PnP/Power Manager
- `06-windows-security-model-tokens-acls.md` — Access tokens, SIDs, DACLs/ACEs, Mandatory Integrity Control, UAC, privileges
- `07-windows-boot-process-uefi-bootmgr.md` — bootmgfw.efi → winload → ntoskrnl phases, smss → services → logon chain
- `08-windows-registry-and-system-services.md` — Registry hives, SCM, svchost, Task Scheduler, WMI, Event Log, ETW

- Updated progress tracker in root README.md (all 17 sections marked Done)

### Notes
- All case study files include rich analogies for complex concepts and Linux↔Windows comparisons throughout
- **THE REPOSITORY IS NOW COMPLETE: 17 folders, ~115 files, all content written**
- All files follow consistent style: ASCII diagrams, comparison tables, real-world connections, interview Q&A

## Session 7 — 2026-04-09

### Completed
- Full audit of all ~115 files across 17 sections for missing explanations, undefined terms, and complex topics lacking analogies
- Applied ~70 inline fixes across all sections:

#### Sections 00-02 (Introduction, System Calls, Processes)
- MMU definition, GDT/IDT explanations, VFS definition, page tables forward references
- KPTI expansion, IDT expansion in boot file, DKMS definition, eBPF verifier explanation
- Top-half/bottom-half interrupt analogy (restaurant: waiter vs kitchen)
- Process states analogy (university students), load average definition
- Context switching analogy (chef cooking multiple dishes), MMU/COW clarification

#### Sections 03-05 (Threads, CPU Scheduling, Concurrency)
- GIL (Global Interpreter Lock) expanded definition
- User-level threads analogy (shared phone line)
- CPU burst analogy (sprints and rest periods), CFS parenthetical definition
- Exponential averaging explanation (weather forecast analogy)
- NUMA analogy (office building supply closets), interconnect definition
- Mutex vs spinlock analogy (waiting room vs door), RAII definition
- Semaphore analogy (parking lot with electronic sign)
- Condition variable analogy (restaurant pager system)
- Banker's Algorithm analogy (bank loan approval)

#### Sections 06-10 (IPC, Memory, Storage, File Systems, I/O)
- Shared memory analogy (whiteboard), ftok() explanation
- SRAM vs DRAM explanation, CR3 register definition
- Reference/dirty bit definitions, Clock algorithm analogy (security guard)
- TLB analogy (speed dial), working set analogy (student's desk)
- FTL and P/E cycles explanations, SCAN elevator analogy, XOR parity explanation
- File allocation analogies (theater seats, treasure hunt, book index)
- Inode indirection analogy (filing system), link count explanation
- Journaling analogy (pilot's pre-flight checklist), B+ tree definition
- PCIe lanes explanation, IOMMU definition, DMA analogy (delivery key)
- Polling analogy (mailbox), interrupts analogy (doorbell)
- EAGAIN explanation, epoll analogy (hand-raising vs checking each student)

#### Sections 11-16 (Security, Virtualization, Networking, Modern, Case Studies)
- ROP analogy (ransom note from newspaper), gadget definition
- VMX root/non-root analogy (puppet theater)
- pivot_root explanation, PSI definition, cgroups v1 vs v2 analogy (building managers)
- NAPI hybrid interrupt/polling explanation
- Nagle's algorithm analogy (bus stop batching), VXLAN motivation
- Scatter-Gather DMA explanation
- eBPF bytecode explanation, verifier static analysis detail, kprobes/uprobes definitions
- Unikernel analogy (food truck vs restaurant)

### Notes
- No content was rewritten — all changes are inline additions (2-4 sentences each)
- All analogies use consistent > **Analogy:** blockquote format
- All key terms use **bold** formatting
- Repository now has comprehensive explanations for all referenced concepts
