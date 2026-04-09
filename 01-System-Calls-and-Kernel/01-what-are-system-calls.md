# What Are System Calls?

## The Core Idea

A system call is a **controlled entry point into the kernel**. It's the mechanism that lets your user-space program request services that only the kernel can provide: reading files, creating processes, allocating memory, sending network packets.

Think of it like a bank teller window. You (the user program) can't walk into the vault (kernel memory) and grab cash yourself. Instead, you fill out a form (set up registers), ring the bell (execute a trap instruction), and the teller (kernel) processes your request on the other side of the bulletproof glass (privilege boundary).

```
WHY SYSTEM CALLS EXIST
======================

  Without syscalls:                    With syscalls:
  
  +-----------+                        +-----------+
  | User App  | --- direct access ---> | User App  |
  +-----------+     to hardware        +-----------+
       |            (chaos, no              |
       v            protection)        [ TRAP GATE ] <-- controlled entry
  +-----------+                        +-----------+
  |  Hardware |                        |  Kernel   | <-- validates request
  +-----------+                        +-----------+
                                            |
                                       +-----------+
                                       |  Hardware |
                                       +-----------+
```

## Why Apps Can't Call Kernel Functions Directly

Three reasons this would be catastrophic:

1. **Memory protection**: The kernel lives in a separate, protected address space. User programs literally cannot see kernel memory -- the MMU (Memory Management Unit) enforces this in hardware. Attempting to access kernel addresses from user mode triggers a segmentation fault.

2. **Privilege separation**: Certain CPU instructions (like `hlt` to halt the processor, or writing to I/O ports) are privileged. The CPU only allows them in Ring 0 (kernel mode). User code runs in Ring 3.

3. **Validation and security**: If any program could directly call kernel functions, a buggy or malicious app could corrupt kernel data structures, crash the system, or read other processes' memory. The syscall interface lets the kernel **validate every request** before executing it.

## System Call vs Function Call vs Library Call

| Aspect | Function Call | Library Call | System Call |
|--------|-------------|-------------|-------------|
| **Example** | `mySort(arr)` | `printf("hi")` | `write(1, "hi", 2)` |
| **Privilege change?** | No | No (but may trigger a syscall internally) | Yes -- user mode to kernel mode |
| **Performance cost** | ~1 ns (just a CALL instruction) | ~1-10 ns | ~100-1000 ns (mode switch overhead) |
| **Where it runs** | Same process, same mode | Same process, same mode | Kernel address space, Ring 0 |
| **Who provides it?** | Your code | libc, libm, etc. | The OS kernel |

The key insight: `printf()` is a **library call** that internally calls `write()`, which is a **system call**. Many library functions are wrappers around one or more system calls.

## The System Call Number Table

Every system call has a unique **number** that identifies it. When your program wants to invoke a syscall, it puts the syscall number in a specific register (e.g., `rax` on x86-64 Linux) and triggers the trap.

The kernel maintains a **syscall table** -- an array of function pointers indexed by syscall number:

```
LINUX SYSCALL TABLE (x86-64, simplified)
========================================

  Number | Name        | Kernel Handler Function
  -------+-------------+---------------------------
    0    | read        | sys_read()
    1    | write       | sys_write()
    2    | open        | sys_open()
    3    | close       | sys_close()
   39    | getpid      | sys_getpid()
   57    | fork        | sys_fork()
   59    | execve      | sys_execve()
   60    | exit        | sys_exit()
   61    | wait4       | sys_wait4()
  ...    | ...         | ...
  ~450+  |             | (and growing with each kernel release)
```

You can see the full table on your Linux system at `/usr/include/asm/unistd_64.h` or by searching for the kernel's syscall table source.

Linux has **over 450 system calls** as of recent kernels. Most programs use a small fraction of them. A typical web server might use 30-50 distinct syscalls.

## The Common System Calls

Here are the ones you'll see most often:

| Syscall | Purpose | Category |
|---------|---------|----------|
| `open` | Open a file, return a file descriptor | File I/O |
| `read` | Read bytes from a file descriptor | File I/O |
| `write` | Write bytes to a file descriptor | File I/O |
| `close` | Close a file descriptor | File I/O |
| `fork` | Create a new process (copy of current) | Process |
| `exec` | Replace current process image with new program | Process |
| `wait` | Wait for a child process to finish | Process |
| `exit` | Terminate the current process | Process |
| `mmap` | Map files or devices into memory | Memory |
| `brk` | Change the data segment size (heap) | Memory |
| `socket` | Create a network endpoint | Networking |
| `pipe` | Create a unidirectional data channel | IPC |

## How a System Call Actually Works (Overview)

Here's the complete path when your program calls `write(1, "hello", 5)`:

```
THE LIFE OF A SYSTEM CALL
==========================

  +---------------------+
  | Your C Program      |
  |   write(1,"hello",5)|    1. You call write() in your code
  +---------------------+
           |
           v
  +---------------------+
  | libc Wrapper         |
  |   - put 1 in rax    |    2. libc sets up registers:
  |     (syscall # for   |       rax = syscall number (1)
  |      write)          |       rdi = fd (1 = stdout)
  |   - put args in      |       rsi = buffer pointer
  |     rdi, rsi, rdx    |       rdx = count (5)
  |   - execute SYSCALL  |    3. Execute the SYSCALL instruction
  +---------------------+
           |
     ======|============ PRIVILEGE BOUNDARY (Ring 3 -> Ring 0)
           |
           v
  +---------------------+
  | Kernel Entry Point   |
  |   - save user regs  |    4. CPU switches to kernel mode
  |   - switch to kernel |       Saves user state
  |     stack            |       Looks up handler in syscall table
  +---------------------+
           |
           v
  +---------------------+
  | sys_write() handler  |
  |   - validate fd      |    5. Kernel validates arguments
  |   - validate buffer  |       Checks permissions
  |   - copy from user   |       Performs the actual I/O
  |   - do the I/O       |
  +---------------------+
           |
           v
  +---------------------+
  | Return Path          |
  |   - set return value |    6. Put return value in rax
  |   - restore user regs|       Restore user registers
  |   - SYSRET/IRET      |       Switch back to Ring 3
  +---------------------+
           |
     ======|============ PRIVILEGE BOUNDARY (Ring 0 -> Ring 3)
           |
           v
  +---------------------+
  | Back in your program |
  |   - write() returns 5|    7. libc returns the result to you
  |     (bytes written)  |       (or sets errno on failure)
  +---------------------+
```

The total overhead of this privilege boundary crossing is roughly **100-1000 nanoseconds**, depending on the architecture and whether kernel page-table isolation (KPTI) is enabled (a Meltdown/Spectre mitigation that adds cost).

---

## Real-World Connection

- **Docker's `seccomp` profiles** restrict which syscalls a container can make. The default Docker profile blocks about 44 of the ~450 syscalls (including dangerous ones like `reboot` and `kexec_load`). Understanding the syscall table is essential for writing custom seccomp profiles.

- **`strace`** is the tool you'll use most often to observe system calls in practice. Running `strace ls` shows you every syscall that `ls` makes. This is indispensable for debugging "why is my program hanging?" or "why is this file not being found?"

- **Performance**: The `vDSO` (virtual Dynamic Shared Object) is a trick Linux uses to make certain read-only syscalls (like `gettimeofday`) execute entirely in user space, avoiding the trap overhead. This matters in high-frequency trading and other latency-sensitive systems.

---

## Interview Angle

**Q: What happens under the hood when a program calls `open()` to open a file?**

A: The C library's `open()` wrapper places the syscall number for `open` (typically 2 on x86-64 Linux) into `rax`, the file path pointer into `rdi`, and the flags into `rsi`. It then executes the `SYSCALL` instruction, which triggers a trap. The CPU switches from Ring 3 to Ring 0, saves user-space registers, and jumps to the kernel's syscall entry point. The kernel looks up `sys_open()` in the syscall table using the number in `rax`, validates the path (checking permissions via the filesystem's inode), allocates a file descriptor in the process's file descriptor table, and returns the fd number (or a negative error code). The CPU then switches back to Ring 3 via `SYSRET`, and the libc wrapper returns the fd to the caller.

**Q: Why is a system call more expensive than a regular function call?**

A: A regular function call is just a `CALL` instruction -- push the return address and jump. A system call requires: (1) saving all user-space registers, (2) switching to the kernel stack, (3) changing the CPU privilege level from Ring 3 to Ring 0, (4) running the kernel handler, (5) restoring everything and switching back. With KPTI (kernel page table isolation) enabled for Meltdown mitigation, there's an additional cost of switching page tables on every kernel entry/exit. This adds up to roughly 100-1000ns vs ~1ns for a function call.

---

**Next**: [02-kernel-structure-and-responsibilities.md](02-kernel-structure-and-responsibilities.md)
