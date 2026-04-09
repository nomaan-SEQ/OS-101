# Access Control Lists and Capabilities

The access matrix from the previous section is a conceptual model -- you can't store a matrix with millions of users and millions of files. There are two practical approaches: store permissions with each **object** (Access Control Lists) or store permissions with each **subject** (Capabilities). Both implement the same matrix, but they have fundamentally different security properties.

## Two Ways to Slice the Access Matrix

```
Access Matrix:
                File1    File2    File3
           +--------+--------+--------+
  Alice    | r, w   | r      |        |
           +--------+--------+--------+
  Bob      | r      |        | r, w   |
           +--------+--------+--------+

ACL approach (store by column -- with each object):
  File1: [(Alice, rw), (Bob, r)]
  File2: [(Alice, r)]
  File3: [(Bob, rw)]

Capability approach (store by row -- with each subject):
  Alice: [(File1, rw), (File2, r)]
  Bob:   [(File1, r), (File3, rw)]
```

## Access Control Lists (ACLs)

An ACL is attached to each **object** (file, directory, device). It lists every subject that can access the object and what they can do.

### Unix: Simplified ACL

Standard Unix permissions are a very compact ACL with only three entries:

```
-rw-r--r--  1  alice  devteam  4096  Apr 1 10:00  config.txt
              |        |
              owner    group

ACL (implicit):
  config.txt: [(alice, rw-), (devteam, r--), (everyone else, r--)]
```

This is simple but inflexible. What if you want alice and bob to have write access, but charlie (also in devteam) should only have read? Standard Unix can't express this.

### POSIX ACLs: Extended Access Control

POSIX ACLs extend the model with per-user and per-group entries:

```bash
# View extended ACL
$ getfacl project/data.csv
# file: project/data.csv
# owner: alice
# group: devteam
user::rw-           # Owner (alice)
user:bob:rw-        # Specific user: bob gets read+write
user:charlie:r--    # Specific user: charlie gets read only
group::r--          # Owning group (devteam)
group:analytics:r-- # Additional group
mask::rw-           # Maximum permissions for named users/groups
other::---          # Everyone else: no access

# Set extended ACL
$ setfacl -m u:bob:rw project/data.csv      # Give bob read+write
$ setfacl -m g:analytics:r project/data.csv  # Give analytics group read
$ setfacl -x u:charlie project/data.csv      # Remove charlie's entry

# Set default ACL for new files in a directory
$ setfacl -d -m g:devteam:rw project/       # New files inherit this ACL
```

The **mask** is a ceiling on permissions for all named users and groups (not the owner). If the mask is `r--`, bob's `rw-` entry is effectively `r--`. This makes it easy to restrict all non-owner access with a single change.

### Windows DACLs: Fine-Grained ACLs

Windows Discretionary ACLs (DACLs) are far more granular than Unix permissions:

```
File: C:\Project\data.csv

DACL entries (Access Control Entries / ACEs):
+----------------------------------------------------+
| Type  | Principal         | Permissions              |
+-------+-------------------+--------------------------+
| Allow | DOMAIN\alice      | Full Control             |
| Allow | DOMAIN\bob        | Read, Write              |
| Deny  | DOMAIN\charlie    | Write                    |
| Allow | DOMAIN\devteam    | Read, Execute            |
| Allow | BUILTIN\Admins    | Full Control             |
+----------------------------------------------------+
```

Key differences from Unix:
- **Explicit Deny** entries (Unix has no deny -- only grant or absence of grant)
- **Inheritance** from parent directories (complex but powerful)
- **Many more permission types**: Read, Write, Execute, Delete, Change Permissions, Take Ownership, etc.
- **Order matters**: Deny entries are evaluated before Allow entries

### ACL Properties

**Strengths:**
- Easy to answer "who can access this file?" -- just read the ACL
- Easy to change permissions on an object -- modify one ACL
- Natural fit for file systems (permissions stored with the file's inode)

**Weaknesses:**
- Hard to answer "what can this user access?" -- must scan ALL ACLs on ALL objects
- Hard to revoke ALL of a user's access -- must find and modify every ACL they appear in
- Vulnerable to the **confused deputy problem** (more on this below)

## Capabilities

A capability is an **unforgeable token** held by a subject that grants specific rights to a specific object. Think of it like a key: whoever holds the key can open the door, regardless of who they are.

```
Analogy:

ACL approach:                          Capability approach:
+----------+                           +----------+
| Door     |                           | Alice    |
| Sign:    |                           | Keyring: |
| Alice: Y |                           |  [Key A] | ---> opens Door A
| Bob:  Y  |                           |  [Key B] | ---> opens Door B
| Eve:  N  |                           +----------+
+----------+
                                       +----------+
The door checks                        | Bob      |
who you are.                           | Keyring: |
                                       |  [Key A] | ---> opens Door A
                                       +----------+

                                       You present a key.
                                       The door doesn't care who you are.
```

### Properties of Capabilities

For a capability system to work, capabilities must be:
1. **Unforgeable** -- you can't create a capability from nothing
2. **Transferable** (in some systems) -- you can pass your capability to another process
3. **Specific** -- a capability names a specific object and specific rights

### Linux Capabilities: A Practical Example

Linux capabilities aren't pure capability tokens -- they're a capability-like system that breaks root into fine-grained permissions. Each process has three capability sets:

```
Process Capability Sets:
+------------------------------------------+
| Permitted    | Maximum capabilities the   |
|              | process CAN have           |
+--------------+----------------------------+
| Effective    | Capabilities currently     |
|              | ACTIVE (checked by kernel) |
+--------------+----------------------------+
| Inheritable  | Capabilities that can be   |
|              | inherited across exec()    |
+--------------+----------------------------+
```

```bash
# View capabilities of a running process
$ cat /proc/self/status | grep Cap
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 000001ffffffffff
CapAmb: 0000000000000000

# Decode capability bitmask
$ capsh --decode=000001ffffffffff
0x000001ffffffffff=cap_chown,cap_dac_override,cap_dac_read_search,...

# Key capabilities
CAP_NET_BIND_SERVICE  # Bind to ports < 1024
CAP_NET_RAW           # Use raw sockets (ping)
CAP_SYS_ADMIN         # Mount, sethostname, many admin ops (the "new root")
CAP_SYS_PTRACE        # Trace any process (strace, gdb)
CAP_NET_ADMIN          # Configure networking (iptables)
CAP_SYS_MODULE         # Load kernel modules
CAP_SYS_RAWIO          # Direct I/O access (/dev/mem)
```

### Container Capability Dropping

Docker starts containers with a reduced capability set by default, and you can restrict further:

```bash
# Docker's default: drops many dangerous capabilities
# By default, containers do NOT get:
#   CAP_SYS_ADMIN, CAP_NET_ADMIN, CAP_SYS_MODULE, etc.

# Drop ALL capabilities, add back only what's needed
$ docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# View container capabilities
$ docker run --rm alpine cat /proc/1/status | grep Cap
```

## ACL vs Capability: The Tradeoffs

| Property | ACL | Capability |
|----------|-----|------------|
| **Stored with** | Object (file, resource) | Subject (process, user) |
| **Answers easily** | "Who can access this file?" | "What can this process access?" |
| **Revocation** | Easy per-object, hard per-user | Hard (must find and revoke all tokens) |
| **Delegation** | Hard (need to modify object's ACL) | Easy (just pass the token) |
| **Ambient authority** | Yes -- identity-based decisions | No -- must explicitly present capability |
| **Confused deputy** | Vulnerable | Resistant |
| **Real examples** | Unix permissions, Windows DACL, POSIX ACLs | Linux capabilities, file descriptors, JWT tokens |

## The Confused Deputy Problem

This is the key theoretical difference between ACLs and capabilities, and it has real practical consequences.

```
The Setup:
  - Compiler runs with privileges to write to /billing (for logging)
  - User asks compiler to output to /billing/important_data
  - Compiler checks: "Can I write to /billing/important_data?" -- YES (it has access)
  - Compiler overwrites critical billing data

  The compiler (the "deputy") was confused about WHETHER it should use
  its own authority vs the user's authority.

+-----------+     "Compile foo.c, output to /billing/data"     +----------+
|           +------------------------------------------------->|          |
|   User    |                                                   | Compiler |
| (no /billing |                                                | (has /billing |
|  access)  |                                                   |  access)     |
+-----------+                                                   +-----+----+
                                                                      |
                                              Writes to /billing/data |
                                              using ITS OWN authority |
                                                                      v
                                                              /billing/data
                                                              (overwritten!)
```

**With capabilities**, this can't happen. The user would need to pass a capability for `/billing/data` to the compiler. Since the user doesn't have that capability, they can't pass it, and the compiler won't use its own billing capability for user-requested operations.

**File descriptors** in Unix are actually capabilities: when you open a file, the kernel gives you an integer (the fd) that grants specific access. You can pass this fd to another process (via fork or Unix socket). The receiving process can use the fd without having permission to open the file itself.

## Real-World Connection

**Docker --cap-drop/--cap-add**: Production containers should drop all capabilities and add back only what's needed. A container running a web server needs CAP_NET_BIND_SERVICE (for port 80) and nothing else. This is defense in depth -- even if the container is compromised, the attacker can't mount filesystems, load kernel modules, or trace other processes.

```yaml
# Kubernetes securityContext
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
  runAsNonRoot: true
  readOnlyRootFilesystem: true
```

**Cloud IAM policies as ACLs**: AWS S3 bucket policies are ACLs -- they list who can do what with the bucket. IAM roles with attached policies are capability-like -- the role carries permissions that any process assuming the role inherits.

**POSIX ACLs in shared storage**: NFS and shared filesystems in enterprise environments rely on POSIX ACLs for fine-grained access control beyond the basic owner/group/others model. Teams with complex sharing needs (multiple groups with different access levels) can't get by with standard Unix permissions alone.

**JWT tokens as capabilities**: A JWT token in a web application is conceptually a capability -- it's an unforgeable token (signed) that grants specific access (claims/scopes) to specific resources. The server doesn't look up who you are in a database -- it validates the token you present. This is the capability model applied to web services.

## Interview Angle

**Q: What's the difference between an ACL and a capability?**

A: An ACL is stored with the object (file/resource) and lists which subjects can access it -- like a guest list at a door. A capability is stored with the subject (process/user) and grants access to a specific object -- like a key. ACLs make it easy to see who can access a resource and to revoke per-resource access, but hard to revoke all of a user's access. Capabilities make it easy to delegate access (pass a token) and resist the confused deputy problem, but make revocation harder.

**Q: What is the confused deputy problem?**

A: It occurs when a privileged program (the "deputy") is tricked into misusing its authority on behalf of an unprivileged caller. Classic example: a compiler with write access to a billing directory is asked by a user (who has no billing access) to output to that directory. With ACLs, the compiler checks its own permissions and succeeds. With capabilities, the user would need to provide a capability token for the billing directory -- which they don't have. Capabilities solve this by requiring explicit presentation of authority rather than relying on ambient identity-based permissions.

**Q: How does Docker use capabilities for container security?**

A: Docker starts containers with a reduced set of Linux capabilities -- it drops dangerous ones like CAP_SYS_ADMIN (mount, namespace operations), CAP_NET_ADMIN (network configuration), and CAP_SYS_MODULE (kernel module loading) by default. For production, best practice is `--cap-drop=ALL --cap-add=<only what's needed>`. In Kubernetes, this is configured through the securityContext in pod specs. This ensures that even if an attacker compromises the container, they can't perform privileged kernel operations.

---

**Next:** [Common Attacks: Buffer Overflow and Privilege Escalation](04-common-attacks-buffer-overflow-privilege-escalation.md) -- how attackers break through these protection mechanisms.
