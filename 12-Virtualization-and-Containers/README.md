# Virtualization and Containers

Modern computing runs on a simple but powerful idea: **don't waste hardware**. A single physical server sitting at 5% CPU utilization is a costly mistake. Virtualization solves this by letting multiple operating systems share one machine, each believing it has the hardware to itself. Containers take this further — instead of running entire OSes, they isolate individual applications using the kernel's own mechanisms, achieving similar isolation at a fraction of the cost.

Together, virtualization and containers form the backbone of cloud computing. Every EC2 instance is a virtual machine. Every microservice deployment likely runs in a container. Understanding how these technologies work — from hypervisors down to Linux namespaces — is essential for any engineer working with modern infrastructure.

## Why This Matters

- **Cloud computing IS virtualization** — AWS, GCP, and Azure are massive virtualization platforms
- **Containers are how modern applications ship** — Docker and Kubernetes dominate deployment
- **Performance debugging requires understanding the stack** — is your latency from the hypervisor, the container runtime, or your code?
- **Security boundaries differ dramatically** — VMs and containers offer very different isolation guarantees
- **Interview staple** — "explain the difference between a VM and a container" is one of the most common systems questions

## Prerequisites

Before diving in, make sure you're comfortable with:

| Prerequisite | Why You Need It |
|---|---|
| [01 - System Calls and Kernel](../01-System-Calls-and-Kernel/) | Virtualization intercepts system calls and privileged operations |
| [02 - Processes](../02-Processes/) | Containers ARE processes with extra isolation |
| [07 - Memory Management](../07-Memory-Management/) | VMs require translating multiple levels of virtual addresses |
| [11 - Security and Protection](../11-Security-and-Protection/) | Protection rings and privilege levels are central to how hypervisors work |

## Reading Order

| # | Topic | What You'll Learn |
|---|---|---|
| 1 | [Virtualization Concepts and Hypervisors](01-virtualization-concepts-and-hypervisors.md) | What virtualization is, Type 1 vs Type 2 hypervisors, full vs paravirtualization |
| 2 | [Hardware-Assisted Virtualization](02-hardware-assisted-virtualization.md) | Intel VT-x/AMD-V, how CPUs natively support VMs, EPT/NPT address translation |
| 3 | [OS-Level Virtualization: Namespaces and Cgroups](03-os-level-virtualization-namespaces-cgroups.md) | Linux namespaces, control groups — the kernel primitives that make containers possible |
| 4 | [Containers: Docker and containerd](04-containers-docker-containerd.md) | Container images, runtimes, the Docker stack, alternative runtimes |
| 5 | [VM vs Container Tradeoffs](05-vm-vs-container-tradeoffs.md) | When to use which, hybrid approaches, microVMs, the isolation spectrum |
| 6 | [Orchestration and Kubernetes Basics](06-orchestration-kubernetes-basics.md) | Why orchestration exists, Kubernetes as a "distributed OS," mapping OS concepts to K8s |

## Key Interview Questions

1. **What is the difference between a Type 1 and Type 2 hypervisor?** Where does KVM fit?
2. **Why couldn't x86 be virtualized efficiently before Intel VT-x/AMD-V?** What was the workaround?
3. **Explain how Linux namespaces and cgroups work together to create containers.** What does each provide?
4. **What is the difference between a virtual machine and a container?** When would you choose one over the other?
5. **What happens when a container exceeds its memory limit?** How does this differ from a VM exceeding its allocated RAM?
6. **How does Kubernetes use OS-level primitives** (namespaces, cgroups, iptables) to manage containers at scale?
