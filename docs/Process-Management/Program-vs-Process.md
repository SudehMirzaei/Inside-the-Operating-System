# Program vs. Process

## Introduction

One of the most fundamental distinctions in operating systems is the difference between a program and a process. While these terms are often used interchangeably in casual conversation, they refer to fundamentally different things. A program is a static entity—a file on disk containing instructions and data. A process is a dynamic entity—a running instance of a program with state, resources, and a lifecycle.

Understanding this distinction is essential because it clarifies:

- Why the same application can run multiple times simultaneously
- How operating systems manage resources
- What happens when you "run" a program
- Why processes can be in different states
- How isolation and protection work

This document explores the program/process distinction in depth, from definitions through practical implications.

---

## The Core Distinction

### Definitions

```
Program:
├── Static entity
├── Executable file on disk
├── Contains code and initial data
├── Passive (does nothing by itself)
├── Can be copied and shared
├── Has no state
└── Example: /usr/bin/firefox

Process:
├── Dynamic entity
├── Program in execution
├── Has state (registers, memory, etc.)
├── Active (executes instructions)
├── Unique per instance
├── Has lifecycle (birth to death)
└── Example: A running Firefox browser

The Relationship:
Program is to Process as:
├── Recipe is to Cooking
├── Sheet music is to Performance
├── Blueprint is to Building
└── Class is to Object (in OOP)
```

### The Essential Difference

```
Program (Static):              Process (Dynamic):
┌───────────────────┐          ┌───────────────────┐
│  File on disk     │          │  In memory        │
│  Bytes            │          │  Executing        │
│  No state         │          │  Has state        │
│  Can be copied    │          │  Unique instance  │
│  Passive          │          │  Active           │
│  Persists         │          │  Exists briefly   │
└───────────────────┘          └───────────────────┘

Key Insight:
├── A program is a set of instructions
├── A process is those instructions being followed
├── Same program can yield many processes
└── Processes come and go; programs persist
```

---

## Programs in Detail

### What is a Program?

A program is an executable file stored on disk:

```
Program Composition:

Executable File (e.g., a.out, firefox.exe):
┌─────────────────────────────────────┐
│  Header                             │
│  ├── Magic number                   │
│  ├── Entry point                    │
│  ├── Architecture info              │
│  └── Metadata                       │
├─────────────────────────────────────┤
│  Text Segment                       │
│  ├── Machine instructions           │
│  ├── Read-only                      │
│  └── Shared among processes         │
├─────────────────────────────────────┤
│  Data Segment                       │
│  ├── Initialized global variables   │
│  ├── Initialized static variables   │
│  └── Read-write                     │
├─────────────────────────────────────┤
│  BSS Segment (Uninitialized Data)   │
│  ├── Uninitialized globals          │
│  ├── Zero-initialized               │
│  └── Not stored in file (just size) │
├─────────────────────────────────────┤
│  Other Sections                     │
│  ├── Symbol table                   │
│  ├── Debug info                     │
│  ├── Relocation info                │
│  └── Additional metadata            │
└─────────────────────────────────────┘

Properties:
├── Exists on disk
├── Has a name
├── Has permissions
├── Occupies storage space
├── Can be copied
├── Can be executed
└── Does nothing by itself
```

### Program Formats

Different operating systems use different executable formats:

```
Common Executable Formats:

ELF (Executable and Linkable Format):
├── Linux, BSD, Solaris
├── Standard on Unix-like systems
├── Flexible and extensible
├── Used for executables, libraries, object files
└── Magic number: 0x7F 'E' 'L' 'F'

PE (Portable Executable):
├── Windows
├── Based on COFF format
├── .exe, .dll, .sys files
├── Magic number: 'M' 'Z'
└── Complex but well-documented

Mach-O:
├── macOS, iOS
├── Mach Object format
├── Executables, libraries, object files
├── Supports fat binaries (multiple architectures)
└── Magic numbers: various

Others:
├── a.out (old Unix)
├── COFF (legacy Windows/Unix)
├── Wasm (WebAssembly)
└── Various specialized formats

Common Characteristics:
├── Header with metadata
├── Code section(s)
├── Data section(s)
├── Entry point specification
├── Architecture information
└── Loading instructions
```

### Program Storage

Programs are organized on disk in a specific way:

```
Program on Disk:

┌─────────────────────────────────────┐
│  File System                        │
│                                     │
│  /usr/bin/                          │
│  ├── firefox        (program)      │
│  ├── gcc            (program)      │
│  ├── vim            (program)      │
│  └── ...                           │
│                                     │
│  Each program:                      │
│  ├── Is a file                      │
│  ├── Has permissions (rwxr-xr-x)   │
│  ├── Has owner                     │
│  ├── Has size                      │
│  ├── Has timestamps                │
│  └── Contains executable code      │
│                                     │
└─────────────────────────────────────┘

Multiple Programs Can Share:
├── Libraries (libc, libm)
├── Resources (icons, config files)
├── Documentation
└── Common code
```

---

## Processes in Detail

### What is a Process?

A process is a program in execution with state and resources:

```
Process Components:

┌─────────────────────────────────────────────┐
│                 PROCESS                      │
│                                              │
│  1. Program Code (from executable)          │
│     ├── Loaded into memory                  │
│     ├── Shared, read-only                   │
│     └── Same for all instances              │
│                                              │
│  2. Data (initialized, uninitialized)       │
│     ├── Loaded from executable              │
│     ├── Each process has its own copy       │
│     └── Can be modified                     │
│                                              │
│  3. Heap (dynamic memory)                   │
│     ├── Grows/shrinks at runtime            │
│     ├── malloc/free operations              │
│     └── Each process has its own            │
│                                              │
│  4. Stack (function calls)                  │
│     ├── Local variables                     │
│     ├── Return addresses                    │
│     ├── Function parameters                 │
│     └── Each process has its own            │
│                                              │
│  5. Program Counter (PC)                    │
│     ├── Current instruction address         │
│     ├── Unique to each process              │
│     └── Changes as process runs             │
│                                              │
│  6. CPU Registers                           │
│     ├── General purpose registers           │
│     ├── Stack pointer                       │
│     ├── Flags                               │
│     └── Unique to each process              │
│                                              │
│  7. Process ID (PID)                        │
│     ├── Unique identifier                   │
│     ├── Used by kernel and users            │
│     └── Different for each process          │
│                                              │
│  8. Resources                               │
│     ├── Open files                          │
│     ├── File descriptors                    │
│     ├── Memory mappings                     │
│     ├── Signals                             │
│     └── IPC objects                         │
│                                              │
└─────────────────────────────────────────────┘
```

### Process Attributes

Each process has attributes that distinguish it:

```
Process Attributes:

Identity:
├── PID (Process ID)
├── PPID (Parent Process ID)
├── Process Group ID
├── Session ID
├── User ID (UID)
├── Group ID (GID)
└── Effective/Real IDs

State:
├── Current state (running, ready, etc.)
├── Priority (nice value)
├── Scheduling policy
├── CPU affinity
└── Time slice remaining

Resources:
├── Address space
├── Open file descriptors
├── Signal handlers
├── Timers
├── Memory mappings
└── IPC resources

Accounting:
├── CPU time used (user/system)
├── Wall clock time
├── Memory usage
├── I/O counts
├── Page faults
└── Context switches

Security:
├── Credentials (UID, GID)
├── Capabilities
├── Security context (SELinux)
├── Seccomp filters
└── Resource limits (ulimits)
```

### Process Creation

When a program is executed, a process is created:

```
From Program to Process:

Before:
├── Program on disk
├── No process exists
├── No memory allocated
├── No CPU time assigned
└── Just a file

Creation Steps:
1. Kernel reads executable header
2. Kernel validates executable
3. Kernel allocates memory for process
4. Kernel loads program segments into memory
5. Kernel sets up address space
6. Kernel allocates other resources
7. Kernel initializes CPU state
8. Kernel assigns PID
9. Kernel adds to ready queue
10. Process begins executing

After:
├── Program still on disk (unchanged)
├── Process in memory
├── Memory allocated
├── CPU time assigned
├── PID assigned
└── Active and running
```

---

## The Same Program, Multiple Processes

### How It Works

The same program can have multiple running instances:

```
Multiple Firefox Windows:

/usr/bin/firefox (program on disk):
┌─────────────────────────────────────┐
│  firefox                            │
│  Code: ~100 MB                      │
│  Data: ~50 MB                       │
│  Total: ~150 MB on disk             │
└─────────────────────────────────────┘

Running three windows:

Process 1 (PID 1234):
├── Text segment: Shared (read-only)
├── Data segment: Own copy
├── Heap: Own
├── Stack: Own
├── Registers: Own state
└── Memory: ~200 MB

Process 2 (PID 1235):
├── Text segment: Shared with P1
├── Data segment: Own copy
├── Heap: Own
├── Stack: Own
├── Registers: Own state
└── Memory: ~200 MB

Process 3 (PID 1236):
├── Text segment: Shared with P1, P2
├── Data segment: Own copy
├── Heap: Own
├── Stack: Own
├── Registers: Own state
└── Memory: ~200 MB

Total physical memory:
├── Shared text: ~100 MB (loaded once)
├── Data segments: 3 × 50 MB = 150 MB
├── Heaps: Variable (dynamic)
├── Stacks: 3 × small
└── Total: ~250+ MB

Key point:
├── Code shared for efficiency
├── Data separate for isolation
├── Each process independent
└── Cannot interfere with others
```

### Visual Representation

```
Program vs. Processes:

Program on Disk:
    ┌─────────────────────┐
    │   foo (executable)   │
    │  ┌───────────────┐  │
    │  │  Code         │  │
    │  ├───────────────┤  │
    │  │  Data         │  │
    │  └───────────────┘  │
    └─────────────────────┘
              │
              │ Loading/Execution
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
┌────────┐┌────────┐┌────────┐
│Process ││Process ││Process │
│   1    ││   2    ││   3    │
│PID 100 ││PID 101 ││PID 102 │
│        ││        ││        │
│┌──────┐││┌──────┐││┌──────┐│
││Code  ││││Code  ││││Code  ││ ← shared (COW/read-only)
││(shr) ││││(shr) ││││(shr) ││
│├──────┤││├──────┤││├──────┤│
││Data 1││││Data 2││││Data 3││ ← own copy
│├──────┤││├──────┤││├──────┤│
││Heap 1││││Heap 2││││Heap 3││ ← own
│├──────┤││├──────┤││├──────┤│
││Stack1││││Stack2││││Stack3││ ← own
│└──────┘││└──────┘││└──────┘│
└────────┘└────────┘└────────┘
```

### Real-World Examples

```
Example 1: Text Editors

Command: vim file1.txt
├── Creates process 1 (vim editing file1)
├── Program: /usr/bin/vim (loaded once)
├── Data: contents of file1.txt
└── Independent process

Command: vim file2.txt (different terminal)
├── Creates process 2 (vim editing file2)
├── Program: Same /usr/bin/vim (shared code)
├── Data: contents of file2.txt
└── Independent process

Note:
├── Two windows, two processes
├── Different files being edited
├── Completely independent
├── Changes in one don't affect other
└── Same program, different processes

Example 2: Compilers

Command: gcc file1.c
Command: gcc file2.c

├── Two separate gcc processes
├── Same /usr/bin/gcc program
├── Different input files
├── Independent execution
├── Can run in parallel
└── Different output files

Example 3: Web Servers

Web server architecture:
├── Same program (nginx, apache)
├── Multiple worker processes
├── Each handles different requests
├── Shared code (efficient)
├── Independent state
└── Isolation for reliability
```

---

## Key Differences

### Comprehensive Comparison

| Aspect      | Program  | Process  |
|-------------|----------|----------|
| Nature      | Static   | Dynamic  |
| Location    | Disk     | Memory   |
| Lifetime    | Persistent | Temporary |
| State       | None     | Multiple states |
| Resources   | None     | Many     |
| PID         | No       | Yes      |
| Registers   | No       | Yes      |
| Program Counter | No   | Yes      |
| Stack       | No       | Yes      |
| Execution   | Not executing | Executing |
| Copies      | Can be shared | Unique |
| Modification | Rarely changes | Changes constantly |
| Creation    | Compiled/linked | fork()/exec() |
| Destruction | Deleted  | Terminated |

### The Transformation

```
Program → Process Transformation:

Before (Program):
├── Static bytes on disk
├── No state
├── No resources
├── No execution
└── Nothing happening

Transformation:
├── exec() system call
├── Kernel reads executable
├── Kernel allocates memory
├── Kernel loads code
├── Kernel sets up state
└── Kernel starts execution

After (Process):
├── Code in memory
├── State established
├── Resources allocated
├── CPU executing
└── Active entity
```

---

## Practical Implications

### For Users

```
What Users Should Understand:

1. Same App, Multiple Windows
   ├── Each window may be a process
   ├── Or multiple tabs in one process
   ├── Depends on application design
   └── Explains memory usage

2. Application Crashes
   ├── One process crashes
   ├── Others may survive
   ├── Depends on isolation
   └── Better with process isolation

3. Memory Usage
   ├── Each process uses memory
   ├── Shared code reduces total
   ├── Data is per-process
   └── Affects system performance

4. Task Manager/Activity Monitor
   ├── Shows processes, not programs
   ├── Each process has PID
   ├── Can kill individual processes
   └── Shows resource usage per process
```

### For Programmers

```
What Programmers Should Understand:

1. Process Creation
   ├── fork() creates new process
   ├── exec() loads new program
   ├── Combined for new processes
   └── Process spawning is expensive

2. State Management
   ├── Each process has own state
   ├── Global variables are per-process
   ├── Static variables are per-process
   └── No sharing without IPC

3. Resource Limits
   ├── Per-process limits exist
   ├── File descriptors limited
   ├── Memory limited
   └── CPU time limited

4. Debugging
   ├── Debug individual processes
   ├── Attach debugger to process
   ├── Track by PID
   └── Multi-process debugging complex

5. Performance
   ├── Process creation overhead
   ├── Context switch overhead
   ├── IPC overhead
   └── Consider threads for sharing
```

### For System Administrators

```
What Sysadmins Should Understand:

1. Process Monitoring
   ├── ps, top, htop commands
   ├── /proc file system
   ├── Process accounting
   └── System monitoring tools

2. Resource Management
   ├── Per-process limits (ulimit)
   ├── cgroups for groups of processes
   ├── nice/renice for priority
   └── CPU affinity

3. Troubleshooting
   ├── Identify runaway processes
   ├── Kill misbehaving processes
   ├── Trace system calls (strace)
   └── Analyze process behavior

4. Security
   ├── Process isolation
   ├── Privilege levels
   ├── Sandboxing
   └── Audit processes
```

---

## Special Cases

### Zombie Processes

```
Zombie Process:

Definition:
├── Process that has terminated
├── But parent hasn't read exit status
├── PCB retained in kernel
├── No code, data, or resources
└── Just a placeholder

Why They Exist:
├── Kernel needs to keep exit status
├── Parent must read it (wait())
├── Otherwise parent loses info
├── Trade-off: resource vs. info
└── Standard Unix behavior

Example:
$ ps aux | grep defunct
user  1234  0.0  0.0  0  0  ?  Z  10:30  0:00 [myprocess] <defunct>

Fixing Zombies:
├── Parent calls wait()
├── Parent terminates
├── Or reparented to init
├── Init reaps them automatically
└── Zombies disappear
```

### Orphan Processes

```
Orphan Process:

Definition:
├── Process whose parent has died
├── Still running
├── Reparented to init (PID 1)
├── Init becomes new parent
└── Continues execution

Example:
├── Parent forks child
├── Parent exits quickly
├── Child continues running
├── Child is now orphan
├── Reparented to init
└── Init will reap when done

Use Cases:
├── Daemon processes
├── Background tasks
├── Detached processes
└── Intentionally orphaned

Command:
$ nohup long_running_command &

Effects:
├── Process continues after shell exits
├── Reparented to init
├── stdout/stderr redirected
└── Runs in background
```

### Kernel Threads

```
Kernel Threads:

Definition:
├── Processes that run only in kernel
├── No user-space component
├── Created by kernel
├── Perform kernel tasks
└── Not associated with user program

Examples:
├── kthreadd (parent of kernel threads)
├── kworker (workqueue workers)
├── ksoftirqd (softirq handling)
├── migration (CPU migration)
├── watchdog (system monitor)
└── Many others

Characteristics:
├── No user address space
├── No file descriptors (usually)
├── Run in kernel mode only
├── Cannot be killed by user
└── Part of kernel's operation

Viewing:
$ ps aux | grep '\['
root  2  0.0  0.0  0  0  ?  S  10:30  0:00 [kthreadd]
root  3  0.0  0.0  0  0  ?  S  10:30  0:00 [ksoftirqd/0]
...
```

---

## Common Misconceptions

**Misconception 1:** "Programs and processes are the same"  
**Reality:** Programs are static files; processes are dynamic executions. The same program can have many processes. Programs don't have state; processes do.

**Misconception 2:** "Each program runs as one process"  
**Reality:** Many applications use multiple processes. Chrome uses one per tab. Servers use worker processes. Databases use multiple background processes.

**Misconception 3:** "Processes contain the program"  
**Reality:** Processes execute programs. The program is loaded from disk into the process's memory. The program file remains on disk, unchanged.

**Misconception 4:** "Killing a process deletes the program"  
**Reality:** Killing a process terminates that execution instance. The program file on disk is unaffected. You can run the program again to create a new process.

**Misconception 5:** "Processes are always isolated"  
**Reality:** Processes can share memory (with explicit APIs), communicate (IPC), and interact (signals). Isolation is the default, but sharing is possible.

**Misconception 6:** "A terminated process is gone immediately"  
**Reality:** Terminated processes become zombies until the parent reads their exit status. The PCB remains until then. This is why zombies appear in ps.

**Misconception 7:** "All processes have a parent"  
**Reality:** Every process has a parent except for init (PID 1), which is the ancestor of all processes. Orphaned processes get reparented to init.

---

## Summary

The distinction between a program and a process is fundamental to understanding operating systems. A program is a static, passive entity—an executable file on disk. A process is a dynamic, active entity—a program in execution with state, resources, and a lifecycle. The same program can spawn many processes, each independent and isolated.

### Key Points

1. Programs are static—files on disk with code and data.
2. Processes are dynamic—running instances with state and resources.
3. Same program, many processes—code is shared, data is per-process.
4. Processes have state—registers, memory, and other runtime information.
5. Programs persist; processes come and go—different lifetimes.
6. The transformation occurs through the exec() system call.
7. Special cases include zombies, orphans, and kernel threads.
8. Understanding this distinction is essential for OS concepts.

---

## Key Takeaways

1. Program ≠ Process—one is static, the other dynamic.
2. A process is a program in execution—with all that implies.
3. Sharing code, isolating data—efficient and safe.
4. Every process has a lifecycle—creation to termination.
5. Multiple processes from one program—common and useful.
6. Zombies and orphans—edge cases with important implications.
7. Kernel threads are processes too—but only in kernel space.
8. This distinction clarifies many OS behaviors and concepts.

---

## What's Next?

Continue to Process Lifecycle to learn about the complete journey of a process from creation through termination.

---

## Further Reading

### Books

- "Operating System Concepts" by Silberschatz, Galvin, and Gagne
- "Modern Operating Systems" by Andrew S. Tanenbaum
- "Operating Systems: Three Easy Pieces" by Remzi and Andrea Arpaci-Dusseau
- "The Linux Programming Interface" by Michael Kerrisk

### Online Resources

- OSTEP (ostep.org) — Free textbook
- man pages: execve(2), fork(2), process(5), proc(5)
- Linux Kernel Documentation (kernel.org)

