# Sandboxing and Mandatory Access Control

Standard Unix permissions (DAC) have a fundamental weakness: if an attacker compromises a process, they inherit all of that process's permissions. If it's running as root, the attacker owns the system. Mandatory Access Control (MAC) and sandboxing techniques add layers of defense that restrict processes regardless of their UID -- even root can be confined.

## DAC vs MAC

| Property | DAC (Discretionary) | MAC (Mandatory) |
|----------|-------------------|-----------------|
| **Who sets policy** | Resource owner (user) | System administrator / policy |
| **Can root bypass?** | Yes -- root bypasses all DAC checks | No -- MAC restricts even root |
| **Default in** | Standard Unix, Windows DACL | SELinux, AppArmor, Tomoyo |
| **Philosophy** | Trust users to manage their own files | Don't trust any process, even privileged ones |
| **After compromise** | Attacker gets all of process's permissions | Attacker is still confined by MAC policy |

**Why MAC matters:** DAC assumes the process is behaving correctly. But after a compromise, the process is under attacker control -- DAC permissions become the attacker's permissions. MAC enforces external policies that the compromised process can't change, limiting the blast radius of any breach.

```
DAC only:                              DAC + MAC:

Attacker compromises                   Attacker compromises
web server (root) -->                  web server (root) -->
  Can read /etc/shadow     YES           Can read /etc/shadow     NO (MAC blocks)
  Can load kernel modules  YES           Can load kernel modules  NO (MAC blocks)
  Can access other apps    YES           Can access other apps    NO (MAC blocks)
  Can modify web files     YES           Can modify web files     NO (MAC: read-only)
  Can access network       YES           Can access network       YES (MAC allows)
  Can serve HTTP           YES           Can serve HTTP           YES (MAC allows)
```

## SELinux: Label-Based MAC

SELinux (Security-Enhanced Linux) assigns a **security context (label)** to every process and every file. A central policy defines which labels can access which:

```
Every process has a label:
  system_u:system_r:httpd_t:s0
                    ^^^^^^^
                    This is the TYPE -- the main part used in policy

Every file has a label:
  system_u:object_r:httpd_sys_content_t:s0
                    ^^^^^^^^^^^^^^^^^^
                    File type

Policy rule:
  allow httpd_t httpd_sys_content_t:file { read open getattr };
  
  Translation: processes labeled httpd_t can read files labeled
               httpd_sys_content_t. That's it. Nothing else.

This is called **type enforcement** -- the core mechanism of SELinux. Every access decision is based on the *type* label of the source process and the *type* label of the target resource.

> **Analogy:** Type enforcement is like a hospital badge system. Doctors (type: doctor_t) can access patient records (type: patient_record_t), but cafeteria staff (type: cafe_t) cannot -- even if both are hospital employees. The badge type determines access, not the person's name.
```

### SELinux in Practice

```bash
# Check if SELinux is enabled and in what mode
$ getenforce
Enforcing          # Enforcing = blocks violations
                   # Permissive = logs but allows (for debugging)
                   # Disabled = off

# View process labels
$ ps -eZ | grep httpd
system_u:system_r:httpd_t:s0     1234 ?  00:00:01 httpd

# View file labels
$ ls -Z /var/www/html/index.html
system_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html

# Check for SELinux denials in the audit log
$ ausearch -m avc -ts recent
type=AVC msg=... avc:  denied  { read } for ... scontext=httpd_t tcontext=shadow_t

# Fix a file's label (common troubleshooting)
$ restorecon -v /var/www/html/newfile.html
```

### SELinux Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Enforcing** | Blocks and logs policy violations | Production |
| **Permissive** | Logs violations but allows them | Debugging, policy development |
| **Disabled** | No SELinux at all | Not recommended |

A common mistake: switching to permissive mode "because SELinux was blocking my app." This defeats the purpose. The correct approach is to identify the denial and either fix the file labels or add a policy rule.

## AppArmor: Path-Based MAC

AppArmor is simpler than SELinux. Instead of labels, it uses **file paths** in profiles:

```
# /etc/apparmor.d/usr.sbin.nginx
/usr/sbin/nginx {
    # Network access
    network inet stream,
    network inet6 stream,
    
    # File access
    /var/www/** r,                    # Read web content
    /var/log/nginx/** w,              # Write logs
    /etc/nginx/** r,                  # Read config
    /run/nginx.pid rw,                # PID file
    
    # Deny everything else implicitly
    # No access to /etc/shadow, /home, /root, etc.
}
```

```bash
# Check AppArmor status
$ sudo aa-status
apparmor module is loaded.
34 profiles are loaded.
34 profiles are in enforce mode.
   /usr/sbin/nginx
   /usr/bin/evince
   ...

# Put a profile in complain mode (like SELinux permissive, per-profile)
$ sudo aa-complain /usr/sbin/nginx

# Generate a profile interactively
$ sudo aa-genprof /usr/sbin/myapp
```

### SELinux vs AppArmor

| Feature | SELinux | AppArmor |
|---------|---------|----------|
| **Policy basis** | Labels on files and processes | File paths |
| **Granularity** | Very fine-grained | Moderate |
| **Complexity** | High -- steep learning curve | Lower -- easier to write profiles |
| **Hardlink handling** | Safe (labels follow inode) | Vulnerable (path changes bypass) |
| **Default on** | RHEL, CentOS, Fedora | Ubuntu, SUSE, Debian |
| **Approach** | Whitelist everything explicitly | Profile per application |
| **Network control** | Per-port, per-protocol | Per-protocol only |

## seccomp: System Call Filtering

seccomp restricts **which system calls** a process can make. This is a different layer than file-based MAC -- it controls what kernel operations are available at all.

### seccomp Modes

```
seccomp-strict (original):
  Process can ONLY use: read(), write(), exit(), sigreturn()
  Everything else --> SIGKILL
  Too restrictive for most applications.

seccomp-bpf (modern):
  **seccomp-bpf** stands for "secure computing mode with Berkeley Packet Filter." It repurposes BPF -- originally designed for filtering network packets -- as a programmable filter for system calls. The process loads a small BPF program into the kernel that inspects each syscall and decides what to do.
  Process installs a BPF filter program that decides per-syscall:
  - ALLOW: let the syscall proceed
  - KILL: terminate the process
  - ERRNO: return an error code
  - TRAP: send a signal
  - LOG: allow but log
```

### seccomp-bpf in Action

```
Process makes a syscall
        |
        v
+-------------------+
| seccomp BPF filter|
| (runs in kernel)  |
+---+-------+-------+
    |       |       |
  ALLOW   KILL    ERRNO
    |       |       |
    v       v       v
 Proceed  SIGKILL  Return
 normally          -EPERM
```

```bash
# Docker's default seccomp profile blocks ~44 syscalls including:
#   - mount/umount (no filesystem mounting)
#   - reboot (can't reboot host)
#   - kexec_load (can't load new kernel)
#   - keyctl (can't access kernel keyring)
#   - bpf (can't load BPF programs)
#   - userfaultfd (used in some exploits)

# Run Docker container with NO seccomp (dangerous -- don't do this in prod)
$ docker run --security-opt seccomp=unconfined myimage

# Run with custom seccomp profile
$ docker run --security-opt seccomp=my-profile.json myimage
```

## Sandboxing Approaches: Building Layers

Effective sandboxing combines multiple mechanisms. No single technique is sufficient:

### chroot: The Original Sandbox

```bash
$ chroot /sandbox/myapp /bin/myapp
# Process sees /sandbox/myapp as its root (/)
# Cannot access files above /sandbox/myapp

File system view:
  Host:                    Process sees:
  /                        /sandbox/myapp  -->  /
  /etc/passwd              (inaccessible)
  /sandbox/myapp/          /
  /sandbox/myapp/bin/      /bin/
  /sandbox/myapp/lib/      /lib/
```

**Limitations:** chroot only restricts the filesystem view. A root process can escape chroot with `mkdir()+chroot()+chdir()`. It doesn't restrict network, PIDs, or system calls. It's not a security boundary -- it's a convenience.

### Linux Namespaces: Isolating System Resources

```
+------------------------------------------+
| Host System                              |
|  PID namespace:  1, 2, 3, 1000, ...     |
|  Network:        eth0, 10.0.0.1          |
|  Mounts:         /, /home, /var          |
|  Users:          root(0), alice(1000)    |
|                                          |
|  +----------------------------------+   |
|  | Container (separate namespaces)   |   |
|  |  PID:     1, 2, 3 (own numbering)|   |
|  |  Network: eth0, 172.17.0.2       |   |
|  |  Mounts:  / (own root fs)        |   |
|  |  Users:   root(0) mapped to      |   |
|  |           unprivileged host user  |   |
|  +----------------------------------+   |
+------------------------------------------+
```

| Namespace | What It Isolates | Effect |
|-----------|-----------------|--------|
| **PID** | Process IDs | Container has its own PID 1 |
| **NET** | Network stack | Own IP address, routing, iptables |
| **MNT** | Mount points | Own filesystem view |
| **UTS** | Hostname | Own hostname |
| **IPC** | IPC resources | Own shared memory, semaphores |
| **USER** | UID/GID mapping | Root in container = unprivileged on host |
| **cgroup** | cgroup root | Own cgroup hierarchy |

### Combining Techniques: Defense in Depth

```
Chrome Browser Sandbox:
+-------------------------------------------------------+
| Browser Process (privileged)                          |
|  - Full filesystem access                             |
|  - Network access                                     |
|  - Manages all renderer processes                     |
+-------------------------------------------------------+
        |  IPC (Mojo)
        v
+-------------------------------------------------------+
| Renderer Process (sandboxed)                          |
|  Layer 1: Linux namespaces (PID, NET, USER)          |
|  Layer 2: seccomp-bpf (minimal syscalls)             |
|  Layer 3: No filesystem access (all I/O through IPC) |
|  Layer 4: No network access (browser proxies all)    |
|                                                       |
|  Even if attacker gets code execution here,           |
|  they can't read files, access network, or            |
|  see other processes.                                 |
+-------------------------------------------------------+
```

```
Docker Container Isolation Stack:
+-------------------------------------------------------+
|  Application                                          |
+-------------------------------------------------------+
|  Linux Capabilities (CAP_DROP=ALL, add only needed)   |
+-------------------------------------------------------+
|  seccomp-bpf (block dangerous syscalls)               |
+-------------------------------------------------------+
|  AppArmor or SELinux (restrict file/network access)   |
+-------------------------------------------------------+
|  Linux Namespaces (PID, NET, MNT, USER, UTS, IPC)    |
+-------------------------------------------------------+
|  cgroups (resource limits -- CPU, memory, I/O)        |
+-------------------------------------------------------+
|  Host Kernel                                          |
+-------------------------------------------------------+
```

## Comparison Table: Sandboxing Technologies

| Technology | What It Restricts | Granularity | Escape Difficulty | Setup Complexity |
|-----------|-------------------|-------------|-------------------|-----------------|
| **chroot** | Filesystem view | Coarse | Easy (root can escape) | Trivial |
| **Namespaces** | PIDs, network, mounts, users | Medium | Requires kernel vuln | Moderate |
| **seccomp-bpf** | Available syscalls | Fine (per-syscall) | Requires allowed syscall vuln | Moderate |
| **SELinux** | File/network/IPC access by label | Very fine | Requires policy bypass | High |
| **AppArmor** | File/network access by path | Fine | Path manipulation possible | Moderate |
| **Capabilities** | Root-level operations | Fine (per-capability) | Depends on caps granted | Low |

## Real-World Connection

**Docker's multi-layer security**: Docker doesn't rely on any single isolation mechanism. It combines namespaces (isolated view), cgroups (resource limits), capabilities (reduced root powers), seccomp (restricted syscalls), and optional MAC (AppArmor/SELinux). Each layer catches what the others miss. This is why `--privileged` is so dangerous -- it disables most of these layers.

**Chrome's multi-process sandbox**: Chrome is one of the most aggressive sandboxing users. Renderer processes (which parse untrusted web content) run with seccomp-bpf allowing only ~30 syscalls, in their own PID/network/user namespaces, with no direct filesystem or network access. Every Chrome security advisory that says "sandbox escape" means the attacker chained two exploits -- one for code execution, one to break out of this sandbox.

**Kubernetes Pod Security Standards**: Kubernetes defines three security levels -- Privileged (no restrictions), Baseline (minimal restrictions), and Restricted (hardened). The Restricted profile requires: non-root user, read-only root filesystem, no privilege escalation, dropped capabilities, seccomp profile. These map directly to the OS mechanisms in this section.

**SELinux and containers in RHEL**: Red Hat Enterprise Linux runs SELinux in enforcing mode by default. Each container gets its own SELinux label (container_t), which prevents it from accessing host files (labeled differently). Even if an attacker escapes the container namespace, SELinux blocks access to host resources. This is why RHEL is preferred for high-security environments.

## Interview Angle

**Q: What's the difference between DAC and MAC?**

A: DAC (Discretionary Access Control) lets resource owners set permissions -- the standard Unix chmod model. The problem: if an attacker compromises a root process, DAC provides no further barrier because root bypasses all checks. MAC (Mandatory Access Control) enforces system-wide policies that even root can't override. SELinux and AppArmor are MAC implementations -- they define what each process label/profile can access regardless of UID. MAC provides defense in depth: even after a compromise, the attacker is confined to the compromised process's MAC policy.

**Q: How does seccomp-bpf work and why is it important for container security?**

A: seccomp-bpf lets a process install a BPF filter program that the kernel evaluates on every system call. The filter can allow, block, or log each syscall based on the syscall number and arguments. Docker's default seccomp profile blocks about 44 of the ~300+ Linux syscalls -- including mount, reboot, kexec_load, and others that could compromise the host. This prevents a compromised container from using dangerous kernel interfaces even if it has the right capabilities and MAC permissions.

**Q: Compare SELinux and AppArmor. When would you choose each?**

A: SELinux is label-based -- every file and process gets a security label, and policy rules define which label pairs are allowed. It's very granular but complex to configure. AppArmor is path-based -- profiles specify which file paths a program can access. It's simpler to write and understand but less robust (hardlinks can bypass path-based policies). Choose SELinux for high-security environments where you need fine-grained control (RHEL, government, finance). Choose AppArmor for developer-friendly environments where ease of policy writing matters more (Ubuntu, application sandboxing).

**Q: What makes Docker's `--privileged` flag dangerous?**

A: `--privileged` disables almost all container security: it grants all Linux capabilities, removes the seccomp filter, disables AppArmor/SELinux confinement, and gives the container access to all host devices. The container effectively runs with the same privileges as a host root process. A compromised `--privileged` container can mount the host filesystem, load kernel modules, access hardware directly, and escape trivially. Never use `--privileged` in production -- use `--cap-add` for specific capabilities instead.

---

**Next:** [Secure Boot and Trusted Computing](06-secure-boot-and-trusted-computing.md) -- ensuring the system hasn't been tampered with before it even boots.
