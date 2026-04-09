# OS-Level Virtualization: Namespaces and Cgroups

## The Lighter Alternative

Virtual machines are powerful but heavy. Each VM runs a complete operating system — its own kernel, init system, drivers, libraries. If you have 50 microservices, that's 50 kernels loaded in memory, 50 sets of OS processes, and 50 boot cycles.

OS-level virtualization takes a different approach: **share the kernel, isolate everything else**. Instead of virtualizing hardware, you virtualize the OS itself. A process gets its own view of the system — its own process tree, its own network stack, its own filesystem — while sharing the same kernel with every other process on the machine.

This is the foundation of containers. And it's built on two Linux kernel features: **namespaces** and **cgroups**.

```
VM vs. OS-Level Virtualization:

Virtual Machines:                    OS-Level (Containers):
┌────────┐ ┌────────┐ ┌────────┐    ┌────────┐ ┌────────┐ ┌────────┐
│ App A  │ │ App B  │ │ App C  │    │ App A  │ │ App B  │ │ App C  │
├────────┤ ├────────┤ ├────────┤    ├────────┤ ├────────┤ ├────────┤
│ Libs   │ │ Libs   │ │ Libs   │    │ Libs   │ │ Libs   │ │ Libs   │
├────────┤ ├────────┤ ├────────┤    ├────────┴─┴────────┴─┴────────┤
│Kernel A│ │Kernel B│ │Kernel C│    │       Shared Host Kernel     │
├────────┴─┴────────┴─┴────────┤    ├──────────────────────────────┤
│        Hypervisor            │    │          Hardware            │
├──────────────────────────────┤    └──────────────────────────────┘
│        Hardware              │
└──────────────────────────────┘     One kernel, isolated user spaces
 Three full kernels in memory
```

## Linux Namespaces: Isolate What a Process Can SEE

A **namespace** wraps a global system resource so that processes inside the namespace see their own isolated instance of that resource. A process in a PID namespace sees its own process tree. A process in a network namespace sees its own network interfaces. From inside the namespace, you can't see or affect resources in other namespaces.

### The Seven Namespaces

| Namespace | Flag | What It Isolates | Kernel Version |
|---|---|---|---|
| **Mount** (mnt) | `CLONE_NEWNS` | Filesystem mount points — each namespace has its own mount tree | 2.4.19 (2002) |
| **UTS** | `CLONE_NEWUTS` | Hostname and domain name | 2.6.19 (2006) |
| **IPC** | `CLONE_NEWIPC` | System V IPC, POSIX message queues, shared memory | 2.6.19 (2006) |
| **PID** | `CLONE_NEWPID` | Process IDs — process sees its own PID tree starting from 1 | 2.6.24 (2008) |
| **Network** (net) | `CLONE_NEWNET` | Network stack: interfaces, routing tables, iptables rules, sockets | 2.6.29 (2009) |
| **User** | `CLONE_NEWUSER` | User and group IDs — UID 0 inside maps to unprivileged UID outside | 3.8 (2013) |
| **Cgroup** | `CLONE_NEWCGROUP` | View of the cgroup hierarchy | 4.6 (2016) |

### PID Namespace: Isolated Process Trees

The PID namespace is the most intuitive example. Inside a PID namespace, the first process gets PID 1. It can only see processes in its own namespace.

```
Host PID Namespace:
┌──────────────────────────────────────────────────┐
│  PID 1 (systemd)                                 │
│  ├── PID 452 (sshd)                              │
│  ├── PID 1023 (dockerd)                          │
│  │   ├── PID 1501 (containerd)                   │
│  │   │   ├── PID 2001 ◄─── Container A's init    │
│  │   │   │   └── PID 2045     (sees PID 1)       │
│  │   │   ├── PID 3001 ◄─── Container B's init    │
│  │   │   │   └── PID 3022     (sees PID 1)       │
└──┴───┴───┴───────────────────────────────────────┘

Container A sees:           Container B sees:
┌────────────────┐          ┌────────────────┐
│ PID 1 (nginx)  │          │ PID 1 (redis)  │
│ └── PID 2      │          │ └── PID 2      │
└────────────────┘          └────────────────┘

Same physical host, completely different PID views.
Host sees ALL processes. Containers see only their own.
```

The host can see all processes with their real PIDs (2001, 3001). But inside container A, the nginx process believes it's PID 1. It has no way to see or signal processes in container B.

### Network Namespace: Isolated Network Stacks

Each network namespace gets its own complete network stack:

```
Host Network Namespace:              Container Network Namespace:
┌──────────────────────┐            ┌──────────────────────┐
│ eth0: 10.0.0.5       │            │ eth0: 172.17.0.2     │
│ docker0: 172.17.0.1  │◄──veth──► │                      │
│ lo: 127.0.0.1        │  pair     │ lo: 127.0.0.1        │
│                      │            │                      │
│ Routing table: ...   │            │ Routing table: ...   │
│ iptables rules: ...  │            │ iptables rules: ...  │
└──────────────────────┘            └──────────────────────┘
```

A **veth pair** connects the two namespaces — it's like a virtual ethernet cable with one end in each namespace. The container's `eth0` is one end of the pair; the other end is attached to the `docker0` bridge on the host.

### User Namespace: Safe Root

The user namespace maps UIDs between namespaces. A process can be UID 0 (root) inside its namespace while being an unprivileged user (say, UID 100000) on the host. This means:

- Inside the container: root can install packages, bind to port 80, modify files
- On the host: the process has no special privileges — it can't escape the container to damage the host

This is critical for security. Without user namespaces, a container escape means the attacker has root on the host.

### Mount Namespace: Isolated Filesystem

Each container gets its own mount tree. The container's root filesystem (`/`) is actually a specific directory on the host, made to look like a full filesystem using `pivot_root` or `chroot`:

```
Host filesystem:                    Container sees:
/                                   /
├── bin/                            ├── bin/     (from image)
├── etc/                            ├── etc/     (from image)
├── var/                            ├── var/
│   └── lib/docker/                 │   └── log/ (container's own)
│       └── overlay2/               ├── tmp/     (container's own)
│           └── <container-id>/     └── app/     (application files)
│               └── merged/  ◄──────── This IS the container's "/"
└── home/                           Host's /home? Doesn't exist
                                    in this namespace.
```

## Control Groups (cgroups): Limit What a Process Can USE

Namespaces control **visibility** — what a process can see. Cgroups control **resources** — what a process can consume. Without cgroups, a container could monopolize all CPU, exhaust all memory, or saturate all disk I/O on the host.

A **cgroup** is a group of processes with shared resource limits. You organize processes into a hierarchy, and each node can have limits applied.

### Key Resource Controllers

| Controller | What It Limits | Key Settings |
|---|---|---|
| **cpu** | CPU time | `cpu.max` (hard limit), `cpu.weight` (relative shares) |
| **memory** | RAM and swap | `memory.max` (hard limit), `memory.swap.max` (swap limit) |
| **io** | Disk I/O bandwidth | `io.max` (bytes/sec and IOPS limits per device) |
| **pids** | Number of processes | `pids.max` (prevents fork bombs) |
| **cpuset** | Which CPUs/NUMA nodes | `cpuset.cpus` (pin to specific cores) |

### CPU Limits

```
cpu.max = "200000 100000"
              │       │
              │       └── Period: 100ms (100000 microseconds)
              └────────── Quota: 200ms of CPU time per period

This means: the cgroup can use up to 2 CPU cores worth of time
per 100ms period. (200ms / 100ms = 2 CPUs)

cpu.weight = 100  (default)
cpu.weight = 50   (half the priority when competing)
cpu.weight = 200  (double the priority when competing)

Weight only matters when CPUs are contended. If the system
is idle, even weight=1 gets full CPU.
```

### Memory Limits

Memory limits are hard walls. When a cgroup exceeds `memory.max`, the kernel's OOM killer terminates a process in that cgroup.

```
memory.max = 536870912  (512 MB hard limit)
memory.swap.max = 0     (no swap allowed)

What happens when the container tries to allocate beyond 512 MB?

Container process: malloc(big_chunk)
         │
         ▼
Kernel: "cgroup memory limit exceeded"
         │
         ▼
OOM Killer: kills a process in THIS cgroup only
            (not other containers, not the host)
         │
         ▼
Container's main process dies → container exits

This is why "OOMKilled" is so common in Kubernetes.
```

The beauty of cgroup-scoped OOM: the kernel only kills processes within the offending cgroup. Other containers and the host are unaffected.

### I/O Limits

```
io.max = "8:0 rbps=10485760 wbps=5242880 riops=1000 wiops=500"
          │    │              │             │          │
          │    │              │             │          └── Write IOPS limit
          │    │              │             └── Read IOPS limit
          │    │              └── Write bytes/sec limit (5 MB/s)
          │    └── Read bytes/sec limit (10 MB/s)
          └── Device major:minor (8:0 = /dev/sda)
```

### cgroups v1 vs. v2

| Feature | cgroups v1 | cgroups v2 |
|---|---|---|
| Hierarchy | Multiple trees, one per controller | Single unified hierarchy |
| Controller attachment | Each controller has its own tree | Controllers attached to shared tree |
| Delegation | Fragmented, inconsistent | Clean, supports unprivileged delegation |
| Pressure info (PSI) | Not available | Built-in pressure stall information |
| Default in modern distros | Legacy | Ubuntu 22.04+, Fedora 31+, RHEL 9+ |

cgroups v2 replaced v1's confusing multi-hierarchy model with a single tree. Most modern systems use v2, but you'll still encounter v1 in older kernels and containers.

## Namespaces + Cgroups = Containers

A container is nothing more than a regular process with:
1. **Namespaces** — giving it an isolated view of the system (own PIDs, network, filesystem)
2. **Cgroups** — limiting its resource consumption (CPU, memory, I/O)
3. **A root filesystem** — an image providing the container's `/`

```
What "creating a container" actually means:

1. Create new namespaces:
   unshare(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC)

2. Set up cgroup limits:
   mkdir /sys/fs/cgroup/my-container
   echo "200000 100000" > /sys/fs/cgroup/my-container/cpu.max
   echo "536870912"     > /sys/fs/cgroup/my-container/memory.max
   echo $$             > /sys/fs/cgroup/my-container/cgroup.procs

3. Set up root filesystem:
   mount --bind /var/lib/container/rootfs /mnt
   pivot_root /mnt /mnt/.old
   umount -l /.old

4. Execute the container's entrypoint:
   exec /usr/bin/nginx

That's it. No hypervisor. No guest kernel. Just a Linux process
with isolation and limits applied.
```

### Manual Namespace Manipulation

You can create and enter namespaces manually:

```bash
# Create a new PID and mount namespace, run bash inside it
sudo unshare --pid --mount --fork bash

# Inside: you're PID 1
echo $$  # prints 1

# From another terminal, enter an existing container's namespaces
sudo nsenter --target <container-pid> --pid --net --mount bash
# Now you're inside the container's view of the world
```

`unshare` creates new namespaces. `nsenter` enters existing ones. These are the same primitives that Docker and containerd use under the hood.

## Real-World Connection

**Kubernetes resource limits are cgroups.** When you write this in a pod spec:

```yaml
resources:
  requests:
    cpu: "500m"      # 0.5 CPU cores → cpu.weight
    memory: "256Mi"  # → memory.low (soft guarantee)
  limits:
    cpu: "1000m"     # 1 CPU core → cpu.max = "100000 100000"
    memory: "512Mi"  # → memory.max = 536870912
```

Kubernetes tells the container runtime to set those exact cgroup values. When your pod gets `OOMKilled`, it's because the cgroup's `memory.max` was exceeded and the kernel's OOM killer acted.

**Pod networking is network namespaces.** All containers in a Kubernetes pod share the same network namespace — that's why they can communicate over `localhost`. Each pod gets its own network namespace with a unique IP, connected to the cluster network via veth pairs.

**Docker's `--pid=host` flag** disables the PID namespace — the container can see all host processes. This is useful for debugging but breaks isolation.

## Interview Angle

**Q: What are Linux namespaces and how do they relate to containers?**
A: Namespaces isolate what a process can see. There are seven types: PID (process tree), network (network stack), mount (filesystem), UTS (hostname), IPC (inter-process communication), user (UID mapping), and cgroup (resource hierarchy view). A container is fundamentally a process running inside a set of namespaces, giving it an isolated view of the system while sharing the host kernel.

**Q: What are cgroups and how do they differ from namespaces?**
A: Cgroups (control groups) limit what resources a process can use — CPU time, memory, I/O bandwidth, number of processes. Namespaces control visibility (what you can see), while cgroups control consumption (how much you can use). Together, they form the two pillars of container isolation. A process in a memory cgroup limited to 512MB will be OOM-killed if it exceeds that limit.

**Q: What happens when a container exceeds its memory limit?**
A: The kernel's OOM killer terminates a process within that cgroup. Critically, the OOM kill is scoped to the cgroup — other containers and the host are unaffected. In Kubernetes, this shows up as an `OOMKilled` status. This is different from a VM, where the guest OS handles its own memory limits and might swap to disk instead of killing processes.

**Q: Can you create a container without Docker?**
A: Yes. A container is just a process with namespaces and cgroups. You can use `unshare` to create new namespaces, set up cgroup limits by writing to `/sys/fs/cgroup/`, mount a root filesystem with `pivot_root`, and exec your process. Docker, containerd, and runc are convenience tools that automate this — but the underlying mechanism is pure kernel functionality.

---

Next: [Containers: Docker and containerd](04-containers-docker-containerd.md)
