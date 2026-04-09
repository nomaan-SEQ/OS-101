# Signals

## What Signals Are

A signal is an **asynchronous notification** sent to a process. It tells the process that something happened — a user pressed Ctrl+C, a child process exited, someone called `kill`, a segfault occurred. The process can handle the signal, ignore it, or let the default action happen.

Signals are **not for transferring data**. They carry only a signal number (and optionally a small amount of metadata with `siginfo_t`). Think of signals as interrupts for processes — they break the normal flow of execution to deliver a notification.

```
                    Signal Lifecycle

  1. GENERATED                 2. PENDING               3. DELIVERED
  (someone sends it)          (queued in kernel)        (action taken)
                                                              |
       kill(pid, sig)              kernel                +----+----+
       Ctrl+C                      queues     ---------> | DEFAULT |
       segfault                    signal                | IGNORE  |
       child exit                                        | HANDLER |
                                                         +---------+

  A signal is "pending" between generation and delivery.
  If the signal is blocked, it stays pending until unblocked.
```

## Common Signals

| Signal | Number | Default Action | Triggered By | Use |
|--------|--------|---------------|-------------|-----|
| **SIGHUP** | 1 | Terminate | Terminal closed / `kill -1` | Config reload by convention |
| **SIGINT** | 2 | Terminate | Ctrl+C | Interactive interrupt |
| **SIGQUIT** | 3 | Terminate + core dump | Ctrl+\\ | Quit with debug info |
| **SIGILL** | 4 | Terminate + core dump | Illegal instruction | Hardware/compiler bug |
| **SIGTRAP** | 5 | Terminate + core dump | Breakpoint hit | Debugger (gdb, lldb) |
| **SIGABRT** | 6 | Terminate + core dump | `abort()` | Assertion failure |
| **SIGBUS** | 7 | Terminate + core dump | Bad memory alignment | Hardware error |
| **SIGFPE** | 8 | Terminate + core dump | Division by zero, overflow | Arithmetic error |
| **SIGKILL** | 9 | Terminate | `kill -9` | **Uncatchable** forced termination |
| **SIGUSR1** | 10 | Terminate | User-defined | Application-specific |
| **SIGSEGV** | 11 | Terminate + core dump | Invalid memory access | Null pointer, buffer overflow |
| **SIGUSR2** | 12 | Terminate | User-defined | Application-specific |
| **SIGPIPE** | 13 | Terminate | Write to broken pipe | Reader closed pipe/socket |
| **SIGALRM** | 14 | Terminate | `alarm()` timer expired | Timeouts |
| **SIGTERM** | 15 | Terminate | `kill` (default signal) | Graceful shutdown request |
| **SIGCHLD** | 17 | Ignore | Child process exited/stopped | Child reaping |
| **SIGCONT** | 18 | Continue | `fg`, `kill -CONT` | Resume stopped process |
| **SIGSTOP** | 19 | Stop | `kill -STOP` | **Uncatchable** forced stop |
| **SIGTSTP** | 20 | Stop | Ctrl+Z | Interactive suspend |

Note: Signal numbers may vary across Unix variants. The numbers above are for x86 Linux.

## Signal Handling

A process can respond to a signal in three ways:

### 1. Default Action (SIG_DFL)

Every signal has a default action (terminate, ignore, stop, continue, or core dump). If the process hasn't registered a handler, the default happens.

### 2. Ignore (SIG_IGN)

The process explicitly says "I don't care about this signal." It's silently discarded.

```c
signal(SIGPIPE, SIG_IGN);  // don't kill me on broken pipe
```

### 3. Custom Handler

The process registers a function to run when the signal arrives:

```c
#include <signal.h>

volatile sig_atomic_t shutdown_requested = 0;

void handle_sigterm(int sig) {
    shutdown_requested = 1;  // set a flag, do minimal work
}

int main() {
    struct sigaction sa;
    sa.sa_handler = handle_sigterm;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);

    sigaction(SIGTERM, &sa, NULL);

    while (!shutdown_requested) {
        // main loop: do work
    }

    // clean shutdown: flush buffers, close connections, etc.
    return 0;
}
```

**Always use `sigaction()` over the older `signal()` function.** `signal()` has unportable behavior — on some systems, the handler is reset to SIG_DFL after each delivery. `sigaction()` gives you explicit control.

## SIGKILL and SIGSTOP: The Uncatchable Signals

Two signals **cannot** be caught, blocked, or ignored:

| Signal | Why It's Uncatchable |
|--------|---------------------|
| **SIGKILL (9)** | Guarantees the kernel can always terminate a process. Without this, a buggy or malicious process could install a handler that refuses to exit. |
| **SIGSTOP (19)** | Guarantees the kernel can always pause a process. Used by debuggers and job control. |

This is why `kill -9` is the "last resort" — the process gets no chance to clean up. Always try SIGTERM first and give the process time to shut down gracefully.

```
  Graceful shutdown sequence:

  1. Send SIGTERM (15)     "Please shut down"
       |
  2. Wait (e.g., 30s)     Give process time to clean up
       |
  3. Send SIGKILL (9)     "You had your chance" (forced)
```

This is exactly what Docker does: `docker stop` sends SIGTERM, waits 10 seconds (configurable), then sends SIGKILL.

## Signal Safety

Signal handlers run **asynchronously** — they interrupt your program at any point, including in the middle of library functions. This makes them dangerous if you're not careful.

**The rule:** Only call **async-signal-safe** functions inside signal handlers. The POSIX standard lists exactly which functions are safe.

| Safe in Handlers | NOT Safe in Handlers |
|-----------------|---------------------|
| `write()` (not `printf`!) | `printf()`, `fprintf()` |
| `_exit()` (not `exit()`) | `malloc()`, `free()` |
| `signal()`, `sigaction()` | `exit()` (runs atexit handlers) |
| Setting `volatile sig_atomic_t` variables | Any function that uses global state |
| `read()`, `open()`, `close()` | `syslog()`, `strtol()`, `snprintf()` |

**Why is `printf` unsafe?** Because `printf` uses an internal buffer with a lock. If the signal arrives while the main program is inside `printf` (holding the lock), and the handler also calls `printf`, you get a **deadlock** (or corrupted buffer).

**Best practice:** Do as little as possible in the signal handler. Set a flag (`volatile sig_atomic_t`) and check it in your main loop.

```c
// GOOD: minimal handler
void handler(int sig) {
    flag = 1;  // just set a flag
}

// BAD: doing real work in handler
void handler(int sig) {
    printf("Caught signal %d\n", sig);   // unsafe!
    free(buffer);                         // unsafe!
    fclose(logfile);                      // unsafe!
    exit(0);                              // unsafe! (use _exit)
}
```

## Signals and Threads

In a multithreaded process, signal delivery gets complicated:

- **Process-directed signals** (e.g., `kill(pid, sig)`) are delivered to **any thread** that hasn't blocked the signal. The kernel picks one — you don't get to choose which.
- **Thread-directed signals** (e.g., `pthread_kill(thread, sig)`) go to a specific thread.
- **Synchronous signals** (SIGSEGV, SIGFPE, SIGBUS) go to the thread that caused them.

**Common pattern:** Block all signals in worker threads, and have a dedicated signal-handling thread that calls `sigwait()`:

```c
// In main, before creating threads:
sigset_t set;
sigfillset(&set);
pthread_sigmask(SIG_BLOCK, &set, NULL);  // block all signals

// Create worker threads (they inherit the signal mask)

// Dedicated signal thread:
void *signal_thread(void *arg) {
    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGTERM);
    sigaddset(&set, SIGINT);

    int sig;
    while (1) {
        sigwait(&set, &sig);  // synchronously wait for signals
        // handle sig safely (not in a handler, so all functions are safe)
    }
}
```

This avoids async-signal-safety issues entirely because `sigwait()` delivers the signal synchronously — no handler interruption.

## Common Signal Patterns

### Graceful Shutdown (SIGTERM)

```
Application receives SIGTERM
    |
    v
Stop accepting new requests
    |
    v
Finish processing in-flight requests
    |
    v
Flush buffers, close connections
    |
    v
Exit with status 0
```

### Config Reload (SIGHUP)

```bash
# nginx: reload config without downtime
kill -HUP $(cat /var/run/nginx.pid)
# or: nginx -s reload (which sends SIGHUP internally)
```

The process re-reads its config file and applies changes without restarting. No downtime, no dropped connections.

### Child Reaping (SIGCHLD)

When a child process exits, the parent gets SIGCHLD. The parent must call `waitpid()` to collect the exit status. If it doesn't, the child becomes a **zombie** — dead but still occupying a process table entry.

```c
void handle_sigchld(int sig) {
    // Reap ALL finished children (non-blocking)
    while (waitpid(-1, NULL, WNOHANG) > 0)
        ;
}
```

The `while` loop is important: multiple children may exit before the handler runs (signals don't queue for standard signals — if SIGCHLD is already pending, additional SIGCHLDs are merged).

## Why Signals Are Tricky

| Problem | What Happens |
|---------|-------------|
| **Signals don't queue** | If the same signal is sent twice while blocked, only one delivery occurs |
| **Race conditions** | Signal can arrive between checking a condition and acting on it |
| **Interrupted syscalls** | `read()`, `write()`, `accept()` can return `EINTR` when a signal is caught. You must retry. |
| **Reentrancy** | Handler interrupts the main program at arbitrary points — calling non-reentrant functions corrupts state |
| **Thread complexity** | Which thread gets the signal? Depends on signal masks, signal type, and kernel decisions |
| **Portability** | Signal numbers and behavior differ across Unix variants |

### Handling EINTR

```c
// Robust read that handles signal interruption
ssize_t safe_read(int fd, void *buf, size_t count) {
    ssize_t n;
    do {
        n = read(fd, buf, count);
    } while (n == -1 && errno == EINTR);
    return n;
}
```

Alternatively, use `SA_RESTART` flag with `sigaction()` to automatically restart interrupted syscalls (but not all syscalls are restartable).

## Real-World Connection

- **Ctrl+C** sends SIGINT to the foreground process group. Your shell catches it (that's why the shell doesn't die). The foreground program usually terminates.
- **`docker stop`** sends SIGTERM, waits 10 seconds, then SIGKILL. Your container entrypoint should handle SIGTERM for graceful shutdown.
- **`kill` command** sends SIGTERM by default (not SIGKILL!). `kill -9` is the forced version.
- **Nginx** uses SIGHUP for reload, SIGQUIT for graceful shutdown, SIGUSR1 for log rotation, SIGUSR2 for binary upgrade.
- **Kubernetes** sends SIGTERM to pods during shutdown, with a configurable grace period (default 30s) before SIGKILL.
- **SIGPIPE** is why piping to `head` works without hanging — when `head` closes its stdin after reading enough lines, the upstream process gets SIGPIPE and terminates.
- **SIGSEGV** (segmentation fault) is the signal delivered when your program accesses invalid memory. Crash handlers (like Google Breakpad/Crashpad) catch SIGSEGV to generate crash reports before the process dies.
- **systemd** sends SIGHUP to services when asked to reload, and SIGTERM followed by SIGKILL for stop.

## Interview Angle

**Q: What happens when you run `kill -9 <pid>`? Why should you avoid it?**

A: `kill -9` sends SIGKILL, which unconditionally terminates the process. The kernel handles it — the process never gets a chance to run any code. This means no cleanup: no flushing buffers, no closing database connections, no releasing locks, no writing final log entries, no removing temporary files. You should always try SIGTERM (15) first, which the process can catch and use to shut down gracefully. Only use SIGKILL as a last resort when the process is unresponsive to SIGTERM.

**Q: Why can't you call `printf()` in a signal handler?**

A: Because `printf` is not async-signal-safe. It uses internal buffers and locks. If the signal arrives while the main program is already inside `printf` (holding its buffer lock), the handler's `printf` call will deadlock or corrupt the buffer. The same applies to `malloc`, `free`, and most standard library functions. In a signal handler, you should only call async-signal-safe functions (like `write`, `_exit`) or set a `volatile sig_atomic_t` flag that the main loop checks.

**Q: How does a container handle SIGTERM properly for graceful shutdown?**

A: The container's entrypoint process (PID 1) must register a SIGTERM handler that initiates graceful shutdown — stop accepting new work, finish in-flight requests, flush state, then exit. A common mistake is wrapping your app in a shell script without `exec` — the shell becomes PID 1, receives SIGTERM, but doesn't forward it to the child process. Solutions: use `exec` to replace the shell with your app, or use an init system like `tini` or `dumb-init` that properly forwards signals to child processes.

**Q: What's the difference between SIGTERM, SIGINT, and SIGKILL?**

A: All three can terminate a process, but they serve different purposes. SIGINT (2) means "interactive interrupt" — sent by Ctrl+C, it tells the process the user wants to stop it. SIGTERM (15) means "terminate" — it's the polite, programmatic request for shutdown (what `kill` sends by default). Both can be caught and handled. SIGKILL (9) means "die now" — the kernel enforces it, the process cannot catch or ignore it, and no cleanup occurs. The convention is: Ctrl+C for interactive use, SIGTERM for automated/graceful shutdown, SIGKILL only as a last resort.
