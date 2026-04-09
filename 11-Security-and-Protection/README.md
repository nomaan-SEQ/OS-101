# Security and Protection

The OS sits between every application and the hardware, making it the ultimate gatekeeper. Protection mechanisms ensure processes can't interfere with each other -- a buggy web browser shouldn't corrupt your database. Security mechanisms defend the system against deliberate attacks -- a malicious input shouldn't give an attacker root access. These are related but distinct problems: protection is about isolation between trusted components, security is about defending against untrusted adversaries.

Every major breach you've read about -- container escapes, privilege escalations, buffer overflows -- exploits weaknesses in OS-level security constructs. Understanding these mechanisms isn't just academic: it's what separates engineers who build secure systems from those who build systems that get breached.

## Why This Matters

Security vulnerabilities in OS-level constructs lead to the biggest breaches. A single buffer overflow in a privileged process can give an attacker full control of a machine. A misconfigured container capability can let a tenant escape into the host. A missing access control check can expose every customer's data.

Every layer of the modern stack -- containers, cloud VMs, serverless functions -- ultimately relies on OS protection mechanisms: process isolation, memory protection, access control, secure boot. If you're deploying to production, you're relying on these mechanisms whether you know it or not. Understanding them lets you make informed decisions about security architecture instead of cargo-culting configurations from Stack Overflow.

## Prerequisites

| Prerequisite | Why You Need It |
|---|---|
| [02 - Processes](../02-Processes/README.md) | Protection domains, UID/GID, how processes are isolated |
| [07 - Memory Management](../07-Memory-Management/README.md) | Virtual memory isolation, page permissions (NX bit, read-only pages) |
| [09 - File Systems](../09-File-Systems/README.md) | File permissions, ownership, access control in practice |

## Reading Order

| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 1 | Protection Domains and Access Control | [01-protection-domains-and-access-control.md](01-protection-domains-and-access-control.md) | Protection vs security, least privilege, protection rings, access matrix |
| 2 | Authentication and Authorization | [02-authentication-and-authorization.md](02-authentication-and-authorization.md) | Passwords, MFA, Unix permissions, SUID, Linux capabilities, UID/GID model |
| 3 | ACLs and Capabilities | [03-access-control-lists-and-capabilities.md](03-access-control-lists-and-capabilities.md) | Two ways to implement access control, POSIX ACLs, Docker capability dropping |
| 4 | Common Attacks: Buffer Overflow and Privilege Escalation | [04-common-attacks-buffer-overflow-privilege-escalation.md](04-common-attacks-buffer-overflow-privilege-escalation.md) | Stack smashing, ASLR, DEP, stack canaries, ROP, privilege escalation techniques |
| 5 | Sandboxing and Mandatory Access Control | [05-sandboxing-and-mandatory-access-control.md](05-sandboxing-and-mandatory-access-control.md) | DAC vs MAC, SELinux, AppArmor, seccomp, defense in depth |
| 6 | Secure Boot and Trusted Computing | [06-secure-boot-and-trusted-computing.md](06-secure-boot-and-trusted-computing.md) | Chain of trust, UEFI Secure Boot, TPM, disk encryption, remote attestation |

## Key Interview Questions

1. **What's the difference between protection and security in an OS context?**
   Protection is an internal mechanism -- it ensures processes can't accidentally or intentionally interfere with each other (memory isolation, file permissions). Security is about defending against external threats -- malicious users, network attacks, exploits. Protection provides the building blocks (rings, page tables, access control) that security policies use to defend the system.

2. **Explain the principle of least privilege and give a real-world example.**
   Every process should run with the minimum set of privileges needed to do its job. Example: a web server needs to bind port 80 and read files from /var/www, but it doesn't need to load kernel modules or mount filesystems. In Linux, you give it CAP_NET_BIND_SERVICE instead of running as root. Docker applies this by dropping all capabilities except the ones a container actually needs.

3. **How does ASLR defend against buffer overflow attacks, and what are its limitations?**
   ASLR randomizes the memory layout of a process (stack, heap, libraries, executable) on each execution, so an attacker can't predict where their shellcode or ROP gadgets will land. Limitations: 32-bit systems have limited entropy (can be brute-forced), information leaks can reveal addresses, and the main executable may not be position-independent (no PIE). ASLR is necessary but not sufficient -- it works best combined with stack canaries, DEP/NX, and CFI.

4. **What's the difference between DAC and MAC? Why would you use MAC?**
   DAC (Discretionary Access Control) lets resource owners set permissions -- the standard Unix model. MAC (Mandatory Access Control) enforces system-wide policies that even root can't override. MAC matters because if an attacker compromises a process (even root), MAC policies like SELinux or AppArmor still restrict what that process can access. It's defense in depth: even after a breach, the blast radius is limited.

5. **Why does Secure Boot matter for cloud environments?**
   Secure Boot ensures the chain of trust from firmware to bootloader to kernel -- each stage verifies the next before executing. In cloud environments, you're running on shared hardware. Without Secure Boot, a compromised VM could potentially tamper with boot code to persist across reboots or extract secrets from memory. Technologies like AMD SEV and Intel TDX extend this to encrypt VM memory, preventing even the hypervisor from reading it (confidential computing).
