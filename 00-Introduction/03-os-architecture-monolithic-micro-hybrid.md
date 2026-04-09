# OS Architecture: Monolithic, Microkernel, and Hybrid

## Why Architecture Matters

How you structure the kernel determines the OS's **performance, reliability, and maintainability**. This is one of the oldest and most debated design decisions in OS history (see: the famous Torvalds-Tanenbaum debate).

## The Three Architectures

### 1. Monolithic Kernel

**Everything runs in kernel space.** Process management, memory management, file systems, device drivers, networking — all compiled into a single large binary running in privileged mode.

```
┌──────────────────────────────────────┐
│            User Space                │
│   [App A]   [App B]   [App C]       │
├──────────────────────────────────────┤
│            Kernel Space              │
│  ┌────────────────────────────────┐  │
│  │  Process    Memory    File     │  │
│  │  Mgmt       Mgmt     System   │  │
│  │                                │  │
│  │  Device     Network   IPC     │  │
│  │  Drivers    Stack             │  │
│  │                                │  │
│  │  ALL in one address space      │  │
│  │  ALL in privileged mode        │  │
│  └────────────────────────────────┘  │
├──────────────────────────────────────┤
│             Hardware                 │
└──────────────────────────────────────┘
```

**Examples:** Linux, traditional Unix, FreeBSD, MS-DOS

**Pros:**
- **Fast.** No overhead from message passing between components. A system call goes straight to the relevant subsystem — no context switches in between.
- **Simple communication.** Any kernel component can call any other directly (function call, not IPC).

**Cons:**
- **One bug crashes everything.** A buggy device driver can corrupt the entire kernel's memory and crash the system.
- **Large codebase.** Linux kernel is ~30 million lines. Hard to maintain and debug.
- **Hard to extend safely.** Adding a new driver means adding code to the trusted kernel.

**Mitigation — Loadable Kernel Modules (LKMs):**
Modern monolithic kernels (like Linux) support loadable modules — you can add/remove drivers at runtime without recompiling the kernel. This gives some flexibility while keeping the monolithic architecture.

### 2. Microkernel

**Only the bare minimum runs in kernel space:** inter-process communication (IPC), basic scheduling, and memory management. Everything else — file systems, device drivers, networking — runs as **user-space processes** (called servers).

```
┌──────────────────────────────────────────┐
│              User Space                  │
│  [App A]  [App B]  [File   ] [Device ]  │
│                    [Server ] [Driver ]  │
│                    [Network] [Display]  │
│                    [Server ] [Driver ]  │
│                                          │
│   Apps and OS services are ALL           │
│   user-space processes                   │
├──────────────────────────────────────────┤
│              Kernel Space                │
│  ┌────────────────────────────────────┐  │
│  │  IPC    Scheduling    Memory Mgmt │  │
│  │         (minimal)     (minimal)   │  │
│  │                                    │  │
│  │  Very small — maybe 10K lines      │  │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│               Hardware                   │
└──────────────────────────────────────────┘
```

**Examples:** MINIX 3, QNX, L4 family, seL4, GNU Hurd

**Pros:**
- **Reliability.** If a device driver crashes, it's just a user-space process — restart it without rebooting. The kernel keeps running.
- **Security.** Smaller kernel = smaller attack surface. Easier to formally verify (seL4 is mathematically proven correct).
- **Modularity.** Services are isolated. You can swap out the file system without touching the kernel.

**Cons:**
- **Slower.** A file read that's a single kernel function call in a monolithic kernel becomes: App → IPC to file server → IPC to disk driver → IPC back. Multiple context switches and message copies.
- **Complexity.** Managing all the IPC between services is complex. Debugging distributed interactions is harder than debugging a single kernel.

**The Performance Problem in Detail:**

```
Monolithic (one context switch):
  App ──syscall──► Kernel FS code ──► return to App

Microkernel (multiple context switches):
  App ──msg──► µKernel ──msg──► FS Server ──msg──► µKernel
       ──msg──► Disk Driver ──msg──► µKernel ──msg──► App
```

### 3. Hybrid Kernel

**Compromise approach.** Start with a microkernel philosophy but move performance-critical services (like the file system and some drivers) back into kernel space. Keep the modularity where you can, but avoid the IPC overhead for hot paths.

```
┌──────────────────────────────────────────┐
│              User Space                  │
│  [App A]  [App B]  [Some    ] [Print  ]  │
│                    [Drivers ] [Server ]  │
├──────────────────────────────────────────┤
│              Kernel Space                │
│  ┌────────────────────────────────────┐  │
│  │  IPC    Scheduling    Memory Mgmt │  │
│  │                                    │  │
│  │  File System    Network Stack     │  │
│  │  Core Drivers   (performance-     │  │
│  │                  critical parts)  │  │
│  └────────────────────────────────────┘  │
├──────────────────────────────────────────┤
│               Hardware                   │
└──────────────────────────────────────────┘
```

**Examples:** Windows NT/10/11, macOS (XNU = Mach microkernel + BSD monolithic layer), ReactOS

**Pros:**
- Better performance than pure microkernel
- More modular/reliable than pure monolithic
- Pragmatic balance

**Cons:**
- Complexity of deciding what goes in kernel vs user space
- Still vulnerable to kernel-space driver crashes
- "Worst of both worlds" criticism from purists

## Comparison Table

| Aspect | Monolithic | Microkernel | Hybrid |
|--------|-----------|-------------|--------|
| **Kernel size** | Large (millions of LOC) | Small (thousands of LOC) | Medium |
| **Performance** | Best (direct function calls) | Worst (IPC overhead) | Good |
| **Reliability** | One bug crashes all | Services can restart | Mixed |
| **Security** | Large attack surface | Small, verifiable | Medium |
| **Examples** | Linux, Unix, FreeBSD | MINIX 3, QNX, seL4 | Windows NT, macOS |
| **Used in practice** | Servers, embedded | Safety-critical, aerospace | Desktop, general |

## The Real-World Answer

In practice, the lines are blurred:

- **Linux** is monolithic but uses loadable modules, making it somewhat modular.
- **Windows** is hybrid but has most drivers in kernel space, making it somewhat monolithic.
- **macOS** combines Mach (microkernel) and BSD (monolithic) components in a single kernel space.

The "right" architecture depends on your priorities:
- **Need maximum performance?** → Monolithic (Linux for servers)
- **Need maximum reliability/safety?** → Microkernel (QNX for cars, seL4 for military)
- **Need a pragmatic balance?** → Hybrid (Windows, macOS for desktops)

## Interview Angle

**Q: Compare monolithic and microkernel architectures.**
> In a monolithic kernel, all OS services run in kernel space as a single binary — fast (direct function calls) but a bug anywhere can crash the whole system. In a microkernel, only IPC, scheduling, and basic memory management are in kernel space; everything else runs as user-space servers — more reliable and secure, but slower due to IPC overhead. Linux is monolithic; QNX and MINIX are microkernels. Most real systems (Windows, macOS) use a hybrid approach.

**Q: Why does Linux use a monolithic kernel?**
> Performance. Linus Torvalds argued that the IPC overhead of microkernels was unacceptable for a general-purpose OS. Linux mitigates the downsides of monolithic design with loadable kernel modules (add/remove drivers at runtime) and robust coding practices.

---

**Next:** [04-user-mode-vs-kernel-mode.md](./04-user-mode-vs-kernel-mode.md) — The most fundamental protection boundary in any OS.
