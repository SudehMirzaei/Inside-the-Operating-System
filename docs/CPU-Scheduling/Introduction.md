# Introduction to CPU Scheduling

## Overview

Welcome to the CPU Scheduling section of Inside-the-Operating-System. This module explores one of the most critical responsibilities of any operating system: deciding which process gets to use the CPU at any given moment. CPU scheduling is where the operating system's theoretical concepts meet practical implementation—it directly affects system performance, user experience, and resource utilization.

In this introduction, we'll establish the context for understanding CPU scheduling: why it exists, what problems it solves, how it fits into the broader operating system architecture, and what you'll learn throughout this section.

---

## What is CPU Scheduling?

CPU scheduling is the mechanism by which the operating system determines which process (or thread) should execute on the CPU at any given time. When multiple processes are ready to run, the scheduler—a core component of the OS kernel—must make rapid decisions about resource allocation.

### The Core Question

```
Multiple processes are ready to execute.
Only one CPU (or core) is available.
Which process should run next?
```

This seemingly simple question has profound implications for system performance, responsiveness, and fairness.

### A Simple Analogy

Think of CPU scheduling like a restaurant with one chef (the CPU) and many customers (processes) who have placed orders:

- The chef can only cook one dish at a time.
- Some orders are quick (like salads—short CPU bursts).
- Some orders take long (like roasts—long CPU bursts).
- Customers have different priorities (VIP guests—high-priority processes).
- The restaurant wants to maximize throughput while keeping customers happy.

The scheduler is like the kitchen manager who decides which order the chef prepares next.

---

## Why CPU Scheduling Matters

### The Multi-Process Reality

Modern computer systems run hundreds of processes simultaneously:

```
Typical Desktop System:
├── Operating System Services (50+ processes)
│   ├── Window manager
│   ├── Network daemons
│   ├── File system services
│   └── Security services
├── User Applications (10-30 processes)
│   ├── Web browser (multiple processes/threads)
│   ├── Text editor
│   ├── Media player
│   └── Email client
└── Background Services
    ├── Update checkers
    ├── Backup agents
    └── System monitors
```

On a system with 4 CPU cores, dozens of processes compete for execution time. The scheduler must ensure fair and efficient allocation.

### The Impact of Scheduling Decisions

Scheduling decisions affect:

| Aspect                | Impact                                                    |
|----------------------|----------------------------------------------------------|
| System Responsiveness | How quickly interactive applications respond to user input|
| Throughput           | How many jobs complete per unit time                    |
| Resource Utilization  | How efficiently CPU, I/O devices, and memory are used   |
| User Experience      | Whether the system feels smooth or sluggish              |
| Application Performance| How long individual programs take to complete           |
| System Fairness      | Whether all processes get reasonable CPU time            |

A well-designed scheduler can make a modest system feel fast and responsive. A poorly designed one can make powerful hardware feel sluggish.

---

## Historical Context

Understanding the evolution of CPU scheduling provides insight into why modern schedulers work the way they do.

### Era 1: Batch Processing (1950s-1960s)

```
Characteristics:
├── No scheduling needed (single job at a time)
├── Jobs submitted on punch cards
├── CPU sits idle during I/O operations
└── Goal: Maximize CPU utilization

Scheduling Evolution:
└── Simple sequential execution → Basic job scheduling
```

**Problem:** The CPU was expensive, and idle time was unacceptable. Operators manually scheduled jobs to reduce idle time.

### Era 2: Multiprogramming (1960s-1970s)

```
Characteristics:
├── Multiple jobs loaded into memory simultaneously
├── CPU switches to another job when one waits for I/O
├── Goal: Keep CPU busy at all times
└── Non-preemptive scheduling initially

Key Innovation:
└── The ready queue - jobs waiting for CPU time
```

**Problem:** Without preemption, one long-running job could block all others.

### Era 3: Time-Sharing (1970s-1980s)

```
Characteristics:
├── Interactive computing becomes mainstream
├── Preemptive scheduling with time slices
├── Multiple users share one system
├── Goal: Provide responsive interaction
└── Round-robin and priority-based scheduling

Key Innovation:
└── The timer interrupt enables preemption
```

**Problem:** Balancing responsiveness with throughput and fairness.

### Era 4: Personal Computing (1980s-2000s)

```
Characteristics:
├── Single user, multiple applications
├── GUI applications demand responsiveness
├── Background tasks compete with foreground tasks
├── Goal: Prioritize interactive experience
└── Multilevel feedback queues

Key Innovation:
└── Dynamic priority adjustment based on behavior
```

**Problem:** Different applications have vastly different scheduling needs.

### Era 5: Multi-Core and Mobile (2000s-Present)

```
Characteristics:
├── Multiple cores require parallel scheduling
├── Power efficiency becomes critical
├── Real-time requirements for mobile devices
├── Goal: Balance performance and power consumption
└── Completely Fair Scheduler (Linux), advanced Windows scheduler

Key Innovations:
├── Per-core scheduling decisions
├── Power-aware scheduling
├── Real-time scheduling classes
└── NUMA-aware scheduling
```

---

## The Scheduling Problem Space

CPU scheduling operates within a complex problem space with multiple dimensions:

### 1. Process Characteristics

Processes vary widely in their behavior and requirements:

```
                    Process Characteristics
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
    CPU-bound          I/O-bound          Real-time
        │                  │                  │
    Long CPU           Frequent I/O       Hard deadlines
    bursts             operations         Must meet timing
        │                  │                  │
    Example:           Example:           Example:
    Video              Web browser        Audio processing
    encoding           Database           Industrial control
```

### 2. System Requirements

Different systems have different scheduling priorities:

| System Type | Primary Goals            | Typical Users                        |
|-------------|--------------------------|--------------------------------------|
| Desktop     | Responsiveness, fairness  | Single user                          |
| Server      | Throughput, stability     | Many concurrent users                |
| Real-time   | Determinism, meeting deadlines | Embedded systems, control systems  |
| Mobile      | Power efficiency, responsiveness | Single user, battery-powered       |
| Batch       | Throughput, resource utilization | Scientific computing               |

### 3. Available Resources

```
Scheduling must account for:
├── Number of CPU cores
├── CPU speed and capabilities
├── Memory hierarchy (cache sizes, NUMA)
├── I/O subsystem performance
├── Power budget (mobile, embedded)
└── Real-time constraints
```

---

## The Scheduler's Role in the OS Architecture

CPU scheduling sits at the heart of the operating system kernel:

```
                    User Space
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    Process A      Process B      Process C
        │              │              │
        └──────────────┼──────────────┘
                       │
                 System Call Interface
                       │
┌──────────────────────┼──────────────────────┐
│                      │                      │
│              KERNEL SPACE                   │
│                      │                      │
│   ┌──────────────────┼──────────────────┐   │
│   │                  │                  │   │
│   │           CPU SCHEDULER             │   │
│   │                  │                  │   │
│   │    ┌─────────────┼─────────────┐    │   │
│   │    │             │             │    │   │
│   │  Process      Memory        I/O     │   │
│   │  Manager      Manager       Manager │   │
│   │    │             │             │    │   │
│   └────┼─────────────┼─────────────┼────┘   │
│        │             │             │        │
│      Hardware Abstraction Layer             │
└──────────────────────┬──────────────────────┘
                       │
                    Hardware
                   (CPU, RAM, I/O)
```

### Scheduler Interactions

The scheduler doesn't operate in isolation—it interacts with every other OS component:

| Component           | Interaction                                               |
|---------------------|----------------------------------------------------------|
| Process Manager     | Scheduler uses process states, PCBs                      |
| Memory Manager      | Context switches involve memory mapping changes          |
| I/O Manager         | I/O completion triggers scheduling decisions              |
| Interrupt Handler    | Timer interrupts drive preemptive scheduling              |
| System Call Interface| Process creation/termination invokes scheduler            |

---

## What You'll Learn in This Section

This section progressively builds your understanding of CPU scheduling:

### Foundational Concepts

```
Scheduling-Basics.md
├── Why scheduling is necessary
├── CPU-I/O burst cycles
├── Types of schedulers
├── Scheduling metrics
└── Preemptive vs. non-preemptive
```

### Core Algorithms

```
FCFS.md
├── First-Come-First-Served
├── Simple but flawed
└── Convoy effect

SJF.md
├── Shortest Job First
├── Optimal for average waiting time
└── Requires knowledge of burst times

Round-Robin.md
├── Time-slice based
├── Fair and responsive
└── Quantum size trade-offs

Priority-Scheduling.md
├── Priority-based selection
├── Preemptive and non-preemptive
└── Starvation problem
```

### Advanced Topics

```
Multilevel-Queues.md
├── Multiple ready queues
├── Different algorithms per queue
└── Queue assignment strategies

Real-World-Scheduling.md
├── Linux CFS
├── Windows scheduling
├── macOS scheduling
└── Mobile OS considerations
```

---

## Prerequisites

Before diving into CPU scheduling, you should be familiar with:

### From OS Fundamentals

- Process states and lifecycles
- Kernel vs. user space
- Interrupts and exceptions
- Context switching basics

### From Process Management

- Process Control Blocks (PCBs)
- Process creation and termination
- Threads and their relationship to scheduling
- The ready queue concept

### Conceptual Prerequisites

- Basic data structures (queues, linked lists, trees, heaps)
- Algorithm analysis (big-O notation)
- Basic understanding of computer architecture (CPU, cache, interrupts)

---

## Key Terminology Preview

Here's a quick reference to terms you'll encounter throughout this section:

| Term                   | Definition                                                            |
|------------------------|-----------------------------------------------------------------------|
| Scheduler              | Kernel component that selects processes for CPU execution             |
| Ready Queue            | Data structure holding processes waiting for CPU                      |
| Time Quantum            | Fixed time slice allocated to a process in preemptive scheduling      |
| Context Switch         | Operation of saving/restoring process state when switching            |
| Burst Time             | Duration a process needs the CPU before blocking                      |
| Preemption             | Forcibly removing a process from the CPU                             |
| Starvation             | A process never getting scheduled due to lower priority              |
| Convoy Effect          | Short processes stuck behind long ones                                |
| Throughput             | Number of processes completed per unit time                          |
| Latency                | Time between an event and its response                               |

---

## Common Misconceptions

Let's address some common misconceptions about CPU scheduling:

### Misconception 1: "Scheduling only matters on single-core systems"

**Reality:** Multi-core systems still require scheduling. Each core needs to decide which process runs next. Additionally, multi-core scheduling introduces new challenges like load balancing and cache affinity.

### Misconception 2: "The scheduler runs continuously"

**Reality:** The scheduler runs only at specific points—when a process blocks, when an interrupt occurs, or when a time slice expires. It's event-driven, not continuous.

### Misconception 3: "Processes execute continuously between scheduling decisions"

**Reality:** Processes are constantly interrupted, not just by the scheduler but also by hardware interrupts for I/O, timers, and other events.

### Misconception 4: "Scheduling is the same on all operating systems"

**Reality:** While the basic concepts are universal, implementations vary significantly. Linux, Windows, and macOS each use fundamentally different scheduling algorithms.

### Misconception 5: "The scheduler can perfectly predict process behavior"

**Reality:** Schedulers work with limited information. They can estimate future behavior based on history, but cannot predict with certainty how long a process will need the CPU.

---

## The Big Picture

CPU scheduling connects to everything else in the operating system:

### Upstream Dependencies

```
Process Management
├── Creates processes that need scheduling
├── Defines process states
└── Provides PCB information

Memory Management
├── Context switches involve memory mapping changes
├── Cache and TLB effects impact scheduling decisions
└── Swapping interacts with scheduling
```

### Downstream Effects

```
Scheduling decisions affect:
├── System call performance
├── I/O system utilization
├── File system responsiveness
├── Network performance
└── Overall user experience
```

### System-Wide Optimization

The scheduler is part of a larger optimization problem:

```
                    Resource Allocation
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
    CPU Time           Memory Space      I/O Bandwidth
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                   System Performance
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   Responsiveness      Throughput      Resource Utilization
```

The scheduler's decisions about CPU allocation ripple through the entire system.

---

## Practical Examples

Let's look at real scenarios to illustrate scheduling challenges:

### Scenario 1: The Unresponsive Application

```
User reports: "My video editor freezes when I export a video"

What's happening:
├── Video export is CPU-intensive (long CPU bursts)
├── Without proper scheduling, export monopolizes CPU
├── UI process can't get CPU time to respond
└── System appears frozen

Scheduling solution:
└── Priority scheduling or MLFQ ensures UI gets periodic CPU time
```

### Scenario 2: The Slow File Download

```
User reports: "Downloads are slow while I'm playing games"

What's happening:
├── Download process is I/O-bound (waits for network)
├── Game is CPU/GPU-intensive
├── Download needs quick CPU response when data arrives
└── Delayed scheduling of download process slows throughput

Scheduling solution:
└── I/O-bound processes get priority when I/O completes
```

### Scenario 3: Server Overload

```
Admin reports: "Web server response times increase under load"

What's happening:
├── Many concurrent requests compete for CPU
├── Some requests are quick, others are long-running
├── Long-running requests delay quick ones
└── Average response time increases

Scheduling solution:
└── Priority-based scheduling with aging, or workload-specific tuning
```

---

## Summary

CPU scheduling is a fundamental operating system function that balances competing demands for the processor. It has evolved from simple batch systems to sophisticated modern schedulers that handle multi-core systems, real-time requirements, and power efficiency.

### Key Points

1. CPU scheduling decides which process runs next—a decision made frequently and with significant impact.
2. Processes have diverse characteristics (CPU-bound vs. I/O-bound) that schedulers must accommodate.
3. Different systems have different scheduling priorities—desktop, server, real-time, and mobile systems optimize for different goals.
4. Scheduling has evolved from simple batch processing to complex modern algorithms.
5. The scheduler interacts with all OS components—it's not an isolated subsystem.
6. Common misconceptions about scheduling often stem from oversimplification.

---

## What's Next

Continue to [Scheduling-Basics.md](Scheduling-Basics.md) to learn about:

- The CPU-I/O burst cycle
- Types of schedulers (long-term, medium-term, short-term)
- Key scheduling metrics
- Preemptive vs. non-preemptive scheduling
- Scheduling queues and their implementations

---

## References and Further Reading

### Classic Papers

- Dijkstra, E.W. (1968). "The Structure of the 'THE'-Multiprogramming System"
- Lampson, B.W. (1968). "A Scheduling Philosophy for Multiprocessing Systems"
- Corbató, F.J., et al. (1962). "An Experimental Time-Sharing System"

### Textbooks

- Silberschatz, Galvin, Gagne. "Operating System Concepts" (Chapters on CPU Scheduling)
- Tanenbaum, A.S. "Modern Operating Systems" (Scheduling chapters)
- Stallings, W. "Operating Systems: Internals and Design Principles"

### Online Resources

- Linux kernel documentation on CFS
- Windows Internals documentation on scheduling
- Academic papers on real-time scheduling algorithms

