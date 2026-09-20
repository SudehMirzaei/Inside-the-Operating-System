# Process States

## Introduction

A process is not always running. In fact, at any given moment on a typical system, most processes are not executing—they're waiting for something: CPU time, I/O completion, a signal, a lock, or a child process. The concept of process states captures this reality: the different conditions a process can be in during its lifetime.

Understanding process states is fundamental to understanding how operating systems manage multiple processes. The state of a process determines:

- Whether it can be scheduled for execution
- What events can change its condition
- Where it's stored in kernel data structures
- How the scheduler treats it

This document explores process states in depth: the standard five-state model, how states transition, how they're implemented in real systems (particularly Linux), and why different states exist.

---

## The Standard Five-State Model

### The Five States

Most operating systems textbooks describe five fundamental process states:

```
The Five-State Model:

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                     ┌─────────────┐                         │
│                     │     NEW     │                         │
│                     │  (Created)  │                         │
│                     └──────┬──────┘                         │
│                            │ admit                          │
│                            ▼                                │
│                     ┌─────────────┐                         │
│              ┌─────▶│    READY    │◀─────┐                  │
│              │      │ (Waiting)   │      │                  │
│              │      └──────┬──────┘      │                  │
│              │             │             │                  │
│              │          dispatch     I/O complete           │
│              │             │             │                  │
│              │             ▼             │                  │
│              │      ┌─────────────┐      │                  │
│              │      │   RUNNING   │      │                  │
│              │      │ (Executing) │      │                  │
│              │      └──────┬──────┘      │                  │
│              │             │             │                  │
│              │    ┌────────┴────────┐    │                  │
│         timeout   │                 │ wait                  │
│              │    ▼                 ▼    │                  │
│              │  ┌─────┐      ┌──────────┐│                  │
│              │  │Ready│      │ WAITING  ││                  │
│              └──│again│      │(Blocked) ││                  │
│                 └─────┘      └──────────┘│                  │
│                                │         │                  │
│                                └─────────┘                  │
│                                    ▲                        │
│                                    │                        │
│                              (returns to Ready)             │
│                                                             │
│                     ┌─────────────┐                         │
│                     │ TERMINATED  │                         │
│                     │   (Exit)    │                         │
│                     └─────────────┘                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### State Definitions

Let's examine each state in detail:

```
1. NEW State:

Definition:
├── Process is being created
├── Kernel is initializing PCB
├── Resources being allocated
├── Not yet in ready queue
└── Temporary state

What Happens:
├── PID assignment
├── PCB allocation
├── Memory allocation
├── Program loading
├── Resource allocation
└── Initialization

Duration:
├── Very brief
├── Microseconds typically
├── Longer for large programs
└── Transient state

2. READY State:

Definition:
├── Process is ready to execute
├── All resources available
├── Waiting only for CPU
├── In ready queue
└── Can be scheduled

What Happens:
├── Process waits its turn
├── Scheduler may select it
├── Priority affects wait time
├── Fair scheduling ensures progress
└── Eventually dispatched

Duration:
├── Variable
├── Short on idle systems
├── Long on busy systems
├── Depends on scheduler
└── Multiple processes usually ready

3. RUNNING State:

Definition:
├── Process is executing on CPU
├── Instructions being processed
├── Has CPU control
├── Making progress
└── One per CPU (typically)

What Happens:
├── Instructions fetched/executed
├── State modified
├── System calls may be made
├── I/O may be requested
└── Execution continues

Duration:
├── Until event occurs
├── Time slice expiry
├── I/O request
├── Higher priority arrival
└── Process completion

4. WAITING (BLOCKED) State:

Definition:
├── Process cannot proceed
├── Waiting for event
├── Not using CPU
├── Not in ready queue
└── In wait queue

What Happens:
├── Process requested I/O
├── Waiting for signal
├── Waiting for child
├── Waiting for resource
└── Suspended until event

Duration:
├── Until event occurs
├── I/O completion time
├── Could be long
├── User-driven often
└── Variable

5. TERMINATED State:

Definition:
├── Process finished execution
├── Resources being released
├── May exist as zombie
├── PCB retained temporarily
└── Final state

What Happens:
├── Exit called
├── Resources released
├── Parent notified
├── Exit status stored
├── PCB cleanup

Duration:
├── Until parent reaps
├── Brief if parent waits
├── Long if parent doesn't
├── Zombie state
└── Then fully removed
```

---

## State Transitions

### All Possible Transitions

```
State Transition Matrix:

From \ To    NEW    READY    RUNNING    WAITING    TERMINATED
─────────────────────────────────────────────────────────────
NEW          -       ✓         -          -           ✓*
READY        -       -         ✓          -           ✓*
RUNNING      -       ✓         -          ✓           ✓
WAITING      -       ✓         -          -           ✓*
TERMINATED   -       -         -          -           -

* Rare or unusual transitions

Common Transitions:
├── NEW → READY (admitted)
├── READY → RUNNING (dispatch)
├── RUNNING → READY (preempted)
├── RUNNING → WAITING (blocked)
├── WAITING → READY (event)
├── RUNNING → TERMINATED (exit)

Unusual Transitions:
├── NEW → TERMINATED (creation failed)
├── READY → TERMINATED (killed before running)
├── WAITING → TERMINATED (killed while waiting)
```

### Transition Details

```
Detailed Transition Analysis:

1. NEW → READY (Admit)
   Trigger: Creation complete
   Action: Added to ready queue
   Next: Wait for CPU
   Notes: Standard first transition

2. READY → RUNNING (Dispatch)
   Trigger: Scheduler selects process
   Action: Context switch to process
   Next: Execute instructions
   Notes: Only one process per CPU

3. RUNNING → READY (Preempt/Timeout)
   Trigger: Time slice expires OR higher priority ready
   Action: Process moved to ready queue
   Next: Wait for next dispatch
   Notes: Preemption for fairness

4. RUNNING → WAITING (Block)
   Trigger: I/O request OR wait for event
   Action: Process moved to wait queue
   Next: Another process runs
   Notes: Voluntary yield of CPU

5. WAITING → READY (Event Occurs)
   Trigger: I/O completes OR event happens
   Action: Process moved to ready queue
   Next: Wait for CPU again
   Notes: Process can now continue

6. RUNNING → TERMINATED (Exit)
   Trigger: exit() OR fatal error
   Action: Resources released
   Next: Cleanup, possibly zombie
   Notes: Final transition

7. READY → TERMINATED (Killed)
   Trigger: SIGKILL OR parent kills
   Action: Direct termination
   Next: Cleanup
   Notes: Process never ran

8. WAITING → TERMINATED (Killed)
   Trigger: SIGKILL while blocked
   Action: Remove from wait queue
   Next: Cleanup
   Notes: Woken to die

9. NEW → TERMINATED (Failed Creation)
   Trigger: Resource exhaustion
   Action: Cleanup partial state
   Next: Nothing
   Notes: Rare, error case
```

### Transition Triggers

```
What Triggers Each Transition:

READY → RUNNING:
├── Scheduler decision
├── Time slice availability
├── Priority-based selection
├── CPU idle and process waiting
├── Fair scheduling opportunity
└── Preemption of another process

RUNNING → READY:
├── Time slice expiration
├── Timer interrupt fires
├── Higher priority process ready
├── Higher priority process arrived
├── Round-robin quantum expired
├── Scheduler preemption
└── Voluntary yield (sched_yield)

RUNNING → WAITING:
├── I/O system call (read, write)
├── wait() for child
├── sleep() call
├── Blocking lock acquisition
├── Semaphore wait (P operation)
├── Condition variable wait
├── Message receive (blocking)
├── Socket accept (blocking)
└── Page fault requiring I/O

WAITING → READY:
├── I/O completion interrupt
├── Signal delivery
├── Child termination
├── Lock released by another process
├── Semaphore signal (V operation)
├── Condition variable signal
├── Message arrival
├── Socket data arrival
├── Timer expiration
└── Event occurrence

RUNNING → TERMINATED:
├── Normal exit (exit())
├── Return from main
├── Fatal signal (SIGSEGV, SIGFPE)
├── SIGKILL (uncatchable)
├── Unhandled exception
└── Process self-termination
```

---

## State Queues

### The Data Structures

Each state has associated data structures:

```
State Queues:

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Ready Queue:                                               │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐              │
│  │ P1  │→│ P2  │→│ P3  │→│ P4  │→│ P5  │              │
│  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘              │
│                                                             │
│  Wait Queues (multiple, one per event type):                │
│                                                             │
│  Disk I/O Wait:                                             │
│  ┌─────┐  ┌─────┐                                          │
│  │ P6  │→│ P7  │                                          │
│  └─────┘  └─────┘                                          │
│                                                             │
│  Network Wait:                                              │
│  ┌─────┐  ┌─────┐  ┌─────┐                                 │
│  │ P8  │→│ P9  │→│ P10 │                                 │
│  └─────┘  └─────┘  └─────┘                                 │
│                                                             │
│  Signal Wait:                                               │
│  ┌─────┐                                                    │
│  │ P11 │                                                    │
│  └─────┘                                                    │
│                                                             │
│  Terminated (Zombie) List:                                  │
│  ┌─────┐  ┌─────┐                                          │
│  │ P12 │→│ P13 │                                          │
│  └─────┘  └─────┘                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Ready Queue:
├── One primary queue (or multiple for MLQ)
├── Contains all ready processes
├── Scheduler selects from here
├── Various implementations
└── Central to scheduling

Wait Queues:
├── Multiple queues (one per event type)
├── Each queue for a specific condition
├── Process waits in appropriate queue
├── Removed when event occurs
└── Efficient wake-up mechanism

Zombie List:
├── Terminated processes
├── Waiting for parent
├── PCB retained
├── Not in any queue
└── Eventually reaped
```

### Queue Operations

```
Queue Operations:

Ready Queue Operations:
├── enqueue(process)  - Add to ready queue
├── dequeue()         - Remove for execution
├── peek()            - Look without removing
├── remove(process)   - Remove specific process
└── is_empty()        - Check if empty

Wait Queue Operations:
├── sleep_on(queue)   - Add to wait queue
├── wake_up(queue)    - Wake one/all processes
├── wake_up_one()     - Wake one process
├── wake_up_all()     - Wake all processes
└── remove(process)   - Remove from queue

State Change Operations:
├── set_state(process, new_state)
│   ├── Update PCB state field
│   ├── Move between queues
│   ├── Handle queue-specific logic
│   └── May trigger scheduling
│
├── make_ready(process)
│   ├── Set state to READY
│   ├── Add to ready queue
│   ├── Check for preemption
│   └── Possibly set need_resched
│
├── make_waiting(process, queue)
│   ├── Set state to WAITING
│   ├── Add to specified wait queue
│   ├── Schedule another process
│   └── Yield CPU
│
└── make_terminated(process)
    ├── Set state to TERMINATED
    ├── Release resources
    ├── Notify parent
    └── Mark as zombie
```

### Kernel Data Structures

```
Process State in the PCB:

struct task_struct {  // Linux PCB
    volatile long state;    // Process state
    // State values in Linux:
    // TASK_RUNNING         (0) - Runnable or running
    // TASK_INTERRUPTIBLE   (1) - Interruptible sleep
    // TASK_UNINTERRUPTIBLE (2) - Uninterruptible sleep
    // __TASK_STOPPED       (4) - Stopped
    // __TASK_TRACED        (8) - Traced by debugger
    // EXIT_ZOMBIE         (16) - Zombie
    // EXIT_DEAD           (32) - Fully dead
    
    struct list_head tasks;      // Process list
    struct list_head run_list;   // Ready queue link
    wait_queue_head_t *wait_queue; // Wait queue
    // ... many more fields
};

Ready Queue Implementation:
├── Linked list (simple)
├── Red-black tree (CFS)
├── Multiple queues (MLQ)
├── Priority heap
└── Depends on scheduler

Wait Queue Structure:
struct wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};

struct wait_queue_entry {
    unsigned int flags;
    void *private;              // Usually task pointer
    wait_queue_func_t func;     // Wake-up function
    struct list_head entry;
};
```

---

## Process States in Linux

### Linux-Specific States

Linux uses a rich set of process states:

```
Linux Process States:

Code   Name                    ps Display
─────────────────────────────────────────────
R      TASK_RUNNING            R
       (running or runnable)

S      TASK_INTERRUPTIBLE      S
       (interruptible sleep)

D      TASK_UNINTERRUPTIBLE    D
       (uninterruptible sleep)

T      __TASK_STOPPED          T
       (stopped by signal)

t      __TASK_TRACED           t
       (traced by debugger)

Z      EXIT_ZOMBIE             Z
       (zombie)

X      EXIT_DEAD               X
       (fully dead)

I      TASK_IDLE               I
       (idle kernel thread)
```

### Detailed Linux State Descriptions

```
Linux State Details:

R - TASK_RUNNING:
├── Process is running OR runnable
├── On CPU or in run queue
├── Can be scheduled
├── Includes both running and ready
├── Linux doesn't distinguish running/ready in state field
└── Scheduler knows which is on CPU

S - TASK_INTERRUPTIBLE:
├── Sleeping, waiting for event
├── Can be woken by signal
├── Most common sleep state
├── Example: waiting for I/O, wait(), sleep()
├── Can be interrupted by signals
├── Must check for signal after wake
└── Typical for most sleeps

D - TASK_UNINTERRUPTIBLE:
├── Sleeping, waiting for event
├── Cannot be woken by signal
├── Only woken by specific event
├── Used when interruption would be dangerous
├── Example: disk I/O completion
├── Short duration usually
├── Cannot be killed even with SIGKILL
└── Can cause issues if stuck

T - __TASK_STOPPED:
├── Stopped by signal
├── SIGSTOP, SIGTSTP, SIGTTIN, SIGTTOU
├── Can be resumed with SIGCONT
├── Shell job control (Ctrl+Z)
├── Debugger stops
└── Not sleeping, but stopped

t - __TASK_TRACED:
├── Being traced by debugger
├── ptrace system call
├── Stopped for debugger inspection
├── Can set breakpoints
├── Different from T (semantics)
└── Debugger controls

Z - EXIT_ZOMBIE:
├── Process has terminated
├── Waiting for parent to reap
├── PCB retained
├── No code/data
├── Exit status stored
└── Will be removed when reaped

X - EXIT_DEAD:
├── Fully dead process
├── Being removed from system
├── Transitional state
├── Should be very brief
└── Rarely seen

I - TASK_IDLE:
├── Idle kernel thread
├── Not doing anything
├── Linux 4.14+
├── Doesn't contribute to load average
├── Special case for kernel threads
└── Shows as I in ps
```

### Viewing States with ps

```
ps Command State Indicators:

$ ps aux
USER   PID  %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root     1  0.0  0.1  16764  3444 ?     Ss   Oct01   0:02 /sbin/init
root     2  0.0  0.0      0     0 ?     S    Oct01   0:00 [kthreadd]
root     3  0.0  0.0      0     0 ?     I<   Oct01   0:00 [rcu_gp]
user  1234  0.5  1.2 123456 12345 pts/0 R+   10:30   0:05 ./myapp
user  5678  0.0  0.0  12345  1234 pts/0 S    10:30   0:00 -bash

STAT Column:
├── First character: state
│   ├── R: Running or runnable
│   ├── S: Interruptible sleep
│   ├── D: Uninterruptible sleep
│   ├── T: Stopped
│   ├── t: Traced
│   ├── Z: Zombie
│   ├── I: Idle
│   └── X: Dead
│
├── Additional flags:
│   ├── <: High priority (niced negative)
│   ├── N: Low priority (niced positive)
│   ├── L: Pages locked in memory
│   ├── s: Session leader
│   ├── l: Multi-threaded
│   ├── +: Foreground process group
│   └── Other flags

Examples:
├── Ss: Sleeping session leader
├── R+: Running, foreground
├── I<: Idle, high priority
├── D: Uninterruptible sleep
├── Z: Zombie
└── T: Stopped
```

### State Transitions in Linux

```
Linux State Transition Diagram:

                    fork()
                       │
                       ▼
            ┌─────────────────────┐
            │  TASK_RUNNING       │
            │  (Ready or Running) │
            └──────────┬──────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    schedule()    wait_event()   do_exit()
         │             │             │
         ▼             ▼             ▼
    ┌────────┐  ┌────────────┐  ┌──────────┐
    │ Running│  │ INTERRUPT- │  │ EXIT_    │
    │ on CPU │  │ IBLE       │  │ ZOMBIE   │
    └───┬────┘  └──────┬─────┘  └────┬─────┘
        │              │             │
    preempt()     wake_up()     wait() by parent
        │              │             │
        └──────────────┘             ▼
                       │         ┌──────────┐
                       │         │ EXIT_DEAD│
                       │         └──────────┘
                       │
            ┌─────────────────────┐
            │  TASK_RUNNING       │
            │  (Ready for CPU)    │
            └─────────────────────┘

Key Linux Insights:
├── TASK_RUNNING covers both running and ready
├── Scheduler decides which TASK_RUNNING process runs
├── Sleeping states (S, D) are for waiting
├── Interruptible sleep (S) can be woken by signals
├── Uninterruptible sleep (D) only by event
├── Stopped (T) is signal-controlled
├── Zombie (Z) is terminated but unreaped
└── All states are in task_struct.state field
```

---

## Why Different States?

### Design Rationale

Different states serve different purposes:

```
Why Have Multiple States?

1. Scheduling Information
   ├── Scheduler needs to know who can run
   ├── Ready vs. Waiting is crucial
   ├── Ready processes compete for CPU
   ├── Waiting processes don't
   └── Efficient scheduling requires state

2. Resource Management
   ├── Different states use different resources
   ├── Running: CPU + memory
   ├── Ready: memory only
   ├── Waiting: memory only (mostly)
   ├── Terminated: minimal
   └── Resource accounting per state

3. Wake-Up Mechanism
   ├── Wait queues organize waiting processes
   ├── Event wakes appropriate processes
   ├── Efficient wake-up
   ├── No polling required
   └── State identifies wait condition

4. Debugging and Monitoring
   ├── ps shows state
   ├── Administrators can see what's happening
   ├── Troubleshoot stuck processes
   ├── Identify resource issues
   └── State provides visibility

5. Correct Behavior
   ├── State prevents incorrect operations
   ├── Can't schedule waiting process
   ├── Can't wake running process
   ├── State ensures consistency
   └── Foundation of correct operation
```

### Historical Evolution

```
Historical State Models:

Early Batch Systems:
├── No states needed (one job at a time)
├── Job runs to completion
├── Or fails and stops
├── Simple model
└── No multiplexing

Multiprogramming Era:
├── Multiple jobs in memory
├── Need to know which can run
├── Ready vs. Waiting distinction
├── Blocked on I/O
└── Three-state model (Ready/Running/Blocked)

Time-Sharing Era:
├── Add preemption
├── Process can be preempted
├── Returns to ready queue
├── Add terminated state
└── Four or five-state model

Modern Systems:
├── More states (Linux has 7+)
├── Detailed distinctions
├── Interruptible vs. uninterruptible sleep
├── Stopped vs. traced
├── Zombie vs. dead
└── Rich state information
```

---

## Special Cases

### Kernel Threads

Kernel threads have special state considerations:

```
Kernel Thread States:

Differences from User Processes:
├── Run only in kernel space
├── No user address space
├── No user state
├── Created by kernel
├── Various purposes

States:
├── Same basic states
├── May have special idle state
├── Cannot be killed by signals
├── Not associated with user
└── Examples: kthreadd, kworker

Idle Kernel Threads:
├── TASK_IDLE state
├── Not doing anything
├── Won't run until needed
├── Doesn't count for load
├── Linux 4.14+
└── Shows as I in ps

Examples:
$ ps aux | grep '\['
root  2  0.0  0.0  0  0 ?  S  [kthreadd]
root  3  0.0  0.0  0  0 ?  I  [rcu_gp]
root  4  0.0  0.0  0  0 ?  I  [rcu_par_gp]
root  5  0.0  0.0  0  0 ?  I  [slub_flushwq]
root  7  0.0  0.0  0  0 ?  S  [kworker/0:0H]
```

### Zombie Processes

Zombies are in a special state:

```
Zombie State:

Definition:
├── Process has terminated
├── Parent hasn't reaped it
├── PCB retained
├── Exit status stored
└── No other resources

Why Zombies Exist:
├── Kernel must keep exit status
├── Parent needs to read it
├── wait() returns status
├── Then PCB is freed
├── Historical design choice
└── Standard Unix behavior

Zombie Creation:
1. Child calls exit()
2. Kernel releases resources
3. Kernel retains PCB
4. Kernel sets state to zombie
5. Kernel notifies parent (SIGCHLD)
6. Parent calls wait()
7. Kernel returns status
8. Kernel frees PCB
9. Zombie gone

Issues:
├── Consumes kernel memory
├── Uses PID
├── Can exhaust PIDs
├── Not usually critical
├── Large number can be problem
└── Fix: parent must wait

Example:
$ ps aux | grep defunct
user  1234  0.0  0.0  0  0 ?  Z  10:30  0:00 [myprocess] <defunct>
```

### Orphan Processes

Orphans are handled automatically:

```
Orphan Process:

Definition:
├── Process whose parent died
├── Still running
├── Reparented to init (PID 1)
├── Init becomes new parent
└── Continues execution

Automatic Handling:
├── Kernel detects parent death
├── Kernel finds new parent (init)
├── Reparenting is automatic
├── No explicit action needed
├── Init will reap eventually
└── Ensures cleanup

Example:
├── Process A creates child B
├── A terminates
├── B is now orphan
├── B reparented to init
├── B continues running
└── Init reaps B when done

Uses:
├── Daemon processes
├── Background tasks
├── Detached processes
├── Long-running services
└── Intentional orphaning

Command:
$ nohup command &
$ disown
# Both create orphans (effectively)
```

---

## Practical Implications

### For Application Programmers

```
What Programmers Should Know:

1. Blocking Operations
   ├── Cause transition to WAITING
   ├── Process yields CPU
   ├── Other processes can run
   ├── Return when operation completes
   ├── Example: read(), write(), wait()
   └── Efficient use of CPU

2. Process State Visibility
   ├── getpid(), getppid()
   ├── /proc/PID/status
   ├── Can check own state
   ├── Limited information
   └── Full state visible to admins

3. Synchronization
   ├── Mutexes, semaphores cause waiting
   ├── Process blocks until acquired
   ├── WAITING state
   ├── Woken when available
   └── Foundation of concurrency

4. Signals
   ├── Interrupt process at any time
   ├── Wake interruptible sleeps
   ├── Not uninterruptible sleeps
   ├── Can change state
   └── Handle carefully

5. Exit Handling
   ├── exit() causes TERMINATED
   ├── Parent must wait()
   ├── Otherwise zombie
   ├── Proper cleanup important
   └── Avoid zombie accumulation
```

### For System Administrators

```
What Sysadmins Should Know:

1. Monitoring
   ├── ps aux / ps -ef
   ├── top / htop
   ├── Watch process states
   ├── Identify problem states
   └── Take action when needed

2. Common States and Issues

   S (Sleeping):
   ├── Normal for most processes
   ├── Waiting for I/O or event
   ├── Usually fine
   └── Many is normal

   D (Uninterruptible):
   ├── Waiting for hardware I/O
   ├── Cannot be killed
   ├── Long time indicates problem
   ├── May indicate:
   │   ├── Disk issues
   │   ├── NFS hang
   │   ├── Driver bug
   │   └── Hardware failure
   └── Investigate if persistent

   T (Stopped):
   ├── Job control
   ├── Debugger
   ├── Can be resumed
   ├── Usually intentional
   └── SIGCONT to resume

   Z (Zombie):
   ├── Parent needs to wait()
   ├── Usually harmless
   ├── Many indicates bug
   ├── Check parent process
   └── Fix parent or kill parent

   R (Running):
   ├── Expected for active process
   ├── Always running could indicate:
   │   ├── CPU-intensive work
   │   ├── Infinite loop
   │   └── Busy-wait (bug)
   └── Check CPU usage

3. Troubleshooting
   ├── High CPU with R state
   │   └── Investigate process
   ├── Stuck in D state
   │   └── Check hardware, NFS
   ├── Accumulating zombies
   │   └── Check parent, fix
   └── Many processes in wait
       └── Normal unless excessive

4. Commands
   $ ps aux | grep " D "     # D state processes
   $ ps aux | grep " Z "     # Zombies
   $ top -o state            # Sort by state
   $ kill -STOP PID          # Stop process
   $ kill -CONT PID          # Continue process
   $ kill -9 PID             # Force kill
```

---

## Common Misconceptions

**Misconception 1:** "A process is either running or not"  
**Reality:** There are multiple "not running" states (ready, waiting, stopped, zombie), each with different semantics. A ready process will run soon; a waiting process won't until an event occurs.

**Misconception 2:** "Waiting and sleeping are the same"  
**Reality:** "Waiting" and "sleeping" are related but not identical. A process waiting for I/O is blocked. A process sleeping via sleep() is also blocked, but for a timer. "Interruptible" vs. "uninterruptible" sleep further distinguishes.

**Misconception 3:** "Zombies consume significant resources"  
**Reality:** Zombies consume only a PCB (a few KB). They don't use CPU or hold memory pages. The main concern is PID exhaustion if thousands accumulate.

**Misconception 4:** "Killed processes disappear immediately"  
**Reality:** Killed processes become zombies until reaped. The parent must wait() to fully remove them. If the parent doesn't wait, the zombie persists.

**Misconception 5:** "You can kill any process with SIGKILL"  
**Reality:** SIGKILL cannot be caught, but it doesn't work on processes in uninterruptible sleep (D state). They will terminate when they leave D state.

**Misconception 6:** "Ready and running are distinct states in Linux"  
**Reality:** Linux's TASK_RUNNING covers both running and ready. The scheduler distinguishes them, but the state field doesn't. This is different from the textbook five-state model.

**Misconception 7:** "Stopped processes are terminated"  
**Reality:** Stopped processes (T state) are paused, not terminated. They can be resumed with SIGCONT. This is used for job control (Ctrl+Z) and debugging.

---

## Summary

Process states represent the different conditions a process can be in during its lifetime. The standard five-state model (New, Ready, Running, Waiting, Terminated) captures the essential states, though real systems like Linux have more. State transitions occur in response to specific events, and each state has associated kernel data structures (queues).

### Key Points

1. Five fundamental states: New, Ready, Running, Waiting, Terminated.
2. State transitions are triggered by specific events.
3. Ready queue holds processes waiting for CPU.
4. Wait queues hold processes waiting for events.
5. Linux has more states: R, S, D, T, t, Z, I, X.
6. Interruptible vs. uninterruptible sleep matters for signal handling.
7. Zombies are terminated but unreaped processes.
8. Special cases: kernel threads, orphans, zombies.

---

## Key Takeaways

1. States represent reality: Not all processes are running.
2. Transitions are event-driven: Each state change is triggered.
3. Linux has rich states: More than the textbook five.
4. Wait queues are efficient: No polling needed.
5. Signal handling depends on state: Interruptible vs. not.
6. Zombies must be reaped: Parent responsibility.
7. Tools reveal states: ps, top, /proc.
8. Understanding states is essential: For debugging and optimization.

---

## What's Next?

Continue to Process Control Block to explore the kernel data structure that represents a process and stores its state.

---

## Further Reading

### Books

- "Operating System Concepts" by Silberschatz, Galvin, and Gagne
- "Modern Operating Systems" by Andrew S. Tanenbaum
- "Operating Systems: Three Easy Pieces" by Remzi and Andrea Arpaci-Dusseau
- "Linux Kernel Development" by Robert Love
- "Understanding the Linux Kernel" by Bovet and Cesati

### Online Resources

- OSTEP (ostep.org) — Free textbook
- man pages: ps(1), proc(5), signal(7)
- Linux Kernel Documentation (kernel.org)
- "The Linux Programming Interface" by Michael Kerrisk

