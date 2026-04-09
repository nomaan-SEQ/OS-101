# User Mode vs Kernel Mode

## Why This Is the Most Important Boundary

Every modern CPU has (at minimum) two execution modes. This is **the** fundamental protection mechanism that makes everything else possible — memory protection, process isolation, secure system calls. If you understand nothing else about OS internals, understand this.

```
┌─────────────────────────────────────────────┐
│                USER MODE                    │
│                                             │
│  Your applications run here.                │
│  - Limited instruction set                  │
│  - Can't access hardware directly           │
│  - Can't access other processes' memory     │
│  - Must ask the kernel via system calls     │
│                                             │
├═════════════════════════════════════════════╡  ← Protection boundary
│                                             │     (hardware-enforced)
│               KERNEL MODE                   │
│                                             │
│  The OS kernel runs here.                   │
│  - Full instruction set                     │
│  - Direct hardware access                   │
│  - Access to all memory                     │
│  - Can do anything                          │
│                                             │
└─────────────────────────────────────────────┘
```

## How It Works (Hardware Level)

The CPU has a **mode bit** (or privilege level register) that tracks the current execution mode:

- **Mode bit = 1 (User Mode):** The CPU restricts which instructions can execute. Privileged instructions (like writing to I/O ports, modifying page tables, or halting the CPU) will trigger a **trap** (exception) if attempted.
- **Mode bit = 0 (Kernel Mode):** No restrictions. All instructions are allowed.

### x86 Protection Rings

x86 processors have 4 privilege levels (rings), though most OSes only use 2:

```
        Ring 0 (Kernel Mode)
       ┌─────────────────────┐
       │   OS Kernel         │  ← Full hardware access
       │                     │
     ┌─┴─────────────────────┴─┐
     │   Ring 1 & 2            │  ← Rarely used (some hypervisors)
     │   (mostly unused)       │
   ┌─┴─────────────────────────┴─┐
   │       Ring 3 (User Mode)    │  ← Applications run here
   │   Apps, Libraries, Shell    │
   └─────────────────────────────┘
```

- **Ring 0:** Kernel — full privileges
- **Ring 1–2:** Almost never used (originally for device drivers; some hypervisors use Ring -1 via VT-x)
- **Ring 3:** User applications — restricted

## What User Mode Cannot Do

These operations are **privileged** and will cause a CPU exception if attempted in user mode:

| Forbidden Action | Why It's Restricted |
|-----------------|-------------------|
| Access I/O ports directly | Could corrupt devices or steal data |
| Modify page tables | Could access any process's memory |
| Disable interrupts | Could lock up the entire system |
| Halt the CPU | Would freeze the machine |
| Load new GDT/IDT | Could subvert the protection mechanism itself |
| Write to model-specific registers | Could alter CPU behavior |

## How the Transition Happens

There are three ways to switch from user mode to kernel mode:

### 1. System Call (Intentional)
The application **explicitly requests** OS services. This is the normal, intended path.

```
User Mode                          Kernel Mode
─────────                          ───────────
App calls read()
    │
    ├──► libc wraps it into
    │    syscall instruction
    │
    ├──► CPU switches to ────────► Kernel's syscall handler
         kernel mode                    │
                                        ├──► Validates arguments
                                        ├──► Performs the operation
                                        ├──► Prepares return value
                                        │
    App resumes ◄──── CPU switches ◄────┘
                      back to user mode
```

### 2. Interrupt (External Event)
A hardware device (keyboard, disk, timer) signals the CPU. The CPU pauses the current user program, switches to kernel mode, and runs the **interrupt handler**.

```
User Mode                          Kernel Mode
─────────                          ───────────
App running normally
    │
    │  ◄── Timer fires!
    │
    ├──► CPU saves state ────────► Kernel's interrupt handler
         switches to kernel             │
                                        ├──► Handle the interrupt
                                        │    (e.g., schedule next process)
                                        │
    App resumes ◄──── CPU switches ◄────┘
    (or different       back to
     app resumes)       user mode
```

### 3. Exception/Trap (Error)
The user program does something illegal (divide by zero, access invalid memory, execute privileged instruction). The CPU traps into kernel mode.

```
User Mode                          Kernel Mode
─────────                          ───────────
App divides by zero
    │
    ├──► CPU detects error ──────► Kernel's exception handler
         switches to kernel             │
                                        ├──► Decide what to do
                                        │    (usually: kill the process,
                                        │     send SIGSEGV/SIGFPE)
                                        │
    Process terminated ◄────────────────┘
```

## The Cost of Mode Switching

Switching between user mode and kernel mode is **not free**:

1. **Save user-mode registers** to the process's kernel stack
2. **Switch the mode bit** (change privilege level)
3. **Jump to the kernel handler** (look up in interrupt/syscall table)
4. **Execute kernel code**
5. **Restore user-mode registers**
6. **Switch the mode bit back**
7. **Return to user code**

This overhead is why:
- Batching system calls is faster than many small ones (buffered I/O vs unbuffered)
- `mmap()` can be faster than `read()` for large files (fewer mode switches)
- `io_uring` in modern Linux minimizes mode switches for I/O

**Approximate cost:** A mode switch takes ~1-5 microseconds on modern hardware. Sounds small, but at millions of syscalls per second, it adds up.

## Real-World Connections

### Why Your App Can't Crash the OS
Your application runs in user mode. Even if it has a bug and tries to write to address `0x0000`, the **hardware** (MMU) catches this in user mode and triggers a trap. The kernel handles the trap by killing your process — not by crashing.

### Why Device Drivers Are Dangerous
In Linux, device drivers run in **kernel mode** (Ring 0). A buggy driver has full access to all memory and hardware. This is why driver bugs are the #1 cause of kernel crashes (blue screen / kernel panic).

### Why Docker Containers Are NOT VMs
Containers share the **same kernel**. All container processes are in user mode, making system calls to the same kernel. A kernel exploit in one container compromises all containers on the host. VMs, by contrast, have separate kernels.

### Why Spectre/Meltdown Was a Big Deal
These vulnerabilities allowed user-mode code to **read kernel-mode memory** through CPU speculative execution bugs. They broke the fundamental user/kernel boundary at the hardware level, requiring OS-level mitigations (KPTI) that added ~5% overhead to system calls.

## Interview Angle

**Q: What is the difference between user mode and kernel mode?**
> User mode is a restricted CPU execution mode where applications run — they can't access hardware directly, modify page tables, or execute privileged instructions. Kernel mode has unrestricted access to all hardware and memory. The CPU's mode bit enforces this boundary in hardware. Applications cross from user to kernel mode via system calls, and the CPU also switches on interrupts or exceptions.

**Q: Why do we need two modes? Can't we just trust all programs?**
> Without dual-mode protection, any buggy or malicious program could corrupt other processes' memory, access any file, or crash the entire system. The mode boundary is what makes multi-process systems safe. Even a small bug (null pointer dereference) in kernel mode crashes the whole OS — that's why we minimize what runs in kernel mode.

**Q: What triggers a switch from user mode to kernel mode?**
> Three things: (1) system calls — the application explicitly requests a service, (2) interrupts — hardware signals the CPU (timer, keyboard, disk), (3) exceptions — the program does something illegal (segfault, divide by zero). In all three cases, the CPU saves state, switches the mode bit, and jumps to the appropriate kernel handler.

---

**Next:** [05-how-os-boots.md](./05-how-os-boots.md) — The complete journey from pressing the power button to a running OS.
