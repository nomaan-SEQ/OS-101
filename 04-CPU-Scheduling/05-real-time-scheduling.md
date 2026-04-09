# Real-Time Scheduling

Real-time scheduling is about **deadlines**, not speed. A real-time system must complete work within a guaranteed time bound. Missing a deadline isn't just poor performance — in hard real-time systems, it can mean a car's airbag deploys too late or a pacemaker misses a heartbeat. This section covers the unique scheduling challenges and algorithms for time-critical systems.

---

## Hard Real-Time vs Soft Real-Time

```
Hard Real-Time:
  Deadline: 10ms
  |=== Task ===>|
  0            10ms
  Must finish HERE or system fails.
  
  Examples:
  - Airbag deployment: must fire within 10ms of impact
  - Pacemaker: must pace heart within exact timing window
  - Anti-lock braking: must modulate brakes within milliseconds
  - Industrial robot arm: must stop at precise position

Soft Real-Time:
  Deadline: 33ms (30fps video)
  |=== Frame render ===>|
  0                    33ms
  Should finish here, but if it misses...
  you get a dropped frame, not a crash.
  
  Examples:
  - Video playback: dropped frames = stutter, not disaster
  - Audio streaming: late samples = clicks/pops
  - Online gaming: late frame = lag spike
  - VoIP: late packet = brief audio gap
```

| Aspect | Hard Real-Time | Soft Real-Time |
|---|---|---|
| Deadline miss | System failure, potential harm | Degraded quality, recoverable |
| Guarantee | Must be mathematically provable | Statistical / best-effort |
| OS type | Dedicated RTOS (VxWorks, QNX, FreeRTOS) | General-purpose with RT extensions |
| Scheduling | Admission control — reject if can't guarantee | Best-effort, allow deadline misses |
| Examples | Medical devices, avionics, automotive | Multimedia, gaming, telecom |
| Overhead tolerance | Low — predictability over throughput | Moderate — throughput matters too |

---

## Rate Monotonic Scheduling (RMS)

A **static priority** algorithm for periodic tasks. Each task has a fixed period, and priority is assigned based on that period: **shorter period = higher priority**.

```
Task    Period    Burst    Priority
 T1      4ms      1ms     Highest (shortest period)
 T2      6ms      2ms     Lower
 T3     12ms      3ms     Lowest (longest period)

Timeline (RMS):
    0    2    4    6    8   10   12
    |T1|T2 |T1|T2|T1| T3  |T1|T2|T3  |...
    
T1 (period=4): runs at t=0, 4, 8, 12...
T2 (period=6): runs at t=1, 4.something...
T3 (period=12): runs whenever T1 and T2 aren't running
```

**Key properties:**
- Priority is fixed (static) — assigned at design time based on period
- Optimal among static-priority algorithms for independent periodic tasks
- Simple to implement and analyze

**Schedulability bound** (Liu & Layland): RMS can guarantee all deadlines if:

```
U = sum(Ci/Ti) <= n * (2^(1/n) - 1)

Where:
  Ci = burst time of task i
  Ti = period of task i
  n  = number of tasks

For large n, this converges to ~69.3% (ln 2)

Example: 3 tasks
  U = 1/4 + 2/6 + 3/12 = 0.25 + 0.33 + 0.25 = 0.83
  Bound = 3 * (2^(1/3) - 1) = 3 * 0.26 = 0.78
  
  0.83 > 0.78 --> NOT guaranteed schedulable by RMS
  (May still work in practice, but no guarantee)
```

The ~69.3% utilization bound means RMS can waste up to 30% of CPU capacity. This is the cost of guaranteeing deadlines with static priorities.

---

## Earliest Deadline First (EDF)

A **dynamic priority** algorithm: at any scheduling decision, run the task whose deadline is nearest. Priority changes at every scheduling point.

```
Task    Period    Burst    
 T1      4ms      1ms     
 T2      6ms      2ms     

Timeline (EDF):
t=0: T1 deadline=4, T2 deadline=6 --> T1 runs (closer deadline)
t=1: T1 done. T2 deadline=6 --> T2 runs
t=3: T2 done. Idle until t=4.
t=4: T1 deadline=8, T2 deadline(still)=6 --> T2 runs? No, T2 next
     instance not until t=6. T1 runs.
t=5: T1 done.
t=6: T1 deadline=8, T2 deadline=12 --> T1 has closer, but T1 isn't
     ready. T2 runs.
...
```

**Key properties:**
- Theoretically optimal — can schedule any task set up to **100% CPU utilization** (vs RMS's ~69%)
- Dynamic priorities — more complex to implement
- Harder to analyze in the presence of overload (when deadlines will be missed, behavior is unpredictable)

| Aspect | RMS | EDF |
|---|---|---|
| Priority type | Static (based on period) | Dynamic (based on deadline) |
| Utilization bound | ~69% (guaranteed) | 100% (theoretical) |
| Optimal for | Static-priority algorithms | All uniprocessor algorithms |
| Implementation | Simple — priorities fixed at design time | Complex — priorities recomputed at each decision |
| Overload behavior | Predictable (lowest-priority task misses) | Unpredictable (cascade of misses) |
| Used in | Most RTOS implementations | Linux SCHED_DEADLINE |

---

## Priority Inversion and Priority Inheritance

**Priority inversion** is one of the most subtle and dangerous bugs in real-time systems. It occurs when a high-priority task is blocked waiting for a resource held by a low-priority task, which itself is preempted by a medium-priority task.

```
Priority: H > M > L

Normal expectation: H runs first, then M, then L

Priority Inversion scenario:
                                                    
Time -->  1    2    3    4    5    6    7    8    9
         
  H:     .....waiting..........[=========]
  M:     .....[=======M runs========].....
  L:     [=L=]                        [L=]
         ^    ^                ^      ^
         |    |                |      L releases lock,
         |    M preempts L     |      H finally runs
         |    (M has no        |
         L    dependency!)     H arrives, needs lock
         grabs                 held by L, blocks!
         lock                  

  H is blocked by L (who holds the lock)
  M preempts L (M is higher priority than L)
  So H waits for M, even though H > M!
  
  Effective priority: M > H > L  (INVERTED!)
```

This is not just a theoretical problem. It caused one of the most famous software bugs in history.

### The Mars Pathfinder Bug (1997)

The Mars Pathfinder rover experienced repeated system resets on Mars due to priority inversion:

1. A **low-priority** meteorological task held a shared mutex (bus access)
2. A **high-priority** data bus task needed the same mutex and blocked
3. A **medium-priority** communications task preempted the low-priority task
4. The high-priority task was blocked for so long that a watchdog timer fired and reset the system

NASA engineers diagnosed the problem remotely and uploaded a fix: **priority inheritance**.

### Priority Inheritance Protocol

When a low-priority task holds a resource needed by a high-priority task, temporarily **boost the low-priority task's priority** to match the high-priority task.

```
With Priority Inheritance:

Time -->  1    2    3    4    5    6    7
         
  H:     .....waiting...[========H runs=====]
  M:     .....waiting...........[====M runs=]
  L:     [=L=][L boosted to H's priority!][done]
         ^    ^         ^      
         |    H arrives, L inherits H's
         |    needs lock priority, can't be
         L    held by L  preempted by M!
         grabs
         lock

  L's priority is temporarily raised to H's level
  M cannot preempt L anymore
  L finishes quickly, releases lock
  H runs immediately
  M runs after H
  
  Correct priority order restored: H > M > L
```

| Without Inheritance | With Inheritance |
|---|---|
| H waits for M to finish (unbounded!) | H waits only for L to release lock |
| Deadline miss likely for H | Bounded delay for H |
| Medium-priority tasks can cause arbitrary delay | Low-priority task can't be preempted while holding critical resource |

---

## Real-Time in Linux

Linux is not a hard real-time OS, but it provides soft real-time scheduling policies:

| Policy | Type | Description |
|---|---|---|
| `SCHED_FIFO` | Real-time | Fixed priority (1-99), runs until blocks or higher priority arrives |
| `SCHED_RR` | Real-time | Like FIFO but with time quantum among same-priority tasks |
| `SCHED_DEADLINE` | Real-time | EDF-based, specify runtime/deadline/period per task |
| `SCHED_OTHER` | Normal | Default CFS policy for regular tasks |
| `SCHED_BATCH` | Normal | For non-interactive CPU-bound work |
| `SCHED_IDLE` | Normal | Lowest priority, runs only when nothing else needs CPU |

```
Linux Priority Space:
  
  Priority 0-99:   Real-time (SCHED_FIFO / SCHED_RR)
                    Higher number = higher priority
                    Always preempts normal tasks
  
  Priority 100-139: Normal (SCHED_OTHER / CFS)
                    Controlled by nice values (-20 to +19)
                    nice -20 = priority 100 (highest normal)
                    nice +19 = priority 139 (lowest normal)
```

```bash
# Run a process with real-time FIFO scheduling, priority 50
sudo chrt -f 50 ./my_realtime_process

# Run with real-time Round Robin, priority 30
sudo chrt -r 30 ./my_process

# Run with SCHED_DEADLINE
# (runtime=5ms, deadline=10ms, period=20ms)
sudo chrt -d --sched-runtime 5000000 \
              --sched-deadline 10000000 \
              --sched-period 20000000 0 ./my_process

# Check scheduling policy of a running process
chrt -p <PID>
```

### Why General-Purpose OSes Struggle with Hard Real-Time

| Challenge | Why It Matters |
|---|---|
| Non-preemptible kernel sections | Kernel code may disable preemption for synchronization |
| Interrupt latency | Interrupt handlers can delay scheduling decisions |
| Memory management | Page faults cause unpredictable delays |
| Lock contention | Kernel locks can block real-time tasks |
| Garbage collection | Languages with GC add unpredictable pauses |

The **PREEMPT_RT** patch set makes the Linux kernel more real-time-friendly by making most kernel code preemptible and converting spinlocks to mutexes with priority inheritance. It's been gradually merged into mainline Linux and is suitable for soft real-time and some firm real-time workloads.

For hard real-time requirements, dedicated RTOSes (FreeRTOS, VxWorks, QNX, Zephyr) are used instead. Some architectures run a hard RTOS alongside Linux, with the RTOS handling time-critical tasks and Linux handling everything else.

---

## Real-World Connection

- **Audio production**: DAWs (Digital Audio Workstations) use real-time scheduling to ensure audio buffers are filled before the sound card needs them. On Linux, JACK audio server runs with `SCHED_FIFO`. Buffer underruns cause audible clicks.
- **Autonomous vehicles**: Self-driving cars use hard real-time systems for braking and steering (often QNX or safety-certified Linux). The perception and planning stack may use soft real-time on a separate compute unit.
- **Cloud/Kubernetes**: Kubernetes has no real-time scheduling concept. For latency-sensitive workloads (trading, gaming servers), you must configure the underlying OS: use `isolcpus` to dedicate cores, set `SCHED_FIFO` for critical threads, and disable kernel features that cause jitter (CPU frequency scaling, transparent huge pages).
- **Gaming**: Game engines typically need soft real-time. Maintaining 60fps means each frame must complete within 16.67ms. The game loop is essentially a periodic real-time task with a 16.67ms period.

---

## Interview Angle

**Q: Explain priority inversion with a real-world example. How is it solved?**

A: Priority inversion occurs when a high-priority task is indirectly blocked by a medium-priority task. Classic example: the Mars Pathfinder on Mars had a high-priority bus management task blocked on a mutex held by a low-priority meteorological task. A medium-priority communications task kept preempting the low-priority task, preventing it from releasing the mutex. The high-priority task starved and a watchdog timer rebooted the system. The fix is priority inheritance: when a low-priority task holds a resource needed by a high-priority task, temporarily boost the low-priority task's priority so it can finish and release the resource without being preempted by medium-priority tasks.

**Q: When would you use SCHED_FIFO vs SCHED_DEADLINE in Linux?**

A: Use `SCHED_FIFO` when you have tasks with clear static priority relationships and you want the highest-priority task to always preempt lower ones — simple and predictable. Use `SCHED_DEADLINE` when tasks have explicit timing requirements (runtime, deadline, period) and you want EDF-like scheduling — it's more sophisticated and can achieve higher CPU utilization. `SCHED_DEADLINE` is also better for isolating real-time tasks because the kernel performs admission control and won't accept a task if it can't guarantee its deadline.

**Q: Why can't you just use Linux for a pacemaker?**

A: Standard Linux cannot guarantee bounded worst-case response times. The kernel has non-preemptible sections, interrupt handlers can introduce unpredictable delays, page faults can stall a process, and the scheduler's behavior is optimized for average throughput rather than worst-case latency. Even with PREEMPT_RT, Linux provides soft real-time guarantees — very good average case, but no mathematical proof of worst-case bounds. Medical devices require certified hard real-time systems (IEC 62304) where every execution path has a provable upper time bound.

---

Next: [Linux CFS and Real-World Schedulers](06-linux-cfs-and-real-world-schedulers.md)
