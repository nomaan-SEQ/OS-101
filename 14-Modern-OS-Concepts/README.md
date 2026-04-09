# 14 - Modern OS Concepts

The operating system is not a static artifact. Every decade brings new application patterns that stress old abstractions and demand new ones. Monoliths gave way to microservices, virtual machines gave way to containers, and now containers may give way to WebAssembly modules. Each shift redraws the boundary between "what the application does" and "what the OS provides." This section explores where that boundary is moving -- and why it matters to every engineer building systems today.

This is the capstone of the theoretical sections. Everything you have learned -- processes, memory, filesystems, networking, virtualization -- comes together here as building blocks for the next generation of OS design.

---

## Why This Matters

Modern systems interviews rarely ask you to implement `malloc` from scratch. They ask you to explain **why a Kubernetes pod can talk to another pod on a different node**, or **how eBPF lets you observe production systems without restarting anything**, or **why WebAssembly might replace containers for some workloads**. These topics sit at the intersection of OS fundamentals and distributed systems -- exactly where senior engineering conversations happen.

Understanding modern OS concepts helps you:
- **Design systems** that work with the OS instead of fighting it
- **Debug production issues** that span the kernel, container runtime, and orchestrator
- **Evaluate new technologies** by understanding what OS primitives they build on
- **Answer interview questions** that distinguish senior engineers from junior ones

---

## Prerequisites

This section builds on everything before it. You should be comfortable with:

| Section | Why You Need It |
|---|---|
| [02 - Process Management](../02-Process-Management/README.md) | Microservices multiply processes; scheduling matters more |
| [03 - Memory Management](../03-Memory-Management/README.md) | Unikernels and Wasm rethink address spaces |
| [06 - File Systems](../06-File-Systems/README.md) | Distributed file systems extend local FS concepts |
| [08 - I/O Systems](../08-IO-Systems/README.md) | eBPF hooks into I/O paths; XDP processes packets |
| [09 - Networking Fundamentals](../09-Networking-Fundamentals/README.md) | Service meshes, overlay networks, RDMA |
| [12 - Virtualization and Containers](../12-Virtualization-and-Containers/README.md) | **Essential** -- containers are the baseline for everything here |
| [13 - Security](../13-Security/README.md) | eBPF security, Wasm sandboxing, capability-based models |

If you skipped Section 12, go back and read it first. Containers are the assumed starting point for every topic in this section.

---

## Reading Order

| # | Topic | What You Will Learn |
|---|---|---|
| 01 | [Microservices and OS Implications](01-microservices-and-os-implications.md) | How microservice architectures change demands on the kernel |
| 02 | [eBPF and Kernel Observability](02-ebpf-and-kernel-observability.md) | Running sandboxed programs inside the kernel for tracing, networking, and security |
| 03 | [Unikernels and Library OS](03-unikernels-and-library-os.md) | Compiling your app and a minimal OS into a single image |
| 04 | [Wasm Runtimes and the OS Boundary](04-wasm-runtimes-and-os-boundary.md) | WebAssembly as a new isolation and portability layer |
| 05 | [OS for Distributed Systems](05-os-for-distributed-systems.md) | Treating a cluster of machines like a single computer |

Read them in order. Each topic builds on the mental model established by the previous one: microservices create the problem, eBPF observes it, unikernels and Wasm offer alternative isolation models, and distributed OS concepts tie it all together at cluster scale.

---

## Key Interview Questions

1. **"How does a microservice architecture change the demands on the Linux kernel compared to a monolithic application?"**
   Think: more processes, more context switches, more sockets, more network traffic (east-west), kernel parameter tuning.

2. **"What is eBPF and why is it significant for production observability?"**
   Think: sandboxed kernel programs, no kernel modification, attach to hook points, verifier ensures safety, used by Cilium/Falco/bpftrace.

3. **"Compare containers, unikernels, and WebAssembly modules as isolation mechanisms."**
   Think: isolation granularity, startup time, image size, attack surface, debugging difficulty, maturity.

4. **"What does it mean to treat a cluster as a single computer? What OS abstractions have distributed equivalents?"**
   Think: scheduler -> Kubernetes scheduler, filesystem -> distributed FS, memory -> distributed cache, IPC -> service mesh.

5. **"If you were designing a serverless platform today, would you use containers, unikernels, or Wasm? Why?"**
   Think: cold start requirements, security model, developer experience, ecosystem maturity, workload characteristics.
