# Round Robin (RR) Scheduling

## Introduction

Round Robin (RR) is one of the most widely used CPU scheduling algorithms in modern operating systems, particularly for time-sharing and interactive systems. Unlike FCFS (which lets processes run to completion) or SJF (which prioritizes short jobs), Round Robin gives each process a small, fixed slice of CPU time called a time quantum (or time slice) and then moves on to the next process in the queue.

The name "Round Robin" comes from the French phrase ruban rond (round ribbon), referring to a document where signatories arranged their names in a circle so no one could be identified as signing first—symbolizing equality. In scheduling, this translates to equal treatment: every process gets a fair share of the CPU.

Round Robin is the foundation of modern preemptive scheduling and is used (in various forms) by Windows, Linux, macOS, and virtually every general-purpose operating system.

---

## How Round Robin Works

### The Basic Principle

Round Robin operates on a simple, fair principle: each process gets a small, fixed amount of CPU time in rotation.

```
Round Robin Algorithm:

1. The ready queue is maintained as a circular FIFO queue
2. The first process in the queue is selected to run
3. The process runs for at most one time quantum (q)
4. When the quantum expires:
   ├── If process is not finished, it goes to the tail of the queue
   ├── If process completes, it leaves the system
   └── If process blocks for I/O, it moves to a wait queue
5. The scheduler selects the next process from the head of the queue
6. Repeat from step 2
```

### Visual Representation

```
Round Robin with Time Quantum = 4 ms

Ready Queue (circular):
    ┌─────────────────────────────────────┐
    │                                     │
    ▼                                     │
┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
│ P1  │→│ P2  │→│ P3  │→│ P4  │→│ P5  │──┘
└─────┘  └─────┘  └─────┘  └─────┘  └─────┘
  │
  └──▶ CPU runs P1 for 4ms, then moves to tail

CPU Execution:
[P1: 0-4ms] → [P2: 4-8ms] → [P3: 8-12ms] → [P4: 12-16ms] → [P5: 16-20ms]
     ↓ (if not done)
[P1: 20-24ms] → [P2: 24-28ms] → ...
```

### The Circular Queue

The key data structure is a circular FIFO queue:

```
Round Robin Ready Queue:

    Head                    Tail
    ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
    │ P2  │→│ P3  │→│ P4  │→│ P1  │───┐
    └─────┘  └─────┘  └─────┘  └─────┘   │
                                         │
    When P1's quantum expires, it goes ──┘
    to the tail (after P4)
    
    New state:
    ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
    │ P3  │→│ P4  │→│ P1  │→│ P5  │───┐
    └─────┘  └─────┘  └─────┘  └─────┘   │
                                         │
    (P2 completed, P5 arrived)
```

### Key Characteristics

| Property                             | Value                           |
|-------------------------------------|---------------------------------|
| Preemptive?                         | Yes (preemptive by timer)      |
| Starvation possible?                | No (everyone gets a turn)      |
| Requires burst time knowledge?      | No                              |
| Implementation complexity            | Low-Medium                      |
| Fairness                            | Excellent                       |
| Response time                       | Excellent (bounded)             |
| Context switch overhead              | High (frequent switches)        |

---

## The Time Quantum

The time quantum (or time slice) is the most critical parameter in Round Robin scheduling. It determines how long each process runs before being preempted.

### Quantum Size Trade-offs

```
Time Quantum Selection:

Very Small Quantum (e.g., 1ms):
├── Excellent response time
├── Very fair scheduling
├── BUT: High context switch overhead
├── CPU spends significant time switching
└── Reduced throughput

Very Large Quantum (e.g., 1000ms):
├── Low context switch overhead
├── High throughput
├── BUT: Poor response time
├── Behaves like FCFS
└── Interactive processes suffer

Optimal Quantum:
├── Large enough to amortize context switch cost
├── Small enough to provide good response time
├── Typically 10-100ms in modern systems
└── Balance between overhead and responsiveness
```

### The Context Switch Overhead Problem

```
Quantifying Overhead:

Context switch time: ~1-10 microseconds (modern hardware)
Process run time: quantum

Overhead ratio = context_switch_time / quantum

Examples:
├── Quantum = 1 ms: Overhead = 1-10% (significant)
├── Quantum = 10 ms: Overhead = 0.1-1% (acceptable)
├── Quantum = 100 ms: Overhead = 0.01-0.1% (minimal)
└── Quantum = 1000 ms: Overhead = 0.001-0.01% (negligible)

Rule of thumb: 
├── Quantum should be much larger than context switch time
├── Quantum >> context_switch_time
├── Typical: 100x to 1000x larger
└── This keeps overhead below 1%
```

### Visualization of Quantum Effects

```
Comparison of Different Quantum Sizes:

Workload: 5 processes, each needs 10ms CPU

Quantum = 2ms (Small):
[P1][P2][P3][P4][P5][P1][P2][P3][P4][P5]...
├── 25 context switches
├── Excellent response time
└── High overhead

Quantum = 10ms (Medium):
[P1: 0-10ms][P2: 10-20ms][P3: 20-30ms][P4: 30-40ms][P5: 40-50ms]
├── 5 context switches
├── Good response time
└── Low overhead

Quantum = 50ms (Large):
[P1: 0-10ms done][P2: 10-20ms done][P3: 20-30ms done]...
├── Behaves like FCFS
├── Poor response for last processes
└── Minimal overhead
```

### Quantum Selection Guidelines

```
Guidelines for Choosing Quantum:

1. Consider User Expectations
   ├── Humans perceive delays < 100ms as "instant"
   ├── Response time should feel immediate
   └── Quantum should be < 100ms for interactive systems

2. Consider Context Switch Cost
   ├── Quantum should be >> switch cost
   ├── At least 100x switch time
   └── Typically 10-100ms

3. Consider System Load
   ├── More processes → smaller quantum needed
   ├── Response time ≈ (number of processes) × quantum
   └── n × q should be reasonable for interactive users

4. Consider Process Types
   ├── I/O-bound: short bursts, benefit from small quantum
   ├── CPU-bound: long bursts, prefer larger quantum
   └── Mixed: compromise quantum

5. Typical Values
   ├── Linux CFS: no fixed quantum (dynamic)
   ├── Windows: ~15-30ms (varies by priority)
   ├── Traditional Unix: 100ms
   └── Modern interactive systems: 10-50ms
```

---

## Detailed Examples

### Example 1: Basic Round Robin

Scenario: Four processes arrive at time 0, quantum = 4ms:

| Process | Burst Time |
|---------|------------|
| P1      | 8 ms      |
| P2      | 6 ms      |
| P3      | 4 ms      |
| P4      | 2 ms      |

Execution:

```
Timeline:
0-4 ms:   P1 runs (remaining: 4ms)
4-8 ms:   P2 runs (remaining: 2ms)
8-12 ms:  P3 runs (completes at 12ms)
12-14 ms: P4 runs (completes at 14ms)
14-18 ms: P1 runs (completes at 18ms)
18-20 ms: P2 runs (completes at 20ms)

Gantt Chart:
0    4    8    12   14   18   20
|----|----|----|----|----|----|
  P1   P2   P3   P4   P1   P2

Queue Evolution:
Start:  [P1, P2, P3, P4]
After P1: [P2, P3, P4, P1]
After P2: [P3, P4, P1, P2]
After P3 completes: [P4, P1, P2]
After P4 completes: [P1, P2]
After P1 completes: [P2]
After P2 completes: []
```

Calculations:

```
Waiting Times:
P1: (4-0) + (14-8) = 4 + 6 = 10 ms
    (waited 0-4? No, ran 0-4. Waited 4-14, ran 14-18)
    Actually: P1 runs 0-4, waits 4-14, runs 14-18
    Waiting = 4 to 14 = 10 ms... but wait, P1 also waited between quantum
    Let me recalculate:
    P1: runs 0-4, waits 4-14, runs 14-18
    Waiting time = 14 - 4 = 10 ms (time in ready queue)

P2: runs 4-8, waits 8-18, runs 18-20
    Waiting = 18 - 8 = 10 ms

P3: runs 8-12, completes
    Waiting = 8 - 0 = 8 ms (waited for P1 and P2)

P4: runs 12-14, completes
    Waiting = 12 - 0 = 12 ms (waited for P1, P2, P3)

Average Waiting = (10 + 10 + 8 + 12) / 4 = 10 ms

Turnaround Times:
P1: 18 - 0 = 18 ms
P2: 20 - 0 = 20 ms
P3: 12 - 0 = 12 ms
P4: 14 - 0 = 14 ms

Average Turnaround = (18 + 20 + 12 + 14) / 4 = 16 ms

Response Times (time to first execution):
P1: 0 ms
P2: 4 ms
P3: 8 ms
P4: 12 ms

Average Response = (0 + 4 + 8 + 12) / 4 = 6 ms
```

### Example 2: Round Robin with Different Arrival Times

Scenario: Quantum = 3ms:

| Process | Arrival | Burst |
|---------|---------|-------|
| P1      | 0       | 5     |
| P2      | 1       | 4     |
| P3      | 2       | 2     |
| P4      | 3       | 3     |

Execution:

```
Timeline:
0-3 ms: P1 runs (remaining: 2ms)
       At 1ms: P2 arrives → queue: [P1, P2]
       At 2ms: P3 arrives → queue: [P1, P2, P3]
       At 3ms: P4 arrives → queue: [P1, P2, P3, P4]
       P1 quantum expires

3-6 ms: P2 runs (remaining: 1ms)
       Queue after P1 requeued: [P2, P3, P4, P1]

6-8 ms: P3 runs (completes at 8ms)
       Queue: [P4, P1, P2]

8-11 ms: P4 runs (remaining: 0ms? No, burst was 3, so completes at 11ms)
       Actually P4 burst = 3, runs 8-11, completes
       Queue: [P1, P2]

11-13 ms: P1 runs (remaining 0ms, completes)
         P1 had 2ms remaining, runs 11-13, completes
         Queue: [P2]

13-14 ms: P2 runs (remaining 0ms, completes)
         P2 had 1ms remaining, runs 13-14, completes

Gantt Chart:
0  1  2  3    6    8    11   13  14
|--|--|--|----|----|----|----|--|
  P1      P2     P3   P4   P1   P2
  (arrivals happen during P1's quantum)

Wait, let me redo this more carefully:

Time 0: P1 arrives, starts running (quantum 3ms)
Time 1: P2 arrives, joins queue
Time 2: P3 arrives, joins queue
Time 3: P4 arrives, joins queue. P1's quantum expires.
        Queue: [P2, P3, P4, P1]
        
Time 3-6: P2 runs (quantum 3ms, but only needs 4ms total)
Time 6: P2's quantum expires (used 3ms, remaining 1ms)
        Queue: [P3, P4, P1, P2]
        
Time 6-8: P3 runs (needs 2ms, completes at 8ms)
          Queue: [P4, P1, P2]
          
Time 8-11: P4 runs (needs 3ms, completes at 11ms)
           Queue: [P1, P2]
           
Time 11-13: P1 runs (remaining 2ms, completes at 13ms)
            Queue: [P2]
            
Time 13-14: P2 runs (remaining 1ms, completes at 14ms)
            Queue: []

Gantt Chart:
0        3    6   8    11   13 14
|--------|----|---|----|----|--|
    P1     P2   P3   P4   P1  P2
```

Calculations:

```
Waiting Times (time in ready queue):
P1: waits 3-11 = 8 ms
P2: waits 6-13 = 7 ms
P3: waits 6-6 = 0 ms (arrived at 2, ran at 6, but was in queue from 2-6)
    Actually P3 waits from 2 to 6 = 4 ms
P4: waits from 3 to 8 = 5 ms

Let me recalculate:
P1: Arrived 0, ran 0-3, waited 3-11, ran 11-13
    Waiting = 11 - 3 = 8 ms

P2: Arrived 1, ran 3-6, waited 6-13, ran 13-14
    Waiting = (3-1) + (13-6) = 2 + 7 = 9 ms
    But 3-1 includes time before first run... 
    
Actually, waiting time = turnaround - burst:
P1: Turnaround = 13 - 0 = 13, Waiting = 13 - 5 = 8 ms
P2: Turnaround = 14 - 1 = 13, Waiting = 13 - 4 = 9 ms
P3: Turnaround = 8 - 2 = 6, Waiting = 6 - 2 = 4 ms
P4: Turnaround = 11 - 3 = 8, Waiting = 8 - 3 = 5 ms

Average Waiting = (8 + 9 + 4 + 5) / 4 = 6.5 ms

Response Times:
P1: 0 - 0 = 0 ms
P2: 3 - 1 = 2 ms
P3: 6 - 2 = 4 ms
P4: 8 - 3 = 5 ms

Average Response = (0 + 2 + 4 + 5) / 4 = 2.75 ms
```

### Example 3: Effect of Different Quantum Sizes

Scenario: Five processes with burst times: 10, 8, 6, 4, 2 ms

#### Quantum = 2 ms:

```
Timeline:
0-2:   P1 (rem: 8)
2-4:   P2 (rem: 6)
4-6:   P3 (rem: 4)
6-8:   P4 (rem: 2)
8-10:  P5 (rem: 0, done)
10-12: P1 (rem: 6)
12-14: P2 (rem: 4)
14-16: P3 (rem: 2)
16-18: P4 (rem: 0, done)
18-20: P1 (rem: 4)
20-22: P2 (rem: 2)
22-24: P3 (rem: 0, done)
24-26: P1 (rem: 2)
26-28: P2 (rem: 0, done)
28-30: P1 (rem: 0, done)

Total: 30 ms
Context switches: 15
Average Waiting: High (long completion)
Average Response: 0-8ms (excellent)
```

#### Quantum = 4 ms:

```
Timeline:
0-4:   P1 (rem: 6)
4-8:   P2 (rem: 4)
8-12:  P3 (rem: 2)
12-16: P4 (rem: 0, done)
16-18: P5 (rem: 0, done) - only needs 2ms
18-22: P1 (rem: 2)
22-26: P2 (rem: 0, done)
26-28: P3 (rem: 0, done)
28-30: P1 (rem: 0, done)

Total: 30 ms
Context switches: 9
Average Response: 0-16ms (good)
```

#### Quantum = 10 ms:

```
Timeline:
0-10:  P1 (rem: 0, done)
10-18: P2 (rem: 0, done)
18-24: P3 (rem: 0, done)
24-28: P4 (rem: 0, done)
28-30: P5 (rem: 0, done)

Total: 30 ms
Context switches: 5
Average Response: 0-28ms (poor, like FCFS)
```

---

## Mathematical Analysis

### Response Time

For n processes each requiring at least one quantum:

```
Best case (new process arrives just before its turn):
Response time ≈ 0

Worst case (process just missed its turn):
Response time ≈ (n-1) × q

Average response time ≈ (n-1) × q / 2

Where:
├── n = number of processes in ready queue
├── q = time quantum

Example: n=10 processes, q=20ms
├── Worst case: 9 × 20 = 180ms
├── Average: ~90ms
└── This is why q must be small!
```

### Turnaround Time

For a process with burst time b, requiring k = ceil(b/q) quanta:

```
Best case (all quanta consecutive):
Turnaround ≈ b

Worst case (interleaved with n-1 other processes):
Turnaround ≈ b + (n-1) × q × k
          ≈ b + (n-1) × q × ceil(b/q)
          ≈ b + (n-1) × b (approximately, when b >> q)
          ≈ n × b

Example: n=5, b=10ms, q=2ms
├── k = 5 quanta needed
├── Worst case turnaround ≈ 10 + 4×2×5 = 50ms
├── Compare to SJF: 10ms
└── RR has much higher turnaround but better response
```

### Context Switch Overhead

```
Number of context switches:

For each process with burst b:
├── Number of quanta = ceil(b/q)
├── Each quantum (except last) causes a context switch

Total context switches ≈ Σ(ceil(bi/q) - 1) for all processes i

Example: 5 processes with bursts [10, 8, 6, 4, 2], q=2:
├── P1: ceil(10/2) - 1 = 4 switches
├── P2: ceil(8/2) - 1 = 3 switches
├── P3: ceil(6/2) - 1 = 2 switches
├── P4: ceil(4/2) - 1 = 1 switch
├── P5: ceil(2/2) - 1 = 0 switches
└── Total: 10 context switches

If context switch = 10μs:
├── Overhead = 10 × 10μs = 100μs
├── Total runtime = 30ms + 0.1ms
└── Overhead = 0.33% (acceptable)
```

### The Quantum-Response Time Relationship

```
Response Time vs. Quantum:

Response Time
    │
    │  *
    │ *  *
    │*    *
    │      *
    │       *
    │        *
    │         *
    │          *
    │           *
    └──────────────────── Quantum
    0              Large
    (Small)        (FCFS-like)

Optimal quantum:
├── Where curve flattens
├── Diminishing returns beyond this point
├── Typically 10-100ms
└── Depends on context switch cost
```

---

## Advantages of Round Robin

1. Fairness

```
Every process gets equal CPU time:
├── No process can monopolize CPU
├── Each process gets 1/n of CPU time
├── Simple definition of fairness
└── No starvation possible

Fairness property:
├── If n processes always ready
├── Each gets q time units every n×q interval
├── Guaranteed minimum service rate = 1/n
└── Predictable for users
```

2. Excellent Response Time

```
Response time bounded by (n-1) × q:
├── Interactive processes feel responsive
├── Users perceive system as fast
├── No process waits indefinitely
└── Good for time-sharing systems

Practical example:
├── 10 processes, q = 20ms
├── Maximum wait for first response = 180ms
├── Well within human perception threshold
└── System feels interactive
```

3. No Starvation

```
All processes make progress:
├── Every process eventually reaches head of queue
├── Even CPU-bound processes get CPU time
├── No process can be indefinitely delayed
├── Fair allocation guaranteed
└── Suitable for general-purpose systems

Contrast with SJF:
├── SJF: Long jobs can starve
├── RR: All jobs get service
└── RR is inherently fairer
```

4. Simple Implementation

```
Round Robin is straightforward:
├── FIFO queue with circular nature
├── Timer interrupt for preemption
├── Simple to add new processes (enqueue)
├── Simple to remove completed processes
└── Easy to understand and debug

Data structures needed:
├── Circular FIFO queue
├── Timer for quantum expiration
├── Process states (ready, running, waiting)
└── No complex calculations
```

5. Good for Interactive Systems

```
Round Robin excels at:
├── Time-sharing systems
├── Multi-user systems
├── Desktop/laptop systems
├── Any system with interactive users
└── Mixed workloads

Why it works:
├── Users don't wait long for response
├── Background tasks don't block interactive ones
├── Fair resource distribution
└── Predictable behavior
```

---

## Disadvantages of Round Robin

1. Higher Average Waiting Time

```
Compared to SJF:
├── RR has higher average waiting time
├── Because long jobs get rescheduled repeatedly
├── Short jobs wait behind long jobs' quanta
└── Not optimal for average metrics

Example:
Processes: P1(10ms), P2(4ms), P3(2ms)
Quantum = 2ms

RR Execution:
0-2:   P1 (rem: 8)
2-4:   P2 (rem: 2)
4-6:   P3 (rem: 0, done)
6-8:   P1 (rem: 6)
8-10:  P2 (rem: 0, done)
10-12: P1 (rem: 4)
12-14: P1 (rem: 2)
14-16: P1 (rem: 0, done)

Waiting times:
P1: 16 - 10 = 6ms
P2: 10 - 4 = 6ms
P3: 4 - 2 = 2ms
Average: 4.67ms

SJF (for comparison):
0-2:   P3
2-6:   P2
6-16:  P1

Waiting times:
P1: 6ms, P2: 2ms, P3: 0ms
Average: 2.67ms

RR is 75% worse for average waiting time
```

2. Context Switch Overhead

```
Frequent context switches:
├── Each quantum expiration causes a switch
├── Small quantum → many switches
├── Overhead reduces effective CPU time
└── Throughput decreases

Quantifying overhead:
├── Context switch time: tc
├── Quantum: q
├── Overhead ratio: tc / (q + tc)
├── Effective CPU utilization: q / (q + tc)
└── Want q >> tc

Example: tc = 10μs, q = 1ms
├── Overhead = 10μs / 1010μs ≈ 1%
├── Effective utilization = 99%
└── Acceptable but not ideal

Example: tc = 10μs, q = 100μs
├── Overhead = 10μs / 110μs ≈ 9%
├── Effective utilization = 91%
└── Too much overhead
```

3. Quantum Selection Difficulty

```
No single "best" quantum:
├── Depends on workload
├── Depends on system goals
├── Depends on hardware
├── Must balance response vs. overhead

Consequences of wrong quantum:
├── Too small: excessive overhead
├── Too large: poor response (FCFS-like)
├── Wrong for workload: suboptimal performance
└── Requires tuning and adjustment
```

4. Poor for CPU-Bound Workloads

```
Round Robin favors short jobs:
├── Long CPU-bound jobs rescheduled repeatedly
├── Each reschedule incurs cache effects
├── Total execution time longer than necessary
├── Throughput suffers

Cache effects:
├── Process evicted from cache during wait
├── Must reload data after resumption
├── Cache misses increase
└── Effective speed decreases

Example:
P1 (CPU-bound, 100ms burst), q=10ms
├── P1 runs 10ms, gets 10ms of work done
├── Cache fills with P1's data
├── P1 preempted, other processes run
├── Cache evicted, filled with other data
├── P1 resumes, cache misses, reloads data
└── Effective progress < 10ms per quantum
```

5. Not Suitable for Real-Time Systems

```
Round Robin has no timing guarantees:
├── No deadline awareness
├── No priority support
├── No bounded worst-case response
├── Unsuitable for hard real-time

Real-time requirements:
├── Deterministic response time
├── Deadline-driven scheduling
├── Priority-based preemption
├── RR cannot meet these needs
```

---

## Variants and Extensions

1. **Weighted Round Robin**

Each process gets a quantum proportional to its weight:

```
Weighted Round Robin:

Process weights:
├── P1: weight 3 → gets 3× base quantum
├── P2: weight 2 → gets 2× base quantum
├── P3: weight 1 → gets 1× base quantum

Base quantum = 5ms
├── P1 gets 15ms per turn
├── P2 gets 10ms per turn
├── P3 gets 5ms per turn

Example timeline:
0-15:  P1
15-25: P2
25-30: P3
30-45: P1
45-55: P2
55-60: P3
...

Use case: Proportional share scheduling
├── Each process gets CPU proportional to weight
├── Useful for service differentiation
└── Basis for proportional share schedulers
```

2. **Deficit Round Robin**

Improves weighted RR for variable-size packets/jobs:

```
Deficit Round Robin:

Each process has a "deficit counter":
├── Initially 0
├── Each round: deficit += quantum
├── Process runs if deficit ≥ burst
├── deficit -= actual_burst
├── If process blocks, deficit carries over
└── Ensures long-term fairness

Example:
P1: quantum=10, deficit=0
P2: quantum=10, deficit=0

Round 1:
P1 deficit=10, runs for 8ms, deficit=2
P2 deficit=10, runs for 12ms, deficit=-2

Round 2:
P1 deficit=12, runs for 10ms, deficit=2
P2 deficit=8, runs for 6ms, deficit=2

Fairness achieved over time
```

3. **Virtual Round Robin**

Separates processes into "new" and "used" categories:

```
Virtual Round Robin:

Two queues:
├── New processes queue (higher priority)
├── Used processes queue (lower priority)

Behavior:
├── New process runs for first quantum from new queue
├── If not finished, moves to used queue
├── Used queue processes run with smaller quantum
├── Priority given to new processes
└── Reduces response time for short interactive jobs

Advantage: Better for bursty interactive processes
```

4. **Round Robin with Priority**

Combines RR with priority scheduling:

```
Round Robin with Priority:

Multiple priority queues:
├── Each queue uses Round Robin internally
├── Higher priority queues served first
├── Within each queue, processes get equal quanta
└── Combines fairness with priority

Structure:
Priority 0: [P1] → [P2] → [P3]  (RR with quantum q0)
Priority 1: [P4] → [P5] → [P6]  (RR with quantum q1)
Priority 2: [P7] → [P8]         (RR with quantum q2)

Scheduler serves highest non-empty queue
```

5. **Multilevel Feedback Queue (MLFQ)**

Round Robin is often used within MLFQ levels:

```
MLFQ with Round Robin:

Queue 0 (Highest): RR with quantum = 10ms
Queue 1 (Medium): RR with quantum = 20ms
Queue 2 (Lowest): RR with quantum = 40ms

Process behavior:
├── New process starts in Queue 0
├── Uses full quantum → moves to Queue 1
├── Uses full quantum again → moves to Queue 2
├── Blocks for I/O → stays or moves up
└── Combines RR fairness with adaptive priority

This is essentially how many modern OS schedulers work!
```

---

## Comparison with Other Algorithms

### Round Robin vs. FCFS

```
Same workload: P1(10ms), P2(6ms), P3(4ms), P4(2ms)

FCFS:
0        10       16       20  22
|--------|--------|--------|--|
    P1       P2       P3    P4

Response times: P1=0, P2=10, P3=16, P4=20
Average response: 11.5 ms

Round Robin (q=4ms):
0    4    8    12   14   18   20  22
|----|----|----|----|----|----|--|
  P1   P2   P3   P4   P1   P2   P1 P3

Response times: P1=0, P2=4, P3=8, P4=12
Average response: 6 ms

RR provides 48% better average response time
```

### Round Robin vs. SJF

```
Same workload: P1(10ms), P2(6ms), P3(4ms), P4(2ms)

SJF:
0  2  6    12       22
|--|--|----|--------|
  P4 P3  P2     P1

Response times: P4=0, P3=2, P2=6, P1=12
Average response: 5 ms
Average waiting: 4 ms
Average turnaround: 10 ms

Round Robin (q=4ms):
0    4    8    12   14   18   20  22
|----|----|----|----|----|----|--|
  P1   P2   P3   P4   P1   P2   P1 P3

Response times: P1=0, P2=4, P3=8, P4=12
Average response: 6 ms
Average waiting: 6 ms
Average turnaround: 11.5 ms

SJF better for waiting/turnaround
RR better for fairness and predictability
```

### Round Robin vs. Priority Scheduling

```
Same workload with priorities:
P1 (priority 3, low), P2 (priority 1, high), 
P3 (priority 2, medium), P4 (priority 1, high)

Priority Scheduling:
0  2  6    12       22
|--|--|----|--------|
  P2 P4  P3     P1

(P2 and P4 both priority 1, FCFS between them)

Response times: P2=0, P4=2, P3=6, P1=12
Average response: 5 ms
But low priority P1 waited 12ms

Round Robin (q=4ms):
0    4    8    12   14   18   20  22
|----|----|----|----|----|----|--|
  P1   P2   P3   P4   P1   P2   P1 P3

Response times: P1=0, P2=4, P3=8, P4=12
Average response: 6 ms
All processes get service quickly

RR provides fairness, Priority provides importance
```

---

## Implementation Details

### Data Structures

```c
// Round Robin Scheduler Implementation

typedef struct Process {
    int pid;
    int burst_time;
    int remaining_time;
    int arrival_time;
    int state;
    struct Process *next;
} Process;

typedef struct {
    Process *head;
    Process *tail;
    int size;
} ReadyQueue;

typedef struct {
    ReadyQueue ready_queue;
    int time_quantum;
    Process *current_process;
    int current_quantum_remaining;
} RRScheduler;

// Initialize scheduler
void rr_init(RRScheduler *sched, int quantum) {
    sched->ready_queue.head = NULL;
    sched->ready_queue.tail = NULL;
    sched->ready_queue.size = 0;
    sched->time_quantum = quantum;
    sched->current_process = NULL;
    sched->current_quantum_remaining = 0;
}

// Add process to ready queue (at tail)
void rr_enqueue(RRScheduler *sched, Process *proc) {
    proc->next = NULL;
    
    if (sched->ready_queue.tail == NULL) {
        sched->ready_queue.head = proc;
        sched->ready_queue.tail = proc;
    } else {
        sched->ready_queue.tail->next = proc;
        sched->ready_queue.tail = proc;
    }
    sched->ready_queue.size++;
    proc->state = READY;
}

// Remove process from ready queue (from head)
Process* rr_dequeue(RRScheduler *sched) {
    if (sched->ready_queue.head == NULL) {
        return NULL;
    }
    
    Process *proc = sched->ready_queue.head;
    sched->ready_queue.head = proc->next;
    
    if (sched->ready_queue.head == NULL) {
        sched->ready_queue.tail = NULL;
    }
    
    proc->next = NULL;
    sched->ready_queue.size--;
    return proc;
}

// Timer tick handler
void rr_timer_tick(RRScheduler *sched) {
    if (sched->current_process == NULL) {
        // No process running, schedule next
        rr_schedule(sched);
        return;
    }
    
    sched->current_process->remaining_time--;
    sched->current_quantum_remaining--;
    
    if (sched->current_process->remaining_time <= 0) {
        // Process completed
        sched->current_process->state = TERMINATED;
        sched->current_process = NULL;
        rr_schedule(sched);
    } else if (sched->current_quantum_remaining <= 0) {
        // Quantum expired, preempt
        rr_preempt(sched);
    }
}

// Preempt current process
void rr_preempt(RRScheduler *sched) {
    if (sched->current_process != NULL) {
        // Move to tail of ready queue
        rr_enqueue(sched, sched->current_process);
        sched->current_process = NULL;
    }
    rr_schedule(sched);
}

// Schedule next process
void rr_schedule(RRScheduler *sched) {
    if (sched->current_process != NULL) {
        return;  // Already running
    }
    
    Process *next = rr_dequeue(sched);
    if (next != NULL) {
        sched->current_process = next;
        sched->current_quantum_remaining = sched->time_quantum;
        next->state = RUNNING;
        // Context switch to next process
    }
}

// Handle process blocking (I/O wait)
void rr_block(RRScheduler *sched) {
    if (sched->current_process != NULL) {
        sched->current_process->state = WAITING;
        sched->current_process = NULL;
        rr_schedule(sched);
    }
}

// Handle I/O completion
void rr_unblock(RRScheduler *sched, Process *proc) {
    rr_enqueue(sched, proc);
    
    // If no process running, schedule
    if (sched->current_process == NULL) {
        rr_schedule(sched);
    }
}
```

### Handling Edge Cases

```
Edge Cases in Round Robin:

1. Process Finishes Before Quantum Expires
   ├── Remove from system
   ├── Schedule next process immediately
   └── Don't waste remaining quantum time

2. Process Blocks for I/O
   ├── Move to wait queue
   ├── Schedule next process
   ├── Don't count remaining quantum
   └── Process returns to ready queue tail when I/O completes

3. New Process Arrives
   ├── Add to tail of ready queue
   ├── Don't preempt current process (unless quantum expired)
   └── Wait for its turn

4. Empty Ready Queue
   ├── CPU idle
   ├── Wait for new process arrival or I/O completion
   └── Enter low-power state

5. Quantum Expiration During System Call
   ├── Kernel handles system call
   ├── Quantum check on return to user mode
   └── Or preempt in kernel if kernel preemptible

6. Multiple Ready Queues
   ├── Each priority level has its own RR queue
   ├── Scheduler selects from highest non-empty queue
   └── RR within each level
```

---

## Practical Considerations

### Quantum Selection in Real Systems

```
Real-World Quantum Values:

Traditional Unix:
├── Quantum: 100ms
├── Balanced for batch and interactive
└── Simple implementation

Linux (before CFS):
├── Quantum: 10-200ms (dynamic)
├── Based on process priority
├── Higher priority → larger quantum
└── Adjusted by nice value

Linux (CFS):
├── No fixed quantum
├── Uses virtual runtime
├── Target latency: 6-24ms
├── Minimum granularity: 0.75-6ms
└── Adapts to number of running processes

Windows:
├── Quantum: 15-30ms (varies)
├── Short quantum for foreground
├── Long quantum for background
├── Adjusts based on system state
└── Client vs. server configurations differ

macOS:
├── Variable quantum
├── Based on priority and QoS
├── Interactive processes get preference
└── Power-aware adjustments
```

### Interactive vs. Batch Tuning

```
Tuning Round Robin for Different Workloads:

Interactive System:
├── Small quantum (10-20ms)
├── Prioritize response time
├── Accept higher overhead
├── Optimize for user perception
└── Typical: Desktop, mobile

Batch System:
├── Large quantum (100-1000ms)
├── Prioritize throughput
├── Minimize context switches
├── Optimize for total completion time
└── Typical: Servers, clusters

Mixed System:
├── Medium quantum (20-50ms)
├── Balance response and throughput
├── Consider using MLFQ instead
├── Adaptive quantum adjustment
└── Typical: General-purpose servers
```

---

## Common Misconceptions

**Misconception 1:** "Round Robin is always fair"

**Reality:** Round Robin is fair in terms of CPU time allocation, but not in terms of outcomes. A process that arrives just after its turn waits longer than one arriving just before. Also, processes that block frequently (I/O-bound) may get more total CPU time than CPU-bound processes because they return to the queue quickly.

**Misconception 2:** "Smaller quantum is always better"

**Reality:** Smaller quantum improves response time but at the cost of increased context switch overhead. There's an optimal quantum that balances these factors. For very small quanta, the system spends more time switching than executing.

**Misconception 3:** "Round Robin has no overhead"

**Reality:** Round Robin has significant overhead compared to non-preemptive algorithms due to frequent context switches, timer interrupts, and cache effects. The overhead is the price of preemption and fairness.

**Misconception 4:** "Round Robin treats all processes equally"

**Reality:** Basic Round Robin treats all processes equally in terms of CPU time per turn. However, this can be unfair in other ways—a process that needs 1ms gets the same quantum as one that needs 100ms. Variants like Weighted RR address this.

**Misconception 5:** "Round Robin is obsolete"

**Reality:** Round Robin principles are used in virtually every modern general-purpose OS, often within multilevel feedback queue systems. While pure RR is rarely used, its core idea—time-sliced preemptive scheduling—is fundamental to modern computing.

---

## Hands-On Exercises

### Exercise 1: Basic Round Robin

**Problem:** Four processes arrive at time 0:

| Process | Burst |
|---------|-------|
| P1      | 10    |
| P2      | 6     |
| P3      | 4     |
| P4      | 8     |

**Quantum = 3ms. Calculate average waiting time, turnaround time, and response time.**

**Solution:**

```
Timeline:
0-3:   P1 (rem: 7)
3-6:   P2 (rem: 3)
6-9:   P3 (rem: 1)
9-12:  P4 (rem: 5)
12-15: P1 (rem: 4)
15-18: P2 (rem: 0, done)
18-21: P3 (rem: 0, done) - only needed 1ms, runs 18-19
19-22: P4 (rem: 2)
22-25: P1 (rem: 1)
25-27: P4 (rem: 0, done) - needed 2ms, runs 25-27
27-28: P1 (rem: 0, done) - needed 1ms

Wait, let me redo carefully:

Time 0: Queue: [P1, P2, P3, P4]
0-3: P1 runs (rem 7). Queue after: [P2, P3, P4, P1]
3-6: P2 runs (rem 3). Queue after: [P3, P4, P1, P2]
6-9: P3 runs (rem 1). Queue after: [P4, P1, P2, P3]
9-12: P4 runs (rem 5). Queue after: [P1, P2, P4]
12-15: P1 runs (rem 4). Queue after: [P2, P3]
15-18: P2 runs (rem 0, done). Queue after: [P3]
18-19: P3 runs (rem 0, done) - only needed 1ms. Queue after: []
```

**Gantt Chart:**
```
0  3  6  9  12 15 18
|--|--|--|--|---|---|
  P1 P2 P3 P4 P1
```

**Completion times:**
- P1: 28
- P2: 18
- P3: 19
- P4: 27

**Turnaround:**
- P1: 28
- P2: 18
- P3: 19
- P4: 27

**Average:** (28+18+19+27)/4 = 23 ms

**Waiting = Turnaround - Burst:**
- P1: 28-10 = 18
- P2: 18-6 = 12
- P3: 19-4 = 15
- P4: 27-8 = 19

**Average:** (18+12+15+19)/4 = 16 ms

**Response (first run):**
- P1: 0
- P2: 3
- P3: 6
- P4: 9

**Average:** (0+3+6+9)/4 = 4.5 ms
```

### Exercise 2: Effect of Quantum Size

**Problem:** Compare quantum = 2ms, 4ms, and 8ms for the same workload as Exercise 1. Which is best?

**Solution:**

```
Quantum = 2ms:
Response: P1=0, P2=2, P3=4, P4=6 → Avg = 3 ms
Context switches: More (need to calculate)
Waiting: Higher (more interleaving)

Quantum = 4ms:
Response: P1=0, P2=4, P3=8, P4=12 → Avg = 6 ms
Context switches: Fewer
Waiting: Lower than q=2

Quantum = 8ms:
Response: P1=0, P2=8, P3=14, P4=18 → Avg = 10 ms
Context switches: Much fewer
Waiting: Lowest of the three
But response time is poor

Best choice depends on goals:
├── Interactive system: q=2 or 4
├── Batch system: q=8 or larger
└── Balanced: q=4
```

### Exercise 3: Round Robin with I/O

**Problem:** Two processes:

- P1: CPU burst 6ms, I/O 10ms, CPU burst 6ms
- P2: CPU burst 12ms

**Quantum = 4ms. Draw the execution timeline.**

**Solution:**

```
Time 0: Both arrive. Queue: [P1, P2]

0-4: P1 runs (CPU rem: 2)
4: P1's quantum expires. Queue: [P2, P1]

4-8: P2 runs (CPU rem: 8)
8: P2's quantum expires. Queue: [P1, P2]

8-10: P1 runs (CPU rem: 0, first burst done)
10: P1 blocks for I/O (10ms). Queue: [P2]
    P1 will be ready at 20ms

10-14: P2 runs (CPU rem: 4)
14: P2's quantum expires. Queue: [P2] (only process)

14-18: P2 runs (CPU rem: 0, done)
18: P2 completes.

20: P1's I/O completes. P1 joins ready queue.
20-24: P1 runs (CPU rem: 2)
24: P1's quantum expires. Queue: [P1]

24-26: P1 runs (CPU rem: 0, done)
26: P1 completes.

Gantt Chart:
0    4    8   10   14   18   20   24  26
|----|----|---|----|----|----|----|--|
  P1   P2   P1   P2   P2   IDLE P1   P1
       (P1 blocks at 10 for I/O)

Timeline notes:
- CPU idle from 18-20 (P1 waiting for I/O, P2 done)
- P1's I/O overlaps with P2's execution (good!)

P1: 
├── Turnaround: 26 ms
├── CPU time: 12 ms
├── I/O wait: 10 ms
└── Waiting: 26 - 12 - 10 = 4 ms

P2:
├── Turnaround: 18 ms
├── CPU time: 12 ms
└── Waiting: 18 - 12 = 6 ms

Average Waiting: 5 ms
CPU Utilization: 24/26 = 92.3%
```

---

## Summary

Round Robin scheduling is the cornerstone of modern preemptive time-sharing systems. By giving each process a small, fixed time quantum in rotation, it achieves fairness and excellent response time—critical properties for interactive systems. While it has higher overhead and worse average waiting time than SJF, its fairness and predictability make it the preferred choice for general-purpose operating systems.

### Key Points

1. Round Robin assigns fixed time quanta to processes in circular order.
2. The time quantum is the critical parameter—balancing response time against context switch overhead.
3. Fairness and response time are Round Robin's primary strengths.
4. Higher average waiting time compared to SJF is its main weakness.
5. Context switch overhead limits how small the quantum can be.
6. Variants exist (Weighted RR, Deficit RR, Virtual RR) for specialized needs.
7. Modern OSes use RR principles within multilevel feedback queue systems.
8. Quantum selection depends on system type and workload characteristics.

---

## Key Takeaways

1. Round Robin is the foundation of time-sharing—its principles underpin modern scheduling.
2. The time quantum must balance response time and context switch overhead.
3. Fairness comes at a cost—RR has higher waiting times than SJF.
4. No starvation occurs in Round Robin—every process gets service.
5. Response time is bounded by (n-1) × q, making the system feel responsive.
6. Small quantum → better response but more overhead; large quantum → less overhead but worse response.
7. Typical quantum values range from 10-100ms in modern systems.
8. Understanding Round Robin is essential for understanding modern scheduling algorithms.

---

## Further Reading

- FCFS.md: The non-preemptive baseline algorithm
- SJF.md: Optimal for average waiting time but with starvation risk
- Priority-Scheduling.md: Adding importance-based selection
- Multilevel-Queues.md: Combining RR with other algorithms
- Real-World-Scheduling.md: How Linux CFS and Windows schedulers use RR concepts

