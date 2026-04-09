# Windows Security Model: Tokens and ACLs

## The Security Reference Monitor

At the core of Windows security is the **Security Reference Monitor (SRM)**, a kernel component that performs access checks every time a process opens a handle to any object. Every `CreateFile()`, `OpenProcess()`, `RegOpenKeyEx()`, and similar call triggers an SRM access check.

The SRM answers one question: "Does this subject (represented by a token) have the requested access to this object (protected by a security descriptor)?" This is a mandatory check -- there is no way around it short of running in kernel mode.

Linux has a similar checkpoint in `inode_permission()` and the LSM (Linux Security Modules) hooks, but the Windows model is more centralized. In Linux, basic permission checks are done by the VFS layer (mode bits), extended checks by POSIX ACLs, and mandatory checks by SELinux/AppArmor. In Windows, all of this is unified in the SRM.

## Access Tokens: Your Security Badge

Every process and thread in Windows carries an **access token** -- a kernel object that describes the security identity and capabilities of that process. An access token is like an all-access badge at a conference. It shows who you are (SID), which groups you belong to (group SIDs), what special permissions you have (privileges), and your trust level (integrity level).

```
Access Token for process "notepad.exe"
+--------------------------------------------------+
| User SID: S-1-5-21-...-1001 (Alice)              |
+--------------------------------------------------+
| Group SIDs:                                       |
|   S-1-5-32-544  (Administrators)   [Deny-only]   |
|   S-1-5-32-545  (Users)            [Enabled]     |
|   S-1-5-21-...- (Dev Team)         [Enabled]     |
|   S-1-5-11      (Authenticated Users) [Enabled]  |
+--------------------------------------------------+
| Privileges:                                       |
|   SeShutdownPrivilege              [Disabled]     |
|   SeChangeNotifyPrivilege          [Enabled]      |
|   SeUndockPrivilege                [Disabled]     |
+--------------------------------------------------+
| Integrity Level: Medium (S-1-16-8192)             |
+--------------------------------------------------+
| Session ID: 1                                     |
+--------------------------------------------------+
| Default DACL: (for new objects created by this    |
|   process)                                        |
+--------------------------------------------------+
```

### Token Contents

| Field | Purpose | Linux Equivalent |
|---|---|---|
| User SID | Who this process runs as | UID (uid_t) |
| Group SIDs | Groups the user belongs to | Supplementary group list (getgroups) |
| Privileges | Special capabilities (e.g., debug, backup) | Linux capabilities (CAP_SYS_PTRACE, etc.) |
| Integrity level | Trust level (Low, Medium, High, System) | No direct equivalent (SELinux types are closest) |
| Session ID | Logon session | Session leader (setsid) |
| Default DACL | Permissions for new objects | umask |

When a child process is created, it inherits the parent's token. When a service starts, it gets a token based on the configured service account (LocalSystem, LocalService, NetworkService, or a specific user).

## SIDs: Security Identifiers

A **SID (Security Identifier)** is a unique, immutable identifier for a user, group, computer, or domain. SIDs look like:

```
S-1-5-21-3623811015-3361044348-30300820-1013
| | |  |                                 |
| | |  +-- Identifier authority          +-- RID (Relative ID)
| | +-- Sub-authority count                   1013 = specific user
| +-- Revision
+-- "S" prefix
```

**Well-known SIDs** you should recognize:

| SID | Identity | Significance |
|---|---|---|
| S-1-5-18 | Local System | Highest privilege, used by kernel services |
| S-1-5-19 | Local Service | Reduced privileges, no network credentials |
| S-1-5-20 | Network Service | Like Local Service but with network credentials |
| S-1-5-32-544 | Administrators | Built-in Administrators group |
| S-1-5-32-545 | Users | Built-in Users group |
| S-1-1-0 | Everyone | All users, including anonymous |
| S-1-5-11 | Authenticated Users | All logged-in users |
| S-1-16-4096 | Low Integrity | Browser sandbox level |
| S-1-16-8192 | Medium Integrity | Standard user level |
| S-1-16-12288 | High Integrity | Elevated administrator level |
| S-1-16-16384 | System Integrity | Kernel and core services |

SIDs solve a problem that Unix UIDs have: they are globally unique (domain + RID), so they work across networked environments. Unix UIDs are local to a machine -- UID 1001 on server A is not the same user as UID 1001 on server B unless you use LDAP/NIS. SIDs inherently include the domain, making them unique across an entire Active Directory forest.

## Security Descriptors: Protecting Objects

Every securable object (file, registry key, process, mutex, etc.) has a **security descriptor** containing:

```
Security Descriptor
+--------------------------------------------+
| Owner SID: S-1-5-21-...-1001 (Alice)       |
+--------------------------------------------+
| Group SID: S-1-5-21-...-513 (Domain Users) |
+--------------------------------------------+
| DACL (Discretionary ACL):                  |
|   Who can access this object and how        |
|   +------+----------+-------------------+  |
|   | Type | SID      | Permissions       |  |
|   +------+----------+-------------------+  |
|   | Allow| Alice    | Full Control      |  |
|   | Allow| Dev Team | Read, Execute     |  |
|   | Deny | Bob      | Write             |  |
|   | Allow| Users    | Read              |  |
|   +------+----------+-------------------+  |
+--------------------------------------------+
| SACL (System ACL):                         |
|   Auditing rules (log who accesses what)    |
|   +------+----------+-------------------+  |
|   | Audit| Everyone | Write (failures)  |  |
|   +------+----------+-------------------+  |
+--------------------------------------------+
```

The **Owner** can always modify the security descriptor (take ownership back, change permissions). The **Group** is mostly for POSIX compatibility. The **DACL** controls access. The **SACL** controls auditing.

## DACLs and ACEs: The Access Decision

A **DACL (Discretionary Access Control List)** is an ordered list of **ACEs (Access Control Entries)**. When the SRM evaluates access, it walks the DACL from top to bottom:

1. For each ACE, check if it applies to the requesting user (by SID or group membership).
2. If an **Allow** ACE matches and grants the requested permission, accumulate that permission.
3. If a **Deny** ACE matches, immediately deny that permission.
4. Continue until all requested permissions are granted or a deny is encountered.
5. If the end of the DACL is reached and not all requested permissions are accumulated, access is denied.

**Important**: ACE order matters. Deny ACEs are conventionally placed before Allow ACEs, which means denies take effect first. This is why "Deny always wins" in practice.

```
Access Check Example:

User: Bob (member of "Dev Team" and "Users")
Requests: Read + Write access to report.docx

DACL evaluation:
1. Deny | Bob | Write       --> Bob wants Write, DENY matched. Write is denied.
2. Allow | Dev Team | Read  --> Bob is in Dev Team, Read accumulated.
3. Allow | Users | Read     --> Bob is in Users, Read already accumulated.

Result: Read = GRANTED, Write = DENIED
Overall: ACCESS DENIED (because not all requested permissions were granted)
```

### Permission Inheritance

NTFS permissions support sophisticated inheritance. When you set permissions on a folder, they can flow down to all children:

| Inheritance Flag | Meaning |
|---|---|
| This folder only | Does not propagate |
| This folder, subfolders, and files | Full inheritance |
| Subfolders and files only | Applies to children but not the folder itself |
| This folder and subfolders | No files |
| This folder and files | No subfolders |

This granularity far exceeds Linux's model, where new files get permissions based on `umask` and the parent directory's setgid bit. POSIX ACLs on Linux provide default ACLs for inheritance, but the inheritance model is simpler than Windows'.

## Mandatory Integrity Control (MIC)

**Mandatory Integrity Control** adds a second layer of access checking on top of DACLs. Even if the DACL would allow access, MIC can block it based on integrity levels.

Integrity levels are like security clearance levels. A "Low" clearance process (browser) can read public data but cannot write to "Medium" or "High" areas -- even if the DACL would normally allow it.

```
Integrity Level Hierarchy:

System (S-1-16-16384)     Kernel services, trusted installers
    |
High (S-1-16-12288)       Elevated administrator processes
    |
Medium (S-1-16-8192)      Standard user processes (default)
    |
Low (S-1-16-4096)         Sandboxed processes (browsers, readers)
    |
Untrusted (S-1-16-0)      Most restricted
```

The key rule: **No Write Up**. A process cannot write to objects at a higher integrity level. A Low integrity browser process cannot modify Medium integrity user files, even if the user owns both.

MIC is what makes browser sandboxes work on Windows:
- Chrome/Edge renderer processes run at **Low** integrity
- They can read files (with DACL permission) but cannot write to user directories (Medium integrity)
- If the renderer is compromised, the attacker is confined to Low integrity -- they cannot modify user files, install software, or change system settings

Linux has no built-in equivalent. SELinux and AppArmor provide mandatory access control, but they use type-based (SELinux) or path-based (AppArmor) policies rather than integrity levels. The concept is similar though: a second, mandatory layer of access control that overrides discretionary (DACL/mode bit) permissions.

## UAC: User Account Control

**UAC (User Account Control)** is the mechanism that separates administrator privileges from everyday use. When an administrator logs in, Windows creates **two tokens**:

```
Administrator Login
        |
        v
+------------------+          +------------------+
| Filtered Token   |          | Full Token       |
| (Medium integrity)|         | (High integrity) |
|                  |          |                  |
| Admin group:     |          | Admin group:     |
|   DENY-ONLY      |          |   ENABLED        |
| Privileges:      |          | Privileges:      |
|   Most disabled   |         |   All available  |
+------------------+          +------------------+
        |                              |
        v                              v
  Normal processes              Elevated processes
  (Explorer, browsers)          (installers, admin tools)
```

By default, all processes get the **filtered token** -- the Administrators group SID is present but marked as deny-only (cannot be used for Allow ACEs, only Deny). When an application requests elevation, UAC shows the consent dialog, and if approved, a **new process** is created with the full, elevated token.

This is why "Run as administrator" creates a separate process -- you cannot elevate an existing process. The elevated process has a different token with High integrity and full administrator privileges.

In Linux, the equivalent is `sudo` -- running a command with elevated privileges. The difference is that UAC operates at the token level (two pre-built tokens, one filtered) while sudo operates at the process level (starts a new shell with root UID). Linux has no equivalent of the "deny-only" group marking or the split token concept.

## Privileges: Fine-Grained Capabilities

Privileges are special permissions that are not tied to a specific object. They grant system-wide capabilities:

| Privilege | Allows | Linux Equivalent |
|---|---|---|
| SeDebugPrivilege | Open any process for debugging | CAP_SYS_PTRACE |
| SeBackupPrivilege | Read any file (bypass DACLs) | CAP_DAC_READ_SEARCH |
| SeRestorePrivilege | Write any file (bypass DACLs) | CAP_DAC_OVERRIDE |
| SeTakeOwnershipPrivilege | Take ownership of any object | CAP_FOWNER |
| SeShutdownPrivilege | Shut down the system | CAP_SYS_BOOT |
| SeLockMemoryPrivilege | Lock pages in memory (large pages) | CAP_IPC_LOCK |
| SeImpersonatePrivilege | Impersonate a client | No direct equivalent |
| SeLoadDriverPrivilege | Load/unload kernel drivers | CAP_SYS_MODULE |

A critical detail: **privileges exist in the token but are disabled by default**. The process must explicitly call `AdjustTokenPrivileges()` to enable a privilege before using it. This is a defense-in-depth measure -- even if a process has a privilege in its token, it must consciously activate it.

This is similar to how Linux capabilities work: `CAP_SYS_PTRACE` in the effective set must be raised before it takes effect. The Windows model is slightly more explicit because each privilege has an enabled/disabled flag.

## Impersonation: Acting as Another User

Windows supports **impersonation**, where a thread temporarily assumes the security context (token) of another user. This is critical for server applications:

1. Client connects to a server (e.g., file server)
2. Server authenticates the client
3. Server thread **impersonates** the client's token
4. Server accesses files on behalf of the client -- SRM checks against the client's token, not the server's
5. Server **reverts** to its own token

This ensures that the server cannot accidentally (or maliciously) access files that the client should not be able to see. Each thread can have its own impersonation token, separate from the process token.

Linux has no direct equivalent at the thread level. Linux servers typically use `seteuid()` or `setfsuid()` to change the process's effective UID, but this affects the entire process (or uses `setresuid` in a forked child). The closest equivalent is Linux's keyring-based authentication or passing credentials over Unix domain sockets.

## Comparison: Linux Security vs Windows Security

| Feature | Linux | Windows |
|---|---|---|
| User identity | UID (integer, local) | SID (globally unique, includes domain) |
| Group identity | GID (integer, local) | Group SID (globally unique) |
| Basic permissions | Mode bits (rwxrwxrwx) | DACL with ACEs |
| Extended permissions | POSIX ACLs (optional) | Always full ACLs |
| Permission inheritance | umask + setgid + default ACLs | Detailed inheritance flags per ACE |
| Capabilities | Linux capabilities (CAP_*) | Privileges (Se*Privilege) |
| Mandatory access control | SELinux (types) / AppArmor (paths) | Mandatory Integrity Control (levels) |
| Privilege escalation | sudo / su | UAC (split token + elevation prompt) |
| Per-thread identity | Rarely used (setresuid) | Impersonation tokens (common) |
| Auditing | auditd + audit rules | SACL on every object |
| Network identity | Kerberos/LDAP (manual setup) | Active Directory + Kerberos (built-in) |

## Real-World Connection

When a penetration tester compromises a Windows machine, the first thing they examine is the stolen process's token: What SIDs does it contain? What privileges are available (even if disabled)? If `SeDebugPrivilege` is present, they can attach to any process and dump credentials from LSASS. If `SeImpersonatePrivilege` is present, they can use "potato" attacks to escalate from a service account to SYSTEM. Understanding tokens is the foundation of Windows privilege escalation.

When an enterprise administrator designs a file server, they plan DACL inheritance carefully. A common mistake is setting complex explicit permissions on thousands of files instead of letting inheritance flow from a few well-designed parent folders. The result is slow access checks (the SRM must evaluate a large DACL) and impossible-to-audit security. Good design uses broad inherited permissions with minimal explicit overrides.

When a developer builds a Windows service, they choose the service account carefully: LocalSystem has unlimited access (dangerous), NetworkService has minimal local access but can authenticate on the network, and a managed service account provides a balance. The principle of least privilege applies -- give the service the minimum token it needs.

## Interview Angle

**Q: How does the Windows security model compare to Linux?**

A: Windows uses SIDs (globally unique identifiers) instead of UIDs (local integers), which inherently works across networked domains. Access control is always through DACLs (ordered lists of Allow/Deny ACEs) rather than Linux's simpler mode bits. Windows adds Mandatory Integrity Control as a second layer -- even if the DACL allows access, a Low-integrity process cannot write to Medium-integrity objects. Privileges in Windows (SeDebugPrivilege, SeBackupPrivilege) map to Linux capabilities (CAP_SYS_PTRACE, CAP_DAC_READ_SEARCH) but exist in the access token and must be explicitly enabled. UAC splits administrator tokens into filtered and full versions, while Linux uses sudo for elevation. Windows supports per-thread impersonation tokens (critical for servers), which Linux lacks at the thread level.

**Q: What is Mandatory Integrity Control and why does it matter?**

A: MIC assigns integrity levels (Untrusted, Low, Medium, High, System) to both processes and objects. The key rule is "no write up" -- a process cannot write to objects at a higher integrity level, regardless of DACL permissions. This is how browser sandboxes work: renderer processes run at Low integrity and cannot modify user files (Medium) even if compromised. MIC acts as a safety net when DACLs are misconfigured or when malware runs in a sandboxed process. Linux's equivalent is SELinux or AppArmor mandatory access control, but they use policy-based rules rather than hierarchical integrity levels.

---

Next: [Boot Process: UEFI to Desktop](07-windows-boot-process-uefi-bootmgr.md)
