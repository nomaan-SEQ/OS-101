# Pipes and FIFOs

## Anonymous Pipes

Pipes are the oldest and simplest Unix IPC mechanism. When you type `ls | grep foo` in a shell, you're using a pipe. The concept is beautifully simple: a **unidirectional byte stream** between two processes.

Key characteristics:
- **Unidirectional** — data flows in one direction only (one writer, one reader)
- **Related processes only** — the pipe must be created before `fork()`, so only parent-child (or sibling) processes can share it
- **No persistence** — the pipe exists only while the processes are alive
- **Byte stream** — no message boundaries; the reader gets a continuous stream of bytes

## How pipe() Works

The `pipe()` system call creates a pipe and returns two file descriptors:

```c
int fd[2];
pipe(fd);
// fd[0] = read end   (you read FROM this)
// fd[1] = write end   (you write TO this)
```

```
              pipe(fd)
                 |
    fd[0] ------+------ fd[1]
   (read end)   |    (write end)
                 |
         Kernel Buffer
         (circular queue,
          typically 64KB)
```

The pipe is backed by a **kernel buffer** — a fixed-size circular queue. On Linux, the default size is 64KB (since kernel 2.6.11). You can query or change it with `fcntl(fd, F_GETPIPE_SZ)`.

## How Shell Pipes Work

When you type `ls | grep foo`, the shell does this:

```
Step 1: Shell creates a pipe
         pipe(fd) --> fd[0]=read, fd[1]=write

Step 2: Shell forks first child (for "ls")
         Child 1:
           - close(fd[0])          // doesn't need read end
           - dup2(fd[1], STDOUT)   // redirect stdout to pipe write end
           - close(fd[1])          // close original fd (dup2 made a copy)
           - exec("ls")            // replace process with ls

Step 3: Shell forks second child (for "grep foo")
         Child 2:
           - close(fd[1])          // doesn't need write end
           - dup2(fd[0], STDIN)    // redirect stdin to pipe read end
           - close(fd[0])          // close original fd
           - exec("grep", "foo")   // replace process with grep

Step 4: Shell closes both pipe ends (it's not using them)
         close(fd[0])
         close(fd[1])

Step 5: Shell waits for both children to finish
         waitpid(child1)
         waitpid(child2)
```

Visually:

```
  +----------+          +----------+
  |    ls    |          |   grep   |
  |          |   pipe   |          |
  | stdout --|---->-----|--> stdin |
  |  (fd[1]) |  kernel  | (fd[0])  |
  +----------+  buffer  +----------+
                64KB
```

The crucial insight: **each process thinks it's reading/writing to normal files.** `ls` writes to stdout, unaware it's a pipe. `grep` reads from stdin, unaware of the source. This is the power of Unix's "everything is a file descriptor" philosophy.

## Blocking Behavior

Pipes have built-in flow control through blocking:

| Scenario | What Happens |
|----------|-------------|
| **Write to full pipe** | Writer blocks until reader consumes some data |
| **Read from empty pipe** | Reader blocks until writer produces data |
| **Read when all write ends closed** | `read()` returns 0 (EOF) — this is how the reader knows the writer is done |
| **Write when all read ends closed** | Writer receives **SIGPIPE** signal (default: terminate). If SIGPIPE is ignored, `write()` returns -1 with `errno = EPIPE` |

This is why closing unused pipe ends is critical. If the shell forgets to close `fd[1]` in the `grep` child, `grep` will never see EOF — it will block forever waiting for more data.

## Pipe Buffer Size

```
Linux pipe buffer (default 64KB since kernel 2.6.11):

  +----------------------------------------------------------+
  |  Writer writes here -->  [data data data]  --> Reader     |
  +----------------------------------------------------------+
                     Circular kernel buffer

  - Writes <= PIPE_BUF (4KB on Linux) are ATOMIC
    (won't interleave with other writers)
  - Writes > PIPE_BUF may interleave if multiple writers exist
  - Buffer full: writer blocks (or gets EAGAIN in non-blocking mode)
  - Buffer empty: reader blocks (or gets EAGAIN in non-blocking mode)
  - EAGAIN is an error code meaning "try again" — the operation can't
    complete right now without blocking, but it's not a real error.
```

The **PIPE_BUF** atomicity guarantee matters when multiple processes write to the same pipe (e.g., a logging pipe). Writes up to PIPE_BUF bytes (4096 bytes on Linux) are guaranteed to be atomic — they won't interleave with writes from other processes.

## Named Pipes (FIFOs)

Anonymous pipes only work between related processes. **Named pipes (FIFOs)** solve this by giving the pipe a name in the filesystem.

```bash
# Create a named pipe
mkfifo /tmp/myfifo

# Terminal 1: write to it
echo "hello" > /tmp/myfifo

# Terminal 2: read from it
cat /tmp/myfifo    # outputs: hello
```

```
  Process A                              Process B
  (unrelated)                            (unrelated)
  +----------+                           +----------+
  |          |    /tmp/myfifo            |          |
  |  write --|-----> [FIFO] ----------->|-- read   |
  |          |   (filesystem entry,      |          |
  +----------+    kernel buffer)         +----------+
```

FIFOs appear as special files in the filesystem (type `p` in `ls -l`), but they're backed by the same kernel buffer as anonymous pipes. The filesystem entry is just a rendezvous point — no data is stored on disk.

**FIFO behavior:**
- Opening a FIFO for reading **blocks** until another process opens it for writing (and vice versa)
- Once both ends are open, it behaves exactly like an anonymous pipe
- Still unidirectional, still byte stream, still local-only
- The FIFO file persists after processes exit (but the data doesn't)

## Limitations of Pipes and FIFOs

| Limitation | Implication |
|-----------|-------------|
| **Unidirectional** | Need two pipes for bidirectional communication |
| **Byte stream** | No message boundaries — reader must parse where one message ends and another begins |
| **Local only** | Cannot communicate across machines |
| **Anonymous pipes: related processes only** | Must be created before fork() |
| **FIFOs: no message types or priorities** | All data treated equally, FIFO order only |
| **No multiplexing** | A pipe connects exactly two endpoints |

## Real-World Connection

- **Shell scripting** is built on pipes. Complex data transformations are composed from simple tools: `find . -name "*.log" | xargs grep ERROR | sort | uniq -c | sort -rn | head -10`
- **CGI web servers** used pipes to connect the HTTP server to CGI scripts (stdin/stdout)
- **systemd** uses pipes for service stdout/stderr logging (connected to journald)
- **Docker** `docker logs` streams container output through pipes
- **Build systems** like Make pipe compiler output through error-parsing tools

## Interview Angle

**Q: Walk me through what happens at the system call level when a shell executes `ls | grep foo`.**

A: The shell calls `pipe()` to create a pipe, getting two file descriptors. It forks twice. The first child closes the read end of the pipe, uses `dup2()` to redirect its stdout to the pipe's write end, closes the original write fd, then calls `exec("ls")`. The second child closes the write end, redirects its stdin to the pipe's read end with `dup2()`, then calls `exec("grep", "foo")`. The parent (shell) closes both pipe ends and calls `waitpid()` for both children. `ls` writes to stdout, which goes into the pipe buffer. `grep` reads from stdin, which comes from the pipe buffer.

**Q: What happens if a process writes to a pipe but nobody is reading?**

A: The writer receives a SIGPIPE signal, whose default action is to terminate the process. This is why `head -1 /etc/passwd | yes` doesn't hang — when `head` exits after reading one line, `yes` gets SIGPIPE and terminates. If the process has a SIGPIPE handler or ignores the signal, the `write()` call returns -1 with `errno` set to EPIPE.

**Q: What's the difference between an anonymous pipe and a named pipe (FIFO)?**

A: Anonymous pipes are created with `pipe()` and only work between related processes (parent-child or siblings) because file descriptors are inherited through `fork()`. Named pipes (FIFOs) are created with `mkfifo` and exist as entries in the filesystem, so any process with the right permissions can open them. Both use the same kernel buffer mechanism and are unidirectional byte streams.

---

**Next:** [Message Queues](03-message-queues.md) — structured messages with boundaries and priorities, solving key limitations of pipes.
