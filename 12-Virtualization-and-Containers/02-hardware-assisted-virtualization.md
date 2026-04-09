# Hardware-Assisted Virtualization

## The x86 Virtualization Problem

Classical trap-and-emulate virtualization requires that all sensitive instructions cause a trap when executed by unprivileged code. The x86 architecture didn't cooperate. Several instructions behaved differently in user mode (Ring 3) but **didn't trap** — they silently returned different results or were simply ignored.

For example:
- `SGDT` / `SIDT` — store the descriptor table register. Works in any ring but reveals whether the OS is running on real hardware or in a VM (the hypervisor relocates these tables).
- `POPF` — pops flags from the stack. In Ring 0, it modifies the interrupt flag. In Ring 3, it silently ignores the interrupt flag change — no trap, no error.
- `PUSHF` — pushes flags including the I/O privilege level, which differs between real Ring 0 and virtualized Ring 0.

These are **sensitive but not privileged** instructions — exactly what Popek and Goldberg said would break virtualization.

### The Binary Translation Workaround

VMware's breakthrough solution (late 1990s): **binary translation**. Before executing guest code, the hypervisor scans the instruction stream and replaces problematic instructions with safe equivalents that trap or emulate correctly.

```
Binary Translation:

  Guest code (original):      Guest code (translated):
  ┌──────────────────┐        ┌──────────────────────┐
  │ mov eax, 1       │   →    │ mov eax, 1           │  (safe, run as-is)
  │ popf             │   →    │ call vmm_handle_popf │  (replaced!)
  │ sgdt [mem]       │   →    │ call vmm_handle_sgdt │  (replaced!)
  │ add ebx, ecx     │   →    │ add ebx, ecx         │  (safe, run as-is)
  └──────────────────┘        └──────────────────────┘
```

This worked — VMware ran unmodified guest OSes on x86 — but it was slow. Scanning and rewriting code at runtime is expensive, and the translated code runs slower than native instructions.

## The Hardware Solution: Intel VT-x and AMD-V

In 2005-2006, Intel (VT-x, codenamed "Vanderpool") and AMD (AMD-V, codenamed "Pacifica") independently added hardware virtualization extensions to their CPUs. These extensions fundamentally redesigned how the CPU handles virtualization.

### VMX Root and VMX Non-Root Modes

The key idea: create **two separate worlds** for the CPU to operate in.

```
CPU Privilege Modes WITH VT-x:

┌─────────────────────────────────────────┐
│            VMX Root Mode                │
│         (Hypervisor World)              │
│                                         │
│  Ring 0 ─── Hypervisor kernel           │
│  Ring 3 ─── Hypervisor userspace        │
│             (e.g., QEMU)               │
├─────────────────────────────────────────┤
│          VMX Non-Root Mode              │
│           (Guest World)                 │
│                                         │
│  Ring 0 ─── Guest OS kernel             │
│  Ring 3 ─── Guest applications          │
│                                         │
│  Privileged instructions → VM Exit      │
│  to hypervisor (VMX Root)               │
└─────────────────────────────────────────┘
```

- **VMX root mode**: where the hypervisor runs. Full hardware access. All instructions work normally.
- **VMX non-root mode**: where the guest runs. The guest OS runs in Ring 0 — it thinks it has full privilege — but the CPU intercepts sensitive operations and exits to the hypervisor.

> **Analogy:** VMX root and non-root modes are like a puppet theater. The hypervisor (puppeteer, VMX root) has full control of the stage. The guest OS (puppet, VMX non-root) believes it's performing independently, but whenever it tries to do something privileged (like accessing hardware), the strings pull it back to the puppeteer (VM exit) who handles it and returns control (VM entry).

This solves the x86 problem elegantly: the guest runs at Ring 0 (so all instructions behave as the guest expects), but the CPU has a hardware mechanism to trap to the hypervisor when needed.

### The VM Entry / VM Exit Cycle

The CPU constantly transitions between the hypervisor and the guest:

```
VM Entry / VM Exit Cycle:

    Hypervisor (VMX Root)          Guest (VMX Non-Root)
    ─────────────────────          ────────────────────
            │
            │  VM Entry
            │  (load guest state from VMCS)
            ├──────────────────────────►│
            │                           │  Guest executes
            │                           │  instructions
            │                           │  normally...
            │                           │
            │                           │  Privileged instruction
            │                           │  or external interrupt
            │         VM Exit           │
            │◄──────────────────────────┤
            │  (save guest state,       │
            │   load host state)        │
            │                           │
    Hypervisor handles the              │
    exit reason, then...               │
            │                           │
            │  VM Entry                 │
            ├──────────────────────────►│
            │                           │  Guest resumes
                    ... and so on
```

**VM Entry**: hypervisor loads guest state and transfers control to the guest. The CPU switches to VMX non-root mode.

**VM Exit**: the CPU detects a condition that requires hypervisor attention (privileged instruction, I/O access, interrupt, page fault) and transfers control back. The CPU saves guest state, loads hypervisor state, and switches to VMX root mode.

The hypervisor examines the exit reason, handles it (emulate the instruction, inject a virtual interrupt, etc.), and performs another VM Entry to resume the guest.

### The VMCS (Virtual Machine Control Structure)

Each VM has a **VMCS** — a hardware-defined data structure in memory that stores everything the CPU needs for VM Entry and VM Exit:

| VMCS Field | Purpose |
|---|---|
| Guest state area | Guest registers, CR3, segment registers, instruction pointer |
| Host state area | Hypervisor registers to restore on VM Exit |
| VM-execution controls | What triggers a VM Exit (which instructions, which interrupts) |
| VM-exit controls | What to save/restore on exit |
| VM-entry controls | What to restore on entry, event injection |
| Exit information | Why the VM Exit happened (exit reason, qualification) |

The VMCS is a per-VM structure. The hypervisor uses `VMPTRLD` to load a VMCS and `VMREAD`/`VMWRITE` to access its fields. This hardware-managed state makes VM transitions fast — the CPU knows exactly what to save and restore.

## Second Level Address Translation (SLAT)

VMs introduce **two levels of address translation**. The guest OS maintains its own page tables (guest virtual to guest physical), but the guest's "physical" addresses aren't real — they need to be translated to actual host physical addresses.

```
Address Translation with SLAT (EPT/NPT):

  Guest Process          Guest OS             Hypervisor/Hardware
  ─────────────          ────────             ───────────────────

  Guest Virtual    Guest Page Tables     EPT/NPT Tables
  Address (GVA) ──────────────────► Guest Physical  ──────────────► Host Physical
                                    Address (GPA)                   Address (HPA)

  0x7fff1000  ────► 0x40000  ─────────────────────────► 0x1A80000

  Two levels of translation, both handled in hardware
```

### Without SLAT: Shadow Page Tables

Before hardware SLAT, the hypervisor maintained **shadow page tables** — a single set of tables that mapped directly from guest virtual to host physical. Every time the guest modified its page tables, the hypervisor had to update the shadow copy. This was extremely expensive:

- Every guest page table write caused a VM Exit
- The hypervisor had to decode the write, update the shadow tables, then resume
- Page table-heavy workloads (process creation, context switches) were painfully slow

### With SLAT: EPT (Intel) and NPT (AMD)

Hardware SLAT adds a second page table walker to the CPU:

| Technology | Vendor | How It Works |
|---|---|
| Extended Page Tables (EPT) | Intel | CPU walks guest page tables (GVA→GPA), then walks EPT tables (GPA→HPA) |
| Nested Page Tables (NPT) | AMD | Same concept, different implementation |

The guest manages its own page tables freely — no VM Exits for page table modifications. The CPU handles both levels of translation in hardware with a **two-dimensional page walk**. On a TLB miss, the CPU walks both tables and caches the final GVA→HPA mapping.

The downside: a TLB miss is more expensive (the CPU may need to walk up to 24 memory references for a 4-level guest + 4-level EPT table). But TLB hits are identical to native, and misses are still far cheaper than shadow page table exits.

## I/O Virtualization

I/O is the last major piece. VMs need access to disks, networks, and other devices, but you can't give multiple VMs direct hardware access without chaos.

### Approaches to I/O Virtualization

| Approach | How It Works | Performance |
|---|---|---|
| **Emulated devices** | Hypervisor presents a fake device (e.g., emulated Intel e1000 NIC). Guest uses standard drivers. Every I/O operation traps to hypervisor. | Slow — many VM Exits |
| **Paravirtual I/O (virtio)** | Guest uses a special driver that knows it's in a VM. Communicates with hypervisor through shared memory rings, minimizing traps. | Good — fewer VM Exits |
| **Device passthrough** | A physical device is assigned directly to one VM using IOMMU (VT-d / AMD-Vi). The VM accesses real hardware. | Near-native — no hypervisor in data path |
| **SR-IOV** | A single physical device presents multiple "virtual functions" (VFs), each assignable to a different VM. Hardware-level multiplexing. | Near-native — hardware does the sharing |

**IOMMU (VT-d / AMD-Vi)**: the I/O equivalent of EPT. It translates device DMA addresses so a device assigned to a VM can only access that VM's memory. Without IOMMU, a passthrough device could DMA into any memory on the system — a massive security hole.

## Performance: How Close to Native?

With full hardware-assisted virtualization (VT-x + EPT + virtio or passthrough):

| Workload | Typical Overhead |
|---|---|
| CPU-bound computation | 1-2% (runs natively in VMX non-root) |
| Memory-intensive | 2-5% (EPT walk overhead on TLB misses) |
| I/O with virtio | 5-10% (shared memory rings, still some hypervisor involvement) |
| I/O with passthrough/SR-IOV | 1-3% (hardware handles it) |
| I/O with emulated devices | 20-50% (every operation traps) |

The takeaway: modern hardware-assisted virtualization achieves **near-native performance** for most workloads. The VM overhead that used to be 30-50% is now in the single digits.

## Real-World Connection

Every cloud VM you've ever used runs on hardware-assisted virtualization. When you launch an EC2 instance:

1. AWS's Nitro hypervisor (based on KVM) uses VT-x to create a VMX non-root environment for your VM
2. EPT maps your VM's "physical" memory to actual host RAM
3. Nitro offloads networking and storage to dedicated hardware (SR-IOV and custom ASICs), keeping the hypervisor out of the data path
4. The result: your VM runs at near-native speed, and AWS can pack hundreds of VMs per host efficiently

When you run a VirtualBox VM on your laptop, it also uses VT-x — that's why VirtualBox checks for hardware virtualization support on startup and warns you if it's disabled in BIOS.

## Interview Angle

**Q: Why couldn't x86 be efficiently virtualized before VT-x/AMD-V?**
A: x86 had sensitive instructions (like SGDT, POPF) that didn't trap when executed in Ring 3 — they silently behaved differently instead of causing a fault. This violated the Popek-Goldberg requirements. The hypervisor couldn't intercept these instructions using trap-and-emulate alone. VMware worked around this with binary translation (rewriting guest code at runtime), and Xen used paravirtualization (modifying the guest OS). Both worked but had significant overhead.

**Q: What happens during a VM Exit?**
A: The CPU saves the guest's state (registers, instruction pointer, etc.) into the VMCS, loads the hypervisor's state from the VMCS, and switches from VMX non-root to VMX root mode. The hypervisor then reads the exit reason from the VMCS, handles it (emulates an instruction, injects an interrupt, etc.), and eventually performs a VM Entry to resume the guest. VM Exits are expensive (hundreds to thousands of cycles), so minimizing them is critical for performance.

**Q: What is EPT/NPT and why does it matter?**
A: Extended Page Tables (Intel) and Nested Page Tables (AMD) provide hardware-accelerated two-level address translation for VMs. Without them, the hypervisor had to maintain shadow page tables — intercepting every guest page table modification, which caused many expensive VM Exits. With EPT/NPT, the guest manages its own page tables freely, and the CPU hardware walks both guest and host tables automatically. This dramatically reduces VM Exit frequency for memory-intensive workloads.

**Q: What is SR-IOV?**
A: Single Root I/O Virtualization. A hardware standard where a single physical device (like a network card) presents multiple independent "virtual functions," each of which can be directly assigned to a different VM. This gives each VM near-native I/O performance without hypervisor involvement in the data path, and without dedicating an entire physical device to one VM.

---

Next: [OS-Level Virtualization: Namespaces and Cgroups](03-os-level-virtualization-namespaces-cgroups.md)
