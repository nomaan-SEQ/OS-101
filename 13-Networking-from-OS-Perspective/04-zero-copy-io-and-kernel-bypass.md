# Zero-Copy I/O and Kernel Bypass

The kernel network stack is general-purpose, correct, and full-featured. It is also slow for workloads that need maximum throughput. Every packet passes through multiple layers, each copying data, switching context, and consuming CPU cycles. For a web server sending a static file, the data is copied four times before it reaches the wire — even though the application never modifies it. For a load balancer forwarding packets, the kernel processes every protocol layer even though no local application needs the data.

Zero-copy and kernel bypass techniques exist to eliminate this waste. They range from simple syscalls (sendfile) to radical approaches that remove the kernel from the data path entirely (DPDK).

## The Problem: Traditional Network I/O

Consider serving a static file over HTTP — the most common operation on the internet:

```c
// Traditional approach
char buf[4096];
int file_fd = open("index.html", O_RDONLY);
int sock_fd = accept(listen_fd, ...);

while ((n = read(file_fd, buf, sizeof(buf))) > 0) {
    write(sock_fd, buf, n);
}
```

Here is what actually happens in the kernel:

```
                    read()                          write()
User Space:      ┌─────────┐                    ┌──────────┐
                 │ User    │                    │ User     │
                 │ Buffer  │                    │ Buffer   │
                 └────▲────┘                    └────┬─────┘
                      │ copy 2                       │ copy 3
─────────────────────────────────────────────────────────────────
Kernel Space:    ┌────┴────┐                    ┌────▼─────┐
                 │ Page    │                    │ Socket   │
                 │ Cache   │                    │ Buffer   │
                 └────▲────┘                    └────┬─────┘
                      │ copy 1                       │ copy 4
─────────────────────────────────────────────────────────────────
Hardware:        ┌────┴────┐                    ┌────▼─────┐
                 │  Disk   │                    │   NIC    │
                 └─────────┘                    └──────────┘

  4 data copies + 4 context switches (user→kernel→user→kernel)
```

**Four copies**:
1. Disk → kernel page cache (DMA)
2. Kernel page cache → user buffer (`read()` syscall)
3. User buffer → kernel socket buffer (`write()` syscall)
4. Kernel socket buffer → NIC (DMA)

**Four context switches**: user→kernel (read), kernel→user (read returns), user→kernel (write), kernel→user (write returns).

The application never modifies the data. Copies 2 and 3 exist only because the data must pass through user space. This is pure waste.

## sendfile(): Eliminate the User-Space Copy

`sendfile()` transfers data directly from one file descriptor to another inside the kernel. The data never enters user space.

```c
#include <sys/sendfile.h>

int file_fd = open("index.html", O_RDONLY);
int sock_fd = accept(listen_fd, ...);

// Send file directly to socket — no user-space buffer needed
sendfile(sock_fd, file_fd, &offset, file_size);
```

```
User Space:      (nothing — data never enters user space)

─────────────────────────────────────────────────────────
Kernel Space:    ┌──────────┐     ┌──────────┐
                 │  Page    │────►│  Socket  │
                 │  Cache   │     │  Buffer  │
                 └────▲─────┘     └────┬─────┘
                      │ copy 1         │ copy 2
─────────────────────────────────────────────────────────
Hardware:        ┌────┴─────┐     ┌────▼─────┐
                 │   Disk   │     │   NIC    │
                 └──────────┘     └──────────┘

  2 data copies + 2 context switches (one syscall: user→kernel→user)
```

With hardware that supports **scatter-gather DMA**, the NIC can read directly from the page cache, eliminating even copy 2. **Scatter-Gather DMA (SG-DMA)** allows the DMA engine to transfer data to/from multiple non-contiguous memory regions in a single operation. Instead of requiring data to be in one contiguous buffer, the NIC can gather data from several buffers (e.g., header in one location, payload in another) and send them as one packet — or scatter an incoming packet's parts to different buffers. This avoids expensive memory copies to assemble contiguous buffers. This gets us down to a single DMA copy from disk and a DMA read from memory to NIC -- truly minimal.

**Who uses sendfile()**:
- **Nginx**: Enabled by default (`sendfile on;` in config) for static file serving
- **Apache**: Uses sendfile for static content
- **Java NIO**: `FileChannel.transferTo()` calls sendfile() on Linux
- **Go**: `net.(*TCPConn).ReadFrom()` uses sendfile when sending a file

## splice() and tee(): Kernel-to-Kernel Data Movement

`splice()` moves data between two file descriptors where at least one is a pipe, without copying through user space. `tee()` duplicates data in a pipe without consuming it.

```c
// Proxy: receive from network, send to network — no user-space copy
int pipe_fds[2];
pipe(pipe_fds);

// Move data from input socket to pipe
splice(input_sock, NULL, pipe_fds[1], NULL, 65536, SPLICE_F_MOVE);

// Move data from pipe to output socket
splice(pipe_fds[0], NULL, output_sock, NULL, 65536, SPLICE_F_MOVE);
```

This is useful for proxies and data pipelines. HAProxy uses splice() when proxying TCP connections — it never needs to read the data into a user-space buffer if it is not inspecting it.

## mmap() + write(): Memory-Mapped Approach

Map the file into the process's address space, then write to the socket:

```c
void *addr = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, file_fd, 0);
write(sock_fd, addr, file_size);
munmap(addr, file_size);
```

This eliminates the explicit `read()` copy — the kernel can serve `write()` directly from the page cache (which the mmap points to). But it still requires a copy from page cache to socket buffer, and the `write()` syscall overhead.

**mmap vs sendfile**: sendfile is simpler and generally preferred for the "send a file to a socket" case. mmap is more flexible — you can inspect or transform the data before writing — but it adds TLB pressure and page fault overhead.

## io_uring: Modern Batched I/O (Brief Revisit)

We covered io_uring in section 10. For networking, it provides:

- **Batched syscalls**: Submit many send/recv operations at once, reducing syscall overhead
- **Zero-copy send**: `IORING_OP_SEND_ZC` sends data from user buffers without copying to socket buffers (Linux 6.0+)
- **Fixed buffers**: Pre-register buffers to avoid per-operation mapping overhead
- **Registered file descriptors**: Skip fd-to-file lookup per operation

io_uring does not bypass the kernel stack — packets still go through TCP/IP processing. But it dramatically reduces the syscall overhead and can eliminate user-to-kernel copies for sends.

## Kernel Bypass: Remove the Kernel Entirely

For the highest performance, remove the kernel from the data path. Let user-space code talk directly to the NIC hardware.

### DPDK (Data Plane Development Kit)

DPDK binds the NIC to a user-space driver (UIO or VFIO), removing it from kernel control entirely. The application polls the NIC directly — no interrupts, no kernel network stack, no context switches.

```
Traditional:                         DPDK:

App                                  App
 │ syscall                            │ direct memory access
 ▼                                    │ (no syscall)
┌──────────┐                          │
│  Kernel  │                          │
│  TCP/IP  │                          │
│  Stack   │                          │
│  Driver  │                          │
└────┬─────┘                          │
     │ interrupt                      │
     ▼                                ▼
┌──────────┐                    ┌──────────┐
│   NIC    │                    │   NIC    │
└──────────┘                    └──────────┘
                                (NIC bound to user-space driver)
```

**How DPDK works**:
1. NIC is unbound from the kernel driver and bound to `vfio-pci` (or `uio_pci_generic`)
2. The NIC's memory-mapped registers and DMA ring buffers are mapped into user space
3. The application polls the ring buffer in a busy loop — no interrupts
4. Packets are in user-space memory — no kernel copy
5. The application implements whatever protocol processing it needs

**Performance**: DPDK applications routinely handle 10-100 million packets per second on commodity hardware. The kernel stack typically handles 1-5 million pps at best.

**Trade-offs**:
- Dedicates CPU cores to polling (100% CPU usage even when idle)
- No kernel protocol stack — you implement TCP yourself (or use a user-space stack like mTCP, Seastar)
- NIC is invisible to kernel tools (`ip`, `ifconfig`, `tcpdump`)
- Application complexity is much higher

**Who uses DPDK**: high-frequency trading systems, telecom packet processors, software-defined networking (Open vSwitch with DPDK datapath), custom load balancers.

### XDP (eXpress Data Path)

XDP takes a different approach: instead of bypassing the kernel, it runs eBPF programs at the earliest possible point IN the kernel — at the NIC driver level, before any stack processing.

```
                    Packet arrives at NIC
                            │
                            ▼
                    ┌───────────────┐
                    │   XDP/eBPF    │  ← Runs HERE, before sk_buff
                    │   Program     │     allocation, before any
                    │               │     stack processing
                    │  Decisions:   │
                    │  XDP_DROP     │──► Packet dropped (DDoS defense)
                    │  XDP_TX       │──► Send back out same NIC (reflection)
                    │  XDP_REDIRECT │──► Send to different NIC/CPU/socket
                    │  XDP_PASS     │──► Continue to normal kernel stack
                    └───────┬───────┘
                            │ XDP_PASS
                            ▼
                    ┌───────────────┐
                    │  Normal       │
                    │  kernel       │
                    │  network      │
                    │  stack        │
                    └───────────────┘
```

**Three XDP modes**:
- **Native XDP**: eBPF runs in the NIC driver. Requires driver support. Best performance.
- **Generic XDP**: Runs after `sk_buff` allocation. Works on any NIC. Slower.
- **Offloaded XDP**: eBPF runs on the NIC hardware itself (SmartNICs). Frees the CPU entirely.

**XDP advantages over DPDK**:
- Works WITH the kernel — non-XDP traffic proceeds normally
- Does not dedicate CPU cores to polling
- Standard kernel tools still work
- Can selectively process only certain packets
- eBPF verifier guarantees program safety (no crashes, no infinite loops)

**Who uses XDP**:
- **Facebook/Meta Katran**: XDP-based L4 load balancer handling all incoming traffic
- **Cloudflare**: XDP for DDoS mitigation — drops millions of malicious packets per second at wire speed
- **Cilium**: Uses XDP for Kubernetes network policy enforcement and load balancing

### AF_XDP: User-Space Access to Pre-Filtered Packets

AF_XDP is a socket type that lets user-space applications receive packets that XDP has redirected. It combines the filtering power of XDP with user-space packet processing:

```
NIC → XDP program (filter/classify) → AF_XDP socket → User application
```

The XDP program runs at driver level, drops unwanted traffic, and redirects interesting packets to an AF_XDP socket. The user application reads from this socket using a zero-copy shared memory ring buffer.

This gives you DPDK-like performance for specific traffic flows while the rest of the kernel stack operates normally.

## Comparison Table

| Approach | Data Copies | Context Switches | Kernel Stack | Latency | Complexity | Use Case |
|----------|------------|-----------------|-------------|---------|------------|----------|
| Traditional (read+write) | 4 | 4 | Full | High | Low | General purpose |
| sendfile() | 2 (or 1 with SG-DMA) | 2 | Full | Medium | Low | Static file serving |
| splice() | 0 (pipe-based) | 2 | Full | Medium | Medium | TCP proxying |
| io_uring zero-copy | 0-1 | ~0 (batched) | Full | Medium-Low | Medium | High-connection servers |
| DPDK | 0 (user space) | 0 | None | Lowest | Very High | Packet processing, HFT |
| XDP | 0 (driver level) | 0 | Optional | Lowest | High | DDoS mitigation, LB |
| AF_XDP | 0 (shared ring) | ~0 | XDP filter only | Low | High | Selective packet processing |

## Real-World Connection

**CDN static file serving (sendfile)**: A CDN edge server's primary job is sending cached files to clients. Nginx with `sendfile on` eliminates the user-space copy, and with `tcp_nopush on` it batches headers with file data for efficient transmission. This combination means a CDN server can saturate a 10 Gbps link while using minimal CPU — the data flows from page cache to NIC with kernel assistance but without user-space involvement.

**High-frequency trading (DPDK / kernel bypass)**: In HFT, every microsecond matters. The kernel network stack adds 5-10+ microseconds of latency per packet. DPDK eliminates this. Trading firms run custom network stacks in user space, polling NICs directly, with the application processing market data and generating orders in a single tight loop. Some use kernel bypass combined with FPGA NICs for sub-microsecond latency.

**Cloud load balancers (XDP)**: Cloud providers need load balancers that handle millions of connections. Traditional approaches (HAProxy, nginx) process every packet through the full kernel stack. Facebook's Katran uses XDP to make load-balancing decisions at the driver level. The eBPF program inspects the packet header, looks up the backend in an eBPF map, rewrites the destination, and sends it out — all before a single `sk_buff` is allocated. This handles 10M+ packets per second per core.

**Cloudflare DDoS protection (XDP)**: During a DDoS attack, millions of malicious packets hit the server every second. Processing each through the full kernel stack would overwhelm the CPU. Cloudflare's XDP programs drop attack packets at the driver level — a single `XDP_DROP` return discards the packet before any memory is allocated. They can filter 10+ Gbps of attack traffic while the server continues serving legitimate requests through the normal stack.

## Interview Angle

**Q: What is the problem with traditional network I/O for file serving?**

A: Serving a file over a socket using `read()` and `write()` requires four data copies: disk to page cache (DMA), page cache to user buffer (read syscall), user buffer to socket buffer (write syscall), and socket buffer to NIC (DMA). It also requires four context switches. The application never modifies the data, so copies 2 and 3 are pure waste. `sendfile()` eliminates both by transferring data directly between file descriptors inside the kernel — the data never enters user space.

**Q: What is the difference between DPDK and XDP?**

A: DPDK completely removes the NIC from kernel control. The NIC is bound to a user-space driver, and the application polls it directly. There is no kernel involvement at all — no interrupts, no protocol stack, no syscalls. This gives maximum performance but requires the application to implement its own protocol processing, and the NIC becomes invisible to kernel tools.

XDP runs eBPF programs inside the kernel at the NIC driver level, before the normal network stack processes the packet. It can drop, redirect, or modify packets at near-wire speed, but packets that pass through continue to the normal kernel stack. XDP works with the kernel rather than replacing it, is safer (eBPF verifier), and does not dedicate CPU cores to polling.

DPDK is better for applications that need to process every packet (custom protocol stacks, HFT). XDP is better for selective processing (DDoS filtering, load balancing) where most traffic should still use the kernel stack.

**Q: When would you use sendfile() vs DPDK?**

A: sendfile() for application servers sending files to clients — it is simple, well-supported, and nginx uses it by default. DPDK for infrastructure that processes raw packets at extreme rates — load balancers, firewalls, telecom gear. sendfile() optimizes within the kernel; DPDK replaces the kernel. Most engineers will use sendfile() (indirectly via nginx/Apache). Very few will write DPDK applications — but understanding why it exists helps you reason about performance at the infrastructure level.
