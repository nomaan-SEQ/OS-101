# Microservices and OS Implications

When you break a monolithic application into dozens or hundreds of microservices, you do not just change your software architecture -- you fundamentally change what you ask the operating system to do. The kernel that comfortably ran one Java process with 50 threads now must manage 200 containers, each with its own network namespace, filesystem view, and resource limits. This is not a minor scaling adjustment. It is a qualitative shift in how the OS is used.

---

## From Monolith to Microservices: What Changes at the OS Level

```
MONOLITH                              MICROSERVICES
+----------------------------+        +------+ +------+ +------+
|                            |        | Svc  | | Svc  | | Svc  |
|   One Process              |        |  A   | |  B   | |  C   |
|   Many Threads             |        +------+ +------+ +------+
|   Shared Memory            |        +------+ +------+ +------+
|   Local Function Calls     |        | Svc  | | Svc  | | Svc  |
|   One Filesystem View      |        |  D   | |  E   | |  F   |
|   One Network Namespace    |        +------+ +------+ +------+
|                            |           |        |        |
+----------------------------+           +--------+--------+
       |                                  Network (HTTP/gRPC)
   localhost                              Overlay Network
```

| Concern | Monolith | Microservices |
|---|---|---|
| Process count | 1 (or few) | Hundreds |
| Communication | Function calls (nanoseconds) | Network calls (milliseconds) |
| Isolation | Threads share address space | Each service has own namespace |
| Resource limits | One set of limits for the app | Per-service cgroup limits |
| Discovery | Method lookup (compiler/linker) | Service discovery (DNS, API) |
| Debugging | Debugger, stack trace | Distributed tracing (Jaeger, Zipkin) |
| Deployment | One artifact | Many artifacts, independent lifecycles |

---

## OS Primitives That Enable Microservices

Microservices do not run on magic. They run on well-known Linux kernel features, composed in specific ways.

### Containers = Namespaces + cgroups

Each microservice typically runs in its own container, which provides:

- **PID namespace**: the service sees only its own processes
- **Network namespace**: the service gets its own network stack (IP, ports, routing table)
- **Mount namespace**: the service sees its own filesystem view
- **cgroups**: CPU, memory, I/O limits per service

```
Host Kernel
+----------------------------------------------------------+
|                                                          |
|  +------------------+  +------------------+              |
|  | Container A      |  | Container B      |              |
|  | PID ns: 1,2,3    |  | PID ns: 1,2      |             |
|  | Net ns: 10.0.1.5 |  | Net ns: 10.0.1.6 |             |
|  | Mnt ns: /app     |  | Mnt ns: /app     |             |
|  | cgroup: 512MB    |  | cgroup: 1GB      |              |
|  +------------------+  +------------------+              |
|                                                          |
|  Shared Kernel: scheduling, memory management, TCP/IP    |
+----------------------------------------------------------+
```

### Overlay Networks

Services on different hosts need to communicate as if they were on the same network. Overlay networks (VXLAN, Geneve) encapsulate packets so that `Service A on Host 1` can reach `Service B on Host 3` using a flat virtual network.

```
Host 1                          Host 3
+-----------+                   +-----------+
| Service A |                   | Service B |
| 10.244.1.5|                   | 10.244.3.8|
+-----+-----+                   +-----+-----+
      |                               |
      | VXLAN encapsulation           | VXLAN decapsulation
      v                               ^
+-----+------+    Physical Net   +----+-------+
| eth0       +==================>+ eth0       |
| 192.168.1.1|                   | 192.168.1.3|
+------------+                   +------------+
```

### cgroups for Resource Isolation

Each microservice gets resource limits enforced by the kernel:

```
cgroup: /kubepods/pod-abc123/container-xyz
  cpu.max: 200000 100000      # 2 CPU cores max
  memory.max: 536870912        # 512 MB
  io.max: 253:0 rbps=10485760  # 10 MB/s read
```

Without cgroups, a misbehaving service could consume all host CPU or memory and kill neighboring services -- the "noisy neighbor" problem.

---

## The Sidecar Pattern

In a microservice architecture, every service needs cross-cutting concerns: logging, metrics, retries, mTLS, circuit breaking. Instead of embedding this logic in every service, you deploy a **sidecar** -- a helper process that shares the service's network namespace.

```
Pod (shared network namespace: 10.244.1.5)
+------------------------------------------+
|                                          |
|  +-------------+    +----------------+   |
|  | Application |    | Sidecar Proxy  |   |
|  | (your code) |    | (Envoy)        |   |
|  |             |    |                |   |
|  | :8080  -----+--->| :15001 --------+---+--> To other services
|  |             |    |                |   |
|  +-------------+    +----------------+   |
|                                          |
+------------------------------------------+
```

Because the sidecar shares the pod's network namespace, it can intercept all inbound and outbound traffic using iptables rules. The application does not know the sidecar exists -- it just sends HTTP to `service-b:8080`, and the sidecar handles mTLS, retries, and load balancing transparently.

### Service Mesh

When every service has a sidecar proxy, the collection of sidecars forms a **service mesh** (e.g., Istio, Linkerd). The mesh provides:

- **mTLS everywhere**: encrypted service-to-service communication without app changes
- **Retries and circuit breaking**: resilience logic at the network layer
- **Observability**: every request is traced without instrumenting application code
- **Traffic control**: canary deployments, A/B testing at the network level

From the OS perspective, a service mesh means: more processes (each sidecar is an Envoy process), more iptables rules, more network hops (app -> sidecar -> network -> sidecar -> app), and more memory consumption.

---

## Challenges for the OS

Running hundreds of microservices on a single host (or across a cluster) pushes the kernel in ways a monolith never did.

### More Context Switches

Each microservice is its own process (or set of processes). The scheduler must context-switch between hundreds of processes instead of scheduling threads within one process. This increases scheduling overhead and can degrade cache locality.

### More Network Traffic (East-West)

Monoliths communicate internally via function calls. Microservices communicate over the network. Traffic that used to be local function calls is now TCP connections between containers.

```
Traffic Patterns:

MONOLITH:                    MICROSERVICES:
                             
  Client                       Client
    |                            |
    v                            v
  [App]                       [Gateway]
    |                          / | \
    v                         v  v  v
  [DB]                      [A] [B] [C]   <-- "North-South"
                             |\ /| \|
                             v v  v  v
                            [D] [E] [F]   <-- "East-West"
                                          (internal service-to-service)
```

East-west traffic can be 10-100x the volume of north-south traffic. The kernel network stack processes far more packets.

### More File Descriptors and Sockets

Each service maintains connections to other services, databases, caches, and message queues. A single host running 50 microservices might have thousands of open sockets.

### Kernel Parameter Tuning

Default kernel parameters are tuned for general workloads. Microservices at scale require tuning:

| Parameter | Default | Microservice Tuning | Why |
|---|---|---|---|
| `net.core.somaxconn` | 128 | 4096+ | More concurrent connections |
| `fs.file-max` | 65536 | 1000000+ | More open files/sockets |
| `net.ipv4.ip_local_port_range` | 32768-60999 | 1024-65535 | More ephemeral ports for outbound connections |
| `net.core.netdev_max_backlog` | 1000 | 5000+ | Handle packet bursts |
| `net.ipv4.tcp_tw_reuse` | 0 | 1 | Reuse TIME_WAIT sockets faster |
| `vm.max_map_count` | 65530 | 262144+ | More memory-mapped regions (JVM, Elasticsearch) |

Forgetting to tune these is a classic production incident: "everything works in staging, but in production with 200 services we run out of ephemeral ports."

---

## Real-World Connection

**Kubernetes + Istio**: The dominant microservice platform. Kubernetes manages container lifecycle (namespaces, cgroups, scheduling). Istio deploys Envoy sidecars for service mesh capabilities. Together, they use virtually every OS primitive discussed here.

**AWS ECS/Fargate**: Amazon's container orchestrator. Fargate is "serverless containers" -- AWS manages the host, you define the container. Under the hood: cgroups for isolation, overlay networking for connectivity, kernel tuning done by AWS.

**Kernel tuning at scale**: Companies like Netflix and Uber publish their kernel tuning parameters because at microservice scale, defaults break. Netflix's `sysctl` configurations for their container fleet are publicly documented -- they tune hundreds of kernel parameters.

**The cost of microservices**: Segment famously wrote about their move back from microservices to a monolith because the operational overhead (managing hundreds of services, debugging distributed failures, kernel tuning) outweighed the benefits for their scale.

---

## Interview Angle

**Q: How does a microservice architecture change the demands on the Linux kernel compared to a monolithic application?**

A: A monolith runs as one process with many threads sharing an address space -- communication is local function calls, there is one set of resource limits, and the kernel handles a modest number of context switches. Microservices flip this: you have hundreds of separate processes, each in its own container (PID/network/mount namespaces + cgroups). Communication moves from in-process to over-the-network, which means far more TCP connections, more file descriptors, more ephemeral port usage, and more east-west network traffic. The scheduler handles more context switches with worse cache locality. Kernel parameters that were fine for a monolith (somaxconn, file-max, ip_local_port_range) need aggressive tuning. You also need overlay networking so services on different hosts can communicate, and potentially a service mesh (Envoy sidecars) which adds even more processes and iptables rules. The kernel does not do fundamentally different things -- it does the same things at much higher volume and with more isolation boundaries.

**Q: What is a sidecar proxy and why does it share the pod's network namespace?**

A: A sidecar proxy (like Envoy in Istio) handles cross-cutting network concerns -- mTLS, retries, circuit breaking, observability -- so application code does not have to. It shares the pod's network namespace so it can intercept all traffic via iptables rules without the application knowing. Because they share the same network namespace, the sidecar sees the same IP address and can bind to ports that redirect traffic through it. The application sends a plain HTTP request to another service, and the sidecar transparently upgrades it to mTLS, adds tracing headers, and handles retries.

**Q: What happens if you do not tune kernel parameters for a microservice deployment?**

A: You hit resource limits that were invisible with a monolith. The most common failures: running out of ephemeral ports (ip_local_port_range too narrow -- services cannot make outbound connections), hitting the file descriptor limit (fs.file-max -- too many sockets open), connection backlog overflow (somaxconn too low -- connections get dropped under load), and memory-mapped region limits (vm.max_map_count -- JVM-based services crash). These failures are insidious because they work fine in development and staging, then break at production scale.

---

Next: [eBPF and Kernel Observability](02-ebpf-and-kernel-observability.md)
