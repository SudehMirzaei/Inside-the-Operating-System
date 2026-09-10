# Priority Scheduling

## Introduction

Priority Scheduling is a CPU scheduling algorithm that selects processes based on their assigned priority. Instead of treating all processes equally (as in Round Robin) or prioritizing by burst time (as in SJF), Priority Scheduling introduces the concept of importance: some processes are simply more critical than others and deserve preferential access to the CPU.

This scheduling approach mirrors real-world decision-making. In a hospital emergency room, a patient with a heart attack is treated before someone with a sprained ankle—not because they arrived first or need less treatment, but because their condition is more critical. Similarly, in an operating system, a process handling a system interrupt or a real-time task should run before a background indexing job.

Priority Scheduling is used in various forms across virtually all modern operating systems. It's a fundamental component of real-time systems, server workloads, and even general-purpose OSes (often combined with other algorithms like Round Robin within multilevel queue structures).

---

## How Priority Scheduling Works

### The Basic Principle

Priority Scheduling operates on a straightforward principle: each process is assigned a priority, and the CPU is allocated to the highest-priority ready process.

```
Priority Scheduling Algorithm:

1. Each process is assigned a priority value
2. The ready queue is maintained in priority order
3. When the CPU becomes available:
   ├── Select the process with highest priority
   ├── (If tie, use FCFS or Round Robin among equal priorities)
   └── Dispatch selected process
4. Non-preemptive version: process runs until it blocks or completes
5. Preemptive version: higher-priority arrival preempts current process
```

### Priority Representation

Priorities can be represented in two conventions:

```
Convention 1: Lower number = Higher priority (Unix/Linux style)

Priority 0: Highest (most important)
Priority 1: High
Priority 2: Medium
...
Priority 20: Low
Priority 39: Lowest (least important)

This is the convention used in Unix/Linux "nice" values.

Convention 2: Higher number = Higher priority (Windows style)

Priority 31: Highest (real-time critical)
Priority 15: High
Priority 8: Normal
Priority 1: Lowest (idle)

This is the convention used in Windows priority classes.
```

We'll use lower number = higher priority throughout this document to align with Unix conventions (and clarify when we switch).

### Visual Representation

```
Priority Queue:

Highest Priority
    ↓
┌─────────────────────┐
│ Queue 0: [P1] [P2]  │  ← System processes
├─────────────────────┤
│ Queue 1: [P3] [P4]  │  ← Interactive processes
├─────────────────────┤
│ Queue 2: [P5] [P6]  │  ← Batch processes
├─────────────────────┤
│ Queue 3: [P7] [P8]  │  ← Background processes
└─────────────────────┘
    ↑
Lowest Priority

Scheduler always selects from highest non-empty priority level
```

### Key Characteristics

| Property                             | Value                             |
|-------------------------------------|-----------------------------------|
| Preemptive?                         | Can be either                     |
| Starvation possible?                | Yes (without aging)               |
| Requires burst time knowledge?      | No                                |
| Requires priority knowledge?         | Yes                               |
| Implementation complexity            | Low-Medium                        |
| Fairness                            | Depends on priority assignment     |
| Response time                       | Excellent for high priority        |
| Real-time capable?                  | Yes (with proper configuration)    |

---

## Types of Priority Scheduling

### 1. Non-Preemptive Priority Scheduling

Once a process starts, it runs to completion (or until it blocks):

```
Non-Preemptive Priority Scheduling:

Behavior:
├── Highest priority process selected at scheduling points
├── Runs until completion or I/O block
├── New higher-priority processes wait
├── Lower overhead (fewer context switches)
└── Simpler to implement

Scheduling points:
├── Process completes
├── Process blocks for I/O
├── Process voluntarily yields
└── NOT on new process arrival

Example:
Time 0: P1 (priority 3) arrives, starts running
Time 1: P2 (priority 1, higher) arrives
        P1 continues (non-preemptive)
Time 5: P1 completes
        P2 runs now
```

### 2. Preemptive Priority Scheduling

A higher-priority process can interrupt a running lower-priority process:

```
Preemptive Priority Scheduling:

Behavior:
├── Highest priority process always runs
├── New higher-priority process preempts current
├── Higher overhead (more context switches)
├── Better for real-time systems
└── Standard in modern OSes

Scheduling points:
├── Process completes
├── Process blocks for I/O
├── Process voluntarily yields
├── New process arrives (higher priority)
├── I/O completion (higher priority wake-up)

Example:
Time 0: P1 (priority 3) arrives, starts running
Time 1: P2 (priority 1, higher) arrives
        P2 preempts P1 immediately
Time 3: P2 completes
        P1 resumes
```

### Comparison of Approaches

| Aspect                      | Non-Preemptive   | Preemptive        |
|-----------------------------|------------------|-------------------|
| Implementation               | Simpler          | More complex       |
| Overhead                     | Lower            | Higher             |
| Responsiveness               | Poor for urgent   | Excellent          |
| Context Switches             | Fewer            | More               |
| Real-Time Suitable           | No (mostly)      | Yes                |
| Priority Inversion           | Less problematic  | Can occur          |
| Typical Use                 | Batch systems      | Interactive, real-time |

---

## Priority Assignment

How are priorities assigned? Several approaches exist:

### 1. Static Priority Assignment

Priorities are fixed at process creation and never change:

```
Static Assignment Methods:

By Process Type:
├── Kernel processes: Priority 0-5
├── Real-time processes: Priority 6-10
├── System services: Priority 11-15
├── Interactive users: Priority 16-20
├── Batch processes: Priority 21-30
└── Background tasks: Priority 31-39

By User/Group:
├── Root/admin: Higher priority
├── Regular users: Normal priority
├── Guest users: Lower priority
└── Per-user quota systems

By Application Type:
├── Audio/video playback: High priority
├── Games: High priority
├── Compilers: Normal priority
├── Backup software: Low priority
└── System maintenance: Lowest priority

Advantages:
├── Predictable
├── Simple to implement
├── Easy to understand

Disadvantages:
├── Cannot adapt to changing conditions
├── Misclassification permanent
├── Potential for starvation
```

### 2. Dynamic Priority Assignment

Priorities can change during process execution:

```
Dynamic Assignment Triggers:

Process Behavior:
├── I/O-bound → increase priority (was waiting, needs quick service)
├── CPU-bound → decrease priority (has been using CPU heavily)
├── Interactive → boost priority (user waiting)
└── Background → lower priority (no user impact)

Time-Based:
├── Aging: increase priority with waiting time
├── CPU usage: decrease priority with CPU consumption
├── I/O wait: increase priority when woken from I/O
└── Recent behavior: weight recent history

System State:
├── System load: adjust based on CPU utilization
├── I/O load: prioritize processes that reduce I/O queues
├── Memory pressure: prioritize memory-light processes
└── Power state: adjust for power efficiency

User Actions:
├── Foreground/background toggle
├── Priority adjustment (nice command)
├── Real-time request
└── Resource limits
```

### 3. Hybrid Approach

Most modern systems use a combination:

```
Hybrid Priority Assignment (Unix-like):

Base Priority:
├── Determined by process type and user
├── Static component for predictability
└── Acts as anchor for dynamic adjustments

Dynamic Adjustment:
├── Based on CPU usage history
├── Uses "nice" value as modifier
├── Adjusts within a range around base priority
└── Cannot exceed boundaries

Example (Linux):
├── Base priority: 0-139 (0 highest, 139 lowest)
├── Real-time: 0-99
├── User processes: 100-139
├── Nice value: -20 to +19 (affects 100-139 range)
├── Dynamic adjustment: ±5 based on behavior
└── Final priority used for scheduling
```

---

## Detailed Examples

### Example 1: Basic Non-Preemptive Priority Scheduling

Scenario: Five processes arrive at time 0:

| Process | Burst Time | Priority |
|---------|------------|----------|
| P1      | 10 ms     | 3        |
| P2      | 1 ms      | 1 (highest) |
| P3      | 2 ms      | 4        |
| P4      | 1 ms      | 5 (lowest)  |
| P5      | 5 ms      | 2        |

Execution (lower number = higher priority):

```
Sorted by priority:
P2 (1), P5 (2), P1 (3), P3 (4), P4 (5)

Gantt Chart:
0  1     6        16   18   19
|--|-----|--------|----|----|
  P2   P5     P1     P3   P4

Timeline:
0-1:   P2 runs (highest priority)
1-6:   P5 runs (second highest)
6-16:  P1 runs (third highest)
16-18: P3 runs (fourth)
18-19: P4 runs (lowest)
```

Calculations:

```
Waiting Times:
P2: 0
P5: 1
P1: 6
P3: 16
P4: 18

Average Waiting = (0 + 1 + 6 + 16 + 18) / 5 = 8.2 ms

Turnaround Times:
P2: 1 - 0 = 1 ms
P5: 6 - 0 = 6 ms
P1: 16 - 0 = 16 ms
P3: 18 - 0 = 18 ms
P4: 19 - 0 = 19 ms

Average Turnaround = (1 + 6 + 16 + 18 + 19) / 5 = 12 ms
```

### Example 2: Preemptive Priority Scheduling

Scenario: Processes arrive at different times:

| Process | Arrival | Burst | Priority |
|---------|---------|-------|----------|
| P1      | 0       | 10    | 3        |
| P2      | 2       | 4     | 1 (highest) |
| P3      | 4       | 2     | 2        |
| P4      | 6       | 3     | 4        |
| P5      | 8       | 1     | 5        |

Execution (preemptive):

```
Timeline:
0-2:   P1 runs (only process). Remaining: 8ms
2:     P2 arrives (priority 1, higher than P1's priority 3)
       P2 preempts P1
2-4:   P2 runs (remaining: 2ms)
4:     P3 arrives (priority 2, higher than P2's priority 1? No, P2 is higher)
       P2 continues
4-6:   P2 runs (completes at 6ms)
6:     P2 done. P4 arrives (priority 4)
       Ready: P1 (pri 3, rem 8), P3 (pri 2, burst 2), P4 (pri 4, burst 3)
       Select P3 (highest priority: 2)
6-8:   P3 runs (completes at 8ms)
8:     P3 done. P5 arrives (priority 5)
       Ready: P1 (pri 3, rem 8), P4 (pri 4, burst 3), P5 (pri 5, burst 1)
       Select P1 (highest priority: 3)
8-16:  P1 runs (completes at 16ms)
16:    P1 done
       Ready: P4 (pri 4), P5 (pri 5)
       Select P4 (higher priority: 4)
16-19: P4 runs (completes at 19ms)
19:    P4 done
       Select P5
19-20: P5 runs (completes at 20ms)

Gantt Chart:
0  2  4  6  8              16   19 20
|--|--|--|--|--------------|----|--|
  P1 P2 P2 P3      P1         P4  P5

Wait, P2 runs 2-6 (4ms total):
0  2  4  6  8              16   19 20
|--|--|--|--|--------------|----|--|
  P1 P2 P2 P3      P1         P4  P5
```

Calculations:

```
Completion Times:
P1: 16 ms
P2: 6 ms
P3: 8 ms
P4: 19 ms
P5: 20 ms

Turnaround Times:
P1: 16 - 0 = 16 ms
P2: 6 - 2 = 4 ms
P3: 8 - 4 = 4 ms
P4: 19 - 6 = 13 ms
P5: 20 - 8 = 12 ms

Average Turnaround = (16 + 4 + 4 + 13 + 12) / 5 = 9.8 ms

Waiting Times:
P1: (2-0) + (8-6) = 2 + 2 = 4 ms
    (waited 0-2? No, ran 0-2. Preempted at 2, resumed at 8)
    Waiting = 8 - 2 = 6 ms

P2: 0 ms (arrived at 2, ran immediately)
P3: 6 - 4 = 2 ms (arrived at 4, ran at 6)
P4: 16 - 6 = 10 ms (arrived at 6, ran at 16)
P5: 19 - 8 = 11 ms (arrived at 8, ran at 19)

Average Waiting = (6 + 0 + 2 + 10 + 11) / 5 = 5.8 ms

Response Times:
P1: 0 - 0 = 0 ms
P2: 2 - 2 = 0 ms
P3: 6 - 4 = 2 ms
P4: 16 - 6 = 10 ms
P5: 19 - 8 = 11 ms

Average Response = (0 + 0 + 2 + 10 + 11) / 5 = 4.6 ms
```

### Example 3: Priority Scheduling with I/O

Scenario: Processes with I/O operations:

| Process | CPU Burst 1 | I/O Time | CPU Burst 2 | Priority |
|---------|--------------|----------|--------------|----------|
| P1      | 5 ms        | 10 ms    | 5 ms        | 2        |
| P2      | 3 ms        | 5 ms     | 3 ms        | 1 (highest) |
| P3      | 8 ms        | 0 ms     | 0 ms        | 3        |

Execution (preemptive):

```
Timeline:
0:     P1 and P2 arrive. P1 has priority 2, P2 has priority 1.
       P2 has higher priority, runs first.

0-3:   P2 runs (first CPU burst, 3ms)
3:     P2 blocks for I/O (10ms? No, 5ms). P2 will be ready at 8ms.
       Ready: P1 (priority 2, burst 5), P3 arrives at... let's say P3 arrives at time 3.
       Actually, let's say P3 arrives at time 0 too.

Let me redo with P3 arriving at time 0:

0:     All three arrive. P2 has highest priority (1).
0-3:   P2 runs (first CPU burst done)
3:     P2 blocks for I/O (5ms), ready at 8ms
       Ready: P1 (pri 2), P3 (pri 3)
       P1 has higher priority

3-8:   P1 runs (first CPU burst done at 8ms)
8:     P1 blocks for I/O (10ms), ready at 18ms
       P2's I/O completed at 8ms! P2 is ready.
       Ready: P2 (pri 1, second CPU burst 3ms), P3 (pri 3, burst 8ms)
       P2 has highest priority

8-11:  P2 runs (second CPU burst done, completes at 11ms)
11:    P2 done
       Ready: P3 (pri 3, burst 8ms)

11-19: P3 runs (completes at 19ms)
19:    P3 done
       P1's I/O completed at 18ms. P1 is ready.

19-24: P1 runs (second CPU burst, 5ms, completes at 24ms)
24:    P1 done

Gantt Chart:
0  3  8  11       19        24
|--|--|--|--------|---------|
  P2 P1 P2    P3       P1

Wait, P1 runs 3-8, P2 runs 8-11, P3 runs 11-19, P1 runs 19-24.

Gantt Chart:
0  3     8  11       19        24
|--|-----|--|--------|---------|
  P2   P1  P2    P3       P1
```

Calculations:

```
P1: 
├── Completion: 24 ms
├── Turnaround: 24 - 0 = 24 ms
├── CPU time: 10 ms
├── I/O time: 10 ms
└── Waiting: 24 - 10 - 10 = 4 ms

P2:
├── Completion: 11 ms
├── Turnaround: 11 - 0 = 11 ms
├── CPU time: 6 ms
├── I/O time: 5 ms
└── Waiting: 11 - 6 - 5 = 0 ms

P3:
├── Completion: 19 ms
├── Turnaround: 19 - 0 = 19 ms
├── CPU time: 8 ms
└── Waiting: 19 - 8 = 11 ms

Average Waiting = (4 + 0 + 11) / 3 = 5 ms
Average Turnaround = (24 + 11 + 19) / 3 = 18 ms
```

---

## The Starvation Problem

The most significant issue with Priority Scheduling is starvation (also called indefinite blocking):

### What is Starvation?

```
Starvation Scenario:

System state:
├── Continuous stream of high-priority processes arriving
├── Low-priority process waits in queue
├── Higher priority processes always selected first
├── Low-priority process never gets CPU
└── "Starved" of CPU time

Example:
High-priority processes arrive every 5ms, each needing 10ms CPU
Low-priority process needs 100ms CPU, has been waiting

Timeline:
0-10:   High-priority job 1
5-15:   High-priority job 2 (arrived at 5, waits until 10)
10-20:  High-priority job 3
...

The low-priority process never runs!
```

### Real-World Starvation Scenarios

```
Scenario 1: System Monitoring
├── Monitoring daemon runs at highest priority
├── Wakes every 10ms to check system state
├── Takes 2ms CPU each time (20% CPU total)
├── Background backup process (low priority) never runs
└── Backup never completes

Scenario 2: Web Server Under Load
├── User requests (high priority) arrive continuously
├── Each request needs quick processing
├── Log rotation (low priority) waits indefinitely
├── Log files grow unbounded
└── Eventually disk fills up

Scenario 3: Real-Time System with Periodic Tasks
├── High-priority periodic task every 5ms
├── Task execution time: 3ms
├── CPU utilization: 60%
├── Low-priority task needs 50% CPU
├── Cannot meet 50% requirement (only 40% available)
└── Low-priority task starves
```

### The Aging Solution

Aging gradually increases the priority of waiting processes:

```
Aging Algorithm:

1. Each process has a priority and an "age" counter
2. Age increments each time the process is passed over
3. Priority is adjusted by age:
   Effective Priority = Base Priority - Age × Aging Factor
   (lower number = higher priority)
4. Eventually, even lowest priority process becomes highest
5. Guarantees every process eventually runs

Example:
Base priorities: P1=1, P2=2, P3=3
Aging factor: 0.1 per time unit

Time 0: P1=1, P2=2, P3=3 → Select P1
Time 1: P1=1, P2=1.9, P3=2.8 → Select P1
Time 10: P1=1, P2=1, P3=2 → Tie between P1 and P2
Time 20: P1=1, P2=0, P3=1 → Select P2
Time 30: P1=1, P2=0, P3=0 → Tie, select P1 or P2
Time 40: P1=1, P2=0, P3=-1 → Select P3!

Every process eventually runs.
```

### Other Solutions to Starvation

```
1. Aging (most common)
   ├── Gradually increase priority of waiting processes
   ├── Guarantees eventual service
   ├── Requires tracking wait time
   └── Configurable aging rate

2. Priority Inheritance
   ├── Low-priority process holding lock inherits high priority
   ├── Prevents priority inversion
   ├── Common in real-time systems
   └── Complex to implement correctly

3. Priority Ceiling
   ├── Each resource has a priority ceiling
   ├── Process holding resource runs at ceiling priority
   ├── Prevents deadlock and inversion
   └── Requires pre-declared resource priorities

4. Time Quotas
   ├── Each priority level gets minimum CPU time
   ├── Even lowest priority gets some service
   ├── Uses time-slicing at system level
   └── Common in fair-share systems

5. Guaranteed Minimum Service
   ├── Reserve CPU percentage for each priority class
   ├── Higher classes get more, but all get some
   ├── Prevents complete starvation
   └── Used in proportional share schedulers
```

---

## Priority Inversion

A related problem to starvation is priority inversion, where a high-priority process is indirectly blocked by a low-priority process.

### The Classic Scenario

```
Priority Inversion Scenario:

Processes:
├── H: High priority
├── M: Medium priority
└── L: Low priority

Shared resource: Mutex lock M

Timeline:
1. L acquires lock M
2. H needs lock M, blocks waiting for L
3. M becomes ready (higher priority than L)
4. M preempts L
5. H waits for L, L waits for M
6. H effectively blocked by M (priority inversion!)

Timeline visualization:

L: [===holding lock===][blocked by M][...]
M:                     [===running===][...]
H: [blocked on lock][blocked][blocked][===runs===]

H (highest priority) is delayed by M (medium priority)
due to L (lowest priority) holding a shared lock.
```

### Real-World Consequence: Mars Pathfinder

```
Mars Pathfinder (1997):

Problem:
├── Lander experienced repeated system resets
├── Watchdog timer was resetting system
├── High-priority task (bus management) missed deadlines
├── Root cause: priority inversion

Details:
├── Low-priority task held information bus lock
├── High-priority task (bus management) needed lock
├── Medium-priority task (communications) preempted low
├── High-priority task waited for medium to finish
├── Watchdog timer expired → system reset

Solution:
├── Jet Propulsion Laboratory enabled priority inheritance
├── Low-priority task inherited high priority
├── Could not be preempted by medium task
├── High-priority task met its deadline
└── System stabilized

This incident made priority inversion famous!
```

### Solutions to Priority Inversion

```
1. Priority Inheritance Protocol
   ├── Process holding lock inherits priority of highest waiter
   ├── L inherits H's priority, cannot be preempted by M
   ├── When lock released, priority returns to original
   └── Simple but can cause chain blocking

2. Priority Ceiling Protocol
   ├── Each lock has a ceiling priority (highest user's priority)
   ├── Process holding lock runs at ceiling priority
   ├── Prevents deadlock and limits inversion
   └── Requires knowing max priority in advance

3. Highest Locker Protocol
   ├── Process holding lock runs at highest possible priority
   ├── Very conservative
   ├── Prevents inversion completely
   └── May over-prioritize lock holders

4. Lock-Free Algorithms
   ├── Avoid locks entirely
   ├── Use atomic operations
   ├── No priority inversion possible
   └── Complex to implement correctly
```

---

## Priority Scheduling in Real Systems

### Unix/Linux Nice Values

```
Unix Nice Value System:

Range: -20 (highest priority) to +19 (lowest)
Default: 0

Relationship:
├── Lower nice = higher priority = more CPU
├── "Nice" name: higher nice = nicer to other users
├── Only root can set negative nice values
├── Regular users can only increase nice (lower priority)

Example:
Process A: nice = -10 (high priority)
Process B: nice = 0 (normal)
Process C: nice = 10 (low priority)

Process A gets more CPU than B, which gets more than C.

Priority calculation:
├── Static priority = 120 + nice (for normal processes)
├── Range: 100-139 (100 = nice -20, 139 = nice +19)
├── Real-time: 0-99
└── Used in traditional Unix scheduler

Modern Linux (CFS):
├── Nice value affects CPU time allocation
├── Weight proportional to priority
├── Uses virtual runtime for fairness
└── No strict priority preemption
```

### Windows Priority Classes

```
Windows Priority Classes:

6 Priority Classes:
├── REALTIME_PRIORITY_CLASS (24) - Highest
├── HIGH_PRIORITY_CLASS (13) - Important tasks
├── ABOVE_NORMAL_PRIORITY_CLASS (10) - Slightly elevated
├── NORMAL_PRIORITY_CLASS (8) - Default
├── BELOW_NORMAL_PRIORITY_CLASS (6) - Slightly lower
└── IDLE_PRIORITY_CLASS (4) - Lowest

Plus 7 relative priorities within each class:
├── TIME_CRITICAL (+15 or +2 within class)
├── HIGHEST (+2 within class)
├── ABOVE_NORMAL (+1 within class)
├── NORMAL (0 within class)
├── BELOW_NORMAL (-1 within class)
├── LOWEST (-2 within class)
└── IDLE (-15 or special)

Result: 32 total priority levels (0-31)

Dynamic Adjustment:
├── Foreground window gets priority boost
├── I/O completion boosts priority
├── CPU starvation causes priority boost
├── Anti-starvation: after 4 seconds, boost to 15
└── Boosts decay over time
```

### Real-Time Priority Scheduling

```
Real-Time Priority Scheduling:

Requirements:
├── Deterministic response time
├── Deadline guarantees
├── No starvation of critical tasks
├── Priority strictly enforced
└── Preemption always enabled

Common algorithms:
├── Rate Monotonic (static priority)
│   ├── Shorter period → higher priority
│   ├── Optimal for periodic tasks
│   └── Proven schedulability bound
│
├── Earliest Deadline First (dynamic priority)
│   ├── Nearest deadline → highest priority
│   ├── Optimal for any workload (if schedulable)
│   ├── Higher overhead
│   └── Requires deadline knowledge
│
└── Priority-based with Inheritance
    ├── Static priorities with inheritance
    ├── Prevents priority inversion
    └── Used in POSIX real-time systems

POSIX Real-Time Scheduling:
├── SCHED_FIFO: Fixed priority, no time slicing
├── SCHED_RR: Fixed priority, round robin within priority
├── SCHED_OTHER: Normal time-sharing
└── Priority range: 1-99 (higher = more important)
```

---

## Comparison with Other Algorithms

### Priority vs. FCFS

```
Same workload:
P1 (priority 1, highest, burst 10ms)
P2 (priority 2, burst 4ms)
P3 (priority 3, lowest, burst 6ms)

FCFS (arrival order: P1, P2, P3):
0        10       14       20
|--------|--------|--------|
    P1       P2       P3

Priority (with preemption):
0        10       14       20
|--------|--------|--------|
    P1       P2       P3

Same here because P1 has both highest priority and earliest arrival.

If arrival order was P3, P2, P1:
FCFS:
0    6        10       20
|----|--------|--------|
  P3     P2       P1

Priority:
0        10       14       20
|--------|--------|--------|
    P1       P2       P3

Different orders based on priority vs. arrival.
```

### Priority vs. SJF

```
Priority and SJF both have starvation potential:
├── SJF: Long jobs starve
├── Priority: Low-priority jobs starve
├── Both need aging to prevent starvation
└── Priority more flexible (importance-based)

Relationship:
├── SJF can be modeled as priority scheduling
├── Priority = 1 / burst_time
├── Shorter burst → higher priority
└── Both minimize specific metrics

Key difference:
├── SJF optimizes average waiting time
├── Priority optimizes importance-based service
├── SJF cannot express "urgent" vs. "routine"
└── Priority cannot express "short" vs. "long"
```

### Priority vs. Round Robin

```
Same workload:
P1 (priority 1, burst 10ms)
P2 (priority 2, burst 10ms)
P3 (priority 3, burst 10ms)

Round Robin (q=3ms):
0  3  6  9  12 15 18 21 24 27 30
|--|--|--|--|--|--|--|--|--|--|
  P1 P2 P3 P1 P2 P3 P1 P2 P3 P1
All processes finish around same time.

Priority (preemptive):
0                  10                  20       30
|------------------|-------------------|--------|
         P1                  P2            P3

P1 finishes at 10ms, P2 at 20ms, P3 at 30ms.

Different philosophies:
├── RR: Fairness, equal treatment
├── Priority: Importance, differentiated service
└── Choice depends on system requirements
```

### Summary Comparison Table

| Aspect               | FCFS   | SJF   | RR    | Priority                 |
|----------------------|--------|-------|-------|---------------------------|
| Preemption            | No     | Optional | Yes   | Optional                  |
| Starvation            | No     | Yes (long) | No    | Yes (low priority)        |
| Fairness              | Yes (arrival) | No    | Yes (equal) | Depends on priorities     |
| Response Time         | Poor   | Good  | Excellent | Excellent (high priority) |
| Throughput            | Poor   | Excellent | Moderate | Depends on mix          |
| Complexity            | Very Low | Medium | Low  | Medium                    |
| Real-Time             | No     | No    | No    | Yes                       |

---

## Implementation Details

### Data Structures

```c
// Priority Scheduling Implementation

#define MAX_PRIORITY 40  // Priority levels 0-39

typedef struct Process {
    int pid;
    int burst_time;
    int remaining_time;
    int arrival_time;
    int priority;           // Lower = higher priority
    int age;                // For aging
    int state;
    struct Process *next;
} Process;

// Priority queue (one per priority level)
typedef struct {
    Process *head;
    Process *tail;
    int size;
} PriorityQueue;

typedef struct {
    PriorityQueue queues[MAX_PRIORITY];
    int preemptive;         // 1 = preemptive, 0 = non-preemptive
    int aging_enabled;
    int aging_interval;     // Time units between aging steps
    Process *current_process;
    int current_time;
} PriorityScheduler;

// Initialize scheduler
void priority_init(PriorityScheduler *sched, int preemptive) {
    for (int i = 0; i < MAX_PRIORITY; i++) {
        sched->queues[i].head = NULL;
        sched->queues[i].tail = NULL;
        sched->queues[i].size = 0;
    }
    sched->preemptive = preemptive;
    sched->aging_enabled = 0;
    sched->current_process = NULL;
    sched->current_time = 0;
}

// Add process to its priority queue
void priority_enqueue(PriorityScheduler *sched, Process *proc) {
    int pri = proc->priority;
    proc->next = NULL;
    
    if (sched->queues[pri].tail == NULL) {
        sched->queues[pri].head = proc;
        sched->queues[pri].tail = proc;
    } else {
        sched->queues[pri].tail->next = proc;
        sched->queues[pri].tail = proc;
    }
    sched->queues[pri].size++;
    proc->state = READY;
}

// Find highest priority process
Process* priority_find_highest(PriorityScheduler *sched) {
    for (int i = 0; i < MAX_PRIORITY; i++) {
        if (sched->queues[i].size > 0) {
            // Remove from head of this queue
            Process *proc = sched->queues[i].head;
            sched->queues[i].head = proc->next;
            
            if (sched->queues[i].head == NULL) {
                sched->queues[i].tail = NULL;
            }
            sched->queues[i].size--;
            proc->next = NULL;
            return proc;
        }
    }
    return NULL;  // No process ready
}

// Schedule next process
void priority_schedule(PriorityScheduler *sched) {
    if (sched->current_process != NULL) {
        return;  // Already running
    }
    
    Process *next = priority_find_highest(sched);
    if (next != NULL) {
        sched->current_process = next;
        next->state = RUNNING;
        // Context switch to next
    }
}

// Handle new process arrival
void priority_on_arrival(PriorityScheduler *sched, Process *proc) {
    priority_enqueue(sched, proc);
    
    if (sched->preemptive && sched->current_process != NULL) {
        // Check if new process has higher priority
        if (proc->priority < sched->current_process->priority) {
            // Preempt current process
            Process *preempted = sched->current_process;
            preempted->state = READY;
            priority_enqueue(sched, preempted);
            sched->current_process = NULL;
            priority_schedule(sched);
        }
    }
}

// Aging implementation
void priority_age_processes(PriorityScheduler *sched) {
    if (!sched->aging_enabled) return;
    
    for (int i = 1; i < MAX_PRIORITY; i++) {
        Process *proc = sched->queues[i].head;
        while (proc != NULL) {
            Process *next = proc->next;
            proc->age++;
            
            // If aged enough, promote to higher priority
            if (proc->age >= sched->aging_interval) {
                // Remove from current queue
                // Add to higher priority queue
                // (Implementation details omitted for brevity)
                proc->age = 0;
                proc->priority--;
            }
            proc = next;
        }
    }
}

// Timer tick
void priority_tick(PriorityScheduler *sched) {
    sched->current_time++;
    
    if (sched->current_process != NULL) {
        sched->current_process->remaining_time--;
        
        if (sched->current_process->remaining_time <= 0) {
            // Process completed
            sched->current_process->state = TERMINATED;
            sched->current_process = NULL;
            priority_schedule(sched);
        }
    }
    
    // Aging check
    if (sched->aging_enabled && 
        sched->current_time % sched->aging_interval == 0) {
        priority_age_processes(sched);
    }
}
```

### Efficient Implementation with Heap

```
For many priority levels, a heap can be more efficient:

Binary Heap Implementation:
├── All ready processes in a min-heap
├── Key: priority (lower = higher priority)
├── O(log n) insertion
├── O(log n) extraction of minimum
├── O(1) peek at highest priority
└── Better than array of queues for sparse priorities

When to use which:
├── Array of queues: Few priority levels (e.g., 40)
├── Binary heap: Many priority levels, sparse usage
├── Both valid, depends on workload
└── Modern systems often use more complex structures
```

---

## Advanced Topics

### 1. Priority Scheduling with Multiple Queues

Combines priority with queue-based scheduling:

```
Multilevel Priority Queue:

Priority 0 (Highest):
└── Round Robin with quantum = 10ms

Priority 1:
└── Round Robin with quantum = 20ms

Priority 2:
└── FCFS

Priority 3 (Lowest):
└── FCFS (runs when nothing else)

Scheduling:
├── Always serve highest non-empty priority
├── Within each level, use specified algorithm
├── Combines priority with fairness
└── Common in modern OSes
```

### 2. Fair Share Scheduling

Extends priority to user/group level:

```
Fair Share Scheduling:

Concept:
├── Each user/group gets a share of CPU
├── Priority computed from share usage
├── Prevents one user from monopolizing
└── Fair at user level, not process level

Implementation:
├── Track CPU usage per user/group
├── Compute priority based on usage vs. share
├── Users below share get higher priority
├── Users above share get lower priority
└── Converges to fair allocation

Example:
Users A, B, C each entitled to 1/3 of CPU
A uses 50% → priority decreased
B uses 10% → priority increased
C uses 40% → priority slightly decreased
Over time, converges to 33% each.
```

### 3. Lottery Scheduling

A probabilistic approach to priority:

```
Lottery Scheduling:

Concept:
├── Each process gets lottery tickets
├── More tickets = higher priority
├── Scheduler draws random ticket
├── Process holding winning ticket runs
└── Probabilistic fairness

Example:
P1: 50 tickets (priority 1)
P2: 30 tickets (priority 2)
P3: 20 tickets (priority 3)

Total tickets: 100
P1 has 50% chance of selection
P2 has 30% chance
P3 has 20% chance

Advantages:
├── Simple to implement
├── Proportional share guaranteed in expectation
├── Flexible ticket allocation
└── Easy to add/remove processes

Disadvantages:
├── Probabilistic (not deterministic)
├── Short-term unfairness possible
└── Not suitable for real-time
```

---

## Common Misconceptions

**Misconception 1:** "Higher priority always means better performance"

**Reality:** Higher priority for one process means lower priority for others. Over-prioritizing can cause starvation and reduce overall system throughput. Priority should reflect actual importance, not arbitrary preference.

**Misconception 2:** "Priority Scheduling eliminates all fairness issues"

**Reality:** Priority Scheduling introduces its own fairness issues—starvation of low-priority processes. It trades equal treatment for importance-based treatment, which can be equally problematic.

**Misconception 3:** "Priority inversion only happens in poorly designed systems"

**Reality:** Priority inversion can occur in any system using priority scheduling with shared resources. Even well-designed systems (like Mars Pathfinder) can experience it. Solutions like priority inheritance are necessary.

**Misconception 4:** "Higher number always means higher priority"

**Reality:** Priority conventions vary. Unix uses lower number = higher priority. Windows uses higher number = higher priority. Always verify the convention in a given system.

**Misconception 5:** "Modern systems don't use priority scheduling"

**Reality:** Priority Scheduling is fundamental to modern OSes. Windows has 32 priority levels, Linux uses nice values, and real-time systems rely heavily on priorities. It's often combined with other algorithms (like RR within priorities).

---

## Hands-On Exercises

### Exercise 1: Basic Priority Scheduling

**Problem:** Five processes arrive at time 0:

| Process | Burst | Priority |
|---------|-------|----------|
| P1      | 10    | 3        |
| P2      | 1     | 1        |
| P3      | 2     | 4        |
| P4      | 1     | 5        |
| P5      | 5     | 2        |

Calculate average waiting and turnaround for non-preemptive priority scheduling.

**Solution:**

```
Order (by priority, lower = higher):
P2 (1), P5 (2), P1 (3), P3 (4), P4 (5)

Gantt Chart:
0  1     6        16   18   19
|--|-----|--------|----|----|
  P2   P5     P1     P3   P4

Completion Times:
P2: 1, P5: 6, P1: 16, P3: 18, P4: 19

Turnaround:
P2: 1, P5: 6, P1: 16, P3: 18, P4: 19
Average: (1+6+16+18+19)/5 = 12 ms

Waiting:
P2: 0, P5: 1, P1: 6, P3: 16, P4: 18
Average: (0+1+6+16+18)/5 = 8.2 ms
```

### Exercise 2: Preemptive Priority

**Problem:** Same as Exercise 1 but with preemption. Processes arrive at times:
P1: 0, P2: 1, P3: 2, P4: 3, P5: 4.

**Solution:**

```
0-1: P1 runs (only process)
1: P2 arrives (priority 1, higher than P1's 3)
   P2 preempts P1. P1 has 9ms remaining.
1-2: P2 runs (completes at 2ms)
2: P3 arrives (priority 4, lower than P1's 3)
   P1 resumes (higher priority)
2-3: P1 runs (rem: 8ms)
3: P4 arrives (priority 5, lower than P1)
   P1 continues
3-4: P1 runs (rem: 7ms)
4: P5 arrives (priority 2, higher than P1's 3)
   P5 preempts P1. P1 has 7ms remaining.
4-9: P5 runs (completes at 9ms)
9: P1 resumes (highest priority among remaining)
9-16: P1 runs (completes at 16ms)
16: P3 (priority 4) runs (completes at 18ms)
18: P4 (priority 5) runs (completes at 19ms)

Gantt Chart:
0  1  2     4        9              16   18 19
|--|--|-----|--------|--------------|----|--|
  P1 P2   P1      P5        P1         P3  P4

Wait, P2 runs 1-2:
0  1  2     4        9              16   18 19
|--|--|-----|--------|--------------|----|--|
  P1 P2   P1      P5        P1         P3  P4

Actually P1 runs 2-4 (2ms), then P5 runs 4-9, then P1 runs 9-16.

Completion: P1=16, P2=2, P3=18, P4=19, P5=9
Turnaround: P1=16, P2=1, P3=16, P4=16, P5=5
Average: (16+1+16+16+5)/5 = 10.8 ms

Waiting: 
P1: 16 - 10 = 6 ms
P2: 2 - 1 - 1 = 0 ms

P3: 18 - 2 - 2 = 14 ms
P4: 19 - 3 - 1 = 15 ms
P5: 9 - 4 - 5 = 0 ms

Average: (6+0+14+15+0)/5 = 7 ms
```

### Exercise 3: Priority with Aging

**Problem:** Design an aging scheme to prevent starvation.

**Solution Approach:**

```
Aging Scheme:
├── Base priorities: P1=1, P2=2, P3=3
├── Aging increment: 0.5 per time unit
├── When waiting, effective priority increases
├── At threshold, promote to next level

Example:
Time 0: P1(1), P2(2), P3(3) → Run P1
Time 5: P1(1), P2(2-2.5=-0.5), P3(3-2.5=0.5) → Run P2
Time 10: P1(1), P3(0.5-2.5=-2) → Run P3
...

Aging ensures each process runs within bounded time.
```

---

## Summary

Priority Scheduling is a fundamental CPU scheduling algorithm that allocates the CPU based on process importance rather than arrival order (FCFS), burst time (SJF), or equality (Round Robin). It provides the mechanism for expressing differentiated service—critical for real-time systems, servers, and any environment where some processes are genuinely more important than others.

### Key Points

1. Priority Scheduling selects the highest-priority process for execution.
2. Can be preemptive or non-preemptive—preemptive is standard for interactive systems.
3. Starvation is the primary problem—low-priority processes may never run.
4. Aging solves starvation by gradually increasing the priority of waiting processes.
5. Priority inversion occurs when low-priority processes block high-priority ones.
6. Priority inheritance and ceiling protocols solve priority inversion.
7. All modern OSes use priority scheduling in various forms.
8. Real-time systems depend on priority scheduling for deadline guarantees.

---

## Key Takeaways

1. Priority reflects importance—not all processes are equal.
2. Starvation is a real risk—aging mechanisms are essential for general-purpose systems.
3. Priority inversion is subtle but critical—especially in real-time systems.
4. Preemption enables responsiveness—critical for interactive and real-time workloads.
5. Priority assignment is a policy decision—static vs. dynamic, user vs. system.
6. Priority Scheduling is a building block—used within larger scheduling frameworks.
7. Windows, Linux, and macOS all use priority—each with different conventions.
8. Understanding priorities is essential for system administration and optimization.

---

## Further Reading

- SJF.md: Scheduling by burst time—related to priority scheduling
- Round-Robin.md: Equal treatment among equal priorities
- Multilevel-Queues.md: Combining priority with other algorithms
- Real-World-Scheduling.md: Linux CFS, Windows scheduler, and more
- Scheduling-Basics.md: Fundamental concepts and terminology

