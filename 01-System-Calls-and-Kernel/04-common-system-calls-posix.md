# Common System Calls and POSIX

## Why POSIX Matters

**POSIX** (Portable Operating System Interface) is a set of standards that defines the system call interface, shell utilities, and threading APIs that Unix-like operating systems should provide. When you write code against POSIX syscalls, it runs on Linux, macOS, FreeBSD, and other compliant systems with minimal changes.

Linux is "mostly POSIX-compliant" but also provides many Linux-specific syscalls. Knowing which is which matters when you care about portability.

## System Calls by Category

### Process Management

| Syscall | What It Does | When You'd Use It |
|---------|-------------|-------------------|
| `fork()` | Create a child process (copy of parent) | Spawning worker processes, shell job control |
| `execve(path, argv, envp)` | Replace process image with new program | Running a different program after fork |
| `wait(status)` / `waitpid(pid, status, opts)` | Wait for child process to change state | Parent collecting child exit status |
| `exit(status)` | Terminate current process | Clean process shutdown |
| `getpid()` / `getppid()` | Get process ID / parent process ID | Logging, PID files, process management |
| `kill(pid, signal)` | Send a signal to a process | Asking a process to terminate (`SIGTERM`) or checking if it's alive (`kill -0`) |
| `clone(flags)` | Create process/thread (Linux-specific, flexible fork) | Thread creation (used by `pthread_create`), container namespaces |

```
FORK + EXEC PATTERN (how shells launch programs)
==================================================

  Parent (bash, pid=100)
       |
       | fork()
       |-------------> Child (pid=200)
       |                    |
       | waitpid(200)       | execve("/bin/ls", ...)
       | (blocks...)        |
       |                   [ls runs, prints files]
       |                    |
       |                   exit(0)
       |<--- child exits ---
       | waitpid returns
       | (continues)
```

### File I/O

| Syscall | What It Does | When You'd Use It |
|---------|-------------|-------------------|
| `open(path, flags, mode)` | Open file, return file descriptor | Any file access -- returns fd 3, 4, 5... |
| `close(fd)` | Close a file descriptor | Releasing resources, avoiding fd leaks |
| `read(fd, buf, count)` | Read bytes from fd into buffer | Reading file contents, network data, pipe input |
| `write(fd, buf, count)` | Write bytes from buffer to fd | Writing file contents, stdout, network output |
| `lseek(fd, offset, whence)` | Move the file read/write position | Random access in files (seek to byte 1000) |
| `stat(path, statbuf)` / `fstat(fd, statbuf)` | Get file metadata (size, perms, timestamps) | Checking file existence, size, modification time |
| `dup2(oldfd, newfd)` | Duplicate fd oldfd to newfd | Shell I/O redirection (`>`, `<`, `2>&1`) |

```
FILE DESCRIPTOR OPERATIONS
===========================

  int fd = open("/tmp/data.txt", O_RDWR | O_CREAT, 0644);
  //  fd = 3 (first available after stdin=0, stdout=1, stderr=2)

  write(fd, "hello", 5);     // write 5 bytes at current position
  lseek(fd, 0, SEEK_SET);    // rewind to beginning
  read(fd, buf, 5);          // read back "hello"
  close(fd);                  // release fd 3

  // I/O redirection with dup2:
  int log = open("output.log", O_WRONLY | O_CREAT, 0644);
  dup2(log, 1);              // stdout (fd 1) now points to output.log
  printf("this goes to the file\n");  // write() on fd 1 -> output.log
```

### Directory Operations

| Syscall | What It Does | When You'd Use It |
|---------|-------------|-------------------|
| `mkdir(path, mode)` | Create a directory | File management, temp directories |
| `rmdir(path)` | Remove an empty directory | Cleanup operations |
| `getdents(fd, buf, count)` | Read directory entries (low-level) | Implementing `ls`, file scanners |
| `chdir(path)` | Change working directory | `cd` in shells, setting process working directory |

Note: `opendir()` and `readdir()` are libc functions (not direct syscalls) that wrap `open()` + `getdents()` on Linux.

### Memory Management

| Syscall | What It Does | When You'd Use It |
|---------|-------------|-------------------|
| `mmap(addr, len, prot, flags, fd, off)` | Map files/devices into memory, or allocate anonymous memory | Large allocations, memory-mapped files, shared memory |
| `munmap(addr, len)` | Unmap a memory region | Releasing mmap'd regions |
| `brk(addr)` / `sbrk(increment)` | Change heap end (data segment break) | Small allocations (malloc uses this for <128KB) |
| `mprotect(addr, len, prot)` | Change memory protection | Making memory executable (JIT), guard pages |
| `madvise(addr, len, advice)` | Hint to kernel about memory usage | `MADV_DONTNEED` to release pages, `MADV_HUGEPAGE` for THP |

```
MALLOC UNDER THE HOOD
======================

  malloc(64)                malloc(256 * 1024)
       |                         |
       v                         v
  brk() / sbrk()            mmap(NULL, 256*1024,
  (extend heap by               PROT_READ|PROT_WRITE,
   small amount)                 MAP_PRIVATE|MAP_ANONYMOUS,
                                 -1, 0)
  
  glibc's malloc uses brk    glibc's malloc uses mmap
  for allocations < 128KB    for allocations >= 128KB
  (tunable with mallopt)     (returns independent region)
```

### Inter-Process Communication (IPC)

| Syscall | What It Does | When You'd Use It |
|---------|-------------|-------------------|
| `pipe(fds[2])` | Create a unidirectional byte stream | Shell pipes (`ls \| grep foo`), parent-child communication |
| `socket(domain, type, proto)` | Create a network/IPC endpoint | Network communication, Unix domain sockets |
| `shmget(key, size, flags)` | Create/access shared memory segment | High-performance IPC (database shared buffers) |
| `msgget(key, flags)` | Create/access message queue | Producer-consumer patterns (System V IPC) |
| `eventfd(initval, flags)` | Create an event notification fd | Lightweight signaling between threads/processes |

```
PIPE: HOW "ls | grep .txt" WORKS
==================================

  Shell (bash):
    int fds[2];
    pipe(fds);         // fds[0]=read end, fds[1]=write end

    if (fork() == 0) {            // Child 1: ls
        close(fds[0]);             // close read end
        dup2(fds[1], 1);          // stdout -> pipe write end
        execve("/bin/ls", ...);
    }

    if (fork() == 0) {            // Child 2: grep
        close(fds[1]);             // close write end
        dup2(fds[0], 0);          // stdin -> pipe read end
        execve("/bin/grep", ["grep", ".txt"], ...);
    }

    close(fds[0]); close(fds[1]); // parent closes both ends
    wait(NULL); wait(NULL);        // wait for both children
```

### Multiplexing and Miscellaneous

| Syscall | What It Does | When You'd Use It |
|---------|-------------|-------------------|
| `select(nfds, readfds, writefds, errfds, timeout)` | Wait for I/O on multiple fds (legacy) | Old network servers, max 1024 fds |
| `poll(fds, nfds, timeout)` | Wait for I/O on multiple fds (better) | Moderate-scale I/O multiplexing |
| `epoll_create` / `epoll_ctl` / `epoll_wait` | Scalable I/O event notification (Linux) | High-performance servers (nginx, Redis, Node.js) |
| `ioctl(fd, request, arg)` | Device-specific control operations | Terminal settings, network interface config, device control |
| `fcntl(fd, cmd, arg)` | File descriptor control | Non-blocking I/O, file locking, fd flags |

```
I/O MULTIPLEXING EVOLUTION
===========================

  select (1983)     poll (1986)      epoll (2002, Linux)
  
  O(n) per call     O(n) per call    O(1) per event
  Max 1024 fds      No fd limit      No fd limit
  Copies fd sets    Copies pollfd    Kernel maintains state
  each call         array each call  (epoll_ctl modifies)

  Modern servers use epoll (Linux), kqueue (BSD/macOS), or IOCP (Windows)
```

## strace: Your Syscall Debugger

`strace` traces every system call a process makes. It's one of the most useful debugging tools in the Linux toolkit.

```
EXAMPLE: strace on a simple "cat /etc/hostname"
=================================================

  $ strace cat /etc/hostname

  execve("/usr/bin/cat", ["cat", "/etc/hostname"], ...) = 0
  ...                                           # loader setup
  openat(AT_FDCWD, "/etc/hostname", O_RDONLY)  = 3
  fstat(3, {st_mode=S_IFREG|0644, st_size=12}) = 0
  read(3, "my-server\n", 131072)               = 10
  write(1, "my-server\n", 10)                  = 10
  read(3, "", 131072)                          = 0   # EOF
  close(3)                                     = 0
  exit_group(0)                                = ?

  Key observations:
  - openat returns fd 3
  - read gets 10 bytes from the file
  - write sends those 10 bytes to stdout (fd 1)
  - second read returns 0 (end of file)
  - process exits with status 0
```

Useful `strace` flags:

| Flag | Purpose | Example |
|------|---------|---------|
| `-e trace=file` | Only show file-related syscalls | `strace -e trace=file nginx` |
| `-e trace=network` | Only show network syscalls | `strace -e trace=network curl google.com` |
| `-e trace=process` | Only show process syscalls | `strace -e trace=process bash -c "ls"` |
| `-p PID` | Attach to running process | `strace -p 1234` |
| `-c` | Summary: count time and calls per syscall | `strace -c ./myapp` |
| `-f` | Follow child processes (fork) | `strace -f ./server` |
| `-T` | Show time spent in each syscall | `strace -T cat /etc/hosts` |

The `-c` flag is particularly useful for performance analysis:

```
  $ strace -c ls /tmp
  
  % time   seconds  calls  errors  syscall
  ------ --------- ------ ------- --------
   45.23  0.000120     12         read
   22.61  0.000060      8         openat
   12.03  0.000032      3         write
    8.27  0.000022     15       2 stat
    ...
  ------ --------- ------ ------- --------
  100.00  0.000266     58       4 total
```

## POSIX vs Linux-Specific Syscalls

| Category | POSIX (portable) | Linux-specific |
|----------|-----------------|----------------|
| **Process** | `fork`, `exec`, `wait`, `kill` | `clone`, `unshare`, `setns`, `pidfd_open` |
| **I/O** | `read`, `write`, `select`, `poll` | `epoll_*`, `io_uring_*`, `splice`, `sendfile` |
| **Memory** | `mmap`, `munmap`, `mprotect` | `madvise`, `remap_file_pages`, `userfaultfd` |
| **File** | `open`, `close`, `stat`, `mkdir` | `openat2`, `statx`, `fanotify_*`, `name_to_handle_at` |
| **IPC** | `pipe`, `socket`, `shmget`, `msgget` | `eventfd`, `signalfd`, `timerfd`, `memfd_create` |
| **Security** | `setuid`, `setgid` | `seccomp`, `landlock_*`, `cap_*` |

If you're building something that only runs on Linux (most cloud services), use the Linux-specific calls -- they're often more efficient and flexible. If you're building portable software (libraries, CLI tools), stick to POSIX.

---

## Real-World Connection

- **nginx** uses `epoll` for its event loop, which is why it handles 10,000+ concurrent connections efficiently. `select` would require scanning all fds every time; `epoll` only returns the fds that are actually ready.

- **`strace` in production**: When a production service hangs, attaching `strace -fp <pid>` reveals what syscall it's blocked on. Often it's a `futex()` call (mutex contention), a `read()` on a network socket (waiting for a slow dependency), or an `openat()` on an NFS mount (filesystem hang).

- **`sendfile()`** is a Linux-specific syscall that copies data between file descriptors in kernel space, avoiding the user-space buffer entirely. Web servers like nginx use it to serve static files -- the file data goes straight from the page cache to the network socket without ever being copied into user space.

- **`io_uring`** is the newest Linux I/O interface (kernel 5.1+), providing a shared ring buffer between user space and kernel for submitting and completing I/O operations with minimal syscall overhead. It's becoming the standard for high-performance database and storage systems.

---

## Interview Angle

**Q: What is the difference between `fork()` and `exec()`? Why do we need both?**

A: `fork()` creates a new process that is a copy of the parent (same code, same data, same open files). `exec()` replaces the current process's memory image with a new program. They're separate because Unix philosophy gives you flexibility: after `fork()`, the child can set up its environment (redirect I/O with `dup2`, set environment variables, change directory) before calling `exec()` to run the new program. Shells depend on this pattern -- `fork()`, then configure pipes and redirections in the child, then `exec()` the command. If you only had a single "spawn" call, you'd need to pass all that configuration as parameters.

**Q: Explain the differences between `select`, `poll`, and `epoll`.**

A: `select` is the oldest, limited to 1024 file descriptors (FD_SETSIZE), and requires rebuilding the fd set and copying it to/from kernel space on every call -- O(n) per call. `poll` removes the fd limit and uses a simpler array interface, but still requires passing the entire array to the kernel each call -- still O(n). `epoll` fundamentally changes the model: `epoll_create` makes a kernel object, `epoll_ctl` registers/modifies fds once, and `epoll_wait` returns only the ready fds -- O(1) per event, O(ready_fds) per call. For 10,000 connections where 10 are active, `epoll` processes 10 events while `select`/`poll` scan 10,000. This is why `epoll` is the foundation of every high-performance Linux server.

**Q: How would you debug a process that seems to be hanging?**

A: First, `strace -p <pid>` to see what syscall it's blocked on. If it's stuck in `futex()`, it's probably a mutex deadlock or contention -- use `gdb` to inspect the stack. If it's stuck in `read()` or `recvfrom()`, it's waiting on I/O from a network peer or file -- check the remote service. If it's stuck in `nanosleep()`, it's intentionally sleeping. `strace -c -p <pid>` gives a statistical summary of syscall usage. Combine with `/proc/<pid>/stack` to see the kernel stack trace, and `/proc/<pid>/fd/` to see open file descriptors and what they point to.

---

**Next**: [05-kernel-modules-and-extensions.md](05-kernel-modules-and-extensions.md)
