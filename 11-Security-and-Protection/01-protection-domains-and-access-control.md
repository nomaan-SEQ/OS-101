# Protection Domains and Access Control

Before we talk about hackers and exploits, we need to understand the OS mechanisms that keep processes from stepping on each other. Protection is the internal mechanism -- it ensures that a bug in your web browser can't corrupt your database, and that a user-space program can't rewrite kernel memory. Security builds on top of protection to defend against deliberate attacks.

## Protection vs Security

These terms get used interchangeably, but they solve different problems:

| Concept | What It Addresses | Mechanism | Example |
|---------|-------------------|-----------|---------|
| **Protection** | Internal isolation -- processes don't interfere | Hardware rings, page tables, file permissions | Process A can't read Process B's memory |
| **Security** | External threats -- defending against adversaries | Authentication, encryption, auditing, MAC | Attacker can't exploit a buffer overflow to get root |

Protection gives you the **building blocks** (hardware rings, memory isolation, access rights). Security uses those blocks to build **policies** that defend against deliberate attacks.

## The Principle of Least Privilege

This is the single most important idea in OS security:

> **Every process (and every user) should operate with the minimum set of privileges necessary to complete its task.**

Why this matters:
- If a web server only has permission to read /var/www and bind port 80, a compromised web server can't read /etc/shadow or load kernel modules
- If a container drops all capabilities except what it needs, a container escape has limited blast radius
- If a database process can't exec() new programs, an attacker who gets SQL injection can't spawn a shell

The principle applies at every level: user accounts, processes, containers, microservices, cloud IAM roles.

## Protection Domains

A **protection domain** defines what a process (or subject) is allowed to do. Formally, it's a set of (object, access-rights) pairs:

```
Domain D1 = { (File1, {read, write}), (File2, {read}), (Printer, {print}) }
Domain D2 = { (File1, {read}), (File3, {read, write, execute}) }
Domain D3 = { (File2, {read, write}), (Printer, {print}), (Network, {send}) }
```

Each domain is like a "hat" a process wears -- it determines what objects the process can touch and what operations it can perform on them. A process executes within exactly one domain at any time, but it can **switch domains** under controlled circumstances.

### Real Examples of Domains

| Domain | What Defines It | Access Rights |
|--------|----------------|---------------|
| Regular user process | UID/GID of the user | Files owned by user, group files, world-readable files |
| Root process | UID 0 | Almost everything (DAC-wise) |
| Kernel mode | CPU ring 0 | All hardware, all memory, all instructions |
| Docker container | Namespaces + capabilities + seccomp | Restricted subset of host resources |
| Browser renderer (Chrome) | seccomp + namespaces | Almost nothing -- can't even open files |

## Protection Rings (Deeper Dive)

We introduced rings in `00-Introduction`. Here's the full picture:

```
+--------------------------------------------------+
|                    Ring 3                          |
|              User Applications                    |
|                                                   |
|    +------------------------------------------+   |
|    |              Ring 2                       |   |
|    |         Device Drivers (theory)           |   |
|    |                                           |   |
|    |    +----------------------------------+   |   |
|    |    |           Ring 1                  |   |   |
|    |    |      OS Services (theory)         |   |   |
|    |    |                                   |   |   |
|    |    |    +---------------------------+  |   |   |
|    |    |    |        Ring 0              |  |   |   |
|    |    |    |       OS Kernel            |  |   |   |
|    |    |    |  (full hardware access)    |  |   |   |
|    |    |    +---------------------------+  |   |   |
|    |    +----------------------------------+   |   |
|    +------------------------------------------+   |
+--------------------------------------------------+

        Most Privileged (inner) -----> Least Privileged (outer)
```

**In practice**, modern OSes use only two rings:
- **Ring 0**: Kernel (full hardware access, all instructions)
- **Ring 3**: Everything else (user processes, even drivers on some architectures)

Rings 1 and 2 are mostly unused. However, **Ring -1** (VMX root mode) exists for hypervisors, and **Ring -2** (SMM) exists for firmware. These matter for virtualization and secure boot.

### How Ring Transitions Work

```
User Space (Ring 3)                  Kernel Space (Ring 0)
+------------------+                 +-------------------+
|                  |   syscall       |                   |
|  Application     +---------------->  Kernel handler   |
|                  |   (trap to      |                   |
|                  |    ring 0)      |  - validate args  |
|                  |                 |  - do privileged  |
|                  |   sysret        |    operation      |
|                  <----------------+  - return result   |
+------------------+                 +-------------------+
```

The CPU enforces this: a ring 3 process literally **cannot** execute privileged instructions (like `wrmsr`, `hlt`, or direct I/O port access). Attempting it triggers a hardware exception.

## Domain Switching

Processes need to change domains in controlled ways. A regular user sometimes needs to do privileged things (change their password, bind a low port). The OS provides controlled mechanisms:

### System Calls (User -> Kernel Domain)

The most common domain switch. The CPU traps to ring 0, executes kernel code, and returns. The kernel validates everything before acting.

### SUID Programs (User -> Different User Domain)

```bash
$ ls -la /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 Apr  1 10:00 /usr/bin/passwd
   ^
   SUID bit: runs as file OWNER (root), not as the calling user
```

When you run `passwd`, it temporarily runs with root's effective UID, so it can write to /etc/shadow. This is a **domain switch** from your user domain to root's domain -- but only for that specific program.

The danger: if a SUID-root program has a bug, an attacker can exploit it to get root access. This is why there should be as few SUID programs as possible.

### Linux Capabilities (Fine-Grained Domain)

Instead of switching to the full root domain, grant specific capabilities:

```bash
# Give a program just the ability to bind low ports -- nothing else
$ sudo setcap cap_net_bind_service=ep /usr/bin/myserver
```

Now `myserver` can bind port 80 without being root. This is a much narrower domain than full root access.

## The Access Matrix

The access matrix is the conceptual model for all access control. It's a table where:
- **Rows** = subjects (users, processes, domains)
- **Columns** = objects (files, devices, memory segments)
- **Cells** = the set of allowed operations

```
                    Objects
              File1    File2    File3    Printer    Network
           +--------+--------+--------+----------+---------+
  Alice    | r, w   | r      |        | print    |         |
           +--------+--------+--------+----------+---------+
  Bob      | r      |        | r, w, x|          | send    |
Subjects   +--------+--------+--------+----------+---------+
  Web      |        | r      |        |          | send,   |
  Server   |        |        |        |          | recv    |
           +--------+--------+--------+----------+---------+
  Kernel   | r,w,x  | r,w,x  | r,w,x  | *        | *       |
           +--------+--------+--------+----------+---------+

  r = read, w = write, x = execute, * = all operations
```

This matrix perfectly captures "who can do what to which object." But there's a problem:

### Why the Access Matrix Can't Be Stored Directly

Consider a real system:
- 10,000 users
- 1,000,000 files
- That's 10 billion cells, most of them **empty**

The matrix is huge and sparse. We need efficient representations. There are two fundamental approaches:

```
Access Matrix
     |
     +---> Store by COLUMN (per object) ---> Access Control Lists (ACLs)
     |     "For each file, who can access it?"
     |
     +---> Store by ROW (per subject) -----> Capabilities
           "For each user/process, what can they access?"
```

Both are valid. Both have tradeoffs. We'll cover them in detail in [file 03](03-access-control-lists-and-capabilities.md).

## Real-World Connection

**Docker and protection domains**: A Docker container is essentially a custom protection domain. It combines Linux namespaces (isolated view of system resources), dropped capabilities (limited root powers), seccomp filters (restricted syscalls), and optional MAC policies (AppArmor/SELinux). Each of these narrows the domain.

**Cloud IAM as access matrices**: AWS IAM policies are essentially access matrix entries: "Role X can perform actions {s3:GetObject, s3:PutObject} on resource {arn:aws:s3:::mybucket/*}." The row is the role, the columns are AWS resources, and the cells are allowed actions. The same conceptual model scales from a single Unix box to a global cloud infrastructure.

**Microservices and least privilege**: Each microservice should run as a separate user with access to only its own database, its own secrets, and the specific network endpoints it needs. This is the principle of least privilege applied at the service level -- if the payment service is compromised, the attacker can't read the user database.

**Chrome's multi-process model**: Chrome runs each tab in a separate process with a very restricted protection domain (seccomp + namespaces). The renderer process can't open files, can't access the network directly, can't read other tabs' memory. It can only communicate with the browser process through a narrow IPC channel. This is protection domains in action.

## Interview Angle

**Q: What is a protection domain and how does it relate to least privilege?**

A: A protection domain is the set of (object, access-rights) pairs that define what a process can do. For example, a web server's domain might include (read, /var/www/*), (bind, port 80), (send/recv, network). The principle of least privilege says this domain should be as small as possible -- if the web server doesn't need to write files, don't include write access. In Linux, you implement this with user accounts, file permissions, capabilities, seccomp filters, and MAC policies.

**Q: Why do modern OSes only use two of the four x86 protection rings?**

A: Rings 1 and 2 were intended for OS services and drivers at intermediate privilege levels, but the added complexity wasn't worth it. Using just ring 0 (kernel) and ring 3 (everything else) simplifies the design while still providing the critical isolation boundary. Virtualization added ring -1 (hypervisor) when the need for another privilege level actually materialized. The lesson: only add privilege levels when there's a clear isolation need.

**Q: How does SUID work and why is it a security risk?**

A: The SUID (Set User ID) bit on an executable causes it to run with the file owner's effective UID instead of the caller's UID. For example, `/usr/bin/passwd` is owned by root with SUID set, so any user can run it and it executes with root privileges to update /etc/shadow. The risk: if a SUID-root program has a vulnerability (buffer overflow, race condition), an attacker can exploit it to execute arbitrary code as root. Mitigations include minimizing SUID programs and using Linux capabilities for fine-grained privilege instead.

---

**Next:** [Authentication and Authorization](02-authentication-and-authorization.md) -- proving who you are and determining what you're allowed to do.
