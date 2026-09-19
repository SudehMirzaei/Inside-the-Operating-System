# Privilege Levels

## Introduction

Privilege levels are the hardware-enforced mechanisms that make the kernel/user space separation possible. Without them, the distinction between privileged kernel code and unprivileged user code would be merely a convention—easily violated by buggy or malicious programs. The CPU itself enforces these levels, refusing to execute certain instructions or access certain memory when running in an unprivileged state.

This document explores privilege levels in depth: what they are, how they work at the hardware level, how operating systems use them, and how they form the foundation of system security.

---

## The Fundamental Concept

### What Are Privilege Levels?

Privilege levels define what code is allowed to do. The CPU operates at different privilege levels depending on what code is currently executing. Higher privilege means more capabilities; lower privilege means more restrictions.

```
The Privilege Spectrum:

Highest Privilege (Most Capable, Most Trusted)
    │
    │  Ring 0: Kernel
    │  ├── Can execute all instructions
    │  ├── Can access all memory
    │  ├── Can control hardware
    │  └── Full system control
    │
    │  Ring 1-2: (Rarely used)
    │  ├── Historically for device drivers
    │  └── Seldom implemented today
    │
    │  Ring 3: User Applications
    │  ├── Limited instructions
    │  ├── Restricted memory access
    │  ├── Cannot control hardware
    │  └── Must use system calls
    │
Lowest Privilege (Least Capable, Least Trusted)
```

### Why Privilege Levels Matter

```
Without Hardware Privilege Levels:

Scenario: Malicious Program
├── Program wants to read kernel memory
├── No hardware restriction
├── Program reads passwords directly
├── Program modifies kernel code
├── System completely compromised
└── No way to prevent this

Scenario: Buggy Program
├── Program has pointer bug
├── Writes to random memory address
├── Address happens to be kernel data
├── Kernel crashes
├── System halts
└── All work lost

With Hardware Privilege Levels:

Scenario: Malicious Program
├── Program tries to read kernel memory
├── CPU checks privilege level
├── Access denied (user mode)
├── CPU raises exception
├── Program terminated
└── System protected

Scenario: Buggy Program
├── Program has pointer bug
├── Writes to random address
├── Address is kernel memory
├── CPU raises exception
├── Program terminated (segfault)
└── Kernel unaffected
```

### The Analogy

Privilege levels are like security clearances in a government facility:

- **Ring 0 (Kernel)**: Top secret clearance — access to everything
- **Ring 1-2 (Rarely Used)**: Secret clearance — limited access
- **Ring 3 (User)**: Public clearance — access to public areas only

Just as a visitor cannot enter a secure area regardless of what they claim, user code cannot perform privileged operations regardless of what it tries.

---

## The x86 Protection Ring Model

### The Four Rings

The x86 architecture defines four privilege levels, called rings:

```
x86 Protection Rings:

                    ┌─────────────────────────┐
                    │                         │
                    │      Ring 0             │
                    │      (Kernel)           │
                    │  ┌───────────────────┐  │
                    │  │    Ring 1         │  │
                    │  │  ┌─────────────┐  │  │
                    │  │  │   Ring 2    │  │  │
                    │  │  │ ┌─────────┐ │  │  │
                    │  │  │ │ Ring 3  │ │  │  │
                    │  │  │ │ (User)  │ │  │  │
                    │  │  │ └─────────┘ │  │  │
                    │  │  └─────────────┘  │  │
                    │  └───────────────────┘  │
                    └─────────────────────────┘

Ring 0 - Highest Privilege:
├── Kernel mode
├── Full hardware access
├── All instructions available
├── All memory accessible
└── Where the OS kernel runs

Ring 1 - High Privilege:
├── Historically for device drivers
├── Rarely used today
├── Some hypervisor functions
└── Mostly unused

Ring 2 - Medium Privilege:
├── Historically for I/O subsystems
├── Rarely used today
├── Some system services
└── Mostly unused

Ring 3 - Lowest Privilege:
├── User mode
├── Limited instructions
├── Restricted memory access
├── Where applications run
└── Most common non-kernel mode
```

### Why Four Rings?

The four-ring design was intended to support a hierarchical protection model:

```
Original Intent (Multics-inspired):

Ring 0: Kernel
├── Core OS functionality
├── Memory management
├── Process scheduling
└── Highest privilege

Ring 1: System Services
├── I/O drivers
├── File systems
├── System utilities
└── High privilege

Ring 2: Extensions
├── User-installed drivers
├── Custom services
├── Extensions
└── Medium privilege

Ring 3: Applications
├── User programs
├── Most software
├── Limited privilege
└── Lowest privilege

Reality:
├── Most OSes use only Ring 0 and Ring 3
├── Rings 1 and 2 unused
├── Complexity vs. benefit trade-off
├── Simpler is better
└── Two-level model dominates

Why Rings 1-2 Are Unused:
├── Modern OS design prefers simplicity
├── Device drivers in kernel (performance)
├── Or user space (reliability)
├── Middle rings add complexity
├── No strong use case
└── Two levels sufficient
```

### Current Privilege Level (CPL)

The CPU tracks the current privilege level in the CS (Code Segment) register:

```
CPL (Current Privilege Level):

Stored in: CS register bits 0-1
Values: 0, 1, 2, 3 (ring number)
Meaning: Current privilege of executing code

Examples:
├── CPL = 0 → Kernel mode
├── CPL = 3 → User mode
├── CPL = 1, 2 → Rarely used

Usage:
├── Every memory access checked against CPL
├── Every instruction checked against CPL
├── Every privileged operation checked
└── Hardware enforces restrictions

CS Register Structure (x86-64):
┌──────────────────────────────────────────────┐
│ 63                             16  2  1  0   │
│ ┌───────────────────────────┐ ┌──┬──┬──┐    │
│ │      Base (ignored)       │ │  │R │CPL│   │
│ └───────────────────────────┘ └──┴──┴──┘    │
│                                    │   │     │
│                                    │   └── RPL (Requested Privilege)
│                                    └────── CPL (Current Privilege)
└──────────────────────────────────────────────┘
```

### Descriptor Privilege Level (DPL)

Memory segments and gates have a DPL that specifies the minimum privilege required to access them:

```
DPL (Descriptor Privilege Level):

Stored in: Segment descriptors, gate descriptors
Values: 0, 1, 2, 3
Meaning: Required privilege to access

Access Rules:
├── Data segment: CPL ≤ DPL (higher or equal privilege)
├── Code segment: CPL = DPL (exact match usually)
├── Call gate: CPL ≤ DPL (for privilege increase)
└── Interrupt gate: CPL any (controlled entry)

Example:
├── Kernel code segment: DPL = 0
│   └── Only accessible when CPL = 0
├── User code segment: DPL = 3
│   └── Only accessible when CPL = 3
├── Kernel data segment: DPL = 0
│   └── Only accessible when CPL = 0
└── User data segment: DPL = 3
    └── Accessible when CPL ≤ 3 (always)

Access Check:
if (CPL <= DPL) {
    // Access allowed
} else {
    // Access denied
    // Raise #GP (General Protection Fault)
}
```

### RPL (Requested Privilege Level)

The RPL allows code to specify a lower privilege for a memory access:

```
RPL (Requested Privilege Level):

Stored in: Segment selector (bits 0-1)
Purpose: Prevent privilege escalation attacks

Example Attack Without RPL:
├── User mode (CPL=3)
├── Calls kernel via system call
├── Passes pointer to kernel data
├── Kernel accesses via pointer
├── Uses user's segment selector
├── Effective privilege = user (CPL=3)
├── But kernel could access kernel data
├── User effectively reads kernel memory
└── Security vulnerability!

Solution: RPL
├── Kernel sets RPL = 3 for user pointers
├── Effective privilege = min(CPL, RPL)
├── For user pointer: min(0, 3) = 0... wait
├── Actually: max(CPL, RPL) for access check
├── For kernel accessing user data: max(0, 3) = 3
├── Therefore access checked at level 3
└── Kernel data (DPL=0) denied

Effective Privilege Level:
EPL = max(CPL, RPL)
```

---

## Privileged Instructions

Certain CPU instructions can only execute at Ring 0:

### Categories of Privileged Instructions

```
Privileged Instructions (Partial List):

1. Memory Management
   ├── MOV to/from CR0, CR2, CR3, CR4
   ├── MOV to/from DR0-DR7
   ├── LGDT, SGDT (Global Descriptor Table)
   ├── LIDT, SIDT (Interrupt Descriptor Table)
   ├── LLDT, SLDT (Local Descriptor Table)
   ├── LTR, STR (Task Register)
   ├── INVLPG (Invalidate TLB entry)
   └── INVPCID (Invalidate PCID)

2. Interrupt Control
   ├── CLI (Clear Interrupt Flag)
   ├── STI (Set Interrupt Flag)
   ├── LIDT (Load IDT)
   └── Related interrupt management

3. I/O Operations
   ├── IN (read from I/O port)
   ├── OUT (write to I/O port)
   ├── INS, OUTS (string I/O)
   └── (Can be relaxed with IOPL)

4. CPU Control
   ├── HLT (Halt CPU)
   ├── WBINVD (Writeback and invalidate cache)
   ├── INVD (Invalidate cache)
   ├── RDMSR, WRMSR (Model-Specific Registers)
   ├── RDPMC (Performance counters, usually)
   ├── SWAPGS (Kernel GS base)
   ├── SYSEXIT, SYSRET (System call return)
   └── Various CPU state instructions

5. Protection
   ├── LMSW, SMSW (Machine Status Word)
   ├── LAR, LSL (Segment access rights)
   ├── VERR, VERW (Verify segment)
   └── Related protection instructions

6. Virtualization
   ├── VMXON, VMXOFF
   ├── VMLAUNCH, VMRESUME
   ├── VMCALL, VMFUNC
   └── Various VMX instructions

7. Other
   ├── WRMSR (Write MSR)
   ├── RDMSR (Read MSR)
   ├── CLTS (Clear Task Switched flag)
   └── Many more
```

### What Happens When User Code Executes a Privileged Instruction

```
Privileged Instruction Violation:

1. User code (CPL=3) executes privileged instruction
2. CPU checks instruction's privilege requirement
3. Requirement: CPL=0 (or specific level)
4. Current level: CPL=3
5. Check fails
6. CPU raises #GP (General Protection Fault)
7. Control transfers to kernel exception handler
8. Kernel handles fault
9. Usually: terminate process with SIGSEGV or SIGILL

Example:
User program tries: 
    cli    ; disable interrupts

Execution:
├── CPU decodes CLI instruction
├── Checks CPL
├── CPL=3 (user mode)
├── CLI requires CPL=0 (IOPL check actually)
├── Check fails
├── #GP raised (or #UD)
├── Kernel handler called
├── Kernel sees user violated privilege
├── Sends SIGSEGV to process
├── Process terminated
└── User sees "Segmentation fault"

Note: CLI actually uses IOPL, not CPL
      So behavior depends on IOPL setting
      But the principle is the same
```

### The IOPL (I/O Privilege Level)

The IOPL is a special case—it allows controlled access to I/O instructions:

```
IOPL (I/O Privilege Level):

Stored in: RFLAGS register (bits 12-13)
Values: 0, 1, 2, 3
Meaning: Privilege required for I/O instructions

Behavior:
├── If CPL ≤ IOPL: I/O instructions allowed
├── If CPL > IOPL: I/O instructions fault
├── Default: IOPL = 0 (only kernel)
├── Can be set to 3 (allow user I/O)
└── Per-task setting (usually)

Uses:
├── Kernel: IOPL = 0 (secure)
├── Virtual 8086 mode: IOPL = 3
├── Some drivers: IOPL = 3
├── DOS emulation
└── Legacy support

RFLAGS Register:
┌──────────────────────────────────────────────┐
│  Bit 12-13: IOPL                             │
│  Bit 9: IF (Interrupt Flag)                  │
│  Bit 8: TF (Trap Flag)                       │
│  Bit 7: SF (Sign Flag)                       │
│  ... and many others ...                     │
└──────────────────────────────────────────────┘

Example:
Task with IOPL = 3 (user mode, CPL=3):
├── CPL = 3, IOPL = 3
├── CPL ≤ IOPL → I/O allowed
├── Can execute IN/OUT
├── Can disable interrupts
└── Useful for some drivers

Task with IOPL = 0 (normal user):
├── CPL = 3, IOPL = 0
├── CPL > IOPL → I/O not allowed
├── IN/OUT causes #GP
├── Cannot disable interrupts
└── Standard user protection
```

---

## Memory Protection

Privilege levels enforce memory protection through page tables:

### Page Table Permissions

Every memory page has permission bits in the page table:

```
Page Table Entry (x86-64):

┌──────────────────────────────────────────────────────┐
│  Bit 63: NX (No Execute)                            │
│  ...                                                  │
│  Bit 8: G (Global)                                  │
│  Bit 7: PS (Page Size)                              │
│  Bit 6: D (Dirty)                                   │
│  Bit 5: A (Accessed)                                │
│  Bit 4: PCD (Cache Disable)                         │
│  Bit 3: PWT (Write-Through)                         │
│  Bit 2: U/S (User/Supervisor)                       │
│  Bit 1: R/W (Read/Write)                            │
│  Bit 0: P (Present)                                 │
├──────────────────────────────────────────────────────┤
│  Bits 51-12: Physical address                        │
└──────────────────────────────────────────────────────┘

Key Bits for Protection:

- **U/S (User/Supervisor) - Bit 2:**
  - 0 = Supervisor only (kernel)
  - 1 = User accessible
  - Checked against CPL
  - Fundamental isolation

- **R/W (Read/Write) - Bit 1:**
  - 0 = Read only
  - 1 = Read/write
  - Checked on writes

- **P (Present) - Bit 0:**
  - 0 = Page not in memory
  - 1 = Page in memory
  - Fault if not present
  - Triggers page fault

- **NX (No Execute) - Bit 63:**
  - 0 = Executable
  - 1 = Not executable
  - Prevents code execution
  - Security feature (DEP)
```

### Access Control Rules

```
Memory Access Rules:

For any memory access:
1. Determine effective privilege (max of CPL, RPL)
2. Look up page table entry
3. Check U/S bit
4. Check R/W bit
5. Check P bit
6. Check NX (for instruction fetch)

Rules for U/S bit:
├── U/S = 0 (supervisor page)
│   ├── CPL ≤ 1? Access allowed
│   ├── CPL > 1? Access denied (#PF)
│   └── Only kernel can access
│
└── U/S = 1 (user page)
    ├── All CPL allowed
    ├── User can access
    └── Kernel can access

Rules for R/W bit:
├── Access is read: OK if page present
├── Access is write:
│   ├── R/W = 0 (read-only)? Denied (#PF)
│   ├── R/W = 1 (writable)? Allowed
│   └── Checked only for writes

Rules for instruction fetch:
├── NX = 1? Instruction fetch denied (#PF)
├── NX = 0? Fetch allowed
└── Prevents data execution

Combined Checks:
├── Privilege check (CPL vs U/S)
├── Write check (if write, R/W)
├── Execute check (if fetch, NX)
├── Present check (P bit)
└── All must pass for access
```

### Example: User Access to Kernel Memory

```
Scenario: User program reads kernel memory

1. User program (CPL=3) reads address 0xFFFFFFFF80000000
2. CPU walks page tables
3. Finds PTE: U/S=0 (supervisor page)
4. Check: CPL=3 > U/S=0
5. Access denied
6. CPU raises #PF (Page Fault)
7. Kernel's page fault handler runs
8. Handler determines:
   ├── User mode accessing supervisor page
   ├── This is a protection violation
   ├── Not a valid page fault
   └── Security violation
9. Kernel sends SIGSEGV to process
10. Process terminated
11. User sees "Segmentation fault"

Result:
├── Kernel memory protected
├── User program terminated
├── System continues normally
└── No security breach
```

### Protection Example Table

```
Access Matrix (CPL vs Page U/S bit):

              U/S=0 (Supervisor)    U/S=1 (User)
CPL=0         Allowed               Allowed
(Kernel)      (Full access)         (Full access)

CPL=3         Denied                Allowed
(User)        (#PF if accessed)     (Restricted by R/W)

Notes:
├── Kernel can access everything
├── User can only access user pages
├── Read/write permissions separate
├── NX controls instruction fetch
└── Complete isolation maintained

Special Cases:
├── User cannot write read-only pages
├── User cannot execute NX pages
├── Kernel can bypass (careful!)
├── SMAP/SMEP (supervisor mode access prevention)
└── Additional protections for kernel
```

---

## Transitions Between Privilege Levels

### Legitimate Transitions

There are only a few ways to legitimately change privilege level:

```
Legitimate Level Transitions:

1. User → Kernel (Increase Privilege)
   ├── System calls
   │   ├── INT instruction (old)
   │   ├── SYSENTER/SYSCALL (modern)
   │   └── Well-defined entry points
   │
   ├── Interrupts
   │   ├── Hardware interrupt
   │   ├── CPU jumps to handler
   │   └── Handler runs in kernel
   │
   ├── Exceptions
   │   ├── CPU detects fault
   │   ├── Kernel handler runs
   │   └── Handles exception
   │
   └── Controlled by gates/instructions

2. Kernel → User (Decrease Privilege)
   ├── IRET instruction
   │   ├── Return from interrupt
   │   ├── Restores saved state
   │   └── Lowers privilege
   │
   ├── SYSRET/SYSEXIT
   │   ├── Return from system call
   │   ├── Fast path
   │   └── Lowers privilege
   │
   └── Careful state restoration

3. Kernel → Kernel (Same Level)
   ├── Function calls
   ├── Interrupt during kernel execution
   ├── Nested interrupts
   └── No privilege change

4. User → User (Same Level)
   ├── Regular function calls
   ├── No privilege change
   └── Same privilege
```

### System Call Mechanism

The system call instruction performs a controlled privilege increase:

```
System Call (SYSCALL instruction, x86-64):

1. User prepares arguments (registers)
2. User executes SYSCALL instruction
3. CPU performs:
   ├── Saves RIP to RCX
   ├── Saves RFLAGS to R11
   ├── Loads new CS from STAR MSR (CPL 0)
   ├── Loads new RIP from LSTAR MSR
   ├── Loads new SS
   └── Jumps to kernel entry
4. Kernel runs at CPL=0
5. Kernel saves user context
6. Kernel handles system call
7. Kernel executes SYSRET instruction
8. CPU performs:
   ├── Restores CS from STAR MSR (CPL 3)
   ├── Restores RIP from RCX
   ├── Restores RFLAGS from R11
   ├── Restores SS
   └── Returns to user mode
9. User program continues

Key Points:
├── Privilege change is automatic
├── Controlled by MSRs
├── Cannot be subverted
├── Well-defined entry/exit
├── Hardware-enforced
└── Foundation of security
```

### Interrupt Transition

Interrupts also cause a privilege increase:

```
Interrupt Transition:

1. User mode executing (CPL=3)
2. Hardware interrupt occurs
3. CPU performs:
   ├── Determines interrupt vector
   ├── Looks up IDT entry
   ├── Checks DPL (usually 0)
   ├── Saves SS, RSP
   ├── Saves RFLAGS
   ├── Saves CS, RIP
   ├── If privilege change:
   │   ├── Loads new SS from TSS
   │   ├── Loads new RSP from TSS
   │   └── Uses kernel stack
   ├── Clears IF (interrupt flag) - usually
   └── Jumps to handler (CPL=0)
4. Interrupt handler runs in kernel
5. Handler processes interrupt
6. Handler executes IRET
7. CPU performs:
   ├── Restores RIP, CS
   ├── Restores RFLAGS
   ├── Restores RSP, SS (if changed)
   └── Returns to user mode
8. User program continues

IRET Privilege Check:
├── IRET can lower privilege (kernel → user)
├── IRET can raise privilege (in some cases)
├── Checks segment DPL vs. CPL
├── General Protection if invalid
└── Prevents privilege escalation
```

### Gate Descriptors

Gates control how control transfers occur:

```
Gate Descriptors:

Call Gate:
├── Allows controlled call to higher privilege
├── Specifies entry point
├── Specifies target privilege level
├── Used for system calls (historically)
├── Rarely used in modern systems
└── Complex but flexible

Interrupt Gate:
├── Used for interrupt handling
├── Specifies handler
├── Usually DPL=0
├── Can be DPL=3 (user-invokable)
├── Clears IF when entering
└── Standard for hardware interrupts

Trap Gate:
├── Similar to interrupt gate
├── Does NOT clear IF
├── Used for exceptions where interrupts matter
├── Debug trap
└── Less common

Task Gate:
├── Performs full task switch
├── Complex and rarely used
├── Hardware context switch
├── Used in old systems
└── Largely obsolete

Gate Descriptor Structure (x86-64):
┌──────────────────────────────────────────────┐
│  Offset (63-32)                              │
│  Reserved                                    │
│  Type (Call/Interrupt/Trap)                  │
│  DPL (0-3)                                   │
│  P (Present)                                 │
│  Segment Selector (CS)                       │
│  IST (Interrupt Stack Table)                 │
│  Offset (31-16)                              │
│  Offset (15-0)                               │
└──────────────────────────────────────────────┘
```

### The TSS (Task State Segment)

The TSS holds stack pointers for privilege transitions:

```
TSS (Task State Segment):

Purpose:
├── Stores kernel stack pointers
├── Used during privilege transitions
├── Provides trusted stack
├── Essential for security
└── Referenced by hardware

Contents (relevant):
├── RSP0 (Ring 0 stack pointer)
├── RSP1 (Ring 1 stack pointer)
├── RSP2 (Ring 2 stack pointer)
├── IST1-IST7 (Interrupt Stack Table)
├── I/O Map Base Address
└── Various state fields (mostly unused)

Privilege Transition Using TSS:
1. Interrupt occurs while CPL=3
2. CPU needs to switch to kernel stack
3. Hardware reads TSS
4. Loads RSP from TSS.RSP0
5. Loads SS from TSS
6. Switches to kernel stack
7. Saves user state on kernel stack
8. Runs handler

IST (Interrupt Stack Table):
├── Alternative stacks for specific interrupts
├── Used for critical exceptions
├── NMI, double fault, machine check
├── Ensures valid stack even in bad cases
├── Configured per interrupt
└── Improves reliability

Linux Usage:
├── Each CPU has its own TSS
├── TSS holds kernel stack pointer
├── Switched during context switch
├── Updated on privilege transition
└── Essential for multitasking
```

---

## Privilege Levels on Other Architectures

### ARM Exception Levels (EL)

ARM uses Exception Levels instead of rings:

```
ARM Exception Levels:

EL0 - Application
├── User applications
├── Least privilege
├── Limited instruction set
├── No system control
└── Equivalent to x86 Ring 3

EL1 - Operating System
├── Kernel code
├── Privileged operations
├── System control
├── Memory management
└── Equivalent to x86 Ring 0

EL2 - Hypervisor
├── Virtual machine management
├── Controls EL1
├── Manages VMs
└── No x86 equivalent (new layer)

EL3 - Secure Monitor
├── Secure world (TrustZone)
├── Highest privilege
├── Switches between worlds
└── Firmware level

ARM Transition Rules:
├── Can only transition via exceptions
├── ERET instruction returns
├── SVC (Supervisor Call) for syscalls
├── HVC (Hypervisor Call) for hypervisor
├── SMC (Secure Monitor Call) for secure
└── Controlled transitions

Comparison:
├── EL0 = x86 Ring 3
├── EL1 = x86 Ring 0
├── EL2 = x86 hypervisor (root mode)
├── EL3 = x86 SMM (System Management Mode)
└── Similar concepts, different organization
```

### RISC-V Privilege Levels

RISC-V defines three privilege levels:

```
RISC-V Privilege Levels:

U (User) Mode:
├── User applications
├── Least privilege
├── Limited CSRs
├── No privileged instructions
└── Equivalent to x86 Ring 3

S (Supervisor) Mode:
├── Operating system kernel
├── Memory management (Sv39, Sv48)
├── Interrupt handling
├── System calls
└── Equivalent to x86 Ring 0

M (Machine) Mode:
├── Highest privilege
├── Firmware / bootloader
├── Hardware access
├── Cannot be trapped
└── Similar to x86 SMM

Transition Mechanisms:
├── ECALL: Environment call (syscall)
├── EBREAK: Breakpoint
├── SRET: Supervisor return
├── MRET: Machine return
└── Well-defined transitions
```

### Comparison Table

| Architecture | User Level          | Kernel Level     | Hypervisor      | Firmware        |
|--------------|---------------------|------------------|------------------|------------------|
| x86          | Ring 3              | Ring 0           | Ring -1          | Ring -2, -3      |
| ARM          | EL0                 | EL1              | EL2              | EL3              |
| RISC-V       | U                   | S                | HS/VS            | M                |
| MIPS         | User                | Kernel           | -                | -                |
| PowerPC      | Problem             | Supervisor       | Hypervisor       | -                |

---

## Modern Extensions

### Hypervisor Mode (Ring -1)

Virtualization introduces a new privilege level:

```
Hypervisor Mode:

Purpose:
├── Run virtual machines
├── Control guest OS
├── Manage virtualization
├── Isolate VMs
└── New privilege level

x86:
├── VMX root mode (Intel VT-x)
├── SVM mode (AMD-V)
├── Hypervisor runs in root mode
├── Guest OS runs in non-root mode
├── Ring 0 guest = Ring 0 restricted
└── Hypervisor can intercept everything

ARM:
├── EL2 (Hypervisor)
├── Above EL1 (kernel)
├── Controls VMs
├── Manages guest OSes
└── Standard for virtualization

RISC-V:
├── H extension
├── HS mode (Hypervisor Supervisor)
├── VS mode (Virtual Supervisor)
├── Nested virtualization
└── Flexible design

Hierarchy:
┌─────────────────────────────────────┐
│  Hypervisor (Ring -1)               │
│  ├── Highest practical privilege    │
│  ├── Controls VMs                   │
│  ├── Manages hardware               │
│  └── Provides isolation             │
├─────────────────────────────────────┤
│  Kernel (Ring 0)                    │
│  ├── Runs in VM or on bare metal    │
│  ├── Full control of its domain     │
│  └── Cannot escape VM               │
├─────────────────────────────────────┤
│  User (Ring 3)                      │
│  └── Applications                   │
└─────────────────────────────────────┘
```

### System Management Mode (SMM)

x86 has an even more privileged mode for firmware:

```
System Management Mode (SMM):

Privilege: Ring -2 (below hypervisor)
Purpose: Firmware operations
Trigger: System Management Interrupt (SMI)

Characteristics:
├── Hidden from OS
├── OS unaware of execution
├── Used for power management
├── Hardware error handling
├── Thermal management
├── BIOS/UEFI operations
└── Higher privilege than kernel

Security Concerns:
├── OS cannot control SMM
├── SMM can access all memory
├── Hidden execution environment
├── Potential for rootkits
├── Intel ME (Management Engine) even worse
└── Research area for security

Controversial:
├── Opaque to OS
├── Potential security risk
├── Hard to audit
├── Vendor-controlled
└── Ongoing research
```

### Intel Management Engine (ME)

The Intel ME is an even deeper level:

```
Intel Management Engine:

Privilege: Ring -3 (deepest)
Purpose: Out-of-band management
Location: Separate processor in chipset
OS Access: None

Capabilities:
├── Full system access
├── Independent of main CPU
├── Network access
├── Persistent
├── Always running (when powered)
├── Deep sleep support
└── Cannot be disabled easily

Uses:
├── Remote management (AMT)
├── Boot guard
├── Trusted execution
├── DRM support
├── Enterprise management
└── Various vendor features

Security Concerns:
├── Closed source
├── Backdoor potential
├── Cannot be audited
├── Exploitable
├── Persistent
└── NSA concerns

Other Vendors:
├── AMD PSP (Platform Security Processor)
├── ARM TrustZone (different model)
├── Apple Secure Enclave
├── Google Titan
└── Various security chips
```

### TrustZone (ARM)

ARM uses a different approach—secure vs. non-secure worlds:

```
ARM TrustZone:

Two Worlds:
├── Secure World
│   ├── Trusted code
│   ├── Runs at EL3/EL1S
│   ├── Handles secrets
│   └── Protected resources
│
└── Non-Secure World
    ├── Normal OS
    ├── Runs at EL1NS/EL0
    ├── Applications
    └── Protected from secure

Switching:
├── SMC instruction
├── Secure Monitor (EL3)
├── Manages transitions
├── Saves/restores state
└── Isolates memory

Uses:
├── DRM
├── Payment
├── Biometric data
├── Cryptographic keys
├── Secure boot
└── Trusted execution

Advantages:
├── Hardware isolation
├── Simple model
├── Efficient
├── Well-supported
└── Widely deployed

Comparison:
├── Different from x86 rings
├── Parallel worlds vs. hierarchy
├── Better for some use cases
├── Complementary to rings
└── Both approaches valid
```

---

## Practical Implications

### For Application Programmers

```
What Application Programmers Should Know:

1. User Mode Restrictions
   ├── Cannot execute privileged instructions
   ├── Cannot access kernel memory
   ├── Cannot access other processes
   ├── Must use system calls
   └── Cannot change privilege level

2. System Calls
   ├── Only way to request OS services
   ├── Expensive (privilege switch)
   ├── Minimize when possible
   ├── Use library functions
   └── Cache results if possible

3. Memory Model
   ├── Own virtual address space
   ├── Isolated from other processes
   ├── Cannot see kernel memory
   ├── Virtual addresses (not physical)
   └── Protection enforced by hardware

4. Error Handling
   ├── Privilege violations cause SIGSEGV
   ├── Invalid instructions cause SIGILL
   ├── Check return values
   ├── Handle errors gracefully
   └── Don't crash the system

5. Security
   ├── Cannot bypass protection
   ├── Cannot read kernel memory
   ├── Cannot modify kernel code
   ├── Attack surface limited
   └── Vulnerabilities still exist
```

### For System Programmers

```
What System Programmers Should Know:

1. Kernel Mode
   ├── Full hardware access
   ├── Can execute all instructions
   ├── Can access all memory
   ├── Higher privilege (Ring 0)
   └── More responsibility

2. Privilege Transitions
   ├── Careful save/restore
   ├── Validate user input
   ├── Never trust user pointers
   ├── Check all arguments
   └── Prevent privilege escalation

3. Memory Management
   ├── Set up page tables
   ├── Configure U/S bits correctly
   ├── Set R/W permissions
   ├── Handle page faults
   └── Maintain isolation

4. System Calls
   ├── Validate all arguments
   ├── Check pointers carefully
   ├── Use copy_from_user/copy_to_user
   ├── Never direct access to user memory
   └── Prevent TOCTOU attacks

5. Security
   ├── Attack surface is critical
   ├── Buffer overflows are vulnerabilities
   ├── Every bug can be exploited
   ├── Careful with concurrency
   └── Defense in depth

6. Testing
   ├── Privilege-related bugs are serious
   ├── Test with malicious inputs
   ├── Fuzzing is important
   ├── Code review critical
   └── Security audits necessary
```

---

## Common Misconceptions

**Misconception 1**: "Rings 1 and 2 are used by modern OSes"

**Reality**: Modern general-purpose operating systems typically use only Ring 0 (kernel) and Ring 3 (user). Rings 1 and 2 are largely unused, despite being part of the architecture. Some specialized systems may use them, but it's rare.

**Misconception 2**: "Privilege levels prevent all security issues"

**Reality**: Privilege levels prevent many attacks but not all. Vulnerabilities in kernel code, side-channel attacks (Spectre, Meltdown), and firmware attacks can bypass or ignore privilege levels. Defense in depth is essential.

**Misconception 3**: "User code can never affect kernel"

**Reality**: User code can affect the kernel through system calls (which is the intended interface). Malicious or buggy user code can exploit kernel vulnerabilities to gain kernel privileges. The privilege boundary is a strong protection but not absolute.

**Misconception 4**: "There are only two privilege levels"

**Reality**: While most systems use two levels in practice (kernel and user), the hardware supports more. Modern x86 has added hypervisor mode (Ring -1) and firmware modes (Ring -2, -3). ARM has four exception levels. The model is more complex than often presented.

**Misconception 5**: "Privilege levels are only about instructions"

**Reality**: Privilege levels control both instruction execution and memory access. The Memory Management Unit (MMU) enforces memory protection using privilege levels. Both are equally important.

**Misconception 6**: "Higher privilege means faster execution"

**Reality**: Privilege level doesn't affect execution speed of instructions. Higher privilege means more capabilities, not faster execution. However, the overhead of crossing privilege levels (system calls) does impact performance.

**Misconception 7**: "The kernel runs in a single privilege level"

**Reality**: Modern CPUs have multiple levels "above" the kernel (hypervisor, SMM, ME). Even within Ring 0, the kernel has its own internal privilege model (e.g., capabilities, namespace restrictions). The picture is more nuanced than a simple two-level model.

---

## Summary

Privilege levels are hardware-enforced mechanisms that control what code can do. The x86 architecture uses four protection rings, with Ring 0 for the kernel and Ring 3 for user applications. Other architectures (ARM, RISC-V) use similar concepts with different names. Privilege levels control both instruction execution and memory access, forming the foundation of system security.

### Key Points

1. Ring 0 is kernel mode—full hardware access and privileges.
2. Ring 3 is user mode—restricted access for applications.
3. Rings 1-2 are rarely used in modern systems.
4. Privileged instructions can only execute in kernel mode.
5. Memory protection uses U/S bits in page tables.
6. Transitions occur through system calls, interrupts, and exceptions.
7. The TSS provides kernel stacks for privilege transitions.
8. Modern extensions include hypervisor mode (Ring -1) and firmware modes.

---

## Key Takeaways

1. Hardware enforces privilege — not just convention.
2. Two levels dominate — Ring 0 (kernel) and Ring 3 (user).
3. Memory protection is privilege-based — U/S bits in page tables.
4. Controlled transitions exist — only specific mechanisms change privilege.
5. The TSS is essential — provides trusted kernel stacks.
6. Modern systems add layers — hypervisor, SMM, TrustZone.
7. Attack surface is critical — privilege bugs are security vulnerabilities.
8. Defense in depth is necessary — privilege levels are one layer among many.

---

## What's Next?

Continue to Boot Process to see how the system transitions through privilege levels during startup, from firmware to bootloader to kernel to user space.

---

## Further Reading

### Books

- "Intel 64 and IA-32 Architectures Software Developer's Manual" — Volume 3: System Programming Guide
- "ARM Architecture Reference Manual"
- "Operating Systems: Three Easy Pieces" by Remzi and Andrea Arpaci-Dusseau
- "Modern Operating Systems" by Andrew S. Tanenbaum
- "Understanding the Linux Kernel" by Bovet and Cesati
- "Linux Kernel Development" by Robert Love

### Papers

- "The UNIX Time-Sharing System" by Ritchie and Thompson (1974)
- "seL4: Formal Verification of an OS Kernel" by Klein et al. (2009)
- "Spectre Attacks: Exploiting Speculative Execution" by Kocher et al. (2018)
- "Meltdown: Reading Kernel Memory from User Space" by Lipp et al. (2018)

### Online Resources

- OSDev Wiki — Protection rings and privilege levels
- Intel Software Developer Manuals (intel.com)
- ARM Developer Documentation (developer.arm.com)
- OSTEP (ostep.org) — Free textbook
- Linux Kernel Documentation (kernel.org)

