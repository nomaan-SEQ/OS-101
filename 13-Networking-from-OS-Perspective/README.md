# Networking from the OS Perspective

The network is the most I/O-intensive subsystem in a modern OS. Every HTTP request, database query, and container-to-container call flows through the kernel's network stack — a pipeline of protocol processing, buffer management, and hardware interaction that the OS must orchestrate efficiently. This section is not a networking course. It is about how the operating system implements networking: how packets traverse kernel layers, how sockets map to file descriptors, how containers get isolated networks, and how high-performance systems bypass the kernel entirely.

Understanding the OS networking layer explains why your server runs out of ephemeral ports, why Docker containers can talk to each other, why Nginx serves static files so fast, and why cloud load balancers can handle millions of packets per second.

## Why This Matters

- **System performance**: Network I/O is the bottleneck for most backend services — understanding the kernel path reveals where latency hides
- **Container networking**: Every Kubernetes pod, Docker container, and cloud VM relies on virtual networking primitives the kernel provides
- **Debugging production issues**: TIME_WAIT accumulation, SYN floods, buffer exhaustion — these are OS-level problems that require OS-level understanding
- **High-performance systems**: CDNs, trading platforms, and DDoS mitigation systems push beyond the kernel's default path — you need to know what they are bypassing and why

## Prerequisites

| Section | Why You Need It |
|---------|----------------|
| [01 - System Calls and Kernel](../01-System-Calls-and-Kernel/) | Socket operations are syscalls — you need to understand the user/kernel boundary |
| [10 - I/O Systems](../10-IO-Systems/) | I/O multiplexing (epoll, io_uring) is how servers handle thousands of connections |
| [12 - Virtualization and Containers](../12-Virtualization-and-Containers/) | Network namespaces are the foundation of container networking |

## Reading Order

| # | File | Topic | Key Concepts |
|---|------|-------|-------------|
| 1 | [Network Stack in the Kernel](01-network-stack-in-the-kernel.md) | How packets flow through the kernel | sk_buff, softirq, netfilter, conntrack, TCP states |
| 2 | [Sockets API — TCP and UDP](02-sockets-api-tcp-udp.md) | The programming interface for network I/O | socket lifecycle, backlog queues, socket options, SYN floods |
| 3 | [Network Namespaces and Virtual Networking](03-network-namespaces-and-virtual-networking.md) | How containers get isolated networks | veth pairs, bridges, VXLAN, eBPF, CNI |
| 4 | [Zero-Copy I/O and Kernel Bypass](04-zero-copy-io-and-kernel-bypass.md) | Eliminating copies and the kernel from the data path | sendfile, DPDK, XDP, AF_XDP |

## Key Interview Questions

1. **Trace the path of an incoming TCP packet from the NIC to the application.** Interviewers want to see you walk through: NIC → DMA → interrupt → driver → softirq → IP processing → TCP processing → socket buffer → recv(). Each layer matters.

2. **Why does TIME_WAIT exist and how does it affect high-traffic servers?** This tests whether you understand TCP connection teardown and the practical impact of ephemeral port exhaustion on servers handling many short-lived connections.

3. **How does Docker container networking work under the hood?** Explain veth pairs, Linux bridges, iptables NAT rules, and how packets flow from one container to another — or from a container to the outside world.

4. **What is zero-copy I/O and when would you use it?** Compare traditional I/O (4 copies) with sendfile (2 copies) and kernel bypass (0 kernel copies). Connect to real systems: nginx static serving, CDNs, high-frequency trading.

5. **What is the difference between DPDK and XDP?** Both bypass the traditional kernel stack, but DPDK moves everything to user space while XDP runs eBPF programs at the driver level. Different trade-offs in flexibility, complexity, and integration.
