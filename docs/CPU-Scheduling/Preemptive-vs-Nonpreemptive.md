# Preemptive vs. Non-Preemptive Scheduling

## Introduction

One of the most fundamental distinctions in CPU scheduling is whether the operating system can forcibly remove a running process from the CPU (preemptive) or must wait for the process to voluntarily release the CPU (non-preemptive). This distinction shapes everything about how a scheduler behaves: its responsiveness, its overhead, its complexity, its fairness, and its suitability for different types of systems.

The choice between preemptive and non-preemptive scheduling is not merely technical—it reflects a philosophical decision about the relationship between the operating system and running processes. Non-preemptive scheduling treats processes as trusted collaborators who will eventually yield the CPU. Preemptive scheduling treats the OS as an authority that can interrupt any process at any time.

Understanding this distinction is essential for grasping why modern operating systems work the way they do, and why specific algorithms are chosen for specific environments.

---

## The Fundamental Question

### What Does "Preemption" Mean?

The word preempt comes from the Latin *praeemere*, meaning "to buy before" or "to seize beforehand." In scheduling, preemption means the operating system can seize the CPU from a running process before that process has finished using it.

```
The Core Question:

When a higher-priority or newly-arrived process needs the CPU,
should the OS:

Option A: Wait for the current process to finish or block
         (Non-Preemptive)

Option B: Interrupt the current process, save its state,
         and give the CPU to the new process
         (Preemptive)
```

### A Simple Analogy

Consider a doctor (CPU) treating patients (processes) in an emergency room:

**Non-Preemptive Doctor:**

```
Doctor starts treating Patient A (broken arm).
Patient B arrives with a heart attack.
Doctor continues treating Patient A until finished.
Then treats Patient B.
(Patient A's treatment is not urgent, but the doctor 
 doesn't stop once started.)
```

**Preemptive Doctor:**

```
Doctor starts treating Patient A (broken arm).
Patient B arrives with a heart attack.
Doctor immediately stops treating Patient A,
treats Patient B (life-threatening),
then returns to Patient A.
(More responsive to emergencies, but requires 
 stopping and resuming treatment.)
```

The preemptive approach is clearly better for emergencies—but it has costs: the doctor must carefully document where treatment stopped (save state), and switching between patients has overhead.

---

## Non-Preemptive Scheduling

### Definition

In non-preemptive scheduling, once a process is allocated the CPU, it runs until it either:

1. Terminates (completes execution)
2. Blocks (waits for I/O or an event)
3. Voluntarily yields (calls yield() or sleep())

The scheduler cannot interrupt a running process, no matter how urgent another process may be.

```
Non-Preemptive Scheduling Flow:

Process states:
                    ┌─────────────────────────────┐
                    │                             │
   New ──▶ Ready ──▶ Running ──▶ Terminated     │
             ▲         │                         │
             │         │                         │
             │    Process blocks                 │
             │    for I/O                        │
             │         ▼                         │
             └──── Waiting ◀─────────────────────┘
                       │
                       │ I/O completes
                       └──────▶ Ready

Key: Running → Ready transition NEVER happens
     (Process only leaves Running by completing or blocking)
```

### Scheduling Points

```
Non-Preemptive Scheduling Decisions Occur Only When:

1. Process Terminates
   ├── Process has finished execution
   ├── Scheduler must pick next process
   └── Most common scheduling point

2. Process Blocks for I/O
   ├── Process cannot continue without data
   ├── CPU becomes idle
   ├── Scheduler picks another process
   └── Prevents CPU waste

3. Process Voluntarily Yields
   ├── Process calls yield(), sleep(), or similar
   ├── Uncommon in practice
   ├── Programmer explicitly releases CPU
   └── Cooperative behavior

NOT a scheduling point:
├── New process arrival (process waits in queue)
├── Timer interrupt (no preemption)
├── Higher priority process arrival (waits)
└── I/O completion (unless it causes current to block)
```

### Examples of Non-Preemptive Algorithms

```
Non-Preemptive Scheduling Algorithms:

1. First-Come, First-Served (FCFS)
   ├── Process runs until completion or block
   ├── Simplest non-preemptive algorithm
   ├── No interruption mechanism
   └── Fair in arrival order

2. Non-Preemptive Shortest Job First (SJF)
   ├── Shortest job selected at scheduling points
   ├── Runs to completion once started
   ├── Optimal for average waiting time
   └── Requires burst time knowledge

3. Non-Preemptive Priority Scheduling
   ├── Highest priority process selected
   ├── Runs to completion once started
   ├── Higher priority arrivals wait
   └── Potential for priority inversion

4. Highest Response Ratio Next (HRRN)
   ├── Selects process with highest response ratio
   ├── Non-preemptive
   └── Prevents starvation through aging
```

### Visual Example

```
Non-Preemptive Scheduling Example:

Processes:
├── P1: burst 5ms, arrives at 0
├── P2: burst 3ms, arrives at 1
├── P3: burst 8ms, arrives at 2

Timeline (Non-Preemptive FCFS):

0        5        8                16
|────────|────────|────────────────|
    P1       P2          P3

Key observations:
├── P1 starts at 0, runs to 5 (no interruption)
├── P2 arrives at 1, waits (cannot preempt P1)
├── P3 arrives at 2, waits (cannot preempt P1)
├── P1 completes at 5, P2 starts
├── P2 completes at 8, P3 starts
└── P3 completes at 16

Even though P2 and P3 arrived, they had to wait for P1.
```

---

## Preemptive Scheduling

### Definition

In preemptive scheduling, the operating system can forcibly remove a process from the CPU at any time, typically through a hardware timer interrupt. The process's state is saved, and it can be resumed later.

```
Preemptive Scheduling Flow:

Process states:
                    ┌─────────────────────────────┐
                    │                             │
   New ──▶ Ready ◀──▶ Running ──▶ Terminated   │
             ▲    │    │                         │
             │    │    │ blocks for I/O          │
             │    │    ▼                         │
             │    │  Waiting ◀───────────────────┘
             │    │    │
             │    │    │ I/O completes
             │    │    └──────▶ Ready
             │    │
             │    └── Preempted (timer or higher priority)
             │
             └─────── Back to Ready queue

Key: Running → Ready transition IS possible
     (Process can be interrupted and rescheduled)
```

### Scheduling Points

```
Preemptive Scheduling Decisions Occur When:

1. Process Terminates
   └── Same as non-preemptive

2. Process Blocks for I/O
   └── Same as non-preemptive

3. Process Voluntarily Yields
   └── Same as non-preemptive

4. Timer Interrupt (Time Quantum Expires)
   ├── Hardware timer fires periodically
   ├── OS regains control
   ├── Current process moved back to ready queue
   └── Scheduler picks next process

5. Higher Priority Process Arrives
   ├── New process has higher priority than current
   ├── Current process preempted immediately
   └── New process runs

6. I/O Completion Wakes Higher Priority Process
   ├── I/O completes for waiting process
   ├── Woken process has higher priority than current
   └── Current process preempted
```

### Examples of Preemptive Algorithms

```
Preemptive Scheduling Algorithms:

1. Round Robin (RR)
   ├── Preemption by timer (fixed time quantum)
   ├── Most common preemptive algorithm
   ├── Fair allocation of CPU time
   └── Excellent response time

2. Preemptive Shortest Remaining Time First (SRTF)
   ├── New shorter job preempts current
   ├── Optimal average waiting time
   ├── Higher overhead than non-preemptive
   └── Requires burst time estimation

3. Preemptive Priority Scheduling
   ├── Higher priority process preempts current
   ├── Standard in real-time systems
   ├── Requires priority inversion protection
   └── Potential for starvation (without aging)

4. Multilevel Feedback Queue (MLFQ)
   ├── Preemptive with multiple queues
   ├── Processes move between queues
   ├── Adaptive to process behavior
   └── Used in modern OSes

5. Earliest Deadline First (EDF)
   ├── Process with nearest deadline runs
   ├── Preemptive by nature
   ├── Optimal for real-time scheduling
   └── Requires deadline knowledge
```

### Visual Example

```
Preemptive Scheduling Example (Round Robin, q=2ms):

Same processes as before:
├── P1: burst 5ms, arrives at 0
├── P2: burst 3ms, arrives at 1
├── P3: burst 8ms, arrives at 2

Timeline:

0  2  4  6  8  10 12 14 16 18
|--|--|--|--|--|--|--|--|--|
  P1 P2 P1 P3 P2 P1 P3 P3 P3
  ^  ^  
  |  |
  |  P2 arrives at 1, gets CPU at 2
  P1 runs 0-2, preempted by timer

Detailed:
0-2: P1 runs (remaining: 3ms)
2-4: P2 runs (arrived at 1, starts at 2)
4-6: P1 runs (remaining: 1ms)
6-8: P3 runs (arrived at 2, starts at 6)
8-10: P2 runs (remaining: 1ms)
10-12: P1 runs (completes at 12ms)
12-14: P3 runs (remaining: 6ms)
14-16: P3 runs (remaining: 4ms)
16-18: P3 runs (remaining: 2ms)... 

Wait, P3 needs 8ms, so it needs 4 quanta:
P3 runs: 6-8 (2ms), 12-14 (2ms), 14-16 (2ms), 16-18 (2ms) = 8ms total
P3 completes at 18ms.

Gantt Chart:
0  2  4  6  8  10 12 14 16 18
|--|--|--|--|--|--|--|--|--|
  P1 P2 P1 P3 P2 P1 P3 P3 P3

Key observation: Every process gets CPU time quickly!
├── P1's response time: 0ms
├── P2's response time: 1ms (arrived at 1, started at 2)
├── P3's response time: 4ms (arrived at 2, started at 6)
└── Much better than non-preemptive for response time
```

---

## Detailed Comparison

### Side-by-Side Comparison Table

| Aspect                      | Non-Preemptive       | Preemptive           |
|-----------------------------|-----------------------|-----------------------|
| Interruption                 | Never interrupts running process | Can interrupt at any time |
| Scheduling Points            | Completion, blocking, yielding | Timer, higher priority arrival |
| Response Time                | Poor (long waits possible) | Good (bounded response) |
| Overhead                     | Low (fewer context switches) | Higher (frequent switches) |
| Complexity                   | Simple to implement     | Complex (synchronization needed) |
| Fairness                     | Can be poor             | Better (time slicing) |
| Real-Time                    | Suitable                | Yes (with proper design) |
| CPU Utilization              | Can be poor with I/O    | Better with I/O        |
| Data Integrity               | Simpler (no mid-operation interrupts) | Requires careful locking |

### The Overhead Trade-off

The most significant practical difference is overhead:

```
Context Switch Overhead Comparison:

Non-Preemptive Example:
├── 5 processes, each burst 20ms
├── Process runs until completion
├── Context switches: 4 (between processes)
├── Overhead: 4 × 10μs = 40μs
├── Total time: 100ms + 0.04ms
└── Overhead: 0.04%

Preemptive Example (Round Robin, q=10ms):
├── 5 processes, each burst 20ms
├── Each needs 4 quanta
├── Context switches: 5 × 4 - 1 = 19
├── Overhead: 19 × 10μs = 190μs
├── Total time: 100ms + 0.19ms
└── Overhead: 0.19%

Preemptive overhead is ~5x higher
But response time is dramatically better
```

### Synchronization Complexity

Non-preemptive scheduling simplifies kernel design:

```
Non-Preemptive Kernel Advantages:

1. No Race Conditions Within Kernel
   ├── Kernel code runs to completion
   ├── No concurrent access to kernel data
   ├── Simpler locking (or no locking)
   └── Easier to reason about correctness

2. No Preemption During Critical Sections
   ├── Process holding lock cannot be interrupted
   ├── No need for preemption disabling
   ├── Simpler lock implementation
   └── Fewer deadlock scenarios

3. Predictable Kernel Execution
   ├── Kernel operation completion time predictable
   ├── No unexpected context switches
   ├── Easier debugging
   └── Deterministic behavior

Preemptive Kernel Challenges:

1. Kernel Data Structure Protection
   ├── Any kernel data can be accessed concurrently
   ├── Requires fine-grained locking
   ├── Complex synchronization protocols
   └── Potential for deadlock and races

2. Preemption During System Calls
   ├── System call can be interrupted mid-execution
   ├── Must be able to resume cleanly
   ├── Requires careful state management
   └── Increases kernel complexity

3. Interrupt Handling
   ├── Interrupts can arrive at any time
   ├── Must be handled safely
   ├── Cannot leave shared data inconsistent
   └── Requires careful design
```

---

## Practical Implications

### When Non-Preemptive is Preferred:

1. **System Requirements:**
   ├── Throughput is more important than response
   ├── No interactive users
   ├── Batch processing environment
   └── Simple embedded system

2. **Workload Characteristics:**
   ├── All processes are similar length
   ├── CPU-bound processes dominate
   ├── Predictable workloads
   └── No urgent tasks

3. **Implementation Constraints:**
   ├── Limited development resources
   ├── Simple kernel desired
   ├── Trust between processes
   └── Cooperative multitasking acceptable

### When Preemptive is Preferred:

1. **System Requirements:**
   ├── Response time is critical
   ├── Interactive users
   ├── Real-time deadlines
   └── Server with mixed workloads

2. **Workload Characteristics:**
   ├── Mixed process types
   ├── I/O-bound and CPU-bound mix
   ├── Unpredictable arrivals
   └── Urgent tasks exist

3. **Implementation Constraints:**
   ├── Development resources available
   ├── Complex kernel acceptable
   ├── Safety/security critical
   └── Modern OS expectations

---

## Configuration Recommendations

```
Recommended Configurations by System Type:

Desktop/Laptop:
├── Preemptive kernel
├── Round Robin or CFS
├── Small time quantum (10-50ms)
├── Dynamic priority adjustments
└── Focus: Responsiveness

Server:
├── Preemptive kernel (or voluntary)
├── Priority-based scheduling
├── Larger time quantum (100ms)
├── Focus: Throughput
└── Consider: Reduced preemption

Real-Time Embedded:
├── Fully preemptive kernel (PREEMPT_RT or RTOS)
├── Priority-based or EDF
├── Small time quantum (1-10ms)
├── Priority inheritance
└── Focus: Determinism

Simple Embedded:
├── Non-preemptive or cooperative
├── FCFS or simple priority
├── Event-driven
├── Focus: Simplicity
└── Low power consumption

Batch/Compute Cluster:
├── Non-preemptive kernel
├── SJF or FCFS
├── Large quantum or no preemption
├── Focus: Throughput
└── Minimal overhead
```

---

## Summary

The choice between preemptive and non-preemptive scheduling is fundamental to operating system design. Non-preemptive scheduling treats processes as trusted entities that will eventually yield the CPU, offering simplicity, low overhead, and better throughput. Preemptive scheduling treats the OS as an authority that can interrupt any process, offering responsiveness, fairness, and real-time capability at the cost of increased complexity and overhead.

### Key Points

1. Preemptive scheduling can interrupt a running process; non-preemptive cannot.
2. Scheduling points differ: Non-preemptive schedules on completion/blocking; preemptive also on timer and priority events.
3. Preemption requires hardware support (timer interrupts) and kernel complexity.
4. Non-preemptive is simpler but offers poorer response time.
5. Preemptive is essential for interactive and real-time systems.
6. Most modern OSes are preemptive but with controlled non-preemptible regions.
7. Real-time systems require strict preemption with bounded latency.
8. The choice involves trade-offs between responsiveness, overhead, throughput, and complexity.

---

## Key Takeaways

1. Preemption is a policy decision—it determines whether the OS can interrupt processes.
2. Non-preemptive is simpler—fewer context switches, no mid-operation races, easier to reason about.
3. Preemptive is more responsive—essential for interactive systems and real-time deadlines.
4. Timer interrupts enable preemption—the hardware foundation of preemptive scheduling.
5. Modern kernels use hybrid approaches—preemptive with controlled non-preemptible sections.
6. Real-time systems require strict preemption—with bounded latency and priority inheritance.
7. The overhead of preemption is real—context switches, cache effects, kernel complexity.
8. Choose based on requirements—no universal "best" choice exists.

---

## Further Reading

- Scheduling-Basics.md: Core scheduling concepts including preemption basics
- FCFS.md: Classic non-preemptive scheduling algorithm
- Round-Robin.md: Preemptive scheduling with time quanta
- Priority-Scheduling.md: Preemption based on priority
- Real-World-Scheduling.md: Linux CFS, Windows, and PREEMPT_RT implementations

