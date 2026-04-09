# Network Stack in the Kernel

The kernel's network stack is a layered pipeline that transforms raw bytes on a wire into structured data an application can read — and vice versa. Every network operation you perform, from a simple `curl` to a high-throughput database replication stream, passes through this pipeline. Understanding its structure tells you where latency comes from, where packets can be filtered, and why certain optimizations exist.

## The Layers, Mapped to Reality

The kernel network stack maps roughly to the TCP/IP model (and loosely to OSI). But unlike textbook diagrams, the kernel implementation has very concrete boundaries:

```
┌─────────────────────────────────────────┐
│         APPLICATION (User Space)         │
│   Your code calls send(), recv(), etc.   │
│   Interacts via socket file descriptors  │
├─────────────────────────────────────────┤  ← syscall boundary
│         SOCKET LAYER (Kernel)            │
│   Socket buffers (send/receive queues)   │
│   Protocol dispatch                      │
├─────────────────────────────────────────┤
│       TRANSPORT LAYER (Kernel)           │
│   TCP: segmentation, flow control,       │
│        congestion control, retransmit    │
│   UDP: simple datagram dispatch          │
├─────────────────────────────────────────┤
│        NETWORK LAYER (Kernel)            │
│   IP: routing decisions, fragmentation,  │
│       TTL decrement, header checksum     │
├─────────────────────────────────────────┤
│     LINK LAYER (Kernel + Driver)         │
│   Ethernet framing, MAC addresses,       │
│   ARP resolution, VLAN tagging           │
├─────────────────────────────────────────┤
│     NIC DRIVER + HARDWARE                │
│   DMA transfers, ring buffers,           │
│   hardware interrupts, offloading        │
└─────────────────────────────────────────┘
```

**Key insight**: the kernel owns layers 2 through 4. User space only touches the socket API. This is why kernel bypass techniques (covered in file 04) are such a big deal — they skip all of this.

## Path of an Incoming Packet

This is one of the most important flows to understand. Walk through it layer by layer:

```
 Wire
  │
  ▼
┌──────────┐
│   NIC    │  1. NIC receives Ethernet frame
│          │  2. DMA copies frame to ring buffer in kernel memory
│          │  3. NIC raises HARDWARE INTERRUPT
└────┬─────┘
     │
     ▼
┌──────────┐
│  Driver  │  4. Interrupt handler runs (top half — very fast)
│          │  5. Schedules NAPI poll (softirq NET_RX_SOFTIRQ)
│          │  6. Driver pulls packets from ring buffer
└────┬─────┘
     │
     ▼
┌──────────┐
│ IP Layer │  7. Validate IP header, checksum
│          │  8. Routing decision: local delivery or forward?
│          │  9. Reassemble fragments if needed
└────┬─────┘
     │
     ▼
┌──────────┐
│TCP Layer │  10. Match to connection (4-tuple lookup)
│          │  11. Validate sequence numbers, checksums
│          │  12. Process TCP state machine
│          │  13. Place data in socket receive buffer
└────┬─────┘
     │
     ▼
┌──────────┐
│  Socket  │  14. Wake up process blocked on recv()/epoll()
│  Buffer  │  15. Application reads data from socket buffer
└──────────┘
```

**Steps 1-3: Hardware**. The NIC uses DMA (Direct Memory Access) to copy frames directly into a ring buffer in kernel memory — no CPU involvement for the copy itself. Then the NIC fires a hardware interrupt.

**Steps 4-6: Driver + NAPI**. The interrupt handler is minimal -- it disables further interrupts from the NIC and schedules a softirq. **NAPI (New API)** is a hybrid interrupt/polling mechanism for network drivers. When a packet arrives, the NIC triggers an interrupt. But instead of firing an interrupt for every subsequent packet (which would overwhelm the CPU at high rates), NAPI switches to polling mode — the kernel periodically checks for new packets in a batch. Once the burst subsides, it switches back to interrupt mode. This adaptive approach prevents interrupt storms while maintaining low latency during light traffic.

**Steps 7-9: IP layer**. Routing, fragmentation reassembly, and the first Netfilter hook points (PREROUTING).

**Steps 10-13: TCP layer**. The TCP state machine processes the packet — handling SYN, ACK, FIN, data, window updates, etc.

**Steps 14-15: Socket buffer to application**. Data sits in the socket receive buffer until the application calls `recv()` or is notified via `epoll`.

## Path of an Outgoing Packet

The reverse direction:

```
Application calls send()
     │
     ▼
┌──────────┐
│  Socket  │  1. Data copied from user buffer to socket send buffer
│  Buffer  │  2. TCP decides when to send (Nagle, congestion window)
└────┬─────┘
     │
     ▼
┌──────────┐
│TCP Layer │  3. Create TCP segment (headers, sequence numbers)
│          │  4. Calculate checksum (or offload to NIC)
└────┬─────┘
     │
     ▼
┌──────────┐
│ IP Layer │  5. Route lookup: which interface? which next hop?
│          │  6. Add IP header, fragment if MTU exceeded
└────┬─────┘
     │
     ▼
┌──────────┐
│  Driver  │  7. Add Ethernet header (ARP for MAC address)
│          │  8. Place in NIC transmit ring buffer via DMA
└────┬─────┘
     │
     ▼
┌──────────┐
│   NIC    │  9. NIC reads from ring buffer, sends on wire
│          │  10. TX completion interrupt → free sk_buff
└──────────┘
```

Notice step 2: `send()` returning does NOT mean data reached the network. It means data was copied to the kernel socket buffer. TCP decides when to actually transmit based on Nagle's algorithm, the congestion window, and the receive window.

> **Analogy for Nagle's algorithm:** Nagle's algorithm is like a postal worker who waits at your mailbox -- instead of making a trip to the post office for every single letter, they collect letters for a short time and deliver them all in one batch. This is efficient for bulk mail, but terrible when you need an urgent reply and the postal worker is sitting on your letter waiting for more.

## sk_buff: The Kernel's Packet Data Structure

Every packet flowing through the kernel is represented by a `struct sk_buff` (socket buffer). This is one of the most important data structures in the kernel:

```
struct sk_buff {
    // Pointers to different protocol headers
    unsigned char *head;      // Start of allocated buffer
    unsigned char *data;      // Start of current data
    unsigned char *tail;      // End of current data
    unsigned char *end;       // End of allocated buffer

    // Protocol headers (set as packet traverses layers)
    struct ethhdr   *mac_header;
    struct iphdr    *network_header;
    struct tcphdr   *transport_header;

    struct sock      *sk;      // Associated socket
    struct net_device *dev;    // Network device
    // ... many more fields
};
```

**Why this design matters**: as a packet moves up the stack, the kernel does not copy data between layers. Instead, it adjusts pointers within the same `sk_buff`. The `data` pointer is moved forward as each header is consumed. This is what makes the stack efficient — zero copies between layers.

```
Incoming packet processing:

  head ──► ┌──────────────┐
           │  Ethernet hdr │ ◄── data (at link layer)
           ├──────────────┤
           │   IP header   │ ◄── data (after link processing)
           ├──────────────┤
           │  TCP header   │ ◄── data (after IP processing)
           ├──────────────┤
           │   Payload     │ ◄── data (at socket layer)
  tail ──► └──────────────┘
```

## Netfilter: Packet Filtering Hooks

Netfilter is the kernel framework that lets you inspect and manipulate packets at specific hook points. `iptables` and `nftables` are the user-space tools that configure Netfilter rules.

```
                    Incoming packet
                          │
                          ▼
                   ┌──────────────┐
                   │  PREROUTING  │  ← DNAT happens here
                   └──────┬───────┘
                          │
                    Routing decision
                    /             \
                   ▼               ▼
            ┌───────────┐   ┌───────────┐
            │   INPUT   │   │  FORWARD  │  ← Packets for other hosts
            └─────┬─────┘   └─────┬─────┘
                  │               │
                  ▼               ▼
           Local process    ┌──────────────┐
                  │         │ POSTROUTING  │  ← SNAT/MASQUERADE
                  ▼         └──────┬───────┘
            ┌───────────┐         │
            │  OUTPUT   │         ▼
            └─────┬─────┘     Outgoing
                  │
                  ▼
           ┌──────────────┐
           │ POSTROUTING  │
           └──────┬───────┘
                  │
                  ▼
              Outgoing
```

**The five hooks**:
- **PREROUTING**: First hook for all incoming packets, before routing. DNAT (destination NAT) happens here.
- **INPUT**: For packets destined to the local machine.
- **FORWARD**: For packets being routed through this machine to another destination.
- **OUTPUT**: For locally-generated outgoing packets.
- **POSTROUTING**: Last hook before packets leave. SNAT (source NAT) and MASQUERADE happen here.

## conntrack: Connection Tracking

`conntrack` is the kernel module that tracks the state of network connections. It enables stateful firewalls and NAT.

For every connection, conntrack maintains an entry:

| Field | Example |
|-------|---------|
| Protocol | TCP |
| Source IP:Port | 192.168.1.10:45234 |
| Dest IP:Port | 10.0.0.5:443 |
| State | ESTABLISHED |
| Timeout | 432000 (5 days for ESTABLISHED TCP) |

**Why conntrack matters**:
- **Stateful firewall**: Allow return traffic for established connections without explicit rules
- **NAT**: Track the mapping so return packets can be translated back
- **Docker networking**: Every container-to-host and container-to-external connection goes through conntrack + NAT

**conntrack table exhaustion** is a real production issue. The default table size (`nf_conntrack_max`) is often 65536. A busy server with many short-lived connections can fill this table, causing new connections to be dropped. Symptoms: random connection failures, "nf_conntrack: table full, dropping packet" in kernel logs.

## TCP Connection States from the OS Perspective

The kernel maintains a state machine for every TCP connection:

```
Client                                Server

socket()                              socket(), bind(), listen()
connect() ──── SYN ──────────────►    (SYN received)
[SYN_SENT]                            [SYN_RECV]
              ◄──── SYN+ACK ────
[ESTABLISHED]                         
         ──── ACK ──────────────►     [ESTABLISHED]

         ... data transfer ...

close() ────── FIN ─────────────►     (FIN received)
[FIN_WAIT_1]                          [CLOSE_WAIT]
              ◄──── ACK ────          
[FIN_WAIT_2]                          close()
              ◄──── FIN ────          [LAST_ACK]
[TIME_WAIT]                           
         ──── ACK ──────────────►     [CLOSED]
              
(waits 2*MSL)
[CLOSED]
```

**The TIME_WAIT problem**: After closing a connection, the initiating side enters TIME_WAIT for 2 * MSL (Maximum Segment Lifetime — typically 60 seconds on Linux). During this time, the (source IP, source port, dest IP, dest port) tuple is occupied.

Why this matters for production:
- A server handling thousands of short-lived connections per second can exhaust ephemeral ports
- Each TIME_WAIT socket consumes ~400 bytes of kernel memory
- Check with: `ss -s` or `netstat -an | grep TIME_WAIT | wc -l`

Mitigations:
- `net.ipv4.tcp_tw_reuse = 1`: Allow reusing TIME_WAIT sockets for new outgoing connections (safe with timestamps)
- `SO_REUSEADDR`: Allow binding to an address that has TIME_WAIT connections
- Connection pooling: Reuse connections instead of creating new ones

## Real-World Connection

**Docker networking and iptables**: When a Docker container exposes a port, Docker creates iptables rules in the PREROUTING and POSTROUTING chains. Traffic to the host's published port is DNAT'd to the container's IP. Return traffic is MASQUERADE'd. This entire flow passes through conntrack. On hosts running hundreds of containers with frequent connections, conntrack table exhaustion is a common failure mode.

**TIME_WAIT in microservices**: Service meshes and microservice architectures create many short-lived HTTP connections. A service making 10,000 requests/second with connection-per-request can accumulate 600,000 TIME_WAIT sockets (10,000 * 60 seconds). This is why connection pooling and HTTP keep-alive are critical — they are not just nice-to-haves, they prevent port exhaustion.

**NAPI and high-throughput NICs**: Modern 10/25/100 Gbps NICs can deliver millions of packets per second. Without NAPI's polling model, each packet would trigger an interrupt, and the CPU would spend all its time in interrupt handlers (livelock). NAPI batches packet processing, and techniques like RSS (Receive Side Scaling) distribute packets across multiple CPU cores.

## Interview Angle

**Q: Walk me through what happens when a packet arrives at a Linux server.**

A: The NIC receives the frame and uses DMA to copy it into a ring buffer in kernel memory. The NIC raises a hardware interrupt. The driver's interrupt handler (top half) runs briefly — it disables further interrupts and schedules a softirq (NET_RX_SOFTIRQ). The softirq handler (NAPI poll) processes packets in batches from the ring buffer. Each packet is wrapped in an `sk_buff` structure and passed up: first to the IP layer for routing and header validation, then to the TCP layer where it is matched to a connection via 4-tuple lookup and processed through the TCP state machine. The payload is placed into the socket's receive buffer, and any process waiting on that socket (via `recv()` or `epoll`) is woken up.

**Q: What is conntrack and why does it cause problems at scale?**

A: conntrack is the kernel's connection tracking module. It maintains a hash table of all active connections, enabling stateful firewalls and NAT. The problem is the table has a fixed maximum size (default ~65K entries). In environments with many short-lived connections — like Docker hosts or load balancers — the table fills up and new connections are silently dropped. The fix is to increase `nf_conntrack_max`, tune timeouts, or use eBPF-based solutions like Cilium that bypass conntrack entirely.

**Q: Why does the kernel use sk_buff instead of copying data between layers?**

A: Efficiency. Rather than copying packet data as it moves through protocol layers, the kernel adjusts pointers within a single `sk_buff` structure. As each layer consumes its header, the `data` pointer advances forward, revealing the next layer's header — all without moving any bytes. This zero-copy-between-layers approach is essential for high-throughput packet processing.

---

Next: [Sockets API — TCP and UDP](02-sockets-api-tcp-udp.md)
