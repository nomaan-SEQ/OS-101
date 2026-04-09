# Linux Security: LSM, SELinux, AppArmor, and Beyond

## The Core Idea

Traditional Unix security is **Discretionary Access Control (DAC)** — file owners set permissions (rwx) and any process running as root bypasses all checks. This model has a fundamental weakness: if an attacker compromises any root process, they own the system. A single vulnerability in Apache running as root gives the attacker unrestricted access to everything.

Linux addresses this with multiple layers of security hardening: **Linux Security Modules (LSM)** for mandatory access control, **capabilities** for fine-grained privilege splitting, and **seccomp-bpf** for syscall filtering. Together, these create a defense-in-depth strategy where compromising one layer does not grant full system access.

---

## Linux Security Modules (LSM) Framework

LSM is a framework that inserts **security hook points** throughout the kernel. Before the kernel performs a security-sensitive operation (opening a file, creating a process, binding a socket, mounting a filesystem), it calls the registered LSM hooks, giving security modules a chance to deny the operation.

**LSM is like security checkpoints throughout an airport. At each critical point — entering the terminal (process creation), accessing a gate (file access), boarding a plane (network connection) — a security guard (LSM hook) checks your credentials against the policy. The guard can wave you through or deny access, but the airport itself (kernel) does not need to understand the security logic — it just calls the guard at each checkpoint.**

```
Application calls open("/etc/shadow", O_RDONLY)
         │
         ▼
    VFS: vfs_open()
         │
         ▼
    DAC check: file permissions (rwx) + owner/group
         │ (passes)
         ▼
    LSM hook: security_file_open()
         │
         ├── SELinux: Does this process's domain have
         │            read access to shadow_t type?
         │            → Check policy → DENY
         │
         ├── AppArmor: Does this process's profile
         │             allow reading /etc/shadow?
         │             → Check profile → DENY
         │
         └── No LSM: No further checks → ALLOW
```

LSM hooks are placed at over 200 points in the kernel. Multiple LSMs can **stack** — the operation must pass ALL of them:

| LSM | Type | Stacking Role |
|-----|------|---------------|
| SELinux | Major (label-based MAC) | Primary (exclusive with AppArmor) |
| AppArmor | Major (path-based MAC) | Primary (exclusive with SELinux) |
| Smack | Major (simplified MAC) | Primary (alternative to SELinux/AppArmor) |
| TOMOYO | Major (path-based MAC) | Primary (alternative) |
| Yama | Minor (ptrace restrictions) | Stacks with any major LSM |
| Lockdown | Minor (kernel integrity) | Stacks with any major LSM |
| Landlock | Minor (unprivileged sandboxing) | Stacks with any major LSM |
| BPF-LSM | Minor (programmable hooks) | Stacks with any major LSM |

---

## SELinux Deep Dive

SELinux (Security-Enhanced Linux) was developed by the NSA and is the most powerful (and most complex) LSM. It implements **Type Enforcement** — every process runs in a **domain** and every file has a **type**, and a policy matrix defines exactly which domains can access which types with which operations.

### Security Contexts

Every process and file has a security context (label) with four fields:

```
user : role : type : level

Examples:
  Process:  system_u:system_r:httpd_t:s0
  File:     system_u:object_r:httpd_content_t:s0

  user    = SELinux user (system_u, unconfined_u, staff_u)
  role    = SELinux role (system_r, object_r, staff_r)
  type    = The critical field — the domain (process) or type (file)
  level   = MLS/MCS sensitivity level (s0, s0-s15:c0.c1023)
```

The **type** field does most of the work. Policy rules reference types:

```
# Allow the httpd_t domain to read files of type httpd_content_t
allow httpd_t httpd_content_t:file { read open getattr };

# Allow httpd_t to connect to TCP port type http_port_t
allow httpd_t http_port_t:tcp_socket { name_connect };

# Deny everything else by default (deny is implicit)
```

### Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Enforcing** | Denies and logs policy violations | Production |
| **Permissive** | Logs but allows violations | Policy development and debugging |
| **Disabled** | SELinux completely off | Not recommended |

```bash
# Check current mode:
getenforce

# Temporarily switch to permissive:
sudo setenforce 0

# Check a file's context:
ls -Z /var/www/html/index.html
# system_u:object_r:httpd_content_t:s0 /var/www/html/index.html

# Check a process's context:
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0  1234 ?  httpd
```

### The audit2allow Workflow

When SELinux denies an operation, it logs an AVC (Access Vector Cache) denial in `/var/log/audit/audit.log`:

```
type=AVC msg=audit(...): avc:  denied  { read } for
  pid=1234 comm="httpd"
  name="custom.conf" dev="sda1" ino=56789
  scontext=system_u:system_r:httpd_t:s0
  tcontext=system_u:object_r:default_t:s0
  tclass=file permissive=0
```

The `audit2allow` tool reads these denials and generates policy rules:

```bash
# Generate a policy module from recent denials:
sudo audit2allow -a -M my_httpd_fix

# Review the generated policy (ALWAYS review before installing):
cat my_httpd_fix.te

# Install the policy module:
sudo semodule -i my_httpd_fix.pp
```

SELinux has a reputation for being complex — and it is. The default policies (targeted policy in RHEL/Fedora) cover hundreds of applications with thousands of rules. But the reward is genuine Mandatory Access Control: even if an attacker gains root through a vulnerability in httpd, they are still confined to the `httpd_t` domain and can only access types that the policy allows.

---

## AppArmor

AppArmor takes a fundamentally different approach from SELinux: instead of labeling every file and process, it uses **path-based profiles** that define what each program can access by file path.

```
# /etc/apparmor.d/usr.sbin.nginx
/usr/sbin/nginx {
    # Capabilities
    capability net_bind_service,
    capability setuid,
    capability setgid,

    # Binary and libraries
    /usr/sbin/nginx mr,
    /lib/x86_64-linux-gnu/** mr,

    # Web content (read-only)
    /var/www/** r,
    /srv/www/** r,

    # Configuration
    /etc/nginx/** r,

    # Logs (append only)
    /var/log/nginx/*.log w,

    # PID and socket files
    /run/nginx.pid rw,
    /run/nginx/*.sock rw,

    # Network
    network inet stream,
    network inet6 stream,

    # Deny everything else (implicit)
}
```

| Aspect | SELinux | AppArmor |
|--------|---------|----------|
| Model | Label-based (type enforcement) | Path-based (file paths in profiles) |
| Granularity | Very fine (types, roles, MLS levels) | Medium (paths, capabilities, network) |
| Complexity | High (steep learning curve) | Lower (profiles are readable) |
| Filesystem support | Works on any filesystem (labels in xattrs) | Path-based — renaming files changes access |
| Default distro | RHEL, Fedora, CentOS | Ubuntu, SUSE, Debian |
| Profile creation | Complex policy language | `aa-genprof` (guided interactive tool) |

### Profile Generation

```bash
# Start interactive profile generation for nginx:
sudo aa-genprof /usr/sbin/nginx
# (Use the application normally, aa-genprof monitors denials and asks you to allow/deny)

# Update profile from recent logs:
sudo aa-logprof

# Set profile to enforce mode:
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx

# Set to complain mode (like SELinux permissive — log but allow):
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx
```

---

## seccomp-bpf: Syscall Filtering

seccomp-bpf goes deeper than file access control — it restricts which **system calls** a process can make. A BPF (Berkeley Packet Filter) program inspects each syscall number and its arguments, then decides what to do:

| Action | Effect |
|--------|--------|
| `SCMP_ACT_ALLOW` | Allow the syscall |
| `SCMP_ACT_KILL` | Kill the process immediately |
| `SCMP_ACT_KILL_THREAD` | Kill only the calling thread |
| `SCMP_ACT_TRAP` | Send SIGSYS signal (can be caught) |
| `SCMP_ACT_ERRNO(val)` | Return an error code instead of executing |
| `SCMP_ACT_LOG` | Allow but log the syscall |
| `SCMP_ACT_TRACE` | Notify a ptrace tracer |

### Docker's seccomp Profile

Docker applies a default seccomp profile that blocks approximately 44 dangerous syscalls:

```
Blocked syscalls include:
- mount, umount2         Can't mount filesystems
- reboot                 Can't reboot the host
- kexec_load            Can't load a new kernel
- init_module, delete_module  Can't load kernel modules
- swapon, swapoff       Can't manage swap
- ptrace                Can't trace other processes (by default)
- keyctl                Can't access kernel keyring
- bpf                   Can't load BPF programs
- userfaultfd           Can't use user-space page fault handling
```

This means even if an attacker breaks out of application sandboxing inside a container, they cannot load kernel modules, mount arbitrary filesystems, or reboot the host. The syscall is rejected at the kernel level before it can do anything.

### Writing seccomp Filters

```c
// Using libseccomp (high-level API):
scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);  // Default: kill on any syscall

// Whitelist only the syscalls we need:
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

// Apply the filter:
seccomp_load(ctx);

// From this point, ONLY read, write, exit, and exit_group are allowed.
// Any other syscall kills the process.
```

---

## Capabilities: Splitting Root Power

Traditional Unix has a binary privilege model: you are root (can do everything) or you are not (restricted by DAC). Linux capabilities split root's power into ~40 individual privileges:

| Capability | What It Grants |
|-----------|---------------|
| `CAP_NET_BIND_SERVICE` | Bind to ports below 1024 |
| `CAP_NET_RAW` | Use raw sockets (ping, packet capture) |
| `CAP_NET_ADMIN` | Network configuration (routes, iptables, interfaces) |
| `CAP_SYS_ADMIN` | Catch-all: mount, swapon, namespaces, and ~30 more operations |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permission checks |
| `CAP_DAC_READ_SEARCH` | Bypass file read and directory search permissions |
| `CAP_SETUID` / `CAP_SETGID` | Change process UID/GID |
| `CAP_KILL` | Send signals to any process |
| `CAP_SYS_PTRACE` | Trace any process with ptrace |
| `CAP_SYS_CHROOT` | Use chroot() |
| `CAP_CHOWN` | Change file ownership |
| `CAP_FOWNER` | Bypass ownership checks on files |

### Why CAP_SYS_ADMIN is Dangerous

`CAP_SYS_ADMIN` is essentially "everything that did not get its own capability." It grants mount, swapon, namespace creation, BPF program loading, device configuration, and dozens more operations. Granting `CAP_SYS_ADMIN` to a process is almost as dangerous as giving it full root. When reviewing container security, always check whether `CAP_SYS_ADMIN` is in the capability set.

### File Capabilities

Instead of making a binary setuid-root, you can grant it specific capabilities:

```bash
# Allow a program to bind low ports without root:
sudo setcap cap_net_bind_service=+ep /usr/bin/myserver

# Check capabilities on a file:
getcap /usr/bin/myserver
# /usr/bin/myserver cap_net_bind_service=ep

# Remove capabilities:
sudo setcap -r /usr/bin/myserver
```

Each process has three capability sets:
- **Permitted**: Maximum capabilities the process can use
- **Effective**: Currently active capabilities (checked by the kernel)
- **Inheritable**: Capabilities passed across `exec()`

---

## Landlock: Unprivileged Sandboxing

Landlock (merged in kernel 5.13) is the newest LSM, and it has a unique property: **unprivileged applications can use it to restrict themselves.** No root, no policy files, no admin configuration needed.

A program can call Landlock to say "from this point on, I can only access these directories and these ports." This is sandboxing from the inside — the application gives up its own privileges.

```
Before Landlock:
  Application can access: /etc, /usr, /home, /tmp, network, everything

Application calls landlock_restrict_self():
  "I only need: /var/www (read), /tmp (read+write), port 443 (connect)"

After Landlock:
  Application can ONLY access: /var/www (r), /tmp (rw), port 443
  Everything else is denied, even if DAC permissions would allow it
  The restriction is PERMANENT for this process — cannot be lifted
```

Landlock stacks with other LSMs (SELinux, AppArmor) — the most restrictive policy wins.

---

## Real-World Connection

**SELinux saving the day:** The 2014 Shellshock vulnerability in Bash affected millions of servers. On systems with SELinux in enforcing mode, the impact was dramatically reduced — even though the attacker could execute arbitrary commands through Bash, SELinux's type enforcement prevented those commands from accessing files and network resources outside the compromised process's domain. The exploit worked, but the blast radius was contained.

**Container security layers:** A well-secured Docker container uses ALL of these mechanisms simultaneously: user namespaces (UID mapping), seccomp-bpf (syscall filtering), capabilities (dropped to minimum), and AppArmor or SELinux (MAC confinement). Each layer catches what the others miss. This defense-in-depth is why container escapes are rare when security features are properly configured.

**Capabilities in practice:** Many distributions now ship programs with file capabilities instead of setuid root. `ping` used to be setuid root (it needs raw sockets). Modern Linux systems give it just `CAP_NET_RAW`: `getcap /usr/bin/ping` shows `cap_net_raw=ep`. If `ping` has a vulnerability, the attacker gets raw socket access — not root.

---

## Interview Angle

**Q: What is the difference between DAC and MAC in Linux?**

A: DAC (Discretionary Access Control) is the traditional Unix permission model — file owners set rwx permissions, and root bypasses everything. It is "discretionary" because the owner decides who has access. MAC (Mandatory Access Control), implemented by LSMs like SELinux and AppArmor, enforces system-wide policies that even root cannot override. A process running as root but confined by SELinux to the `httpd_t` domain still cannot read files of type `shadow_t`. MAC adds a layer that DAC cannot provide: protection against compromised privileged processes.

**Q: How does seccomp-bpf protect containers?**

A: seccomp-bpf installs a BPF filter that inspects every syscall a process makes. Docker's default profile blocks approximately 44 dangerous syscalls (mount, reboot, kexec_load, init_module, etc.). Even if an attacker achieves code execution inside a container, they cannot make these blocked syscalls — the kernel rejects them before execution. This prevents container escape vectors that rely on mounting filesystems, loading kernel modules, or manipulating kernel state. The filter applies to the process and all its children, and cannot be removed once installed.

**Q: What is the difference between SELinux and AppArmor?**

A: SELinux uses labels (security contexts) attached to files and processes, with a policy matrix defining allowed interactions between domains (process labels) and types (file labels). It is very powerful and fine-grained but has a steep learning curve. AppArmor uses path-based profiles that reference file paths rather than labels — this is simpler to understand and write but has limitations (renaming a file changes its access path). SELinux is default on RHEL/Fedora; AppArmor is default on Ubuntu/SUSE. Both implement MAC (Mandatory Access Control) through the LSM framework.

---

**Next**: [07-linux-boot-process-in-depth.md](07-linux-boot-process-in-depth.md)
