# Multilevel Queue Scheduling

## Introduction

Multilevel Queue (MLQ) scheduling represents a significant evolution in CPU scheduling algorithms. Instead of treating all processes as a single homogeneous group, MLQ recognizes that different types of processes have fundamentally different scheduling requirements. Interactive processes need quick response times, batch processes need throughput, and system processes need the highest priority—all competing for the same CPU.

Multilevel Queue scheduling addresses this by partitioning processes into multiple queues based on their characteristics, with each queue having its own scheduling algorithm. This document explores how MLQ works, its various configurations, its advantages and disadvantages, and how it serves as a foundation for even more sophisticated scheduling algorithms.

---

## The Core Concept

### Basic Idea

Multilevel Queue scheduling divides the ready queue into several separate queues, each containing processes with similar characteristics:

```
Traditional Single Queue:
┌─────────────────────────────────────┐
│  P1, P2, P3, P4, P5, P6, P7, P8   │──▶ CPU
│  (All process types mixed)          │
└─────────────────────────────────────┘

Multilevel Queues:
┌─────────────────────────────┐
│  System Processes           │──▶ CPU
│  (P1, P2)                   │    ↑
├─────────────────────────────┤    │ Highest Priority
│  Interactive Processes      │────┤
│  (P3, P4, P5)               │    │
├─────────────────────────────┤    │
│  Batch Processes            │────┤
│  (P6, P7, P8)               │    │ Lowest Priority
└─────────────────────────────┘    │
                                    │
        Scheduler selects from highest priority non-empty queue
```

### Why Multiple Queues?

Processes have fundamentally different characteristics:

| Process Type   | CPU Burst Pattern               | Scheduling Requirement                |
|----------------|----------------------------------|---------------------------------------|
| System         | Short, urgent                   | Highest priority, immediate service    |
| Interactive     | Short bursts, frequent I/O      | Quick response, preemptive            |
| Batch          | Long bursts, rare I/O           | Throughput, non-preemptive acceptable  |
| Background      | Variable, low urgency           | Can wait, lowest priority              |

A single scheduling algorithm cannot optimally serve all these different types:

```
Problem with Single Algorithm:
├── FCFS: Batch processes delay interactive ones
├── SJF: Requires burst time knowledge, starves long processes
├── RR: Unnecessary overhead for batch processes
├── Priority: Risk of starvation, complex tuning
└── None serves all process types well

Solution:
└── Different algorithms for different queues
```

---

## MLQ Architecture

### Basic Structure

A typical MLQ system has the following components:

```
Multilevel Queue System Architecture:

┌─────────────────────────────────────────────────────────┐
│                    Scheduler                             │
│                                                         │
│  ┌───────────────┐                                      │
│  │ Queue 0       │  System Processes                    │
│  │ (Highest)     │  Algorithm: Priority or RR           │
│  └───────┬───────┘                                      │
│          │                                              │
│  ┌───────┴───────┐                                      │
│  │ Queue 1       │  Interactive Processes               │
│  │               │  Algorithm: Round Robin              │
│  └───────┬───────┘                                      │
│          │                                              │
│  ┌───────┴───────┐                                      │
│  │ Queue 2       │  Batch Processes                     │
│  │               │  Algorithm: FCFS                     │
│  └───────┬───────┘                                      │
│          │                                              │
│  ┌───────┴───────┐                                      │
│  │ Queue 3       │  Background Processes                │
│  │ (Lowest)      │  Algorithm: FCFS                     │
│  └───────────────┘                                      │
│                                                         │
│  Scheduling between queues: Fixed Priority or Time Slice│
└─────────────────────────────────────────────────────────┘
```

### Queue Classification Criteria

Processes are assigned to queues based on various criteria:

```
Classification Methods:

1. Process Type
   ├── System processes
   ├── Interactive processes
   ├── Batch processes
   └── Background processes

2. Memory Requirements
   ├── Small memory footprint → higher priority
   ├── Large memory footprint → lower priority
   └── Prevents memory thrashing

3. Process Priority
   ├── User-specified priority
   ├── System-assigned priority
   └── Real-time requirements

4. Resource Requirements
   ├── CPU-intensive processes
   ├── I/O-intensive processes
   └── Balanced processes

5. User Type
   ├── System administrator
   ├── Regular users
   └── Guest users
```

---

## Scheduling Between Queues

The key decision in MLQ is how to allocate CPU time between different queues. Two main approaches exist:

### 1. Fixed Priority Preemptive Scheduling

Each queue has an absolute priority over lower-priority queues:

```
Fixed Priority Rules:
├── Queue 0 (highest) runs before Queue 1
├── Queue 1 runs before Queue 2
├── Queue 2 runs before Queue 3
├── Lower queue runs only when all higher queues empty
└── Preemptive: new process in higher queue preempts lower queue

Visual Representation:

Timeline with Fixed Priority:
Queue 0: [P1][P2]      [P3]         [P4]
Queue 1:        [P5][P6]   [P7]
Queue 2:                            [P8][P9]
         ─────────────────────────────────────▶
         System processes always preempt others
```

### 2. Time-Sliced Scheduling

Each queue receives a percentage of CPU time:

```
Time-Slice Allocation Example:
├── Queue 0 (System): 50% of CPU time
├── Queue 1 (Interactive): 30% of CPU time
├── Queue 2 (Batch): 15% of CPU time
└── Queue 3 (Background): 5% of CPU time

Round-Robin between queues:
Time:    0    50    80    95   100
         |-----|-----|-----|-----|
Queue:    Q0    Q1    Q2    Q3    Q0 ...

Alternative: Proportional allocation
Queue 0 gets 2x time slice of Queue 1
Queue 1 gets 2x time slice of Queue 2
And so on...
```

### Comparison of Approaches

| Aspect                 | Fixed Priority          | Time-Sliced          |
|------------------------|------------------------|----------------------|
| Responsiveness         | Excellent for high priority | Good for all queues  |
| Starvation Risk        | High for low queues    | Low (all queues get time) |
| Implementation         | Simple                 | More complex         |
| Flexibility            | Rigid priorities        | Adjustable time slices|
| Use Case               | System with clear priority hierarchy | Mixed workloads needing fairness  |
| Overhead               | Low                    | Higher (timer management) |

---

## Scheduling Within Queues

Each queue can use a different scheduling algorithm suited to its process type:

### Example Configuration

```
Queue 0 (System Processes):
├── Algorithm: Priority Scheduling (preemptive)
├── Rationale: System processes have internal priorities
├── Time Quantum: N/A (run to completion or block)
└── Preemption: High priority system process preempts lower

Queue 1 (Interactive Processes):
├── Algorithm: Round Robin
├── Quantum: Small (10-50 ms)
├── Rationale: Responsive to user input
└── Preemption: Time slice expiration

Queue 2 (Batch Processes):
├── Algorithm: FCFS or SJF
├── Quantum: N/A (non-preemptive)
├── Rationale: Maximize throughput
└── Preemption: Only by higher queues

Queue 3 (Background Processes):
├── Algorithm: FCFS
├── Quantum: N/A
├── Rationale: Simple, runs when nothing else needs CPU
└── Preemption: Constantly preempted by higher queues
```

### Algorithm Selection Guidelines

| Process Type         | Recommended Algorithm  | Why                                          |
|----------------------|------------------------|----------------------------------------------|
| System               | Priority               | Internal system priorities matter            |
| Interactive          | Round Robin (small quantum) | Quick response to all users                  |
| Batch                | SJF or FCFS            | Optimize throughput                          |
| Background           | FCFS                   | Simplicity, runs when CPU idle              |

---

## Detailed Example

### Scenario Setup

Consider a system with three queues:

```
System Configuration:
├── Queue 0: System processes (Priority Scheduling)
├── Queue 1: Interactive processes (Round Robin, q=4ms)
├── Queue 2: Batch processes (FCFS)
└── Scheduling between queues: Fixed Priority Preemptive
```

### Process Set

| Process | Type        | Queue      | Arrival Time | CPU Burst | Priority (within queue) |
|---------|-------------|------------|--------------|-----------|--------------------------|
| P1      | System      | Q0         | 0            | 8 ms      | High                     |
| P2      | Interactive | Q1         | 0            | 10 ms     | N/A (RR)                 |
| P3      | Interactive | Q1         | 1            | 6 ms      | N/A (RR)                 |
| P4      | Batch       | Q2         | 0            | 20 ms     | N/A (FCFS)               |
| P5      | System      | Q0         | 3            | 4 ms      | Medium                   |
| P6      | Batch       | Q2         | 2            | 8 ms      | N/A (FCFS)               |

### Execution Timeline

```
Fixed Priority Preemptive MLQ Execution:

Time:    0    4    8    12   16   20   24   28   32   36   40   44   48   52   56
         |----|----|----|----|----|----|----|----|----|----|----|----|----|----|
         
Q0:      [P1  ][P1  ][P5  ][P5  ]
Q1:            [P2  ][P2  ][P3  ][P2  ][P3  ][P2  ]
Q2:                              [P4  ][P4  ][P4  ][P4  ][P4  ][P6  ][P6  ]

Detailed Timeline:

0-4 ms: Q0 runs P1 (highest priority queue)
        Q1 has P2, Q2 has P4 waiting

4-8 ms: P1 continues (system process, non-preemptive within queue)

8-12 ms: P5 arrives at 3ms, runs now (system process)
         P1 completed at 8ms

12-16 ms: P5 continues

16-20 ms: Q0 empty, Q1 runs P2 (Round Robin quantum=4ms)

20-24 ms: Q1 runs P3 (RR next in queue)

24-28 ms: Q1 runs P2 again (RR cycle)

28-32 ms: Q1 runs P3 (completes P3's remaining 2ms)
          P3 done, P2 still has 2ms remaining

32-36 ms: Q1 runs P2 (completes P2's last 2ms)
          Q1 empty now

36-56 ms: Q2 runs P4 (FCFS, batch process)
          P4 runs for 20ms

56-64 ms: Q2 runs P6 (next in FCFS batch queue)
          P6 runs for 8ms

All processes complete at 64ms
```

### Performance Metrics

```
P1 (System): 
├── Waiting: 0 ms
├── Turnaround: 8 ms
└── Response: 0 ms

P2 (Interactive):
├── Waiting: 4 + 8 + 8 = 20 ms
├── Turnaround: 36 ms
└── Response: 16 ms (first runs at 16ms)

P3 (Interactive):
├── Waiting: 3 + 4 + 4 = 11 ms
├── Turnaround: 32 ms
└── Response: 20 ms (first runs at 20ms)

P4 (Batch):
├── Waiting: 16 ms
├── Turnaround: 56 ms
└── Response: 36 ms (first runs at 36ms)

P5 (System):
├── Waiting: 5 ms (arrived at 3, ran at 8)
├── Turnaround: 16 ms
└── Response: 5 ms

P6 (Batch):
├── Waiting: 54 ms
├── Turnaround: 64 ms
└── Response: 56 ms (first runs at 56ms)

Average Waiting: (0 + 20 + 11 + 16 + 5 + 54) / 6 = 17.67 ms
Average Turnaround: (8 + 36 + 32 + 56 + 16 + 64) / 6 = 35.33 ms
```

---

## Types of Multilevel Queue Configurations

### 1. Two-Queue System (Simplest)

```
Simple Two-Level Configuration:

┌─────────────────────┐
│  Foreground Queue    │  Interactive processes
│  (Round Robin)      │  Small time quantum
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Background Queue    │  Batch processes
│  (FCFS)             │  Run when foreground empty
└─────────────────────┘

Used in: Simple timesharing systems
Advantages: Simple, separates interactive from batch
Disadvantages: Too coarse for complex systems
```

### 2. Three-Queue System (Common)

```
Three-Level Configuration:

┌─────────────────────┐
│  Queue 0: System     │  Kernel processes
│  (Priority)         │  Very high priority
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 1: Interactive│  User applications
│  (Round Robin)      │  Medium priority
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 2: Batch      │  Background jobs
│  (FCFS)             │  Low priority
└─────────────────────┘

Used in: Many traditional OS designs
Advantages: Good balance of simplicity and functionality
Disadvantages: Limited flexibility
```

### 3. Four-Queue System (Comprehensive)

```
Four-Level Configuration:

┌─────────────────────┐
│  Queue 0: Real-Time  │  Hard deadlines
│  (EDF/Rate Monotonic)│  Highest priority
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 1: System     │  Kernel processes
│  (Priority)         │  Very high priority
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 2: Interactive│  User applications
│  (Round Robin)      │  Medium priority
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 3: Batch      │  Background jobs
│  (FCFS)             │  Low priority
└─────────────────────┘

Used in: Real-time capable systems
Advantages: Handles real-time, system, interactive, and batch
Disadvantages: Complexity, potential starvation
```

### 4. Five-Queue System (Extended)

```
Extended Configuration with Multiple Priorities:

┌─────────────────────┐
│  Queue 0: Critical   │  Emergency processes
│  (Priority)         │  Must run immediately
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 1: Real-Time  │  Deadline-driven
│  (EDF)              │  Guaranteed timing
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 2: System     │  Kernel services
│  (Priority)         │  OS functionality
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 3: Interactive│  User-facing
│  (Round Robin)      │  Responsive
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Queue 4: Background │  Low urgency
│  (FCFS)             │  Opportunistic
└─────────────────────┘

Used in: Complex real-time and embedded systems
Advantages: Fine-grained control
Disadvantages: Significant complexity
```

---

## Process Classification

### How are processes assigned to queues?

Several methods exist:

#### 1. Static Classification

Processes are assigned to a queue at creation time and never move:

```
Static Assignment Methods:

Based on Process Type:
├── System processes → Queue 0
├── Interactive processes → Queue 1
├── Batch processes → Queue 2
└── Assigned by process creator or OS

Based on User Priority:
├── Root/admin processes → Higher queues
├── Regular user processes → Middle queues
├── Background user processes → Lower queues
└── Unix "nice" values influence assignment

Based on Resource Requirements:
├── Small memory footprint → Higher queues
├── Large memory footprint → Lower queues
├── High CPU requirements → Lower queues
└── I/O intensive → Higher queues
```

#### 2. Dynamic Classification

Processes can move between queues based on behavior (though this overlaps with Multilevel Feedback Queues):

```
Dynamic Reclassification Triggers:

Process Behavior Changes:
├── Interactive process becomes batch → Move down
├── Batch process needing interactivity cannot move up
└── Requires monitoring and thresholds

Resource Usage Patterns:
├── Exceeds CPU time limit → Move down
├── Frequent I/O operations → Move up
└── Memory usage changes → Reclassify

User Interaction:
├── User brings process to foreground → Move up
├── User backgrounds process → Move down
└── Explicit priority changes → Reclassify
```

### Classification Criteria Examples

```
Real-World Classification (Linux-like):

Real-Time Queue (Highest):
├── Processes with SCHED_FIFO or SCHED_RR policy
├── Requires root privileges
├── Kernel threads
└── Audio/video processing

System Queue:
├── Init/systemd
├── Kernel daemons
├── Device drivers
└── Critical services

Interactive Queue:
├── GUI applications
├── Terminal sessions
├── Web browsers
└── Text editors

Batch Queue (Lowest):
├── Compilation jobs
├── Backup processes
├── Log rotation
├── System updates
└── Scientific computations
```

---

## Advantages of Multilevel Queues

1. **Appropriate Scheduling for Different Process Types**

```
Each process type gets suitable treatment:

Interactive Processes:
├── Round Robin with small quantum
├── Quick response to user input
├── Frequent scheduling opportunities
└── Smooth user experience

Batch Processes:
├── FCFS or SJF
├── Long uninterrupted execution
├── Minimal context switch overhead
└── Optimal throughput

System Processes:
├── Highest priority
├── Preemptive over user processes
├── Critical operations complete quickly
└── System stability maintained
```

2. **Clear Priority Structure**

```
Priority hierarchy is explicit and understandable:

Advantages:
├── System administrators understand scheduling
├── Predictable behavior
├── Easy to configure
├── Documentation straightforward
└── Troubleshooting simplified

Example: If system is slow:
├── Check if batch queue is starving
├── Verify interactive queue configuration
├── Examine system queue for runaway processes
└── Adjust queue parameters as needed
```

3. **Protection from Process Interference**

```
Process isolation benefits:

Interactive processes protected:
├── Batch job cannot preempt interactive work
├── User applications remain responsive
└── System doesn't freeze during background tasks

System processes protected:
├── User processes cannot delay kernel work
├── Critical operations complete on time
└── System stability enhanced
```

4. **Flexible Configuration**

```
Each queue independently configurable:

Adjustable Parameters:
├── Queue priority
├── Scheduling algorithm per queue
├── Time quantum (if RR)
├── Queue size limits
├── Preemption behavior
├── Classification criteria

This allows system tuning for:
├── Different workload types
├── Changing system requirements
├── Performance optimization
└── Resource allocation policies
```

---

## Disadvantages of Multilevel Queues

1. **Starvation Risk**

```
Fixed priority between queues can starve lower queues:

Starvation Scenario:
├── Continuous system process activity
├── Interactive processes never get CPU
├── Batch processes starve completely
└── Background work never completes

Example:
System monitoring daemon wakes every 10ms
├── Always has work in Queue 0
├── Lower queues starve
├── User applications freeze
└── System appears hung

Mitigation:
├── Time-slicing between queues
├── Aging mechanisms
├── Guaranteed minimum CPU time
└── Starvation detection and alerts
```

2. **Rigidity**

```
Processes cannot move between queues (in basic MLQ):

Problems:
├── Process behavior changes not accommodated
├── Interactive process becoming batch stays in interactive queue
├── Batch process needing interactivity cannot move up
├── Misclassification permanent
└── No adaptation to changing workloads

Consequences:
├── Inefficient scheduling
├── Poor resource utilization
├── User frustration
└── Manual intervention required
```

3. **Configuration Complexity**

```
Setting up MLQ requires many decisions:

Questions to Answer:
├── How many queues?
├── What priority for each queue?
├── Which algorithm per queue?
├── What time quantum for RR queues?
├── How to classify processes?
├── Fixed priority or time-slicing?
└── Preemption rules?

Tuning Challenges:
├── Optimal configuration workload-dependent
├── Changes over time
├── Difficult to predict
└── Requires expertise
```

4. **Overhead**

```
Additional overhead compared to single queue:

Scheduling Overhead:
├── Queue selection logic
├── Priority comparisons
├── Multiple queue management
├── Classification overhead
└── Potential timer overhead for time-slicing

Context Switch Implications:
├── Higher priority queue arrivals cause preemption
├── More context switches
├── Cache effects
└── Reduced throughput for lower queues
```

---

## MLQ vs. Other Scheduling Algorithms

### Comparison with Single-Queue Algorithms

| Aspect             | MLQ                   | FCFS                   | SJF                    | RR                     | Priority               |
|--------------------|----------------------|------------------------|------------------------|------------------------|------------------------|
| Complexity          | High                 | Very Low               | Medium                 | Low                    | Medium                 |
| Responsiveness      | Excellent for high priority | Poor                | Good                   | Excellent              | Variable               |
| Throughput          | Good                 | Poor                   | Excellent              | Moderate               | Variable               |
| Starvation Risk     | High (lower queues)  | None                   | High (long jobs)       | None                   | High (low priority)    |
| Flexibility         | High                 | None                   | Low                    | Moderate               | Moderate               |
| Overhead            | Moderate-High        | Minimal                 | Low                    | Moderate               | Low                    |

### When to Use MLQ

MLQ is most appropriate when:

```
Good Use Cases:

1. Systems with Clear Process Categories
   ├── Real-time systems with distinct priority levels
   ├── Server systems with system/user separation
   └── Embedded systems with fixed task types

2. Mixed Workloads
   ├── Combination of interactive and batch
   ├── Systems with both critical and non-critical work
   └── Environments with varying process requirements

3. Predictable Environments
   ├── Process characteristics known in advance
   ├── Stable workload patterns
   └── Controlled execution environment

4. When Process Mobility Not Required
   ├── Processes don't change behavior significantly
   ├── Classification remains valid over time
   └── Static assignment acceptable
```

---

## Implementation Considerations

### Data Structures

```
MLQ Implementation Components:

struct mlq_scheduler {
    // Array of ready queues
    ready_queue *queues[NUM_QUEUES];
    
    // Scheduling algorithm per queue
    sched_algorithm algorithms[NUM_QUEUES];
    
    // Time slice per queue (if applicable)
    int time_slices[NUM_QUEUES];
    
    // Current active queue
    int current_queue;
    
    // Scheduling between queues
    inter_queue_policy policy;  // 0=Fixed Priority, 1=Time Sliced
};

ready_queue {
    Process *head;
    Process *tail;
    int size;
    int max_size;
};
```

### Scheduling Decision Logic

```
MLQ Scheduling Algorithm:

function mlq_schedule():
    if (policy == FIXED_PRIORITY):
        // Find highest priority non-empty queue
        for i in 0 to NUM_QUEUES-1:
            if (queues[i].size > 0):
                return select_from_queue(i)
        return null  // All queues empty
    
    else if (policy == TIME_SLICED):
        // Rotate through queues
        for i in 0 to NUM_QUEUES-1:
            queue_index = (current_queue + i) % NUM_QUEUES
            if (queues[queue_index].size > 0):
                current_queue = queue_index
                return select_from_queue(queue_index)
        return null  // All queues empty

function select_from_queue(queue_index):
    algorithm = algorithms[queue_index]
    queue = queues[queue_index]
    
    switch(algorithm):
        case FCFS:
            return dequeue_front(queue)
        case ROUND_ROBIN:
            return dequeue_round_robin(queue)
        case PRIORITY:
            return dequeue_highest_priority(queue)
        case SJF:
            return dequeue_shortest_job(queue)
```

### Preemption Handling

```
Preemption Rules in MLQ:

1. Higher Queue Arrival:
   ├── New process arrives in higher priority queue
   ├── Current process is in lower queue
   └── Decision: Preempt current process

2. Within Same Queue:
   ├── Algorithm-dependent
   ├── RR: Time quantum expiration
   ├── Priority: Higher priority process arrival
   └── FCFS: No preemption

3. Time-Slice Expiration (between queues):
   ├── Current queue's time slice exhausted
   ├── Move to next queue in rotation
   └── Save current process state
```

---

## Real-World Examples

### 1. Traditional Unix Systems

```
Unix-like MLQ Configuration:

Priority Levels (Solaris example):
├── Interrupt threads (highest)
├── Real-time processes
├── System processes
├── Interactive processes
├── Batch processes
└── Idle process (lowest)

Implementation:
├── Fixed priority between classes
├── Time-sharing within interactive class
├── Real-time processes always preempt
└── Idle process runs only when nothing else
```

### 2. Windows Scheduling

```
Windows Priority Classes:

Priority Classes (32 levels):
├── Real-time (16-31): System critical
├── High (13-15): Important system
├── Above Normal (10-12): User elevated
├── Normal (7-9): Default user
├── Below Normal (4-6): Background
└── Idle (1-3): Screen savers, background

Within each class:
├── Round Robin for same priority
├── Dynamic priority adjustment
├── I/O completion boosts priority
└── Thread priority relative to class
```

### 3. Real-Time Systems

```
RTOS MLQ Configuration (VxWorks-like):

Priority Queues:
├── Queue 0: Critical tasks (deadline < 1ms)
├── Queue 1: High priority (deadline < 10ms)
├── Queue 2: Medium priority (deadline < 100ms)
├── Queue 3: Low priority (no strict deadline)
└── Queue 4: Background (opportunistic)

Characteristics:
├── Fixed priority preemptive
├── Strict priority ordering
├── No time-slicing between queues
├── Real-time tasks never starved
└── Background tasks may never run
```

---

### MLQ Configuration Examples

#### Example 1: Desktop System

```
Desktop MLQ Configuration:

Queue 0: System Processes
├── Scheduling: Priority
├── Quantum: N/A
├── Processes: Kernel, drivers, services
└── Goal: System stability

Queue 1: Interactive Applications
├── Scheduling: Round Robin
├── Quantum: 20 ms
├── Processes: GUI apps, editors, browsers
└── Goal: Responsiveness

Queue 2: Background Applications
├── Scheduling: FCFS
├── Processes: Compilers, media encoders
└── Goal: Complete when foreground empty

Queue 3: System Maintenance
├── Scheduling: FCFS
├── Processes: Log rotation, cleanup
└── Goal: Opportunistic execution

Inter-Queue: Fixed Priority (Q0 > Q1 > Q2 > Q3)
```

#### Example 2: Server System

```
Server MLQ Configuration:

Queue 0: Critical Services
├── Scheduling: Priority
├── Processes: Database, web server
└── Goal: Serve requests quickly

Queue 1: Network Processing
├── Scheduling: Round Robin
├── Quantum: 10 ms
├── Processes: Packet processing, I/O handling
└── Goal: High throughput

Queue 2: User Jobs
├── Scheduling: FCFS
├── Processes: Batch processing, reports
└── Goal: Complete eventually

Queue 3: Maintenance
├── Scheduling: FCFS
├── Processes: Log rotation, cleanup
└── Goal: Opportunistic execution

Inter-Queue: Time-Sliced (50%, 30%, 15%, 5%)
```

#### Example 3: Real-Time System

```
Real-Time MLQ Configuration:

Queue 0: Hard Real-Time
├── Scheduling: Earliest Deadline First
├── Processes: Control loops, safety systems
└── Goal: Meet all deadlines

Queue 1: Soft Real-Time
├── Scheduling: Rate Monotonic
├── Processes: Audio/video processing
└── Goal: Meet most deadlines

Queue 2: System
├── Scheduling: Priority
├── Processes: OS services
└── Goal: System functionality

Queue 3: Non-Critical
├── Scheduling: Round Robin
├── Processes: Logging, diagnostics
└── Goal: Best effort

Inter-Queue: Fixed Priority, strict ordering
```

---

## Performance Analysis

### Metrics Comparison

Consider a workload with mixed process types:

```
Workload:
├── 2 System processes (short bursts)
├── 3 Interactive processes (medium bursts)
├── 2 Batch processes (long bursts)
└── All arrive at time 0

MLQ Configuration:
├── Q0 (System): Priority, fixed priority highest
├── Q1 (Interactive): RR, quantum=10ms
├── Q2 (Batch): FCFS, lowest priority
```

vs. Single RR (quantum=10ms):

| Metric               | MLQ                     | Single RR              |
|----------------------|------------------------|------------------------|
| Response Time        | ~0ms                   | ~0-10ms                |
| Turnaround           | Minimal                | Moderate               |
| CPU Utilization      | Higher                 | Lower                  |

### Bottleneck Analysis

```
MLQ Performance Bottlenecks:

1. High-Priority Queue Saturation
   ├── Queue 0 always has work
   ├── Lower queues starve
   └── Solution: Time-slicing or admission control

2. Queue Transition Overhead
   ├── Preemption when higher queue gets process
   ├── Context switch costs
   └── Solution: Batch arrivals, hysteresis

3. Classification Errors
   ├── Process in wrong queue
   ├── Inefficient scheduling
   └── Solution: Dynamic reclassification

4. Timer Management
   ├── Multiple time quanta to track
   ├── Timer interrupt overhead
   └── Solution: Efficient timer data structures
```

---

## MLQ Variants

### 1. Multilevel Feedback Queue (MLFQ)

The most important MLQ variant—processes can move between queues:

```
MLFQ Features:
├── Processes start in highest queue
├── Use too much CPU → move down
├── Starving too long → move up
├── Adaptive to process behavior
└── Combines MLQ with feedback

Advantages over MLQ:
├── No permanent classification
├── Adapts to changing behavior
├── Reduces starvation
└── Better overall performance

(Detailed in separate document)
```

### 2. Priority Inheritance MLQ

Handles priority inversion problems:

```
Priority Inheritance:
├── Low-priority process holds lock
├── High-priority process waits for lock
├── Low-priority process temporarily inherits high priority
├── Prevents priority inversion
└── Common in real-time MLQ systems

Example:
Without inheritance:
Q0: High priority process waits for lock
Q2: Low priority process holds lock, runs slowly
Q1: Medium processes preempt Q2
Result: Q0 starves despite highest priority

With inheritance:
Q0: High priority process waits for lock
Q2: Low priority process inherits Q0's priority
Q1: Cannot preempt (lower than inherited priority)
Result: Lock released quickly, Q0 resumes
```

### 3. Proportional Share MLQ

Each queue gets guaranteed CPU percentage:

```
Proportional Share:
├── Queues have weights
├── CPU allocated proportional to weights
├── Uses virtual time concept
├── Ensures minimum service
└── Prevents complete starvation

Example:
Queue weights: Q0=5, Q1=3, Q2=2
Total weight = 10
CPU allocation: Q0=50%, Q1=30%, Q2=20%

Implementation:
├── Track virtual time per queue
├── Schedule queue with lowest virtual time
├── When queue runs, virtual time advances
├── Proportional to weight
└── Ensures fair allocation
```

---

## Common Problems and Solutions

### Problem 1: Starvation

```
Symptom: Lower queue processes never execute

Detection:
├── Monitor process waiting times
├── Check if any queue never empties
├── Log starvation incidents

Solutions:
├── Implement aging (gradually increase priority)
├── Use time-slicing between queues
├── Reserve minimum CPU time for each queue
├── Implement starvation detection and alerts
└── Allow manual priority adjustment
```

### Problem 2: Priority Inversion

```
Symptom: High-priority process blocked by low-priority process

Scenario:
├── High-priority process needs lock held by low-priority
├── Low-priority process preempted by medium-priority
├── High-priority process waits indefinitely
└── System appears hung

Solutions:
├── Priority inheritance
├── Priority ceiling protocol
├── Lock-free algorithms where possible
├── Careful lock design
└── Real-time specific solutions
```

### Problem 3: Configuration Drift

```
Symptom: System performance degrades over time

Cause:
├── Workload characteristics change
├── Original queue configuration suboptimal
├── New process types not anticipated
└── No automatic adaptation

Solutions:
├── Periodic configuration review
├── Automated performance monitoring
├── Dynamic adjustment mechanisms
├── Feedback-based tuning
└── Consider MLFQ instead
```

### Problem 4: Queue Overflow

```
Symptom: Processes rejected or lost when queue full

Cause:
├── Queue size limits reached
├── Burst of process arrivals
├── Lower queues not draining
└── Insufficient queue capacity

Solutions:
├── Dynamic queue resizing
├── Admission control for new processes
├── Priority-based queue admission
├── Load shedding (drop lowest priority)
└── Back-pressure mechanisms
```

---

## Implementation Example

### Simplified MLQ Implementation

```c
// Simplified Multilevel Queue Scheduler

#include <stdio.h>
#include <stdlib.h>

#define NUM_QUEUES 3
#define QUEUE_SYSTEM 0
#define QUEUE_INTERACTIVE 1
#define QUEUE_BATCH 2

typedef enum {
    ALG_FCFS,
    ALG_ROUND_ROBIN,
    ALG_PRIORITY
} SchedAlgorithm;

typedef struct Process {
    int pid;
    int arrival_time;
    int burst_time;
    int remaining_time;
    int priority;
    int queue_level;
    struct Process *next;
} Process;

typedef struct {
    Process *head;
    Process *tail;
    int size;
    SchedAlgorithm algorithm;
    int quantum;
} ReadyQueue;

typedef struct {
    ReadyQueue queues[NUM_QUEUES];
    int policy;  // 0=Fixed Priority, 1=Time Sliced
    int current_queue;
    int queue_time_remaining;
} MLQScheduler;

// Initialize scheduler
void init_scheduler(MLQScheduler *sched) {
    // Queue 0: System processes (Priority)
    sched->queues[0].head = NULL;
    sched->queues[0].tail = NULL;
    sched->queues[0].size = 0;
    sched->queues[0].algorithm = ALG_PRIORITY;
    sched->queues[0].quantum = 0;
    
    // Queue 1: Interactive processes (Round Robin)
    sched->queues[1].head = NULL;
    sched->queues[1].tail = NULL;
    sched->queues[1].size = 0;
    sched->queues[1].algorithm = ALG_ROUND_ROBIN;
    sched->queues[1].quantum = 10;
    
    // Queue 2: Batch processes (FCFS)
    sched->queues[2].head = NULL;
    sched->queues[2].tail = NULL;
    sched->queues[2].size = 0;
    sched->queues[2].algorithm = ALG_FCFS;
    sched->queues[2].quantum = 0;
    
    sched->policy = 0;  // Fixed priority
    sched->current_queue = 0;
    sched->queue_time_remaining = 0;
}

// Add process to appropriate queue
void add_process(MLQScheduler *sched, Process *proc, int queue_level) {
    ReadyQueue *queue = &sched->queues[queue_level];
    proc->queue_level = queue_level;
    proc->next = NULL;
    
    if (queue->tail == NULL) {
        queue->head = proc;
        queue->tail = proc;
    } else {
        queue->tail->next = proc;
        queue->tail = proc;
    }
    queue->size++;
}

// Select next process from a queue
Process* select_from_queue(ReadyQueue *queue) {
    if (queue->size == 0) return NULL;
    
    Process *selected = NULL;
    Process *prev = NULL;
    Process *curr = queue->head;
    
    switch (queue->algorithm) {
        case ALG_FCFS:
            // Remove from head
            selected = queue->head;
            queue->head = queue->head->next;
            if (queue->head == NULL) queue->tail = NULL;
            break;
            
        case ALG_ROUND_ROBIN:
            // Remove from head (same as FCFS for this simple implementation)
            selected = queue->head;
            queue->head = queue->head->next;
            if (queue->head == NULL) queue->tail = NULL;
            break;
            
        case ALG_PRIORITY:
            // Find highest priority process
            Process *highest_prev = NULL;
            Process *highest = queue->head;
            prev = queue->head;
            curr = queue->head->next;
            
            while (curr != NULL) {
                if (curr->priority < highest->priority) {
                    highest_prev = prev;
                    highest = curr;
                }
                prev = curr;
                curr = curr->next;
            }
            
            // Remove highest priority process
            if (highest_prev == NULL) {
                queue->head = highest->next;
                if (queue->head == NULL) queue->tail = NULL;
            } else {
                highest_prev->next = highest->next;
                if (highest->next == NULL) queue->tail = highest_prev;
            }
            selected = highest;
            break;
    }
    
    if (selected != NULL) {
        queue->size--;
    }
    return selected;
}

// Main scheduling function
Process* schedule(MLQScheduler *sched) {
    if (sched->policy == 0) {
        // Fixed priority: find highest non-empty queue
        for (int i = 0; i < NUM_QUEUES; i++) {
            if (sched->queues[i].size > 0) {
                sched->current_queue = i;
                return select_from_queue(&sched->queues[i]);
            }
        }
    } else {
        // Time sliced: rotate through queues
        if (sched->queue_time_remaining <= 0) {
            // Move to next queue
            sched->current_queue = (sched->current_queue + 1) % NUM_QUEUES;
            sched->queue_time_remaining = 10;  // Default slice
        }
        
        for (int i = 0; i < NUM_QUEUES; i++) {
            int q = (sched->current_queue + i) % NUM_QUEUES;
            if (sched->queues[q].size > 0) {
                return select_from_queue(&sched->queues[q]);
            }
        }
    }
    return NULL;
}

// Handle time quantum expiration
void handle_quantum_expired(MLQScheduler *sched, Process *proc) {
    if (sched->queues[proc->queue_level].algorithm == ALG_ROUND_ROBIN) {
        // Re-add to tail of same queue
        add_process(sched, proc, proc->queue_level);
    } else {
        // Non-preemptive queue, process continues
        // (In real implementation, process wouldn't be interrupted)
    }
}
```

---

## Summary

Multilevel Queue scheduling partitions processes into multiple queues based on their characteristics, with each queue potentially using a different scheduling algorithm. This approach allows the system to provide appropriate scheduling for different process types—responsive Round Robin for interactive processes, efficient FCFS for batch processes, and highest priority for system processes.

### Key Points

1. MLQ divides processes into multiple queues based on type, priority, or resource requirements.
2. Each queue can use a different scheduling algorithm suited to its process type.
3. Fixed priority or time-slicing determines CPU allocation between queues.
4. Process classification can be static or dynamic.
5. Starvation of lower queues is a significant risk with fixed priority scheduling.
6. MLQ provides flexibility but requires careful configuration.
7. Real-world systems often use MLQ concepts, even if not in pure form.
8. MLFQ extends MLQ by allowing processes to move between queues.

---

## Key Takeaways

1. Different process types need different scheduling approaches—MLQ recognizes and addresses this.
2. Fixed priority between queues provides clear precedence but risks starvation.
3. Time-slicing between queues is fairer but more complex to implement.
4. Process classification is critical—correct assignment determines scheduling effectiveness.
5. MLQ works best when process types are known and stable.
6. The main limitation is lack of adaptability to changing process behavior.
7. MLFQ addresses this limitation by allowing queue migration.
8. Understanding MLQ is essential for understanding modern scheduling systems.

---

## Further Reading

- [Multilevel-Feedback-Queue.md](Multilevel-Feedback-Queue.md): MLQ variant allowing process movement between queues
- [Priority-Scheduling.md](Priority-Scheduling.md): Algorithm commonly used within MLQ queues
- [Round-Robin.md](Round-Robin.md): Scheduling algorithm for interactive queues
- [Real-World-Scheduling.md](Real-World-Scheduling.md): How modern OSes implement MLQ concepts
- [Scheduling-Basics.md](Scheduling-Basics.md): Fundamental scheduling concepts

