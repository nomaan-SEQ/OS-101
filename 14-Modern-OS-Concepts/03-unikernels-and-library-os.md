# Unikernels and Library OS

Every time your web server handles a request, it runs on top of a general-purpose operating system that includes USB drivers, Bluetooth stacks, sound subsystems, and hundreds of other components your server will never use. This is the cost of generality. Unikernels ask a radical question: what if you compiled only the OS components your application actually needs, directly into the application itself, and ran that single image on a hypervisor?

---

## The Problem with General-Purpose OS

A typical Linux server installation includes millions of lines of kernel code. Your application uses a fraction of it.

```
Traditional VM Stack:
+----------------------------------+
|         Application              |  <-- Your code
+----------------------------------+
|   Runtime (libc, language VM)    |
+----------------------------------+
|                                  |
|     Full Linux Kernel            |  <-- Millions of lines
|   (drivers, schedulers, FS,     |      Most never executed
|    networking, crypto, USB,      |      by your application
|    Bluetooth, sound, ...)        |
|                                  |
+----------------------------------+
|          Hypervisor              |
+----------------------------------+
|          Hardware                |
+----------------------------------+

Image size: 500 MB - several GB
Boot time: 10-60 seconds
Attack surface: Large (shell, utilities, unused services)
```

Every line of code that runs in your VM but is never used by your application is:
- **Wasted memory** (loaded but unused)
- **Wasted boot time** (initialized but unneeded)
- **Attack surface** (exploitable but unnecessary)

---

## Library OS: Link Only What You Need

A **Library OS** treats OS functionality as a library. Instead of running on top of a separate kernel, you link the OS components your application needs directly into your application binary.

```
Traditional OS:                    Library OS:

App calls read()                   App calls read()
    |                                  |
    v                                  v
  syscall                           Function call (no mode switch)
    |                                  |
    v                                  v
  Kernel VFS                        Linked FS library
    |                                  |
    v                                  v
  Kernel block layer                Linked block library
    |                                  |
    v                                  v
  Kernel driver                     Linked driver library
    |                                  |
    v                                  v
  Hardware                          Hardware (via hypervisor)
```

Key insight: there is no user/kernel boundary. No syscall overhead. No mode switch. The application and the OS code run in the same address space, at the same privilege level.

Historical note: this is not new. Exokernel (MIT, 1995) and Nemesis (Cambridge, 1994) explored these ideas. Unikernels are the modern realization, made practical by ubiquitous hypervisors.

---

## Unikernels: The Full Picture

A **unikernel** takes the Library OS idea and produces a single-purpose machine image that runs directly on a hypervisor.

> **Analogy:** A unikernel is like a food truck instead of a restaurant. A restaurant (traditional OS) has separate areas — kitchen, dining room, storage, offices — with walls and doors between them (privilege boundaries). A food truck strips all that away: one person, one open space, everything within arm's reach. It's extremely efficient for serving one menu, but you can't add a second chef or serve a different cuisine without building a whole new truck.

```
Unikernel Stack:
+----------------------------------+
|                                  |
|   Application + OS Libraries     |  <-- Single binary
|   (your code + TCP/IP +          |      Single address space
|    minimal FS + crypto +         |      Single process
|    TLS + whatever you need)      |      No shell, no login
|                                  |
+----------------------------------+
|          Hypervisor              |  <-- Xen, KVM, etc.
+----------------------------------+
|          Hardware                |
+----------------------------------+

Image size: 100 KB - few MB
Boot time: Milliseconds
Attack surface: Minimal
```

### Properties of a Unikernel

- **Single address space**: no user/kernel split, no page table switches
- **Single process**: no process scheduler needed
- **No shell**: you cannot SSH into a unikernel
- **No unused code**: only linked libraries are included
- **Immutable**: to change anything, you recompile and redeploy

---

## Advantages and Disadvantages

| Advantage | Why It Matters |
|---|---|
| **Tiny image size** | KBs to few MBs vs GBs for a VM. Faster storage, faster transfer, faster deployment. |
| **Fast boot** | Milliseconds vs seconds/minutes. Enables instant scaling and serverless cold starts. |
| **Minimal attack surface** | No shell, no unused services, no package manager. Nothing to exploit that your app does not use. |
| **No context switch overhead** | Single address space means no user/kernel mode switches. Function calls instead of syscalls. |
| **Strong isolation** | Runs as a VM on a hypervisor -- hardware-level isolation, not just namespace isolation. |

| Disadvantage | Why It Is Hard |
|---|---|
| **Difficult to debug** | No shell, no `gdb`, no `strace`, no `top`. You cannot SSH in and poke around. |
| **Must recompile for any change** | Configuration change? Recompile. Library update? Recompile. Security patch? Recompile and redeploy. |
| **Limited language support** | Most unikernel frameworks support only specific languages (MirageOS = OCaml, IncludeOS = C++). |
| **No multi-process** | Cannot fork, cannot run background daemons, cannot pipe between processes. |
| **Small ecosystem** | Few libraries, few tools, limited community compared to Linux. |
| **Debugging in production** | When something goes wrong, you have very limited introspection. |

---

## Unikernel Implementations

| Project | Language | Hypervisor | Notable Feature |
|---|---|---|---|
| **MirageOS** | OCaml | Xen, Solo5 | Type-safe, pioneered the modern unikernel concept |
| **Unikraft** | C, C++, Go, Rust, Python | KVM, Xen, Firecracker | Modular, POSIX-compatible, most practical for adoption |
| **IncludeOS** | C++ | KVM, VirtualBox | Minimal, fast boot, designed for C++ services |
| **OSv** | Java, C, Node.js, Ruby | KVM, Xen, VirtualBox | Runs unmodified Linux applications (partial POSIX) |
| **Nanos** | Any (runs ELF binaries) | KVM, Xen, cloud | Runs existing Linux binaries as unikernels |

---

## Unikernel vs Container vs VM

| Aspect | VM (Linux) | Container (Docker) | Unikernel |
|---|---|---|---|
| **Isolation** | Hardware (hypervisor) | OS-level (namespaces) | Hardware (hypervisor) |
| **Image size** | GBs | 10s-100s MB | KBs - few MBs |
| **Boot time** | 10-60 seconds | ~1 second | Milliseconds |
| **Attack surface** | Full OS (large) | Shared kernel (medium) | Minimal (small) |
| **Debugging** | Full tools (SSH, gdb, strace) | Full tools (exec into container) | Very limited |
| **Multi-process** | Yes | Yes | No |
| **Ecosystem** | Massive | Large and growing | Small |
| **Dev experience** | Familiar | Familiar | Requires learning |
| **Kernel sharing** | No (own kernel) | Yes (shared host kernel) | No (library kernel) |
| **Density** | Low (10s per host) | High (100s per host) | Very high (1000s per host) |

```
Isolation Strength vs Development Ease:

  Strong +  VM        Unikernel
Isolation |   \         /
          |    \       /
          |     \     /
          |    Container
          |
  Weak    +---------------------------+
          Easy                    Hard
              Development Ease
```

---

## Why Unikernels Have Not Gone Mainstream

Despite compelling advantages, unikernels remain niche. The reasons are instructive:

1. **Development friction**: building, testing, and iterating on a unikernel is harder than `docker build`. Developers optimize for feedback speed.

2. **Debugging is painful**: when production breaks, engineers need to inspect state. Without a shell, logs, or standard tools, troubleshooting is difficult.

3. **Containers are "good enough"**: containers provide a pragmatic middle ground -- reasonable isolation, familiar tools, fast enough startup, huge ecosystem. For most use cases, the marginal benefit of unikernels does not justify the cost.

4. **Organizational inertia**: teams have invested in container tooling, CI/CD pipelines, monitoring stacks. Switching to unikernels requires rethinking the entire deployment story.

5. **Wasm is emerging**: WebAssembly runtimes offer some of the same benefits (small, fast startup, sandboxed) with a better development experience. More on this in the next section.

---

## Real-World Connection

**Network Functions Virtualization (NFV)**: Unikernels are best suited for running network functions -- firewalls, load balancers, routers -- where you need minimal latency, fast boot, and the workload is well-defined. Telecom companies experiment with unikernels for this.

**Serverless platforms**: The millisecond boot time makes unikernels attractive for serverless. If a cold start takes 5ms instead of 500ms, the "cold start problem" mostly disappears. AWS Firecracker (used by Lambda) is not a unikernel, but it borrows the idea of minimal, purpose-built VMs.

**IoT and edge**: Resource-constrained devices benefit from tiny images and minimal attack surface. A unikernel running a sensor data collector on an edge device makes sense where a full Linux stack does not.

**Security-critical applications**: When you need the strongest isolation with the smallest attack surface, unikernels are compelling. The fewer lines of code running, the fewer potential vulnerabilities.

---

## Interview Angle

**Q: What is a unikernel and how does it differ from a container?**

A: A unikernel is a single-purpose machine image where the application and only the OS components it needs are compiled together into one binary that runs directly on a hypervisor. There is no separate kernel, no user/kernel boundary, no shell, no multi-process support. A container, by contrast, is a process (or group of processes) on a shared Linux kernel, isolated by namespaces and cgroups. The key differences: unikernels have hardware-level isolation (hypervisor) vs OS-level isolation (namespaces), much smaller images (KBs vs MBs), faster boot (milliseconds vs seconds), but much worse debugging experience (no shell, no standard tools) and no multi-process support.

**Q: Why have unikernels not replaced containers despite their advantages?**

A: Development experience. Containers let you `docker exec` into a running container, attach a debugger, inspect logs, and iterate quickly. Unikernels require a full recompile for any change and offer almost no runtime introspection. The container ecosystem (Docker, Kubernetes, monitoring tools, CI/CD pipelines) is massive and mature. The benefits of unikernels (smaller image, faster boot, smaller attack surface) are real but marginal for most workloads compared to the cost of adopting an entirely different development and operational model. Containers hit a practical sweet spot.

**Q: Where might unikernels make sense despite the tradeoffs?**

A: Three scenarios: (1) Network functions (firewalls, load balancers) where the workload is well-defined, latency matters, and you do not need to debug interactively. (2) Serverless platforms where millisecond cold starts eliminate the cold start problem. (3) Security-critical edge deployments where minimal attack surface is paramount and the device runs a single, well-defined function.

---

Next: [Wasm Runtimes and the OS Boundary](04-wasm-runtimes-and-os-boundary.md)
