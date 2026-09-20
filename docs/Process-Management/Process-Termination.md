# Process Termination

## Introduction

Every process eventually comes to an end. Whether it completes successfully, encounters an error, is killed by a user, or is terminated by the operating system, all processes must terminate. Process termination is the counterpart to process creation—it's the mechanism by which processes release their resources, notify their parent, and are finally removed from the system.

Understanding process termination is essential because:

- Every process has a lifecycle that ends in termination
- Improper termination handling leads to resource leaks and zombie processes
- Termination semantics affect application design
- Signal-based termination is fundamental to Unix/Linux
- Understanding cleanup helps debug resource issues

This document explores process termination in depth: the ways processes can terminate, the system calls involved, what happens during cleanup, the special states of zombies and orphans, and best practices for proper termination handling.

---

## Ways Processes Terminate

### Categories of Termination

```
Process Termination Categories:

1. Normal Termination (Voluntary)
   ├── Return from main()
   ├── Call to exit()
   ├── Call to _exit() or _Exit()
   └── Return from start routine (threads)

2. Error Termination (Voluntary)
   ├── exit() with non-zero status
   ├── Explicit error handling
   ├── Program-detected error
   └── Controlled shutdown

3. Fatal Error (Involuntary)
   ├── Signal from OS
   ├── SIGSEGV (segmentation fault)
   ├── SIGFPE (arithmetic error)
   ├── SIGBUS (bus error)
   ├── SIGILL (illegal instruction)
   └── SIGABRT (abort)

4. Killed by Another Process (Involuntary)
   ├── SIGTERM (polite request)
   ├── SIGKILL (forced)
   ├── SIGINT (Ctrl+C)
   ├── SIGQUIT (Ctrl+\)
   └── Various other signals

5. Parent Termination (Indirect)
   ├── Parent dies
   ├── Child reparented to init
   ├── Or killed with parent
   └── Depends on configuration
```

### Termination Signal Defaults

```
Common Termination Signals:

SIGTERM (15):
├── Polite termination request
├── Can be caught and handled
├── Can be ignored
├── Default: terminate
├── Graceful shutdown opportunity
└── Standard way to request termination

SIGKILL (9):
├── Forced termination
├── Cannot be caught
├── Cannot be ignored
├── Cannot be blocked
├── Immediate termination
└── Last resort

SIGINT (2):
├── Interrupt from keyboard
├── Ctrl+C
├── Can be caught
├── Default: terminate
└── Interactive interruption

SIGQUIT (3):
├── Quit from keyboard
├── Ctrl+\
├── Can be caught
├── Default: terminate and core dump
└── Debugging

SIGHUP (1):
├── Hangup detected
├── Terminal closed
├── Can be caught
├── Default: terminate
└── Often used to reload config

SIGSEGV (11):
├── Segmentation violation
├── Invalid memory access
├── Can be caught (rarely useful)
├── Default: terminate and core dump
└── Programming error

SIGFPE (8):
├── Arithmetic exception
├── Division by zero
├── Can be caught
├── Default: terminate and core dump
└── Programming error

SIGABRT (6):
├── Abort signal
├── Raised by abort()
├── Can be caught
├── Default: terminate and core dump
└── Assertion failure

SIGPIPE (13):
├── Write to broken pipe
├── Can be caught
├── Default: terminate
└── Broken pipe error
```

---

## Normal Termination

### Return from main()

The most common way a process terminates normally:

```
Return from main():

Code:
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello, World!\n");
    return 0;  // Normal termination
}

What Happens:
1. main() returns 0
2. C runtime startup code (crt0) runs
3. Return value passed to exit()
4. exit() called with status 0
5. Process terminates

Behind the Scenes:
├── _start() calls main()
├── main() returns to _start()
├── _start() calls exit(ret)
├── exit() performs cleanup
└── Process terminates

Status:
├── 0 = success
├── Non-zero = failure
├── Convention, not enforced
├── Shell uses it ($?)
└── Parent reads via wait()

Return Value:
├── int typically
├── Only low 8 bits used
├── 0-255 range (Unix)
├── 256+ wraps
└── Some systems: more bits
```

### The exit() Function

The standard C library function for normal termination:

```
exit() Function:

Signature:
#include <stdlib.h>
void exit(int status);
_Noreturn void exit(int status);  // C11

Behavior:
1. Call all atexit() handlers
   ├── In reverse order of registration
   ├── LIFO (stack-like)
   ├── Can register up to ATEXIT_MAX
   └── Common: cleanup functions
   
2. Flush stdio buffers
   ├── Flush stdout, stderr
   ├── Close all streams (implicitly)
   ├── Write buffered data
   └── Important for output
   
3. Remove temporary files
   ├── Files created with tmpfile()
   ├── Automatic cleanup
   └── Others may remain
   
4. Call _exit(status)
   ├── System call
   ├── Direct kernel interface
   ├── No further cleanup
   └── Process terminates

Key Points:
├── Does not return
├── Marked _Noreturn (C11)
├── Clean shutdown
├── Standard way to exit
└── Recommended for normal exit

atexit() Example:
#include <stdlib.h>
#include <stdio.h>

void cleanup1(void) {
    printf("Cleanup 1\n");
}

void cleanup2(void) {
    printf("Cleanup 2\n");
}

int main() {
    atexit(cleanup1);
    atexit(cleanup2);
    printf("Main\n");
    exit(0);
}

Output:
Main
Cleanup 2
Cleanup 1

(Reverse order: LIFO)
```

### The _exit() Function

The system call for immediate termination:

```
_exit() Function:

Signature:
#include <unistd.h>
void _exit(int status);
_Noreturn void _exit(int status);  // C11

Behavior:
1. No atexit() handlers called
2. No stdio flushing
3. No cleanup by library
4. Direct kernel call
5. Immediate termination

Use Cases:
├── After fork() in child
│   ├── Avoid double flushing
│   ├── Avoid duplicate cleanup
│   ├── Child didn't inherit everything
│   └── Safe to use _exit()
│
├── In signal handlers
│   ├── Safe function (async-signal-safe)
│   ├── exit() is not safe in signal handler
│   ├── _exit() is safe
│   └── Use _exit() in handlers
│
├── When cleanup would be wrong
│   ├── Error paths
│   ├── After exec failure
│   ├── Emergency termination
│   └── Explicit control needed
│
└── Performance-critical paths
    ├── No cleanup overhead
    ├── Faster termination
    ├── Rarely worth it
    └── Debugging harder

Example (fork/exec):
pid_t pid = fork();
if (pid == 0) {
    execl("/bin/ls", "ls", NULL);
    // If we get here, exec failed
    perror("exec failed");
    _exit(127);  // Correct: use _exit, not exit
} else {
    // Parent
    wait(NULL);
}

Why _exit() Here:
├── Child shares buffers with parent
├── exit() would flush those buffers
├── Double output possible
├── _exit() avoids this
└── Standard practice
```

### Comparison: exit() vs. _exit()

```
exit() vs. _exit():

Feature                exit()          _exit()
─────────────────────────────────────────────────
atexit handlers        Yes             No
stdio flushing         Yes             No
Temp file cleanup      Yes             No
Library cleanup        Yes             No
System call            Calls _exit     Direct
Async-signal-safe      No              Yes
Use in signal handler  No              Yes
Use after fork()       No              Yes
Speed                  Slower          Faster
Standard C             Yes             POSIX

Guidelines:
├── Normal exit: exit()
├── After fork in child: _exit()
├── In signal handler: _exit()
├── Emergency exit: _exit()
└── Default: exit()
```

---

## Abnormal Termination

### Signal-Based Termination

Signals cause most abnormal terminations:

```
Signal Handling Flow:

1. Signal Generated
   ├── Hardware exception (SIGSEGV)
   ├── Software raise (raise(), kill())
   ├── User action (Ctrl+C)
   ├── Timer expiration
   └── Various sources

2. Signal Pending
   ├── Signal marked for delivery
   ├── Process may be running
   ├── Signal is pending
   └── Will be delivered soon

3. Signal Delivery
   ├── Before returning to user mode
   ├── Kernel checks pending signals
   ├── Determines action:
   │   ├── Ignore (SIG_IGN)
   │   ├── Default action
   │   └── Custom handler
   └── Action taken

4. Default Action for Termination Signals
   ├── Kernel initiates termination
   ├── Process state set to zombie
   ├── Exit status set (signal info)
   ├── Parent notified
   └── Process terminates

Signal Number in Exit Status:
├── Parent can detect signal death
├── WIFSIGNALED(status) macro
├── WTERMSIG(status) returns signal
├── Status indicates signal number
└── Parent can distinguish from normal exit

Example:
int status;
wait(&status);

if (WIFEXITED(status)) {
    printf("Normal exit with %d\n",
           WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("Killed by signal %d\n",
           WTERMSIG(status));
}
```

### Core Dumps

Certain signals produce core dumps:

```
Core Dumps:

Signals That Core Dump by Default:
├── SIGQUIT (3)
├── SIGILL (4)
├── SIGABRT (6)
├── SIGFPE (8)
├── SIGSEGV (11)
├── SIGBUS (7)
└── Various others

Core Dump Contents:
├── Memory image of process
├── Register values
├── Signal information
├── Process metadata
└── Debugging information

Where Core Dumps Go:
├── Traditionally: ./core
├── Modern Linux: /var/lib/systemd/coredump/
├── Depends on core_pattern
├── Can be piped to program
└── Often disabled by default

Controlling Core Dumps:
$ ulimit -c unlimited    # Enable
$ ulimit -c 0            # Disable

/proc/sys/kernel/core_pattern:
$ cat /proc/sys/kernel/core_pattern
|/usr/lib/systemd/systemd-coredump %P %u %g ...

Analyzing Core Dumps:
$ gdb ./program core
(gdb) bt                 # Backtrace
(gdb) info registers     # Registers
(gdb) info locals        # Local variables
(gdb) quit

Purpose:
├── Debugging crashes
├── Post-mortem analysis
├── Identify bugs
├── Support diagnostics
└── Rarely in production
```

### raise() and kill()

Programs can send signals to themselves or others:

```
Sending Signals:

raise(int sig):
├── Send signal to self
├── Equivalent to: kill(getpid(), sig)
├── Simple interface
├── Used for self-termination
└── Example: raise(SIGTERM)

kill(pid_t pid, int sig):
├── Send signal to process
├── pid > 0: process with that PID
├── pid == 0: current process group
├── pid == -1: all processes (privileged)
├── pid < -1: process group |pid|
├── Returns 0 on success
├── Returns -1 on error
└── Example: kill(1234, SIGTERM)

abort():
├── Raise SIGABRT
├── Terminates process
├── Generates core dump
├── Used for assertions
├── Cannot be caught (usually)
└── Abrupt termination

assert() Example:
#include <assert.h>

int divide(int a, int b) {
    assert(b != 0);  // If fails: abort()
    return a / b;
}

If assertion fails:
divide: divide.c:4: divide: Assertion `b != 0' failed.
Aborted (core dumped)

kill Examples:
# Send SIGTERM to process 1234
$ kill 1234
$ kill -TERM 1234
$ kill -15 1234

# Send SIGKILL (force)
$ kill -9 1234
$ kill -KILL 1234

# Send to all processes (privileged)
$ kill -TERM -1

# List all signals
$ kill -l
```

### The abort() Function

Immediate termination with a core dump:

```
abort() Function:

Signature:
#include <stdlib.h>
_Noreturn void abort(void);

Behavior:
1. Unblock SIGABRT
2. Raise SIGABRT
3. If signal caught and handler returns:
   ├── abort() continues anyway
   ├── Re-raises SIGABRT
   ├── This time with default action
   ├── Process terminates
   └── Core dump generated
4. If signal not caught:
   ├── Default action
   ├── Terminate with core dump
   └── Process ends

Guarantees:
├── Terminates process
├── Cannot be prevented
├── Signal handler can clean up
├── But cannot prevent termination
├── Core dump usually
└── Noreturn attribute

Use Cases:
├── Assertion failures
├── Fatal program errors
├── Unrecoverable state
├── Programming errors
├── Defensive programming
└── Debugging aid

Not Recommended For:
├── Recoverable errors
├── User input errors
├── Resource limitations
├── Normal error paths
└── Production error handling

Best Practice:
├── Use abort() for bugs
├── Use exit() for expected errors
├── Use proper error handling
├── Only abort when truly broken
└── Document abort conditions
```

---

## Process Cleanup

### What Happens During Termination

```
Termination Cleanup Sequence:

1. Kernel Gains Control
   ├── exit() called or signal arrives
   ├── Kernel takes over
   ├── User code no longer runs
   └── Kernel begins cleanup

2. Release User Resources
   ├── Close file descriptors
   ├── Release memory mappings
   ├── Free physical memory
   ├── Close network sockets
   ├── Release IPC resources
   └── Clean up various objects

3. Update Process State
   ├── Set state to EXIT_ZOMBIE
   ├── Store exit status
   ├── Store exit signal (if any)
   ├── Save accounting information
   ├── Mark as terminated
   └── Update timestamps

4. Notify Parent
   ├── Send SIGCHLD to parent
   ├── Wake parent if sleeping
   ├── Parent can call wait()
   ├── Parent can ignore
   └── Parent can handle signal

5. Reparent Children
   ├── If process has children
   ├── Children become orphans
   ├── Reparented to init (PID 1)
   ├── Or to nearest subreaper
   ├── Kernel updates PCBs
   └── Init will reap them

6. Release Most Resources
   ├── Memory freed
   ├── Files closed
   ├── IPC cleaned up
   ├── Locks released
   ├── Timers removed
   └── Most kernel objects freed

7. Retain PCB (Zombie)
   ├── PCB not freed yet
   ├── Exit status retained
   ├── Waiting for parent
   ├── Minimal resource usage
   └── Zombie state

8. Parent Reaps
   ├── Parent calls wait()
   ├── Exit status returned
   ├── PCB freed
   ├── Removed from process table
   ├── PID released
   └── Process completely gone

Timeline:
Process exits
    │
    ▼
[Cleanup most resources]
    │
    ▼
[Zombie - waiting for parent]
    │
    ▼
Parent calls wait()
    │
    ▼
[PCB freed, PID released]
    │
    ▼
Process gone
```

### Resource Release Details

```
Resources Released on Termination:

Memory:
├── User address space freed
├── Pages returned to free pool
├── Page tables freed
├── Kernel stack freed
├── Various kernel objects
└── All memory returned

Files:
├── File descriptors closed
├── Reference counts decremented
├── Files closed if last reference
├── File offsets discarded
├── Locks released
└── Open file table updated

IPC:
├── Message queues detached
├── Shared memory detached
├── Semaphores released
├── Pipes closed
├── Sockets closed
└── IPC objects cleaned

Signals:
├── Signal handlers cleared
├── Pending signals discarded
├── Signal masks irrelevant
├── Timers removed
└── Signal queue cleared

Scheduling:
├── Removed from run queue
├── Removed from wait queue
├── Priority info cleared
├── CPU time accounted
├── Context switches counted
└── Scheduler updated

Other:
├── File locks released
├── Semaphores released
├── Shared resources disconnected
├── Accounting finalized
├── Audit logs updated
└── Security context cleared
```

---

## Zombie Processes

### What is a Zombie?

A zombie is a terminated process that hasn't been reaped:

```
Zombie Process:

Definition:
├── Process that has terminated
├── But parent hasn't called wait()
├── PCB retained in kernel
├── Exit status stored
├── No other resources
└── Waiting for parent

Why Zombies Exist:
├── Kernel must keep exit status
├── Parent needs to read it
├── wait() returns status
├── Then PCB is freed
├── Historical design choice
├── Standard Unix behavior
└── Trade-off: resource vs. info

What a Zombie Retains:
├── PID (still allocated)
├── PCB (kernel memory)
├── Exit status
├── Resource usage stats
├── Parent pointer
└── Minimal information

What a Zombie Does NOT Have:
├── No user memory
├── No file descriptors
├── No kernel stack
├── No code or data
├── No execution
├── No scheduling
└── Just metadata
```

### Creating and Observing Zombies

```
Creating a Zombie:

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child exits immediately
        printf("Child exiting\n");
        exit(0);
    } else {
        // Parent doesn't wait
        printf("Parent sleeping (child is zombie)\n");
        sleep(60);  // Child becomes zombie
        printf("Parent exiting\n");
        // Child reaped by init after parent exits
    }
    return 0;
}

Output:
Child exiting
Parent sleeping (child is zombie)

Observing:
$ ps aux | grep defunct
user  1234  0.0  0.0  0  0 ?  Z  10:30  0:00 [myprog] <defunct>

$ ps aux | grep -w Z
# Shows all zombie processes

$ ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'

STAT column:
├── Z indicates zombie
├── <defunct> in command
├── [name] in brackets
└── No resources shown
```

### Zombie Problems and Solutions

```
Zombie Problems:

1. PID Exhaustion
   ├── Each zombie holds a PID
   ├── PIDs are limited (e.g., 32768)
   ├── Thousands of zombies
   ├── Cannot create new processes
   └── System malfunction

2. Kernel Memory
   ├── Each zombie has a PCB
   ├── PCBs use kernel memory
   ├── Many zombies = wasted memory
   ├── Usually small (few KB)
   └── Can add up

3. System Monitoring Confusion
   ├── Zombies appear in process lists
   ├── Count toward process totals
   ├── Confuse monitoring
   ├── Alert on high counts
   └── Operational overhead

Solutions:

1. Parent Calls wait()
   ├── Standard solution
   ├── Reaps zombies
   ├── Frees resources
   ├── Returns exit status
   └── Best practice

2. Parent Calls waitpid()
   ├── More specific
   ├── Wait for specific child
   ├── Non-blocking option (WNOHANG)
   ├── More control
   └── Flexible

3. Signal Handler for SIGCHLD
   ├── Handle child termination
   ├── Call wait() in handler
   ├── Reap zombies automatically
   ├── Beware of race conditions
   └── Common pattern

4. Set SIGCHLD to SIG_IGN
   ├── Kernel auto-reaps
   ├── Child immediately cleaned up
   ├── No zombies created
   ├── Linux-specific
   └── Not portable

5. Double Fork Technique
   ├── Fork twice
   ├── Middle process exits
   ├── Grandchild reparented to init
   ├── Init reaps automatically
   └── Used by daemons

6. Kill Parent
   ├── If parent is buggy
   ├── Children reparented to init
   ├── Init reaps them
   ├── Fixes zombies
   └── Last resort
```

### Complete Zombie Handling Example

```
Proper Zombie Handling:

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <signal.h>

// Signal handler for SIGCHLD
void sigchld_handler(int sig) {
    (void)sig;  // Unused parameter
    int status;
    pid_t pid;
    
    // Reap all terminated children
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status)) {
            printf("Child %d exited with %d\n",
                   pid, WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Child %d killed by signal %d\n",
                   pid, WTERMSIG(status));
        }
    }
}

int main() {
    // Install SIGCHLD handler
    struct sigaction sa;
    sa.sa_handler = sigchld_handler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = SA_RESTART | SA_NOCLDSTOP;
    
    if (sigaction(SIGCHLD, &sa, NULL) == -1) {
        perror("sigaction");
        return 1;
    }
    
    // Create several children
    for (int i = 0; i < 5; i++) {
        pid_t pid = fork();
        
        if (pid == 0) {
            // Child
            printf("Child %d (PID %d) starting\n",
                   i, getpid());
            sleep(i + 1);
            printf("Child %d (PID %d) exiting\n",
                   i, getpid());
            exit(i);
        } else if (pid < 0) {
            perror("fork");
            return 1;
        }
        // Parent doesn't wait here
    }
    
    // Parent continues
    printf("Parent (PID %d) waiting for children...\n",
           getpid());
    
    // Do other work while children run
    sleep(10);
    
    printf("Parent exiting\n");
    return 0;
}

Key Points:
├── SIGCHLD handler called on child exit
├── Handler reaps child immediately
├── No zombies accumulate
├── WNOHANG prevents blocking
├── Loop handles multiple children
├── SA_RESTART restarts syscalls
├── SA_NOCLDSTOP ignores stop signals
└── Clean and correct
```

---

## Orphan Processes

### What is an Orphan?

An orphan is a process whose parent has died:

```
Orphan Process:

Definition:
├── Process whose parent terminated
├── Still running
├── Reparented to init (PID 1)
├── Or to subreaper
├── Continues execution
└── Init will reap eventually

Why Orphans Occur:
├── Parent exits before child
├── Parent killed
├── Parent doesn't wait
├── Intentional orphaning
└── Daemon creation

Handling:
├── Automatic reparenting
├── Kernel does this
├── Init becomes new parent
├── Init reaps when done
├── No explicit action needed
└── Standard Unix behavior

Example:
$ nohup long_job &
$ exit
├── Shell forks child
├── Child runs long_job
├── Shell exits
├── Child becomes orphan
├── Reparented to init
└── Continues running

Comparison with Zombie:
├── Zombie: terminated, waiting for parent
├── Orphan: running, parent has died
├── Different states
├── Different handling
└── Both handled by kernel
```

### Creating Orphans

```
Orphan Creation Example:

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child
        printf("Child PID: %d, PPID: %d\n",
               getpid(), getppid());
        sleep(2);  // Wait for parent to exit
        printf("Child PID: %d, PPID: %d\n",
               getpid(), getppid());
        printf("Child is now an orphan\n");
        exit(0);
    } else {
        // Parent
        printf("Parent PID: %d\n", getpid());
        printf("Parent exiting immediately\n");
        exit(0);
        // Child continues as orphan
    }
}

Output:
Parent PID: 1000
Parent exiting immediately
Child PID: 1001, PPID: 1000
(After 2 seconds)
Child PID: 1001, PPID: 1     # Reparented to init
Child is now an orphan

Observations:
├── Child's PPID changes from 1000 to 1
├── Reparenting is automatic
├── Child continues running
├── Init is now the parent
└── Init will reap when child exits
```

### Daemon Creation

The classic use of orphaning is creating daemons:

```
Daemon Creation (Traditional):

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>

void daemonize(void) {
    pid_t pid;
    
    // 1. Fork and exit parent
    pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(1);
    }
    if (pid > 0) {
        exit(0);  // Parent exits
    }
    // Child continues as orphan
    
    // 2. Create new session
    if (setsid() < 0) {
        perror("setsid");
        exit(1);
    }
    // Now session leader, no controlling terminal
    
    // 3. Fork again and exit parent
    pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(1);
    }
    if (pid > 0) {
        exit(0);  // First child exits
    }
    // Grandchild continues (not session leader)
    // Prevents acquiring controlling terminal
    
    // 4. Change working directory
    if (chdir("/") < 0) {
        perror("chdir");
        exit(1);
    }
    // Release current directory
    
    // 5. Redirect stdin, stdout, stderr to /dev/null
    int fd = open("/dev/null", O_RDWR);
    if (fd < 0) {
        perror("open");
        exit(1);
    }
    dup2(fd, STDIN_FILENO);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    if (fd > STDERR_FILENO) {
        close(fd);
    }
    // Daemon has no terminal output
    
    // 6. Set file mode creation mask
    umask(0);
    // Or specific mask like 022
    
    // 7. Daemon is now running
    // Continue with daemon work...
}

int main() {
    daemonize();
    
    // This is now a daemon
    while (1) {
        // Do work
        sleep(60);
    }
    return 0;
}

Why Double Fork:
├── First fork: Parent exits, child orphaned
├── setsid(): Child becomes session leader
├── Second fork: Session leader exits
├── Grandchild not session leader
├── Cannot acquire controlling terminal
├── Truly detached from terminal
└── Standard daemon pattern

Modern Alternative:
├── systemd handles daemonization
├── Use Type=simple in service file
├── No double fork needed
├── Simpler and cleaner
└── Recommended in systemd systems
```

---

## wait() and waitpid()

### The wait() System Call

Parent processes use wait() to reap children:

```
wait() Function:

Signature:
#include <sys/wait.h>
pid_t wait(int *status);

Behavior:
├── Blocks until a child terminates
├── Or until a child is stopped/continued
├── Returns child's PID
├── Stores status in *status
├── Reaps the zombie child
├── Returns -1 on error (no children)

When Called:
├── Child terminates
├── wait() returns immediately
├── Child's exit status available
├── Zombie child reaped
└── PCB of child freed

If Called Before Child Exits:
├── Parent blocks
├── Parent enters WAITING state
├── Other processes run
├── Child eventually exits
├── Parent wakes up
└── wait() returns

If Multiple Children:
├── wait() returns on first child exit
├── Which child? Any terminated child
├── Non-deterministic
├── Use waitpid() for specific
└── Loop for all children

Status Interpretation:
├── WIFEXITED(status): normal exit?
├── WEXITSTATUS(status): exit code
├── WIFSIGNALED(status): killed?
├── WTERMSIG(status): signal number
├── WIFSTOPPED(status): stopped?
├── WSTOPSIG(status): stop signal
├── WCOREDUMP(status): core dumped?
└── Various macros
```

### The waitpid() System Call

More flexible than wait():

```
waitpid() Function:

Signature:
#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *status, int options);

pid Argument:
├── pid > 0: Wait for specific child
├── pid == 0: Wait for child in same process group
├── pid == -1: Wait for any child (like wait())
├── pid < -1: Wait for child in group |pid|
└── More specific control

Options:
├── 0: Block until child terminates
├── WNOHANG: Don't block, return immediately
├── WUNTRACED: Also report stopped children
├── WCONTINUED: Also report continued children
└── Combination of options

Return Value:
├── > 0: PID of terminated child
├── 0: WNOHANG and no child ready
├── -1: Error (no child, invalid, etc.)
└── errno set on error

Advantages over wait():
├── Can wait for specific child
├── Can wait for process group
├── Can be non-blocking (WNOHANG)
├── More control
├── More flexible
└── Preferred in modern code

Example:
#include <sys/wait.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main() {
    pid_t children[3];
    
    // Create 3 children
    for (int i = 0; i < 3; i++) {
        children[i] = fork();
        if (children[i] == 0) {
            sleep(i + 1);
            exit(i);
        }
    }
    
    // Wait for each specifically
    for (int i = 0; i < 3; i++) {
        int status;
        pid_t pid = waitpid(children[i], &status, 0);
        
        if (WIFEXITED(status)) {
            printf("Child %d exited with %d\n",
                   pid, WEXITSTATUS(status));
        }
    }
    return 0;
}

Output:
Child 1 exited with 0
Child 2 exited with 1
Child 3 exited with 2
```

### Non-Blocking Wait

WNOHANG allows polling without blocking:

```
Non-Blocking Wait:

Example:
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid == 0) {
        sleep(5);
        exit(42);
    }
    
    // Poll for child without blocking
    int status;
    pid_t result;
    
    while ((result = waitpid(pid, &status, WNOHANG)) == 0) {
        printf("Child still running...\n");
        sleep(1);
    }
    
    if (result == pid) {
        printf("Child exited with %d\n",
               WEXITSTATUS(status));
    }
    return 0;
}

Output:
Child still running...
Child still running...
Child still running...
Child still running...
Child still running...
Child exited with 42

Uses:
├── Periodic check
├── Do other work while waiting
├── Event loop integration
├── Avoid blocking
└── Polling pattern
```

### Best Practices

```
wait() Best Practices:

1. Always Wait for Children
   ├── Prevents zombies
   ├── Releases resources
   ├── Gets exit status
   ├── Good hygiene
   └── Essential in servers

2. Handle EINTR
   ├── Wait can be interrupted
   ├── Signal during wait
   ├── Returns -1, errno=EINTR
   ├── Retry the call
   └── Or handle appropriately

3. Check Return Value
   ├── Always check result
   ├── -1 indicates error
   ├── errno tells why
   ├── Handle errors properly
   └── Don't ignore

4. Use waitpid for Specific
   ├── Know which child
   ├── Better control
   ├── Preferred in most code
   ├── More flexible
   └── Modern approach

5. Handle Multiple Children
   ├── Loop until all reaped
   ├── WNOHANG for non-blocking
   ├── Handle in signal handler
   ├── Or synchronously
   └── Depends on design

6. Signal Handler Approach
   ├── SIGCHLD handler
   ├── Reap children there
   ├── Use WNOHANG to avoid blocking
   ├── Loop for multiple children
   └── Careful with race conditions

Example - Robust waitpid:
int status;
pid_t pid;

while ((pid = waitpid(-1, &status, 0)) > 0) {
    if (WIFEXITED(status)) {
        // Normal exit
    } else if (WIFSIGNALED(status)) {
        // Killed by signal
    }
}
if (pid < 0 && errno != ECHILD) {
    // Error other than "no children"
    perror("waitpid");
}
```

---

## Process Termination in Different Systems

### Unix/Linux

```
Unix/Linux Termination:

Signals:
├── Rich signal mechanism
├── SIGTERM, SIGKILL, etc.
├── Catching signals possible
├── Default actions defined
└── Standard across Unix

Exit:
├── exit() and _exit()
├── Status 0-255
├── Parent reads via wait()
├── Zombie until reaped
└── Well-defined behavior

Commands:
$ kill PID              # SIGTERM
$ kill -9 PID           # SIGKILL
$ killall name          # All by name
$ pkill pattern         # By pattern
$ pkill -9 name         # Force kill
$ kill -l               # List signals

Viewing:
$ ps aux                # Show processes
$ ps -ef                # Show PPID
$ ps aux | grep Z       # Find zombies
$ ps -eo pid,ppid,stat,cmd  # Custom format

/proc:
$ cat /proc/PID/status  # Process status
$ cat /proc/PID/stat    # Process stats
$ ls /proc/PID/fd/      # File descriptors
```

### Windows

```
Windows Termination:

ExitProcess():
├── Windows equivalent of exit()
├── Terminates process
├── Calls DLL cleanup
├── Closes handles
├── Sets exit code
└── Notifies waiters

TerminateProcess():
├── Forcible termination
├── Like SIGKILL
├── Cannot be prevented
├── Abrupt
├── May leave resources in bad state
└── Last resort

Exit Codes:
├── DWORD (32-bit)
├── Set by ExitProcess or return
├── Read by WaitForSingleObject
├── GetExitCodeProcess
└── Not limited to 8 bits

Waiting for Termination:
WaitForSingleObject(hProcess, INFINITE);
DWORD exitCode;
GetExitCodeProcess(hProcess, &exitCode);

Tools:
├── Task Manager
├── Process Explorer
├── tasklist
├── taskkill
├── PowerShell Stop-Process
└── Various other tools

Differences from Unix:
├── Different API
├── Different signals (events, not signals)
├── No zombies (different model)
├── Handles instead of PIDs
├── Different security model
└── Different tools
```

### macOS

```
macOS Termination:

Unix-like:
├── BSD heritage
├── Standard Unix signals
├── exit() and _exit()
├── Zombies handled similarly
├── wait() and waitpid()
└── Familiar to Unix users

Additional Features:
├── launchd manages processes
├── Different from systemd/init
├── Application lifecycle
├── Cocoa termination
├── Graceful shutdown
└── Specific to macOS

killall:
$ killall Safari        # Terminate all
$ killall -9 app        # Force kill

Activity Monitor:
├── GUI process viewer
├── Shows processes
├── Can terminate processes
├── Shows resource usage
└── macOS-specific
```

---

## Common Misconceptions

**Misconception 1:** "Terminated processes disappear immediately"  
**Reality:** Terminated processes become zombies until the parent reaps them via wait(). The PCB remains until then. Only after wait() is the process completely removed.

**Misconception 2:** "SIGKILL is immediate"  
**Reality:** SIGKILL can't be caught or ignored, but termination still involves some kernel processing. Also, processes in uninterruptible sleep (D state) won't terminate until they leave D state.

**Misconception 3:** "You can kill any process with SIGKILL"  
**Reality:** SIGKILL doesn't work on zombie processes (already terminated) or on processes stuck in uninterruptible kernel operations. Root can kill most processes, but even root can't kill zombies.

**Misconception 4:** "Zombies consume significant resources"  
**Reality:** Zombies consume only a PCB (typically a few KB of kernel memory) and a PID. They don't use CPU, don't hold user memory, and don't have file descriptors. Their main risk is PID exhaustion if thousands accumulate.

**Misconception 5:** "exit() and _exit() are the same"  
**Reality:** exit() performs cleanup (atexit handlers, stdio flushing, temp file cleanup) while _exit() does not. _exit() is a system call; exit() is a library function that calls _exit(). Use _exit() in child processes after fork() and in signal handlers.

**Misconception 6:** "All signals terminate processes"  
**Reality:** Only some signals terminate by default. Others stop, continue, ignore, or have other default actions. SIGCHLD, for example, is ignored by default. SIGSTOP stops but doesn't terminate.

**Misconception 7:** "Orphans are problematic"  
**Reality:** Orphans are handled automatically by reparenting to init. They continue running normally and are cleaned up when they terminate. Orphans are a normal part of Unix process management.

---

## Summary

Process termination is how processes end their lifecycle. Processes can terminate normally (return from main, exit()), abnormally (via signals like SIGSEGV or SIGKILL), or be killed by other processes. During termination, resources are released, the parent is notified, and the process becomes a zombie until reaped. Orphaned processes are reparented to init.

### Key Points

1. Multiple termination ways: exit(), signals, fatal errors.
2. exit() vs. _exit(): cleanup vs. immediate.
3. Signals terminate abnormally: SIGTERM, SIGKILL, etc.
4. Zombies: terminated but unreaped processes.
5. Orphans: reparented to init when parent dies.
6. wait() and waitpid(): reap zombies, get status.
7. SIGCHLD: notification of child termination.
8. Proper cleanup: essential for resource management.

---

## Key Takeaways

1. All processes end — normal or abnormal termination.
2. exit() does cleanup, _exit() doesn't.
3. Signals cause most abnormal terminations.
4. Zombies must be reaped — parent responsibility.
5. Orphans are handled automatically.
6. wait() gets status and reaps.
7. SIGCHLD handling is essential for servers.
8. Proper termination prevents resource leaks.

---

## What's Next?

Continue to Context Switching to learn how the CPU switches between processes, saving and restoring execution state.

---

## Further Reading

**Books**

- "Operating System Concepts" by Silberschatz, Galvin, and Gagne
- "Modern Operating Systems" by Andrew S. Tanenbaum
- "Operating Systems: Three Easy Pieces" by Remzi and Andrea Arpaci-Dusseau
- "The Linux Programming Interface" by Michael Kerrisk
- "Linux Kernel Development" by Robert Love
- "Advanced Programming in the UNIX Environment" by Stevens and Rago

**Online Resources**

- OSTEP (ostep.org) — Free textbook
- man pages: exit(3), _exit(2), wait(2), waitpid(2), signal(7)
- Linux Kernel Documentation (kernel.org)
- "The Linux Programming Interface" — Chapters 24-26
- signal(7) — Complete signal reference

