# Linux cgroups and Namespaces Deep Dive

## The Core Idea

Containers are not a single kernel feature — they are the combination of two independent mechanisms: **namespaces** (what a process can SEE) and **cgroups** (what a process can USE). Namespaces provide isolation — a process sees its own PID space, its own network stack, its own filesystem view. cgroups provide resource control — a process is limited to a specific amount of CPU, memory, and I/O bandwidth.

Neither mechanism alone creates a container. Namespaces without cgroups give isolation without resource limits (a "container" could consume all the host's CPU). cgroups without namespaces give resource limits without isolation (the process still sees all other processes and the full filesystem). Together, they create the complete container abstraction that Docker, Kubernetes, and every container runtime depend on.

---

## Namespaces

**Namespaces are like one-way mirrors in an interrogation room. The container (on the suspect's side) sees only its own isolated world — its own processes, its own network, its own filesystem mounts. The host (on the detective's side) can see everything, including all the containers and their contents. The container does not even know it is contained.**

Each namespace type isolates a specific aspect of the system:

| Namespace | Flag | What It Isolates |
|-----------|------|-----------------|
| **PID** | `CLONE_NEWPID` | Process ID space |
| **Network** | `CLONE_NEWNET` | Network devices, IP addresses, ports, routes, iptables |
| **Mount** | `CLONE_NEWNS` | Filesystem mount table |
| **UTS** | `CLONE_NEWUTS` | Hostname and domain name |
| **IPC** | `CLONE_NEWIPC` | System V IPC, POSIX message queues |
| **User** | `CLONE_NEWUSER` | User and group IDs (UID/GID mapping) |
| **Cgroup** | `CLONE_NEWCGROUP` | Cgroup root directory view |
| **Time** | `CLONE_NEWTIME` | System clocks (boot time, monotonic) — kernel 5.6+ |

### Namespace Syscalls

Three syscalls manipulate namespaces:

| Syscall | Purpose |
|---------|---------|
| `clone()` | Create a new process in new namespace(s) |
| `unshare()` | Move the calling process into new namespace(s) |
| `setns()` | Join an existing namespace (via fd from `/proc/PID/ns/`) |

```bash
# See a process's namespace memberships:
ls -la /proc/self/ns/
# lrwxrwxrwx 1 user user 0 ... cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 user user 0 ... ipc -> 'ipc:[4026531839]'
# lrwxrwxrwx 1 user user 0 ... mnt -> 'mnt:[4026531841]'
# lrwxrwxrwx 1 user user 0 ... net -> 'net:[4026531840]'
# lrwxrwxrwx 1 user user 0 ... pid -> 'pid:[4026531836]'
# lrwxrwxrwx 1 user user 0 ... user -> 'user:[4026531837]'
# lrwxrwxrwx 1 user user 0 ... uts -> 'uts:[4026531838]'

# The number in brackets is the namespace inode — processes in
# the same namespace share the same inode number.
```

---

## PID Namespace Deep Dive

The PID namespace gives each container its own process ID space. A process has a different PID inside its namespace than it has from the host's perspective:

```
Host PID namespace:
  PID 1 (systemd)
  PID 1000 (dockerd)
  PID 2001 (container's init, seen from host)
  PID 2002 (container's app, seen from host)

Container PID namespace:
  PID 1 (container's init)     ← Same process as host PID 2001
  PID 2 (container's app)      ← Same process as host PID 2002
```

### Nested PID Namespaces

PID namespaces can be hierarchical — a namespace can create child namespaces:

```
Host PID NS (level 0)
├── PID 1 (systemd)
├── PID 1000 (container A, init)
│   └── Container A PID NS (level 1)
│       ├── PID 1 (container A init)
│       ├── PID 2 (container A app)
│       └── PID 3 (nested container)
│           └── Nested PID NS (level 2)
│               ├── PID 1 (nested init)
│               └── PID 2 (nested app)
└── PID 2000 (container B, init)
    └── Container B PID NS (level 1)
        └── PID 1 (container B init)
```

A process is visible in its own namespace AND all ancestor namespaces (with different PIDs at each level). A process cannot see anything in sibling or child namespaces.

### PID 1 in Containers: The Zombie Problem

In a PID namespace, PID 1 has special responsibilities:
- **Zombie reaping**: Orphaned processes are reparented to PID 1, which must call `wait()` to collect their exit status. If PID 1 does not reap, zombies accumulate.
- **Signal handling**: PID 1 ignores SIGTERM and SIGINT by default (only responds to signals it explicitly handles).

Most applications are not designed to be PID 1. Running `nginx` or `java` directly as PID 1 means zombies are never reaped and graceful shutdown may not work. Solutions:

| Tool | Approach |
|------|----------|
| **tini** | Minimal init (6 KB binary), reaps zombies, forwards signals |
| **dumb-init** | Similar to tini, slightly different signal handling |
| **Docker --init** | Injects tini automatically (`docker run --init`) |
| **systemd in container** | Full init system (heavy, but handles everything) |

---

## Network Namespace Deep Dive

A network namespace provides a completely isolated network stack:

| Component | Isolated Per Namespace |
|-----------|----------------------|
| Network interfaces | Each namespace has its own (including its own loopback) |
| IP addresses | Independent addressing per namespace |
| Routing table | Separate routes |
| iptables/nftables rules | Separate firewall rules |
| Socket bindings | Port 80 in namespace A is different from port 80 in namespace B |
| /proc/net/* | Shows only this namespace's network state |

### veth Pairs: Connecting Namespaces

A **veth (virtual Ethernet) pair** is like a virtual cable with a plug on each end. One end goes in the container's namespace, the other stays in the host's namespace (usually connected to a bridge):

```
Host Network Namespace                Container Network Namespace
┌────────────────────────┐           ┌────────────────────────┐
│                        │           │                        │
│  eth0 (physical NIC)   │           │  eth0 (veth end)       │
│  172.16.0.10           │           │  172.17.0.2            │
│                        │           │                        │
│  docker0 (bridge)      │           │  lo (loopback)         │
│  172.17.0.1            │           │  127.0.0.1             │
│       │                │           │                        │
│       ├── veth_abc ─────── virtual cable ── eth0            │
│       ├── veth_def ─────── (to container B)                 │
│       └── veth_ghi ─────── (to container C)                 │
│                        │           │                        │
│  iptables: NAT rules   │           │  iptables: (empty)     │
│  for container traffic  │           │                        │
└────────────────────────┘           └────────────────────────┘
```

### How Docker Sets Up Container Networking

Step by step:
1. Docker creates a new network namespace for the container
2. Creates a veth pair (two linked virtual interfaces)
3. Moves one end of the veth pair into the container's namespace
4. Attaches the other end to the `docker0` bridge on the host
5. Assigns an IP address to the container's interface (from the docker subnet)
6. Sets the container's default route to the bridge IP
7. Adds iptables NAT rules on the host to masquerade container traffic

---

## User Namespace

User namespaces provide UID/GID mapping — a process can be UID 0 (root) inside its namespace while being an unprivileged user (e.g., UID 100000) on the host.

```
Mapping: /proc/PID/uid_map
Container UID    Host UID    Range
    0            100000       65536

Container GID: /proc/PID/gid_map
Container GID    Host GID    Range
    0            100000       65536
```

This means:
- Container's root (UID 0) maps to host UID 100000 (unprivileged)
- Container's UID 1 maps to host UID 100001
- Container's UID 1000 maps to host UID 101000

**Rootless containers** depend on user namespaces. Podman's rootless mode creates a user namespace where the user is root inside the container but their regular unprivileged self on the host. Even if an attacker escapes the container as "root," they land on the host as an unprivileged user — dramatically limiting the damage.

User namespaces also enable unprivileged container creation: without user namespaces, creating PID/network/mount namespaces requires `CAP_SYS_ADMIN` (essentially root). With a user namespace, an unprivileged user can create child namespaces because they are root within their user namespace.

---

## cgroups v2 Deep Dive

cgroups (control groups) limit, account for, and isolate resource usage. cgroups v2 (the modern interface, default since kernel 5.8) uses a **unified hierarchy** — one tree for all controllers.

**cgroups v2 is like a budget for each department. CPU budget limits how much compute they can use, memory budget caps their RAM allocation, I/O budget limits disk bandwidth. Go over budget and you get throttled. Blow through a hard limit and the OOM killer shuts down your department.**

### The Hierarchy

```
/sys/fs/cgroup/                          Root cgroup
├── cgroup.controllers                   Available controllers
├── cgroup.subtree_control               Active controllers for children
├── system.slice/                        System services
│   ├── docker.service/                  Docker daemon
│   │   ├── cgroup.procs                 PIDs in this cgroup
│   │   ├── cpu.max                      CPU limit
│   │   ├── memory.max                   Memory limit
│   │   └── io.max                       I/O limit
│   └── sshd.service/
├── user.slice/                          User sessions
│   └── user-1000.slice/
│       └── session-1.scope/
└── docker/                              Container cgroups
    ├── abc123.../                        Container A
    │   ├── cpu.max
    │   ├── memory.max
    │   ├── memory.current               Current memory usage
    │   └── pids.max                     Max processes
    └── def456.../                        Container B
```

### CPU Controller

```bash
# cpu.max format: "QUOTA PERIOD" (both in microseconds)

# Allow 100% of one CPU (100ms every 100ms):
echo "100000 100000" > /sys/fs/cgroup/mygroup/cpu.max

# Allow 50% of one CPU (50ms every 100ms):
echo "50000 100000" > /sys/fs/cgroup/mygroup/cpu.max

# Allow 200% CPU (2 full cores):
echo "200000 100000" > /sys/fs/cgroup/mygroup/cpu.max

# No limit (default):
echo "max 100000" > /sys/fs/cgroup/mygroup/cpu.max

# Relative weight (for proportional sharing):
echo "50" > /sys/fs/cgroup/mygroup/cpu.weight   # Default is 100
```

Docker translation: `docker run --cpus=0.5` sets `cpu.max` to `"50000 100000"`.

### Memory Controller

```bash
# Hard limit: process gets OOM-killed if it exceeds this
echo "536870912" > /sys/fs/cgroup/mygroup/memory.max    # 512 MB

# Soft limit (high watermark): throttle allocation, reclaim aggressively
echo "268435456" > /sys/fs/cgroup/mygroup/memory.high   # 256 MB

# Current usage:
cat /sys/fs/cgroup/mygroup/memory.current               # Bytes currently used

# Memory stats:
cat /sys/fs/cgroup/mygroup/memory.stat
# anon 134217728          Anonymous pages (heap, stack)
# file 67108864           Page cache pages
# slab 4194304            Kernel slab allocations
# ...

# Swap limit:
echo "0" > /sys/fs/cgroup/mygroup/memory.swap.max       # No swap allowed
```

The difference between `memory.max` (hard) and `memory.high` (soft) is critical:
- **memory.max**: Exceeding this triggers the cgroup OOM killer. Processes are killed.
- **memory.high**: Exceeding this causes the kernel to aggressively reclaim memory from this cgroup (writeback dirty pages, reclaim page cache, swap). Processes are slowed but not killed. This is usually what you want.

Docker: `docker run --memory=512m` sets `memory.max`. There is no direct Docker flag for `memory.high` (you must set it via cgroup files directly).

### I/O Controller

```bash
# Limit read/write bandwidth on a specific device:
# Format: "MAJOR:MINOR TYPE=LIMIT"

# Limit device 8:0 (sda) to 1 MB/s read:
echo "8:0 rbps=1048576" > /sys/fs/cgroup/mygroup/io.max

# Limit device 8:0 to 500 IOPS write:
echo "8:0 wiops=500" > /sys/fs/cgroup/mygroup/io.max

# Combined limits:
echo "8:0 rbps=10485760 wbps=5242880 riops=1000 wiops=500" > io.max

# Check device major:minor:
ls -la /dev/sda
# brw-rw---- 1 root disk 8, 0 ...
```

### PID Controller

```bash
# Limit number of processes (prevent fork bomb):
echo "100" > /sys/fs/cgroup/mygroup/pids.max

# Current count:
cat /sys/fs/cgroup/mygroup/pids.current
```

---

## PSI: Pressure Stall Information

PSI (merged in kernel 4.20) measures how much processes are **stalled** waiting for resources. It is a much better indicator of resource pressure than utilization metrics:

```bash
cat /proc/pressure/cpu
# some avg10=2.50 avg60=1.30 avg300=0.85 total=12345678
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

cat /proc/pressure/io
# some avg10=5.20 avg60=3.10 avg300=1.50 total=23456789
# full avg10=3.10 avg60=1.80 avg300=0.90 total=12345678
```

| Line | Meaning |
|------|---------|
| **some** | Percentage of time at least ONE task is stalled |
| **full** | Percentage of time ALL tasks are stalled (nothing productive is happening) |

**systemd-oomd** uses PSI memory pressure to proactively kill cgroups under memory pressure BEFORE the kernel OOM killer triggers. It monitors `memory.pressure` files per cgroup and kills the highest-pressure cgroup when the system is under duress. This is more intelligent than the kernel OOM killer, which uses a simple scoring heuristic.

Per-cgroup PSI is also available:
```bash
cat /sys/fs/cgroup/docker/abc123/memory.pressure
cat /sys/fs/cgroup/docker/abc123/io.pressure
```

---

## Building a Container from Scratch (Conceptual Walkthrough)

To truly understand how containers work, here is a conceptual walkthrough of what a container runtime does — no Docker magic, just raw kernel primitives:

```bash
# Step 1: Create new namespaces for isolation
# unshare creates a new process with its own PID, mount, UTS, network, and user namespaces
unshare --pid --mount --uts --net --user --fork --map-root-user /bin/bash

# Step 2: Inside the new namespaces, set up the environment

# Set hostname (UTS namespace isolates this)
hostname my-container

# Mount a new proc filesystem (reflects our PID namespace)
mount -t proc proc /proc

# Create a minimal root filesystem (or use an extracted container image)
# In reality, the runtime extracts an OCI image's layers into a directory

# Step 3: Set up networking
# (From the HOST, because the container has an empty network namespace)
# Create veth pair:
#   ip link add veth0 type veth peer name veth1
# Move one end into the container:
#   ip link set veth1 netns <container_pid>
# Configure addresses on both ends

# Step 4: Apply cgroup resource limits
# Create a cgroup for this container:
#   mkdir /sys/fs/cgroup/my-container
# Set limits:
#   echo "100000 100000" > /sys/fs/cgroup/my-container/cpu.max
#   echo "536870912" > /sys/fs/cgroup/my-container/memory.max
#   echo "100" > /sys/fs/cgroup/my-container/pids.max
# Move the container process into the cgroup:
#   echo <container_pid> > /sys/fs/cgroup/my-container/cgroup.procs

# Step 5: Apply security restrictions
# Set up seccomp-bpf filter (block dangerous syscalls)
# Drop capabilities (keep only what is needed)
# Apply AppArmor/SELinux profile

# Step 6: Pivot to the container's root filesystem
# pivot_root new_root old_root
# umount old_root
# exec the container's entrypoint (e.g., /bin/nginx)
```

This is essentially what Docker, containerd, and runc do — with much more error handling, image management, and configuration. But at the kernel level, a container is just: namespaces + cgroups + security policies + a filesystem.

---

## Real-World Connection

**Kubernetes resource requests and limits:** When you set `resources.limits.memory: 512Mi` in a Kubernetes pod spec, the kubelet translates this to `memory.max` in the container's cgroup. `resources.requests.memory: 256Mi` affects scheduling decisions and sets `memory.low` (the protected minimum). Understanding cgroups makes Kubernetes resource management transparent — it is not magic, just cgroup configuration.

**Docker container networking:** Every `docker run` creates a network namespace, a veth pair, and iptables rules. `docker network create` creates additional bridges. `docker run --net=host` skips namespace creation entirely, giving the container the host's full network stack. Understanding this makes debugging container networking issues straightforward: check veth pairs with `ip link`, check bridges with `brctl show`, check NAT with `iptables -t nat -L`.

**Rootless containers and security:** Podman runs containers as an unprivileged user using user namespaces. This means even a container escape gives the attacker only the privileges of an unprivileged user on the host — not root. This is a fundamental security improvement over Docker's default root-daemon model and is increasingly the default in security-conscious environments.

---

## Interview Angle

**Q: What are Linux namespaces and how do they enable containers?**

A: Namespaces provide per-process isolation of system resources. Each namespace type isolates a specific aspect: PID namespace gives a process its own PID space, network namespace gives its own network stack, mount namespace gives its own filesystem view, and so on. A container uses multiple namespaces together so the contained process sees an isolated environment — its own PID 1, its own network interfaces, its own hostname, its own filesystem. From inside, it looks like a separate machine. From outside, it is just a regular process with restricted visibility. The key syscalls are `clone()` (create process in new namespaces), `unshare()` (move into new namespaces), and `setns()` (join existing namespaces).

**Q: Explain the difference between cgroups v2 memory.max and memory.high.**

A: `memory.max` is a hard limit — if a cgroup's memory usage reaches this value and the kernel cannot reclaim enough pages, the cgroup's OOM killer activates and kills a process within the cgroup. `memory.high` is a soft limit — when usage exceeds this value, the kernel aggressively throttles memory allocation for the cgroup, reclaiming pages and slowing down the processes, but does not kill them. In practice, you should set `memory.high` to your expected working set and `memory.max` higher as a safety net. This gives processes a chance to release memory under pressure before being killed.

**Q: Why do containers need a proper init process (like tini) as PID 1?**

A: In a PID namespace, PID 1 has two special responsibilities: reaping orphaned zombie processes (by calling `wait()`) and handling signals correctly (PID 1 ignores SIGTERM/SIGINT by default). Most applications are not designed for this role. If your application spawns child processes that die, they become zombies that are never cleaned up, eventually hitting the PID limit. And if Docker sends SIGTERM for a graceful shutdown, an unprepared PID 1 may ignore it, forcing Docker to wait and then SIGKILL after a timeout. An init shim like tini is lightweight (6 KB), reaps zombies, and forwards signals to the application — solving both problems.

---

**Previous**: [07-linux-boot-process-in-depth.md](07-linux-boot-process-in-depth.md)
