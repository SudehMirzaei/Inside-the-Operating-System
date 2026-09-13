# Inside the Operating System

A comprehensive, beginner-to-advanced guide to understanding how operating systems work from the inside out. This repository covers the fundamental concepts, core subsystems, and internal mechanisms that power modern operating systems.

---

## 📖 About This Repository

Inside the Operating System is a structured learning resource that takes you on a deep dive into the internals of operating systems. Whether you're a student, a developer preparing for systems interviews, or a curious engineer, this repository provides clear, detailed explanations of how operating systems manage hardware, execute programs, and provide services to applications.

The content is organized progressively—starting with foundational concepts and building up to advanced topics like virtual memory, file systems, and I/O subsystems.

---

## Table of Contents

### OS Fundamentals
- [Introduction](docs/OS-Fundamentals/Introduction.md)
- [What is an Operating System](docs/OS-Fundamentals/What-is-an-Operating-System.md)
- [OS Goals and Responsibilities](docs/OS-Fundamentals/OS-Goals-and-Responsibilities.md)
- [OS Architecture](docs/OS-Fundamentals/OS-Architecture.md)
- [Kernel and User Space](docs/OS-Fundamentals/Kernel-and-User-Space.md)
- [Privilege Levels](docs/OS-Fundamentals/Privilege-Levels.md)
- [Boot Process](docs/OS-Fundamentals/Boot-Process.md)
- [Interrupts and Exceptions](docs/OS-Fundamentals/Interrupts-and-Exceptions.md)

### Process Management
- [Introduction](docs/Process-Management/Introduction.md)
- [Program vs Process](docs/Process-Management/Program-vs-Process.md)
- [Process Lifecycle](docs/Process-Management/Process-Lifecycle.md)
- [Process States](docs/Process-Management/Process-States.md)
- [Process Control Block](docs/Process-Management/Process-Control-Block.md)
- [Process Creation](docs/Process-Management/Process-Creation.md)
- [Process Termination](docs/Process-Management/Process-Termination.md)
- [Context Switching](docs/Process-Management/Context-Switching.md)
- [Threads](docs/Process-Management/Threads.md)
- [Processes vs Threads](docs/Process-Management/Processes-vs-Threads.md)

### Program Execution
- [Introduction](docs/Program-Execution/Introduction.md)
- [From Source Code to Executable](docs/Program-Execution/From-Source-Code-to-Executable.md)
- [Compilation](docs/Program-Execution/Compilation.md)
- [Linking](docs/Program-Execution/Linking.md)
- [Executable Files](docs/Program-Execution/Executable-Files.md)
- [Program Loading](docs/Program-Execution/Program-Loading.md)
- [Process Address Space](docs/Program-Execution/Process-Address-Space.md)
- [Dynamic Linking](docs/Program-Execution/Dynamic-Linking.md)
- [Program Startup](docs/Program-Execution/Program-Startup.md)
- [How a Program Runs](docs/Program-Execution/How-a-Program-Runs.md)

### Memory Management
- [Introduction](docs/Memory-Management/Introduction.md)
- [Physical Memory](docs/Memory-Management/Physical-Memory.md)
- [Virtual Memory Introduction](docs/Memory-Management/Virtual-Memory-Introduction.md)
- [Process Address Space](docs/Memory-Management/Process-Address-Space.md)
- [Stack and Heap](docs/Memory-Management/Stack-and-Heap.md)
- [Memory Allocation](docs/Memory-Management/Memory-Allocation.md)
- [Paging](docs/Memory-Management/Paging.md)
- [Page Tables](docs/Memory-Management/Page-Tables.md)
- [TLB](docs/Memory-Management/TLB.md)
- [Memory Protection](docs/Memory-Management/Memory-Protection.md)
- [Memory-Mapped Files](docs/Memory-Management/Memory-Mapped-Files.md)

### CPU Scheduling
- [Introduction](docs/CPU-Scheduling/Introduction.md)
- [CPU and Execution](docs/CPU-Scheduling/CPU-and-Execution.md)
- [Scheduling Basics](docs/CPU-Scheduling/Scheduling-Basics.md)
- [Preemptive vs Nonpreemptive](docs/CPU-Scheduling/Preemptive-vs-Nonpreemptive.md)
- [FCFS](docs/CPU-Scheduling/FCFS.md)
- [SJF](docs/CPU-Scheduling/SJF.md)
- [Round Robin](docs/CPU-Scheduling/Round-Robin.md)
- [Priority Scheduling](docs/CPU-Scheduling/Priority-Scheduling.md)
- [Multilevel Queues](docs/CPU-Scheduling/Multilevel-Queues.md)
- [Real World Scheduling](docs/CPU-Scheduling/Real-World-Scheduling.md)

### System Calls
- [Introduction](docs/System-Calls/Introduction.md)
- [User Space vs Kernel Space](docs/System-Calls/User-Space-vs-Kernel-Space.md)
- [What is a System Call](docs/System-Calls/What-is-a-System-Call.md)
- [System Call Mechanism](docs/System-Calls/System-Call-Mechanism.md)
- [System Call Interface](docs/System-Calls/System-Call-Interface.md)
- [Process System Calls](docs/System-Calls/Process-System-Calls.md)
- [File System Calls](docs/System-Calls/File-System-Calls.md)
- [Memory System Calls](docs/System-Calls/Memory-System-Calls.md)
- [I/O System Calls](docs/System-Calls/I-O-System-Calls.md)
- [System Call Trace](docs/System-Calls/System-Call-Trace.md)

### File Systems
- [Introduction](docs/File-Systems/Introduction.md)
- [Files and Directories](docs/File-Systems/Files-and-Directories.md)
- [File Descriptors](docs/File-Systems/File-Descriptors.md)
- [File System Architecture](docs/File-Systems/File-System-Architecture.md)
- [Disk Organization](docs/File-Systems/Disk-Organization.md)
- [File Allocation](docs/File-Systems/File-Allocation.md)
- [Inodes](docs/File-Systems/Inodes.md)
- [Directory Structure](docs/File-Systems/Directory-Structure.md)
- [Permissions](docs/File-Systems/Permissions.md)
- [Journaling](docs/File-Systems/Journaling.md)

### I/O Systems
- [Introduction](docs/I-O-Systems/Introduction.md)
- [I/O Architecture](docs/I-O-Systems/I-O-Architecture.md)
- [Devices and Controllers](docs/I-O-Systems/Devices-and-Controllers.md)
- [Device Drivers](docs/I-O-Systems/Device-Drivers.md)
- [Interrupt Driven I/O](docs/I-O-Systems/Interrupt-Driven-I-O.md)
- [DMA](docs/I-O-Systems/DMA.md)
- [Buffering and Caching](docs/I-O-Systems/Buffering-and-Caching.md)
- [I/O System Calls](docs/I-O-Systems/I-O-System-Calls.md)

### Virtual Memory
- [Introduction](docs/Virtual-Memory/Introduction.md)
- [Why Virtual Memory](docs/Virtual-Memory/Why-Virtual-Memory.md)
- [Demand Paging](docs/Virtual-Memory/Demand-Paging.md)
- [Page Faults](docs/Virtual-Memory/Page-Faults.md)
- [Page Replacement](docs/Virtual-Memory/Page-Replacement.md)
- [FIFO](docs/Virtual-Memory/FIFO.md)
- [LRU](docs/Virtual-Memory/LRU.md)
- [Working Set](docs/Virtual-Memory/Working-Set.md)
- [Memory Overcommit](docs/Virtual-Memory/Memory-Overcommit.md)

---

## 🎯 Who Is This For?

- Students studying operating systems, computer architecture, or systems programming
- Developers preparing for technical interviews at systems-focused companies
- Engineers who want to deepen their understanding of how computers really work
- Anyone curious about the invisible software that powers every computer

---

## 🚀 How to Use This Repository

1. Start with OS Fundamentals if you're new to operating systems.
2. Follow the sections in order for a structured learning path.
3. Jump to specific topics if you're looking for particular concepts.
4. Read the Introduction of each section first to understand the context.
5. Cross-reference between sections—topics are interconnected.

---

## 🗺️ Learning Path

```
Beginner Path:
OS-Fundamentals → Process-Management → Program-Execution

Intermediate Path:
Memory-Management → CPU-Scheduling → System-Calls

Advanced Path:
File-Systems → I-O-Systems → Virtual-Memory
```

---

## 📝 Content Format

Each document in this repository follows a consistent structure:

- Introduction — Context and overview
- Core Concepts — Detailed explanations with diagrams
- Examples — Practical, concrete illustrations
- Comparisons — Trade-offs and alternatives
- Key Takeaways — Summary of important points
- Further Reading — Related topics and references

---

## 🤝 Contributing

Contributions are welcome! If you find errors, want to add examples, or improve explanations:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

---

## 📄 License

This project is intended for educational purposes. Feel free to use, share, and adapt the content with attribution.

---

## ⭐ Acknowledgments

This repository is inspired by classic operating systems textbooks, academic courses, and the collective knowledge of the systems programming community.

---

Happy Learning! 🚀

Understanding operating systems is understanding the foundation of all modern computing.
