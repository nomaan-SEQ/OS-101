# Sockets API — TCP and UDP

A socket is the OS abstraction for a network communication endpoint. To the kernel, it is a file descriptor — the same kind of thing as an open file or a pipe. To the programmer, it is a set of syscalls that create, configure, and transfer data over network connections. The socket API is the boundary between your application and the kernel's entire network stack.

Every web server, database client, message queue, and RPC framework you have ever used calls these same syscalls underneath.

## Socket API Lifecycle

### TCP Server

```
Server                              Client
──────                              ──────

socket(AF_INET, SOCK_STREAM, 0)     socket(AF_INET, SOCK_STREAM, 0)
  │  → creates socket fd               │
  ▼                                     │
bind(fd, addr, len)                     │
  │  → assigns IP:port                  │
  ▼                                     │
listen(fd, backlog)                     │
  │  → marks socket as passive          │
  │  → creates SYN + accept queues      │
  ▼                                     ▼
accept(fd, ...) ◄──── 3-way ────► connect(fd, addr, len)
  │  → blocks until                     │
  │    connection arrives               │
  │  → returns NEW fd for               │
  │    this connection                   │
  ▼                                     ▼
read(conn_fd, buf, len) ◄──────► write(fd, buf, len)
write(conn_fd, buf, len) ──────► read(fd, buf, len)
  │                                     │
  ▼                                     ▼
close(conn_fd)                    close(fd)
```

Critical details:
- `listen()` creates TWO queues: the SYN queue (half-open connections) and the accept queue (completed connections). The `backlog` parameter historically controlled both, but on modern Linux it sets the accept queue size.
- `accept()` returns a NEW file descriptor for each connection. The original listening socket stays open to accept more connections.
- A typical server calls `accept()` in a loop, or uses `epoll` to be notified when new connections are ready.

### TCP Client

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
connect(fd, &server_addr, sizeof(server_addr));
// connect() triggers the 3-way handshake:
//   Client → SYN → Server
//   Server → SYN+ACK → Client
//   Client → ACK → Server
// connect() returns when handshake completes (or times out)

write(fd, request, request_len);
read(fd, response, response_len);
close(fd);
```

`connect()` is a blocking call by default — it does not return until the TCP handshake completes or fails. In non-blocking mode, it returns immediately with `EINPROGRESS`, and you use `epoll` to wait for the connection to be established.

### UDP

```c
int fd = socket(AF_INET, SOCK_DGRAM, 0);
bind(fd, &local_addr, sizeof(local_addr));  // optional for sender

// No connect() needed — each sendto() specifies the destination
sendto(fd, data, len, 0, &dest_addr, sizeof(dest_addr));
recvfrom(fd, buf, buf_len, 0, &sender_addr, &addr_len);

close(fd);
```

No connection setup, no state machine, no guaranteed delivery. Each `sendto()` is an independent datagram. The kernel does not buffer or reorder — what arrives is what you get.

## The Three-Way Handshake from Syscall Perspective

```
Client Process            Kernel (Client)         Kernel (Server)           Server Process
──────────────            ───────────────         ───────────────           ──────────────

connect(fd, ...)  ──►  Send SYN ──────────────► Receive SYN
[blocks]                [SYN_SENT]                Create entry in SYN queue
                                                  [SYN_RECV]
                                                  Send SYN+ACK ─────────►
                        Receive SYN+ACK ◄──────
                        Send ACK ──────────────► Receive ACK
                        [ESTABLISHED]             Move to accept queue
connect() returns ◄──                             [ESTABLISHED]
                                                                            accept() returns ◄──
                                                                            (new conn_fd)
```

Notice: `connect()` on the client returns after the handshake completes. `accept()` on the server returns when a fully-established connection is waiting in the accept queue. The SYN/SYN+ACK/ACK exchange is entirely handled by the kernel — user space never sees individual handshake packets.

## Socket Options That Matter

| Option | Level | Purpose | Practical Use |
|--------|-------|---------|---------------|
| `SO_REUSEADDR` | SOL_SOCKET | Allow binding to address in TIME_WAIT | Restart server without "address already in use" error |
| `SO_REUSEPORT` | SOL_SOCKET | Multiple sockets bind to same port | Kernel distributes connections across worker processes |
| `TCP_NODELAY` | IPPROTO_TCP | Disable Nagle's algorithm | Send small packets immediately (reduce latency) |
| `SO_KEEPALIVE` | SOL_SOCKET | Send probes on idle connections | Detect dead peers (default: probe after 2 hours) |
| `SO_RCVBUF` | SOL_SOCKET | Set receive buffer size | Increase for high-throughput, long-latency links |
| `SO_SNDBUF` | SOL_SOCKET | Set send buffer size | Controls how much data can be buffered before send() blocks |
| `TCP_QUICKACK` | IPPROTO_TCP | Disable delayed ACKs | Reduce latency for request-response protocols |

### Nagle's Algorithm and TCP_NODELAY

> **Analogy:** Nagle's algorithm is like waiting at a bus stop to collect more passengers before departing. Instead of sending a bus (packet) for every single passenger (byte), the algorithm waits briefly to batch small writes into a single, fuller packet. This improves bandwidth efficiency but adds a small delay — which is why interactive applications (SSH, gaming) disable it with `TCP_NODELAY`.

Nagle's algorithm buffers small outgoing packets and waits until either (a) enough data accumulates to fill a segment, or (b) an ACK arrives for previously sent data. This reduces the number of tiny packets on the wire.

The problem: for request-response protocols (Redis, HTTP/2), Nagle adds latency. You send a small request, Nagle holds it waiting for more data, but there is no more data — you are waiting for the response.

```
Without TCP_NODELAY (Nagle ON):
  App: write(4 bytes) → Kernel: "hold, maybe more coming..."
  App: (waiting for response)
  Kernel: (timer expires, sends the 4 bytes)     ← unnecessary delay
  
With TCP_NODELAY (Nagle OFF):
  App: write(4 bytes) → Kernel: send immediately
  Server: receives, responds
  App: gets response quickly
```

This is why Redis, memcached, and most latency-sensitive services set `TCP_NODELAY`.

### SO_REUSEPORT: Kernel-Level Load Balancing

Without `SO_REUSEPORT`, only one socket can bind to a given port. A server typically has one process accept all connections, then distributes them to workers. This creates a bottleneck.

With `SO_REUSEPORT`, multiple processes (or threads) each create their own socket bound to the same port. The kernel distributes incoming connections across all of them using a hash.

```
Without SO_REUSEPORT:              With SO_REUSEPORT:

  Client connections                 Client connections
         │                                │
         ▼                           ┌────┼────┐
  ┌─────────────┐                    ▼    ▼    ▼
  │  Listener   │              ┌────┐ ┌────┐ ┌────┐
  │  (1 socket) │              │ W1 │ │ W2 │ │ W3 │
  └──────┬──────┘              │sock│ │sock│ │sock│
    ┌────┼────┐                └────┘ └────┘ └────┘
    ▼    ▼    ▼                 Each worker has its
   W1   W2   W3                 own socket on :8080
```

Nginx uses this since version 1.9.1. It measurably improves performance by eliminating lock contention on the accept queue.

## Backlog Queue: SYN Queue and Accept Queue

When a SYN arrives at a listening socket, the kernel must manage two stages:

```
Client SYN ──►  ┌───────────────┐     ┌───────────────┐
                │   SYN Queue   │ ──► │  Accept Queue  │ ──► accept()
                │ (half-open)   │     │  (completed)   │     returns fd
                │               │     │                │
                │ Stores:       │     │ Stores:        │
                │  - SYN_RECV   │     │  - ESTABLISHED │
                │    entries    │     │    connections  │
                └───────────────┘     └───────────────┘
                 Size: controlled      Size: backlog
                 by kernel             parameter in
                 (tcp_max_syn_backlog)  listen()
```

- **SYN queue**: Holds connections that have received a SYN but not yet completed the handshake. Size controlled by `net.ipv4.tcp_max_syn_backlog`.
- **Accept queue**: Holds fully established connections waiting for the application to call `accept()`. Size set by the `backlog` argument to `listen()`.

### SYN Flood Attack

An attacker sends millions of SYN packets with spoofed source IPs. The server allocates SYN queue entries for each one and sends SYN+ACK, but the ACK never arrives (the source IP is fake). The SYN queue fills up, and legitimate connections are rejected.

```
Attacker                          Server
────────                          ──────

SYN (src=1.2.3.4) ──────────►   SYN queue: [1.2.3.4] 
SYN (src=5.6.7.8) ──────────►   SYN queue: [1.2.3.4, 5.6.7.8]
SYN (src=9.10.11.12) ────────►  SYN queue: [1.2.3.4, 5.6.7.8, 9.10.11.12]
... millions more ...            SYN queue: FULL → drop new SYNs

Legitimate client:
SYN ──────────────────────────►  DROPPED (queue full)
```

### SYN Cookies: The Defense

With SYN cookies enabled (`net.ipv4.tcp_syncookies = 1`), the server does NOT allocate SYN queue entries. Instead, it encodes connection state (timestamps, MSS, etc.) into the SYN+ACK sequence number itself. When the final ACK arrives, the server reconstructs the connection state from the sequence number.

No queue entry needed until the handshake completes. The SYN queue cannot be exhausted.

## send() and recv() Buffer Semantics

A critical misunderstanding: `send()` and `recv()` do NOT send or receive complete messages. They transfer bytes to/from kernel buffers.

```c
// WRONG — assumes all data arrives in one recv()
char buf[1024];
recv(fd, buf, 1024, 0);
process(buf);  // buf might contain 100 bytes, not 1024

// CORRECT — loop until you have enough data
int total = 0;
while (total < expected_len) {
    int n = recv(fd, buf + total, expected_len - total, 0);
    if (n <= 0) break;  // error or connection closed
    total += n;
}
```

Similarly, `send()` may not send all your data at once:

```c
// send() might only accept part of your data
int sent = send(fd, big_buffer, 1000000, 0);
// sent might be 65536 — the rest is still in your buffer
```

**Why**: `send()` copies data into the kernel's socket send buffer. If the buffer is full (TCP flow control, slow receiver), `send()` either blocks (blocking mode) or returns a partial count (non-blocking mode). `recv()` returns whatever is currently available in the receive buffer — which depends on TCP segment boundaries, congestion, and timing.

## TCP vs UDP from the OS Perspective

| Aspect | TCP (SOCK_STREAM) | UDP (SOCK_DGRAM) |
|--------|-------------------|-------------------|
| Connection state | Kernel maintains full state machine per connection | No connection state in kernel |
| Kernel memory per "connection" | ~400 bytes for socket + buffers | Minimal — just the socket |
| Ordering | Kernel reorders packets using sequence numbers | No reordering — datagrams delivered as received |
| Buffering | Kernel manages send/receive buffers with flow control | Kernel buffers datagrams, drops if buffer full |
| Partial delivery | Yes — recv() returns arbitrary byte counts | No — each recvfrom() returns one complete datagram |
| Backpressure | TCP flow control slows the sender | None — sender can overrun receiver |
| Syscalls for setup | socket, bind, listen, accept, connect | socket, bind |
| Kernel CPU cost | Higher (checksums, state machine, timers, retransmit) | Lower (stateless, minimal processing) |

## Real-World Connection

**Nginx and SO_REUSEPORT**: Before SO_REUSEPORT, nginx used a single listening socket and worker processes competed to `accept()` connections from it — requiring a mutex (the "thundering herd" problem on older kernels). With SO_REUSEPORT, each worker has its own socket. The kernel distributes connections, eliminating contention. Latency under load dropped measurably.

**Redis and TCP_NODELAY**: Redis commands are tiny (often under 100 bytes). With Nagle's algorithm enabled, the kernel would buffer these tiny writes, waiting for more data or an ACK. For a key-value store where microsecond latency matters, this is unacceptable. Redis sets TCP_NODELAY on every client connection.

**Accept queue overflow in production**: If your application cannot call `accept()` fast enough (CPU-bound, slow initialization), the accept queue fills up. New connections get a RST or are silently dropped. Monitor with: `ss -ltn` — the `Send-Q` column on LISTEN sockets shows the accept queue limit, `Recv-Q` shows the current queue depth.

**Ephemeral port exhaustion**: A client making many short-lived connections (e.g., an HTTP client calling a microservice) uses one ephemeral port per connection. Linux defaults to ports 32768-60999, giving ~28,000 ports. With 60-second TIME_WAIT, you can exhaust these at just ~467 connections/second. Solutions: connection pooling, `net.ipv4.tcp_tw_reuse`, or expanding the ephemeral range.

## Interview Angle

**Q: Explain the difference between the SYN queue and the accept queue.**

A: When a SYN arrives, the kernel creates an entry in the SYN queue (half-open connection in SYN_RECV state). When the three-way handshake completes (ACK received), the entry moves to the accept queue (fully established). The application's `accept()` call pulls from the accept queue. The SYN queue is vulnerable to SYN flood attacks — the attacker fills it with spoofed SYNs. SYN cookies defend against this by encoding state in the sequence number instead of allocating queue entries.

**Q: Why do you need to call recv() in a loop?**

A: TCP is a byte stream, not a message protocol. When data arrives, the kernel places it in the socket receive buffer. `recv()` returns whatever bytes are currently available, which may be less than what you requested — due to TCP segmentation, network timing, or the send side writing in chunks. You must loop, accumulating bytes, until you have a complete application-level message. The kernel makes no guarantees about message boundaries.

**Q: When would you use SO_REUSEPORT?**

A: When running a multi-process server (like nginx) where each process should have its own listening socket on the same port. Without it, you either need a single accept loop that distributes connections (bottleneck), or workers competing for a shared socket (lock contention). With SO_REUSEPORT, the kernel hashes incoming connections and distributes them across sockets, giving each worker its own accept queue. This improves throughput and reduces tail latency.

---

Next: [Network Namespaces and Virtual Networking](03-network-namespaces-and-virtual-networking.md)
