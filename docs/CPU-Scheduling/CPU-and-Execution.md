# CPU and Execution

## Introduction

Before diving into scheduling algorithms, it's essential to understand how the CPU actually executes instructions and why scheduling is fundamentally necessary. The CPU is the workhorse of the computer—it fetches, decodes, and executes billions of instructions per second. Yet, despite its incredible speed, the CPU is often idle, waiting for data from memory, disks, or networks.

This document explores the CPU's execution model, the instruction cycle, hardware features that support scheduling, and the fundamental mismatch between CPU speed and I/O device speed that makes scheduling essential. Understanding these hardware realities helps explain why schedulers make the decisions they do.

---

## The CPU: A Brief Overview

### What is the CPU?

The Central Processing Unit (CPU) is the primary component responsible for executing program instructions. It consists of several key parts:

```
                    CPU Architecture
    ┌─────────────────────────────────────────────┐
    │                                             │
    │   ┌─────────────┐    ┌─────────────────┐   │
    │   │   Control    │    │  Arithmetic &   │   │
    │   │    Unit      │───▶│  Logic Unit     │   │
    │   │   (CU)       │    │  (ALU)          │   │
    │   └─────────────┘    └─────────────────┘   │
    │          │                    │             │
    │          ▼                    ▼             │
    │   ┌─────────────────────────────────────┐   │
    │   │            Registers                │   │
    │   │  (PC, SP, General Purpose, Flags)   │   │
    │   └─────────────────────────────────────┘   │
    │                    │                        │
    │                    ▼                        │
    │   ┌─────────────────────────────────────┐   │
    │   │        Cache Memory (L1, L2, L3)    │   │
    │   └─────────────────────────────────────┘   │
    │                                             │
    └─────────────────────────────────────────────┘
```

### Key CPU Components

| Component                | Function                                        | Relevance to Scheduling                         |
|--------------------------|-------------------------------------------------|------------------------------------------------|
| Control Unit             | Fetches and decodes instructions                | Orchestrates instruction execution              |
| ALU                      | Performs arithmetic and logical operations      | Executes computation                            |
| Registers                | Fast, small storage within CPU                | Hold process state during execution             |
| Program Counter (PC)     | Points to next instruction                      | Must be saved/restored on context switch       |
| Stack Pointer (SP)       | Points to top of stack                          | Saved/restored on context switch                |
| Status Register          | Holds flags and CPU state                       | Indicates interrupt enable/disable              |
| Cache                    | Fast memory near CPU                            | Affects context switch overhead                 |

---

## The Instruction Execution Cycle

The CPU operates in a continuous cycle known as the fetch-decode-execute cycle (or instruction cycle):

### The Basic Cycle

```
        ┌──────────────────────────────────────────┐
        │                                          │
        ▼                                          │
    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
    │  FETCH  │───▶│ DECODE  │───▶│ EXECUTE │─────┘
    └─────────┘    └─────────┘    └─────────┘
        │              │              │
        │              │              │
        ▼              ▼              ▼
   Read instruction  Interpret   Perform operation
   from memory       opcode      (ALU, memory access,
   (at PC address)   and operands control transfer)
```

### Detailed Steps

```
1. FETCH Phase:
   ├── Read instruction from memory address in PC
   ├── Instruction loaded into Instruction Register (IR)
   └── PC incremented to next instruction

2. DECODE Phase:
   ├── Control unit interprets opcode
   ├── Determines operands needed
   ├── Identifies operation type
   └── Sets up execution

3. EXECUTE Phase:
   ├── ALU performs computation, OR
   ├── Memory read/write occurs, OR
   ├── Control flow changes (jump, branch, call)
   └── Results stored in registers or memory

4. INTERRUPT CHECK:
   ├── Check for pending interrupts
   ├── If interrupt pending, handle it
   └── Otherwise, continue with next instruction
```

### Example: Executing a Simple Addition

```
Assembly instruction:  ADD R1, R2, R3
(Add contents of R2 and R3, store result in R1)

C-level:  int result = a + b;

Machine code: 0x00830820 (simplified)
              ├── Opcode: ADD
              ├── Source register 1: R2
              ├── Source register 2: R3
              └── Destination register: R1

Execution steps:
1. FETCH: Read 0x00830820 from memory at PC
2. DECODE: Interpret as ADD operation
3. EXECUTE: ALU adds R2 + R3, stores in R1
4. UPDATE: Set flags (zero, carry, overflow)
5. NEXT: PC points to next instruction
```

---

## The Instruction Pipeline

Modern CPUs use instruction pipelining to execute multiple instructions simultaneously at different stages:

### Pipelined Execution

```
Without Pipelining (5 cycles per instruction):

Instruction 1:  [FETCH][DECODE][EXECUTE][MEMORY][WRITEBACK]
Instruction 2:                                      [FETCH][DECODE]...

Time:           1      2      3       4       5      6      7

With Pipelining (1 instruction per cycle after warm-up):

Instruction 1:  [FETCH][DECODE][EXECUTE][MEMORY][WRITEBACK]
Instruction 2:         [FETCH][DECODE][EXECUTE][MEMORY][WRITEBACK]
Instruction 3:                [FETCH][DECODE][EXECUTE][MEMORY][WRITEBACK]
Instruction 4:                       [FETCH][DECODE][EXECUTE][MEMORY][WRITEBACK]

Time:           1      2      3       4      5      6      7      8
```

### Pipeline Stages (Typical 5-Stage Pipeline)

| Stage | Function                          | Hardware Unit                    |
|-------|-----------------------------------|----------------------------------|
| IF    | Instruction Fetch                 | Fetch unit, instruction cache    |
| ID    | Instruction Decode                | Decoder, register file read      |
| EX    | Execute                           | ALU, address calculation         |
| MEM   | Memory Access                     | Data cache access                |
| WB    | Write Back                        | Register file write              |

### Pipeline Hazards

Pipelines face hazards that reduce efficiency:

```
1. Data Hazards:
   Instruction 2 depends on result of Instruction 1
   
   ADD R1, R2, R3    ; R1 = R2 + R3
   SUB R4, R1, R5    ; R4 = R1 - R5 (needs R1!)
   
   Solution: Forwarding, stalling

2. Control Hazards:
   Branch instructions change execution flow
   
   BEQ R1, R2, target  ; Branch if equal
   ADD R3, R4, R5      ; May not execute if branch taken
   
   Solution: Branch prediction, speculative execution

3. Structural Hazards:
   Two instructions need the same hardware resource
   
   LOAD R1, [addr]   ; Needs memory access
   STORE R2, [addr]  ; Also needs memory access
   
   Solution: Separate caches, more hardware
```

### Relevance to Scheduling

Pipelining affects scheduling in several ways:

- Context switches flush the pipeline: All in-flight instructions are discarded
- Pipeline refill takes time: After a switch, the pipeline must refill (5-15 cycles)
- Frequent context switches reduce throughput: Pipeline overhead becomes significant
- Cache effects amplify: Pipeline relies on fast cache access; cold caches hurt performance

---

## CPU Execution Modes

CPUs operate in different privilege modes that affect what instructions can execute:

### Privilege Levels (Rings)

```
                    x86 Protection Rings

        ┌───────────────────────────────────┐
        │          Ring 0 (Kernel)          │
        │  ┌─────────────────────────────┐  │
        │  │     Ring 1 (Unused)         │  │
        │  │  ┌───────────────────────┐  │  │
        │  │  │   Ring 2 (Unused)     │  │  │
        │  │  │  ┌─────────────────┐  │  │  │
        │  │  │  │  Ring 3 (User)  │  │  │  │
        │  │  │  │                 │  │  │  │
        │  │  │  └─────────────────┘  │  │  │
        │  │  └───────────────────────┘  │  │
        │  └─────────────────────────────┘  │
        └───────────────────────────────────┘
```

| Ring | Privilege | Usage Example Instructions                          |
|------|-----------|----------------------------------------------------|
| 0    | Highest   | Kernel mode; Hardware control, memory management, interrupt handling |
| 1-2  | Medium    | Device drivers (rarely used); Specialized I/O operations |
| 3    | Lowest    | User applications; Regular computation, system calls |

### Mode Transitions

```
User Mode → Kernel Mode:
├── System calls (INT 0x80, SYSCALL instruction)
├── Interrupts (hardware devices need attention)
├── Exceptions (page faults, divide by zero)
└── Traps (debugging, breakpoints)

Kernel Mode → User Mode:
├── Return from interrupt/syscall (IRET, SYSRET)
├── Process scheduling (context switch to user process)
└── Signal delivery
```

### Scheduling Implications

The scheduler runs in kernel mode, where it has full access to hardware:

```
Scheduler Privileges:
├── Can access all memory
├── Can modify page tables
├── Can manipulate process control blocks
├── Can program timer hardware
├── Can disable/enable interrupts
└── Can perform context switches
```

---

## The CPU-Memory Hierarchy

The memory hierarchy has profound implications for scheduling:

### Memory Hierarchy Overview

```
                    Memory Hierarchy

        ┌────────────────────────────────────┐
        │           CPU Registers            │  ~1 cycle
        │         (100s of bytes)            │  ~0.3 ns
        ├────────────────────────────────────┤
        │          L1 Cache                  │  ~4 cycles
        │         (32-64 KB)                 │  ~1 ns
        ├────────────────────────────────────┤
        │          L2 Cache                  │  ~10-14 cycles
        │         (256KB-1MB)                │  ~3-5 ns
        ├────────────────────────────────────┤
        │          L3 Cache                  │  ~40-75 cycles
        │         (4-32 MB)                  │  ~10-20 ns
        ├────────────────────────────────────┤
        │          Main Memory (RAM)         │  ~100-300 cycles
        │         (8-64 GB)                  │  ~50-100 ns
        ├────────────────────────────────────┤
        │       Solid State Drive (SSD)      │  ~100,000+ cycles
        │         (256GB-4TB)                │  ~10-100 μs
        ├────────────────────────────────────┤
        │       Hard Disk Drive (HDD)        │  ~10,000,000+ cycles
        │         (1-20 TB)                  │  ~5-10 ms
        └────────────────────────────────────┘
```

### The Cache Hierarchy

Modern CPUs have multiple levels of cache:

```
                    Cache Architecture

    CPU Core 0              CPU Core 1
    ┌─────────────┐        ┌─────────────┐
    │  Registers  │        │  Registers  │
    ├─────────────┤        ├─────────────┤
    │  L1 Cache   │        │  L1 Cache   │
    │  (I + D)    │        │  (I + D)    │
    ├─────────────┤        ├─────────────┤
    │  L2 Cache   │        │  L2 Cache   │
    └──────┬──────┘        └──────┬──────┘
           │                      │
           └──────────┬───────────┘
                      │
                ┌─────┴─────┐
                │ L3 Cache   │
                │ (Shared)   │
                └─────┬─────┘
                      │
                ┌─────┴─────┐
                │ Main Memory│
                └───────────┘
```

### Cache Characteristics

| Cache Level      | Size              | Latency      | Sharing                      | Typical Content                 |
|------------------|-------------------|--------------|------------------------------|----------------------------------|
| L1 Instruction    | 32-64 KB          | ~4 cycles    | Per core                     | Recently executed instructions   |
| L1 Data           | 32-64 KB          | ~4 cycles    | Per core                     | Recently accessed data           |
| L2                | 256 KB-1 MB       | ~10-14 cycles| Per core                     | Unified instruction+data         |
| L3                | 4-32 MB           | ~40-75 cycles| Shared across cores          | Larger working set               |

### Scheduling and Cache Affinity

When a process runs, it loads data into caches. If it's rescheduled to a different core, it loses that cache warmth:

```
Cache Affinity Concept:

Process P running on Core 0:
├── P's data loaded in Core 0's L1/L2 cache
├── P's instructions in Core 0's instruction cache
├── P's working set in shared L3 cache
└── Fast execution due to cache hits

Process P moved to Core 1:
├── Core 1's L1/L2 cache does NOT contain P's data
├── Core 1 must fetch from L3 or main memory
├── Initial execution is slower (cache misses)
└── Eventually warms up Core 1's cache

Scheduling implications:
├── Prefer scheduling process on same core (cache affinity)
├── Only migrate when necessary (load balancing)
├── Context switch overhead includes cache effects
└── "Warm" cache can be 10-100x faster than "cold" cache
```

---

## CPU Speed vs. I/O Speed Mismatch

The fundamental reason for scheduling is the enormous speed gap between the CPU and I/O devices:

### The Speed Gap

```
Operations possible in 1 second:

CPU: ~3-5 billion instructions executed
├── Register access: ~3 billion
├── L1 cache access: ~1 billion
├── L2 cache access: ~300 million
├── Main memory access: ~30 million

I/O Operations:
├── SSD read: ~10,000-100,000 operations
├── HDD read: ~100-200 operations
├── Network packet: ~1 million packets (small)
└── Keyboard input: ~10 characters (human typing)
```

### Visualizing the Mismatch

```
If CPU cycle = 1 second (metaphorical scaling):

CPU Instruction:        1 second
L1 Cache Access:        4 seconds
L2 Cache Access:        14 seconds
Main Memory Access:     3 minutes
SSD Read:               1-3 days
HDD Read:               1-3 months
Network Round Trip:     6 months
Human Reaction:         10 years
```

### The I/O Wait Problem

Without scheduling, the CPU wastes most of its time waiting for I/O:

```
Single Process Execution (No Scheduling):

Time:    0ms        50ms       100ms      150ms
         |           |           |           |
CPU:     [███████░░░░░░░░░░░░░░░░███████░░░░░]
         |  CPU    |  I/O Wait  |  CPU    | I/O
         |  Burst  |  (Disk)    |  Burst  | Wait
         
CPU Utilization: ~30% (70% wasted)

With Scheduling (Multiple Processes):

Process 1: [███████░░░░░░░░░░░░░░░░███████░░░░░]
Process 2: [░░░░░░░███████████░░░░░░░░░░░░█████]
Process 3: [░░░░░░░░░░░░░░░░██████░░░░░░░░░░░░░]
Combined:  [███████████████████████████████████]
         
CPU Utilization: ~95% (5% scheduling overhead)
```

---

## Hardware Support for Scheduling

Modern CPUs include hardware features specifically designed to support operating system scheduling:

### 1. Timer Hardware

```
Programmable Interval Timer (PIT) / High Precision Event Timer (HPET):

Functions:
├── Generates periodic interrupts
├── Interrupt frequency programmable by OS
├── Triggers scheduler at regular intervals
└── Enables preemptive scheduling

Typical Configuration:
├── Timer frequency: 100-1000 Hz
├── Interrupt every: 1-10 ms
├── Called: "tick" or "jiffy"
└── Scheduler runs on each tick (or multiple ticks)
```

### 2. Interrupt Controller

```
Advanced Programmable Interrupt Controller (APIC):

Functions:
├── Receives interrupts from devices
├── Prioritizes interrupts
├── Routes interrupts to appropriate CPU cores
├── Supports inter-processor interrupts (IPIs)
└── Enables scheduler to be invoked on demand

Scheduling-Related Interrupts:
├── Timer interrupts (scheduling decisions)
├── I/O completion interrupts (wake up waiting processes)
├── Inter-processor interrupts (reschedule on another core)
└── System call interrupts (explicit yield)
```

### 3. Memory Management Hardware

```
Memory Management Unit (MMU) and TLB:

Functions:
├── Translates virtual to physical addresses
├── Enforces memory protection
├── TLB caches recent translations
└── Supports per-process address spaces

Scheduling Implications:
├── Context switch requires TLB flush or tagged entries
├── ASIDs (Address Space Identifiers) reduce TLB flushes
├── Page table base register must be updated
└── Cache and TLB effects impact scheduling decisions
```

### 4. Atomic Instructions

```
Hardware atomic operations:

Available Instructions:
├── Test-and-Set (TAS)
├── Compare-and-Swap (CAS)
├── Load-Linked/Store-Conditional (LL/SC)
├── Fetch-and-Add
└── Atomic increment/decrement

Scheduling-Related Uses:
├── Implementing scheduler locks
├── Manipulating ready queue safely
├── Updating process states atomically
└── Implementing synchronization primitives
```

### 5. CPU Features for Virtualization

```
Hardware Virtualization Support:

Features:
├── Intel VT-x / AMD-V
├── Nested Page Tables (EPT/NPT)
├── Virtual Process IDs (VPIDs)
├── I/O MMU Virtualization (VT-d)

Scheduling Implications:
├── Hypervisor scheduling of virtual CPUs
├── Guest OS scheduling within VMs
├── Nested scheduling challenges
└── Hardware support reduces virtualization overhead
```

---

## Multi-Core and Multi-Processor Systems

Modern systems have multiple CPU cores, adding complexity to scheduling:

### Multi-Core Architecture

```
                    Multi-Core System

    ┌─────────────────────────────────────────┐
    │              CPU Socket                  │
    │  ┌──────────┐  ┌──────────┐            │
    │  │  Core 0  │  │  Core 1  │  ...       │
    │  │ ┌──────┐ │  │ ┌──────┐ │            │
    │  │ │ L1/L2 │ │  │ │ L1/L2 │ │           │
    │  │ └──────┘ │  │ └──────┘ │            │
    │  └────┬─────┘  └────┬─────┘            │
    │       └──────┬──────┘                   │
    │        ┌─────┴─────┐                    │
    │        │ L3 Cache   │                   │
    │        └─────┬─────┘                    │
    └──────────────┼──────────────────────────┘
                   │
              Memory Bus
                   │
              Main Memory
```

### Multi-Core Scheduling Challenges

| Challenge         | Description                                    | Solution Approach                      |
|-------------------|------------------------------------------------|-----------------------------------------|
| Load Balancing     | Ensure all cores are utilized                  | Work stealing, periodic rebalancing    |
| Cache Affinity     | Keep processes on same core                    | Prefer local scheduling, migrate only when needed |
| NUMA Effects       | Memory access time varies by location          | NUMA-aware scheduling                   |
| Shared Cache       | Cores compete for L3 cache                     | Consider cache footprint in scheduling  |
| Power Efficiency    | Idle cores can be powered down                 | Consolidate work on fewer cores        |
| Synchronization     | Scheduler data structures shared                | Per-core run queues, lock-free algorithms |

### Per-Core Run Queues

Modern schedulers use per-core run queues for scalability:

```
                    Per-Core Scheduling Architecture

    Core 0          Core 1          Core 2          Core 3
    ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
    │Run Queue│     │Run Queue│     │Run Queue│     │Run Queue│
    │        │     │        │     │        │     │        │
    │ Process│     │ Process│     │ Process│     │ Process│
    │   A    │     │   C    │     │   E    │     │   G    │
    │   B    │     │   D    │     │   F    │     │   H    │
    └────────┘     └────────┘     └────────┘     └────────┘
         │              │              │              │
         └──────────────┼──────────────┘              │
                        │                             │
              ┌─────────┴─────────┐                   │
              │  Load Balancer    │───────────────────┘
              │  (periodic check) │
              └───────────────────┘
```

---

## CPU Performance Counters

Modern CPUs include hardware performance counters that schedulers can use:

### Available Metrics

```
Performance Monitoring Unit (PMU) provides:

├── Instruction count
├── Cycle count
├── Cache misses (L1, L2, L3)
├── Branch mispredictions
├── TLB misses
├── Memory accesses
├── Stalled cycles
└── Power consumption estimates
```

### Scheduler Use of Performance Counters

| Metric                 | Scheduling Use                                         |
|------------------------|-------------------------------------------------------|
| Cache misses           | Detect cache-thrashing processes, adjust priority      |
| Branch mispredictions  | Identify poorly optimized code                          |
| Stalled cycles         | Identify I/O-bound vs. CPU-bound                       |
| Instruction count      | Measure actual CPU usage                               |
| Power estimates        | Power-aware scheduling on mobile                       |

### Example: Using Counters for Process Classification

```
Reading performance counters during execution:

Process A:
├── Instructions per cycle: 0.9 (high IPC)
├── Cache miss rate: 2% (good cache usage)
├── Classification: Efficient CPU-bound process

Process B:
├── Instructions per cycle: 0.2 (low IPC)
├── Cache miss rate: 45% (poor cache usage)
├── Classification: Memory-bound or I/O-bound process

Scheduler can use this information to:
├── Prioritize I/O-bound processes
├── Group processes with similar characteristics
├── Detect and penalize cache-thrashing processes
└── Make informed load balancing decisions
```

---

## The Execution Timeline

Let's trace what happens from when a process is scheduled to when it yields the CPU:

### Process Execution Lifecycle

```
1. SCHEDULER SELECTS PROCESS
   ├── Scheduler runs in kernel mode
   ├── Selects next process from ready queue
   ├── Loads process's saved state
   └── Transitions to user mode

2. PROCESS EXECUTES IN USER MODE
   ├── Instructions execute on CPU
   ├── Process uses its address space
   ├── Cache fills with process data
   ├── TLB fills with process translations
   └── Execution continues until...

3. INTERRUPT OR SYSCALL OCCURS
   ├── Timer interrupt fires (time slice expired)
   ├── OR process makes system call (I/O request)
   ├── OR hardware interrupt arrives (I/O complete)
   ├── CPU transitions to kernel mode
   └── Control transfers to interrupt handler

4. KERNEL HANDLES EVENT
   ├── Interrupt handler runs
   ├── Determines what happened
   ├── Updates process state
   └── May invoke scheduler

5. SCHEDULER MAKES DECISION
   ├── Current process may be preempted
   ├── New process selected (or same continues)
   ├── Context switch performed if needed
   └── Control returns to selected process

6. PROCESS RESUMES (same or different)
   ├── Process state restored
   ├── CPU returns to user mode
   ├── Execution continues
   └── Cycle repeats
```

---

## Understanding CPU Time vs. Wall Time

A crucial distinction in scheduling:

### Definitions

```
Wall Clock Time:
├── Total elapsed real-world time
├── Includes time waiting for CPU
├── Includes time waiting for I/O
├── What users perceive
└── Measured with gettimeofday()

CPU Time:
├── Time actually executing on CPU
├── Only counts CPU burst execution
├── Does NOT include I/O wait
├── What scheduler allocates
└── Measured with times() or getrusage()
```

### Example

```
Process timeline:
0ms        50ms       200ms      250ms      400ms
|-----------|-----------|-----------|-----------|
  CPU (50ms)  I/O (150ms)  CPU (50ms)  I/O (150ms)

Wall Time: 400ms (total elapsed)
CPU Time: 100ms (50ms + 50ms of actual CPU usage)

CPU Utilization: 100/400 = 25%
I/O Wait Time: 300ms
```

### Scheduling Metrics Relationship

| Metric               | Uses                       | Measures                            |
|----------------------|----------------------------|-------------------------------------|
| Turnaround Time      | Wall time                  | Submission to completion            |
| Waiting Time         | Wall time                  | Time in ready queue                  |
| CPU Time             | CPU time                   | Actual processing                     |
| Response Time        | Wall time                  | Submission to first output         |
| Throughput           | Wall time                  | Completions per second             |

---

## Power Management and CPU Execution

Modern CPUs have power states that interact with scheduling:

### CPU Power States (C-States)

```
C-States (Idle States):

C0 - Active (executing instructions)
├── Full power consumption
├── Instructions executing
└── Various performance levels (P-states)

C1 - Halt
├── No instructions executing
├── Low power consumption
├── Quick wake-up (~1 μs)
└── Entered when CPU idle

C2 - Stop Clock
├── Clock stopped
├── Lower power than C1
├── Slower wake-up (~5 μs)
└── Deeper idle state

C3 - Sleep
├── Caches flushed
├── Much lower power
├── Slower wake-up (~50 μs)
└── Deep idle state

C6+ - Deep Power Down
├── Voltage reduced
├── Minimal power consumption
├── Slowest wake-up (~100+ μs)
└── Used for extended idle periods
```

### P-States (Performance States)

```
P-States (Performance Levels):

P0 - Maximum performance
├── Highest clock frequency
├── Highest voltage
├── Maximum power consumption
└── Best for CPU-intensive work

P1-Pn - Intermediate levels
├── Progressively lower frequency
├── Lower voltage
├── Better power efficiency
└── Trade-off: performance vs. power

Scheduling Implications:
├── Scheduler can request P-state changes
├── CPU-bound workloads → higher P-state
├── Idle or light workload → lower P-state
├── Mobile devices aggressively manage P-states
└── Turbo Boost dynamically exceeds P0 temporarily
```

### Power-Aware Scheduling

Modern schedulers consider power consumption:

```
Power-Aware Scheduling Strategies:

1. CPU Consolidation:
   ├── Pack processes onto fewer cores
   ├── Idle cores enter deep C-states
   ├── Saves power when load is light
   └── Cost: reduced parallelism

2. Frequency Scaling:
   ├── Adjust CPU frequency based on load
   ├── Light load → lower frequency
   ├── Heavy load → higher frequency
   └── Governed by "cpufreq" in Linux

3. Workload Classification:
   ├── Identify compute-intensive processes
   ├── Schedule them efficiently
   ├── Avoid unnecessary wake-ups
   └── Batch timer interrupts when possible
```

---

## Real-World CPU Examples

Let's examine actual CPUs to ground these concepts:

### Intel Core i7 (Desktop)

```
Intel Core i7-13700K:

Specifications:
├── 16 cores (8 P-cores + 8 E-cores)
├── 24 threads (hyperthreading on P-cores)
├── P-cores: 3.4-5.4 GHz
├── E-cores: 2.5-4.2 GHz
├── L1: 80 KB per P-core (32KB I + 48KB D)
├── L2: 2 MB per P-core, 4 MB per E-core cluster
├── L3: 30 MB shared
└── TDP: 125W (base), 253W (turbo)

Scheduling Considerations:
├── P-cores for latency-sensitive tasks
├── E-cores for background tasks
├── Scheduler must understand core types
├── Hybrid architecture requires intelligent scheduling
└── Windows 11 and modern Linux handle this
```

### AMD Ryzen (Desktop)

```
AMD Ryzen 9 7950X:

Specifications:
├── 16 cores (all identical)
├── 32 threads (SMT)
├── Base: 4.5 GHz, Boost: 5.7 GHz
├── L1: 64 KB per core
├── L2: 1 MB per core
├── L3: 64 MB (two 32MB CCX)
└── TDP: 170W

Scheduling Considerations:
├── Chiplet design (two CCDs)
├── Cross-CCD communication slower
├── Scheduler should prefer same-CCD scheduling
└── NUMA-like effects within single socket
```

### Apple Silicon (Mobile/Desktop)

```
Apple M2 Pro:

Specifications:
├── 10-12 cores (performance + efficiency)
├── Performance cores: ~3.5 GHz
├── Efficiency cores: ~2.4 GHz
├── Unified memory architecture
├── Neural engine, GPU on same die
└── Very power efficient

Scheduling Considerations:
├── Efficiency cores for background tasks
├── Performance cores for interactive tasks
├── Tight integration with OS scheduler
├── Quality of Service (QoS) classes
└── Aggressive power management
```

---

## Common CPU-Related Performance Issues

Understanding CPU execution helps diagnose performance problems:

### 1. Cache Thrashing

```
Symptoms: High CPU usage but slow performance

Cause:
├── Process working set exceeds cache size
├── Constantly evicting and reloading cache lines
├── High cache miss rate (>10%)

Diagnosis:
├── Use performance counters to check miss rate
├── perf stat -e cache-misses program

Solutions:
├── Reduce working set size
├── Improve data locality
├── Reconsider algorithm design
└── Scheduler may penalize thrashing processes
```

### 2. CPU Starvation

```
Symptoms: Process never seems to run

Cause:
├── Lower priority than other processes
├── Scheduler always picks higher priority
├── Process in multilevel queue never advances

Diagnosis:
├── Check process priority (nice value in Linux)
├── Monitor waiting time in ready queue
├── Check if starvation is occurring

Solutions:
├── Increase process priority
├── Use aging in scheduler
├── Adjust scheduling policy
└── Ensure fairness mechanisms are active
```

### 3. Excessive Context Switching

```
Symptoms: System feels slow despite low CPU usage

Cause:
├── Too many processes/threads
├── Very small time quantum
├── Frequent synchronization overhead
├── High interrupt rate

Diagnosis:
├── vmstat shows high cs (context switch) rate
├── > 50,000 switches/second is excessive
├── Check number of runnable processes

Solutions:
├── Reduce thread count
├── Increase time quantum
├── Batch operations to reduce interrupts
└── Review application design
```

---

## Summary

The CPU is a complex piece of hardware with characteristics that directly influence scheduling decisions. Understanding how the CPU executes instructions, manages memory, and interacts with the rest of the system is essential for understanding why schedulers work the way they do.

### Key Points

1. The CPU executes instructions in a fetch-decode-execute cycle, pipelined for efficiency.
2. The memory hierarchy (registers → cache → RAM → disk) has enormous performance implications for scheduling.
3. The CPU-I/O speed mismatch is the fundamental reason scheduling exists—the CPU must be kept busy while processes wait for I/O.
4. Hardware features like timers, interrupt controllers, and atomic instructions support the scheduler's work.
5. Multi-core systems introduce challenges like load balancing and cache affinity.
6. Performance counters provide information schedulers can use to make better decisions.
7. Power management is increasingly important, especially for mobile devices.
8. Understanding CPU architecture helps diagnose scheduling-related performance problems.

---

## Key Takeaways

1. The CPU is fast, I/O is slow—scheduling bridges this gap by keeping the CPU busy with other processes during I/O waits.
2. Context switching has real hardware costs—pipeline flushes, cache effects, and TLB misses all contribute to overhead.
3. Cache affinity matters—rescheduling a process on the same core can significantly improve performance.
4. Modern CPUs are complex—multiple cores, deep caches, power states, and performance counters all affect scheduling.
5. The scheduler relies on hardware support—timers, interrupts, and atomic instructions enable preemptive scheduling.
6. Performance is measurable—hardware counters provide detailed information about execution efficiency.
7. Power and performance trade-offs are increasingly important in scheduling decisions.
8. Understanding CPU execution is essential for understanding scheduling algorithms and their trade-offs.

---

## Further Reading

- [Scheduling-Basics.md](Scheduling-Basics.md): Core scheduling concepts and terminology
- [Context-Switching.md](Context-Switching.md): Detailed exploration of context switch mechanics
- [Preemptive-vs-Nonpreemptive.md](Preemptive-vs-Nonpreemptive.md): Scheduling interruption strategies
- [Real-World-Scheduling.md](Real-World-Scheduling.md): How modern OSes implement scheduling on real hardware
- [CPU-Scheduling/Introduction.md](CPU-Scheduling/Introduction.md): Overview of the scheduling module

