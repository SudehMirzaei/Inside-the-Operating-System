# FCFS (First-Come, First-Served) Scheduling

## Introduction

First-Come, First-Served (FCFS) is the simplest CPU scheduling algorithm—and also one of the oldest. As its name suggests, FCFS schedules processes in the exact order they arrive in the ready queue. The first process to request the CPU gets it first; the second waits until the first is done, and so on.

While FCFS is rarely used as the primary scheduling algorithm in modern operating systems, understanding it is essential because:

1. It provides a baseline for comparing other algorithms
2. It illustrates fundamental scheduling concepts and problems
3. It demonstrates why more sophisticated algorithms were developed
4. It's still used in specific contexts where simplicity is paramount

This document explores FCFS in depth: how it works, its advantages, its significant drawbacks, and the important lessons it teaches about CPU scheduling.

---

## How FCFS Works

### The Basic Principle

FCFS operates on a simple principle: processes are executed in the order they arrive. The ready queue is a simple FIFO (First-In, First-Out) queue.

```
FCFS Algorithm:

1. Process arrives and enters ready queue
2. Process is placed at the tail of the queue
3. When CPU becomes available:
   ├── Process at head of queue gets the CPU
   ├── Process runs to completion (or until it blocks)
   └── Process is removed from queue
4. If process blocks for I/O:
   ├── Process moves to waiting queue
   ├── Next process in ready queue gets CPU
   └── When I/O completes, process returns to ready queue tail
5. Repeat from step 3
```

### Visual Representation

```
Ready Queue (FIFO):

    Tail                        Head
    ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
    │ P5  │→│ P4  │→│ P3  │→│ P2  │→│ P1  │→ CPU
    └─────┘  └─────┘  └─────┘  └─────┘
    (Newest)                    (Oldest)
    
    P1 arrived first → P1 gets CPU first
    P2 arrived second → P2 goes next
    And so on...
```

### Pseudocode

```
FCFS Scheduler:

function schedule():
    if ready_queue is empty:
        return null  // No process to schedule
    
    if current_process is running:
        // Non-preemptive: don't interrupt
        return current_process
    
    // Select process at head of queue
    next_process = ready_queue.dequeue()
    return next_process

function on_process_arrival(process):
    ready_queue.enqueue(process)

function on_process_completion(process):
    // Process is done, schedule next
    if ready_queue is not empty:
        next_process = ready_queue.dequeue()
        dispatch(next_process)

function on_process_block(process):
    // Process blocked for I/O
    if ready_queue is not empty:
        next_process = ready_queue.dequeue()
        dispatch(next_process)
```

---

## Detailed Examples

### Example 1: Basic FCFS Execution

#### Scenario: Three processes arrive in order P1, P2, P3 at time 0 with CPU burst times:

| Process | Arrival Time | Burst Time |
|---------|--------------|------------|
| P1      | 0            | 24 ms      |
| P2      | 0            | 3 ms       |
| P3      | 0            | 3 ms       |

### Gantt Chart:

```
0                        24   27   30
|─────────────────────────|────|────|
            P1              P2    P3

Timeline:
  0 ms: P1 starts execution (first in queue)
 24 ms: P1 completes, P2 starts (next in queue)
 27 ms: P2 completes, P3 starts
 30 ms: P3 completes, all processes done
```

### Calculations:

```
P1: Completion Time = 24 ms
    Turnaround Time = 24 - 0 = 24 ms
    Waiting Time = 24 - 24 = 0 ms

P2: Completion Time = 27 ms
    Turnaround Time = 27 - 0 = 27 ms
    Waiting Time = 27 - 3 = 24 ms

P3: Completion Time = 30 ms
    Turnaround Time = 30 - 0 = 30 ms
    Waiting Time = 30 - 3 = 27 ms

Average Turnaround Time = (24 + 27 + 30) / 3 = 27 ms
Average Waiting Time = (0 + 24 + 27) / 3 = 17 ms
```

### Example 2: Different Arrival Times

#### Scenario: Processes arrive at different times:

| Process | Arrival Time | Burst Time |
|---------|--------------|------------|
| P1      | 0            | 8 ms       |
| P2      | 1            | 4 ms       |
| P3      | 2            | 9 ms       |
| P4      | 3            | 5 ms       |

### Gantt Chart:

```
0        8        12              21       26
|────────|────────|───────────────|────────|
    P1      P2          P3           P4

Timeline:
  0 ms: P1 arrives, CPU idle, P1 starts
  1 ms: P2 arrives, joins queue behind P1
  2 ms: P3 arrives, joins queue behind P2
  3 ms: P4 arrives, joins queue behind P3
  8 ms: P1 completes, P2 starts
 12 ms: P2 completes, P3 starts
 21 ms: P3 completes, P4 starts
 26 ms: P4 completes
```

### Calculations:

```
P1: Turnaround = 8 - 0 = 8 ms
    Waiting = 8 - 8 = 0 ms

P2: Turnaround = 12 - 1 = 11 ms
    Waiting = 11 - 4 = 7 ms

P3: Turnaround = 21 - 2 = 19 ms
    Waiting = 19 - 9 = 10 ms

P4: Turnaround = 26 - 3 = 23 ms
    Waiting = 23 - 5 = 18 ms

Average Turnaround = (8 + 11 + 19 + 23) / 4 = 15.25 ms
Average Waiting = (0 + 7 + 10 + 18) / 4 = 8.75 ms
```

### Example 3: FCFS with I/O Operations

#### Real processes perform I/O, which affects FCFS behavior:

#### Scenario: Two processes with I/O operations:

| Process | CPU Burst 1 | I/O Wait | CPU Burst 2 |
|---------|--------------|----------|--------------|
| P1      | 10 ms        | 20 ms    | 10 ms        |
| P2      | 5 ms         | 10 ms    | 5 ms         |

### Gantt Chart:

```
0        10        15        30        40        45
|────────|────────|─────────|─────────|────────|
  P1(CPU) P2(CPU)  P1(I/O)   P1(CPU)   P2(CPU)
                    P2(I/O)             
                    
Detailed Timeline:
  0-10 ms: P1 executes (first CPU burst)
  10-30 ms: P1 waits for I/O (20 ms)
  10-15 ms: P2 executes (first CPU burst) — CPU was idle
  15-25 ms: P2 waits for I/O (10 ms)
  15-30 ms: Both processes waiting, CPU IDLE
  30-40 ms: P1 executes (second CPU burst) — P1's I/O completed
  40-45 ms: P2 executes (second CPU burst) — P2's I/O completed at 25ms, waited 15ms
```

### Calculations:

```
P1: Turnaround = 40 - 0 = 40 ms
    CPU Time = 20 ms
    Waiting Time = 40 - 20 - 20(I/O) = 0 ms (only waits when blocked for I/O)

P2: Turnaround = 45 - 0 = 45 ms
    CPU Time = 10 ms
    Waiting Time = 45 - 10 - 10(I/O) = 25 ms (includes 15ms waiting while P1 runs)

CPU Utilization = 30 ms busy / 45 ms total = 66.7%
```

*Note: FCFS with I/O allows other processes to run when the current process blocks, but the non-preemptive nature still causes inefficiency (CPU idle from 25-30 ms).*

---

## Characteristics of FCFS

### Algorithm Properties

| Property                           | Value                                           | Explanation                                           |
|------------------------------------|-------------------------------------------------|------------------------------------------------------|
| Preemptive?                        | No                                              | Process runs until it blocks or completes            |
| Starvation possible?               | No                                              | Every process eventually reaches queue head          |
| Priority levels?                   | None                                            | All processes treated equally                         |
| Requires burst time knowledge?     | No                                              | Only arrival order matters                            |
| Implementation complexity           | Very low                                       | Simple FIFO queue                                    |
| Overhead                           | Minimal                                         | No complex calculations needed                        |

### Advantages

```
1. Simple to Implement
   ├── Just a FIFO queue
   ├── Minimal code required
   ├── Easy to debug
   └── Predictable behavior

2. Low Overhead
   ├── No complex calculations
   ├── O(1) queue operations
   ├── Minimal context switches
   └── No timer interrupt needed for scheduling

3. Fair in Arrival Order
   ├── Every process eventually runs
   ├── No starvation possible
   ├── Transparent to users
   └── Natural queuing behavior

4. Deterministic
   ├── Same input → same schedule
   ├── Easy to reproduce results
   └── Useful for debugging
```

### Disadvantages

```
1. The Convoy Effect
   ├── Short processes wait behind long ones
   ├── Average waiting time increases dramatically
   ├── Poor system responsiveness
   └── I/O devices underutilized

2. Non-Preemptive Nature
   ├── Cannot respond to high-priority events
   ├── Interactive systems feel unresponsive
   ├── One long process blocks all others
   └── Real-time requirements cannot be met

3. Poor Average Metrics
   ├── High average waiting time
   ├── High average turnaround time
   ├── Suboptimal CPU utilization with I/O
   └── Worst-case performance for short jobs

4. No Priority Support
   ├── All processes treated equally
   ├── System processes wait for user processes
   ├── Cannot handle emergency situations
   └── Unsuitable for mixed workloads
```

---

## The Convoy Effect in Detail

The convoy effect is FCFS's most famous problem, named by analogy to a convoy of slow trucks blocking faster cars on a single-lane road.

### What is the Convoy Effect?

When a CPU-bound process (long CPU burst) runs first, all I/O-bound processes (short CPU bursts) queue up behind it. During the long CPU burst, I/O devices sit idle. When the CPU-bound process finally blocks for I/O, the I/O-bound processes quickly finish their CPU bursts and block for I/O, leaving the CPU idle while the CPU-bound process does I/O.

```
Convoy Effect Illustration:

System Configuration:
├── 1 CPU
├── 1 disk drive
├── P1: CPU-bound (long bursts, rare I/O)
├── P2, P3, P4: I/O-bound (short bursts, frequent I/O)

Timeline with FCFS:

Time:    0        50       60       70       100      110
         |         |        |        |        |        |
CPU:     [P1       ][P2][P3][P4][IDLE       ][P2][P3][P4]
              CPU-bound        I/O-bound processes    
              process runs     run quickly, then block

Disk:    [IDLE     ][P2    ][P3    ][P4    ][P1 I/O  ][P2][P3]
              Disk idle!    I/O-bound processes    CPU idle!
              (50ms wasted)  all doing I/O          (30ms wasted)

Result:
├── CPU and Disk alternate being busy/idle
├── Poor resource utilization
├── Long average waiting times
└── System feels sluggish
```

### Quantifying the Convoy Effect

#### Example Calculation:

Processes:

- P1: CPU-bound, 50 ms CPU burst, 10 ms I/O, 50 ms CPU burst
- P2-P5: I/O-bound, 5 ms CPU burst, 20 ms I/O, 5 ms CPU burst

### FCFS Order (P1 first):

```
0        50        55        60        65
|────────|────────|─────────|─────────|────────|
    P1(CPU) P2(CPU)  P1(I/O)   P1(CPU)   P2(CPU)
```

### Average Waiting Time:

- P1: 0 ms
- P2: Waits 50 ms (behind P1)
- P3: Waits 55 ms (behind P1 and P2)
- P4: Waits 60 ms (behind P1, P2, and P3)

**Total Average:** 
Average Waiting Time = (0 + 50 + 55 + 60) / 4 = 41.25 ms

### Optimal Order (I/O-bound first):

```
0    5    25     30            125
|----|----|-----|---------------|
   P2   P3   P4       P1(CPU)
```

Average Waiting Time = (15 + 0 + 5 + 10) / 4 = 7.5 ms

**Improvement:** 41.25 / 7.5 = 5.5x better

---

## FCFS with Different Process Types

### CPU-Bound Processes

```
CPU-Bound Process Behavior:
├── Long CPU bursts
├── Rare I/O operations
├── Benefits from long execution periods
└── FCFS actually works reasonably well

Example:
P1 (CPU-bound): 100 ms burst
P2 (CPU-bound): 50 ms burst

FCFS: P1 (0-100ms), P2 (100-150ms)
Turnaround: P1=100ms, P2=150ms
Average: 125ms

SJF would be: P2 (0-50ms), P1 (50-150ms)
Turnaround: P1=150ms, P2=50ms
Average: 100ms

FCFS is 25% worse than optimal for this case
```

### I/O-Bound Processes

```
I/O-Bound Process Behavior:
├── Short CPU bursts
├── Frequent I/O operations
├── Needs prompt CPU service after I/O completes
└── Suffers badly under FCFS

Example:
P1 (CPU-bound): 100 ms CPU burst
P2 (I/O-bound): 5 ms CPU, 20 ms I/O, 5 ms CPU

FCFS Timeline:
0        100      105      125      130
|────────|────────|─────────|─────────|
  P1(CPU)  P2(CPU)  P2(I/O)  P2(CPU)

P2's response: Waits 100ms before first CPU allocation
P2's I/O device: Idle for 100ms (should have been used!)

Optimal Timeline:
0    5    25   30             125
|----|----|----|----------------|
  P2   P2   P2      P1(CPU)

P2's response: Immediate (0ms wait)
P2's I/O device: Busy from 5-25ms (optimal utilization)
P1 unaffected significantly (still completes at ~125ms)
```

### Mixed Workloads

FCFS performs poorly with mixed workloads because it doesn't account for process characteristics:

```
Mixed Workload Issues:

1. Short interactive processes wait behind batch jobs
   ├── User clicks button → waits 200ms for batch job
   └── System feels unresponsive

2. I/O-bound processes starve I/O devices
   ├── Disk sits idle while CPU-bound process runs
   └── Overall system throughput decreases

3. No mechanism to prioritize urgent processes
   ├── System update can't preempt user application
   └── Critical tasks wait in line like everyone else

4. Poor cache behavior with long-running processes
   ├── Long process may thrash cache
   └── Short processes lose cache warmth while waiting
```

---

## Implementation Details

### Data Structures

FCFS requires minimal data structures:

```
Ready Queue Implementation:

Option 1: Simple Linked List
struct node {
    Process *process;
    struct node *next;
};

struct queue {
    struct node *head;
    struct node *tail;
};

Operations:
├── enqueue(process): O(1) - add to tail
├── dequeue(): O(1) - remove from head
├── peek(): O(1) - look at head
└── is_empty(): O(1)

Option 2: Circular Array (bounded number of processes)
Option 3: Kernel's built-in list structure (Linux list_head)
```

### Integration with Process States

```
FCFS and Process State Transitions:

                    ┌─────────────────────────────┐
                    │                             │
   New ──▶ Ready ──▶ Running ──▶ Terminated     │
             ▲         │                         │
             │         │ blocks for I/O          │
             │         ▼                         │
             └──── Waiting ◀─────────────────────┘
                       │
                       │ I/O completes
                       └──────▶ Ready (tail of queue)

Key FCFS Behaviors:
├── Running → Ready: Never happens (non-preemptive)
├── Running → Waiting: Process blocks, schedule next
├── Running → Terminated: Process done, schedule next
├── Waiting → Ready: Process returns to queue tail
└── Ready → Running: Process reaches queue head
```

### Example Implementation (Pseudo-C)

```c
// Simplified FCFS scheduler implementation

typedef struct Process {
    int pid;
    int arrival_time;
    int burst_time;
    int remaining_time;
    int state;  // READY, RUNNING, WAITING, TERMINATED
    // ... other PCB fields
} Process;

typedef struct QueueNode {
    Process *process;
    struct QueueNode *next;
} QueueNode;

typedef struct ReadyQueue {
    QueueNode *head;
    QueueNode *tail;
    int size;
} ReadyQueue;

// Initialize ready queue
ReadyQueue* create_ready_queue() {
    ReadyQueue *q = malloc(sizeof(ReadyQueue));
    q->head = NULL;
    q->tail = NULL;
    q->size = 0;
    return q;
}

// Add process to ready queue (at tail)
void enqueue(ReadyQueue *q, Process *p) {
    QueueNode *node = malloc(sizeof(QueueNode));
    node->process = p;
    node->next = NULL;
    
    if (q->tail == NULL) {
        q->head = node;
        q->tail = node;
    } else {
        q->tail->next = node;
        q->tail = node;
    }
    q->size++;
    
    p->state = READY;
}

// Remove process from ready queue (from head)
Process* dequeue(ReadyQueue *q) {
    if (q->head == NULL) {
        return NULL;  // Queue empty
    }
    
    QueueNode *node = q->head;
    Process *p = node->process;
    q->head = node->next;
    
    if (q->head == NULL) {
        q->tail = NULL;
    }
    
    free(node);
    q->size--;
    return p;
}

// FCFS scheduler
Process* fcfs_schedule(ReadyQueue *q, Process *current) {
    if (current != NULL && current->state == RUNNING) {
        // Non-preemptive: don't interrupt running process
        return current;
    }
    
    // Select next process from queue head
    return dequeue(q);
}

// Handle process arrival
void on_process_arrival(ReadyQueue *q, Process *p) {
    enqueue(q, p);
}

// Handle process blocking (I/O wait)
Process* on_process_block(ReadyQueue *q, Process *p) {
    p->state = WAITING;
    // Move to I/O wait queue (not shown)
    return fcfs_schedule(q, NULL);
}

// Handle process completion
Process* on_process_complete(ReadyQueue *q, Process *p) {
    p->state = TERMINATED;
    return fcfs_schedule(q, NULL);
}

// Handle I/O completion
void on_io_complete(ReadyQueue *q, Process *p) {
    p->state = READY;
    enqueue(q, p);  // Return to ready queue tail
}
```

---

## Mathematical Analysis

### Average Waiting Time

For FCFS, the average waiting time depends heavily on arrival order:

```
Given n processes with burst times b1, b2, ..., bn

Waiting time for process i (when all arrive at time 0):
Wi = b1 + b2 + ... + b(i-1) = Σ(j=1 to i-1) bj

Average Waiting Time:
W_avg = (1/n) * Σ(i=1 to n) Wi
      = (1/n) * Σ(i=1 to n) Σ(j=1 to i-1) bj
      = (1/n) * Σ(j=1 to n) (n - j) * bj

Example (bursts: 24, 3, 3):
W_avg = (1/3) * [(3-1)*24 + (3-2)*3 + (3-3)*3]
      = (1/3) * [48 + 3 + 0]
      = 17 ms

This matches our earlier calculation!
```

### Worst-Case Analysis

FCFS can produce pathological schedules:

```
Worst Case: Short processes arrive after long processes

Processes:
├── P1: 100 ms burst (arrives first)
├── P2: 1 ms burst (arrives second)
├── P3: 1 ms burst (arrives third)

FCFS Order: P1, P2, P3
Average Waiting: (0 + 100 + 101) / 3 = 67 ms

Optimal Order (SJF): P2, P3, P1
Average Waiting: (0 + 1 + 2) / 3 = 1 ms

FCFS is 67x worse than optimal!
```

### The Convoy Effect Formula

```
Given:
├── One CPU-bound process with burst time B
├── n I/O-bound processes with small burst times s1, s2, ..., sn
└── All arrive at time 0, CPU-bound process arrives first

Additional waiting for I/O-bound processes:
├── Each waits at least B time units
├── Total extra waiting = n * B
└── Average extra waiting = B

If I/O-bound processes had run first:
├── CPU-bound process waits Σ(si) time units
├── Total waiting = Σ(si) (typically small)
└── Dramatically better average waiting time
```

---

## FCFS in Practice

### Where FCFS is Still Used

Despite its drawbacks, FCFS appears in various contexts:

```
1. Batch Processing Systems
   ├── Print queues
   ├── Job submission systems
   ├── Background processing
   └── Where fairness by arrival is important

2. Simple Embedded Systems
   ├── Microcontrollers with simple tasks
   ├── No complex scheduling needed
   ├── Predictable execution order
   └── Minimal overhead requirements

3. Specialized Queue Systems
   ├── Network packet queues (FIFO discipline)
   ├── Message queues
   ├── Request handling in web servers (basic)
   └── Database transaction logs

4. As a Component of Larger Systems
   ├── Within priority levels (equal priority = FCFS)
   ├── Fallback when other criteria tie
   └── Queue discipline in multilevel queues
```

### Real-World Examples

```
Example 1: Print Spooler
├── Jobs printed in submission order
├── Fair and predictable
├── Users expect FIFO behavior
├── No priority needed for most printing
└── FCFS is appropriate

Example 2: Batch Job Processing
├── Scientific computing clusters
├── Jobs submitted to queue
├── Executed in order (or with resource constraints)
├── FCFS ensures fairness
└── Long-running jobs expected

Example 3: Simple Task Execution
├── cron jobs (scheduled tasks)
├── Startup scripts
├── System initialization
├── Sequential operations
└── Order matters more than efficiency
```

### Where FCFS is Inappropriate

```
1. Interactive Systems
   ├── Desktop operating systems
   ├── Mobile devices
   ├── Web servers handling user requests
   └── Any system requiring responsiveness

2. Real-Time Systems
   ├── Industrial control
   ├── Audio/video processing
   ├── Medical devices
   ├── Automotive systems
   └── Deadline-driven scheduling required

3. Mixed Workloads
   ├── Systems with both CPU and I/O bound processes
   ├── Servers with varied request types
   ├── Cloud computing environments
   └── Where convoy effect causes problems
```

---

## FCFS Variants and Extensions

### 1. FCFS with Priority Classes

```
Modified FCFS:
├── Multiple FCFS queues at different priorities
├── Higher priority queue always served first
├── Within each queue, FCFS discipline
└── Balances fairness with priority handling

Structure:
Priority 1 (High): [P1] → [P2] → [P3]  (FCFS within)
Priority 2 (Medium): [P4] → [P5]       (FCFS within)
Priority 3 (Low): [P6] → [P7] → [P8] → [P9]  (FCFS within)

Scheduler: Always serve highest non-empty priority queue
```

### 2. FCFS with Aging

```
FCFS with Aging:
├── Processes gain "age" while waiting
├── Age acts as implicit priority
├── Prevents indefinite starvation
├── Modifies pure FCFS behavior

Implementation:
├── Each process has age counter
├── Age increases while in ready queue
├── When age exceeds threshold, process moves to front
└── Essentially becomes priority scheduling with aging
```

### 3. FCFS for Disk Scheduling

```
Disk Scheduling FCFS:
├── Disk requests served in arrival order
├── Simple but inefficient
├── Causes excessive seek time
├── Better algorithms exist (SCAN, C-SCAN, etc.)
└── Demonstrates FCFS limitations in I/O context
```

---

## Comparison with Other Algorithms

### FCFS vs. SJF (Shortest Job First)

```
Example (all arrive at time 0):
Processes: P1(10ms), P2(5ms), P3(2ms)

FCFS:
0        10       15       17
|--------|--------|--------|
    P1      P2       P3
Average Waiting: (0 + 10 + 15) / 3 = 8.33 ms

SJF:
0    2    7              17
|----|----|----------------|
   P3   P2       P1
Average Waiting: (7 + 2 + 0) / 3 = 3 ms

SJF is 2.78x better for average waiting time
```

### FCFS vs. Round Robin

```
Example (all arrive at time 0):
Processes: P1(10ms), P2(5ms), P3(2ms)

FCFS:
0        10       15       17
|--------|--------|--------|
    P1      P2       P3
Response Time: P1=0, P2=10, P3=15
Average Response: 8.33 ms

Round Robin (quantum = 2ms):
0  2  4  6  8  10 12 14 16 17
|--|--|--|--|--|--|--|--|--|
 P1 P2 P3 P1 P2 P1 P2 P1 P1
Response Time: P1=0, P2=2, P3=4
Average Response: 2 ms

RR provides 4x better average response time
But higher context switch overhead (9 switches vs. 2)
```

### FCFS vs. Priority Scheduling

```
Example:
P1 (priority 1, low), P2 (priority 3, medium), P3 (priority 5, high)
Bursts: P1=10ms, P2=5ms, P3=2ms

FCFS:
0        10       15       17
|--------|--------|--------|
    P1      P2       P3
Result: Low priority process runs first, high priority waits

Priority:
0    2    7              17
|----|----|----------------|
   P3   P2       P1
Result: High priority runs first, better for urgent tasks
```

---

## Common Misconceptions

### Misconception 1: "FCFS is always fair"

**Reality:** FCFS is fair in terms of arrival order, but unfair in terms of resource usage. A process that arrives slightly earlier but runs for hours causes significant delays for all later processes. "Fair" in this context means different things to different stakeholders.

### Misconception 2: "FCFS is never used in practice"

**Reality:** FCFS appears in many real systems, especially where simplicity and predictability are valued over optimal performance. Print queues, batch job schedulers, and simple embedded systems often use FCFS.

### Misconception 3: "FCFS has no overhead"

**Reality:** While FCFS has minimal scheduling overhead (no complex calculations), it can cause significant system inefficiency through poor resource utilization, effectively "wasting" CPU cycles through the convoy effect and idle I/O devices.

### Misconception 4: "FCFS is the same as FIFO"

**Reality:** FCFS refers specifically to CPU scheduling where processes execute in arrival order. FIFO (First-In, First-Out) is a more general queue discipline used in many contexts. FCFS uses FIFO queues but adds scheduling semantics.

### Misconception 5: "FCFS with I/O is preemptive"

**Reality:** FCFS remains non-preemptive even with I/O. When a process blocks for I/O, it voluntarily yields the CPU (the scheduler doesn't take it away). The process is not interrupted against its will; it simply cannot continue until I/O completes.

---

## Hands-On Exercises

### Exercise 1: Basic Calculation

**Problem:** Four processes arrive at time 0 with burst times: P1=6, P2=8, P3=7, P4=3. Calculate average waiting time and average turnaround time using FCFS.

**Solution:**

```
Order: P1, P2, P3, P4 (arrival order)

Gantt Chart:
0    6        14       21       24
|----|--------|--------|--------|
   P1    P2       P3       P4

Waiting Times:
P1: 0
P2: 6
P3: 14
P4: 21

Average Waiting = (0 + 6 + 14 + 21) / 4 = 10.25 ms

Turnaround Times:
P1: 6 - 0 = 6
P2: 14 - 0 = 14
P3: 21 - 0 = 21
P4: 24 - 0 = 24

Average Turnaround = (6 + 14 + 21 + 24) / 4 = 16.25 ms
```

### Exercise 2: Different Arrival Times

**Problem:** Processes with different arrival times:

| Process | Arrival | Burst |
|---------|---------|-------|
| P1      | 0       | 5     |
| P2      | 2       | 3     |
| P3      | 4       | 4     |
| P4      | 6       | 2     |

**Solution:**

```
Timeline:
0    5    8    12   14
|----|----|----|----|

