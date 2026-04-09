# Virtualization Concepts and Hypervisors

## What Is Virtualization?

Virtualization is the act of creating a **virtual version of something** — in our case, an entire computer. A virtual machine (VM) is a software-based computer that runs its own operating system and applications as if it were a separate physical machine. The guest OS inside a VM doesn't know (or care) that it's not running on real hardware.

The key insight: if you can make a piece of software look like hardware, you can run multiple operating systems on a single physical machine simultaneously, each fully isolated from the others.

```
Physical Machine WITHOUT Virtualization:
┌─────────────────────────────────┐
│         Application             │
├─────────────────────────────────┤
│        Operating System         │
├─────────────────────────────────┤
│          Hardware               │
└─────────────────────────────────┘
   One OS owns all the hardware

Physical Machine WITH Virtualization:
┌──────────┐ ┌──────────┐ ┌──────────┐
│  App A   │ │  App B   │ │  App C   │
├──────────┤ ├──────────┤ ├──────────┤
│ Guest OS │ │ Guest OS │ │ Guest OS │
│ (Linux)  │ │(Windows) │ │ (Linux)  │
├──────────┴─┴──────────┴─┴──────────┤
│     Hypervisor / VMM               │
├─────────────────────────────────────┤
│           Hardware                  │
└─────────────────────────────────────┘
   Multiple OSes share one machine
```

## The Hypervisor (Virtual Machine Monitor)

The **hypervisor** (also called a Virtual Machine Monitor or VMM) is the software layer that creates and manages virtual machines. It has two core responsibilities:

1. **Multiplexing hardware** — sharing CPU, memory, and I/O among multiple VMs
2. **Isolation** — ensuring one VM cannot see or interfere with another

There are two fundamental architectures for hypervisors.

### Type 1: Bare-Metal Hypervisor

A Type 1 hypervisor runs **directly on the hardware**, with no host operating system underneath. It IS the operating system, but its job is managing VMs instead of running user applications.

```
Type 1 (Bare-Metal) Architecture:
┌────────┐ ┌────────┐ ┌────────┐
│  VM 1  │ │  VM 2  │ │  VM 3  │
│ Guest  │ │ Guest  │ │ Guest  │
│  OS    │ │  OS    │ │  OS    │
├────────┴─┴────────┴─┴────────┤
│      Type 1 Hypervisor       │
│   (runs directly on HW)     │
├──────────────────────────────┤
│        Hardware              │
└──────────────────────────────┘
```

**Examples:**
| Hypervisor | Notes |
|---|---|
| VMware ESXi | Dominant in enterprise data centers |
| Microsoft Hyper-V | Built into Windows Server |
| Xen | Pioneer of paravirtualization, used by early AWS |
| KVM | Linux kernel module — Linux itself becomes the hypervisor |

KVM is interesting because it blurs the line. Linux is a general-purpose OS, but when the KVM module is loaded, the Linux kernel gains hypervisor capabilities. The kernel runs in VMX root mode (more on this in the next section), and each VM is a regular Linux process managed by QEMU. This makes KVM technically a Type 1 hypervisor even though Linux is doing double duty.

### Type 2: Hosted Hypervisor

A Type 2 hypervisor runs **on top of a conventional host operating system**, just like any other application. It relies on the host OS for hardware access.

```
Type 2 (Hosted) Architecture:
┌────────┐ ┌────────┐
│  VM 1  │ │  VM 2  │    ┌─────────┐
│ Guest  │ │ Guest  │    │ Normal  │
│  OS    │ │  OS    │    │  Apps   │
├────────┴─┴────────┤    │         │
│  Type 2 Hypervisor│    │         │
│  (application)    │    │         │
├───────────────────┴────┴─────────┤
│         Host Operating System    │
├──────────────────────────────────┤
│           Hardware               │
└──────────────────────────────────┘
```

**Examples:**
| Hypervisor | Notes |
|---|---|
| VirtualBox | Free, popular for development and learning |
| VMware Workstation | Commercial, feature-rich |
| Parallels | macOS-focused, runs Windows on Mac |
| QEMU (without KVM) | Software emulation, very slow but can emulate different CPU architectures |

Type 2 hypervisors are easier to set up (just install an app) but have more overhead because every hardware access goes through the host OS. They're great for development and testing but not for production server workloads.

## Full Virtualization vs. Paravirtualization

How does the hypervisor handle the guest OS trying to execute privileged instructions?

### Full Virtualization: Trap and Emulate

In full virtualization, the guest OS runs **unmodified** — it doesn't know it's in a VM. When the guest tries to execute a privileged instruction (like modifying page tables or accessing I/O), the CPU traps to the hypervisor, which emulates the instruction and returns control.

```
Full Virtualization (Trap and Emulate):

Guest OS executes privileged instruction
         │
         ▼
   ┌───────────┐
   │ CPU Trap  │  ← Hardware detects privileged
   └─────┬─────┘    instruction from unprivileged mode
         │
         ▼
   ┌───────────────┐
   │  Hypervisor   │  ← Emulates the instruction,
   │  handles trap │    updates virtual hardware state
   └─────┬─────────┘
         │
         ▼
   Return to guest OS
   (guest never knew it was intercepted)
```

**Advantage:** Guest OS needs no modification. Run any OS as-is.
**Disadvantage:** Every privileged instruction causes a trap — expensive context switches.

### Paravirtualization: Hypercalls

In paravirtualization, the guest OS is **modified** to know it's running in a VM. Instead of executing privileged instructions that would trap, the guest explicitly calls the hypervisor through **hypercalls** — like system calls, but to the hypervisor instead of the kernel.

```
Paravirtualization (Hypercalls):

Guest OS needs privileged operation
         │
         ▼
   ┌──────────────┐
   │  Hypercall   │  ← Guest explicitly calls
   │  (like a     │    hypervisor API
   │  syscall)    │
   └──────┬───────┘
          │
          ▼
   ┌───────────────┐
   │  Hypervisor   │  ← Performs the operation
   │  handles call │    directly (no trap overhead)
   └──────┬────────┘
          │
          ▼
   Return to guest OS
```

**Advantage:** Much faster — no trap overhead, batched operations possible.
**Disadvantage:** Guest OS must be modified. Can't run unmodified Windows.

Xen pioneered this approach. Linux guests were modified to make hypercalls, and performance was excellent. However, with hardware-assisted virtualization (next section), the performance gap has largely closed, and full virtualization with hardware support is now the dominant approach.

## Why Virtualize?

| Use Case | How Virtualization Helps |
|---|---|
| Server consolidation | Run 10 VMs on one server instead of buying 10 servers at 5% utilization |
| Isolation | A crash or security breach in one VM doesn't affect others |
| Development/testing | Spin up fresh environments instantly, snapshot and roll back |
| Cloud infrastructure | AWS sells VMs (EC2 instances) — virtualization IS the product |
| Legacy support | Run old OS versions that need specific hardware |
| Live migration | Move a running VM between physical hosts with zero downtime |

## The Popek and Goldberg Requirements

In 1974, Popek and Goldberg formalized what it takes for a CPU architecture to support virtualization efficiently. A CPU is **virtualizable** if it satisfies three properties:

1. **Equivalence** — a program running in a VM behaves identically to running on bare metal
2. **Resource control** — the hypervisor has complete control over all hardware resources
3. **Efficiency** — the vast majority of instructions execute directly on the CPU without hypervisor intervention

The critical requirement: **all sensitive instructions must be privileged instructions**. A "sensitive" instruction is one that behaves differently depending on privilege level or affects the system's resource allocation. If it's privileged, it will trap when executed by a guest, letting the hypervisor intercept it.

The x86 architecture famously violated this requirement — some sensitive instructions would silently succeed in user mode instead of trapping. This made x86 "non-virtualizable" by the classical definition and led to creative workarounds (binary translation) and eventually hardware extensions (VT-x/AMD-V). We'll cover these in the next section.

## Real-World Connection

When you launch an EC2 instance on AWS, you're getting a VM running on a Type 1 hypervisor. AWS originally used Xen (paravirtualization) but migrated to KVM with their custom **Nitro** hypervisor. Google Cloud uses KVM. Azure uses a customized Hyper-V. Every major cloud provider is essentially a hypervisor company that happens to sell compute time.

When you install VirtualBox on your laptop to test a Linux distribution, you're using a Type 2 hypervisor. It's convenient but slower — which is fine for development but not for running production workloads.

## Interview Angle

**Q: What is the difference between a Type 1 and Type 2 hypervisor?**
A: A Type 1 hypervisor runs directly on the hardware (bare-metal) and manages VMs without a host OS — examples are ESXi, KVM, and Hyper-V. A Type 2 hypervisor runs as an application on top of a host OS — examples are VirtualBox and VMware Workstation. Type 1 has less overhead and is used in production/cloud environments. Type 2 is convenient for development.

**Q: Where does KVM fit — Type 1 or Type 2?**
A: KVM is a Type 1 hypervisor implemented as a Linux kernel module. When loaded, the Linux kernel itself becomes the hypervisor. Each VM runs as a regular Linux process (managed by QEMU), but the kernel is in direct control of hardware virtualization. It's Type 1 because there's no separate host OS layer between KVM and the hardware.

**Q: What is the difference between full virtualization and paravirtualization?**
A: In full virtualization, the guest OS runs unmodified — privileged instructions trap to the hypervisor for emulation. In paravirtualization, the guest OS is modified to make explicit hypercalls to the hypervisor, avoiding trap overhead. Paravirtualization was faster historically, but hardware-assisted virtualization has made full virtualization nearly as fast, so it's now the dominant approach.

**Q: Why couldn't x86 be virtualized efficiently using trap-and-emulate alone?**
A: Because x86 violated the Popek-Goldberg requirements — some sensitive instructions (like `SGDT`, `SIDT`, `POPF`) didn't trap when executed in user mode. They silently returned different results instead of faulting, so the hypervisor couldn't intercept them. This required workarounds like binary translation until Intel VT-x and AMD-V added hardware support.

---

Next: [Hardware-Assisted Virtualization](02-hardware-assisted-virtualization.md)
