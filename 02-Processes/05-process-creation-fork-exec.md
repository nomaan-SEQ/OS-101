# Process Creation: fork() and exec()

## The Core Idea

Unix has a unique and elegant model for creating processes: **two separate system calls** that each do one thing.

- **`fork()`** — Creates an exact copy of the calling process (the child)
- **`exec()`** — Replaces the current process's memory with a new program

Almost every process on a Unix/Linux system was created with this two-step pattern. Understanding it is non-negotiable for systems work.

---

## fork(): Clone the Current Process

`fork()` creates a **new process** that is a near-identical copy of the calling process (the parent):

```
Before fork():                    After fork():

┌───────────────┐                 ┌───────────────┐
│  Process 1000 │                 │  Process 1000 │  (parent)
│               │                 │               │
│  Code         │    fork()       │  Code         │
│  Data         │  ──────────>    │  Data         │
│  Heap         │                 │  Heap         │
│  Stack        │                 │  Stack        │
│  Open files   │                 │  Open files   │
│  Registers    │                 │  Registers    │
└───────────────┘                 └───────────────┘
                                         +
                                  ┌───────────────┐
                                  │  Process 1001 │  (child)
                                  │               │
                                  │  Code (same)  │
                                  │  Data (copy)  │
                                  │  Heap (copy)  │
                                  │  Stack (copy) │
                                  │  Open files   │  <- shared with parent!
                                  │  Registers    │
                                  └───────────────┘
```

What's copied:
- Address space (code, data, heap, stack)
- Open file descriptor table (but the underlying file descriptions are shared)
- Signal handlers
- Environment variables
- Current working directory
- UID, GID, and most attributes

What's different:
- PID (child gets a new one)
- PPID (child's parent is the calling process)
- The return value of `fork()` (this is the key trick)

### The fork() Return Value Trick

`fork()` returns **twice** — once in the parent and once in the child. But it returns a different value to each:

| Returns | To Whom | Why |
|---------|---------|-----|
| Child's PID (e.g., 1001) | Parent | So the parent knows the child's PID |
| 0 | Child | So the child knows it's the child |
| -1 | Parent (on failure) | fork() failed (out of memory, PID limit) |

This is how both processes know which role they play after the fork:

```
// Pseudocode — the fork+exec pattern

pid = fork()

if (pid == 0) {
    // --- CHILD PROCESS ---
    // This code runs in the child (PID = new)
    // Typically exec() a new program here
    exec("/bin/ls", ["ls", "-la"])
    
    // exec replaces this process entirely
    // This line only runs if exec FAILS
    print("exec failed!")
    exit(1)
}
else if (pid > 0) {
    // --- PARENT PROCESS ---
    // pid = child's PID
    // Wait for child to finish
    status = wait(NULL)
    print("Child exited with status", status)
}
else {
    // --- FORK FAILED ---
    print("fork() failed")
}
```

Both the parent and child continue executing from the exact same point (the instruction after `fork()`), but the return value lets them take different code paths.

---

## exec(): Replace the Program

`exec()` replaces the current process's code, data, heap, and stack with a completely new program loaded from disk. The PID stays the same — it's the same process, just running a different program.

```
Before exec():                    After exec("/bin/ls"):

┌───────────────┐                 ┌───────────────┐
│  Process 1001 │                 │  Process 1001 │  (same PID!)
│               │                 │               │
│  Code: bash   │                 │  Code: ls     │  <- replaced
│  Data: bash's │    exec()       │  Data: ls's   │  <- replaced
│  Heap: bash's │  ──────────>    │  Heap: fresh  │  <- replaced
│  Stack: bash's│                 │  Stack: fresh │  <- replaced
│               │                 │               │
│  fd 0: stdin  │                 │  fd 0: stdin  │  <- PRESERVED
│  fd 1: stdout │                 │  fd 1: stdout │  <- PRESERVED
│  fd 2: stderr │                 │  fd 2: stderr │  <- PRESERVED
│  PID: 1001    │                 │  PID: 1001    │  <- PRESERVED
└───────────────┘                 └───────────────┘
```

What `exec()` replaces:
- Code (text) segment
- Data, BSS, heap, and stack
- Signal handlers (reset to defaults)
- Memory mappings

What `exec()` preserves:
- PID and PPID
- Open file descriptors (unless marked close-on-exec)
- UID, GID
- Working directory
- Terminal/session info

There's actually a family of exec functions: `execl`, `execv`, `execle`, `execve`, `execlp`, `execvp`. They differ in how you pass arguments and whether they search PATH. The underlying system call is `execve()`.

---

## Why Two Separate Calls?

This is the key design insight and a common interview question.

Separating fork and exec gives the parent a **window of opportunity** between cloning and loading the new program. In that window, the child process can set up its environment:

```
pid = fork()

if (pid == 0) {
    // CHILD: window between fork and exec
    
    // Redirect stdout to a file
    close(1)                          // close stdout
    open("output.txt", O_WRONLY)      // opens as fd 1 (lowest available)
    
    // Change working directory
    chdir("/tmp")
    
    // Drop privileges
    setuid(1000)
    
    // Close sensitive file descriptors
    close(secret_fd)
    
    // NOW execute the program
    exec("/bin/ls", ["ls", "-la"])
}
```

This is how the shell implements:

| Shell Feature | What Happens Between fork and exec |
|---------------|-----------------------------------|
| `ls > output.txt` | Close stdout, open output.txt as fd 1 |
| `cat < input.txt` | Close stdin, open input.txt as fd 0 |
| `cmd1 \| cmd2` | Create pipe, wire up fds in each child |
| `cd /tmp && ls` | `chdir("/tmp")` in child |
| `nice -n 10 cmd` | `setpriority()` in child |

If process creation were a single call (like Windows `CreateProcess`), you'd need a complex API with dozens of parameters to specify all these options upfront. Unix's two-step approach is simpler and more composable.

---

## Copy-on-Write (COW)

A naive `fork()` would copy the parent's entire address space — potentially hundreds of megabytes. This would be very slow, and wasteful when the child is about to call `exec()` and throw it all away.

**Copy-on-Write** solves this: `fork()` doesn't actually copy memory. Instead, the parent and child share the same physical pages, marked as read-only:

```
After fork() with COW:

Parent (PID 1000)             Physical Memory         Child (PID 1001)
┌──────────┐                  ┌──────────┐            ┌──────────┐
│ Page A ──────────────────>  │ Page A   │  <──────────── Page A │
│ (read-only)                 │ (shared) │            (read-only)│
│ Page B ──────────────────>  │ Page B   │  <──────────── Page B │
│ (read-only)                 │ (shared) │            (read-only)│
│ Page C ──────────────────>  │ Page C   │  <──────────── Page C │
│ (read-only)                 │ (shared) │            (read-only)│
└──────────┘                  └──────────┘            └──────────┘

After child writes to Page B:

Parent (PID 1000)             Physical Memory         Child (PID 1001)
┌──────────┐                  ┌──────────┐            ┌──────────┐
│ Page A ──────────────────>  │ Page A   │  <──────────── Page A │
│ (read-only)                 │ (shared) │            (read-only)│
│ Page B ──────────────────>  │ Page B   │            ┌──────────┐
│ (read-write now)            │(parent's)│            │ Page B'  │
│                             └──────────┘  <──────────(child's  │
│                             ┌──────────┐            │  copy)   │
│ Page C ──────────────────>  │ Page C   │  <──────────── Page C │
│ (read-only)                 │ (shared) │            (read-only)│
└──────────┘                  └──────────┘            └──────────┘
```

How it works:
1. `fork()` marks all shared pages as read-only in both page tables
2. When either process writes to a page, the hardware triggers a **page fault**
3. The kernel catches the fault, copies that one page, and gives the writer its own copy
4. Only the pages that are actually modified get copied

This makes `fork()` nearly instant — it just copies the page table, not the actual memory. And if the child immediately calls `exec()`, the old pages are simply unmapped without ever being copied.

---

## wait() and waitpid(): Collecting Child Status

After creating a child with `fork()`, the parent typically needs to:
1. Know when the child finishes
2. Get the child's exit status

That's what `wait()` and `waitpid()` do:

```
pid = fork()

if (pid == 0) {
    // Child does work...
    exec("/bin/ls", ["ls"])
} else {
    // Parent waits for child to finish
    child_pid = waitpid(pid, &status, 0)
    
    if (WIFEXITED(status)) {
        exit_code = WEXITSTATUS(status)
        // exit_code is what the child passed to exit()
    }
}
```

| Function | Behavior |
|----------|----------|
| `wait(&status)` | Block until any child exits |
| `waitpid(pid, &status, 0)` | Block until specific child exits |
| `waitpid(-1, &status, WNOHANG)` | Check if any child exited (non-blocking) |

If the parent never calls `wait()`, the child becomes a **zombie** after exiting (more on this in the next file).

---

## How a Shell Works

Now you can understand how a shell like bash works. The basic loop:

```
┌──────────────────────────────────────────────────────────┐
│                    Shell (bash)                           │
│                                                          │
│  while (true) {                                          │
│      1. Print prompt: "$ "                               │
│      2. Read command: "ls -la /tmp"                      │
│      3. Parse command into program + args                │
│      4. pid = fork()                                     │
│                                                          │
│      if (pid == 0) {                                     │
│          // Child:                                       │
│          // Handle redirections (>, <, |)                 │
│          exec("/bin/ls", ["ls", "-la", "/tmp"])           │
│      } else {                                            │
│          // Parent (shell):                              │
│          wait(pid)  // wait for child to finish          │
│      }                                                   │
│                                                          │
│      // Back to step 1                                   │
│  }                                                       │
└──────────────────────────────────────────────────────────┘
```

For background processes (`ls &`), the shell simply skips the `wait()` and goes straight back to the prompt.

For pipelines (`cat file | grep foo | wc -l`), the shell creates a pipe, forks once per command, and wires up the file descriptors:

```
$ cat file | grep foo | wc -l

Shell forks 3 children:

Child 1 (cat):    stdout → pipe1_write
Child 2 (grep):   stdin  → pipe1_read,  stdout → pipe2_write
Child 3 (wc):     stdin  → pipe2_read

         pipe1              pipe2
cat ───────────> grep ───────────> wc
   write    read      write    read
```

This is only possible because fork and exec are separate — the shell uses the window between them to set up the pipes.

---

## Windows: CreateProcess()

Windows takes a fundamentally different approach. Instead of fork+exec, it has a single `CreateProcess()` call that:
- Creates a new process from scratch
- Loads the specified program
- Sets up the environment in one shot

```
// Windows approach (simplified)
CreateProcess(
    "C:\\Windows\\System32\\cmd.exe",   // program
    "/c dir",                           // arguments
    NULL,                               // security attributes
    NULL,                               // thread security
    TRUE,                               // inherit handles
    0,                                  // creation flags
    NULL,                               // environment
    "C:\\Users\\alice",                 // working directory
    &startup_info,                      // stdin/stdout/stderr setup
    &process_info                       // receives PID, handle
)
```

| Aspect | Unix (fork + exec) | Windows (CreateProcess) |
|--------|-------------------|------------------------|
| Process creation | Two steps, composable | One call, monolithic |
| Flexibility | Child can modify itself between fork and exec | Everything specified upfront in parameters |
| COW optimization | fork() is cheap thanks to COW | N/A — no fork |
| API complexity | Simple individual calls | Complex struct-heavy API |
| Overhead | fork is cheap, exec reloads | Creates from scratch each time |

Neither approach is objectively better — they represent different design philosophies. Unix favors composability; Windows favors explicit control.

---

## Real-World Connection

**How Docker starts containers**: `docker run nginx` ultimately calls `fork()` in the Docker daemon (or containerd), then in the child process applies namespaces (`clone()` with flags), sets up cgroups, pivots the filesystem root, and then calls `exec()` to run the container's entrypoint. The fork-exec pattern is the foundation of containers.

**`posix_spawn()`**: Because fork+exec is so common, POSIX provides `posix_spawn()` which combines both steps with an attribute struct for customization. It's more efficient on systems where fork is expensive (e.g., embedded systems without an MMU for virtual memory or COW support). macOS prefers `posix_spawn` internally.

**Pre-fork servers**: Web servers like Apache and Gunicorn use a "pre-fork" model: the master process starts up, loads configuration and code into memory, then forks N worker processes. Thanks to COW, the workers share the parent's code and read-only data in memory. This is why a Python app using Gunicorn with 4 workers doesn't use 4x the memory.

---

## Interview Angle

**Q: Explain fork() and exec(). Why are they separate calls?**

A: `fork()` creates a child process that is a copy of the parent. `exec()` replaces the current process's memory with a new program. They're separate because the gap between them allows the child to customize its environment — redirect file descriptors, change directories, drop privileges, set up pipes — before loading the new program. This is how shells implement pipes, redirection, and other features. If they were combined, you'd need a massive parameter set to specify all possible customizations upfront.

**Q: What is Copy-on-Write and why does it matter for fork()?**

A: COW means fork() doesn't actually copy the parent's memory. Instead, both processes share the same physical pages, marked read-only. When either process writes to a page, the hardware triggers a page fault, and the kernel copies just that one page. This makes fork() nearly instant regardless of process size, and if the child immediately calls exec(), no pages are ever copied — the old mappings are simply discarded.

**Q: What does fork() return and why?**

A: fork() returns the child's PID to the parent and 0 to the child. This lets both processes — which are executing the same code from the same point — determine their role. The parent gets the child PID so it can wait() for it or send it signals. The child gets 0 so it knows to exec() a new program or take the child code path. On failure, fork() returns -1 to the parent (no child is created).

**Q: How does a shell implement `ls > output.txt`?**

A: The shell forks a child process. In the child, before calling exec, it closes file descriptor 1 (stdout), then opens `output.txt` which gets assigned fd 1 (the lowest available). Now stdout points to the file. Then the child calls exec("/bin/ls"). ls writes to stdout (fd 1), which goes to the file. The parent waits for the child to finish.

---

**Next**: [06-orphan-and-zombie-processes.md](06-orphan-and-zombie-processes.md)
