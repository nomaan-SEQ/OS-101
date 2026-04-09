# I/O Systems

I/O systems manage everything that isn't CPU or memory -- disks, keyboards, network cards, GPUs, USB devices, sensors, and more. The OS must provide a uniform interface to wildly different hardware, handle data transfer efficiently, and keep the CPU productive while slow devices do their work. This is where the rubber meets the road: most real applications spend more time waiting on I/O than computing.

## Why This Matters

I/O is the bottleneck in most real applications. Your database is slow? Probably disk I/O. Your web server can't handle load? Probably network I/O. Your ML training is stalling? Probably GPU data transfer. Understanding I/O models -- blocking vs async, polling vs interrupts, select vs epoll -- is what separates engineers who build systems that handle 100 connections from those who build systems that handle 100,000.

Every performance optimization conversation eventually comes back to I/O: how data moves between devices and memory, how the CPU stays productive during transfers, and how applications manage thousands of concurrent I/O operations without drowning in threads.

## Prerequisites

| Prerequisite | Why You Need It |
|---|---|
| [08 - Storage Management](../08-Storage-Management/README.md) | Disk scheduling, file system layout -- the storage side of I/O |
| [01 - System Calls and Kernel](../01-System-Calls-and-Kernel/README.md) | Interrupts, kernel mode, device drivers live in kernel space |

## Reading Order

| # | Topic | File | What You'll Learn |
|---|-------|------|-------------------|
| 1 | I/O Hardware Overview | [01-io-hardware-overview.md](01-io-hardware-overview.md) | Buses, controllers, port-mapped vs memory-mapped I/O, device registers |
| 2 | Polling, Interrupts, and DMA | [02-io-techniques-polling-interrupts-dma.md](02-io-techniques-polling-interrupts-dma.md) | Three fundamental ways the CPU communicates with devices |
| 3 | Device Drivers | [03-device-drivers.md](03-device-drivers.md) | How the OS talks to specific hardware, driver interfaces, why drivers cause bugs |
| 4 | Blocking, Non-blocking, and Async I/O | [04-blocking-nonblocking-async-io.md](04-blocking-nonblocking-async-io.md) | I/O models from the application's perspective, the C10K problem |
| 5 | I/O Multiplexing: select, poll, epoll | [05-io-multiplexing-select-poll-epoll.md](05-io-multiplexing-select-poll-epoll.md) | Monitoring many connections efficiently, the reactor pattern |

## Key Interview Questions

1. **What's the difference between polling, interrupts, and DMA? When would you use each?**
   Polling: CPU loops checking device status (low latency, wastes CPU). Interrupts: device signals CPU when ready (good CPU utilization, some overhead). DMA: device transfers data to memory without CPU involvement (essential for bulk transfers). Use polling for ultra-low-latency paths, interrupts for moderate-speed devices, DMA for disk/network bulk data.

2. **Explain the difference between blocking, non-blocking, and asynchronous I/O.**
   Blocking: thread sleeps until I/O completes. Non-blocking: call returns immediately with EAGAIN if not ready, app must retry. Async: submit request, get notified on completion. Blocking is simplest but doesn't scale. Async gives best throughput for many concurrent operations.

3. **Why is epoll better than select for handling many connections?**
   select scans ALL file descriptors every call -- O(n). epoll registers FDs once and only returns ready ones -- O(1) for ready events. select also has a 1024 FD limit. epoll is what powers nginx, Redis, and Node.js on Linux.

4. **What is DMA and why is it critical for modern I/O?**
   DMA lets devices transfer data directly to/from memory without CPU involvement. Without DMA, the CPU would have to copy every byte from device to memory, wasting cycles on data shuffling instead of computation. Every disk read, network packet, and GPU transfer uses DMA.

5. **Why are device drivers the #1 source of kernel crashes?**
   Drivers run in kernel space with full hardware access but are often written by hardware vendors (not kernel developers). A bug in a driver can corrupt kernel memory, cause deadlocks, or panic the system. This is why microkernels move drivers to user space and why Linux has strict driver review processes.
