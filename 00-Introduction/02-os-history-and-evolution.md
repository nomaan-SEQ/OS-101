# OS History and Evolution

## Why History Matters

You don't need to memorize dates. But understanding *why* each generation of OS was built the way it was gives you the **reasoning behind modern design decisions**. Every feature in today's OS exists because an older approach had a specific limitation.

## The Timeline

```
1950s          1960s          1970s          1980s          1990s-2000s      2010s+
  │              │              │              │              │                │
Batch     Multiprogramming   Time-sharing   Personal       Distributed      Cloud &
Systems   & Spooling         & Unix         Computers      & Networked      Containers
  │              │              │              │              │                │
  │              │              │              │              │                │
One job    Multiple jobs    Multiple users  One user,      Many machines,   VMs,
at a time  in memory        at once         GUI focus      one OS feel      microservices
```

## Generation by Generation

### 1. No OS (1940s–1950s)
**What:** Programmers had direct access to the hardware. Programs were loaded via punch cards or switches. One program ran at a time. When it finished, the next was manually loaded.

**The problem:** The CPU sat idle while humans loaded programs. Expensive hardware was wasted waiting for slow humans.

### 2. Batch Systems (1950s–1960s)
**What:** An operator collected jobs (programs), batched them together, and fed them to the computer. A simple **resident monitor** loaded and ran jobs one after another automatically.

**Key innovation:** No human intervention between jobs → better hardware utilization.

**The problem:** While a program waited for I/O (reading from tape), the CPU sat completely idle. One slow I/O operation blocked everything.

```
┌──────────────────────────────┐
│     Simple Batch System      │
│                              │
│  Job 1 ──► Job 2 ──► Job 3  │
│                              │
│  CPU idle during I/O waits   │
└──────────────────────────────┘
```

### 3. Multiprogramming (1960s)
**What:** Multiple jobs loaded into memory simultaneously. When one job waited for I/O, the CPU switched to another job.

**Key innovation:** **CPU never sits idle** if there's work to do. This is the birth of process management and scheduling.

**The problem:** All jobs ran to completion without interruption. No interactivity. If you submitted a job, you waited hours for results.

```
┌───────────────────────────────────────┐
│         Multiprogramming              │
│                                       │
│  Memory:  [Job A] [Job B] [Job C]     │
│                                       │
│  CPU:  A runs → A waits for I/O       │
│              → B runs → B waits       │
│                    → C runs → ...     │
│                                       │
│  CPU utilization: much higher!        │
└───────────────────────────────────────┘
```

### 4. Time-Sharing (1960s–1970s)
**What:** The CPU rapidly switches between jobs using **time slices** (e.g., 100ms each). Each user connected via a terminal gets the illusion of having the whole computer.

**Key innovation:** **Interactive computing.** Users can type a command and get a response in seconds, not hours.

**Systems:** CTSS (1961), Multics (1964), **Unix (1969)**.

**The problem:** More users = more overhead. Early systems struggled with memory protection between users.

```
┌──────────────────────────────────────────┐
│            Time-Sharing                  │
│                                          │
│  Time: ──┬──┬──┬──┬──┬──┬──┬──►         │
│          │A │B │C │A │B │C │A │          │
│          └──┴──┴──┴──┴──┴──┴──┘          │
│                                          │
│  Each user gets a small time slice       │
│  Feels like everyone has their own CPU   │
└──────────────────────────────────────────┘
```

### 5. Personal Computers (1980s)
**What:** Computers became cheap enough for one person. OSes shifted focus from multi-user efficiency to **single-user convenience** — GUIs, ease of use.

**Systems:** MS-DOS (1981), Mac OS (1984), Windows (1985), OS/2.

**Key shift:** Protection and multi-user support became less important (at first). Ease of use mattered more. This is why early Windows had almost no memory protection — it was designed for one user.

### 6. Distributed and Networked Systems (1990s–2000s)
**What:** Multiple computers connected via networks, working together. The OS had to manage not just local resources but **network communication, remote file systems, and distributed processes**.

**Systems:** Windows NT, Linux, Solaris, Plan 9.

**Key innovations:** TCP/IP networking built into the kernel, network file systems (NFS), client-server architecture.

### 7. Modern Era: Virtualization, Cloud, and Containers (2010s+)
**What:** Hardware is powerful enough to run multiple OSes on a single machine (VMs). Containers provide lighter-weight isolation. The OS boundary is blurred — is Kubernetes an OS?

**Systems:** Linux dominates servers, Windows dominates desktops, Android/iOS dominate mobile.

**Key innovations:** Hypervisors (Xen, KVM), containers (Docker, containerd), orchestration (Kubernetes), eBPF for kernel programmability.

## The Pattern

Every generation follows the same cycle:

```
Hardware gets cheaper/faster
        │
        ▼
New usage patterns emerge
(more users, more programs, more machines)
        │
        ▼
Old OS design can't handle it
        │
        ▼
New OS abstractions are invented
(multiprogramming, time-sharing, virtual memory, containers)
        │
        ▼
Repeat
```

## What to Take Away

| Era | Key Innovation | Still Relevant Today? |
|-----|----------------|----------------------|
| Batch | Automated job execution | Yes — batch processing, cron jobs, CI/CD pipelines |
| Multiprogramming | Multiple jobs in memory, CPU switching | Yes — this IS modern process management |
| Time-sharing | Interactive computing, time slices | Yes — every multi-user system, every terminal session |
| Personal | GUI, single-user optimization | Yes — desktop UX, Windows/macOS design philosophy |
| Distributed | Networking in the OS, remote resources | Yes — everything is networked now |
| Modern | Virtualization, containers | Yes — cloud computing runs on this |

## Interview Angle

**Q: Why was multiprogramming invented?**
> To improve CPU utilization. In batch systems, the CPU sat idle during I/O operations. Multiprogramming keeps multiple jobs in memory so the CPU can switch to another job when one is waiting for I/O.

**Q: What's the difference between multiprogramming and time-sharing?**
> Multiprogramming switches jobs only when the current job blocks (e.g., on I/O). Time-sharing forces switches at regular intervals (time slices) to give each user interactive response times. Time-sharing is multiprogramming + preemption.

---

**Next:** [03-os-architecture-monolithic-micro-hybrid.md](./03-os-architecture-monolithic-micro-hybrid.md) — Now that you know the history, let's look at how an OS kernel can be structured.
