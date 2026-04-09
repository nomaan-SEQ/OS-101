# Memory Hierarchy and Address Spaces

Before we can understand how an OS manages memory, we need to understand the hardware it manages. Memory isn't one thing — it's a hierarchy of storage technologies, each with different speed, size, and cost characteristics. The OS's job is to create the illusion of fast, large, cheap memory by intelligently moving data between these levels.

## The Memory Hierarchy

Every computer organizes storage in a pyramid. The top is tiny and blazingly fast. The bottom is vast and painfully slow.

```
              +------------+
              | Registers  |   < 1 ns,  ~1 KB
              +------------+
             /              \
        +--------------------+
        |    L1 Cache        |   ~1 ns,   64 KB per core
        +--------------------+
       /                      \
  +--------------------------+
  |       L2 Cache           |   ~4 ns,   256 KB - 1 MB per core
  +--------------------------+
 /                              \
+--------------------------------+
|          L3 Cache              |   ~10 ns,  8 - 64 MB shared
+--------------------------------+
|            RAM (DRAM)          |   ~100 ns, 8 - 512 GB
+--------------------------------+
|          SSD (Flash)           |   ~100 us, 256 GB - 4 TB
+--------------------------------+
|          HDD (Disk)            |   ~10 ms,  1 - 20 TB
+--------------------------------+
```

### Speed, Size, and Cost at Each Level

| Level | Access Time | Typical Size | Cost per GB (approx) | Volatile? |
|-------|-------------|-------------|----------------------|-----------|
| Registers | < 1 ns | ~1 KB | — | Yes |
| L1 Cache | ~1 ns | 32-64 KB per core | — | Yes |
| L2 Cache | ~4 ns | 256 KB - 1 MB per core | — | Yes |
| L3 Cache | ~10 ns | 8-64 MB (shared) | — | Yes |
| RAM (DRAM) | ~100 ns | 8-512 GB | ~$3-5 | Yes |
| SSD | ~100 us (100,000 ns) | 256 GB - 4 TB | ~$0.05-0.10 | No |
| HDD | ~10 ms (10,000,000 ns) | 1-20 TB | ~$0.01-0.03 | No |

The key insight: **there's a 10,000x speed gap between RAM and SSD, and a 100,000x gap between RAM and HDD**. This massive gap is why virtual memory (using disk as overflow for RAM) works but page faults are catastrophically expensive.

### Principle of Locality

The hierarchy works because programs exhibit **locality of reference**:

- **Temporal locality**: if you accessed an address recently, you'll likely access it again soon (loops, frequently used variables).
- **Spatial locality**: if you accessed an address, you'll likely access nearby addresses soon (arrays, sequential code execution).

Caches exploit locality by keeping recently and nearby accessed data in faster storage. When the OS manages virtual memory, it relies on the same principle — pages you've used recently are likely to be used again.

## Virtual (Logical) vs Physical Addresses

Here's the fundamental problem: if every process used physical addresses directly, they'd need to coordinate to avoid stepping on each other's memory. This is impractical and insecure.

The solution: **give every process its own virtual address space**.

- **Physical address**: the actual location in DRAM hardware (what the memory bus uses).
- **Virtual (logical) address**: what the process sees and uses. Each process thinks it starts at address 0 and has a vast, contiguous address space.

```
   Process A's view          Process B's view         Physical Memory
  +----------------+       +----------------+       +------------------+
  | 0x0000: code   |       | 0x0000: code   |       | 0x0000: OS       |
  | 0x1000: data   |       | 0x1000: data   |       | 0x1000: A's code |
  | 0x2000: heap   |       | 0x2000: heap   |       | 0x2000: B's data |
  | ...            |       | ...            |       | 0x3000: A's data |
  | 0xFFFF: stack  |       | 0xFFFF: stack  |       | 0x4000: B's code |
  +----------------+       +----------------+       | ...              |
                                                    +------------------+
  Both processes use          But they map to
  the SAME virtual            DIFFERENT physical
  addresses                   locations
```

This is the core idea: **the same virtual address in two different processes maps to different physical addresses**. Processes are completely isolated — Process A literally cannot name a physical address that belongs to Process B.

## Address Binding: When Are Addresses Resolved?

A program starts as source code with symbolic names (`int x`), goes through compilation, and eventually runs with real addresses. When does the binding from symbolic to real happen?

| Binding Time | How It Works | Flexibility | Example |
|-------------|-------------|-------------|---------|
| **Compile time** | Compiler generates absolute physical addresses | None — process must load at that exact address | MS-DOS .COM files, embedded systems |
| **Load time** | Compiler generates relocatable code, loader assigns addresses when program starts | Can load at any available address, but fixed once running | Early Unix, static linking |
| **Run time** | Addresses are translated on every memory access by hardware | Full flexibility — process can be moved in memory while running | All modern systems (via MMU) |

Modern systems use **run-time binding** because it enables virtual memory, demand paging, and dynamic relocation — features that make multitasking practical.

## The Memory Management Unit (MMU)

Run-time address translation needs to be fast — it happens on every single memory access. You can't do this in software; it must be in hardware. That hardware is the **MMU (Memory Management Unit)**.

```
         CPU                      MMU                    Physical Memory
   +-----------+            +------------+            +------------------+
   | Program   |  virtual   |            |  physical  |                  |
   | generates |  address   | Translates |  address   |   Actual DRAM    |
   | address   | ---------> | virtual to | ---------> |                  |
   | 0x4A3F    |            | physical   |            |                  |
   +-----------+            +------------+            +------------------+
                                  |
                            Uses page table
                            (in memory, managed
                             by OS kernel)
```

The MMU sits between the CPU and memory. Every address the CPU generates goes through the MMU, which:

1. Takes the virtual address from the CPU
2. Looks up the mapping in the **page table** (which the OS sets up)
3. Produces the corresponding physical address
4. Sends the physical address to the memory bus

If the mapping doesn't exist (page not in memory, or invalid access), the MMU generates a **page fault** — a trap to the OS kernel, which decides what to do.

### Why Hardware?

A single instruction like `mov eax, [rbx]` might access memory twice (fetch the instruction, then load the operand). If translation were done in software, each memory access would need dozens of extra instructions. The MMU does it in zero extra cycles on a TLB hit.

## Why Every Process Needs Its Own Address Space

This isn't just a nice feature — it's a fundamental requirement:

1. **Isolation**: a bug in one process (wild pointer, buffer overflow) can't corrupt another process's memory. Without this, one buggy program could take down the entire system.

2. **Security**: process A can't read process B's passwords, encryption keys, or private data. The MMU enforces this at the hardware level.

3. **Simplicity**: every process's code can be compiled assuming it starts at address 0. The linker doesn't need to know where in physical memory the program will land.

4. **Flexibility**: the OS can place pages anywhere in physical memory, move them around (for compaction or swapping), and even share physical frames between processes (shared libraries) — all invisible to the processes.

5. **Overcommitment**: the OS can promise more virtual memory than physically exists, relying on the fact that not all processes use all their memory at once (this is virtual memory, covered in file 05).

## Real-World Connection

- **Linux processes**: every process gets a 48-bit virtual address space on x86-64 (256 TB). Run `cat /proc/self/maps` to see how a process's virtual address space is laid out — you'll see code, heap, stack, shared libraries, all at different virtual addresses.
- **Docker containers**: containers share the host kernel but each has its own virtual address space. Memory isolation comes from the same MMU-based virtual memory, with cgroups adding memory limits on top.
- **Cloud instances**: when you request a VM with 16 GB of RAM, the hypervisor uses a second level of address translation (Extended Page Tables / EPT) to give the guest OS the illusion of owning physical memory.
- **Performance monitoring**: tools like `perf stat` report L1/L2/L3 cache miss rates. Understanding the hierarchy tells you why a 1% L3 miss rate might still be a performance problem (each miss costs ~100 ns to go to RAM).

## Interview Angle

**Q: What's the difference between a virtual address and a physical address?**

A virtual address is what a process uses — it's the address the CPU generates when executing instructions. A physical address is the actual location in DRAM. The MMU translates between them using page tables set up by the OS. This translation happens transparently on every memory access, and it's what enables process isolation, virtual memory, and flexible memory management.

**Q: Why do we need a memory hierarchy? Why not just use the fastest storage for everything?**

Cost and physics. **SRAM** (Static RAM, used in caches) stores each bit using a circuit of six transistors, making it extremely fast but expensive and bulky. **DRAM** (Dynamic RAM, used for main memory) stores each bit as a charge in a tiny capacitor, making it much denser and cheaper but slower -- and it needs constant refreshing because the capacitors leak charge. SRAM is about 100x more expensive per bit than DRAM, and DRAM is about 100x more expensive than flash storage. You physically can't build a system with terabytes of SRAM. The hierarchy works because of locality — programs tend to reuse data they've recently accessed, so keeping a small amount of hot data in fast cache captures most accesses.

**Q: What is the MMU and why is it in hardware?**

The MMU (Memory Management Unit) translates virtual addresses to physical addresses. It must be in hardware because translation happens on every memory access — potentially billions of times per second. A software implementation would multiply the cost of every memory access by 10-100x. The MMU uses a small cache called the TLB to make most translations essentially free.

---

Next: [02 - Contiguous Allocation and Fragmentation](02-contiguous-allocation-and-fragmentation.md)
