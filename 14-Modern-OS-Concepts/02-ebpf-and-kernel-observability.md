# eBPF and Kernel Observability

For decades, if you wanted to add logic to the Linux kernel, you had two choices: modify the kernel source and recompile, or write a kernel module. Both are dangerous -- a bug can crash the entire machine. eBPF changes this equation entirely. It lets you run sandboxed programs **inside** the kernel, with a verifier that guarantees safety, without modifying kernel code or loading risky modules. This is arguably the most significant change to Linux kernel programmability in the last decade.

---

## What Is eBPF?

**eBPF** (extended Berkeley Packet Filter) is a technology that lets you run small, verified programs inside the Linux kernel in response to events -- without changing kernel source code.

The name is historical. The original BPF (1993) was a packet filter for `tcpdump`. "extended" BPF evolved it into a general-purpose in-kernel virtual machine. Today, eBPF is used for far more than packet filtering: observability, networking, security, and profiling.

Think of eBPF as **JavaScript for the kernel**. Just as JavaScript lets you run sandboxed code in the browser without modifying the browser's source code, eBPF lets you run sandboxed code in the kernel without modifying the kernel's source code.

---

## How eBPF Works

```
+--------------------+     +---------------------+     +------------------+
| 1. Write Program   |     | 2. Compile to       |     | 3. Load into     |
| (C, Rust)          +---->+    eBPF Bytecode     +---->+    Kernel        |
|                    |     | (clang/LLVM)         |     | (bpf() syscall)  |
+--------------------+     +---------------------+     +--------+---------+
                                                                |
                           +---------------------+              |
                           | 4. Verifier Checks   |<-------------+
                           | - No infinite loops  |
                           | - Bounded memory     |
                           | - Valid memory access |
                           | - No kernel crash    |
                           +--------+------------+
                                    |
                                    | PASS
                                    v
                           +---------------------+     +------------------+
                           | 5. JIT Compile       |     | 6. Attach to     |
                           |    to Native Code    +---->+    Hook Point     |
                           | (x86, ARM, etc.)     |     | (syscall, XDP,   |
                           +---------------------+     |  tracepoint...)  |
                                                        +--------+---------+
                                                                 |
                                                                 v
                                                        +------------------+
                                                        | 7. Runs on Each  |
                                                        |    Event         |
                                                        | (collect data,   |
                                                        |  filter packets, |
                                                        |  enforce policy) |
                                                        +------------------+
```

**eBPF bytecode** is a platform-independent instruction set (similar to JVM bytecode) designed for safe in-kernel execution. Programs are compiled from C (via Clang/LLVM) into this bytecode, which the kernel's JIT compiler then translates to native machine instructions for near-native performance. The bytecode format enables the verifier to analyze the program before it runs.

### The Verifier: Why eBPF Is Safe

The kernel verifier is the key innovation. Before any eBPF program runs, the verifier statically analyzes it to guarantee:

- **No infinite loops**: programs must terminate (bounded loop iterations)
- **No invalid memory access**: every memory read/write is bounds-checked
- **No kernel crash**: the program cannot dereference null pointers or corrupt kernel state
- **Bounded complexity**: programs have a maximum instruction count (currently ~1 million verified instructions)

The verifier performs **static analysis** by walking every possible execution path. It tracks register states, ensures all memory accesses are within bounds, verifies that pointers aren't leaked, confirms loops are bounded (programs must terminate), and checks that only permitted kernel functions (helpers) are called. Programs that fail verification are rejected before they ever execute.

This is what makes eBPF fundamentally safer than kernel modules.

---

## Hook Points: Where eBPF Programs Attach

eBPF programs do not run in isolation. They attach to **hook points** -- specific events in the kernel where the program fires.

```
User Space
+------------------------------------------------------------+
|  Application (e.g., web server)                            |
|       |                                                    |
|       | syscall: read(), write(), open(), connect()        |
+-------+----------------------------------------------------+
        |
========|====================================================
        v                    KERNEL
+-------+----------------------------------------------------+
|                                                            |
|  Syscall Entry  <--- [eBPF: trace syscalls]                |
|       |                                                    |
|       v                                                    |
|  VFS Layer      <--- [eBPF: trace file operations]         |
|       |                                                    |
|       v                                                    |
|  Network Stack  <--- [eBPF: filter/modify packets]         |
|       |                                                    |
|       v                                                    |
|  TCP/IP         <--- [eBPF: trace connections, RTT]        |
|       |                                                    |
|       v                                                    |
|  Device Driver  <--- [eBPF: XDP - process at NIC level]    |
|       |                                                    |
|  Scheduler      <--- [eBPF: custom scheduling logic]       |
|                                                            |
|  LSM Hooks      <--- [eBPF: security policy enforcement]   |
|                                                            |
|  cgroup Hooks   <--- [eBPF: per-container policy]          |
+------------------------------------------------------------+
```

| Hook Point | What It Catches | Example Use |
|---|---|---|
| **kprobes** | Any kernel function entry/exit | Trace `tcp_connect()` to see all outbound connections |
| **uprobes** | Any userspace function entry/exit | Trace `malloc()` calls in your application |

**kprobes** (kernel probes) let you attach eBPF programs to virtually any kernel function entry/exit point — like placing a wiretap on internal kernel calls. **uprobes** (user probes) do the same for user-space functions. Together, they enable dynamic tracing without modifying source code or recompiling — you can observe function arguments, return values, and timing in production.

| **tracepoints** | Predefined stable kernel events | Trace `sched:sched_switch` for context switch analysis |
| **syscalls** | System call entry/exit | Audit which processes call `open()` on sensitive files |
| **XDP** (eXpress Data Path) | Packet arrival at NIC, before kernel stack | DDoS mitigation, load balancing at line rate |
| **TC** (Traffic Control) | Packet in/out of network stack | Packet manipulation, container networking |
| **LSM** (Linux Security Modules) | Security-relevant operations | Runtime security policy enforcement |

**LSM (Linux Security Modules)** is a framework of hook points placed at security-critical operations in the kernel (opening files, creating sockets, loading programs). SELinux and AppArmor implement their policies through LSM hooks. With eBPF attached to LSM hooks, you can write custom security policies that run in the kernel without building a full LSM module.
| **cgroup** | Per-cgroup events | Container-level network/resource policy |

---

## What eBPF Enables

### 1. Observability Without Kernel Modification

Traditional observability tools (strace, perf) have significant overhead. eBPF programs run inside the kernel and can aggregate data before sending it to user space, drastically reducing overhead.

```
Traditional strace:                    eBPF-based tracing:

User Space    Kernel                   User Space    Kernel
+------+    +--------+                +------+    +--------+
| tool |<===| EVERY  |                | tool |<---| eBPF   |
|      |    | syscall|                |      |    | (only  |
|      |    | copied |                |      |    | sends  |
|      |    | to user|                |      |    | summary|
|      |    | space  |                |      |    | data)  |
+------+    +--------+                +------+    +--------+

Overhead: 10-100x slowdown            Overhead: <5% typically
```

### 2. Networking: XDP and Cilium

**XDP** (eXpress Data Path) processes packets at the earliest possible point -- right when they arrive at the NIC, before the kernel allocates an `sk_buff`. This enables:

- DDoS mitigation at millions of packets per second
- Load balancing without leaving the kernel
- Packet filtering before any kernel overhead

**Cilium** uses eBPF to replace `iptables` for Kubernetes networking. Instead of long chains of iptables rules (which scale poorly), Cilium installs eBPF programs that make routing decisions in O(1) using eBPF maps.

### 3. Security: Runtime Enforcement

eBPF programs attached to LSM hooks can enforce security policy at runtime:

- Block processes from executing specific binaries
- Prevent containers from making unexpected network connections
- Detect and alert on suspicious syscall patterns

### 4. Continuous Profiling

eBPF can sample CPU stacks, memory allocations, and off-CPU time in production with minimal overhead -- enabling always-on profiling that was previously too expensive.

---

## eBPF Tools Ecosystem

| Tool | What It Does | How It Uses eBPF |
|---|---|---|
| **bcc** | Python + C toolkit for writing eBPF programs | Provides pre-built tools: `execsnoop`, `opensnoop`, `tcplife`, `biolatency` |
| **bpftrace** | High-level tracing language | One-liners for kernel tracing: `bpftrace -e 'tracepoint:syscalls:sys_enter_open { printf("%s\n", comm); }'` |
| **Cilium** | Kubernetes CNI and network policy | Replaces iptables with eBPF for pod networking and security |
| **Falco** | Runtime security | Detects suspicious behavior (unexpected shell in container, sensitive file access) |
| **Pixie** | Kubernetes observability | Auto-instruments HTTP, gRPC, DNS, database queries without code changes |
| **Tetragon** | Security observability and enforcement | Traces process execution, file access, network activity with eBPF |

### bpftrace One-Liners (Examples of Power)

```bash
# Count syscalls by process name
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Trace new TCP connections with details
bpftrace -e 'kprobe:tcp_connect { printf("%s -> %s\n", comm, ntop(((struct sock *)arg0)->sk_daddr)); }'

# Histogram of read() sizes
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args->ret > 0/ { @bytes = hist(args->ret); }'

# Trace file opens by PID
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%d %s %s\n", pid, comm, str(args->filename)); }'
```

---

## eBPF vs Alternatives

| Aspect | eBPF | Kernel Modules | Tracepoints (ftrace) | strace |
|---|---|---|---|---|
| **Safety** | Verifier guarantees no crash | Can crash kernel | Safe (read-only) | Safe (ptrace) |
| **Performance** | Near-native (JIT) | Native | Low overhead | High overhead (10-100x) |
| **Programmability** | Arbitrary logic | Arbitrary logic | Predefined events only | Syscalls only |
| **Hot-loadable** | Yes (no reboot) | Yes (but risky) | Always available | Always available |
| **Requires root** | Yes (or CAP_BPF) | Yes | Yes | Yes (or CAP_SYS_PTRACE) |
| **Kernel version** | 4.x+ (5.x+ for full features) | Any | Any | Any |
| **Can modify behavior** | Yes (XDP, TC, LSM) | Yes | No (observe only) | No (observe only) |
| **Scope** | Kernel + userspace | Kernel only | Kernel only | Syscalls only |

The key tradeoff: eBPF gives you almost the power of kernel modules with the safety guarantees of tracepoints.

---

## eBPF Maps: Sharing Data

eBPF programs communicate with user space and with each other through **maps** -- key-value data structures stored in kernel memory.

```
User Space                          Kernel
+-----------+                       +------------------+
|           |   bpf() syscall       |                  |
| Dashboard |<---read/write-------->| eBPF Map         |
| CLI tool  |                       | (hash, array,    |
| Prometheus|                       |  ringbuf, etc.)  |
|           |                       |                  |
+-----------+                       +--------+---------+
                                             ^
                                             |
                                    +--------+---------+
                                    | eBPF Program     |
                                    | (attached to     |
                                    |  hook point)     |
                                    +------------------+
```

Map types include hash maps, arrays, ring buffers, LRU caches, per-CPU arrays, and more. This is how an eBPF program counting syscalls (running in kernel) can expose that data to a Prometheus exporter (running in user space).

---

## Real-World Connection

**Netflix**: Uses eBPF extensively for performance analysis and troubleshooting in production. Brendan Gregg (formerly at Netflix) created many of the foundational eBPF tracing tools and methodologies.

**Meta (Facebook)**: Uses eBPF for load balancing (Katran), replacing dedicated hardware load balancers with XDP-based software load balancers that process millions of packets per second.

**Cloudflare**: Uses XDP-based eBPF programs for DDoS mitigation, dropping malicious packets before they reach the kernel network stack.

**Google**: Uses eBPF for fleet-wide observability and security monitoring across millions of machines.

**Cilium in Kubernetes**: Increasingly the default CNI for Kubernetes clusters, replacing iptables (which does not scale well beyond thousands of rules) with eBPF programs that make O(1) routing decisions.

---

## Interview Angle

**Q: What is eBPF and why is it significant for production observability?**

A: eBPF lets you run sandboxed programs inside the Linux kernel without modifying kernel source code or loading kernel modules. A verifier statically checks every program to guarantee it cannot crash the kernel, and a JIT compiler ensures near-native performance. You write a program, attach it to a hook point (syscall entry, tracepoint, network event, etc.), and it fires on each event. For observability, this means you can trace syscalls, function calls, network connections, and file operations in production with minimal overhead (typically under 5%) -- unlike strace which can slow a process by 10-100x. The program aggregates data inside the kernel (using eBPF maps) and sends only summaries to user space, reducing the data volume dramatically. Tools like bpftrace, Cilium, Falco, and Pixie all build on eBPF. It has effectively become the standard mechanism for deep Linux observability and programmable networking.

**Q: How does the eBPF verifier make it safer than kernel modules?**

A: Kernel modules run with full kernel privileges -- a bug can dereference a null pointer, corrupt memory, or enter an infinite loop, crashing the entire machine. The eBPF verifier statically analyzes the program before it runs and rejects anything that could be unsafe: no unbounded loops, no out-of-bounds memory access, no null dereferences, bounded instruction count. If the verifier rejects it, it never executes. This means eBPF programs are provably safe in ways that kernel modules can never be. You get most of the programmability of kernel modules with the safety of a sandboxed environment.

**Q: What is XDP and why is it used for DDoS mitigation?**

A: XDP (eXpress Data Path) is the earliest hook point for eBPF in the network path -- it fires when a packet arrives at the NIC, before the kernel allocates any data structures for it. This means you can drop malicious packets before they consume kernel resources (no sk_buff allocation, no TCP stack processing, no socket lookup). At this level, a single server can process and filter millions of packets per second. Companies like Cloudflare use XDP to drop DDoS traffic at the edge with software, replacing or supplementing expensive hardware solutions.

---

Next: [Unikernels and Library OS](03-unikernels-and-library-os.md)
