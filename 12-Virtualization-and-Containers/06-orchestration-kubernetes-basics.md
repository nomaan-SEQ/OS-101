# Orchestration and Kubernetes Basics

## Why Orchestration?

Running one container is easy. Running 500 containers across 50 machines is a distributed systems problem. You need to answer:

- Which machine has enough CPU and memory for this container?
- What happens when a machine dies and its 15 containers disappear?
- How do containers find and talk to each other when they move between machines?
- How do you roll out a new version without downtime?
- How do you ensure the right number of copies are always running?

Doing this manually is impossible at scale. Container orchestration automates it.

## Kubernetes (K8s): The Dominant Orchestrator

Kubernetes is a container orchestration platform originally built by Google, based on lessons from their internal system (Borg). It's become the industry standard for running containerized workloads at scale.

We won't cover Kubernetes as a tutorial here. Instead, we'll examine it through an OS lens — because Kubernetes solves the same problems as an operating system, just at cluster scale.

### Core Concepts (OS Perspective)

```
Kubernetes Architecture:

  ┌─────────────────────────────────────────────────────┐
  │                  Control Plane                      │
  │  ┌──────────┐  ┌───────────┐  ┌─────────────────┐  │
  │  │ API      │  │ Scheduler │  │ Controller       │  │
  │  │ Server   │  │           │  │ Manager          │  │
  │  └──────────┘  └───────────┘  └─────────────────┘  │
  │  ┌──────────────────────────────────────────────┐   │
  │  │              etcd (state store)              │   │
  │  └──────────────────────────────────────────────┘   │
  └─────────────────────────┬───────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
     ┌──────┴──────┐ ┌─────┴───────┐ ┌─────┴───────┐
     │   Node 1    │ │   Node 2    │ │   Node 3    │
     │  ┌───────┐  │ │  ┌───────┐  │ │  ┌───────┐  │
     │  │kubelet│  │ │  │kubelet│  │ │  │kubelet│  │
     │  ├───────┤  │ │  ├───────┤  │ │  ├───────┤  │
     │  │Pod A  │  │ │  │Pod C  │  │ │  │Pod E  │  │
     │  │Pod B  │  │ │  │Pod D  │  │ │  │Pod F  │  │
     │  └───────┘  │ │  └───────┘  │ │  └───────┘  │
     └─────────────┘ └─────────────┘ └─────────────┘
```

**Pod**: the smallest deployable unit. A pod is one or more containers that share namespaces — they share a network namespace (same IP, can talk via localhost), an IPC namespace (shared memory, message queues), and optionally a PID namespace. Think of a pod as an "application unit" — a web server and its log-shipping sidecar in the same pod.

**Node**: a machine (VM or physical) that runs pods. Each node runs a **kubelet** — the agent that receives pod specs from the control plane and tells containerd to create the appropriate containers.

**Scheduler**: decides which node should run a new pod. It examines each node's available CPU, memory, disk, and other constraints (affinity rules, taints, tolerations) and picks the best fit. This is directly analogous to an OS process scheduler, but instead of assigning processes to CPU cores, it assigns pods to machines.

**Controller Manager**: runs control loops that watch the cluster state and make corrections. If a Deployment says "keep 3 replicas running" and one pod dies, the controller creates a replacement. This is like an OS watchdog or service manager (systemd), but for distributed services.

**etcd**: the cluster's state store. Every piece of cluster state (what pods should exist, where they're running, their configuration) lives in etcd. It's the filesystem of the distributed OS.

## How Kubernetes Uses OS Primitives

This is the key insight: Kubernetes doesn't reinvent the wheel. It orchestrates the same kernel primitives we've been studying.

### Pods = Shared Namespaces

When Kubernetes creates a pod with two containers:

```
Pod Creation (what actually happens):

1. kubelet tells containerd to create a "sandbox" container
   (the "pause" container — holds the namespaces open)

2. Sandbox gets:
   - New network namespace (pod IP: 10.244.1.5)
   - New IPC namespace
   - New UTS namespace

3. Application containers join the EXISTING namespaces:
   ┌─────────────────────────────────────────┐
   │  Pod (shared namespaces)                │
   │  ┌─────────────┐  ┌─────────────┐      │
   │  │ Container A  │  │ Container B  │     │
   │  │ (nginx)      │  │ (log-shipper)│     │
   │  │              │  │              │     │
   │  │ Own PID ns   │  │ Own PID ns   │     │
   │  │ Own mount ns │  │ Own mount ns │     │
   │  └──────────────┘  └──────────────┘     │
   │                                         │
   │  Shared: network ns, IPC ns, UTS ns     │
   │  Pod IP: 10.244.1.5                     │
   │  Both containers see localhost the same  │
   └─────────────────────────────────────────┘
```

Container A (nginx on port 80) and Container B (log-shipper on port 9090) share the same network namespace. They communicate via `localhost`. Each has its own PID namespace and mount namespace — they can't see each other's processes or files.

### Resource Management = Cgroups

Kubernetes resource requests and limits map directly to cgroup settings:

| Kubernetes Spec | Cgroup Setting | Effect |
|---|---|---|
| `requests.cpu: "500m"` | `cpu.weight` (proportional) | Guaranteed minimum CPU share when contended |
| `limits.cpu: "1000m"` | `cpu.max = "100000 100000"` | Hard cap: max 1 CPU core per 100ms period |
| `requests.memory: "256Mi"` | `memory.low` (soft) | Memory that won't be reclaimed under pressure |
| `limits.memory: "512Mi"` | `memory.max = 536870912` | Hard cap: OOM kill if exceeded |
| `limits.pids: "100"` | `pids.max = 100` | Prevents fork bombs inside the pod |

When your pod gets `OOMKilled`, this is what happened:

```
Pod exceeds memory.max:
  Container process: malloc() → total exceeds 512Mi
       │
       ▼
  Kernel: cgroup memory limit exceeded
       │
       ▼
  OOM Killer: terminates a process in this cgroup
       │
       ▼
  Container exits → kubelet reports OOMKilled
       │
       ▼
  Controller: "replica count is below desired, create new pod"
       │
       ▼
  Scheduler: "Node 2 has space, schedule there"
       │
       ▼
  New pod starts on Node 2
```

**CPU throttling** is equally common and more insidious. If `cpu.max` is set to 100ms per 100ms period (1 core), and your process tries to use more, the kernel **throttles** it — the process is paused until the next period. Your application doesn't crash, it just gets slow. This is a frequent source of mysterious latency in Kubernetes.

### Network Policies = iptables/eBPF

Kubernetes network policies control which pods can talk to which other pods. Under the hood, these are implemented as:

- **iptables rules** — the traditional approach. Each rule adds an entry to the kernel's packet filtering tables.
- **eBPF programs** — the modern approach (Cilium). Packet filtering logic runs as eBPF bytecode in the kernel, avoiding the performance overhead of long iptables chains.

Both are kernel-level networking primitives — Kubernetes just generates the rules.

### Storage = Mount Namespaces + CSI

When a pod mounts a persistent volume:

1. The CSI (Container Storage Interface) driver provisions the storage (e.g., an EBS volume on AWS)
2. The volume is attached to the node
3. kubelet mounts it into the container's mount namespace
4. The container sees it as a regular directory

This is just the mount namespace at work — the container's view of the filesystem is independent of other containers.

## Kubernetes as a Distributed Operating System

The parallel between a traditional OS and Kubernetes is deep:

| OS Concept | Traditional OS | Kubernetes Equivalent |
|---|---|---|
| **Process** | A running program | Pod |
| **Process scheduler** | Assigns processes to CPU cores | kube-scheduler assigns pods to nodes |
| **Init system** (systemd) | Starts/monitors services on one machine | kubelet starts/monitors pods on one node |
| **Service manager** | Restarts crashed services | Controllers recreate failed pods |
| **Filesystem** | Stores files and metadata | etcd stores cluster state |
| **Virtual memory** | Each process gets its own address space | Each pod gets its own network namespace (IP) |
| **IPC** | Pipes, shared memory, signals | Services, DNS, shared pod namespaces |
| **Users/permissions** | UIDs, file permissions | RBAC, ServiceAccounts, SecurityContexts |
| **Package manager** | apt, yum | Container registries, Helm charts |
| **Cron** | Scheduled tasks | CronJobs |
| **Daemon** | Background service | DaemonSet (one pod per node) |
| **Resource limits** | ulimit, cgroups | Resource requests/limits |

```
The Layered View:

Traditional OS:                     Kubernetes:
┌────────────────────┐              ┌────────────────────┐
│   Applications     │              │   Pods/Containers  │
├────────────────────┤              ├────────────────────┤
│   System Calls     │              │   Kubernetes API   │
├────────────────────┤              ├────────────────────┤
│   Kernel           │              │   Control Plane    │
│   (scheduler,      │              │   (scheduler,      │
│    memory mgmt,    │              │    controllers,    │
│    filesystem,     │              │    etcd,           │
│    networking)     │              │    networking)     │
├────────────────────┤              ├────────────────────┤
│   Hardware         │              │   Cluster (nodes)  │
└────────────────────┘              └────────────────────┘

Same problems, different scale.
```

## Why OS Knowledge Helps Debug Kubernetes

Understanding the OS primitives underneath Kubernetes is the difference between staring at cryptic error messages and knowing exactly where to look.

**OOMKilled pods**: This is the cgroup memory limit being enforced by the kernel's OOM killer. Check `memory.max` vs actual usage. Is the limit too low, or does the app have a memory leak? Look at `kubectl top pod` and the node's `/sys/fs/cgroup/` entries.

**CPU throttling**: The pod isn't using as much CPU as `limits.cpu` allows — but it's still slow. Check `cpu.stat` for `nr_throttled` and `throttled_usec`. Throttling happens when the pod uses its entire `cpu.max` quota within a scheduling period. Consider removing CPU limits (controversial but common) or increasing them.

**DNS resolution failures**: Each pod's network namespace has its own `/etc/resolv.conf`. Kubernetes configures it to point to the cluster DNS (CoreDNS). If CoreDNS is down or network policies block the traffic, DNS fails. Check `nsenter` into the pod's network namespace and test from there.

**Container filesystem full**: The overlay filesystem's writable layer has a size limit. If a container writes too many logs or temp files, the layer fills up and writes fail. Check `docker system df` or inspect the overlay mount. The fix is usually a volume mount for large write paths.

**Networking issues between pods**: Pods on different nodes communicate through a CNI (Container Network Interface) plugin that creates overlay networks. If pod-to-pod traffic fails, the issue is often in the CNI's network plumbing — veth pairs, bridge configuration, VXLAN tunnels, or iptables rules. `nsenter` into each pod's network namespace and trace the packet path.

## Real-World Connection

When a site reliability engineer debugs a production Kubernetes issue, they're ultimately debugging OS primitives:

- "Pod is OOMKilled" = cgroup memory limit exceeded
- "Pod is stuck in Pending" = scheduler can't find a node with enough resources (like an OS that can't allocate memory for a new process)
- "Pod can't reach another pod" = network namespace routing or iptables/eBPF rules are wrong
- "Container is slow" = CPU throttling from cgroup cpu.max
- "Node is NotReady" = kubelet (the init system for containers) has stopped responding

Google's Borg system (Kubernetes' predecessor) managed billions of containers. The lesson they learned: the same OS concepts — scheduling, memory management, isolation, resource control — apply at every scale. A cluster is just a very large computer.

## Interview Angle

**Q: How does Kubernetes use OS-level primitives to manage containers?**
A: Kubernetes builds on kernel mechanisms throughout. Pods use shared namespaces (network, IPC) for co-located containers. Resource requests and limits map to cgroup settings (cpu.max, memory.max, pids.max). Network policies are implemented as iptables rules or eBPF programs. Volume mounts use the mount namespace. The kubelet on each node uses containerd/runc, which create namespaces and cgroups for each container. Understanding these primitives is essential for debugging Kubernetes issues like OOMKills, CPU throttling, and networking failures.

**Q: Why is Kubernetes described as a "distributed operating system"?**
A: Because it solves the same problems as a traditional OS but across a cluster of machines. The kube-scheduler assigns pods to nodes (like a CPU scheduler assigns processes to cores). Controllers restart failed pods (like systemd restarts crashed services). etcd stores cluster state (like a filesystem). RBAC controls access (like Unix permissions). Network namespaces give each pod its own IP (like virtual memory gives each process its own address space). It's the same abstractions at a different scale.

**Q: A pod keeps getting OOMKilled. How do you debug this?**
A: OOMKilled means the cgroup's memory.max was exceeded. First, check the pod's memory limit (`kubectl describe pod`). Then check actual usage (`kubectl top pod` or metrics from Prometheus). If the limit seems reasonable, the app likely has a memory leak — profile it. If the app legitimately needs more memory, increase `resources.limits.memory`. You can also check the node's `/sys/fs/cgroup/` entries for the container's cgroup to see `memory.current`, `memory.max`, and the OOM kill count in `memory.events`. The OOM kill is scoped to the cgroup, so other pods are unaffected.

**Q: What is CPU throttling in Kubernetes and how does it differ from OOM behavior?**
A: CPU throttling happens when a container uses its entire `cpu.max` quota within a scheduling period (typically 100ms). The kernel pauses the process until the next period — the container doesn't crash, it just gets slow. Check `cpu.stat` for `nr_throttled` events. This is different from memory limits: exceeding memory.max triggers an OOM kill (process dies), but exceeding CPU limits triggers throttling (process slows down). Throttling is harder to detect because the pod appears healthy — it's just slow. Some teams remove CPU limits entirely and rely only on CPU requests for scheduling.
