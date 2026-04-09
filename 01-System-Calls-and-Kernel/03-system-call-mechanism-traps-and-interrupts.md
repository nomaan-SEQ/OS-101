# System Call Mechanism: Traps and Interrupts

## The Trap Mechanism

A **trap** (also called a software interrupt) is how user programs voluntarily enter the kernel. It's the CPU instruction that says "I need kernel services -- please switch to Ring 0 and run the appropriate handler."

On x86 systems, there are two ways to trigger a trap:

| Method | Instruction | Era | Speed |
|--------|-----------|-----|-------|
| Legacy (32-bit) | `int 0x80` | Linux 2.x, i386 | Slow (~400+ cycles) |
| Modern (64-bit) | `syscall` | Linux 2.6+, x86-64 | Fast (~100-200 cycles) |

The `syscall` instruction was specifically designed by AMD (and adopted by Intel) to be a faster alternative to `int 0x80`. It avoids the overhead of the full interrupt descriptor table lookup.

## Step-by-Step: What Happens During a System Call

Let's trace exactly what happens when you execute `write(1, "hello", 5)` on x86-64 Linux:

```
DETAILED SYSCALL LIFECYCLE (x86-64 Linux)
==========================================

USER MODE (Ring 3)
------------------
1. libc write() wrapper executes:
     mov  rax, 1          # syscall number for write
     mov  rdi, 1          # fd = stdout
     mov  rsi, buf_ptr    # buffer address
     mov  rdx, 5          # count = 5 bytes
     syscall               # trigger the trap

HARDWARE TRANSITION
-------------------
2. CPU detects SYSCALL instruction:
     a. Save RIP (instruction pointer) into RCX
     b. Save RFLAGS into R11
     c. Load RIP from MSR_LSTAR (kernel entry point address)
     d. Load CS/SS for kernel mode (Ring 0)
     e. Mask RFLAGS (disable interrupts briefly)

     ** CPU is now in Ring 0 **

KERNEL MODE (Ring 0)
--------------------
3. Kernel entry point (entry_SYSCALL_64):
     a. Switch to kernel stack (load RSP from per-CPU data)
     b. Save ALL user registers onto kernel stack (pt_regs struct)
     c. Check that syscall number (rax) is valid (< NR_syscalls)
     d. Index into sys_call_table[rax] to find handler
     e. Call the handler: sys_write(fd=1, buf=ptr, count=5)

4. sys_write() executes:
     a. Validate fd 1 exists in current process's FD table
     b. Check that buf_ptr is a valid user-space address
     c. copy_from_user() to safely read the "hello" bytes
     d. Call the file's write operation (e.g., tty_write for stdout)
     e. Return bytes written (5) or error code

RETURN PATH
------------
5. Kernel return path:
     a. Store return value (5) in pt_regs->rax on the stack
     b. Check for pending signals (maybe SIGTERM arrived)
     c. Check if scheduler wants to preempt (need_resched flag)
     d. Restore user registers from kernel stack
     e. Execute SYSRET instruction

HARDWARE TRANSITION
-------------------
6. CPU detects SYSRET instruction:
     a. Restore RIP from RCX (back to instruction after SYSCALL)
     b. Restore RFLAGS from R11
     c. Switch CS/SS back to user mode (Ring 3)

     ** CPU is now back in Ring 3 **

USER MODE (Ring 3)
------------------
7. libc wrapper:
     a. Check return value in rax
     b. If negative: negate it, store in errno, return -1
     c. If positive: return the value (5 bytes written)
```

## Interrupts: Hardware vs Software

The kernel needs to handle more than just system calls. **Interrupts** are the general mechanism for getting the CPU's attention:

```
INTERRUPT TAXONOMY
===================

  Interrupts
  |
  +-- Synchronous (caused by CPU executing an instruction)
  |   |
  |   +-- Traps (intentional)
  |   |     - syscall / int 0x80    --> system call
  |   |     - int 3                 --> debugger breakpoint
  |   |
  |   +-- Faults (recoverable errors)
  |   |     - Page fault            --> load page from disk/swap
  |   |     - General protection    --> segfault (usually fatal)
  |   |     - Division by zero      --> SIGFPE
  |   |
  |   +-- Aborts (unrecoverable)
  |         - Machine check         --> hardware failure
  |         - Double fault          --> fault during fault handling
  |
  +-- Asynchronous (caused by external hardware)
      |
      +-- Maskable (can be temporarily disabled)
      |     - Timer interrupt       --> scheduling tick
      |     - Disk I/O complete     --> wake waiting process
      |     - Network packet arrived--> process incoming data
      |     - Keyboard press        --> input handling
      |
      +-- Non-Maskable (NMI -- cannot be disabled)
            - Hardware failure
            - Watchdog timeout
            - Performance monitoring
```

| Property | Hardware Interrupt | Software Interrupt (Trap) |
|----------|-------------------|--------------------------|
| **Source** | External device (timer, NIC, disk) | CPU instruction (`syscall`, `int N`) |
| **When** | Asynchronous -- can happen at ANY time | Synchronous -- happens at a specific instruction |
| **Predictable?** | No -- the kernel can't predict when a packet arrives | Yes -- the program explicitly triggers it |
| **Has a "current" process?** | Whatever was running when it fired | Yes -- the process that made the syscall |
| **Can sleep?** | NO -- interrupt context cannot sleep | Yes (in syscall context, the process can sleep) |

## The Interrupt Descriptor Table (IDT)

The **IDT** is a table in memory that tells the CPU where to jump for each interrupt/exception number. It's set up by the kernel early in boot:

```
INTERRUPT DESCRIPTOR TABLE (x86-64)
=====================================

  Vector | Type              | Handler
  -------+-------------------+---------------------------
    0    | Fault             | divide_error (div by zero)
    1    | Trap/Fault        | debug
    3    | Trap              | int3 (breakpoint)
    6    | Fault             | invalid_opcode
   13    | Fault             | general_protection
   14    | Fault             | page_fault
   ...   |                   |
   32-   | Hardware IRQs     | device interrupt handlers
   255   |                   | (timer, disk, network...)
   ...   |                   |
   128   | Trap              | int 0x80 (legacy syscall)
```

The CPU has a register (`IDTR`) that points to this table. When any interrupt or exception fires:
1. CPU reads the vector number
2. Indexes into the IDT
3. Loads the handler address and privilege info from the descriptor
4. Switches privilege level if needed
5. Jumps to the handler

For the modern `syscall` instruction, the CPU bypasses the IDT entirely and uses **Model-Specific Registers (MSRs)** instead, which is faster.

## Top-Half vs Bottom-Half Interrupt Handling

Hardware interrupts need to be handled **fast** because while an interrupt handler runs, that CPU can't do other work (and often, other interrupts on that line are blocked). The solution: split the work.

> **Analogy:** Think of it like a restaurant. The **top half** is the waiter who quickly jots down your order — fast, can't leave the floor. The **bottom half** is the kitchen that actually prepares the meal — can take longer and runs on its own schedule.

```
INTERRUPT HANDLING: TWO HALVES
===============================

  Network card signals: "packet arrived!"
           |
           v
  +------------------------------+
  | TOP HALF (hardirq)           |    Runs immediately, interrupts
  |                              |    disabled on this CPU line
  | - Acknowledge the hardware   |
  | - Copy packet from NIC to    |    MUST be fast (<< 1ms)
  |   kernel buffer              |    CANNOT sleep
  | - Schedule bottom half       |    CANNOT allocate memory (GFP_KERNEL)
  +------------------------------+
           |
           v
  +------------------------------+
  | BOTTOM HALF (deferred work)  |    Runs "soon" with interrupts
  |                              |    re-enabled
  | - Parse packet headers       |
  | - Route the packet           |    Can do more work
  | - Deliver to socket buffer   |    Still runs in interrupt context
  | - Wake up waiting process    |    (softirq) or process context
  +------------------------------+    (workqueue)
```

Linux provides several bottom-half mechanisms:

| Mechanism | Context | Can Sleep? | Use Case |
|-----------|---------|-----------|----------|
| **Softirq** | Interrupt context | No | High-frequency: networking, block I/O |
| **Tasklet** | Interrupt context (built on softirq) | No | Per-device deferred work |
| **Workqueue** | Process context (kernel thread) | Yes | Can do complex work, allocate memory |
| **Threaded IRQ** | Process context | Yes | Modern drivers, simpler model |

### Softirqs and ksoftirqd

Softirqs are the highest-priority bottom-half mechanism. There are a fixed number of softirq types (10 in current Linux):

```
  SOFTIRQ TYPES
  ==============
  HI_SOFTIRQ          -- high-priority tasklets
  TIMER_SOFTIRQ       -- timer callbacks
  NET_TX_SOFTIRQ      -- network transmit
  NET_RX_SOFTIRQ      -- network receive
  BLOCK_SOFTIRQ       -- block device I/O completion
  IRQ_POLL_SOFTIRQ    -- I/O polling
  TASKLET_SOFTIRQ     -- normal-priority tasklets
  SCHED_SOFTIRQ       -- scheduler load balancing
  HRTIMER_SOFTIRQ     -- high-resolution timers
  RCU_SOFTIRQ         -- RCU (Read-Copy-Update) callbacks
```

If softirqs pile up faster than they can be processed, the kernel offloads them to the **`ksoftirqd`** kernel thread (one per CPU), which runs at normal priority so it doesn't starve user processes.

## The Complete Lifecycle Diagram

Putting it all together:

```
COMPLETE INTERRUPT/SYSCALL LIFECYCLE
=====================================

  USER SPACE                         KERNEL SPACE
  ==========                         ============

  Program running
       |
       |---(syscall instruction)---> Syscall entry
       |                                  |
       |                             Save registers
       |                             Look up handler
       |                             Execute handler
       |                                  |
       |                             [handler may sleep,
       |                              waiting for I/O]
       |                                  |
       |            MEANWHILE:            |
       |                                  |
  Other process    <----(timer IRQ)---> Schedule tick
  gets CPU time         (preemption)     - update timers
       |                                 - check time slices
       |                                 - maybe context switch
       |
       |            <---(disk IRQ)----> I/O completion
       |                                 - top half: ack device
       |                                 - bottom half: wake
       |                                   sleeping process
       |
  Original process <--(context switch)-- Scheduler picks
  resumes                                original process
       |                                  |
       |                             Syscall return path
       |                             - check pending signals
       |                             - restore registers
       |                                  |
       |<---(sysret instruction)---  Return to user mode
       |
  Program continues
  with return value
```

---

## Real-World Connection

- **Networking performance**: The Linux networking stack processes packets in softirq context (`NET_RX_SOFTIRQ`). When your server handles 1M+ packets/sec, `ksoftirqd` can consume significant CPU time. Tools like `mpstat` show `%soft` (softirq time), and techniques like RSS (Receive Side Scaling) spread the interrupt load across CPUs.

- **Latency spikes**: If your application experiences occasional latency spikes, they might be caused by interrupt storms -- a burst of hardware interrupts that steals CPU time. `cat /proc/interrupts` shows interrupt counts per CPU per device. Binding interrupts to specific CPUs (`/proc/irq/N/smp_affinity`) is a common production tuning technique.

- **KPTI overhead**: After the Meltdown vulnerability was disclosed, Linux added Kernel Page Table Isolation, which requires switching page tables on every syscall entry/exit. This added 5-30% overhead to syscall-heavy workloads. Cloud providers saw measurable performance regression. This is why understanding the syscall mechanism matters -- security mitigations at this level have real cost.

---

## Interview Angle

**Q: Walk me through what happens at the hardware level when a process makes a system call on x86-64.**

A: The libc wrapper places the syscall number in `rax` and arguments in `rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9`. It then executes the `SYSCALL` instruction. The CPU saves `RIP` to `RCX` and `RFLAGS` to `R11`, loads the kernel entry point from the `LSTAR` MSR, sets the code segment to kernel mode (Ring 0), and masks certain flags. The kernel entry code (`entry_SYSCALL_64`) swaps to the kernel stack, pushes all user registers, validates the syscall number, indexes into `sys_call_table`, and calls the handler. On return, the kernel restores registers and executes `SYSRET`, which loads `RIP` from `RCX`, restores `RFLAGS` from `R11`, and returns to Ring 3.

**Q: What is the difference between top-half and bottom-half interrupt handling? Why is it needed?**

A: The top half runs immediately when the hardware interrupt fires, with interrupts disabled on that line. It must be extremely fast -- just acknowledge the hardware and copy critical data. The bottom half runs shortly after with interrupts re-enabled, handling the bulk of the work (parsing, routing, waking processes). This split is needed because running long handlers with interrupts disabled would cause other interrupts to be missed, leading to data loss (dropped packets, lost keystrokes) and increased latency. Linux offers several bottom-half mechanisms: softirqs for high-frequency work (networking), workqueues when the handler needs to sleep or allocate memory.

**Q: Why can't you sleep in interrupt context?**

A: Interrupt context has no backing process -- there's no `task_struct` to put on a wait queue and reschedule later. The scheduler needs a process context to perform a context switch. If you tried to sleep in interrupt context, the scheduler wouldn't know what to do -- there's nothing to "resume" later. Additionally, sleeping while holding a spinlock (common in interrupt handlers) would deadlock the system. This is why workqueues exist: they run deferred work in kernel thread context, where sleeping is safe.

---

**Next**: [04-common-system-calls-posix.md](04-common-system-calls-posix.md)
