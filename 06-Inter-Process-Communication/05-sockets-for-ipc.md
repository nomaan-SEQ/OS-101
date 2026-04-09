# Sockets for IPC

## Unix Domain Sockets

When people think of sockets, they think of networking. But sockets are also the **most versatile local IPC mechanism**. Unix Domain Sockets (UDS) use the socket API with a filesystem path instead of an IP address and port — giving you all the power of the socket interface without any network stack overhead.

```
  Network Socket (TCP):                Unix Domain Socket:

  Process A        Process B           Process A          Process B
  +--------+      +--------+          +--------+         +--------+
  |        |      |        |          |        |         |        |
  | connect|----->| accept |          | connect|-------->| accept |
  |        |      |        |          |        |         |        |
  +--------+      +--------+          +--------+         +--------+
       |               |                   |                  |
  IP: 127.0.0.1:8080                  Path: /var/run/app.sock
  Goes through full                   Bypasses network stack
  TCP/IP stack                        entirely (kernel shortcut)
```

### Key Characteristics

- **Address family:** `AF_UNIX` (also called `AF_LOCAL`)
- **Address:** A filesystem path (e.g., `/var/run/docker.sock`) instead of IP:port
- **Bidirectional** — both sides can send and receive
- **Two modes:**
  - `SOCK_STREAM` — connection-oriented, reliable byte stream (like TCP)
  - `SOCK_DGRAM` — datagram-based, message boundaries preserved (like UDP, but reliable on Unix)
- **Faster than TCP loopback** — no TCP handshake, no checksums, no routing, no packet fragmentation
- **Supports credential passing** — receiver can verify the sender's PID, UID, GID
- **Supports file descriptor passing** — send open file descriptors to another process

### Basic Usage

```c
#include <sys/socket.h>
#include <sys/un.h>

// --- Server side ---
int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);

struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, "/tmp/myapp.sock", sizeof(addr.sun_path) - 1);

unlink("/tmp/myapp.sock");  // remove stale socket file
bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
listen(server_fd, 5);

int client_fd = accept(server_fd, NULL, NULL);
// read/write on client_fd

// --- Client side ---
int sock = socket(AF_UNIX, SOCK_STREAM, 0);

struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, "/tmp/myapp.sock", sizeof(addr.sun_path) - 1);

connect(sock, (struct sockaddr *)&addr, sizeof(addr));
// read/write on sock
```

### Why AF_UNIX is Faster than TCP Loopback

Even `127.0.0.1` traffic goes through the full network stack:

```
  TCP Loopback (127.0.0.1):

  send() -> TCP layer -> IP layer -> Loopback interface
         -> IP layer -> TCP layer -> recv()

  Overhead: TCP handshake, sequence numbers, checksums,
            congestion control, packet headers, ACKs

  Unix Domain Socket:

  send() -> kernel buffer -> recv()

  Overhead: minimal — just buffer copy
```

Benchmarks typically show Unix domain sockets achieving **2-3x higher throughput** and **significantly lower latency** compared to TCP loopback for local communication.

## TCP/UDP Loopback for Local IPC

You can also use regular network sockets on `127.0.0.1` for local IPC:

```c
// Server binds to localhost
bind(server_fd, "127.0.0.1:8080");

// Client connects to localhost
connect(client_fd, "127.0.0.1:8080");
```

**Why would you ever use TCP loopback over Unix sockets?**

- **Portability** — TCP sockets work on every OS, including Windows (which historically lacked good Unix socket support)
- **Easy transition to network** — change the IP address and it works across machines
- **Tooling** — `curl`, `netstat`, Wireshark all work with TCP out of the box
- **Language support** — every language has TCP socket libraries; Unix socket support varies

## socketpair() for Related Processes

For parent-child communication, `socketpair()` creates a pair of connected Unix domain sockets — like `pipe()` but **bidirectional**:

```c
int sv[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, sv);

pid_t pid = fork();
if (pid == 0) {
    // Child: use sv[1]
    close(sv[0]);
    write(sv[1], "hello", 5);
    read(sv[1], buf, sizeof(buf));  // can also read!
} else {
    // Parent: use sv[0]
    close(sv[1]);
    read(sv[0], buf, sizeof(buf));
    write(sv[0], "world", 5);      // can also write!
}
```

This is strictly better than pipes for parent-child IPC when you need bidirectional communication.

## File Descriptor Passing

One of the most powerful features of Unix domain sockets: you can **send open file descriptors** from one process to another. The receiving process gets a new file descriptor that refers to the same underlying file/socket/device.

```
  Process A has fd=5 (open file)
       |
       | sendmsg() with SCM_RIGHTS
       v
  Unix Domain Socket
       |
       | recvmsg() with SCM_RIGHTS
       v
  Process B gets fd=8 (same underlying file!)
```

This is used for:
- **Pre-forking servers:** Parent accepts a connection, passes the connected socket fd to a worker child
- **Privilege separation:** A privileged process opens a restricted resource and passes the fd to an unprivileged process
- **Socket activation:** systemd opens sockets on behalf of services and passes the fds at startup

## Comparison: Sockets vs Other Local IPC

| Feature | Anonymous Pipe | Named Pipe (FIFO) | Shared Memory | Unix Domain Socket | TCP Loopback |
|---------|---------------|-------------------|---------------|-------------------|-------------|
| **Direction** | Unidirectional | Unidirectional | Bidirectional | Bidirectional | Bidirectional |
| **Unrelated processes** | No | Yes | Yes | Yes | Yes |
| **Message boundaries** | No | No | No | SOCK_DGRAM: Yes | TCP: No, UDP: Yes |
| **Speed (relative)** | Fast | Fast | Fastest | Fast | Slower |
| **Cross-machine** | No | No | No | No | Yes |
| **FD passing** | No | No | No | Yes | No |
| **Credential checking** | No | No | No | Yes (SO_PEERCRED) | No |
| **Connection management** | No | No | No | Yes (accept/connect) | Yes |

## When to Use Sockets for IPC

**Choose Unix domain sockets when:**
- You need bidirectional communication
- You want connection management (accept/reject clients)
- You might later need to go cross-machine (swap AF_UNIX for AF_INET)
- You need to pass file descriptors between processes
- You need to verify client credentials (UID/GID)
- You want a well-understood, debuggable protocol

**Choose something else when:**
- Simple parent-child data flow: use a pipe
- Maximum throughput for large shared data: use shared memory
- Just notifying a process: use a signal

## Real-World Connection

- **Docker daemon** listens on `/var/run/docker.sock`. The `docker` CLI connects to this Unix domain socket. This is why `docker` commands require the user to have access to this socket file.
- **MySQL and PostgreSQL** default to Unix domain sockets for local connections. When you connect to `localhost`, the client library uses `/var/run/mysqld/mysqld.sock` or `/var/run/postgresql/.s.PGSQL.5432` instead of TCP. This is faster than TCP loopback.
- **Nginx to PHP-FPM** communication uses Unix domain sockets (`fastcgi_pass unix:/var/run/php-fpm.sock`). This avoids TCP overhead for the most common deployment where both run on the same machine.
- **systemd** uses socket activation: it creates listening sockets and passes them to services via file descriptor passing. The service doesn't need to know the socket path — it receives pre-opened file descriptors.
- **SSH agent forwarding** (`ssh-agent`) communicates through a Unix domain socket specified by `SSH_AUTH_SOCK`.
- **X11/Wayland** window systems use Unix domain sockets for client-server communication between applications and the display server.
- **containerd and CRI-O** (container runtimes) expose their APIs over Unix domain sockets.

## Interview Angle

**Q: Why does Docker use a Unix domain socket instead of a TCP port?**

A: Several reasons. First, performance — Unix domain sockets are faster than TCP loopback because they bypass the entire network stack. Second, security — socket file permissions provide access control (only users in the `docker` group can access `/var/run/docker.sock`), which is simpler and more secure than TCP authentication. Third, credential passing — the daemon can verify the client's UID/GID via `SO_PEERCRED`. TCP would require additional authentication mechanisms. The tradeoff is that the Docker API is only accessible locally unless you explicitly enable TCP (which introduces security risks).

**Q: What is file descriptor passing and why is it useful?**

A: Unix domain sockets allow one process to send an open file descriptor to another process using `sendmsg()` with `SCM_RIGHTS`. The receiving process gets a new fd that points to the same underlying kernel object (file, socket, device). This is useful for privilege separation (a privileged process opens a restricted resource and passes the fd to an unprivileged worker), pre-forking servers (parent accepts connections and distributes them to workers), and socket activation (systemd opens sockets on behalf of services). No other IPC mechanism supports this.

**Q: When would you choose TCP loopback over Unix domain sockets for local IPC?**

A: When portability is more important than performance. TCP works identically on every OS, while Unix domain socket support varies (especially on Windows). TCP also makes the transition to distributed deployment trivial — just change the IP address. Additionally, TCP has better tooling support: you can debug with curl, inspect with Wireshark, and monitor with netstat. For prototyping or applications that might later go cross-machine, TCP loopback is a pragmatic default.

---

**Next:** [Signals](06-signals.md) — asynchronous notifications that are simple in concept but surprisingly tricky in practice.
