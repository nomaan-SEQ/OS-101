# I/O Techniques: Polling, Interrupts, and DMA

The CPU and devices operate at vastly different speeds. A CPU executes billions of instructions per second; a hard disk takes millions of nanoseconds to return data. The fundamental question is: **how does the CPU know when a device is done?** There are exactly three answers, each with different trade-offs.

## Technique 1: Polling (Programmed I/O)

The CPU **repeatedly checks** the device's status register in a tight loop until the device is ready.

```
CPU                          Device Controller
 |                                  |
 |--- Read status register -------->|
 |<-- "Not ready" -----------------|
 |                                  |
 |--- Read status register -------->|
 |<-- "Not ready" -----------------|
 |                                  |  (device working...)
 |--- Read status register -------->|
 |<-- "Ready! Data available" -----|
 |                                  |
 |--- Read data register ---------->|
 |<-- Data byte/word --------------|
 |                                  |
 v                                  v
```

```c
// Polling loop (pseudocode)
while (read_status_register(device) & BUSY_BIT) {
    // spin... spin... spin...
    // CPU is doing NOTHING useful
}
data = read_data_register(device);
```

**Pros:** Dead simple. No interrupt overhead. Lowest possible latency -- you get the data the instant it's ready.

**Cons:** Burns CPU cycles doing nothing. If the device takes 10ms, the CPU wastes millions of cycles checking a register that says "not ready."

**When to use:** When the wait is extremely short (nanoseconds) or when latency matters more than CPU efficiency.

## Technique 2: Interrupts

Instead of the CPU checking the device, the **device signals the CPU** when it's ready. The CPU can do useful work in the meantime.

```
CPU                          Device Controller
 |                                  |
 |--- "Read sector 42" ----------->|
 |                                  |  (device working...)
 |                                  |
 |  (CPU runs other processes)      |
 |  (scheduling, computing, etc.)   |
 |                                  |
 |<======= INTERRUPT! =============|  (device done!)
 |                                  |
 |  [save CPU state]                |
 |  [jump to interrupt handler]     |
 |  [read data from device]         |
 |  [wake up waiting process]       |
 |  [restore CPU state]             |
 |                                  |
 |  (CPU resumes previous work)     |
 v                                  v
```

### Interrupt Handling Steps

1. Device asserts interrupt line (electrical signal on the bus)
2. CPU finishes current instruction
3. CPU saves current state (registers, program counter)
4. CPU looks up the **interrupt vector table** to find the handler address
5. CPU jumps to the **interrupt handler** (Interrupt Service Routine / ISR)
6. Handler reads data from device, processes it, wakes up waiting process
7. CPU restores state, resumes what it was doing

### The Cost of Interrupts

Interrupts aren't free. Each one requires:
- Saving/restoring CPU state (registers, flags)
- Flushing CPU pipeline
- Possible cache pollution (handler code evicts useful cache lines)
- Mode switch to kernel (if not already in kernel)

For very fast devices or very frequent events, interrupt overhead can be significant.

### Interrupt Coalescing

When a device generates thousands of events per second (like a busy network card), you don't want thousands of interrupts per second. **Interrupt coalescing** batches multiple events into a single interrupt.

```
Without coalescing:                With coalescing:
Packet 1 -> Interrupt              Packet 1 -> (wait)
Packet 2 -> Interrupt              Packet 2 -> (wait)
Packet 3 -> Interrupt              Packet 3 -> (wait)
Packet 4 -> Interrupt              Timer fires -> ONE interrupt
                                   (process all 4 packets)
4 context switches!                1 context switch!
```

Linux NAPI (New API) for networking does exactly this: when packet rate is high, it switches from interrupt mode to polling mode. This is a **hybrid approach** -- interrupts for low traffic, polling for high traffic.

## Technique 3: DMA (Direct Memory Access)

With polling and interrupts, the **CPU** moves every byte between device and memory. DMA removes the CPU from the data path entirely.

A **DMA controller** (dedicated hardware) transfers data directly between the device and memory. The CPU just sets up the transfer and gets notified when it's done.

```
CPU                     DMA Controller           Device
 |                           |                      |
 |-- "Transfer 4KB from   ->|                      |
 |    device to addr 0x1000" |                      |
 |                           |                      |
 |  (CPU does other work)    |--- Request data ---->|
 |  (CPU does other work)    |<--- Data chunk 1 ----|
 |  (CPU does other work)    |--- Write to RAM      |
 |  (CPU does other work)    |<--- Data chunk 2 ----|
 |  (CPU does other work)    |--- Write to RAM      |
 |  (CPU does other work)    |<--- Data chunk 3 ----|
 |  (CPU does other work)    |--- Write to RAM      |
 |                           |                      |
 |<=== Interrupt: "Done!" ===|                      |
 |                           |                      |
 |  (process the data at     |                      |
 |   0x1000 in RAM)          |                      |
 v                           v                      v
```

### DMA Setup Steps

1. CPU programs the DMA controller with:
   - Source address (device register or memory address)
   - Destination address (memory address)
   - Transfer size (number of bytes)
   - Transfer direction (device-to-memory or memory-to-device)
2. DMA controller takes over the bus and transfers data
3. CPU is free to execute other code during the transfer
4. DMA controller sends an interrupt when transfer is complete
5. CPU processes the data that's now in memory

### Why DMA is Essential

Without DMA, reading 1 MB from disk would require the CPU to execute ~1 million byte-copy instructions. With DMA, the CPU executes maybe 10 instructions (program DMA, handle completion interrupt) and the hardware moves the data.

**Every significant I/O operation in a modern system uses DMA**: disk reads/writes, network packet send/receive, GPU data transfers, sound playback.

## Comparison Table

| Property | Polling | Interrupts | DMA |
|----------|---------|------------|-----|
| CPU during transfer | Busy-waiting (wasted) | Doing other work | Doing other work |
| Data movement | CPU moves each byte | CPU moves each byte | DMA hardware moves data |
| Latency | Lowest (instant detection) | Higher (interrupt overhead) | Higher (DMA setup overhead) |
| CPU overhead | Very high | Moderate (per interrupt) | Very low (setup + completion only) |
| Throughput | Low (CPU bottleneck) | Moderate | Highest |
| Complexity | Trivial | Moderate (handler code) | Higher (DMA programming) |
| Best for | Ultra-fast devices, short waits | Moderate-speed, infrequent events | Bulk data transfers |

## When to Use Which

```
Decision Tree:

Is the wait extremely short (< microseconds)?
  YES --> Polling (avoid interrupt overhead)
  NO  --> Is this a bulk data transfer?
            YES --> DMA (don't waste CPU copying bytes)
            NO  --> Interrupts (let CPU do useful work)

Special case: Very high event rate?
  --> Hybrid: interrupt coalescing or adaptive polling (NAPI)
```

### Real Examples

| Device / Scenario | Technique | Why |
|-------------------|-----------|-----|
| NVMe SSD read (4KB) | DMA + interrupt | Bulk data transfer, CPU shouldn't copy bytes |
| Network packet receive | DMA + interrupt (coalesced) | Bulk data, high packet rates need coalescing |
| High-frequency trading NIC | Polling (busy-wait) | Need absolute lowest latency, CPU cost doesn't matter |
| Keyboard input | Interrupt | Infrequent events, CPU should not waste cycles waiting |
| GPU texture upload | DMA | Gigabytes of data, CPU must not be in the data path |
| Reading CPU temperature sensor | Polling | Simple register read, takes nanoseconds |

## Real-World Connection

**NVMe uses all three**: NVMe SSDs use DMA to transfer data to memory, interrupts (MSI-X) to signal completion, and the submission/completion queues are essentially a polling-friendly ring buffer structure that can also be interrupt-driven.

**Linux NAPI networking**: At low packet rates, the NIC uses interrupts. When traffic spikes, Linux switches to polling mode (the kernel polls the NIC in a loop). This prevents interrupt storms -- a situation where the CPU spends 100% of its time handling interrupts and can't process any actual packets.

**High-frequency trading**: Trading firms spend millions on NICs that support kernel-bypass and busy-wait polling. They burn entire CPU cores doing nothing but polling a network card, because even the 5-microsecond interrupt latency is too much when trades happen in nanoseconds.

**Cloud disk I/O**: When your EC2 instance reads from EBS, the hypervisor uses DMA to transfer data from the network-attached storage. The "disk read" is actually a network DMA transfer -- which is why EBS latency is higher than local NVMe.

**USB devices**: USB uses interrupt transfers (the protocol-level term, different from CPU interrupts) for keyboards/mice and bulk transfers for storage. The USB host controller uses DMA to move data to memory.

## Interview Angle

**Q: What happens when a process does a `read()` on a disk file? Walk through the I/O path.**

A: The process calls `read()`, which traps into the kernel. The kernel checks the buffer cache -- if the data is cached, it copies to user space and returns. If not, the kernel programs the DMA controller with the disk address, memory destination, and size. The CPU schedules another process to run. The DMA controller transfers data from disk to memory. When done, the DMA controller fires an interrupt. The interrupt handler marks the I/O as complete and wakes up the waiting process. The process resumes and finds its data in the buffer.

**Q: Why not just use DMA for everything?**

A: DMA has setup overhead -- programming the controller, allocating DMA-safe memory buffers, handling the completion interrupt. For tiny transfers (reading a single status register), this overhead exceeds the cost of just having the CPU read the register directly. DMA shines for bulk transfers where the setup cost is amortized over many bytes.

**Q: What is an interrupt storm and how do you prevent it?**

A: An interrupt storm happens when a device generates interrupts faster than the CPU can handle them. The CPU spends 100% of its time entering/exiting interrupt handlers and never processes any actual work. Prevention: interrupt coalescing (batch events into fewer interrupts), adaptive polling (switch to polling under high load -- Linux NAPI does this), and rate limiting on the device.

**Q: In what scenario would you choose polling over interrupts?**

A: When latency matters more than CPU efficiency and the expected wait is very short. High-frequency trading is the classic example -- firms dedicate entire CPU cores to busy-wait polling on network cards because even microsecond interrupt latency is unacceptable. Also useful when polling a device that responds in nanoseconds (CPU temperature sensor, some hardware registers).

---

**Next:** [Device Drivers](03-device-drivers.md) -- the software layer that translates OS commands into device-specific protocols.
