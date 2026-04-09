# Round Robin and Multilevel Queues

FCFS, SJF, and Priority scheduling each have fundamental limitations. Round Robin addresses the responsiveness problem, and multilevel queues combine multiple algorithms to handle different types of workloads simultaneously. The multilevel feedback queue — which lets processes move between queues — is the design that most real schedulers approximate.

---

## Round Robin (RR)

Each process gets a fixed **time quantum** (also called a time slice). When the quantum expires, the process is preempted and placed at the back of the ready queue. The next process in line gets its turn.

```
Ready Queue (FIFO):  [P1] -> [P2] -> [P3] -> [P4]

Time quantum = 4 units

      P1 runs    P2 runs    P3 runs    P4 runs    P1 runs
     (4 units)  (4 units)  (4 units)  (4 units)  (remaining)
    |----------|----------|----------|----------|---------|
    0          4          8         12         16        20
    
    After quantum: P1 goes to back of queue
    [P2] -> [P3] -> [P4] -> [P1]
```

**Properties:**
- Preemptive — the timer interrupt enforces the quantum
- No starvation — every process gets a turn within `n * quantum` time (where n = number of processes)
- Designed for time-sharing / interactive systems
- Fair by default — every process gets equal CPU time

### Example with Quantum = 4

```
Process  Arrival  Burst
  P1       0       6
  P2       0       3
  P3       0       8
  P4       0       4

Gantt Chart (quantum = 4):
| P1  | P2 | P3  | P4  | P1| P3  |
0     4    7    11    15  17   21

t=0:  P1 runs for 4 (remaining: 2)
t=4:  P2 runs for 3 (done, burst < quantum)
t=7:  P3 runs for 4 (remaining: 4)
t=11: P4 runs for 4 (done)
t=15: P1 runs for 2 (done)
t=17: P3 runs for 4 (done)

             Turnaround    Waiting
Process      (Finish-Arr)  (Turn-Burst)
  P1         17-0 = 17     17-6  = 11
  P2          7-0 =  7      7-3  =  4
  P3         21-0 = 21     21-8  = 13
  P4         15-0 = 15     15-4  = 11

Averages:     15.0           9.75
```

### The Quantum Trade-Off

The choice of time quantum is the most critical parameter in Round Robin:

```
Quantum too small (e.g., 1ms):
  |P1|P2|P3|P1|P2|P3|P1|P2|P3|...
  ^ context switch overhead dominates!
  Extreme responsiveness, but most CPU time spent switching.

Quantum too large (e.g., 10 seconds):
  |    P1 (10s)     |    P2 (10s)     |    P3 (10s)     |
  Degenerates into FCFS — loses responsiveness.
  User waits up to 10s * n for a response.

Sweet spot (10-100ms):
  |  P1  |  P2  |  P3  |  P1  |  P2  |...
  Good responsiveness with acceptable overhead.
```

| Quantum Size | Context Switches | Response Time | Throughput | Feels Like |
|---|---|---|---|---|
| Very small (< 1ms) | Extremely high | Excellent | Poor (overhead) | Smooth but slow |
| Small (1-10ms) | High | Very good | Good | Responsive |
| Medium (10-100ms) | Moderate | Good | Good | **Sweet spot** |
| Large (> 100ms) | Low | Poor | Good | Laggy, like FCFS |

**Rule of thumb**: The quantum should be large enough that 80% of CPU bursts complete within a single quantum. This minimizes unnecessary context switches while keeping the system responsive.

---

## Multilevel Queue Scheduling

Real systems run many types of processes with different scheduling needs. A web server's request handler needs low latency. A nightly backup job just needs to finish eventually. Putting them in the same queue doesn't work well.

**Solution**: Multiple ready queues, each with its own scheduling algorithm and priority level.

```
+------------------------------------------+
|  Queue 0: System Processes     [RR, q=8] |  Highest Priority
+------------------------------------------+
|  Queue 1: Interactive/Foreground [RR, q=16]|
+------------------------------------------+
|  Queue 2: Batch Processes       [FCFS]   |
+------------------------------------------+
|  Queue 3: Student/Low Priority  [FCFS]   |  Lowest Priority
+------------------------------------------+

Scheduling between queues:
  Option A: Fixed priority — serve Queue 0 first, then 1, then 2...
            Risk: lower queues starve
  Option B: Time-slice — Queue 0 gets 50%, Queue 1 gets 30%, 
            Queue 2 gets 15%, Queue 3 gets 5%
```

**Key characteristic**: Processes are permanently assigned to a queue based on type. A batch process stays in the batch queue forever.

| Feature | Description |
|---|---|
| Queue assignment | Fixed — based on process type or user class |
| Per-queue algorithm | Each queue can use a different algorithm |
| Inter-queue scheduling | Fixed priority or time-sliced between queues |
| Starvation risk | Yes, if fixed priority between queues |
| Flexibility | Low — can't adapt to changing process behavior |

---

## Multilevel Feedback Queue (MLFQ)

The key improvement: processes can **move between queues** based on their observed behavior. This solves the fundamental problem of not knowing process characteristics in advance.

```
Queue 0 (highest priority): RR with quantum = 8ms
+--------------------------------------------------+
|  New processes start here                        |
|  If process uses full quantum --> demote to Q1   |
|  If process blocks before quantum --> stay in Q0 |
+--------------------------------------------------+
          |  (used full quantum = CPU-bound behavior)
          v
Queue 1 (medium priority): RR with quantum = 16ms
+--------------------------------------------------+
|  If process uses full quantum --> demote to Q2   |
|  If process blocks before quantum --> promote Q0 |
+--------------------------------------------------+
          |  (used full quantum again)
          v
Queue 2 (lowest priority): FCFS
+--------------------------------------------------+
|  CPU-bound processes end up here                 |
|  After waiting long enough --> promote (aging)   |
+--------------------------------------------------+
```

### How It Works

1. **New process enters Queue 0** (highest priority, smallest quantum)
2. **I/O-bound process** uses a tiny CPU burst, blocks before quantum expires, stays in (or returns to) Queue 0 -- gets high priority and quick turnaround
3. **CPU-bound process** uses the full quantum, gets demoted to Queue 1 with a larger quantum
4. If it uses the full quantum again in Queue 1, it drops to Queue 2 (FCFS, lowest priority)
5. **Aging** periodically promotes long-waiting processes back up to prevent starvation

```
Process A (I/O-bound):           Process B (CPU-bound):
  Arrives -> Q0                    Arrives -> Q0
  Uses 2ms, blocks on I/O         Uses full 8ms quantum
  Stays in Q0 (high priority!)    Demoted to Q1
  Returns from I/O -> Q0          Uses full 16ms quantum
  Uses 1ms, blocks again          Demoted to Q2 (FCFS)
  Stays in Q0                     Runs when Q0 and Q1 empty
  
Result: I/O-bound gets fast       Result: CPU-bound still runs,
service, good responsiveness      just with lower priority
```

### Why MLFQ is Brilliant

MLFQ approximates SJF **without knowing burst lengths**:
- Short-burst (I/O-bound) processes stay in high-priority queues -- like SJF picking the shortest job
- Long-burst (CPU-bound) processes sink to low-priority queues -- like SJF putting long jobs last
- The system automatically classifies processes based on observed behavior

### MLFQ Parameters

A fully defined MLFQ needs:

| Parameter | Purpose |
|---|---|
| Number of queues | How many priority levels |
| Algorithm per queue | RR quantum, FCFS, etc. |
| Promotion rules | When to move a process up |
| Demotion rules | When to move a process down |
| Initial queue | Where new processes start |
| Aging threshold | How long before forced promotion |

---

## Comparison

| Aspect | Round Robin | Multilevel Queue | Multilevel Feedback Queue |
|---|---|---|---|
| Preemptive? | Yes | Depends on per-queue algorithm | Yes |
| Process movement | N/A (single queue) | No (fixed assignment) | Yes (based on behavior) |
| Favors | Equal time sharing | Pre-classified process types | I/O-bound (auto-detected) |
| Starvation? | No | Possible between queues | Prevented via aging |
| Needs configuration? | Just quantum | Queue assignments, algorithms | Many parameters |
| Adaptiveness | None | None | High (self-adjusting) |
| Approximates | Fair sharing | Static priority | SJF (without predictions) |

---

## Real-World Connection

- **Linux CFS** doesn't use a traditional MLFQ, but achieves similar results through virtual runtime: I/O-bound processes accumulate less vruntime and naturally get prioritized when they wake up.
- **Windows** uses a multilevel feedback queue with 32 priority levels. Interactive processes get temporary priority boosts when they receive input (window focus, keyboard input), which is a form of the MLFQ promotion mechanism.
- **Kubernetes** scheduling has echoes of multilevel queues: Priority Classes (system-critical, high, normal, low) determine which pods get scheduled first and which get evicted under pressure. But unlike OS scheduling, K8s scheduling is a one-time placement decision, not continuous CPU allocation.
- **Task queues** (Celery, Sidekiq, Bull) in web applications often implement multilevel queues: a high-priority queue for user-facing operations (password resets, payment processing) and a low-priority queue for background jobs (report generation, data cleanup).

---

## Interview Angle

**Q: How do you choose the right time quantum for Round Robin?**

A: The quantum should be large enough that most CPU bursts complete within one quantum (the 80% rule), but small enough to maintain good response time. Typical values are 10-100ms. If your context switch takes 0.1ms and your quantum is 10ms, you lose ~1% to overhead — acceptable. If the quantum is 0.5ms, you lose ~20% — unacceptable. In practice, modern schedulers like CFS don't use a fixed quantum; they compute dynamic time slices based on the number of runnable processes.

**Q: Explain how MLFQ approximates SJF without knowing burst lengths.**

A: MLFQ uses observed behavior as a proxy for burst length. New processes start at the highest priority queue with a small quantum. If a process uses its entire quantum (long burst), it's demoted to a lower queue — the system infers it's CPU-bound. If it blocks before the quantum expires (short burst), it stays at high priority — inferred I/O-bound. Over time, short-burst processes float to the top and long-burst processes sink to the bottom, mimicking SJF ordering without any burst prediction.

**Q: What happens if a clever process games the MLFQ by doing a tiny I/O right before its quantum expires?**

A: This is a real exploit. A process could issue a trivial I/O operation just before its quantum expires, resetting to a high-priority queue and getting more than its fair share of CPU. Modern schedulers defend against this by tracking total CPU time consumed (not just per-burst) and using that for queue assignment. Linux CFS avoids the problem entirely by tracking cumulative virtual runtime rather than using queue demotion.

---

Next: [Multiprocessor and Multicore Scheduling](04-multiprocessor-and-multicore-scheduling.md)
