# Scheduling Basics

## Introduction

CPU scheduling is the foundation of multitasking operating systems. At its core, scheduling answers a simple question: "Which process should run next on the CPU?" While the question seems straightforward, the answer involves balancing competing goals like responsiveness, throughput, fairness, and efficiency.

In this document, we'll explore the fundamental concepts of CPU scheduling—the terminology, the goals, the types of schedulers, and the key metrics used to evaluate scheduling algorithms. This knowledge serves as the foundation for understanding specific scheduling algorithms like First-Come-First-Served (FCFS), Shortest Job First (SJF), Round Robin (RR), and others covered in subsequent documents.

---

## Why Do We Need Scheduling?

### The Problem

On a system with a single CPU (or one core), only one process can execute at any given moment. Yet modern systems typically have dozens or even hundreds of processes loaded in memory simultaneously. Without scheduling, the CPU would execute one process to completion before starting another—a scheme that would cause:

- Poor responsiveness: A long-running batch job would block interactive users from getting any CPU time.
- Wasted CPU cycles: When a process waits for I/O (e.g., reading from disk), the CPU would sit idle instead of running another process.
- Unfair resource distribution: A compute-intensive process could monopolize the CPU indefinitely.

### The Solution

CPU scheduling solves this by interleaving process execution. The scheduler rapidly switches between processes, creating the illusion that multiple programs run simultaneously—even on a single-core system. This is the essence of multiprogramming and time-sharing.

```
Time:    0ms     100ms    200ms    300ms    400ms    500ms
         |--------|--------|--------|--------|--------|
CPU:     |   P1   |   P2   |   P1   |   P3   |   P2   |
         |________|________|________|________|________|

Without scheduling:  P1 (0-400ms) → P2 (400-500ms)
With scheduling:     P1, P2, P3 interleaved in small slices
```

---

## The CPU-I/O Burst Cycle

Understanding scheduling requires understanding how processes actually use the CPU. Process execution typically follows a pattern known as the CPU-I/O burst cycle:

1. **CPU Burst**: The process executes instructions on the CPU (computing, processing data).
2. **I/O Burst**: The process waits for I/O operations to complete (reading a file, receiving network data).
3. **Repeat**.

```
Process Lifecycle:
                
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │  CPU    │────▶│  I/O    │────▶│  CPU    │──▶ ...
  │  Burst  │     │  Wait   │     │  Burst  │
  └─────────┘     └─────────┘     └─────────┘
       ↑                               │
       └───────────────────────────────┘
```

### Types of Processes

Based on their burst patterns, processes can be classified as:

| Type       | CPU Bursts         | I/O Bursts         | Example                            |
|------------|---------------------|---------------------|------------------------------------|
| CPU-bound  | Long, frequent      | Rare, short         | Video encoding, scientific computation |
| I/O-bound  | Short, rare         | Frequent, long      | Web servers, text editors, database queries |
| Balanced   | Moderate            | Moderate            | Most general-purpose applications  |

This distinction matters because scheduling algorithms often need to handle these process types differently. I/O-bound processes should be scheduled promptly when they become ready (since their CPU bursts are short), while CPU-bound processes can tolerate longer waits.

---

## Types of Schedulers

Operating systems typically employ multiple levels of scheduling, each operating at different time scales:

1. **Long-Term Scheduler (Job Scheduler)**
   - **What it does:** Selects which processes are admitted into the ready queue from the pool of submitted jobs.
   - **Frequency:** Runs infrequently (seconds to minutes).
   - **Controls:** The degree of multiprogramming—the number of processes in memory.
   - **Purpose:** Maintains a good mix of CPU-bound and I/O-bound processes.
   - **Notable:** Not present in many modern general-purpose OSes (like Linux/Windows), where processes are admitted immediately. Present in batch processing systems.

2. **Medium-Term Scheduler (Swapper)**
   - **What it does:** Temporarily removes processes from memory (swapping them to disk) and later restores them.
   - **Frequency:** Runs occasionally (when memory pressure is high).
   - **Purpose:** Reduces the degree of multiprogramming when memory is scarce; improves process mix.
   - **Operation:** Moves processes between the "ready" and "suspended ready" states.

3. **Short-Term Scheduler (CPU Scheduler)**
   - **What it does:** Selects which process from the ready queue gets the CPU next.
   - **Frequency:** Runs very frequently (every few milliseconds).
   - **Speed requirement:** Must be extremely fast—if it takes 10ms to schedule and processes run for 100ms, 10% of CPU time is wasted on scheduling overhead.
   - **This is the primary focus of CPU scheduling discussions.**

```
                    Long-Term         Short-Term
                    Scheduler         Scheduler
                        │                  │
   New ──────────▶ Ready Queue ──────▶ CPU ──────▶ Terminated
   Jobs    admit       │     │  dispatch    │
                       │     │              │
                       │     │  preempt     │
                       │     ▼              │
                       │  Waiting Queue ◀───┘
                       │  (I/O wait)
                       │
              Medium-Term Scheduler
              (swap out / swap in)
```

---

## When Does Scheduling Occur?

The CPU scheduler makes decisions at specific moments called scheduling points:

### Non-Preemptive Scheduling Points

The scheduler runs only when the current process voluntarily gives up the CPU:

1. **Process terminates** — the process is done.
2. **Process blocks** — the process waits for I/O, a semaphore, etc.
3. **Process yields** — the process explicitly calls yield() or sleep().

### Preemptive Scheduling Points

The scheduler can also interrupt a running process:

4. **Interrupt arrives** — an I/O device completes an operation, a timer fires, etc. If a higher-priority process becomes ready, the scheduler may preempt.
5. **Timer interrupt** — a time slice expires, forcing the current process to yield.

### Scheduling Decision Table

| Event                                | Preemptive? | Description                                    |
|--------------------------------------|-------------|------------------------------------------------|
| Process terminates                   | No          | Must schedule a new process                    |
| Process blocks for I/O               | No          | Process cannot continue until I/O completes    |
| Process yields                       | No          | Voluntary relinquishment                        |
| I/O completion interrupt              | Yes         | A waiting process becomes ready                 |
| Timer interrupt (quantum expires)    | Yes         | Time slice exhausted                            |
| New process arrives                   | Yes         | New process may have higher priority           |

---

## Preemptive vs. Non-Preemptive Scheduling

This distinction is fundamental to understanding scheduling behavior:

### Non-Preemptive (Cooperative) Scheduling

Once a process gets the CPU, it keeps it until it voluntarily releases it (terminates, blocks, or yields).

```
Advantages:
├── Simpler to implement
├── Lower overhead (fewer context switches)
├── No race conditions on kernel data structures
└── Predictable behavior

Disadvantages:
├── A process can monopolize the CPU indefinitely
├── Poor responsiveness for interactive systems
├── One buggy process can hang the entire system
└── Not suitable for real-time applications
```

### Preemptive Scheduling

The OS can forcibly remove the CPU from a running process (typically via a timer interrupt).

```
Advantages:
├── Better responsiveness
├── Prevents CPU monopolization
├── Essential for time-sharing and real-time systems
├── Enables priority-based service

Disadvantages:
├── More complex implementation
├── Overhead from context switches and timer interrupts
├── Race conditions require careful synchronization
├── Can cause priority inversion problems
```

Most modern general-purpose operating systems (Linux, Windows, macOS) use preemptive scheduling.

---

## Scheduling Queues and States

Scheduling revolves around managing queues of processes in various states:

### Process States (Review)

```
                    ┌──────────────────────────────┐
                    │                              │
   New ──▶ Ready ──▶ Running ──▶ Terminated       │
             ▲         │                           │
             │         │ blocks for I/O            │
             │         ▼                           │
             └──── Waiting ◀───────────────────────┘
                      │
                      │ I/O completes
                      │ (interrupt)
                      └──────────────▶ Ready
```

### Key Queues

| Queue              | Contains                      | Purpose                                 |
|--------------------|-------------------------------|-----------------------------------------|
| Ready Queue        | Processes in READY state      | Awaiting CPU allocation                  |
| Wait Queues        | Processes in WAITING state    | Awaiting I/O completion                  |
| Job Queue          | All processes in the system    | Complete set of processes                |

The ready queue is the scheduler's primary data structure. It can be implemented as:

- FIFO queue (for FCFS scheduling)
- Priority queue (for priority-based scheduling)
- Multiple queues (for multilevel queue scheduling)
- Tree structure (for complex algorithms like CFS in Linux)

---

## Key Scheduling Metrics

To evaluate and compare scheduling algorithms, we need quantitative metrics:

### Core Metrics

| Metric            | Definition                                                 | Goal      |
|-------------------|------------------------------------------------------------|-----------|
| CPU Utilization    | Percentage of time the CPU is busy                         | Maximize  |
| Throughput        | Number of processes completed per unit time               | Maximize  |
| Turnaround Time   | Time from process submission to completion                 | Minimize  |
| Waiting Time      | Total time a process spends in the ready queue             | Minimize  |
| Response Time     | Time from submission to first CPU response (first output/progress) | Minimize  |

### Detailed Calculations

For a process P:

```
Turnaround Time = Completion Time - Arrival Time

Waiting Time = Turnaround Time - Total CPU Burst Time
             = Time in Ready Queue

Response Time = Time of First CPU Allocation - Arrival Time
```

### Example Calculation

Consider three processes arriving at time 0 with CPU bursts of 10ms, 5ms, and 8ms (executed in order P1, P2, P3):

```
Timeline:
0        10   15       23
|─────────|────|────────|
    P1       P2     P3

P1: Turnaround = 10 - 0 = 10ms,  Waiting = 10 - 10 = 0ms
P2: Turnaround = 15 - 0 = 15ms,  Waiting = 15 - 5 = 10ms
P3: Turnaround = 23 - 0 = 23ms,  Waiting = 23 - 8 = 15ms

Average Turnaround = (10 + 15 + 23) / 3 = 16ms
Average Waiting = (0 + 10 + 15) / 3 = 8.33ms
Throughput = 3 processes / 23ms ≈ 0.13 processes/ms
```

---

## Scheduling Algorithm Goals

Different systems have different scheduling priorities:

### General-Purpose (Desktop/Server) Systems

```
Primary Goals:
├── Fairness: All processes get a reasonable share of CPU
├── Responsiveness: Interactive processes get quick service
├── Throughput: High overall completion rate
└── Efficiency: Low scheduling overhead

Secondary Considerations:
├── Priority: Some processes (system, real-time) need preference
└── Predictability: Similar workloads should get similar service
```

### Real-Time Systems

```
Primary Goals:
├── Meeting deadlines: Hard real-time requires guaranteed timing
├── Determinism: Predictable behavior under all conditions
└── Low latency: Minimal delay between event and response

Trade-offs:
└── Throughput and fairness often sacrificed for timing guarantees
```

### Batch Systems

```
Primary Goals:
├── Throughput: Maximize jobs completed per hour
├── CPU Utilization: Keep expensive hardware busy
└── Average Turnaround: Complete jobs quickly

Trade-offs:
└── Response time is less critical (no interactive users)
```

---

## The Scheduling Problem: Formal Definition

Given:

- A set of processes P = {P₁, P₂, ..., Pₙ}
- Each process Pᵢ has:
  - Arrival time aᵢ (when it becomes ready)
  - CPU burst time bᵢ (how long it needs the CPU)
  - Optional: priority pᵢ, deadline dᵢ, I/O pattern

Find an execution schedule (assignment of CPU time to processes) that optimizes one or more metrics subject to constraints.

### Conflicting Objectives

The fundamental challenge of scheduling: objectives often conflict.

#### Trade-off Example
Minimize response time vs. maximize throughput: Frequent context switches improve response time but reduce throughput due to overhead.

Fairness vs. efficiency: Giving I/O-bound processes priority improves overall throughput but may starve CPU-bound processes.

Priority vs. fairness: High-priority processes get service at the expense of low-priority ones.

There is no universally optimal scheduling algorithm—the best algorithm depends on the system's goals and workload characteristics.

---

## Context Switch Overhead

Every scheduling decision that switches processes incurs a context switch cost:

```
Context Switch Operations:
├── Save current process state (registers, PC, stack pointer)
├── Save memory management information
├── Update process control block (PCB)
├── Move process to appropriate queue
├── Select next process (scheduler runs)
├── Load next process state
├── Update memory management unit (MMU) registers
├── Flush/update caches and TLB (implicit cost)
└── Resume execution
```

### Overhead Costs

| Component             | Typical Cost          | Notes                                        |
|-----------------------|-----------------------|----------------------------------------------|
| Direct overhead        | 1-10 microseconds     | Time spent saving/restoring state            |
| Cache effects          | Variable (can be significant) | Cold caches cause more cache misses after switching |
| TLB effects           | Variable              | Address translation entries may need reloading |

### Implication for Scheduler Design

The scheduler must balance:

- Switching too often: High overhead, reduced throughput
- Switching too rarely: Poor responsiveness, potential starvation

Time quantum selection (for preemptive algorithms) directly affects this trade-off.

---

## The Ready Queue: Implementation Considerations

The ready queue is the scheduler's primary data structure. Its implementation affects scheduler performance:

### Implementation Options

| Data Structure                  | Best For                                    | Complexity                      |
|----------------------------------|--------------------------------------------|---------------------------------|
| Simple FIFO queue                 | FCFS, Round Robin                          | O(1) enqueue/dequeue           |
| Priority queue (heap)            | Priority scheduling                        | O(log n) operations            |
| Sorted linked list                | SJF (by burst time)                       | O(n) insertion                 |
| Red-black tree                   | CFS (by virtual runtime)                  | O(log n) operations            |
| Multiple queues                   | Multilevel scheduling                      | O(1) + queue selection         |

### Design Considerations

1. **Speed**: The scheduler runs frequently; queue operations must be fast.
2. **Scalability**: Performance should degrade gracefully with many processes.
3. **Fairness**: Queue implementation can introduce (or prevent) starvation.
4. **Preemption support**: The data structure must support removing the current process and re-inserting it.

---

## Scheduling Algorithm Classification

Scheduling algorithms can be classified along several dimensions:

### By Decision Timing

```
Non-Preemptive:        Preemptive:
├── FCFS              ├── Round Robin
├── SJF (non-preemp)  ├── SRTF (preemptive SJF)
├── Priority (non-preemp) ├── Priority (preemptive)
└── HRRN              ├── Multilevel Feedback Queue
                      └── CFS, MLFQ
```

### By Information Requirements

| Type                      | Information Needed                               | Example                             |
|---------------------------|--------------------------------------------------|-------------------------------------|
| Static/Direct             | Known in advance (burst times, priorities)      | SJF, Priority                       |
| Dynamic/Predictive        | Estimated from history                           | SJF with exponential averaging      |
| No prior knowledge        | Only queue state                                | FCFS, Round Robin                   |

### By System Type

| System Type               | Suitable Algorithms                              |
|---------------------------|--------------------------------------------------|
| Batch                     | FCFS, SJF, Priority (non-preemptive)             |
| Interactive               | Round Robin, Priority (preemptive), MLFQ        |
| Real-time                 | Rate Monotonic, EDF (Earliest Deadline First)   |

---

## Simple Scheduling Examples

Before diving into detailed algorithm documentation, here's a quick preview of how different algorithms handle the same workload:

### Workload

Three processes arrive at time 0 with CPU bursts: P1=24ms, P2=3ms, P3=3ms

#### FCFS (First-Come-First-Served)

```
0                    24  27  30
|─────────────────────|---|---|
         P1             P2   P3

Average waiting time: (0 + 24 + 27) / 3 = 17ms
```

#### SJF (Shortest Job First)

```
0    3   6                    30
|----|----|─────────────────────|
  P2   P3          P1

Average waiting time: (6 + 0 + 3) / 3 = 3ms
```

#### Round Robin (quantum = 4ms)

```
0    4    7   10   14   18   22   26   30
|----|----|----|----|----|----|----|----|
  P1   P2   P3   P1   P1   P1   P1   P1

Average waiting time: 
P1: 0 + (7-4) + (14-10) + (18-14) + (22-18) + (26-22) = 13ms of preemption delay
P2: (4-0) = 4ms
P3: (7-0) = 7ms
Average: (13 + 4 + 7) / 3 = 8ms
```

This simple example shows how dramatically different algorithms can affect performance metrics.

---

## The Convoy Effect

A classic scheduling problem illustrated with FCFS:

### Scenario

- P1: CPU-bound process with 100ms burst
- P2-P5: I/O-bound processes with 10ms bursts each

### The Problem

```
FCFS executes P1 first (100ms). During this time:
├── P2-P5 wait in the ready queue
├── I/O devices sit idle
├── System feels unresponsive
└── This is the "convoy effect" — short processes stuck behind long ones

Timeline:
0                        100 110 120 130 140
|─────────────────────────|----|----|----|----|
           P1               P2   P3   P4   P5
```

### The Solution

Algorithms that prioritize shorter jobs (SJF) or use preemption (Round Robin) avoid the convoy effect by ensuring short processes don't wait behind long ones.

---

## Scheduling and I/O

Real processes perform I/O, which complicates scheduling:

### I/O-Bound Process Behavior

```
I/O-bound process cycle:
CPU (2ms) → I/O wait (20ms) → CPU (2ms) → I/O wait (20ms) → ...

CPU-bound process:
CPU (50ms) → CPU (50ms) → CPU (50ms) → ...
```

### Scheduling Implications

1. I/O-bound processes need prompt service: When their I/O completes, they should be scheduled quickly to issue the next I/O request. This keeps I/O devices busy and improves overall system throughput.
2. Priority for I/O-bound processes: Many schedulers give implicit priority to processes that use less CPU time (they naturally get scheduled more often since they block frequently).
3. CPU-bound processes benefit from longer time slices: They have fewer context switches if allowed to run for extended periods, improving cache efficiency.

### The Ideal Mix

The best system performance occurs with a mix of process types:

```
With good process mix:
├── CPU-bound processes use the CPU while I/O-bound processes wait
├── I/O-bound processes use I/O devices while CPU-bound processes compute
├── Both CPU and I/O devices stay busy
└── Maximum resource utilization
```

---

## Scheduling in Modern Operating Systems

Modern OS schedulers are sophisticated implementations of these basic concepts:

### Linux (Completely Fair Scheduler - CFS)

```
Key Concepts:
├── Tracks "virtual runtime" for each process
├── Schedules the process with the lowest virtual runtime
├── Uses red-black tree for efficient lookup
├── Implicit priority via "niceness" affecting runtime accounting
└── No fixed time slice; dynamic based on system load
```

### Windows (Multilevel Feedback Queue)

```
Key Concepts:
├── 32 priority levels (0-31)
├── Dynamic priority adjustment based on behavior
├── I/O completion boosts priority temporarily
├── Time slice varies with priority level
└── Real-time class for critical processes
```

### macOS (Mach + BSD Hybrid)

```
Key Concepts:
├── Based on Mach scheduler with BSD extensions
├── Priority-based with "handoff" scheduling
├── Gang scheduling support for threads
└── Optimized for interactive desktop experience
```

---

## Summary

Scheduling basics establish the foundation for understanding CPU allocation in operating systems:

| Concept                | Key Points                                                         |
|------------------------|--------------------------------------------------------------------|
| Purpose                | Deciding which process runs next on the CPU                        |
| Burst Cycle            | Processes alternate between CPU and I/O bursts                     |
| Scheduler Types        | Long-term (admission), medium-term (swapping), short-term (CPU)  |
| Preemption             | Preemptive vs. non-preemptive scheduling                          |
| Metrics                | CPU utilization, throughput, turnaround, waiting, response time    |
| Trade-offs             | No algorithm optimizes everything; objectives conflict             |
| Context Switch         | Overhead that limits scheduling frequency                           |
| Convoy Effect          | Short processes stuck behind long ones in FCFS                     |

---

## Key Takeaways

1. CPU scheduling is essential for multitasking and time-sharing systems.
2. Processes exhibit CPU-I/O burst patterns that schedulers should exploit.
3. Preemptive scheduling provides better responsiveness but higher overhead.
4. Multiple metrics exist (turnaround, waiting, response, throughput) and they often conflict.
5. No perfect algorithm exists—the best choice depends on system goals and workload.
6. Context switching is expensive, so schedulers must balance responsiveness against overhead.
7. I/O-bound processes deserve priority to keep I/O devices busy and improve overall throughput.
8. The ready queue implementation significantly affects scheduler performance.

---

## Further Reading

- CPU-and-Execution.md: How the CPU actually executes instructions and why scheduling matters.
- Preemptive-vs-Nonpreemptive.md: Deeper dive into preemption mechanisms.
- FCFS.md: First-Come-First-Served algorithm in detail.
- SJF.md: Shortest Job First and its variants.
- Round-Robin.md: Time-sliced preemptive scheduling.
- Real-World-Scheduling.md: How modern OSes implement these concepts.

