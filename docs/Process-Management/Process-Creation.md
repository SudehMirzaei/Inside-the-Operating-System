# Process Creation

## Introduction

Process creation is the mechanism by which new processes come into existence. Every process on a running system—from the first init process started during boot to the terminal command you just launched—was created by some mechanism. Understanding process creation is essential for understanding how operating systems manage execution, how programs become processes, and how process hierarchies form.

This document explores process creation in depth: the system calls involved, the mechanisms used, the differences between Unix and Windows approaches, the optimizations that make creation efficient, and the practical implications for programmers and system administrators.

---

## The Need for Process Creation

### Why Processes Are Created

```
Common Scenarios for Process Creation:

1. System Startup
   ├── Init process (first process)
   ├── System daemons
   ├── Background services
   └── Login managers

2. User Actions
   ├── Running a command in shell
   ├── Clicking an application icon
   ├── Opening a new terminal
   └── Starting a service

3. Application Behavior
   ├── Web server spawning workers
   ├── Chrome creating tab processes
   ├── Shell running scripts
   └── Make building project

4. Automated Tasks
   ├── Cron jobs
   ├── Systemd timers
   ├── Batch processing
   └── Scheduled maintenance

5. System Events
   ├── USB device inserted
   ├── Network connection
   ├── Hardware interrupt
   └── Login attempt

6. Process Failure/Recovery
   ├── Supervisor restarting worker
   ├── Init respawning getty
   ├── Watchdog restarting service
   └── Cluster rescheduling
```

### What Creation Involves

```
Process Creation Tasks:

1. Identity Assignment
   ├── Allocate unique PID
   ├── Set parent PID (PPID)
   ├── Inherit/set user IDs
   ├── Set process group
   └── Set session ID

2. Resource Allocation
   ├── Allocate PCB
   ├── Allocate address space
   ├── Set up page tables
   ├── Allocate kernel stack
   └── Initialize file descriptors

3. Program Loading (if exec)
   ├── Read executable
   ├── Validate format
   ├── Map segments into memory
   ├── Set up initial stack
   └── Set entry point

4. State Initialization
   ├── Set state to ready
   ├── Initialize CPU context
   ├── Set up registers
   ├── Configure scheduler info
   └── Initialize accounting

5. Resource Inheritance
   ├── File descriptors
   ├── Signal handlers
   ├── Environment variables
   ├── Current directory
   └── Resource limits

6. Registration
   ├── Add to process table
   ├── Add to parent's children
   ├── Add to PID hash
   ├── Add to scheduler queue
   └── Notify parent
```

---

## Unix/Linux Process Creation

### The fork() System Call

The fork() system call is the fundamental mechanism for process creation in Unix-like systems:

```
fork() Overview:

Signature:
#include <unistd.h>
pid_t fork(void);

Return Values:
├── In parent: child's PID (> 0)
├── In child: 0
├── On error: -1, errno set
└── Distinguishes parent from child

What fork() Does:
1. Creates a new process
2. Duplicates parent's address space
3. Duplicates parent's resources
4. Returns twice (once in each process)
5. Both processes continue from fork()

Key Points:
├── Child is a copy of parent
├── Same code, different execution path
├── Return value distinguishes them
├── No arguments needed
└── Simple but powerful
```

### fork() Example

```
Basic fork() Example:

#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        // Fork failed
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        // Child process
        printf("Child: PID=%d, PPID=%d\n", 
               getpid(), getppid());
        return 0;
    } else {
        // Parent process
        printf("Parent: PID=%d, Child PID=%d\n",
               getpid(), pid);
        wait(NULL);  // Wait for child
        return 0;
    }
}

Possible Output:
Parent: PID=1234, Child PID=1235
Child: PID=1235, PPID=1234

Note: Order may vary (scheduling dependent)

Process State After fork():
├── Parent: continues execution, PID=1234
├── Child: new process, PID=1235
├── Both have same code
├── Both continue from fork() return
├── Different return values
└── Branch based on return value
```

### The exec() Family

The exec() family replaces the calling process's program with a new one:

```
exec() Family of Functions:

#include <unistd.h>

execl(path, arg0, arg1, ..., NULL):
├── 'l' = list arguments
├── Path as first argument
├── Variable arguments
├── NULL terminator
└── Example: execl("/bin/ls", "ls", "-la", NULL)

execv(path, argv):
├── 'v' = vector (array) arguments
├── Path as first argument
├── argv array
└── Example: execv("/bin/ls", argv)

execle(path, arg0, ..., NULL, envp):
├── 'l' = list arguments
├── 'e' = environment
├── Custom environment
└── Example: execle("/bin/ls", "ls", NULL, envp)

execve(path, argv, envp):
├── 'v' = vector arguments
├── 'e' = environment
├── Most fundamental
├── System call
└── Example: execve("/bin/ls", argv, envp)

execlp(file, arg0, ..., NULL):
├── 'l' = list arguments
├── 'p' = use PATH
├── Searches PATH
└── Example: execlp("ls", "ls", "-la", NULL)

execvp(file, argv):
├── 'v' = vector arguments
├── 'p' = use PATH
└── Example: execvp("ls", argv)

Common Behavior:
├── Replace current program
├── Do not return on success
├── Return only on error (-1)
├── Keep PID, PPID
├── Keep file descriptors (usually)
├── Reset signal handlers
├── Preserve environment (unless specified)
├── Credentials preserved (unless specified)
├── Resource limits preserved (unless specified)
└── PID unchanged
```

### The fork+exec Pattern

The classic Unix pattern combines fork and exec:

```
fork+exec Pattern:

Standard Process Creation:

#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        // Child: replace with new program
        execl("/bin/ls", "ls", "-la", NULL);
        // If we reach here, exec failed
        perror("exec failed");
        _exit(127);  // Use _exit in child
    } else {
        // Parent: wait for child
        int status;
        waitpid(pid, &status, 0);
        
        if (WIFEXITED(status)) {
            printf("Child exited with %d\n",
                   WEXITSTATUS(status));
        }
        return 0;
    }
}

Why Two Steps?

1. fork() alone is useful:
   ├── Parallel processing
   ├── Multiple workers
   ├── Same program, multiple processes
   └── Server architecture

2. exec() alone would replace parent:
   ├── Original program gone
   ├── Not always desired
   ├── Need parent to continue
   └── Loss of control

3. Combined: best of both:
   ├── Parent continues
   ├── Child becomes new program
   ├── Flexible and powerful
   └── Foundation of Unix

Alternatives:
├── posix_spawn() - Combined operation
├── vfork() - Faster fork (shared memory)
├── clone() - Linux-specific, flexible
└── CreateProcess() - Windows approach
```

### The fork() Implementation

Understanding how fork() is implemented helps explain its behavior:

```
fork() Kernel Implementation (Simplified):

sys_fork():
├── Allocate new task_struct
├── Copy parent's task_struct
│   ├── Copy most fields
│   ├── Generate new PID
│   ├── Reset some fields
│   └── Link to parent
│
├── Copy process resources:
│   ├── Copy mm_struct (memory)
│   │   └── Copy-on-write page tables
│   ├── Copy files_struct (file descriptors)
│   │   └── Increment file reference counts
│   ├── Copy signal handlers
│   │   └── Some reset to default
│   ├── Copy credentials
│   └── Copy various other structures
│
├── Set up child's kernel stack:
│   ├── Allocate kernel stack
│   ├── Set up task state
│   ├── Prepare for return to user
│   └── Set return value to 0
│
├── Add to data structures:
│   ├── Add to task list
│   ├── Add to parent's children
│   ├── Add to PID hash
│   ├── Add to thread group
│   └── Add to run queue (if ready)
│
├── Wake child (make ready):
│   └── Child is now schedulable
│
└── Return to both:
    ├── Parent returns child's PID
    └── Child returns 0

Important Details:
├── Child starts with one thread
├── Child inherits parent's memory (COW)
├── File descriptors shared (not duplicated)
├── Pending signals cleared
├── File locks not inherited
├── Timers not inherited
├── Resource limits inherited
└── Signal handlers inherited
```

---

### Copy-on-Write (COW)

The most important optimization for process creation is copy-on-write:

#### The Problem with Naive fork()

```
Naive fork() (No COW):

Scenario: Parent has 1 GB memory

Without COW:
1. fork() called
2. Kernel copies all 1 GB
3. Child has own 1 GB
4. Parent has 1 GB
5. Total: 2 GB used

Problems:
├── Slow: Copying 1 GB takes time
├── Memory wasteful: Doubles memory
├── Often unnecessary
├── Most forks followed by exec()
│   └── exec() discards memory anyway
└── Poor performance

Timing:
├── Copy 1 GB at ~10 GB/s: 100 ms
├── Plus page table copying
├── Plus other overhead
├── Significant delay
└── Process creation feels slow
```

#### The COW Solution

```
Copy-on-Write fork():

How It Works:
1. fork() called
2. Kernel does NOT copy memory
3. Page tables copied (small)
4. Both processes share physical pages
5. Pages marked read-only
6. On write attempt:
   ├── Page fault occurs
   ├── Kernel copies the page
   ├── Both processes get own copy
   ├── Update page tables
   └── Write proceeds

Visual:
Before fork():
┌─────────┐
│ Parent  │
│ Pages:  │
│ A B C D │
└────┬────┘
     │
     │ fork()
     ▼
Immediately After:
┌─────────┐     ┌─────────┐
│ Parent  │     │ Child   │
│ Page T. │     │ Page T. │
│  ↓↓↓↓   │     │  ↓↓↓↓   │
│ A B C D │◀───▶│ A B C D │  (shared, read-only)
└─────────┘     └─────────┘

After Parent Writes to B:
┌─────────┐     ┌─────────┐
│ Parent  │     │ Child   │
│ Page T. │     │ Page T. │
│  ↓↓↑↓   │     │  ↓↓↓↓   │
│ A B' C D│     │ A B C D │  (B copied for parent)
└─────────┘     └─────────┘

Benefits:
├── Fast fork() (no memory copy)
├── Memory efficient (sharing)
├── Only copied pages use extra memory
├── Most forks followed by exec() → no copy at all
└── Massive optimization

Cost:
├── Page fault overhead on writes
├── More complex kernel code
├── TLB effects on faults
├── Slight complexity in debugging
└── Worth it for performance
```

### COW in Practice

```
COW Example:

Shell running a command:

$ ls -la

1. Shell (bash) has some memory
2. Shell calls fork()
3. Kernel creates child
4. Child shares shell's memory (COW)
5. fork() returns quickly
6. Child calls exec("ls", "-la")
7. Kernel replaces child's memory
8. Child's shared pages released
9. No memory was copied!
10. Shell unaffected

Result:
├── fork() was very fast
├── No memory duplication
├── exec() replaced memory
├── Efficient process creation
└── This is the common case!

Case Where COW Matters:
├── Process forks without exec
├── Both processes continue
├── Parent and child run same code
├── Eventually they diverge
├── COW copies only what's needed
└── Still efficient

Example: Web Server
├── Parent forks worker
├── Worker handles request
├── Worker modifies some data
├── Only those pages copied
├── Shared data remains shared
└── Memory efficient
```

### vfork()

An alternative to fork() for specific cases:

```
vfork() System Call:

Purpose:
├── Faster than fork() in some cases
├── Parent suspended until child execs
├── No memory copying at all
├── Child shares parent's memory
└── Dangerous but fast

Behavior:
1. vfork() called
2. Parent process suspended
3. Child uses parent's memory
4. Child must call exec() or exit()
5. Parent resumes

Use Case:
├── Child immediately execs
├── No memory modification
├── Faster than fork()
├── Deprecated in modern systems
└── Use posix_spawn() instead

Dangers:
├── Child must not modify memory
├── Child must not return from function
├── Child must exec or exit
├── Parent blocked until then
├── Violating these causes undefined behavior
└── Deprecated in favor of posix_spawn()

Modern Replacement:
posix_spawn() - Combines fork+exec
├── Safer than vfork()
├── Often faster than fork+exec
├── Portable
├── Recommended for new code
└── Used by glibc internally
```

---

### posix_spawn()

The posix_spawn() function provides a simpler, more efficient way to create processes:

```
posix_spawn() Overview:

Signature:
#include <spawn.h>

int posix_spawn(
    pid_t *pid,                    // Output: child PID
    const char *path,              // Program to execute
    const posix_spawn_file_actions_t *file_actions,
    const posix_spawnattr_t *attrp,
    char *const argv[],            // Arguments
    char *const envp[]             // Environment
);

Advantages:
├── Combines fork+exec
├── Can be more efficient
├── Handles edge cases
├── Portable (POSIX standard)
├── Simpler to use
└── Less error-prone

File Actions:
├── posix_spawn_file_actions_addopen()
├── posix_spawn_file_actions_addclose()
├── posix_spawn_file_actions_adddup2()
├── Specify what child should do
└── Avoids race conditions

Attributes:
├── Scheduling policy
├── Scheduling parameters
├── Signal mask
├── Signal handlers
├── Process group
└── Various flags

Example:
#include <spawn.h>
#include <stdio.h>
#include <sys/wait.h>

int main() {
    pid_t pid;
    char *argv[] = {"ls", "-la", NULL};
    extern char **environ;
    
    int ret = posix_spawn(&pid, "/bin/ls", NULL, NULL,
                          argv, environ);
    if (ret != 0) {
        perror("posix_spawn failed");
        return 1;
    }
    
    int status;
    waitpid(pid, &status, 0);
    return 0;
}

Implementation:
├── Uses vfork() or fork() internally
├── Optimized by C library
├── Handles all edge cases
├── Efficient for common cases
└── Recommended for new code
```

---

## Windows Process Creation

Windows uses a different approach with CreateProcess():

```
CreateProcess() Function:

Signature:
BOOL CreateProcess(
    LPCSTR lpApplicationName,        // Program path
    LPSTR lpCommandLine,             // Command line
    LPSECURITY_ATTRIBUTES lpProcessAttributes,
    LPSECURITY_ATTRIBUTES lpThreadAttributes,
    BOOL bInheritHandles,            // Inherit handles?
    DWORD dwCreationFlags,           // Flags
    LPVOID lpEnvironment,            // Environment
    LPCSTR lpCurrentDirectory,       // Working dir
    LPSTARTUPINFO lpStartupInfo,     // Startup info
    LPPROCESS_INFORMATION lpProcessInformation
);

Key Differences from Unix:
├── Single call (no fork/exec split)
├── More parameters
├── Explicit control
├── Creates process + initial thread
├── Returns handles to both
└── More efficient for common case

Example:
#include <windows.h>

int main() {
    STARTUPINFO si = {0};
    PROCESS_INFORMATION pi = {0};
    si.cb = sizeof(si);
    
    BOOL success = CreateProcess(
        NULL,                       // Auto-detect
        "notepad.exe",              // Command
        NULL, NULL,                 // Default security
        FALSE,                      // Don't inherit handles
        0,                          // No special flags
        NULL, NULL,                 // Environment, cwd
        &si, &pi                    // Output structures
    );
    
    if (success) {
        printf("Created process with PID: %lu\n",
               pi.dwProcessId);
        printf("Thread ID: %lu\n", pi.dwThreadId);
        
        WaitForSingleObject(pi.hProcess, INFINITE);
        
        CloseHandle(pi.hProcess);
        CloseHandle(pi.hThread);
    }
    return 0;
}

Process vs. Threads:
├── CreateProcess creates both
├── Process is container
├── Initial thread is execution
├── Handle to each returned
├── Must close both
└── Different model from Unix
```

### Unix vs. Windows Comparison

```
Process Creation Models:

Unix/Linux:
├── fork() + exec()
├── Two steps
├── Flexible
├── Child inherits almost everything
├── Return values distinguish parent/child
├── File descriptors shared
├── Classic approach
└── Foundation of Unix philosophy

Windows:
├── CreateProcess()
├── One step
├── Explicit parameters
├── Inheritance is opt-in
├── Returns handles
├── Different security model
├── Efficient for common case
└── Designed for Windows

When to Use Which:
├── Unix: Use fork+exec or posix_spawn
├── Windows: Use CreateProcess
├── Cross-platform: Use portable libraries
├── Simplest: posix_spawn on Unix
├── Most control: fork+exec on Unix
└── Most Windows-like: CreateProcess

Portability Libraries:
├── libuv: Cross-platform async
├── GLib: Cross-platform utilities
├── Boost.Process: C++ library
├── Qt: Cross-platform framework
└── Various others
```

---

## Process Hierarchies

### Parent-Child Relationships

Process creation creates hierarchies:

```
Process Hierarchy:

Unix/Linux (Tree):
init (PID 1)
├── systemd-udevd (device management)
├── sshd (SSH daemon)
│   └── sshd (user session)
│       └── bash (shell)
│           ├── vim (editor)
│           └── ls (command)
├── cron (scheduler)
│   └── some-job (scheduled task)
├── nginx (web server)
│   ├── nginx worker
│   ├── nginx worker
│   └── nginx worker
└── login
    └── bash
        └── firefox
            ├── firefox (renderer)
            ├── firefox (renderer)
            └── firefox (GPU)

Characteristics:
├── Every process has a parent
├── Except init (PID 1)
├── Parent can wait for children
├── Parent death → reparent to init
├── Orphaned processes reparented
└── Tree structure

Commands:
$ pstree              # Tree view
$ ps -ef              # Shows PPID
$ ps auxf             # Forest view

Windows (More Flat):
├── Processes tracked but less hierarchical
├── Parent-child tracked by handle
├── No strict tree structure
├── Processes more independent
├── Handle inheritance opt-in
└── Different model
```

### Inheriting Attributes

Children inherit attributes from parents:

```
Attributes Inherited Across fork():

Inherited:
├── File descriptors
│   ├── Same open files
│   ├── Same file offsets
│   ├── Same access modes
│   └── Shared (not copies)
├── Signal handlers
│   ├── Same handlers
│   ├── Same masks (mostly)
│   └── Pending signals cleared
├── Environment variables
│   ├── Same environment
│   ├── Can be changed
│   └── Independent copies
├── Current working directory
│   ├── Same cwd initially
│   ├── Can change independently
│   └── Independent after fork
├── User and group IDs
│   ├── Same credentials
│   ├── Can be changed (privileges)
│   └── Usually same
├── Resource limits
│   ├── Same ulimits
│   ├── Can be changed
│   └── Independent after fork
└── Other attributes
    ├── Signal masks
    ├── Timers (not inherited - cleared)
    ├── File locks (not inherited)
    └── Various others

NOT Inherited:
├── Pending signals (cleared)
├── File locks
├── Timers
├── Process ID (new PID)
├── Parent's parent (PPID = parent)
├── Resource usage (fresh)
└── Some memory management info

On exec():
├── Text/data/heap/stack replaced
├── Signal handlers reset
├── Signal mask preserved
├── File descriptors preserved (usually)
├── Environment preserved (usually)
├── Credentials preserved (usually)
├── Resource limits preserved (usually)
└── PID unchanged
```

---

## Process Creation in Practice

### Common Patterns

```
Process Creation Patterns:

1. Run a Command (Shell):
   $ ls -la
   ├── Shell forks
   ├── Child execs ls
   ├── Shell waits
   └── Shell continues

2. Background Command:
   $ long_running &
   ├── Shell forks
   ├── Child execs long_running
   ├── Shell doesn't wait
   ├── Shell returns prompt
   └── Child runs in background

3. Pipeline:
   $ ls | grep .txt | wc -l
   ├── Shell creates 3 processes
   ├── ls writes to pipe 1
   ├── grep reads pipe 1, writes pipe 2
   ├── wc reads pipe 2
   └── Shell waits for all

4. Redirection:
   $ command > output.txt
   ├── Shell forks
   ├── Child opens output.txt
   ├── Child dup2s to stdout
   ├── Child execs command
   └── Output goes to file

5. Subshell:
   $ (cd /tmp && ls)
   ├── Shell forks
   ├── Child changes directory
   ├── Child runs ls
   ├── Child exits
   └── Parent's directory unchanged

6. Command Substitution:
   $ files=$(ls)
   ├── Shell forks
   ├── Child runs ls
   ├── Output captured
   ├── Child exits
   └── Variable set from output
```

### Real-World Example: Shell Command

```
Complete Shell Command Execution:

User types: ls -la /tmp

Step 1: Shell Reads and Parses
├── Shell (bash, PID 1000) is waiting
├── User presses Enter
├── Shell reads input
├── Parses: command="ls", args=["ls", "-la", "/tmp"]
├── Determines it's a simple command
└── Prepares to execute

Step 2: Shell Forks
├── Shell calls fork()
├── Kernel creates child (PID 1234)
├── Child is copy of shell
├── Both continue from fork()
├── Parent gets PID 1234
└── Child gets 0

Step 3: Child Execs
├── Child (PID 1234) calls exec
├── execve("/bin/ls", ["ls", "-la", "/tmp"], envp)
├── Kernel replaces child's program
├── Old shell code discarded
├── New ls code loaded
├── Child becomes ls process
└── Child starts executing ls

Step 4: Parent Waits
├── Parent (shell, PID 1000) calls wait()
├── Parent enters WAITING state
├── Parent is blocked
├── Other processes run
└── Parent waits for PID 1234

Step 5: ls Executes
├── ls runs (PID 1234)
├── Reads /tmp directory
├── Formats output
├── Writes to stdout (terminal)
├── Output appears on screen
├── ls finishes
└── ls calls exit(0)

Step 6: Child Terminates
├── Kernel processes exit
├── Resources released
├── Exit status stored
├── Child becomes zombie
├── Parent notified (SIGCHLD)
└── Parent wakes up

Step 7: Parent Reaps
├── Parent's wait() returns
├── Returns PID 1234
├── Status = 0 (success)
├── Zombie child removed
├── Shell continues
└── Prompt displayed again

Timeline:
0ms     Shell parsing input
1ms     fork() called
2ms     Child created
3ms     Child execs ls
4ms     Parent waits
5ms     ls starts executing
10ms    ls finishes reading /tmp
15ms    ls output complete
16ms    ls exits
17ms    Parent reaps
18ms    Shell shows prompt

Total: ~18ms (imperceptible to user)
```

### Server Process Creation

```
Web Server Process Creation:

Nginx architecture:

Master process (PID 1000):
├── Started by systemd
├── Reads configuration
├── Binds to port 80/443
├── Creates worker processes
├── Monitors workers
└── Restarts failed workers

Worker creation:
├── Master calls fork()
├── Child inherits listening sockets
├── Child runs worker code
├── Multiple workers (typically CPU cores)
├── Each handles requests independently
└── Isolation for reliability

Why processes (not threads):
├── Isolation (worker crash doesn't affect others)
├── No shared memory bugs
├── Simpler programming model
├── Better for stability
└── Common for servers

Process hierarchy:
systemd (PID 1)
└── nginx master (PID 1000)
    ├── nginx worker (PID 1001)
    ├── nginx worker (PID 1002)
    ├── nginx worker (PID 1003)
    └── nginx worker (PID 1004)

Benefits:
├── Fault isolation
├── Easy scaling (add workers)
├── Zero-downtime reload
├── Simpler concurrency model
└── Proven in production
```

### Browser Process Architecture

```
Modern Browser Process Architecture:

Chrome/Chromium:

Browser process (main):
├── Manages UI
├── Coordinates tabs
├── Handles user input
├── Network requests
└── Creates renderer processes

Renderer processes (one per tab):
├── Isolated from each other
├── Render web content
├── Run JavaScript
├── Cannot access system directly
└── Sandboxed

GPU process:
├── Hardware acceleration
├── Graphics rendering
├── Shared across tabs
└── Isolated

Plugin processes:
├── Adobe Flash (historical)
├── Isolated for security
├── Can crash independently
└── Doesn't affect browser

Utility processes:
├── Network service
├── Storage service
├── Audio service
└── Various helpers

Benefits:
├── Stability (one tab doesn't crash all)
├── Security (sandboxing)
├── Performance (parallel rendering)
├── Resource limits per process
└── Fault isolation

Process count:
├── Browser: 1
├── Renderers: 1-100+ (one per tab)
├── GPU: 1
├── Utilities: 5-20
├── Total: Can be 100+ processes
└── Normal for modern browsers
```

---

## Process Creation Internals

### Kernel Data Structure Changes

```
PCB Creation in Kernel:

1. Allocate task_struct
   ├── kmem_cache_alloc() in Linux
   ├── From task_struct cache
   ├── Fast allocation
   └── Initialize fields

2. Copy Parent's task_struct
   ├── Duplicate most fields
   ├── Copy memory management info
   ├── Copy file descriptors
   ├── Copy signal handlers
   └── Various other fields

3. Modify for Child
   ├── New PID
   ├── PPID = parent's PID
   ├── Clear pending signals
   ├── Reset some counters
   ├── Initialize state
   └── Set parent pointer

4. Set Up Child Resources
   ├── Allocate kernel stack
   ├── Set up thread_info
   ├── Prepare CPU context
   ├── Set return value
   └── Set up scheduler info

5. Link Into Structures
   ├── Add to task_list
   ├── Add to parent's children
   ├── Add to PID hash
   ├── Add to thread group
   └── Add to run queue (if ready)

6. Wake Child
   ├── If child is ready
   ├── Add to scheduler
   ├── Wake up (if needed)
   └── Child becomes runnable

Timing:
├── Most operations fast
├── COW avoids memory copy
├── File descriptor copy fast
├── Total time: microseconds
└── Millions of forks per second possible
```

### The clone() System Call

Linux provides clone() for fine-grained control:

```
clone() System Call:

Purpose:
├── Fine-grained control over creation
├── Share or copy various resources
├── Foundation for threads
├── Flexible primitive
└── Used by pthread_create()

Signature:
int clone(
    int (*fn)(void *),    // Function to run
    void *child_stack,    // Child's stack
    int flags,            // What to share
    void *arg,            // Argument to fn
    ...
);

Flags Control Sharing:
├── CLONE_VM: Share memory
├── CLONE_FS: Share file system info
├── CLONE_FILES: Share file descriptors
├── CLONE_SIGHAND: Share signal handlers
├── CLONE_THREAD: Same thread group
├── CLONE_NEWNS: New mount namespace
├── CLONE_NEWPID: New PID namespace
├── CLONE_NEWNET: New network namespace
└── Many more

Uses:
├── fork(): clone() with no sharing
├── vfork(): clone() with CLONE_VM | CLONE_VFORK
├── Threads: clone() with CLONE_VM | CLONE_FILES | ...
├── Containers: clone() with namespace flags
└── Various specialized uses

Example (simplified):
// Like fork()
clone(fn, stack, 0, arg);

// Like thread
clone(fn, stack, 
      CLONE_VM | CLONE_FS | CLONE_FILES | 
      CLONE_SIGHAND | CLONE_THREAD,
      arg);

// Container
clone(fn, stack,
      CLONE_NEWPID | CLONE_NEWNET | 
      CLONE_NEWNS,
      arg);
```

---

## Common Misconceptions

**Misconception 1:** "fork() copies all memory"  
**Reality:** fork() uses copy-on-write. Memory is shared until modified. This makes fork() very efficient, especially when followed by exec().

**Misconception 2:** "fork() is slow"  
**Reality:** With COW, fork() is fast. It's the subsequent memory modifications that may be slower due to page faults. For fork+exec (common case), it's very efficient.

**Misconception 3:** "Child processes start executing from main()"  
**Reality:** Child processes continue from the fork() return point, not from main(). The child sees fork() return 0 and continues execution from there.

**Misconception 4:** "exec() creates a new process"  
**Reality:** exec() replaces the current process's program. It doesn't create a new process. The PID remains the same. Only the code and data change.

**Misconception 5:** "fork() and exec() are always used together"  
**Reality:** They're often used together but not always. fork() alone is useful for parallel processing. exec() alone is rare (replaces shell). Combined is the common case.

**Misconception 6:** "Windows uses fork like Unix"  
**Reality:** Windows doesn't have fork(). It uses CreateProcess() which creates a new process and loads a program in one call. Different model, different design.

**Misconception 7:** "You can create processes without a parent"  
**Reality:** Every process (except init) has a parent. When a parent dies, children are reparented to init. There's no "orphan creation" at birth—all processes are created by existing processes.

---

## Summary

Process creation is the mechanism by which new processes come into existence. Unix-like systems use fork() (to create a copy of the current process) and exec() (to replace the program). Windows uses CreateProcess() as a single operation. Copy-on-write optimizes fork() by avoiding memory copies until needed. Process hierarchies form parent-child relationships that organize the system.

### Key Points

1. fork() creates a copy of the calling process.
2. exec() replaces the program in the current process.
3. Combined, they create new processes running new programs.
4. Copy-on-write optimizes fork() by sharing memory.
5. posix_spawn() is the modern, portable alternative.
6. Windows uses CreateProcess() instead of fork+exec.
7. Process hierarchies form parent-child trees.
8. Inheritance determines what children receive from parents.

---

## Key Takeaways

1. Process creation is fundamental — every process comes from another.
2. Unix uses fork+exec — two steps, flexible, powerful.
3. COW makes fork() efficient — no immediate memory copy.
4. posix_spawn is modern — simpler and often faster.
5. Windows is different — CreateProcess does it all in one call.
6. Children inherit much — file descriptors, environment, etc.
7. Hierarchies organize processes — parent-child trees.
8. Real systems use these patterns — browsers, servers, shells.

---

## What's Next?

Continue to Process Termination to learn how processes end, including exit(), wait(), and cleanup mechanisms.

---

## Further Reading

### Books

- "Operating System Concepts" by Silberschatz, Galvin, and Gagne
- "Modern Operating Systems" by Andrew S. Tanenbaum
- "Operating Systems: Three Easy Pieces" by Remzi and Andrea Arpaci-Dusseau
- "The Linux Programming Interface" by Michael Kerrisk
- "Linux Kernel Development" by Robert Love
- "Windows Internals" by Russinovich, Solomon, and Ionescu

### Online Resources

- OSTEP (ostep.org) — Free textbook
- man pages: fork(2), exec(3), posix_spawn(3), clone(2)
- Linux Kernel Documentation (kernel.org)
- "The Linux Programming Interface" — Chapters 24-28

