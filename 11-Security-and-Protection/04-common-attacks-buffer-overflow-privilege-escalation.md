# Common Attacks: Buffer Overflow and Privilege Escalation

Protection mechanisms are only as strong as their implementation. This section covers how attackers break through OS security -- not to teach exploitation, but because understanding attacks is essential to understanding defenses. Every defense mechanism (ASLR, stack canaries, DEP) exists because of a specific attack technique.

## Buffer Overflow: The Classic

A buffer overflow occurs when a program writes data past the end of a buffer, overwriting adjacent memory. In C and C++, there's no automatic bounds checking -- the language trusts the programmer to get it right. Attackers exploit this trust.

### Stack Layout (Normal Operation)

When a function is called, the stack looks like this:

```
High addresses
+---------------------------+
|  Caller's stack frame     |
+---------------------------+
|  Arguments to function    |
+---------------------------+
|  Return address           |  <-- Where to jump when function returns
+---------------------------+
|  Saved frame pointer (RBP)|
+---------------------------+
|  Local variables          |  <-- Including buffers (arrays)
|  char buf[64]             |
+---------------------------+
|  (more locals)            |
+---------------------------+
Low addresses
           Stack grows DOWN
```

### Stack-Based Buffer Overflow

```c
void vulnerable(char *input) {
    char buf[64];
    strcpy(buf, input);  // No bounds check! Copies until null byte.
}
```

If `input` is 100 bytes, `strcpy` writes past `buf[64]` and overwrites:

```
Before overflow:              After overflow (80-byte input):
+--------------------+        +--------------------+
| Return address     |        | 0x41414141414141   |  <-- OVERWRITTEN
| = 0x00400abc       |        | (attacker's addr)  |      with attacker's address
+--------------------+        +--------------------+
| Saved RBP          |        | AAAAAAAAAAAAAAAA   |  <-- OVERWRITTEN
+--------------------+        +--------------------+
| buf[56..63]        |        | AAAAAAAAAAAAAAAA   |  <-- Legitimate buffer space
| buf[48..55]        |        | AAAAAAAAAAAAAAAA   |
| buf[40..47]        |        | AAAAAAAAAAAAAAAA   |
| buf[32..39]        |        | AAAAAAAAAAAAAAAA   |
| buf[24..31]        |        | AAAAAAAAAAAAAAAA   |
| buf[16..23]        |        | AAAAAAAAAAAAAAAA   |
| buf[8..15]         |        | AAAAAAAAAAAAAAAA   |
| buf[0..7]          |        | AAAAAAAAAAAAAAAA   |
+--------------------+        +--------------------+
```

When the function returns, the CPU pops the overwritten return address and jumps to the attacker's chosen location.

### Shellcode Injection

In the classic attack, the attacker puts executable code (shellcode) in the buffer itself, then overwrites the return address to point back into the buffer:

```
+--------------------+
| 0x7fff0040  -------+--+  Return address points INTO the buffer
+--------------------+  |
| Saved RBP (junk)   |  |
+--------------------+  |
| \x90\x90\x90\x90   |  |  NOP sled (padding)
| \x90\x90\x90\x90   |  |
+--------------------+  |
| Shellcode:          |<-+  CPU lands here, executes this:
| \x48\x31\xc0...    |     execve("/bin/sh", NULL, NULL)
| (spawn a shell)     |
+--------------------+

Result: attacker gets a shell with the process's privileges.
If the process is SUID root --> attacker gets root shell.
```

## Defenses Against Buffer Overflow

Each defense addresses a specific aspect of the attack:

### 1. Stack Canaries

A random value placed between the buffer and the return address. Before returning, the function checks if the canary was modified:

```
+--------------------+
| Return address     |
+--------------------+
| Saved RBP          |
+--------------------+
| CANARY = 0xDEAD..  |  <-- Random value, checked before return
+--------------------+
| buf[64]            |
+--------------------+

If overflow overwrites the canary:
  Function checks canary before return
  Canary changed? --> abort() instead of returning
  Attacker can't return to shellcode
```

**Limitation:** The attacker might leak the canary value through a separate vulnerability (format string bug, information disclosure), then include the correct canary in their overflow payload.

```bash
# GCC enables stack canaries by default
$ gcc -fstack-protector-strong program.c   # Default on most distros
$ gcc -fno-stack-protector program.c       # Disable (for CTF/testing only)
```

### 2. ASLR (Address Space Layout Randomization)

Randomize where everything is loaded in memory on each execution:

```
Without ASLR (predictable):        With ASLR (randomized):
+--------------------+              +--------------------+
| Stack at 0x7fff..  |              | Stack at 0x7f3a..  |  <-- Different each run
+--------------------+              +--------------------+
|                    |              |                    |
| Heap at 0x602000   |              | Heap at 0x55c2..   |  <-- Different each run
+--------------------+              +--------------------+
| libc at 0x7f00..   |              | libc at 0x7f1b..   |  <-- Different each run
+--------------------+              +--------------------+
| Code at 0x400000   |              | Code at 0x5632..   |  <-- PIE: different each run
+--------------------+              +--------------------+

Attacker can't hardcode addresses because they change every time.
```

```bash
# Check ASLR status on Linux
$ cat /proc/sys/kernel/randomize_va_space
2    # 0=off, 1=partial (stack/libs), 2=full (stack/libs/heap)

# See it in action: run same program twice, addresses differ
$ cat /proc/self/maps | grep stack
7ffd12340000-7ffd12361000 rw-p ... [stack]
$ cat /proc/self/maps | grep stack
7ffc98760000-7ffc98781000 rw-p ... [stack]   # Different!
```

**Limitation:** 32-bit systems have limited entropy (~16 bits for stack) -- can be brute-forced in minutes. Information leaks can reveal the randomized base addresses.

### 3. DEP / NX Bit (No-Execute)

Mark memory pages as either writable OR executable, never both:

```
Without DEP:                        With DEP:
+--------+-----+-----+             +--------+-----+-----+
| Region | W   | X   |             | Region | W   | X   |
+--------+-----+-----+             +--------+-----+-----+
| Code   | No  | Yes |             | Code   | No  | Yes |
| Stack  | Yes | Yes | <--BAD      | Stack  | Yes | No  | <--shellcode can't run
| Heap   | Yes | Yes | <--BAD      | Heap   | Yes | No  | <--shellcode can't run
+--------+-----+-----+             +--------+-----+-----+

W^X principle: a page can be Writable XOR eXecutable, never both.
If attackers inject code into the stack, the CPU refuses to execute it.
```

**Limitation:** Doesn't prevent **Return-Oriented Programming** (ROP).

### 4. Control Flow Integrity (CFI)

Verify that indirect jumps (function pointers, virtual calls, return addresses) only go to valid targets:

```
Without CFI: attacker can redirect any indirect call anywhere
With CFI:    indirect calls can only target valid function entry points

Normal call:  call [vtable + 0x10]  --> valid_function()    OK
ROP attack:   call [vtable + 0x10]  --> middle_of_gadget()  BLOCKED by CFI
```

CFI is implemented by compilers (Clang CFI, MSVC CFG) and adds runtime checks before indirect calls. It significantly raises the bar for exploitation.

## Return-Oriented Programming (ROP)

ROP bypasses DEP by reusing existing executable code instead of injecting new code:

```
Normal execution:              ROP attack:
                               Attacker chains "gadgets" -- small sequences
function_A:                    ending in RET, found in existing code:
  mov rdi, rax
  call function_B              Gadget 1: pop rdi; ret       (set argument)
  ret                          Gadget 2: pop rsi; ret       (set argument)
                               Gadget 3: call execve; ret   (spawn shell)

Stack after overflow:
+--------------------+
| addr of gadget 3   |  RET jumps here third
+--------------------+
| 0 (NULL)           |  RSI = NULL (second arg to execve)
+--------------------+
| addr of gadget 2   |  RET jumps here second
+--------------------+
| addr of "/bin/sh"  |  RDI = "/bin/sh" (first arg to execve)
+--------------------+
| addr of gadget 1   |  RET jumps here first
+--------------------+

Each gadget does a small operation, then RET pops the next gadget address.
The attacker builds a program from fragments of existing code.
No injected code needed -- DEP is irrelevant.
```

**Defense:** ASLR (randomizes gadget addresses), CFI (validates return targets), shadow stacks (separate protected stack for return addresses).

## Privilege Escalation

Privilege escalation is gaining higher privileges than authorized. Two types:

```
Vertical Escalation:          Horizontal Escalation:
  user --> root                 user A --> user B
  
  +--------+                    +--------+   +--------+
  |  root  |  <-- target        | User A |   | User B | <-- target
  +--------+                    +--------+   +--------+
      ^                              |            ^
      |                              +------------+
  +--------+                    Attacker accesses another
  | user   |                    user's data/session
  +--------+
```

### Common Privilege Escalation Vectors

| Vector | How It Works | Example |
|--------|-------------|---------|
| **SUID vulnerability** | Exploit bug in SUID-root binary | Buffer overflow in /usr/bin/passwd |
| **Kernel exploit** | Bug in kernel code gives ring 0 | Dirty COW (CVE-2016-5195) |
| **Misconfigured sudo** | User can run commands as root | `sudo vim` --> shell escape to root |
| **Writable cron/service** | Modify a script run by root | Edit /etc/cron.d/backup script |
| **Container escape** | Break out of container isolation | Mount host filesystem, kernel exploit |
| **Weak file permissions** | Sensitive files readable by all | /etc/shadow readable by non-root |
| **PATH manipulation** | Trick privileged process into running attacker's binary | Malicious `ls` in PATH before /bin/ls |

### Container Escapes

A container escape is privilege escalation from container to host:

```
Normal container:                After escape:
+---------------------------+    +---------------------------+
| Host OS                   |    | Host OS                   |
|  +-----+  +-----+        |    |  +-----+  +-----+        |
|  |Cont | |Cont |         |    |  |Cont | |Cont |         |
|  | A   | | B   |         |    |  | A --|---> HOST ACCESS  |
|  +-----+  +-----+        |    |  +-----+  +-----+        |
+---------------------------+    +---------------------------+
```

Common escape techniques:
- **Privileged containers** (`--privileged`): full access to host devices and capabilities
- **Mounting host paths**: `docker run -v /:/host` gives access to everything
- **Kernel exploits**: container shares the host kernel -- kernel bug = escape
- **Excessive capabilities**: CAP_SYS_ADMIN enables namespace manipulation

## Other Attack Types (Brief)

### Use-After-Free

```c
char *ptr = malloc(64);
free(ptr);           // Memory freed
// ... later ...
strcpy(ptr, data);   // Writing to freed memory!
// Attacker controls what was allocated in that same spot
```

The freed memory might be reallocated for a different purpose. The dangling pointer now writes to the wrong object -- potentially overwriting function pointers or security-critical data.

### Format String Attacks

```c
printf(user_input);        // VULNERABLE: user controls format string
printf("%s", user_input);  // SAFE: user input is just data

// If user_input = "%x %x %x %x", printf reads values from the stack
// If user_input = "%n", printf WRITES to a stack address
// Attacker can read and write arbitrary memory
```

## Real-World Connection

**Why C/C++ are particularly vulnerable**: C and C++ perform no automatic bounds checking on array access. `buf[100]` on a 64-byte buffer compiles and runs -- it just accesses memory it shouldn't. Languages with memory safety (Rust, Go, Java) prevent buffer overflows at the language level. This is why the industry is moving critical infrastructure to memory-safe languages.

**Dirty COW (CVE-2016-5195)**: A race condition in the Linux kernel's copy-on-write mechanism allowed any local user to write to read-only files. This meant any unprivileged user could modify /etc/passwd to add a root account. It affected every Linux kernel from 2007 to 2016 -- including every Android phone, every cloud VM, every container.

**Chrome's sandbox model**: Chrome runs renderer processes (which parse untrusted HTML/JS) in a severely restricted sandbox: seccomp-bpf (minimal syscalls), namespaces (isolated view), no file system access, no network access. Even if an attacker achieves code execution via a browser exploit, they're trapped in the sandbox. Escaping requires a second exploit in the browser kernel or OS kernel.

**Rust and memory safety**: Rust's ownership system prevents buffer overflows, use-after-free, and data races at compile time. The Linux kernel is increasingly adopting Rust for new drivers and modules. Android rewrote Bluetooth and NFC stacks in Rust, and memory safety bugs in those components dropped to near zero.

## Interview Angle

**Q: Explain how a stack-based buffer overflow works and name three defenses.**

A: A stack buffer overflow occurs when a program writes past the bounds of a stack-allocated buffer (e.g., `strcpy` with no length check), overwriting the saved return address. When the function returns, the CPU jumps to the attacker-controlled address, executing arbitrary code. Three defenses: (1) Stack canaries -- a random value between the buffer and return address, checked before return. (2) ASLR -- randomizes memory layout so the attacker can't predict addresses. (3) DEP/NX -- marks stack and heap as non-executable so injected shellcode can't run.

**Q: What is ASLR and what are its limitations?**

A: ASLR (Address Space Layout Randomization) randomizes the base addresses of the stack, heap, libraries, and executable on each process launch. Attackers can't hardcode addresses for ROP gadgets or shellcode targets. Limitations: (1) 32-bit systems have limited randomization entropy (brute-forceable). (2) Information leaks (format string bugs, side channels) can reveal the randomized addresses. (3) Without PIE (Position Independent Executable), the main binary may be at a fixed address. (4) Doesn't prevent attacks that don't need to know addresses (e.g., relative overwrites).

**Q: What's the difference between vertical and horizontal privilege escalation?**

A: Vertical escalation means gaining higher privilege than authorized -- typically user to root. Examples: kernel exploits, SUID binary bugs, misconfigured sudo. Horizontal escalation means accessing another user's resources at the same privilege level -- user A reads user B's files or hijacks their session. Both are critical: vertical gives system-wide control, horizontal may expose sensitive data across tenants.

**Q: How does Return-Oriented Programming (ROP) bypass DEP?**

A: DEP prevents execution of injected code by marking writable pages as non-executable. ROP sidesteps this entirely by not injecting new code. Instead, the attacker chains together small sequences of existing executable code ("gadgets") that each end with a RET instruction. By carefully arranging addresses on the stack, each RET pops the next gadget's address, building a program from fragments of legitimate code (libc, the binary itself). Since the gadgets are in executable memory, DEP doesn't block them. Defenses include ASLR (randomizes gadget locations), CFI (validates return targets), and shadow stacks.

---

**Next:** [Sandboxing and Mandatory Access Control](05-sandboxing-and-mandatory-access-control.md) -- confining processes even after they've been compromised.
