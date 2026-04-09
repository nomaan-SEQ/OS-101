# I/O Multiplexing: select, poll, epoll

You have 10,000 open network connections. Data could arrive on any of them at any time. You can't afford a thread per connection (memory, context switching). You can't poll each one in a loop (CPU waste). You need the OS to tell you **which connections have data ready** -- and only wake you up for those. This is I/O multiplexing.

## The Problem

```
Without multiplexing (thread-per-connection):

Thread 1:  |---blocked on read(fd1)---|
Thread 2:  |---blocked on read(fd2)---|
Thread 3:  |---blocked on read(fd3)---|
...
Thread 10000: |---blocked on read(fd10000)---|

10,000 threads. 10 GB of stack memory. Most are sleeping.

With multiplexing (one thread):

Thread 1:  |--wait for ANY ready fd--|handle fd7|handle fd42|handle fd1001|--wait--|

One thread. Handles whichever connections have data. Efficient.
```

## select()

The oldest multiplexing call. POSIX standard, available everywhere.

```c
fd_set read_fds;
FD_ZERO(&read_fds);
FD_SET(fd1, &read_fds);  // monitor fd1
FD_SET(fd2, &read_fds);  // monitor fd2
FD_SET(fd3, &read_fds);  // monitor fd3

struct timeval timeout = {5, 0};  // 5 second timeout

// Block until at least one FD is ready (or timeout)
int ready = select(max_fd + 1, &read_fds, NULL, NULL, &timeout);

// Check which FDs are ready
if (FD_ISSET(fd1, &read_fds)) { /* fd1 has data */ }
if (FD_ISSET(fd2, &read_fds)) { /* fd2 has data */ }
if (FD_ISSET(fd3, &read_fds)) { /* fd3 has data */ }
```

### How select Works Internally

```
User Space                    Kernel Space
+------------------+          +------------------+
| fd_set: bitmask  |  copy    | Scan ALL FDs     |
| [1,1,0,1,0,0,1] | ------> | in the bitmask   |
+------------------+          | fd1: not ready   |
                              | fd2: READY       |
+------------------+  copy    | fd3: not ready   |
| fd_set: result   | <------ | fd4: READY       |
| [0,1,0,1,0,0,0] |          +------------------+
+------------------+
Must rebuild fd_set EVERY call (kernel overwrites it)
```

### select Limitations

| Limitation | Impact |
|-----------|--------|
| **FD_SETSIZE = 1024** | Can't monitor more than 1024 file descriptors |
| **O(n) scan every call** | Kernel scans ALL FDs, even if only 1 is ready |
| **fd_set rebuilt every call** | Must copy the entire bitmask to kernel and back |
| **Modifies the fd_set** | Must reinitialize before every call |

select works fine for < 100 FDs. At thousands, performance collapses.

## poll()

Fixes select's FD limit but keeps the same scanning approach.

```c
struct pollfd fds[3];
fds[0] = (struct pollfd){.fd = fd1, .events = POLLIN};
fds[1] = (struct pollfd){.fd = fd2, .events = POLLIN};
fds[2] = (struct pollfd){.fd = fd3, .events = POLLIN};

int ready = poll(fds, 3, 5000);  // 5 second timeout

for (int i = 0; i < 3; i++) {
    if (fds[i].revents & POLLIN) {
        // fds[i].fd has data ready
        read(fds[i].fd, buf, sizeof(buf));
    }
}
```

### poll vs select

| Feature | select | poll |
|---------|--------|------|
| FD limit | 1024 (FD_SETSIZE) | No limit (array-based) |
| Per-call overhead | O(n) scan | O(n) scan |
| Modifies input? | Yes (overwrites fd_set) | No (separate revents field) |
| Portability | POSIX, everywhere | POSIX, everywhere |

poll is better than select (no FD limit, cleaner API), but still **O(n) per call** -- the kernel scans every FD in the array even if only one is ready. At 10,000 FDs, this matters.

## epoll() -- Linux

The breakthrough. **Register once, get notified about ready FDs only.** No scanning.

> **Analogy:** select/poll is like a teacher **calling roll for every student** to see who has a question -- slow when the class has 10,000 students. epoll is like a **notification system**: students press a button when they have a question, and only those names appear on the teacher's screen. The teacher never wastes time asking students who have nothing to say.

```c
// 1. Create epoll instance
int epfd = epoll_create1(0);

// 2. Register file descriptors (do this ONCE per FD)
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = client_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);

// 3. Wait for events (returns ONLY ready FDs)
struct epoll_event events[MAX_EVENTS];
int nready = epoll_wait(epfd, events, MAX_EVENTS, timeout_ms);

// 4. Process only the ready FDs
for (int i = 0; i < nready; i++) {
    int fd = events[i].data.fd;
    read(fd, buf, sizeof(buf));
    process(buf);
}
```

### How epoll Works Internally

```
User Space                         Kernel Space
                                   +-------------------+
epoll_ctl(ADD, fd7) ------------>  | Interest List     |
epoll_ctl(ADD, fd42) ----------->  | [fd7, fd42,       |
epoll_ctl(ADD, fd1001) -------->   |  fd1001, ...]     |
  (one-time registration)          +-------------------+
                                          |
                                   When data arrives on fd42:
                                   Device interrupt -> kernel
                                   Kernel callback adds fd42
                                   to the ready list
                                          |
                                   +-------------------+
epoll_wait() ---- returns ------   | Ready List        |
  only ready FDs!                  | [fd42]            |
  O(ready), not O(total)           +-------------------+
```

Key difference: the kernel maintains a **ready list**. When a device signals data is available (via interrupt/callback), the kernel adds that FD to the ready list. `epoll_wait()` just returns whatever is on the ready list -- it doesn't scan anything.

### Level-Triggered vs Edge-Triggered

| Mode | Behavior | Analogy |
|------|----------|---------|
| **Level-triggered** (default) | epoll_wait returns FD as long as data is available | Alarm stays on while door is open |
| **Edge-triggered** (EPOLLET) | epoll_wait returns FD only when NEW data arrives | Alarm fires only when door opens |

```c
// Edge-triggered mode
ev.events = EPOLLIN | EPOLLET;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

// With edge-triggered, you MUST read ALL available data:
while (true) {
    n = read(fd, buf, sizeof(buf));
    if (n == -1 && errno == EAGAIN) break;  // no more data
    process(buf, n);
}
// If you don't drain the buffer, you won't get notified again!
```

Edge-triggered is more efficient (fewer epoll_wait returns) but harder to use correctly. Most production systems use level-triggered for safety.

## kqueue -- BSD/macOS

BSD's answer to epoll. Conceptually similar, unified event system.

```c
int kq = kqueue();

struct kevent change;
EV_SET(&change, fd, EVFILT_READ, EV_ADD, 0, 0, NULL);

struct kevent event;
int nev = kevent(kq, &change, 1, &event, 1, NULL);
```

kqueue is arguably more elegant than epoll because it handles **multiple event types** in one interface: file I/O, sockets, signals, process events, timers, filesystem changes. epoll only handles FD readiness.

## IOCP -- Windows

Windows I/O Completion Ports take a fundamentally different approach:

```
epoll (readiness model):        IOCP (completion model):
"fd42 is READY to read"        "Read on fd42 is DONE, here's the data"
 You still call read()          Data is already in your buffer
```

| Aspect | epoll (readiness) | IOCP (completion) |
|--------|-------------------|-------------------|
| Notification | "Data is available" | "I/O is finished" |
| Data transfer | App calls read() after notification | Kernel already transferred data |
| Threading | App manages thread pool | OS manages thread pool |
| Concept | "This FD is ready" | "This operation completed" |

IOCP is generally considered a better model because the OS handles more of the complexity. It's what makes Windows a strong server platform despite perceptions.

## io_uring -- Linux 5.1+ (2019)

The newest and most powerful Linux I/O interface. Combines the best ideas from IOCP with Linux-specific optimizations.

```
Architecture:

User Space                    Kernel Space
+---------------------+      +---------------------+
| Submission Queue    |      | Submission Queue    |
| Ring (SQR)          |----->| Processing          |
| [read fd=7, 4KB]   | shared| [execute I/O ops]   |
| [write fd=42, 1KB]  | memory|                    |
| [readv fd=99, 8KB]  |      |                     |
+---------------------+      +---------------------+
                                     |
+---------------------+      +---------------------+
| Completion Queue    |      | Completion Queue    |
| Ring (CQR)          |<-----| Results             |
| [fd=7: 4KB done]   | shared| [completed events]  |
| [fd=42: 1KB done]   | memory|                    |
+---------------------+      +---------------------+

Shared memory = no copying between user and kernel
Ring buffers = no syscalls needed in best case (SQPOLL mode)
```

### io_uring Key Features

| Feature | Benefit |
|---------|---------|
| Shared ring buffers | Zero-copy submission/completion |
| Batched submission | One syscall submits many I/O requests |
| SQPOLL mode | Kernel thread polls SQ -- zero syscalls |
| Fixed buffers/files | Register once, reference by index -- less overhead |
| Linked operations | Chain operations (read then write) |
| Unified interface | Files, network, timers, all through one API |

```c
// io_uring example (simplified)
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);

// Submit a read
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, 4096, 0);
sqe->user_data = 42;  // tag for identifying completion
io_uring_submit(&ring);

// Wait for completion
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
// cqe->res = bytes read, cqe->user_data = 42
io_uring_cqe_seen(&ring, cqe);
```

io_uring is the future of Linux I/O. It's being adopted by databases (RocksDB), web servers, and runtimes.

## Evolution of Linux I/O Multiplexing

```
1983         1986         2002         2019
select()  -> poll()   -> epoll()   -> io_uring
  |            |            |            |
  |            |            |            +-- Async submission/completion
  |            |            |                Zero-copy, batched, universal
  |            |            |
  |            |            +-- O(1) event notification
  |            |                Register once, ready list
  |            |
  |            +-- No FD limit
  |                 Clean API (pollfd array)
  |
  +-- 1024 FD limit, O(n) scan, modifies input
```

## Comparison Table

| Feature | select | poll | epoll | kqueue | io_uring |
|---------|--------|------|-------|--------|----------|
| FD limit | 1024 | None | None | None | None |
| Complexity per call | O(n) | O(n) | O(ready) | O(ready) | O(1) amortized |
| Registration | Per-call | Per-call | One-time | One-time | One-time |
| Event types | FD readiness | FD readiness | FD readiness | FD + signals + timers | Everything |
| Syscalls per I/O | 2 (select+read) | 2 (poll+read) | 2 (wait+read) | 2 (kevent+read) | 0-1 (SQPOLL) |
| Platform | Everywhere | Everywhere | Linux | BSD/macOS | Linux 5.1+ |
| Best for | Portability | Simple servers | High-perf Linux | macOS/BSD servers | Maximum performance |

## The Reactor Pattern

Almost every high-performance server uses the same architecture, called the **reactor pattern**:

```
+------------------+
| Event Loop       |    (single thread or small thread pool)
| (epoll_wait)     |
+--------+---------+
         |
   +-----+-----+
   |     |     |
   v     v     v
 fd7   fd42  fd1001    (ready file descriptors)
   |     |     |
   v     v     v
handler handler handler  (registered callbacks)
   |     |     |
   v     v     v
 read  read  read      (non-blocking operations)
 parse parse parse
 respond respond respond
```

1. Register interest in events (connections, data arrival)
2. Event loop calls `epoll_wait` / `kqueue` / `IOCP`
3. For each ready event, dispatch to the appropriate handler
4. Handler performs non-blocking I/O and processing
5. Go back to step 2

This is the core architecture of:
- **nginx**: Multi-process, each process runs an epoll event loop
- **Redis**: Single-threaded event loop using epoll
- **Node.js**: Single-threaded event loop via libuv
- **Netty (Java)**: Event loop using epoll (or NIO selector)
- **Tokio (Rust)**: Multi-threaded event loop using epoll/kqueue

## Real-World Connection

**nginx vs Apache**: Apache traditionally used thread-per-connection (blocking I/O). nginx uses an event loop with epoll. Result: nginx handles 10x more concurrent connections with less memory. Apache now has an event MPM too, but nginx's architecture was event-driven from the start.

**Redis single-thread**: Redis processes ALL commands in a single event loop thread. This works because Redis operations are in-memory (microseconds). The event loop never blocks on I/O for long. The bottleneck is network bandwidth, not CPU -- one thread can saturate a 10Gbps NIC.

**Node.js libuv**: libuv is the cross-platform I/O library under Node.js. On Linux it uses epoll, on macOS it uses kqueue, on Windows it uses IOCP. Your JavaScript code doesn't change -- libuv abstracts the platform differences.

**Database I/O**: PostgreSQL uses one process per connection (blocking I/O model). This limits it to ~thousands of connections. Connection poolers like PgBouncer sit in front, multiplexing thousands of application connections onto hundreds of PostgreSQL connections.

**Cloud load balancers**: AWS ALB, Google Cloud Load Balancer, and similar services use epoll-based event loops internally. They need to handle millions of concurrent connections efficiently -- thread-per-connection would be impossible at that scale.

**io_uring adoption**: As of 2024+, io_uring is being adopted for serious production use. Libraries like liburing simplify the API. Tokio (Rust) has experimental io_uring support. It's especially impactful for storage-heavy workloads where the reduced syscall overhead matters.

## Interview Angle

**Q: Why is epoll faster than select for 10,000 connections?**

A: Two reasons. First, select scans ALL file descriptors every call -- O(n) -- even if only one is ready. epoll maintains a kernel-side ready list and returns only the ready FDs -- O(ready). Second, select requires copying the entire FD bitmask to kernel space every call, while epoll registers FDs once and reuses the registration. At 10,000 FDs with 10 ready, select scans 10,000 entries; epoll returns 10.

**Q: What's the difference between level-triggered and edge-triggered epoll?**

A: Level-triggered (default): epoll_wait reports an FD as ready as long as there's data in the buffer. If you read 1 byte of 100 available, the next epoll_wait will report it again. Edge-triggered: epoll_wait reports an FD only when its state changes (new data arrives). You must read ALL available data or you won't be notified again. Edge-triggered is more efficient but requires careful coding -- you must drain the buffer in a loop.

**Q: Explain the reactor pattern.**

A: The reactor pattern is an event-driven architecture where a single event loop monitors many I/O sources using multiplexing (epoll/kqueue/IOCP). When an event occurs (data ready, connection arrived), the reactor dispatches to the appropriate handler. The handler performs non-blocking I/O, processes the data, and returns control to the event loop. nginx, Redis, and Node.js all use this pattern. It's efficient because one thread handles thousands of connections without blocking.

**Q: How does io_uring improve on epoll?**

A: io_uring uses shared ring buffers between user space and kernel space, eliminating data copying. It batches submissions (many I/O requests in one syscall) and can even run in poll mode where a kernel thread processes submissions with zero syscalls. Unlike epoll (readiness-based), io_uring is completion-based -- you submit an I/O request and get the completed result, similar to Windows IOCP. It also unifies all I/O types (files, network, timers) in one interface.

**Q: Why does Redis use a single-threaded event loop instead of multiple threads?**

A: Redis operations are in-memory and take microseconds. The event loop processes commands so fast that a single thread can handle hundreds of thousands of operations per second. Multi-threading would add locking overhead and complexity for minimal benefit -- the bottleneck is network I/O, not CPU. Redis 6+ added I/O threads for network read/write marshaling, but command execution stays single-threaded to avoid locks.
