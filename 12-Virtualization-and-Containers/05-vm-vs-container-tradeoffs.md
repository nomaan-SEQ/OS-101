# VM vs Container Tradeoffs

## The Fundamental Difference

Virtual machines virtualize **hardware**. Each VM runs its own kernel. Containers virtualize **the OS**. All containers share the host kernel.

This single architectural difference cascades into every tradeoff: isolation, performance, density, security, and portability.

```
Virtual Machine:                         Container:
┌──────────────────┐                     ┌──────────────────┐
│   Application    │                     │   Application    │
├──────────────────┤                     ├──────────────────┤
│   Libraries      │                     │   Libraries      │
├──────────────────┤                     └────────┬─────────┘
│   Guest Kernel   │ ← Separate kernel           │
├──────────────────┤                              │ Shared kernel
│   Virtual HW     │                              │
├──────────────────┤                     ┌────────┴─────────┐
│   Hypervisor     │                     │   Host Kernel    │
├──────────────────┤                     ├──────────────────┤
│   Hardware       │                     │   Hardware       │
└──────────────────┘                     └──────────────────┘

More layers = more isolation                Fewer layers = less overhead
but more overhead                           but shared attack surface
```

## Comprehensive Comparison

| Dimension | Virtual Machine | Container |
|---|---|---|
| **Isolation mechanism** | Separate kernel + hypervisor | Shared kernel + namespaces/cgroups |
| **Isolation strength** | Strong — kernel boundary | Weaker — kernel is shared |
| **Resource overhead** | Heavy — full OS per VM (hundreds of MB to GBs) | Light — shared kernel, only app + libs (MBs) |
| **Boot time** | Seconds to minutes (full OS boot) | Milliseconds to seconds (just start a process) |
| **Density** | Tens of VMs per host | Hundreds or thousands of containers per host |
| **Guest OS** | Any OS (Linux, Windows, BSD) | Same OS family as host kernel |
| **Kernel version** | Each VM has its own kernel | All containers share host kernel version |
| **Security boundary** | Hardware-enforced (VMX root/non-root) | Software-enforced (namespaces, seccomp, capabilities) |
| **Performance overhead** | 2-10% (with hardware assist) | ~0% for CPU, slight overhead for networking/storage |
| **Image size** | GBs (full OS + app) | MBs to hundreds of MBs (app + libs only) |
| **Portability** | Less portable (hypervisor-specific formats) | Highly portable (OCI standard, runs anywhere) |
| **Live migration** | Supported (move running VM between hosts) | Harder (checkpoint/restore via CRIU, less mature) |
| **Hardware access** | Full virtual hardware (GPU, USB, etc.) | Limited (needs device plugins or privileged mode) |
| **Ecosystem maturity** | Decades of production use | ~10 years of widespread production use |

## When to Use Virtual Machines

VMs are the right choice when you need:

**Strong multi-tenant isolation.** If you're a cloud provider running untrusted customer workloads on shared hardware, you need the hardware-enforced isolation boundary that VMs provide. A kernel vulnerability in one customer's workload cannot affect another customer's VM — they have completely separate kernels.

**Different operating systems.** Need to run Windows alongside Linux? A Windows VM on a Linux host (or vice versa) requires a VM — containers can only run the same OS family as the host kernel.

**Specific kernel requirements.** If your application needs a particular kernel version, kernel modules, or custom kernel parameters, it needs its own kernel — which means a VM.

**Legacy workloads.** Older applications that assume they're the only thing running on a machine, or that need specific hardware configurations, are easier to encapsulate in a VM than to containerize.

**Compliance requirements.** Some regulatory frameworks (healthcare, finance, government) require VM-level isolation between workloads. "Containers share a kernel" is a hard sell to some auditors.

## When to Use Containers

Containers are the right choice when you need:

**Microservices at scale.** Running 200 microservices as 200 VMs wastes enormous resources on 200 duplicate kernels. As containers, they share one kernel and use a fraction of the memory.

**Fast iteration.** Build a container image in seconds, push it, deploy it. VM images take minutes to build and deploy. For CI/CD pipelines, containers are dramatically faster.

**Consistent environments.** "Works on my machine" disappears when development, testing, staging, and production all run the exact same container image.

**High density.** A server that runs 20 VMs can run 200+ containers. The cost savings are significant at scale.

**Identical OS requirements.** If all your services run on Linux, they can all share one kernel. There's no need for per-service kernels.

## The Hybrid Approach: Containers Inside VMs

In practice, most production environments use both. This is how the major cloud Kubernetes services work:

```
Typical Cloud Kubernetes Setup:

┌──────────────────────────────────────────────────┐
│                Physical Server                   │
├──────────────────────────────────────────────────┤
│                  Hypervisor                      │
├──────────────┬───────────────┬───────────────────┤
│    VM (Node) │    VM (Node)  │    VM (Node)      │
│  ┌─────────┐ │  ┌─────────┐  │  ┌─────────┐     │
│  │Container│ │  │Container│  │  │Container│     │
│  │Container│ │  │Container│  │  │Container│     │
│  │Container│ │  │Container│  │  │Container│     │
│  ├─────────┤ │  ├─────────┤  │  ├─────────┤     │
│  │  Kernel │ │  │  Kernel │  │  │  Kernel │     │
└──┴─────────┴─┴──┴─────────┴──┴──┴─────────┴─────┘

  VMs provide tenant isolation (you can't escape to my VM)
  Containers provide density within each VM
```

- **VMs isolate tenants** — customer A's Kubernetes cluster runs in VMs separate from customer B's
- **Containers isolate applications** — within customer A's VMs, individual microservices run as containers
- Best of both worlds: strong isolation boundaries where needed, high density where safe

AWS EKS, Google GKE, and Azure AKS all follow this pattern. Your "nodes" are VMs, and your pods are containers inside those VMs.

## Bridging the Gap: MicroVMs and Sandboxed Containers

The clean VM-vs-container dichotomy is blurring. Several technologies sit in between:

### Firecracker: MicroVMs

Firecracker is a Virtual Machine Monitor (VMM) built by AWS, designed to launch VMs that are as lightweight as containers:

- Boots a minimal Linux kernel in **~125 milliseconds**
- Uses **~5 MB of memory** for the VMM itself
- Provides full VM isolation (separate kernel, KVM-backed)
- Powers **AWS Lambda** and **AWS Fargate**

The idea: when a Lambda function runs, it gets its own microVM. This provides VM-level isolation between different customers' functions, but with container-like startup speed.

### Kata Containers: VM Isolation, Container API

Kata Containers implements the OCI runtime spec (like runc) but instead of creating namespaces, it boots a lightweight VM for each container:

```
Standard Container (runc):         Kata Container:
┌──────────────────────┐           ┌──────────────────────┐
│   Application        │           │   Application        │
├──────────────────────┤           ├──────────────────────┤
│   Namespaces/Cgroups │           │   Guest Kernel       │
├──────────────────────┤           ├──────────────────────┤
│   Host Kernel        │           │   Lightweight VM     │
└──────────────────────┘           ├──────────────────────┤
                                   │   Host Kernel + KVM  │
                                   └──────────────────────┘

Same docker/kubectl commands,      Same docker/kubectl commands,
namespace isolation                VM isolation
```

From the user's perspective, nothing changes — you use `docker run` or `kubectl` the same way. But underneath, each container gets its own kernel and the isolation guarantee of a VM.

### gVisor: User-Space Kernel

Google's gVisor takes yet another approach: intercept all system calls from the container and handle them in a user-space kernel (called "Sentry"), which then makes a limited set of calls to the real kernel.

```
Standard Container:              gVisor Container:
App → syscall → Host Kernel      App → syscall → Sentry (user-space)
                                                    │
                                                    ▼
                                              Host Kernel
                                              (limited syscalls)
```

The attack surface is dramatically reduced — the container never directly touches the host kernel. gVisor powers Google Cloud Run.

## The Isolation Spectrum

```
The Full Spectrum:

  Bare Metal     VM           MicroVM        Kata         gVisor       Container
  ──────────────────────────────────────────────────────────────────────────────►
  No isolation   Full kernel  Fast-boot VM   VM per       User-space   Namespace
                 separation   (~125ms boot)  container    kernel       isolation

  ◄──── Stronger isolation                          Weaker isolation ────►
  ◄──── More overhead                               Less overhead ────►
  ◄──── Slower startup                              Faster startup ────►
  ◄──── Lower density                               Higher density ────►
```

There's no universally "better" option. The right choice depends on your threat model, performance requirements, and operational constraints.

| If Your Priority Is... | Choose |
|---|---|
| Maximum isolation between untrusted tenants | VMs or MicroVMs (Firecracker) |
| Running different OS families | VMs |
| Maximum density and fastest deployment | Standard containers (runc/crun) |
| Container UX with stronger isolation | Kata Containers or gVisor |
| Serverless functions at scale | MicroVMs (Firecracker) |
| Development and CI/CD | Standard containers |
| Regulated environments with strict compliance | VMs (possibly with containers inside) |

## Real-World Connection

**AWS runs both.** EC2 instances are VMs (KVM/Nitro). Lambda functions run in Firecracker microVMs. ECS/EKS containers run inside EC2 VMs. AWS chose the right isolation level for each product.

**Google Cloud Run** uses gVisor to run customer containers with stronger isolation than standard namespaces, accepting the performance trade-off for multi-tenant security.

**Most Kubernetes clusters** run containers inside VMs. When you create a GKE cluster, each node is a VM. Pods are containers inside those VMs. You get VM isolation between clusters and container density within clusters.

**Your laptop** probably uses containers for development (Docker Desktop) and VMs for testing different OSes (VirtualBox/Parallels). Docker Desktop on Mac/Windows actually runs a Linux VM to host the containers — because containers need a Linux kernel.

## Interview Angle

**Q: What is the fundamental difference between a VM and a container?**
A: A VM virtualizes hardware — each VM runs its own kernel and appears to be a complete computer. A container virtualizes the OS — containers share the host kernel and are isolated using namespaces (visibility) and cgroups (resource limits). VMs provide stronger isolation but higher overhead. Containers are lighter and faster but share the kernel attack surface.

**Q: When would you choose a VM over a container?**
A: When you need strong isolation between untrusted tenants (cloud multi-tenancy), when you need different operating systems (Windows + Linux), when compliance requires kernel-level separation, when you need specific kernel versions or modules, or when running legacy workloads that assume dedicated hardware. In practice, most cloud environments use both — VMs for tenant isolation, containers for application density within each tenant's VMs.

**Q: What is Firecracker and why does it exist?**
A: Firecracker is a lightweight VMM built by AWS that boots microVMs in ~125ms with ~5MB overhead. It exists because AWS Lambda needs to run untrusted customer code with VM-level isolation (customers can't see each other's data) but with container-like speed (functions start in milliseconds, not minutes). Traditional VMs were too slow to boot; containers didn't provide strong enough isolation for multi-tenant serverless.

**Q: Why do most Kubernetes clusters run containers inside VMs?**
A: It combines the strengths of both technologies. VMs provide strong hardware-enforced isolation between different tenants or clusters — a kernel vulnerability in one VM can't affect another. Containers provide high density and fast deployment within each VM. Cloud Kubernetes services (EKS, GKE, AKS) all use this pattern: nodes are VMs, pods are containers inside those VMs.

---

Next: [Orchestration and Kubernetes Basics](06-orchestration-kubernetes-basics.md)
