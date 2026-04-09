# Authentication and Authorization

Authentication answers "who are you?" Authorization answers "what can you do?" These are the two fundamental questions every OS asks before granting access to anything. Get authentication wrong and attackers impersonate legitimate users. Get authorization wrong and legitimate users access things they shouldn't.

## Authentication: Proving Identity

Authentication verifies that a subject (user, process, service) is who they claim to be. There are three factors:

| Factor | What It Is | Examples | Strength |
|--------|-----------|----------|----------|
| **Something you know** | A secret the user memorized | Password, PIN, passphrase | Weakest -- can be guessed, phished, leaked |
| **Something you have** | A physical object the user possesses | Hardware key (YubiKey), phone (TOTP), smart card | Medium -- must be physically stolen |
| **Something you are** | A biometric characteristic | Fingerprint, face scan, iris | Strong but not revocable -- you can't change your fingerprint |

**Multi-factor authentication (MFA)** combines two or more factors. Your bank card + PIN is two-factor (have + know). SSH key + password is two-factor (have + know). A password alone is single-factor and insufficient for anything important.

### Password Storage: Why This Matters at the OS Level

The OS stores credentials in `/etc/shadow` (Linux) or SAM database (Windows). How they're stored determines how catastrophic a breach is:

```
NEVER do this:
  username:mysecretpassword          <-- plaintext, game over if leaked

Better:
  username:sha256(password)          <-- hashed, but vulnerable to rainbow tables

Correct:
  username:bcrypt(salt + password)   <-- salted + slow hash, resistant to brute force
```

**Why salting matters:**

```
Without salt:
  User A: password="hello" -> hash=2CF24DBA...
  User B: password="hello" -> hash=2CF24DBA...   <-- same hash! Attacker cracks one, gets both

With salt (random per user):
  User A: salt="x9k2" password="hello" -> hash=F7B3A1C8...
  User B: salt="m4p7" password="hello" -> hash=91D2E5F0...   <-- different hashes
```

**Why slow hashes matter:**
- SHA-256: billions of hashes per second on a GPU -- brute force is fast
- bcrypt/argon2: deliberately slow (tunable cost factor) -- brute force takes years
- `/etc/shadow` on modern Linux uses yescrypt or SHA-512 with rounds

```bash
# /etc/shadow entry (Linux)
$ sudo cat /etc/shadow | grep alice
alice:$6$rounds=5000$salt$hashvalue...:19500:0:99999:7:::
       ^  ^              ^
       |  |              +-- the actual hash
       |  +-- algorithm ($6 = SHA-512, $2b = bcrypt)
       +-- hash parameters
```

### Why /etc/shadow Exists

Originally, password hashes lived in `/etc/passwd` (world-readable). Anyone could grab the hashes and crack them offline. The fix: move hashes to `/etc/shadow` (readable only by root). The `/etc/passwd` file still exists but contains an `x` placeholder where the hash used to be.

## Authorization: What Can You Do?

Once the OS knows who you are, it needs to decide what you're allowed to access. This is the permission model.

### Unix Permission Model

Every file has three sets of permissions for three classes of users:

```
-rwxr-x--x  1  alice  devteam  4096  Apr 1 10:00  script.sh
 |||||||||||
 |+--+--+--+
 |  |  |  |
 |  |  |  +-- Others (everyone else): --x (execute only)
 |  |  +----- Group (devteam):        r-x (read + execute)
 |  +-------- Owner (alice):          rwx (read + write + execute)
 +----------- File type: - = regular file, d = directory, l = symlink
```

| Permission | On Files | On Directories |
|-----------|----------|----------------|
| **r** (read) | Read file contents | List directory contents (ls) |
| **w** (write) | Modify file contents | Create/delete files in directory |
| **x** (execute) | Run as program | Enter directory (cd), access files within |

### Numeric (Octal) Representation

Each permission set is a 3-bit number:

```
r=4  w=2  x=1

rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
--x = 0+0+1 = 1

So -rwxr-x--x = 751
```

```bash
# Common permission patterns
chmod 755 script.sh    # rwxr-xr-x  Owner: full, others: read+execute
chmod 644 config.txt   # rw-r--r--  Owner: read+write, others: read only
chmod 600 secret.key   # rw-------  Owner only, no one else
chmod 777 public/      # rwxrwxrwx  Everyone can do everything (DANGEROUS)
```

### umask: Default Permission Mask

When you create a file, the OS applies a **mask** that removes permissions:

```
Default file permissions:   666  (rw-rw-rw-)
umask:                      022  (----w--w-)
Result:                     644  (rw-r--r--)

Default directory permissions: 777  (rwxrwxrwx)
umask:                         022  (----w--w-)
Result:                        755  (rwxr-xr-x)
```

```bash
$ umask          # Show current mask
0022
$ umask 0077     # Set restrictive mask: only owner gets permissions
```

### SUID and SGID: Dangerous but Necessary

```
Special permission bits:

-rwsr-xr-x    SUID (Set User ID)
   ^          Process runs with FILE OWNER's UID, not caller's
   
-rwxr-sr-x    SGID (Set Group ID)
      ^       Process runs with FILE GROUP's GID, not caller's

drwxrwsr-x    SGID on directory
      ^       New files inherit directory's group, not creator's group
```

**Why SUID exists:** Regular users need to do privileged things sometimes. `passwd` needs to write `/etc/shadow`. `ping` needs raw network sockets. SUID lets specific programs run with elevated privileges without giving users root access.

**The risk:** Every SUID-root binary is a potential privilege escalation vector. If it has a buffer overflow, the attacker gets root.

```bash
# Find all SUID programs on a system
$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/mount
...
# Each one is an attack surface. Minimize this list.
```

### Linux Capabilities: Breaking Root Into Pieces

Traditional Unix has a binary model: you're either root (can do everything) or you're not. Linux capabilities break root's power into ~40 fine-grained permissions:

| Capability | What It Grants | Use Case |
|-----------|---------------|----------|
| CAP_NET_BIND_SERVICE | Bind ports below 1024 | Web servers on port 80/443 |
| CAP_NET_RAW | Raw socket access | ping, network tools |
| CAP_SYS_ADMIN | Catch-all for admin operations | Mount, sethostname, etc. (avoid this) |
| CAP_SYS_PTRACE | Trace/debug other processes | Debuggers, strace |
| CAP_DAC_OVERRIDE | Bypass file permission checks | (dangerous -- like partial root) |
| CAP_SETUID | Change UID | Process managers |
| CAP_CHOWN | Change file ownership | Deployment tools |
| CAP_NET_ADMIN | Network configuration | iptables, routing |

```bash
# View capabilities of a running process
$ getpcaps 1234
1234: cap_net_bind_service=ep

# Set capabilities on a binary (instead of SUID)
$ sudo setcap cap_net_bind_service=ep /usr/bin/myapp

# View capabilities of a binary
$ getcap /usr/bin/myapp
/usr/bin/myapp cap_net_bind_service=ep
```

This is the principle of least privilege in action: instead of SUID root, grant only the specific capability needed.

## Identity in the OS: UID, GID, and the Subtleties

Every process carries multiple identity markers:

```
Process Identity
+-----------------------------------+
| Real UID (RUID)      = 1000      |  Who actually started this process
| Effective UID (EUID)  = 0        |  What permissions apply RIGHT NOW
| Saved UID (SUID)      = 0        |  Saved EUID for later restoration
| Real GID              = 1000     |  Actual group
| Effective GID         = 1000     |  Current group permissions
| Supplementary groups  = 27,100   |  Additional group memberships
+-----------------------------------+
```

**Why three UIDs?**

When you run a SUID-root program:
1. **Real UID** = your UID (1000) -- tracks who you really are
2. **Effective UID** = root (0) -- determines current permissions
3. **Saved UID** = root (0) -- lets the process drop and re-acquire root

```
                  SUID binary runs
User (UID 1000) -----------------> Process
                                    RUID = 1000 (real user)
                                    EUID = 0    (running as root)
                                    SUID = 0    (saved for later)
                                      |
                              seteuid(1000)  -- drop privileges
                                      |
                                    RUID = 1000
                                    EUID = 1000 (now unprivileged)
                                    SUID = 0    (can re-escalate)
                                      |
                              seteuid(0)     -- re-acquire root
                                      |
                                    RUID = 1000
                                    EUID = 0    (root again)
                                    SUID = 0
```

Well-written SUID programs drop privileges immediately after doing the privileged operation, then re-acquire only when needed. This limits the window of vulnerability.

## Real-World Connection

**Container security -- running as non-root**: The biggest container security mistake is running processes as root inside the container. Even though namespaces provide isolation, a kernel vulnerability could let root-in-container become root-on-host. Best practice: run as a non-root user, drop capabilities, use read-only filesystems.

```dockerfile
# Bad: runs as root
FROM nginx
CMD ["nginx"]

# Good: runs as non-root
FROM nginx
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
CMD ["nginx"]
```

**Cloud IAM rooted in OS auth**: AWS IAM, GCP IAM, Azure RBAC all follow the same model as Unix permissions. A role (like a UID) has policies (like permission bits) that grant access to resources (like files). The concepts are identical -- just the scale changes from files on a disk to services across a cloud.

**SSH key authentication**: SSH keys are "something you have" authentication. Your private key (`~/.ssh/id_ed25519`, mode 600) proves your identity without sending a password over the network. The server checks your public key against `~/.ssh/authorized_keys`. This is why key-based SSH is standard for production servers.

**sudo vs su vs SUID**: `sudo` is the modern way to do privileged operations -- it logs who did what, can grant fine-grained access per command, and uses the user's own password. `su` switches to another user entirely. Both rely on the underlying UID/EUID mechanism.

## Interview Angle

**Q: Why should passwords be stored with bcrypt/argon2 instead of SHA-256?**

A: SHA-256 is fast by design -- a GPU can compute billions of hashes per second, making brute-force attacks feasible. bcrypt and argon2 are deliberately slow (adjustable cost factor) and memory-hard (argon2), making brute force impractically expensive. Combined with per-user salts (which prevent rainbow table attacks and ensure identical passwords produce different hashes), they provide the standard defense for credential storage.

**Q: Explain the Unix permission model. What does chmod 750 mean?**

A: Unix permissions assign three access types (read=4, write=2, execute=1) to three user classes (owner, group, others). chmod 750 means: owner gets 7 (rwx = full access), group gets 5 (r-x = read and execute), others get 0 (no access). For a directory, execute means the ability to cd into it and access files within.

**Q: What's the difference between real UID and effective UID?**

A: The real UID identifies who actually started the process (the actual user). The effective UID determines what permissions the process has right now -- it's what the kernel checks for access control decisions. They differ when running SUID programs: if alice runs a SUID-root binary, RUID=alice (who started it) but EUID=root (what permissions apply). Well-written SUID programs use seteuid() to drop to the real UID when elevated privileges aren't needed.

**Q: Why are Linux capabilities better than SUID root?**

A: SUID root gives a program ALL of root's permissions -- if it's compromised, the attacker gets full root. Linux capabilities break root into ~40 individual permissions. A web server that only needs to bind port 80 gets just CAP_NET_BIND_SERVICE. If compromised, the attacker can bind low ports but can't load kernel modules, read shadow files, or do anything else root-like. It's the principle of least privilege applied to root itself.

---

**Next:** [ACLs and Capabilities](03-access-control-lists-and-capabilities.md) -- two ways to implement the access matrix in practice.
