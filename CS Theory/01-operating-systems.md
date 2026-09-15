[🏠 Back to CS Theory Index](./README.md) • [Next: Database Management Systems ➡️](./02-database-management-systems.md)

<div align="center">
  <h1>01. Operating Systems (OS)</h1>
  <p><b>Kernel Architecture, Process Lifecycle, Scheduling, Synchronization, Deadlocks, & Virtual Memory</b></p>
</div>

---

## 📑 Module Index
- [1. OS Architecture & Core Functions](#1-os-architecture--core-functions)
  - [Dual Mode Operation (Kernel Mode vs User Mode)](#dual-mode-operation)
  - [System Calls, Interrupts & Context Switching](#system-calls--interrupts)
- [2. Processes, Threads & Concurrency](#2-processes-threads--concurrency)
  - [Process vs Thread vs Coroutine](#process-vs-thread-vs-coroutine)
  - [Process State Transition Diagram & PCB](#process-lifecycle--pcb)
  - [Fork, Exec, Zombie & Orphan Processes](#process-management-syscalls)
- [3. CPU Scheduling Algorithms](#3-cpu-scheduling-algorithms)
  - [Preemptive vs Non-Preemptive Scheduling](#scheduling-types)
  - [FCFS, SJF / SRTF, Round Robin & MLFQ](#cpu-scheduling-algorithms)
- [4. Process Synchronization & Critical Section](#4-process-synchronization--critical-section)
  - [Race Conditions & The 3 Critical Section Criteria](#critical-section-problem)
  - [Mutex Locks vs Binary & Counting Semaphores](#mutex-vs-semaphores)
  - [Classic Problems: Producer-Consumer, Readers-Writers, Dining Philosophers](#classic-synchronization-problems)
- [5. Deadlocks](#5-deadlocks)
  - [The 4 Coffman Conditions](#the-4-coffman-conditions)
  - [Deadlock Handling Strategies & Banker's Algorithm](#deadlock-handling--bankers-algorithm)
- [6. Memory Management & Virtual Memory](#6-memory-management--virtual-memory)
  - [Logical vs Physical Address Space & MMU](#logical-vs-physical-addressing)
  - [Paging, Multi-Level Page Tables & TLB](#paging--tlb)
  - [Page Faults & Page Replacement Algorithms (LRU, Optimal, FIFO, Clock)](#page-replacement-algorithms)

---

## 1. OS Architecture & Core Functions

### Dual Mode Operation

The OS protects hardware and critical memory structures using hardware-enforced CPU execution modes:

```
┌─────────────────────────────────────────────────────────────┐
│ USER SPACE (Ring 3 / User Mode)                             │
│ • User Applications (Node.js, Browser, React Native engine) │
│ • Restricted CPU instructions; cannot touch hardware directly│
└──────────────────────────────┬──────────────────────────────┘
                               │ System Call (e.g., read(), write(), fork())
                               ▼ (Trap / Software Interrupt switches Mode Bit 1 -> 0)
┌─────────────────────────────────────────────────────────────┐
│ KERNEL SPACE (Ring 0 / Kernel Mode)                         │
│ • OS Kernel (Memory Manager, Scheduler, Device Drivers)     │
│ • Full unrestricted access to CPU instructions & hardware   │
└─────────────────────────────────────────────────────────────┘
```

---

### System Calls & Context Switching

- **System Call:** Programmatic request from user space to kernel space to perform privileged operations (I/O, process creation, network sockets).
- **Context Switch:** The mechanism of saving the state of the currently running process/thread (registers, Program Counter, stack pointer in PCB) and loading the state of another process:
  - **Overhead:** Direct CPU cycle cost (~microseconds) + indirect cache invalidation (L1/L2/L3 cache misses and TLB flush).

---

## 2. Processes, Threads & Concurrency

### Process vs Thread vs Coroutine

```
┌────────────────────────────────── PROCESS A ──────────────────────────────────┐
│  Code Segment  │  Data (Globals)  │  Heap (Dynamic Memory)  │  Open File Descr.│
├───────────────────────────────────────────────────────────────────────────────┤
│  Thread 1: [ Registers | PC | Stack ]   Thread 2: [ Registers | PC | Stack ] │
└───────────────────────────────────────────────────────────────────────────────┘
```

| Dimension | Process | Thread (Kernel Thread) | Coroutine / Green Thread |
| :--- | :--- | :--- | :--- |
| **Address Space** | Isolated memory space (No direct sharing) | Shares Code, Data, Heap with parent process | Shares thread memory space |
| **Creation Cost** | Heavyweight ($O(\text{MBs})$ memory, expensive `fork()`) | Lightweight ($O(\text{KBs})$ stack) | Ultra-lightweight ($O(\text{Bytes})$) |
| **Context Switch** | Expensive (Switches Page Tables & flushes TLB) | Fast (CPU registers & stack only) | Instant (User-space cooperative switch) |
| **Communication** | IPC (Pipes, Sockets, Shared Memory, Message Queues) | Direct shared memory variables | Async/Await, Channels |

---

### Process Lifecycle & PCB

```
          [ NEW ] ──► (Admitted)
                         │
                         ▼
        ┌──────────► [ READY ] ◄──────────┐
        │                │                │
        │ (I/O Done)     │ (Dispatched)   │ (Timer Interrupt / Yield)
        │                ▼                │
   [ WAITING / BLOCKED ] ◄── [ RUNNING ] ─┘
        (Waiting for I/O)         │
                                  ▼ (Exit)
                           [ TERMINATED ]
```

- **Process Control Block (PCB):** Data structure stored in kernel memory containing PID, Process State, Program Counter (PC), CPU Registers, Memory Limits, and Open File Descriptors.

---

### Process Management: Fork, Zombie & Orphan

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork(); // Creates child process with copy-on-write memory

    if (pid == 0) {
        // Child Process
        printf("Child PID: %d, Parent PID: %d\n", getpid(), getppid());
    } else if (pid > 0) {
        // Parent Process
        wait(NULL); // Waits for child to prevent Zombie process!
        printf("Parent finished.\n");
    }
    return 0;
}
```

- **Zombie Process:** A child process that has completed execution (`exit()`), but its entry still remains in the Process Table because the parent has not yet called `wait()` to read its exit status.
- **Orphan Process:** A child process whose parent process terminated before it. The `init` process (PID 1) adopts the orphan and reaps it upon termination.

---

## 3. CPU Scheduling Algorithms

### Preemptive vs Non-Preemptive

- **Non-Preemptive:** Once CPU is assigned to a process, it holds it until it terminates or blocks for I/O.
- **Preemptive:** The OS scheduler can interrupt and pause a currently running process when a higher-priority or shorter task arrives.

---

### Core Scheduling Algorithms

| Algorithm | Preemptive? | Criterion | Pros & Cons |
| :--- | :---: | :--- | :--- |
| **FCFS** (First-Come, First-Served) | No | Arrival Time | Simple; suffers from **Convoy Effect** (short jobs wait behind huge CPU burst). |
| **SJF** (Shortest Job First) | No | Shortest Burst | Optimal average turnaround time; impossible to know exact future burst duration. |
| **SRTF** (Shortest Remaining Time First) | Yes | Remaining Burst Time | Preemptive SJF; minimal average waiting time, but causes starvation for long jobs. |
| **Round Robin (RR)** | Yes | Fixed Time Quantum ($q$) | Fair, responsive for interactive systems. Large $q \rightarrow$ FCFS; small $q \rightarrow$ high context-switch overhead. |
| **MLFQ** (Multi-Level Feedback Queue) | Yes | Multiple priority queues with adaptive quantum | Modern standard in Linux/macOS. Automatically penalizes CPU-heavy jobs and boosts I/O-bound interactive tasks. |

---

## 4. Process Synchronization & Critical Section

### The Critical Section Problem

A **Critical Section** is a code block that accesses shared mutable resources. Any valid synchronization solution must satisfy **3 mandatory criteria**:

1. **Mutual Exclusion:** If process $P_i$ is executing in its critical section, no other processes can be executing in their critical sections.
2. **Progress:** If no process is in its critical section and some wish to enter, selection cannot be postponed indefinitely.
3. **Bounded Waiting:** A bound must exist on the number of times other processes are allowed to enter after a process has made a request (Prevents Starvation).

---

### Mutex vs Semaphores

```
┌──────────────────────────────────┬──────────────────────────────────┐
│              MUTEX               │            SEMAPHORE             │
├──────────────────────────────────┼──────────────────────────────────┤
│ • Binary locking mechanism (0/1) │ • Integer signaling counter ($S$)│
│ • **Ownership:** Only the thread │ • Any thread can signal / post   │
│   that acquired the lock can release it. │   (No ownership).        │
│ • Used for Mutual Exclusion.     │ • Binary ($S=1$) or Counting ($S=N$)│
│                                  │   (Manages access to resource pool)│
└──────────────────────────────────┴──────────────────────────────────┘
```

#### Semaphore Operations:
- `wait(S)` (or `P(S)`): Decrements $S$. If $S \le 0$, blocks until signaled.
- `signal(S)` (or `V(S)`): Increments $S$. Wakes up one waiting process.

---

## 5. Deadlocks

A **Deadlock** is a state where a set of processes are blocked because each process is holding a resource and waiting for another resource acquired by some other process.

### The 4 Coffman Conditions (Must ALL hold simultaneously)

```
1. Mutual Exclusion  ──► Resources cannot be shared simultaneously.
2. Hold and Wait     ──► A process holds >= 1 resource and waits for additional resources.
3. No Preemption     ──► Resources cannot be forcibly confiscated from a process.
4. Circular Wait     ──► P0 waits for P1 -> P1 waits for P2 -> ... -> Pn waits for P0.
```

---

### Banker's Algorithm (Deadlock Avoidance)

Used in resource allocation to ensure the system always remains in a **Safe State** (a sequence $\langle P_1, P_2, \dots, P_n \rangle$ exists where all processes can complete).

$$\text{Need}[i][j] = \text{Max}[i][j] - \text{Allocation}[i][j]$$

- When a process requests resources, the OS simulates the allocation. If the resulting state has a valid safe sequence, grant request; otherwise, make the process wait.

---

## 6. Memory Management & Virtual Memory

### Logical vs Physical Addressing & MMU

```
   CPU ──► [ Logical Address: Page # | Offset ]
                     │
                     ▼
           [ Memory Management Unit (MMU) ] ◄── (Translates via Page Table / TLB)
                     │
                     ▼
           [ Physical Address: Frame # | Offset ] ──► [ Physical RAM ]
```

---

### Paging & Translation Lookaside Buffer (TLB)

- **Page:** Fixed-size block of logical memory (typically 4 KB).
- **Frame:** Fixed-size block of physical RAM.
- **Page Table:** Maps virtual Page Numbers (VPN) to physical Frame Numbers (PFN).
- **TLB (Translation Lookaside Buffer):** Ultra-fast on-chip associative hardware cache for recent virtual-to-physical address mappings:
  $$\text{Effective Access Time (EAT)} = (\text{Hit Ratio} \times T_{\text{TLB}+\text{RAM}}) + ((1 - \text{Hit Ratio}) \times T_{\text{TLB}+2\text{RAM}})$$

---

### Page Replacement Algorithms

When a **Page Fault** occurs and physical RAM is full, the OS must evict a page:

```
Reference String: 7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2
```

1. **FIFO (First-In, First-Out):** Evicts the oldest loaded page. Suffers from **Belady's Anomaly** (adding more physical frames can increase page faults!).
2. **LRU (Least Recently Used):** Evicts the page that has not been used for the longest period. Optimal practical heuristic.
3. **Optimal (OPT / Belady's Min):** Evicts the page that will not be used for the longest period in the *future*. (Theoretical benchmark).
4. **Clock Algorithm (Second Chance):** Uses a circular buffer and reference bit ($0$ or $1$) to approximate LRU with $O(1)$ overhead.

---

[➡️ Continue to Module 02: Database Management Systems (DBMS)](./02-database-management-systems.md)
