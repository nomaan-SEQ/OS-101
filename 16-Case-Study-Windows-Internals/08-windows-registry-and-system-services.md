# Windows Registry and System Services

## The Registry: Windows' Centralized Configuration Database

The registry is like a massive filing cabinet that stores ALL system and application configuration. Where Linux scatters config across `/etc`, `/proc`, environment variables, and dotfiles, Windows centralizes everything in one place. Every driver setting, service configuration, user preference, file association, hardware profile, and application setting lives in the registry.

This is a fundamental design difference. Linux follows the Unix philosophy of small, readable text files. Windows chose a structured, binary database. Both approaches have real tradeoffs:

| Approach | Advantages | Disadvantages |
|---|---|---|
| Linux (scattered text files) | Human-readable, easy to diff/version, simple backup | No standard format, apps parse differently, no atomic multi-key updates |
| Windows (centralized registry) | Atomic transactions, structured queries, single API, consistent format | Binary format, hard to diff, corruption affects everything, bloat over time |

## Registry Structure

The registry is organized as a hierarchy of **hives**, **keys**, and **values**:

```
Registry Root
|
+-- HKEY_LOCAL_MACHINE (HKLM)     Machine-wide settings
|   +-- SYSTEM                     Boot config, drivers, services
|   +-- SOFTWARE                   Installed application settings
|   +-- SAM                        Security Account Manager (user accounts)
|   +-- SECURITY                   Security policies
|   +-- HARDWARE                   Hardware config (volatile, rebuilt each boot)
|
+-- HKEY_USERS (HKU)              All user profiles
|   +-- .DEFAULT                   Default profile template
|   +-- S-1-5-21-...-1001         User profile (by SID)
|
+-- HKEY_CURRENT_USER (HKCU)      Current user (link to HKU\{current SID})
|
+-- HKEY_CLASSES_ROOT (HKCR)      File associations, COM classes
|                                  (merged view of HKLM\SOFTWARE\Classes
|                                   and HKCU\SOFTWARE\Classes)
|
+-- HKEY_CURRENT_CONFIG (HKCC)    Current hardware profile
                                   (link to HKLM\SYSTEM\CurrentControlSet
                                    \Hardware Profiles\Current)
```

### Keys and Values

**Keys** are containers (like folders). **Values** are named data items within keys.

Each value has three parts: a **name**, a **type**, and **data**:

| Type | Description | Example |
|---|---|---|
| REG_SZ | String | `"C:\Program Files\App\app.exe"` |
| REG_EXPAND_SZ | String with environment variable references | `"%SystemRoot%\system32\cmd.exe"` |
| REG_MULTI_SZ | Array of strings | `"TCP\0UDP\0"` |
| REG_DWORD | 32-bit integer | `0x00000001` |
| REG_QWORD | 64-bit integer | `0x0000000100000000` |
| REG_BINARY | Raw binary data | `48 65 6C 6C 6F` |

## Hive Files on Disk

Registry hives are stored as binary files:

| Hive | File Location |
|---|---|
| SYSTEM | `C:\Windows\System32\config\SYSTEM` |
| SOFTWARE | `C:\Windows\System32\config\SOFTWARE` |
| SAM | `C:\Windows\System32\config\SAM` |
| SECURITY | `C:\Windows\System32\config\SECURITY` |
| DEFAULT | `C:\Windows\System32\config\DEFAULT` |
| NTUSER.DAT | `C:\Users\{username}\NTUSER.DAT` (per user) |
| UsrClass.dat | `C:\Users\{username}\AppData\Local\Microsoft\Windows\UsrClass.dat` |

The SYSTEM hive is loaded by **winload.efi** during boot (before the kernel starts) because it contains boot-start driver configuration. User hives (NTUSER.DAT) are loaded when the user logs in and unloaded when they log out.

Each hive file has a transaction log (`.LOG1`, `.LOG2` files) for crash recovery. If the system crashes while writing to a hive, the transaction log is replayed on the next boot to restore consistency. This is the registry's equivalent of filesystem journaling.

## Registry Mapped to Linux Equivalents

If you know Linux, these mappings will help you navigate Windows configuration:

| Windows Registry Path | Linux Equivalent | Contains |
|---|---|---|
| `HKLM\SYSTEM\CurrentControlSet\Services` | `/etc/systemd/system/` | Service definitions and driver configs |
| `HKLM\SYSTEM\CurrentControlSet\Control` | `/etc/sysctl.conf`, `/proc/sys/` | System behavior settings |
| `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` | `/etc/xdg/autostart/`, systemd user units | Auto-start programs |
| `HKLM\SOFTWARE\{Vendor}\{App}` | `/etc/{app}/` config files | Application settings (machine-wide) |
| `HKCU\SOFTWARE\{Vendor}\{App}` | `~/.config/{app}/` or `~/.{app}rc` | Application settings (per-user) |
| `HKU\.DEFAULT` | `/etc/skel/` | Default user profile template |
| `HKLM\SYSTEM\CurrentControlSet\Enum` | `/sys/devices/` + udev | Enumerated hardware devices |
| `HKCR` | `/etc/mime.types` + `~/.local/share/applications/` | File type associations |

## Service Control Manager (SCM / services.exe)

The **Service Control Manager** is Windows' equivalent of systemd -- the master process that manages background services (daemons). It starts at boot (launched by wininit.exe) and controls the lifecycle of every Windows service.

### Service Configuration

Each service is defined by a registry key under `HKLM\SYSTEM\CurrentControlSet\Services\{ServiceName}`:

```
HKLM\SYSTEM\CurrentControlSet\Services\Spooler
    Type          = REG_DWORD 0x00000110  (Win32 Own Process, Interactive)
    Start         = REG_DWORD 0x00000002  (Auto Start)
    ErrorControl  = REG_DWORD 0x00000001  (Normal)
    ImagePath     = REG_EXPAND_SZ "%SystemRoot%\System32\spoolsv.exe"
    DisplayName   = REG_SZ "Print Spooler"
    DependOnService = REG_MULTI_SZ "RPCSS\0http"
    ObjectName    = REG_SZ "LocalSystem"
```

### Service Start Types

| Start Value | Name | Meaning | Linux Equivalent |
|---|---|---|---|
| 0 | Boot | Loaded by winload.efi (storage drivers) | Built into initramfs |
| 1 | System | Loaded during kernel init | Early systemd units |
| 2 | Automatic | Started by SCM at boot | `systemctl enable` |
| 3 | Manual | Started on demand | Socket-activated or manual start |
| 4 | Disabled | Not started | `systemctl disable` |

### Service States

```
                +---> Start Pending ---+
                |                      |
Stopped --------+                      +-----> Running
                                       |         |
                +---> Stop Pending <---+         |
                |                                |
                +---- Paused <---  Pause Pending-+
```

### Service Types

| Type | Description | Example |
|---|---|---|
| Win32OwnProcess | Runs in its own process | spoolsv.exe (Print Spooler) |
| Win32ShareProcess | Shares a svchost.exe process | Many small services |
| KernelDriver | Kernel-mode driver | ntfs.sys, tcpip.sys |
| FileSystemDriver | Filesystem driver | ntfs.sys |

### Management Tools

| Tool | Purpose | Linux Equivalent |
|---|---|---|
| `sc.exe` | Command-line service management | `systemctl` |
| `services.msc` | GUI service management | No equivalent (maybe cockpit) |
| `sc query` | List services and status | `systemctl list-units` |
| `sc start/stop` | Start/stop a service | `systemctl start/stop` |
| `sc config` | Change service settings | `systemctl edit` |
| `Get-Service` (PowerShell) | Query services | `systemctl status` |

## svchost.exe: The Shared Service Host

**svchost.exe** is like a shared office building. Multiple small services share one process (building) to save resources. Modern Windows gives important services their own building (process) for isolation.

```
Traditional (shared):                Modern (isolated):
+-------------------------+          +----------+  +----------+
| svchost.exe (PID 800)  |          | svchost  |  | svchost  |
|   - DHCP Client         |          | (PID 800)|  | (PID 812)|
|   - DNS Client          |          | DHCP     |  | DNS      |
|   - NLA (Network)       |          +----------+  +----------+
+-------------------------+          +----------+
                                     | svchost  |
                                     | (PID 824)|
                                     | NLA      |
                                     +----------+
```

Each svchost.exe instance is started with a command line like:
```
svchost.exe -k netsvcs -p
```

The `-k` parameter specifies a **service group** (defined in the registry), and `-p` enables the per-service SID policy. The registry key `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Svchost` maps group names to the DLLs that should load.

Starting with Windows 10 (on systems with sufficient RAM), Microsoft began splitting services into individual svchost processes. This improves:
- **Reliability**: One crashing service does not take down others in the same process
- **Security**: Each service can have its own restricted token
- **Diagnostics**: Resource usage is visible per-service in Task Manager

In Linux, each daemon typically runs as its own process. The svchost model has no direct Linux equivalent -- it was originally an optimization for Windows' heavier per-process overhead.

## Task Scheduler

The **Task Scheduler** is Windows' equivalent of cron. It runs scheduled tasks at specified times, on system events, or on triggers.

```
Task Scheduler Triggers:
+---------------------------+
| Time-based                |  "Every day at 3am"
| Logon                     |  "When any user logs on"
| Startup                   |  "When the system boots"
| Event-based               |  "When Event ID 1000 occurs"
| Idle                      |  "When the system is idle"
| Session change            |  "When a user locks the screen"
| Registration              |  "Immediately when task is created"
+---------------------------+
```

Task Scheduler is more capable than cron:
- Conditions (only run if on AC power, only if idle, only if network available)
- Multiple triggers per task
- Retry on failure
- Task history and logging
- Security context (run as a specific user, with highest privileges)

| Feature | Windows Task Scheduler | Linux cron |
|---|---|---|
| Time scheduling | Yes (flexible) | Yes (cron expression) |
| Event triggers | Yes (Windows events) | No (use systemd timers + path units) |
| Conditions | Yes (idle, AC power, network) | No (use wrapper scripts) |
| Task history | Built-in logging | Manual logging |
| Management | GUI (taskschd.msc) + schtasks.exe + PowerShell | crontab -e (text) |
| Per-user tasks | Yes | Yes (per-user crontab) |

Linux's **systemd timers** are the modern equivalent and match most Task Scheduler features, including event-based triggers and conditions.

## WMI: Windows Management Instrumentation

**WMI (Windows Management Instrumentation)** provides a standardized way to query and manage system information. It is like a combination of `/proc`, `/sys`, and systemd's D-Bus in Linux -- a unified interface for system introspection.

```
WMI Architecture:
+------------------------------------------------------+
| PowerShell / WMIC / WMI Explorer (consumers)         |
+----------------------------+-------------------------+
                             |
                             v
+----------------------------+-------------------------+
|           WMI Service (winmgmt)                       |
|   CIM Repository (schema + persistent data)           |
+----------------------------+-------------------------+
                             |
                             v
+------------+  +------------+  +-----------------------+
| Win32      |  | Operating   |  | Hardware              |
| Provider   |  | System      |  | Provider              |
| (processes,|  | Provider    |  | (BIOS, CPU,           |
|  services) |  | (registry,  |  |  disk, network)       |
|            |  |  event log) |  |                       |
+------------+  +------------+  +-----------------------+
```

Common WMI queries (PowerShell):
```powershell
Get-WmiObject Win32_OperatingSystem   # OS info
Get-WmiObject Win32_Process           # Running processes
Get-WmiObject Win32_Service           # Installed services
Get-WmiObject Win32_LogicalDisk       # Disk information
Get-CimInstance Win32_Processor       # CPU info (modern CIM cmdlet)
```

| WMI Class | Information | Linux Equivalent |
|---|---|---|
| Win32_Process | Running processes | `/proc/[pid]/` |
| Win32_Service | Windows services | `systemctl list-units` |
| Win32_OperatingSystem | OS version, memory | `uname -a`, `/proc/meminfo` |
| Win32_LogicalDisk | Disk space | `df` |
| Win32_NetworkAdapter | Network interfaces | `/sys/class/net/`, `ip link` |
| Win32_BIOS | BIOS/UEFI info | `dmidecode` |

WMI is also used for **remote management** -- you can query WMI on a remote machine, which makes it the backbone of enterprise monitoring tools like SCCM and SCOM. Linux achieves remote management through SSH + standard tools, or through agent-based monitoring systems.

## Event Log: Centralized Logging

The **Windows Event Log** is a centralized, structured logging system -- like journald/syslog in Linux, but with a more formal event schema.

### Log Categories

| Log | Contains | Linux Equivalent |
|---|---|---|
| System | Kernel, drivers, hardware, core services | `/var/log/kern.log`, `journalctl -k` |
| Application | Application events, crashes | `/var/log/syslog`, `journalctl` |
| Security | Logon/logoff, access checks, audit events | `/var/log/auth.log`, `aureport` |
| Setup | OS installation and updates | `/var/log/dpkg.log` or `/var/log/yum.log` |
| Forwarded Events | Events forwarded from remote machines | Remote syslog |

Each event has:
- **Event ID**: Numeric identifier (e.g., 4624 = successful logon, 7036 = service state change)
- **Source**: Which component generated it
- **Level**: Information, Warning, Error, Critical
- **Timestamp**: When it occurred
- **Description**: Human-readable message with details

### Management Tools

| Tool | Purpose | Linux Equivalent |
|---|---|---|
| Event Viewer (eventvwr.msc) | GUI log browser | `journalctl` with filters |
| `wevtutil` | Command-line log management | `journalctl` |
| `Get-WinEvent` (PowerShell) | Query events with filtering | `journalctl` with match patterns |
| Event Forwarding | Centralize logs from multiple machines | Remote syslog / Loki / ELK |

Common PowerShell queries:
```powershell
Get-WinEvent -LogName Security -MaxEvents 10   # Last 10 security events
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2}  # System errors
```

## ETW: Event Tracing for Windows

**ETW (Event Tracing for Windows)** is a high-performance tracing framework separate from the Event Log. ETW is designed for real-time diagnostics with minimal overhead -- similar to Linux's ftrace, perf, and eBPF.

```
ETW Architecture:
+-------------------+     +-------------------+
| Provider          |     | Provider          |
| (kernel, driver,  |     | (.NET CLR,        |
|  OS component)    |     |  application)     |
+--------+----------+     +--------+----------+
         |                          |
         v                          v
+-------------------------------------------+
|          ETW Session (kernel buffer)        |
|  High-performance ring buffers              |
|  Minimal overhead (can be always-on)        |
+-------------------------------------------+
         |
         v
+-------------------+     +-------------------+
| Real-time         |     | Log file          |
| Consumer          |     | (.etl)            |
| (PerfMon, WPA)    |     |                   |
+-------------------+     +-------------------+
```

ETW is how tools like **Windows Performance Analyzer (WPA)**, **PerfView**, and **dotnet-trace** capture detailed performance data. .NET applications automatically emit ETW events for garbage collection, JIT compilation, exceptions, and thread pool activity.

| Feature | Windows ETW | Linux Equivalent |
|---|---|---|
| Kernel tracing | Yes (built-in providers) | ftrace, perf |
| Application tracing | Yes (custom providers) | USDT probes, LTTng |
| Low overhead | Yes (ring buffers) | Yes (ring buffers) |
| Programmable filtering | Yes (at provider level) | eBPF |
| Analysis tools | WPA, PerfView | perf report, flamegraph |

## Comparison: Linux System Management vs Windows

| Feature | Linux | Windows |
|---|---|---|
| Configuration storage | Text files in /etc, ~/.config | Registry (binary database) |
| Service management | systemd (systemctl) | SCM / services.exe (sc.exe) |
| Service definitions | .service unit files | Registry keys under Services |
| Scheduled tasks | cron, systemd timers | Task Scheduler |
| System information | /proc, /sys, sysfs | WMI (Win32_* classes) |
| Centralized logging | journald, syslog | Windows Event Log |
| Performance tracing | perf, ftrace, eBPF | ETW |
| Remote management | SSH + standard tools | WMI, WinRM, PowerShell Remoting |
| Package management | apt, dnf, pacman | MSI, MSIX, winget, Windows Update |
| Startup programs | systemd units, XDG autostart | Registry Run keys, Startup folder, Task Scheduler |

## Real-World Connection

When a Windows system slows down over time, the registry is often partly to blame. Installed and uninstalled applications leave orphaned registry keys. Services accumulate. Startup programs multiply in the `Run` keys. A systems administrator uses tools like Autoruns (Sysinternals) to audit every auto-start location, `sc query type= all` to list all services, and the registry editor to identify bloated hives.

When a security analyst investigates a compromised machine, the registry is a goldmine of forensic evidence. Malware persistence is often achieved through registry Run keys, service registrations, or scheduled tasks. The `NTUSER.DAT` hive preserves evidence of user activity. The `SYSTEM` hive reveals which drivers and services were loaded. Tools like RegRipper extract and analyze registry artifacts for incident response.

When a DevOps engineer manages a fleet of Windows servers, PowerShell Remoting (built on WinRM) and WMI queries replace SSH and shell scripts. `Get-WinEvent` replaces `journalctl`. `Get-Service` replaces `systemctl`. The tooling is different, but the operational patterns -- monitor services, check logs, manage configuration, schedule maintenance -- are universal.

## Interview Angle

**Q: What is the Windows Registry and how does it compare to Linux's approach to configuration?**

A: The registry is a hierarchical binary database that centralizes all Windows configuration: system settings (HKLM\SYSTEM), application settings (HKLM\SOFTWARE and HKCU\SOFTWARE), user preferences (HKCU), and security data (SAM, SECURITY). Linux distributes configuration across text files in /etc, /proc, /sys, and user home directories. The registry provides atomic transactions (multiple changes succeed or fail together), a consistent API (RegOpenKeyEx, RegSetValueEx), and structured queries (WMI). Text files provide human readability, easy version control (git diff), and simple backup (copy the file). The registry's weakness is opacity -- you cannot easily diff two states or review changes. Linux's weakness is inconsistency -- every application invents its own config format. Hive files (SYSTEM, SOFTWARE, NTUSER.DAT) are stored in `C:\Windows\System32\config\` and loaded at specific times: SYSTEM by winload.efi at boot, NTUSER.DAT at user logon.

**Q: How does the Windows Service Control Manager compare to systemd?**

A: Both manage background services (daemons), handle dependencies, and control service lifecycle (start, stop, restart). SCM stores service definitions in the registry (`HKLM\SYSTEM\CurrentControlSet\Services`), while systemd uses unit files (text files in `/etc/systemd/system/`). SCM supports the svchost.exe shared-hosting model where multiple services run in one process to save resources -- systemd has no equivalent because Linux process overhead is lower. Both support dependency ordering, automatic restart on failure, and service isolation. systemd offers more advanced features: socket activation, cgroup-based resource control, and journal integration. SCM integrates deeply with Windows security: each service runs under a specific account (LocalSystem, LocalService, NetworkService, or a managed service account) with its own access token.

---

This concludes the Windows Internals case study. You now have a comprehensive understanding of how Windows implements the OS concepts you learned in sections 00-14, and you can contrast each design decision with its Linux equivalent from section 15. Both operating systems solve the same fundamental problems -- process management, memory management, file storage, I/O, security, and system configuration -- but with different philosophies, tradeoffs, and histories. Understanding both makes you a stronger systems engineer.
