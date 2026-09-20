# Process Control Block

## Introduction

The Process Control Block (PCB) is the data structure the operating system uses to represent a process. It's the kernel's complete record of everything it needs to know about a process: its identity, its state, its resources, its execution context, and its accounting information.

When the kernel manages hundreds or thousands of processes, it needs an efficient, complete way to track each one. The PCB is that record. Every process has exactly one PCB. When a process is created, a PCB is created. When a process terminates, its PCB is destroyed. Between those events, the PCB is the kernel's authoritative source of information about the process.

Understanding the PCB is essential because it's the foundation of process management: scheduling decisions use PCB fields, context switches save and restore PCB contents, resource allocation is tracked in the PCB, and security checks consult PCB credentials.

This document explores the PCB in depth: what it contains, how it's organized, how it's used, and how real systems implement it.

---

## What is a Process Control Block?

### Definition

```
Process Control Block (PCB):

Definition:
├── Kernel data structure
├── Represents a single process
├── Contains all process metadata
├── Used by kernel for management
├── Also called:
│   ├── Task Control Block (TCB)
│   ├── Process Descriptor
│   ├── Task Struct (Linux)
│   └── EPROCESS (Windows)
└── One PCB per process

Purpose:
├── Track process state
├── Store execution context
├── Record resource usage
├── Enable scheduling decisions
├── Support context switching
├── Manage resource allocation
└── Provide process identity

Location:
├── In kernel memory
├── Not accessible to user
├── Protected from tampering
├── Persists during process lifetime
└── Freed on process termination
```

### The Analogy

Think of a PCB as a patient's medical chart in a hospital:

```
Hospital Analogy:

Medical Chart (PCB):
├── Patient ID (PID)
├── Current condition (state)
├── Treatment plan (program)
├── Medical history (accounting)
├── Current medications (resources)
├── Allergies (permissions)
├── Emergency contacts (parent)
└── Room assignment (memory)

Like a medical chart:
├── One per patient (process)
├── Updated continuously
├── Consulted by staff (kernel)
├── Complete record
├── Essential for care (management)
└── Archived after discharge (terminated)
```

---

## PCB Contents

### Categories of Information

The PCB contains several categories of information:

```
PCB Information Categories:

┌─────────────────────────────────────────────────────────────┐
│                    PROCESS CONTROL BLOCK                    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  1. Process Identification                          │   │
│  │     ├── PID (Process ID)                            │   │
│  │     ├── PPID (Parent PID)                           │   │
│  │     ├── UID (User ID)                               │   │
│  │     ├── GID (Group ID)                              │   │
│  │     ├── Session ID                                  │   │
│  │     └── Process group ID                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  2. Process State                                   │   │
│  │     ├── Current state (ready, running, etc.)        │   │
│  │     ├── Exit status (if terminated)                 │   │
│  │     └── Stop signal (if stopped)                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  3. CPU Context (Registers)                         │   │
│  │     ├── Program Counter (PC)                        │   │
│  │     ├── Stack Pointer (SP)                          │   │
│  │     ├── General purpose registers                   │   │
│  │     ├── Flags register                              │   │
│  │     ├── Floating-point registers                    │   │
│  │     ├── SIMD/vector registers                       │   │
│  │     └── Other CPU state                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  4. CPU Scheduling Information                      │   │
│  │     ├── Priority                                    │   │
│  │     ├── Nice value                                  │   │
│  │     ├── Scheduling policy                           │   │
│  │     ├── Time slice remaining                        │   │
│  │     ├── CPU affinity                                │   │
│  │     └── Recent CPU usage                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  5. Memory Management Information                   │   │
│  │     ├── Page table pointer                          │   │
│  │     ├── Base/limit registers                        │   │
│  │     ├── Segment table                               │   │
│  │     ├── Memory maps                                 │   │
│  │     ├── Virtual memory statistics                   │   │
│  │     └── Memory limits                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  6. Accounting Information                          │   │
│  │     ├── CPU time used (user)                        │   │
│  │     ├── CPU time used (system)                      │   │
│  │     ├── Wall clock time                             │   │
│  │     ├── Memory usage                                │   │
│  │     ├── I/O operations count                        │   │
│  │     ├── Page faults                                 │   │
│  │     ├── Context switches                            │   │
│  │     └── Start time                                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  7. I/O Status Information                          │   │
│  │     ├── Open file descriptors                       │   │
│  │     ├── File descriptor table pointer               │   │
│  │     ├── Current working directory                   │   │
│  │     ├── I/O devices allocated                       │   │
│  │     ├── Pending I/O requests                        │   │
│  │     └── I/O buffers                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  8. Inter-Process Communication                     │   │
│  │     ├── Signal handlers                             │   │
│  │     ├── Pending signals                             │   │
│  │     ├── Signal mask                                 │   │
│  │     ├── Message queues                              │   │
│  │     ├── Shared memory                               │   │
│  │     ├── Semaphores                                  │   │
│  │     └── Pipes                                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  9. Security Information                            │   │
│  │     ├── Credentials (UID, GID)                      │   │
│  │     ├── Effective UID/GID                           │   │
│  │     ├── Saved UID/GID                               │   │
│  │     ├── Capabilities                                │   │
│  │     ├── Security context (SELinux, AppArmor)        │   │
│  │     └── Resource limits                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Detailed Field Descriptions

```
Key PCB Fields in Detail:

1. Process Identification

PID (Process ID):
├── Unique identifier
├── Integer (typically 16 or 32 bits)
├── Assigned by kernel
├── Reusable after process exits
└── Used in system calls

PPID (Parent Process ID):
├── PID of parent process
├── Used for process hierarchy
├── Changes on reparenting
└── Init has PPID 0 (or itself)

UID/GID:
├── User ID (real)
├── Group ID (real)
├── Effective UID/GID
├── Saved UID/GID
├── Used for permission checks
└── Modified by process creation

Process Group ID:
├── For job control
├── Signals to groups
├── Terminal association
└── Managed by shell

Session ID:
├── Login session
├── Group of process groups
├── Terminal association
└── For session management

2. Process State

Current State:
├── Ready, Running, Waiting, etc.
├── Updated on transitions
├── Used by scheduler
└── Visible via ps

Exit Status:
├── Only meaningful when terminated
├── 0 = success, non-zero = error
├── Stored for parent
├── Read via wait()
└── 8-bit value (Unix)

Stop Signal:
├── Signal that stopped process
├── For T state
├── Used for debugging
└── SIGSTOP, SIGTSTP, etc.

3. CPU Context

Program Counter (PC):
├── Address of next instruction
├── Saved on context switch
├── Restored on resume
├── Critical for continuation
└── Architecture-specific

Stack Pointer (SP):
├── Top of current stack
├── User or kernel stack
├── Saved per context
└── Restored on resume

General Registers:
├── RAX, RBX, RCX, RDX, etc. (x86-64)
├── r0-r15 (ARM)
├── Hold computation state
├── Saved on context switch
└── Architecture-specific

Flags Register:
├── Condition codes
├── Status flags
├── Control flags
├── Preserved across switches
└── Example: RFLAGS (x86)

Floating-Point State:
├── FP registers
├── FP control/status
├── May be saved lazily
├── Large context
└── Optimization matters

4. CPU Scheduling Information

Priority:
├── Relative importance
├── Static and dynamic
├── Determines scheduling
├── Range varies by system
└── Example: 0-139 (Linux)

Nice Value:
├── User-adjustable priority
├── Range: -20 to +19 (Unix)
├── Lower = higher priority
├── Only root can lower
└── Affects scheduler decision

Scheduling Policy:
├── How process is scheduled
├── SCHED_NORMAL, SCHED_FIFO, SCHED_RR
├── Real-time vs. normal
├── Determines behavior
└── Can be changed at runtime

Time Slice:
├── Remaining quantum
├── Decremented during execution
├── Reset on scheduling
├── Expires → preemption
└── Affects responsiveness

CPU Affinity:
├── Which CPUs can run process
├── Bitmask of allowed CPUs
├── Improves cache usage
├── Can be set by admin
└── Linux: sched_setaffinity()

5. Memory Management Information

Page Table Pointer:
├── CR3 on x86-64
├── TTBR0/TTBR1 on ARM
├── Physical address of page table
├── Changed on context switch
└── Foundation of isolation

Memory Maps:
├── Virtual memory regions
├── Code, data, heap, stack
├── Shared libraries
├── Memory-mapped files
└── Viewable via /proc/PID/maps

Memory Limits:
├── Maximum memory usage
├── Address space limit
├── Data segment limit
├── Stack limit
└── Enforced by kernel

6. Accounting Information

CPU Time:
├── User time (executing user code)
├── System time (executing kernel code)
├── Total CPU time
├── Used for billing, monitoring
└── Visible via time, top

Wall Clock Time:
├── Total elapsed time
├── From creation to now
├── Includes waiting
├── Different from CPU time
└── Visible via ps

Resource Usage:
├── Page faults (minor, major)
├── Context switches (voluntary, involuntary)
├── I/O operations
├── Signals received
├── Memory usage (RSS, VSZ)
└── Used for profiling

7. I/O Status Information

File Descriptor Table:
├── Array of file descriptors
├── Points to open file objects
├── Inherited from parent
├── Limited (RLIMIT_NOFILE)
└── Example: 0=stdin, 1=stdout, 2=stderr

Open Files:
├── Actually in separate structure
├── PCB points to file table
├── Multiple processes can share
├── Reference counted
└── Closed on process exit

Current Working Directory:
├── Process's notion of "current dir"
├── Used for relative paths
├── Changed via chdir()
├── Inherited from parent
└── Can be viewed via /proc/PID/cwd

8. Inter-Process Communication

Signal Handlers:
├── Per-signal handler functions
├── SIG_DFL, SIG_IGN, or custom
├── Inherited from parent
├── Can be changed
└── Reset on exec()

Pending Signals:
├── Signals received but not yet handled
├── Bitmask of pending signals
├── Delivered when unblocked
└── Kernel manages delivery

Signal Mask:
├── Signals currently blocked
├── Bitmask
├── Can be changed via sigprocmask()
├── Prevents handler invocation
└── Unblock triggers delivery

9. Security Information

Credentials:
├── Real UID/GID
├── Effective UID/GID
├── Saved UID/GID
├── File system UID/GID
├── Used for access checks
└── Modified by setuid programs

Capabilities:
├── Fine-grained privileges
├── Example: CAP_NET_ADMIN
├── Subset of root privileges
├── Inheritable, permitted, effective
└── Linux-specific

Security Context:
├── SELinux labels
├── AppArmor profiles
├── SMACK labels
├── Used for MAC
└── Determined at exec

Resource Limits:
├── CPU time limit
├── File size limit
├── Memory limit
├── Process count limit
├── File descriptor limit
└── Many others (ulimit)
```

---

## PCB Organization

### Where PCBs Live

```
PCB Storage in Kernel:

┌─────────────────────────────────────────────┐
│              KERNEL MEMORY                   │
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │   Process Table                     │    │
│  │                                     │    │
│  │  Array or List of PCBs              │    │
│  │                                     │    │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐  │    │
│  │  │ PCB │ │ PCB │ │ PCB │ │ PCB │  │    │
│  │  │ P1  │ │ P2  │ │ P3  │ │ P4  │  │    │
│  │  └─────┘ └─────┘ └─────┘ └─────┘  │    │
│  │                                     │    │
│  │  Organized by:                      │    │
│  │  ├── PID (hash or array)            │    │
│  │  ├── State (queues)                 │    │
│  │  ├── Parent-child (tree)            │    │
│  │  └── Other criteria                 │    │
│  │                                     │    │
│  └─────────────────────────────────────┘    │
│                                              │
└─────────────────────────────────────────────┘

Linux Approach:
├── task_struct for each process
├── Dynamically allocated
├── Not in contiguous array
├── Multiple data structures:
│   ├── task_list (all processes)
│   ├── children/sibling (parent-child)
│   ├── run_list (ready queue)
│   ├── pid_hash (fast PID lookup)
│   └── Various other lists
└── Flexible but complex

Traditional Unix:
├── Fixed array of PCBs
├── proc[] array
├── Limited number of processes
├── Simple indexing
└── Wastes memory (fixed size)

Modern Systems:
├── Dynamic allocation
├── Various data structures
├── Support many processes
├── Efficient operations
└── Linux, Windows, macOS
```

### Linux task_struct

The Linux PCB is called task_struct:

```
Linux task_struct (Simplified):

struct task_struct {
    // 1. Process state
    volatile long state;            // TASK_RUNNING, etc.
    void *stack;                    // Kernel stack
    
    // 2. Scheduling information
    int prio, static_prio, normal_prio;  // Priorities
    unsigned int rt_priority;       // RT priority
    const struct sched_class *sched_class;  // Scheduler
    struct sched_entity se;         // CFS entity
    struct sched_rt_entity rt;      // RT entity
    unsigned int policy;            // Scheduling policy
    int nr_cpus_allowed;            // CPU affinity
    
    // 3. Process identification
    pid_t pid;                      // Process ID
    pid_t tgid;                     // Thread group ID
    struct task_struct *parent;     // Parent process
    struct list_head children;      // Children list
    struct list_head sibling;       // Sibling list
    
    // 4. Memory management
    struct mm_struct *mm;           // User memory
    struct mm_struct *active_mm;    // Active memory
    struct files_struct *files;     // Open files
    struct fs_struct *fs;           // File system info
    
    // 5. Signal handling
    struct signal_struct *signal;   // Shared signals
    struct sighand_struct *sighand; // Signal handlers
    sigset_t blocked;               // Blocked signals
    struct sigpending pending;      // Pending signals
    
    // 6. Process credentials
    const struct cred *cred;        // Credentials
    const struct cred *real_cred;   // Real credentials
    struct user_struct *user;       // User account
    
    // 7. Accounting
    u64 utime, stime;               // User/system time
    u64 start_time;                 // Start time
    unsigned long nvcsw;            // Voluntary switches
    unsigned long nivcsw;           // Involuntary switches
    
    // 8. Relationships
    struct task_struct *group_leader; // Thread group leader
    struct list_head thread_group;  // Thread list
    
    // 9. Other fields
    char comm[TASK_COMM_LEN];       // Command name
    struct thread_struct thread;    // CPU-specific state
    // ... hundreds more fields
};

Size:
├── Linux: ~1-4 KB base
├── Plus dynamic fields
├── Per thread (not process)
├── Significant kernel memory
└── Optimized for performance
```

### Data Structures for PCBs

```
PCB Organization in Linux:

1. task_list (Doubly-linked list):
   ├── All processes in system
   ├── Circular list
   ├── Used for iteration
   └── Example: ps, top

2. PID Hash Table:
   ├── Fast PID → task_struct lookup
   ├── Hash function on PID
   ├── Handles collisions
   └── O(1) average case

3. Parent-Child Lists:
   ├── children: list of child processes
   ├── sibling: link to siblings
   ├── Enables process tree
   └── Used by wait(), getppid()

4. Run Queue (per CPU):
   ├── Ready processes
   ├── Red-black tree (CFS)
   ├── Priority-based ordering
   └── Scheduler's data structure

5. Wait Queues:
   ├── For each event type
   ├── Sleeping processes
   ├── Efficient wake-up
   └── Event-driven

6. Thread Groups:
   ├── Related threads
   ├── Shared resources
   ├── group_leader points to main
   └── thread_group list

7. Namespace Lists:
   ├── Per namespace
   ├── Container support
   ├── PID namespace
   └── Isolation mechanism

Access Patterns:
├── PID lookup: hash table
├── Iteration: task_list
├── Scheduling: run queue
├── Wake-up: wait queues
├── Hierarchy: parent/children
└── Optimized for each use
```

---

## PCB Usage

### During Process Creation

```
PCB Creation Sequence:

1. Allocate PCB
   ├── Memory from kernel allocator
   ├── Initialize all fields
   ├── Set defaults
   └── Clear sensitive data

2. Assign PID
   ├── Allocate from PID space
   ├── Check for collision
   ├── Reserve PID
   └── Add to PID hash

3. Initialize Identity
   ├── Set PID, PPID
   ├── Set UID, GID
   ├── Set process group
   ├── Set session
   └── Inherit from parent

4. Set State
   ├── Initially TASK_NEW
   ├── Then TASK_RUNNING (ready)
   ├── Or TASK_UNINTERRUPTIBLE
   └── Before scheduling

5. Set Up Scheduling Info
   ├── Inherit priority
   ├── Set nice value
   ├── Choose scheduler
   ├── Set affinity
   └── Initialize accounting

6. Allocate Resources
   ├── Address space (mm_struct)
   ├── File descriptor table
   ├── Signal handlers
   ├── Credentials
   └── Various kernel objects

7. Add to Data Structures
   ├── Add to task_list
   ├── Add to parent's children
   ├── Add to PID hash
   ├── Add to thread group
   └── Add to run queue (if ready)

8. Return to Caller
   ├── fork() returns in parent
   ├── Returns 0 in child
   ├── Both continue
   └── Child may exec()
```

### During Scheduling

```
PCB in Scheduling Decisions:

Scheduler Uses PCB For:

1. Selecting Next Process
   ├── state == TASK_RUNNING?
   ├── Priority (prio, normal_prio)
   ├── Nice value
   ├── Recent CPU usage
   ├── Affinity (cpus_allowed)
   └── Scheduling class

2. Making Decisions
   ├── Compare priorities
   ├── Check affinity
   ├── Consider cache locality
   ├── Balance load
   └── Fairness

3. Updating Statistics
   ├── Update utime, stime
   ├── Increment counters
   ├── Update recent usage
   ├── Adjust dynamic priority
   └── Track for fairness

4. Setting State
   ├── TASK_RUNNING → running
   ├── Or back to run queue
   ├── Update state field
   └── Maintain consistency

5. Managing Queues
   ├── Remove from run queue
   ├── Add to run queue
   ├── Update queue pointers
   └── Maintain structures
```

### During Context Switching

```
PCB in Context Switching:

Context Switch Steps:

1. Save Current Process (P1) State
   ├── Save CPU registers to PCB
   ├── Save program counter
   ├── Save stack pointer
   ├── Save floating-point state
   ├── Update PCB state field
   └── Update accounting

2. Select Next Process (P2)
   ├── Scheduler uses PCBs
   ├── Chooses P2's PCB
   ├── Verify P2 is ready
   └── Prepare to switch

3. Switch Memory Context
   ├── Load P2's page table
   ├── Update CR3 (x86)
   ├── Switch address spaces
   ├── Flush TLB (if needed)
   └── Update MMU state

4. Restore P2's State
   ├── Restore CPU registers
   ├── Restore program counter
   ├── Restore stack pointer
   ├── Restore floating-point
   ├── Set state to running
   └── Update PCB

5. Resume P2 Execution
   ├── Jump to saved PC
   ├── P2 continues
   ├── From where it left off
   └── Seamless continuation

PCB Fields Used:
├── thread.regs (CPU state)
├── thread.fpu (FP state)
├── mm (memory context)
├── state (current state)
├── prio (priority)
├── utime, stime (accounting)
├── nvcsw, nivcsw (switch counts)
└── Many more
```

### During Termination

```
PCB During Termination:

Process Exit Sequence:

1. Process Calls exit()
   ├── Or return from main
   ├── Or killed by signal
   ├── Kernel gains control
   └── Begins cleanup

2. Release Resources
   ├── Close file descriptors
   ├── Free memory
   ├── Release locks
   ├── Remove from queues
   └── Clean up IPC

3. Update PCB
   ├── Set state to EXIT_ZOMBIE
   ├── Store exit code
   ├── Store exit signal (if any)
   ├── Save accounting info
   └── Mark as terminated

4. Notify Parent
   ├── Send SIGCHLD
   ├── Parent can wait()
   ├── Parent may be sleeping
   └── Wake parent if needed

5. Reparent Children
   ├── Children become orphans
   ├── Reparented to init
   ├── Or subreaper
   └── Kernel updates PCBs

6. Wait for Reaping
   ├── PCB retained (zombie)
   ├── Minimal resources
   ├── Waiting for parent
   ├── wait() called by parent
   └── Returns exit status

7. Final Cleanup
   ├── Parent called wait()
   ├── Exit status returned
   ├── PCB freed
   ├── Removed from all lists
   └── Process completely gone

PCB State Changes:
├── TASK_RUNNING → EXIT_ZOMBIE
├── Wait for parent
├── EXIT_ZOMBIE → EXIT_DEAD
├── Brief transition
└── PCB freed
```

---

## Windows EPROCESS

Windows uses EPROCESS as its PCB:

```
Windows EPROCESS (Simplified):

struct _EPROCESS {
    // Object header (Windows executive object)
    KPROCESS Pcb;                   // Kernel process block
    EX_PUSH_LOCK ProcessLock;       // Process lock
    
    // Identification
    HANDLE UniqueProcessId;         // PID
    LIST_ENTRY ActiveProcessLinks;  // All processes
    
    // State
    BOOLEAN ProcessInSession;       // Session status
    UCHAR ExitStatus;               // Exit status
    BOOLEAN HasExited;              // Terminated flag
    
    // Memory management
    PVOID SectionObject;            // Memory sections
    PVOID SectionBaseAddress;       // Base address
    ULONG_PTR VirtualSize;          // Virtual size
    PVOID Vm;                       // VM structures
    
    // Scheduling
    KPRIORITY BasePriority;         // Base priority
    UCHAR PriorityClass;            // Priority class
    
    // Tokens and security
    PACCESS_TOKEN Token;            // Security token
    PSID UniqueProcessId;           // SID
    
    // I/O
    PVOID ObjectTable;              // Object handles
    ULONG HandleCount;              // Handle count
    
    // Threads
    LIST_ENTRY ThreadListHead;      // Thread list
    ULONG ActiveThreads;            // Thread count
    
    // Accounting
    LARGE_INTEGER CreateTime;       // Creation time
    LARGE_INTEGER ExitTime;         // Exit time
    ULONG_PTR KernelTime;           // Kernel time
    ULONG_PTR UserTime;             // User time
    
    // Relationships
    PEPROCESS InheritedFromUniqueProcessId;  // Parent PID
    // ... many more fields
};

Windows Concepts:
├── Process is a container
├── Threads are execution units
├── Handles reference objects
├── Token represents security
├── Process and thread both have structures
├── EPROCESS + ETHREAD
```

---

## PCB Operations

### Common PCB Operations

```
PCB Operations:

Creation:
├── allocate_pcb() - Allocate memory
├── init_pcb() - Initialize fields
├── assign_pid() - Assign PID
├── link_pcb() - Add to structures
└── free_pcb() - On error

Lookup:
├── find_by_pid(pid) - Get PCB by PID
├── find_by_pid_hash(pid) - Hash lookup
├── iterate_all() - Iterate all
├── iterate_children(parent) - Children
└── current() - Current process

State Changes:
├── set_state(process, new_state) - Change state
├── wake_up(process) - Ready process
├── sleep_on(process, queue) - Wait on event
├── stop(process) - Stop process
└── continue(process) - Resume process

Scheduling:
├── enqueue_ready(process) - Add to run queue
├── dequeue_ready() - Select next
├── set_priority(process, prio) - Set priority
├── set_affinity(process, mask) - CPU affinity
└── yield() - Voluntary yield

Resource Management:
├── add_file(process, fd) - Add file descriptor
├── close_file(process, fd) - Close file
├── add_signal(process, sig) - Pending signal
├── set_handler(process, sig, handler)
└── set_cred(process, cred) - Set credentials

Accounting:
├── update_time(process, user, sys)
├── add_page_fault(process)
├── count_switch(process, voluntary)
└── get_stats(process) - Get statistics

Termination:
├── exit(process, status) - Terminate
├── wait(process, child) - Wait for child
├── reap(process) - Free zombie PCB
└── reparent(process, new_parent)
```

### PCB Locking

PCBs are accessed concurrently and need locking:

```
PCB Locking:

Why Lock:
├── Multiple CPUs access PCBs
├── Concurrent kernel threads
├── Interrupt handlers
├── Scheduler modifications
├── Race conditions possible
└── Must ensure consistency

Linux Locking:
├── Multiple locks per task_struct
├── Fine-grained locking
├── Task lock for state changes
├── Signal lock for signal ops
├── PID lock for hash
├── Run queue lock for scheduling
└── Various spinlocks and mutexes

Example: Task Lock
├── spin_lock(&p->pi_lock)  - Priority lock
├── spin_lock(&p->alloc_lock) - Allocation lock
├── spin_lock(&rq->lock) - Run queue lock
└── Different locks for different fields

Locking Strategy:
├── Fine-grained (per-field)
├── Minimize contention
├── Careful ordering
├── Deadlock avoidance
└── Performance critical

Race Conditions Example:
CPU 1: Modifying state from READY to RUNNING
CPU 2: Waking up same process (setting READY)

Without Lock:
├── Race condition
├── Inconsistent state
├── Potential bug
└── Undefined behavior

With Lock:
├── Serialize access
├── Consistent state
├── Correct behavior
└── Synchronization ensured
```

---

## Accessing PCB Information

### From User Space

```
User Space Access to PCB Information:

Direct Access:
├── Not possible (kernel protected)
├── PCBs in kernel memory
├── User cannot read/write
└── Security boundary

Indirect Access:
├── System calls
│   ├── getpid() → PID
│   ├── getppid() → PPID
│   ├── getuid() → UID
│   ├── getgid() → GID
│   ├── times() → CPU times
│   └── getrlimit() → resource limits
│
├── /proc file system (Linux)
│   ├── /proc/PID/status    → State
│   ├── /proc/PID/stat      → Statistics
│   ├── /proc/PID/cmdline   → Command
│   ├── /proc/PID/environ   → Environment
│   ├── /proc/PID/fd/       → File descriptors
│   ├── /proc/PID/maps      → Memory maps
│   ├── /proc/PID/limits    → Resource limits
│   └── Many more files
│
├── ps, top commands
│   ├── Read /proc files
│   ├── Show state, PID, PPID
│   ├── Show resource usage
│   └── Format for display
│
└── System libraries
    ├── getpwuid() - User info
    ├── getgrgid() - Group info
    ├── sysconf() - System limits
    └── Various others

Example: /proc/PID/status
$ cat /proc/self/status
Name:   bash
State:  S (sleeping)
Tgid:   1234
Pid:    1234
PPid:   1000
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000
...
```

### Viewing Process Information

```
Common Commands:

ps aux:
├── Shows all processes
├── State (STAT column)
├── CPU/memory usage
├── Command
└── USER, PID, PPID

ps -ef:
├── Different format
├── Shows PPID explicitly
├── Full command line
└── UID visible

top / htop:
├── Dynamic monitoring
├── Process state
├── CPU/memory usage
├── Real-time updates
└── Sortable, filterable

/proc file system:
├── Detailed information
├── Per-process files
├── Kernel data exposed
├── Read-only usually
└── Some writable (sysctl)

pstree:
├── Process hierarchy
├── Based on PPID
├── Tree visualization
└── Shows relationships

pmap:
├── Memory map of process
├── Based on /proc/PID/maps
├── Virtual memory regions
└── Shows shared libraries
```

---

## Common Misconceptions

**Misconception 1:** "The PCB is stored in user memory"  
**Reality:** The PCB is stored in kernel memory, protected from user access. User programs cannot directly read or modify PCBs. They can only access selected information through system calls.

**Misconception 2:** "The PCB stores the program code"  
**Reality:** The PCB stores metadata about the process, not the program code itself. The program code is in the process's address space (user memory). The PCB points to that memory, but doesn't contain it.

**Misconception 3:** "Each process has a small PCB"  
**Reality:** PCBs can be quite large. Linux's task_struct is typically 1-4 KB, plus additional structures (mm_struct, files_struct, etc.). For thousands of processes, PCBs consume significant kernel memory.

**Misconception 4:** "PCBs are in an array indexed by PID"  
**Reality:** Modern systems don't use fixed arrays. Linux uses dynamic allocation with a PID hash table for lookup. This supports more processes and avoids fixed limits.

**Misconception 5:** "The PCB contains CPU registers"  
**Reality:** The PCB contains saved copies of CPU registers—they're saved when the process is switched out and restored when it's switched back in. The CPU registers themselves are in the CPU, not the PCB.

**Misconception 6:** "You can read any process's PCB via /proc"  
**Reality:** /proc restricts access based on permissions. You can read detailed information about your own processes. For other users' processes, much information is hidden unless you're root.

**Misconception 7:** "Terminated processes have no PCB"  
**Reality:** Terminated processes (zombies) retain their PCB until the parent reads the exit status via wait(). Only then is the PCB freed. This is why zombies consume resources.

---

## Summary

The Process Control Block is the kernel's complete record of a process. It contains process identification, state, CPU context, scheduling information, memory management data, accounting statistics, I/O status, IPC state, and security credentials. The PCB is essential for every aspect of process management: creation, scheduling, context switching, resource allocation, and termination.

### Key Points

1. One PCB per process, stored in kernel memory.
2. Contains all metadata: identity, state, context, resources, accounting.
3. Used by scheduler for decisions and by context switch for save/restore.
4. Linux implementation: task_struct.
5. Windows implementation: EPROCESS.
6. Multiple data structures organize PCBs (lists, hashes, trees).
7. Locking required for concurrent access in SMP systems.
8. User access is limited to system calls and /proc.

---

## Key Takeaways

1. PCB is the process's identity in the kernel.
2. Complete record: Everything the kernel needs to know.
3. Central to scheduling and context switching.
4. Rich data structures for efficient operations.
5. Careful locking ensures consistency.
6. Real implementations are complex (task_struct, EPROCESS).
7. User visibility through system calls and /proc.
8. Understanding the PCB clarifies process management.

---

## What's Next?

Continue to Process Creation to learn how new processes are created, including the fork() and exec() mechanisms.

---

## Further Reading

### Books

- "Operating System Concepts" by Silberschatz, Galvin, and Gagne
- "Modern Operating Systems" by Andrew S. Tanenbaum
- "Operating Systems: Three Easy Pieces" by Remzi and Andrea Arpaci-Dusseau
- "Linux Kernel Development" by Robert Love
- "Understanding the Linux Kernel" by Bovet and Cesati
- "Windows Internals" by Russinovich, Solomon, and Ionescu

### Online Resources

- OSTEP (ostep.org) — Free textbook
- Linux Kernel Documentation (kernel.org)
- man pages: proc(5), fork(2), exec(3)
- "The Linux Programming Interface" by Michael Kerrisk
- Linux source code: include/linux/sched.h

