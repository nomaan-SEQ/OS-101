# OS for Distributed Systems

A traditional operating system manages one machine: one CPU (or a few), one pool of memory, one set of disks, one network stack. But modern applications run across hundreds or thousands of machines. Who manages the cluster? Who schedules work across nodes? Who provides a unified filesystem view? These questions lead to a powerful idea: **the cluster needs an operating system too**.

---

## The Single-Machine OS vs the Distributed OS

Every concept in a single-machine OS has a distributed equivalent. Understanding this mapping is the key mental model for this section.

```
Single Machine                          Cluster of Machines
+------------------+                    +----------------------------------+
| CPU Scheduler    |  ---- maps to ---> | Cluster Scheduler (K8s, Mesos)   |
| Memory (RAM)     |  ---- maps to ---> | Distributed Cache (Redis)        |
| Filesystem       |  ---- maps to ---> | Distributed FS (HDFS, Ceph)      |
| Processes        |  ---- maps to ---> | Pods / Tasks / Jobs              |
| IPC (pipes, shm) |  ---- maps to ---> | Service Mesh, gRPC, message queue|
| Virtual Memory   |  ---- maps to ---> | Distributed Shared Memory, RDMA  |
| Kernel           |  ---- maps to ---> | Orchestrator (Kubernetes)        |
+------------------+                    +----------------------------------+
```

| Single-Machine OS Concept | Distributed Equivalent | Example |
|---|---|---|
| CPU scheduler | Cluster scheduler | Kubernetes scheduler, Mesos, Nomad |
| Process | Pod / Task | Kubernetes pod, ECS task |
| fork() / exec() | Deploy / scale | `kubectl scale`, `docker service scale` |
| kill / signal | Eviction / rolling update | Pod eviction, SIGTERM + grace period |
| RAM | Distributed cache | Redis, Memcached |
| Virtual memory | Distributed shared memory | RDMA, DSM (academic) |
| Local filesystem | Distributed filesystem | HDFS, Ceph, NFS, GFS |
| IPC (pipes, sockets) | Service mesh, message queues | Istio, RabbitMQ, Kafka |
| Page replacement (LRU) | Cache eviction | Redis LRU eviction policy |
| Journaling filesystem | Replicated log / WAL | Raft log, database WAL |
| `/proc` filesystem | Cluster monitoring | Prometheus, Kubernetes API |

---

## Distributed Scheduling

On a single machine, the CPU scheduler decides which process runs on which core. In a cluster, the **cluster scheduler** decides which workload runs on which machine.

### How Kubernetes Scheduling Works

```
User: "Run 3 replicas of my web server, each needs 2 CPU + 4GB RAM"
                    |
                    v
          +------------------+
          | Kubernetes API   |
          | Server           |
          +--------+---------+
                   |
                   v
          +------------------+
          | Scheduler        |
          |                  |
          | 1. Filter nodes  |  <-- Which nodes have enough resources?
          | 2. Score nodes   |  <-- Which eligible node is best?
          | 3. Bind pod      |  <-- Assign pod to winning node
          +--------+---------+
                   |
     +-------------+-------------+
     v             v             v
+---------+   +---------+   +---------+
| Node 1  |   | Node 2  |   | Node 3  |
| 8 CPU   |   | 16 CPU  |   | 8 CPU   |
| 32GB    |   | 64GB    |   | 32GB    |
| Pod: web|   | Pod: web|   | Pod: web|
+---------+   +---------+   +---------+
```

### Scheduling Strategies

| Strategy | What It Does | Analogy |
|---|---|---|
| **Bin packing** | Pack workloads tightly onto fewest nodes | Tetris -- fill gaps to minimize waste |
| **Spreading** | Distribute across nodes for fault tolerance | Do not put all eggs in one basket |
| **Affinity** | Place pods near related pods | Co-locate frontend and cache on same node |
| **Anti-affinity** | Keep pods away from each other | Separate database replicas across failure domains |
| **Topology-aware** | Consider rack, zone, region | Place replicas in different availability zones |
| **Priority/preemption** | High-priority pods evict low-priority ones | Real-time vs batch scheduling |

This is directly analogous to CPU scheduling: bin packing is like work-conserving scheduling, priority/preemption is like priority-based scheduling, and affinity is like processor affinity (pinning threads to cores for cache locality).

---

## Distributed Memory

On a single machine, all processes share physical RAM (isolated by virtual memory). In a distributed system, memory is per-machine. Sharing state across machines requires different approaches.

### Distributed Shared Memory (DSM)

The academic ideal: make the cluster's total memory look like one large address space.

```
DSM (Academic):

Process A (Node 1)         Process B (Node 2)
reads address 0x8000  ---> page fault! ---> fetch page from Node 2
                                            over network
```

In practice, DSM has not succeeded because network latency (microseconds) is 1000x worse than local memory access (nanoseconds). Every "remote page fault" is catastrophically slow compared to a local one.

### What We Actually Use: Distributed Caches

Instead of pretending the network is not there, distributed caches explicitly manage remote data access.

```
Application       Cache (Redis/Memcached)      Database
+---------+       +--------------------+        +--------+
|         |--get->| Key: user:123      |        |        |
|         |<-hit--| Val: {name: "Jo"}  |        |        |
|         |       +--------------------+        |        |
|         |                                     |        |
|         |--get->| Key: user:456      |        |        |
|         |<-miss-| (not found)        |        |        |
|         |-------|------ query ------->------->|        |
|         |<------|------ result ------<--------+        |
|         |--set->| Key: user:456      |        |        |
|         |       | Val: {name: "Mo"}  |        +--------+
+---------+       +--------------------+
```

Distributed caches like Redis and Memcached are, in OS terms, the cluster equivalent of the page cache. Just as the page cache sits between processes and the disk, Redis sits between services and the database.

### RDMA: Remote Direct Memory Access

**RDMA** bypasses the kernel network stack entirely, allowing one machine to read/write another machine's memory directly.

```
Traditional Network:                    RDMA:

Application                             Application
    |                                       |
    v                                       v
  Socket API                             RDMA Verbs
    |                                       |
    v                                       |
  TCP/IP Stack (kernel)                     | (bypass kernel)
    |                                       |
    v                                       v
  NIC Driver                             RDMA NIC (hardware)
    |                                       |
    v                                       v
  Network                               Network
```

RDMA achieves latencies under 2 microseconds -- compared to 50+ microseconds for TCP. It is used in:
- High-performance computing (HPC) clusters
- Distributed databases (e.g., FaRM from Microsoft Research)
- Machine learning training (GPU-to-GPU communication)
- Financial trading systems

---

## Distributed File Systems

On a single machine, the filesystem provides a hierarchical namespace for persistent data. Distributed file systems extend this across machines.

### Evolution of Distributed File Systems

| System | Model | Designed For | Key Property |
|---|---|---|---|
| **NFS** | Client-server | General file sharing | Simple, transparent, POSIX-like |
| **GFS/Colossus** | Distributed, master + chunkservers | Google's web-scale data | Write-once, append-heavy, 64MB chunks |
| **HDFS** | Distributed, NameNode + DataNodes | Hadoop big data workloads | Inspired by GFS, write-once, read-many |
| **Ceph** | Distributed, no single master | General-purpose (block, object, file) | CRUSH algorithm, no single point of failure |
| **Amazon EBS** | Network-attached block device | Cloud VM storage | Replicated, snapshotable, looks like a local disk |

### HDFS Architecture

```
Client                    NameNode                DataNodes
+------+                 +----------+            +---------+
|      |--- "open        | Metadata |            | DN 1    |
|      |    /data/f1" -->| /data/f1 |            | Block A |
|      |                 | Block A: DN1, DN3     | Block C |
|      |<-- "Block A is  | Block B: DN2, DN3     +---------+
|      |    on DN1, DN3" | Block C: DN1, DN2     +---------+
|      |                 +----------+            | DN 2    |
|      |--- read --------------------------->   | Block B |
|      |    Block A from DN1                     |         |
|      |<-- data --------------------------------+---------+
+------+                                         +---------+
                                                 | DN 3    |
                                                 | Block A |
                                                 | Block B |
                                                 +---------+
```

The NameNode is the metadata server (like a filesystem's inode table). DataNodes store actual data blocks (like disk sectors). Blocks are replicated (typically 3x) across DataNodes for fault tolerance -- analogous to RAID but across machines.

### Ceph: No Single Point of Failure

Ceph avoids HDFS's single-master bottleneck. Instead of a NameNode, it uses the **CRUSH algorithm** to compute where data should live based on a deterministic hash. Any node can compute the location of any piece of data without asking a central server.

```
Ceph:

Client: "Where is object X?"
    |
    v
CRUSH(X) = OSD 7, OSD 12, OSD 23    <-- deterministic computation
    |                                     (no central lookup)
    v
Read from OSD 7 (primary)
```

---

## Distributed Consensus

When multiple machines must agree on something (who is the leader? what is the current state?), they need a **consensus protocol**. From an OS perspective:

| OS Concept | Consensus Equivalent |
|---|---|
| Process scheduling (who runs next?) | Leader election (who is the primary?) |
| Journaling (write-ahead log) | Replicated log (Raft log, Paxos log) |
| Lock management | Distributed locks (etcd, ZooKeeper) |
| Filesystem consistency | Linearizable reads/writes |

### Raft: Consensus Made (Relatively) Understandable

Raft is the consensus protocol used by etcd (Kubernetes' brain), CockroachDB, and many other distributed systems.

```
Raft Cluster (3 nodes):

+----------+         +-----------+         +----------+
|  Leader  |--log--->| Follower  |         | Follower |
|  Node 1  |--log--->| Node 2    |         | Node 3   |
+----------+         +-----------+         +----------+
     ^                                          |
     |                                          |
     +--- heartbeat (I'm still alive) ----------+

1. Client sends write to Leader
2. Leader appends to its log
3. Leader replicates log entry to Followers
4. Majority (2/3) acknowledge --> committed
5. Leader responds to client
```

If the leader fails, followers detect missing heartbeats and elect a new leader. This is analogous to how an OS handles a crashed process: detect the failure and recover.

---

## The "Cluster as a Computer" Vision

The ultimate expression of this idea: treat the entire cluster as a single computer, with the orchestrator as its OS.

```
Kubernetes as a Distributed OS:

+---------------------------------------------------+
|                  Kubernetes                        |
|                                                    |
|  API Server    = syscall interface                 |
|  Scheduler     = CPU scheduler                     |
|  etcd          = filesystem (stores all state)     |
|  kubelet       = init process (manages pods)       |
|  kube-proxy    = network stack                     |
|  Pod           = process                           |
|  Service       = DNS / named pipe                  |
|  PV/PVC        = disk mount                        |
|  ConfigMap     = /etc config files                 |
|  Secret        = /etc/shadow                       |
|  Namespace     = user (multi-tenancy)              |
|  RBAC          = file permissions                  |
|  DaemonSet     = kernel module (runs on every node)|
+---------------------------------------------------+
```

**Google Borg** was the internal predecessor to Kubernetes, managing Google's entire fleet. It treats the datacenter as a single computer where you submit jobs and the system handles placement, monitoring, and recovery.

**Microsoft Azure Service Fabric** takes a similar approach, providing a programming model where you write services and the fabric handles distribution, replication, and failover.

---

## Real-World Connection

**Kubernetes as the distributed OS**: Kubernetes has become the de facto "operating system" for the cloud. Just as Linux provides a standard interface for applications on a single machine, Kubernetes provides a standard interface for applications across a cluster. The `kubectl` CLI is, in many ways, the distributed equivalent of shell commands.

**AWS abstractions as distributed OS components**:
- **EBS** (Elastic Block Store) = distributed disk (replicated block device over the network)
- **VPC** (Virtual Private Cloud) = distributed network namespace
- **ELB** (Elastic Load Balancer) = distributed network routing
- **S3** = distributed filesystem (object storage)
- **SQS** = distributed IPC (message queue)

**Vitess**: A distributed MySQL system that shards and replicates MySQL across many nodes, presenting a unified SQL interface. It is, in effect, a distributed database engine -- the distributed equivalent of a storage manager in a single-machine OS.

**RDMA in practice**: Microsoft's Azure uses RDMA extensively for storage traffic between hosts. Google uses RDMA in their Jupiter network fabric for machine learning training, where GPU-to-GPU communication latency directly impacts training time.

---

## Interview Angle

**Q: What does it mean to treat a cluster as a single computer? What OS abstractions have distributed equivalents?**

A: The idea is that an orchestrator like Kubernetes provides the same abstractions a single-machine OS provides, but across a cluster. The scheduler places workloads on nodes (like a CPU scheduler places threads on cores). etcd stores cluster state (like a filesystem stores metadata). Services provide discovery and routing (like DNS and sockets provide IPC). PersistentVolumes provide storage that outlives individual pods (like mounted filesystems outlive processes). The API server provides a programmatic interface to the cluster (like syscalls provide a programmatic interface to the kernel). The key difference is latency: every operation that was nanoseconds on a single machine is microseconds to milliseconds across a network. Distributed systems must design around this -- using eventual consistency, caching, and explicit failure handling in ways that single-machine OS design does not.

**Q: Why has Distributed Shared Memory not succeeded in practice?**

A: DSM tries to make remote memory look like local memory by handling "remote page faults" transparently. The problem is physics: local memory access is ~100 nanoseconds, network round-trip is ~50-500 microseconds. That is a 500-5000x difference. Every remote page fault incurs this penalty, and the transparent abstraction means the programmer cannot optimize for it. In practice, systems use explicit remote data access (Redis, Memcached, database queries) where the programmer knows a network call is happening and can batch requests, cache locally, and handle failures. RDMA narrows the gap (under 2 microseconds) but requires specialized hardware and is not transparent.

**Q: How is Kubernetes scheduling analogous to CPU scheduling?**

A: A CPU scheduler assigns threads to cores based on priority, fairness, and affinity. Kubernetes scheduler assigns pods to nodes based on resource requests (CPU, memory), node affinity/anti-affinity, topology constraints, and priority. Bin packing in Kubernetes (filling nodes efficiently) is like work-conserving scheduling (never idle a core when work is available). Priority-based preemption (high-priority pods evict low-priority ones) maps directly to priority-based CPU scheduling. Topology-aware scheduling (spreading across zones) is like NUMA-aware scheduling (placing threads near their memory). The abstractions are remarkably similar; the scale and failure modes are different.

**Q: Compare HDFS and Ceph as distributed file systems.**

A: HDFS uses a centralized NameNode for metadata -- every file operation asks the NameNode where data lives, then talks directly to DataNodes for actual data. This is simple but the NameNode is a single point of failure and a scalability bottleneck (all metadata must fit in its memory). Ceph eliminates the single master by using the CRUSH algorithm: clients compute data locations deterministically from a cluster map, with no central lookup. This makes Ceph more scalable and resilient, but CRUSH is more complex to understand and operate. HDFS is optimized for write-once, read-many batch analytics (Hadoop). Ceph is general-purpose, supporting block, object, and file interfaces. In practice, HDFS dominates big data analytics; Ceph dominates software-defined storage in private clouds.
