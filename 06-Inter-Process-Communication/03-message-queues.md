# Message Queues

## The Concept

A message queue is a **kernel-managed linked list of messages**. Processes send discrete, typed messages to the queue and other processes receive them. Unlike pipes, message queues preserve **message boundaries** — each send/receive operation handles one complete message.

```
  Process A          Kernel-Managed Queue           Process B
  (sender)                                          (receiver)
  +--------+    +----+----+----+----+----+        +--------+
  |        |--->| M5 | M4 | M3 | M2 | M1 |------>|        |
  | send() |    +----+----+----+----+----+        | recv() |
  +--------+    Messages with types & data        +--------+

  - Messages stay in queue until consumed
  - Receiver can select by message type
  - Queue persists in kernel (survives process exit)
```

Think of it like a mailbox: the sender drops off a letter (with a type label), and the receiver picks up letters — optionally filtering by type.

## Why Message Queues Over Pipes?

| Feature | Pipes | Message Queues |
|---------|-------|---------------|
| Message boundaries | No (byte stream) | Yes (each message is discrete) |
| Direction | Unidirectional | Bidirectional possible (using message types) |
| Message types/priorities | No | Yes — receiver can select specific types |
| Persistence | Dies with processes | Persists until explicitly removed |
| Unrelated processes | FIFOs only | Yes (both System V and POSIX) |
| Multiple readers/writers | Awkward | Designed for it |

## System V Message Queues

The older API, but still widely supported and found in legacy codebases.

### Core Operations

```c
#include <sys/msg.h>

// 1. Create or open a queue
int msqid = msgget(key, IPC_CREAT | 0666);
//   key: identifier (use ftok() to generate from a file path)
//   ftok() converts a file path + project ID into a unique integer key,
//   so unrelated processes can derive the same key independently.
//   Example: key_t key = ftok("/tmp/myapp", 'M');
//   Returns: queue ID

// 2. Define a message structure
struct msgbuf {
    long mtype;       // message type (must be > 0)
    char mtext[256];  // message data (flexible size)
};

// 3. Send a message
struct msgbuf msg = {.mtype = 1, .mtext = "hello"};
msgsnd(msqid, &msg, sizeof(msg.mtext), 0);

// 4. Receive a message
struct msgbuf received;
msgrcv(msqid, &received, sizeof(received.mtext), 0, 0);
//   mtype argument to msgrcv:
//     0  = receive first message (any type)
//     >0 = receive first message of that specific type
//     <0 = receive first message with type <= |mtype|

// 5. Destroy the queue
msgctl(msqid, IPC_RMID, NULL);
```

### Message Type Selection

This is the killer feature. The `mtype` field lets receivers pick specific messages:

```
Queue contents:  [type=3] [type=1] [type=2] [type=1] [type=3]

msgrcv(id, &buf, size, 0, 0)   --> gets [type=3] (first message)
msgrcv(id, &buf, size, 1, 0)   --> gets [type=1] (first type-1)
msgrcv(id, &buf, size, 2, 0)   --> gets [type=2] (first type-2)
msgrcv(id, &buf, size, -2, 0)  --> gets [type=1] (first with type <= 2)
```

This enables patterns like:
- **Priority messages:** Lower type numbers = higher priority, use negative mtype to receive
- **Multiplexed communication:** Different types for different "channels" on the same queue
- **Request-reply:** Client sends with type=server_pid, server responds with type=client_pid

## POSIX Message Queues

The newer, cleaner API with better features.

```c
#include <mqueue.h>

// 1. Create or open a queue
struct mq_attr attr = {
    .mq_maxmsg = 10,     // max messages in queue
    .mq_msgsize = 256    // max message size in bytes
};
mqd_t mq = mq_open("/myqueue", O_CREAT | O_RDWR, 0666, &attr);
//   Name must start with /

// 2. Send a message (with priority)
mq_send(mq, "hello", 5, priority);
//   Higher priority = delivered first

// 3. Receive a message (highest priority first)
char buf[256];
unsigned int prio;
mq_receive(mq, buf, 256, &prio);

// 4. Notification: get alerted when a message arrives
mq_notify(mq, &notification);  // signal or thread callback

// 5. Clean up
mq_close(mq);
mq_unlink("/myqueue");
```

## System V vs POSIX Message Queues

| Feature | System V | POSIX |
|---------|---------|-------|
| **Naming** | Integer key (ftok) | String name (starts with /) |
| **Priority mechanism** | Message types (manual selection) | Built-in priority (highest first) |
| **Notification** | None | mq_notify (signal or thread) |
| **API style** | Older, C-centric | Cleaner, file-descriptor-like |
| **Size configuration** | System-wide defaults | Per-queue attributes |
| **Visibility** | `ipcs -q` | `/dev/mqueue/` filesystem |
| **Cleanup** | Manual (`ipcrm`) or `msgctl(IPC_RMID)` | `mq_unlink()` |
| **Portability** | Available on nearly all Unix | Not available on all systems (macOS has limited support) |

**Recommendation:** Use POSIX message queues for new code. Use System V only if you need compatibility with legacy systems or platforms without POSIX mqueue support.

## Queue Inspection

```bash
# System V: list all message queues
ipcs -q

# System V: remove a queue
ipcrm -q <msqid>

# POSIX: list queues (Linux)
ls /dev/mqueue/

# POSIX: view queue attributes
cat /dev/mqueue/myqueue
```

## Advantages and Disadvantages

**Advantages:**
- Message boundaries preserved — no parsing needed
- Type/priority selection enables flexible routing
- Persist in kernel — processes can restart and reconnect
- Multiple readers/writers without special handling
- Asynchronous — sender doesn't need to wait for receiver

**Disadvantages:**
- Kernel overhead for every send/receive (data copied to/from kernel)
- Size limits: individual messages and total queue size are bounded
- More complex API than pipes
- Not as flexible as sockets (no built-in connection management)
- System V queues can leak if not properly cleaned up

## Real-World Connection

- **Conceptual ancestor of modern message brokers.** Kafka, RabbitMQ, SQS, and Redis Streams all implement the same core idea (message queues) but over the network with persistence, replication, and consumer groups.
- **systemd** uses message-based IPC (D-Bus) for service communication on Linux.
- **POSIX message queues** are used in real-time embedded systems where bounded latency matters.
- **Android's Binder** (the dominant Android IPC) is conceptually a message-passing system, though it uses shared memory for the actual data transfer.
- The evolution from kernel message queues to network message brokers mirrors the shift from single-machine to distributed systems.

## Interview Angle

**Q: What are the advantages of message queues over pipes?**

A: Three main advantages. First, message boundaries are preserved — each send/receive is one complete message, so no parsing is needed. Second, message types or priorities allow selective receiving, enabling patterns like priority processing or multiplexed communication on a single queue. Third, queues persist in the kernel independent of any process, so a sender can enqueue messages before the receiver starts. Pipes have none of these properties — they're unstructured byte streams.

**Q: How do message types enable bidirectional communication on a single queue?**

A: Each process uses the other's PID (or some agreed-upon ID) as the message type. A client sends a request with `mtype = server_pid`, and the server responds with `mtype = client_pid`. Each process calls `msgrcv()` with its own PID as the type filter, receiving only messages addressed to it. This allows multiple clients and a server to share one queue.

**Q: What's the relationship between OS-level message queues and message brokers like Kafka?**

A: They solve the same fundamental problem — decoupled, asynchronous message passing — but at different scales. OS message queues are local, kernel-managed, bounded in size, and limited to a single machine. Network message brokers add persistence to disk, replication across machines, consumer groups, exactly-once delivery semantics, and massive throughput. The mental model is the same; the implementation and scale are different.

---

**Next:** [Shared Memory](04-shared-memory.md) — the fastest IPC mechanism, and why it requires careful synchronization.
