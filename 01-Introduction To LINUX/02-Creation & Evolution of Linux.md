# 02-Creation & Evolution of Linux

---

## 1. Historical Context & Technical Inception (1991)

### The MINIX Factor

#### Andrew Tanenbaum, MINIX, and Microkernel Architecture

In 1987, Professor Andrew Tanenbaum released **MINIX**, a lightweight UNIX-like operating system designed as an educational tool to accompany his textbook *Operating Systems: Design and Implementation*. MINIX was architected around a microkernel paradigm:

```
+-------------------------------------------------------------------------+
|                         MINIX Microkernel Model                         |
|                                                                         |
|  [ User Processes ]    [ File Server ]    [ Network Server ]  (User)    |
|  ============================================================= (IPC)    |
|  [ Microkernel: Interrupts, Message Passing, Process Scheduling ] (Kernel)|
+-------------------------------------------------------------------------+

```

* **Microkernel Isolation:** The core kernel managed only low-level operations—interrupt handling, process scheduling, and IPC message routing. Key operating system abstractions (such as the file system, device drivers, and network stacks) executed as unprivileged user-space server tasks.
* **Educational & Licensing Constraints:** MINIX was released with full source code for study, but its license restricted commercial deployment and modification distribution without purchasing the publisher's media kit. Furthermore, Tanenbaum prioritized design simplicity for undergraduate instruction over performance optimizations, declining patches that added complex production features.

#### Hardware Hardware Constraints & x86 Protected Mode

By 1991, personal computing hardware was shifting from 16-bit chips to the **Intel 80386** 32-bit processor architecture. The 80386 introduced features essential for modern multitasking operating systems:

* **Hardware Protected Mode:** Memory access protection via ring levels ($Ring_0$ to $Ring_3$).
* **Paged Memory Management Unit (MMU):** 32-bit linear address space with 4 KB memory pages, enabling virtual memory paging to disk.
* **Hardware Task Switching:** Support for Task State Segments (TSS) enabling context switches.

MINIX 1.x was constrained by its goal of maintaining compatibility with older 16-bit Intel 8086/8088 machines, leaving much of the 80386 processor's native 32-bit protected mode capabilities underutilized.

#### Torvalds' Initial Motivations

In early 1991, Linus Torvalds, a computer science student at the University of Helsinki, purchased an 80386-based clone PC (33 MHz, 4 MB RAM). Frustrated by MINIX's licensing limitations, lack of 32-bit task switching utilization, and teletype emulator restrictions, Torvalds began writing raw assembly and C code to explore 386 execution contexts:

```
                  +-----------------------------------+
                  |   Torvalds' Initial 386 Exploration|
                  +-----------------------------------+
                                    |
            +-----------------------+-----------------------+
            |                                               |
            v                                               v
  [ Task 1: Terminal Emulator ]           [ Task 2: Disk I/O & Modems ]
  - Direct 80386 Assembly                 - Device Driver Development
  - Asynchronous Serial Driver            - Raw Storage Reading
            |                                               |
            +-----------------------+-----------------------+
                                    |
                                    v
                     [ Need for an Underlying Kernel ]
                     - Process Creation (`fork`)
                     - File System Drivers (`POSIX`)

```

On **August 25, 1991**, Torvalds posted his famous Usenet announcement to the `comp.os.minix` newsgroup:

> *"I'm doing a (free) operating system (just a hobby, won't be big and professional like gnu) for 386(406) AT clones. This has been brewing since april, and is starting to get ready. I'd like any feedback on things people like/dislike in minix, as my OS resembles it somewhat (same physical layout of the file-system (due to practical reasons) among other things)..."*

---

### The Tanenbaum–Torvalds Debate (1992)

In January 1992, Tanenbaum posted a thread titled *"LINUX is obsolete"* to `comp.os.minix`, triggering an architectural debate on kernel design trade-offs.

```
          MICROKERNEL vs. MONOLITHIC ARCHITECTURAL PARADIGM

      +-------------------------------------------------------+
      |                 Microkernel (MINIX)                   |
      |                                                       |
      |  User    [FS] <---> [Dev] <---> [Net]  (User Space)   |
      |  IPC  ===================================== (Context) |
      |  Kernel           [IPC / Sched]        (Kernel Space) |
      +-------------------------------------------------------+
      * High IPC message-passing overhead on 1990s hardware.
      * Fault isolation: Server crash doesn't bring down kernel.

      +-------------------------------------------------------+
      |                 Monolithic (Linux)                    |
      |                                                       |
      |  User               [ Applications ]                  |
      |  Syscall ==================================== (Trap)  |
      |  Kernel  [ VFS | MM | IPC | Drivers | Sched ] (Kernel)|
      +-------------------------------------------------------+
      * Zero IPC overhead for intra-kernel subsystem routines.
      * Single address space: Driver crash can trigger kernel panic.

```

#### Key Technical Arguments

* **Microkernel Argument (Tanenbaum):** Monolithic kernels are obsolete because putting device drivers, file systems, and virtual memory components inside a single execution address space degrades code modularity. A bug in a driver can corrupt kernel memory and crash the entire system.
* **Monolithic Defense (Torvalds):** Microkernels introduce substantial overhead due to continuous Context Switches and IPC message copies between user-space servers and kernel threads. On 1990s hardware, this inter-process message-passing penalty degraded real-world performance.

#### Torvalds' Pragmatic Focus

Torvalds prioritized raw execution speed and functional hardware capability over theoretical design purity. By running drivers, file system routines, and memory management in a unified privileged memory space ($Ring_0$), Linux eliminated IPC message-passing latencies, achieving superior performance on consumer hardware.

---

## 2. GNU Project Integration & Licensing Evolution

### The GNU Project (Richard Stallman & Free Software Foundation)

Initiated by Richard Stallman in 1983, the **GNU Project** aimed to construct a completely free, UNIX-compatible operating system. By 1990, the Free Software Foundation (FSF) had completed key userland components:

* **GNU C Compiler (`gcc`):** High-performance C compiler framework.
* **GNU C Library (`glibc`):** Standard C runtime interfaces and system call wrappers.
* **Command Line Interfaces:** `bash`, `coreutils` (`ls`, `grep`, `tar`, `sed`).
* **Development Tools:** `gdb`, `make`, `bison`.

```
                  GNU System Integration Status (1991)

  +------------------------------------------------------------------+
  |  GNU Userland Framework (COMPLETE)                               |
  |  [ GCC Compiler ] [ Bash Shell ] [ Core Utilities ] [ GDB ]     |
  +------------------------------------------------------------------+
                                   |
                                   v  (Missing System Infrastructure)
  +------------------------------------------------------------------+
  |  GNU Hurd Kernel (INCOMPLETE)                                   |
  |  Complex Microkernel (Mach 3.0 + Mach Servers)                    |
  +------------------------------------------------------------------+

```

The missing component of the GNU vision was the kernel. Development of the official GNU kernel, **GNU Hurd** (a complex collection of servers running atop the Mach microkernel), faced architectural delays. Torvalds' Linux kernel filled this functional gap.

---

### Licensing Shift: From Custom 0.01 to GNU GPLv2

The initial releases of Linux (0.01 and 0.02 in late 1991) carried a custom, restrictive license created by Torvalds:

```c
/* 
 * Excerpt from original Linux 0.01/0.02 copyright terms:
 * 
 * This system is distributed freely... 
 * It may NOT be sold or redistributed for commercial gain.
 */

```

#### Transition to GPLv2

In December 1991 (version 0.11/0.12 release window), Torvalds re-licensed the code base under the **GNU General Public License Version 2 (GPLv2)**.

```
                        GPLv2 Copyleft Engine

     +--------------------------------------------------------+
     |                  Linux Kernel Source                   |
     +--------------------------------------------------------+
                                  |
                                  v
     +--------------------------------------------------------+
     |       GPLv2 Terms (Copyleft Protection Mechanism)      |
     |                                                        |
     |  1. Freedom to run, study, and modify the code.        |
     |  2. Re-distribution MUST include source code.          |
     |  3. Derivative works MUST inherit the same GPLv2 terms. |
     +--------------------------------------------------------+

```

#### Strategic Impact of GPLv2

* **Prevention of Proprietary Fragmentation:** Any commercial entity modifying the kernel and distributing the resulting binary must publish their modifications under GPLv2. This eliminated the risk of proprietary "fork-and-hide" fragmentation seen in classical UNIX variants.
* **Community Contributions:** Corporate competitors (e.g., IBM, HP, Intel) could safely contribute engineering resources to a shared kernel codebase without fear that a competitor would privatize their updates.

---

### The GNU/Linux Naming & Architectural Symbiosis

Technically, **Linux** refers specifically to the kernel—the system's central resource manager:

```
+-----------------------------------------------------------------------+
|                       GNU Operating System Scope                      |
|                                                                       |
|  +-----------------------------------------------------------------+  |
|  | User Application Space (GNU Coreutils, Bash, Toolchains, Glibc) |  |
|  +-----------------------------------------------------------------+  |
|                                  | System Calls                       |
|  ================================v=================================  |
|                                                                       |
|  +-----------------------------------------------------------------+  |
|  | Linux Kernel Scope                                              |  |
|  | [ Memory Mgmt | Scheduler | VFS | Device Drivers | Net Stack ]  |  |
|  +-----------------------------------------------------------------+  |
+-----------------------------------------------------------------------+

```

The FSF advocates for the term **GNU/Linux** to recognize that the userland runtime environment, standard C library, toolchains, and utilities were supplied by the GNU Project, while the kernel below manages raw hardware resources.

---

## 3. Major Kernel Version Milestones & Architectural Evolution

```
                 Linux Kernel Major Version Timeline

  1994          1996              1999-2001           2003            2011-2026+
   |             |                    |                 |                 |
   v             v                    v                 v                 v
[ v1.0 ] ----> [ v2.0 ] -----------> [ v2.2 / v2.4 ] -> [ v2.6 ] ------> [ v3.x - v6.x+ ]
 Monolithic    Initial SMP            Enterprise        CFS Sched,       Live-Patching,
 x86 Only      Alpha/SPARC Ports      USB, LFS          Preemption       Rust Language

```

### Linux 1.0 (March 1994)

* **Architecture Scope:** Single-processor (UP) Intel 80386 architectures.
* **Networking Infrastructure:** Basic implementation of TCP/IP protocol sockets using the BSD Sockets API model.
* **File Systems:** Minix FS support alongside the initial Extended File System (`ext` and `ext2`).
* **Memory Management:** Basic virtual memory management with paging to disk swap spaces and memory-mapped files via primitive `mmap()`.

### Linux 2.0 (June 1996)

* **Symmetric Multiprocessing (SMP):** Initial multi-processor execution support using a single **Big Kernel Lock (BKL)**.

```
                    Early SMP Architecture (Big Kernel Lock)

         CPU 0                                     CPU 1
  +------------------+                      +------------------+
  |  User Execution  |                      |  User Execution  |
  +------------------+                      +------------------+
           | System Call                             | System Call
           v                                         v
  +------------------+                      +------------------+
  |   Acquire BKL    |                      |  Blocked on BKL  |
  +------------------+                      +------------------+
           |                                         :
           v                                         :
  [ Inside Kernel ] <--- (Mutex Lock Boundary) .......

```

* **Multi-Architecture Expansion:** Ported to Non-x86 architectures including Digital Equipment Corporation (DEC) Alpha, SPARC, and MIPS.
* **Module Infrastructure:** Dynamic Kernel Module loading/unloading capabilities without requiring system reboots.

### Linux 2.2 – 2.4 Era (1999 – 2001)

* **Linux 2.2:** Improved SMP execution scalability, introduction of the Directory Cache (`dcache`), and expansion of supported architectures (PowerPC, ARM).
* **Linux 2.4:** Enterprise hardware scaling:
* Support for up to 64 GB of RAM on 32-bit systems using Physical Address Extension (PAE).
* Introduction of the USB bus subsystem, Logical Volume Manager (LVM), netfilter firewall architecture (`iptables`), and raw block I/O improvements.



### Linux 2.6 Revolution (December 2003)

The 2.6 kernel release transformed Linux into an enterprise-grade operating system through major architectural updates:

```
                      Linux 2.6 Scheduler Evolution

   O(1) Scheduler (Ingo Molnar)               Completely Fair Scheduler (CFS)
   - Dual Priority Arrays (Active/Expired)    - Red-Black Tree Data Structure
   - Hardcoded execution time slices          - Virtual runtime tracking (vruntime)
   - $O(1)$ complexity scaling                - $O(\log N)$ latency fairness scaling

```

* **Scheduler Breakthroughs:**
* **$O(1)$ Scheduler:** Introduced by Ingo Molnar, ensuring constant-time scheduling complexity regardless of task count.
* **Completely Fair Scheduler (CFS):** Merged in 2.6.23, using red-black trees to balance CPU time based on virtual runtime metrics.


* **Kernel Preemption:** Allowed high-priority task scheduling even while execution contexts were trapped inside kernel-space system call paths, reducing execution latencies.
* **Native POSIX Thread Library (NPTL):** Replaced legacy LinuxThreads with 1:1 threading models, leveraging the `clone()` system call and `NPTL` for low-overhead thread creation.
* **64-bit Maturation:** Full native optimization for AMD64 / Intel 64 systems.

### Modern Era (3.x to 6.x+)

| Milestone | Key Architectural Features & Paradigm Shifts |
| --- | --- |
| **Linux 3.x (2011)** | Adoption of time-based releases; integration of Btrfs, control groups (`cgroups`), and namespace architectures forming the runtime foundation for modern Linux containers. |
| **Linux 4.x (2015)** | Addition of Kernel Live Patching (`kpatch`/`kgraft`) without system reboots; introduction of extended Berkeley Packet Filters (`eBPF`) for direct in-kernel bytecode execution. |
| **Linux 5.x (2019)** | High-performance asynchronous I/O framework (`io_uring`); initial integration of memory-safe kernel drivers written in the Rust language (5.20 / 6.1). |
| **Linux 6.x+ (2022+)** | Advanced scheduler paradigms (e.g., `EEVDF` - Earliest Eligible Virtual Deadline First), expanded Rust language infrastructure inside main source directories, and support for emerging CPU instruction set architectures like RISC-V. |

---

## 4. Development Governance, Maintenance & Collaboration

### Linus Torvalds' Maintainer Hierarchy

The Linux kernel is maintained through a hierarchical governance structure designed to manage thousands of incoming patches per release cycle:

```
                       Kernel Maintainer Hierarchy

                    +-------------------------------+
                    |  Linus Torvalds (Mainline)    |
                    +-------------------------------+
                                    ^
                                    | Git Pull Requests
                    +-------------------------------+
                    |   Subsystem Maintainers       |
                    | (e.g., mm, net, ext4, arm64)  |
                    +-------------------------------+
                                    ^
                                    | Reviewed Commits / Pulls
                    +-------------------------------+
                    |  Sub-Maintainers / Reviewers  |
                    +-------------------------------+
                                    ^
                                    | Patch Emails / Signed-off-by
                    +-------------------------------+
                    | Individual Developers / LKM   |
                    +-------------------------------+

```

* **The Dictator & Benevolent Dictator Model:** Torvalds retains ultimate authority over mainline branch integration. Subsystem maintainers manage specific domains (e.g., networking, VFS, memory management).
* **Patch Flow & LKML:** Code submission traditionally flows through public mailing lists (e.g., the Linux Kernel Mailing List - `lkml`). Proposed patches undergo code review, sign-offs (`Signed-off-by:` tags), and testing before maintainers send pull requests up the hierarchy.

---

### Creation of Git (2005)

#### BitKeeper License Breakdown

From 1998 to 2002, the kernel project struggled to manage source code patches using traditional tarballs and patch files. In 2002, the kernel project adopted **BitKeeper**, a proprietary Distributed Version Control System (DVCS) that offered free usage tier access to open-source developers.

In early 2005, after reverse-engineering efforts targeting BitKeeper's networking protocols raised licensing concerns, BitKeeper's parent company revoked the community's free usage tier.

#### Design Goals of Git

In April 2005, Torvalds stepped away from kernel maintenance for weeks to write **Git** from scratch. He designed Git to address requirements specific to the Linux codebase:

```
                   Git Directed Acyclic Graph (DAG) Model

      Commit A (SHA-1) <--- Commit B (SHA-1) <--- Commit C (SHA-1)
                                                       ^
                                                       |
                                                  [ Branch: Main ]

```

1. **Distributed Model:** Every developer maintains a complete local clone of the repository history, eliminating centralized network bottlenecks.
2. **Cryptographic Integrity:** Use of SHA-1 hash functions to guarantee tree state integrity.
3. **Directed Acyclic Graph (DAG) Storage:** Storing commit objects as immutable nodes in a DAG, enabling branch, merge, and rebase operations.
4. **Performance at Scale:** Optimized for merging large codebases containing tens of thousands of individual files within seconds.

---

## 5. Historical Code Snippet: Early x86 Assembly Context Switch

The following assembly excerpt, adapted from early Linux releases (0.01/0.11), illustrates how the kernel executed context switches across tasks using the 80386 CPU's hardware Task State Segment (TSS) capabilities:

```assembly
/*
 * Early Linux Kernel (0.01/0.11) Task Switch Routine
 * Uses the 80386 Hardware 'ljmp' to a Task State Segment (TSS) Descriptor.
 */

.globl switch_to
.align 2

switch_to:
    pushl %ebp
    movl %esp,%ebp
    pushl %ecx
    pushl %edx
    pushl %ebx
    
    /* Fetch task array index parameter from the stack */
    movl 8(%ebp),%ecx
    
    /* Compare request against current running process pointer */
    cmpl %ecx,current
    je 1f
    
    /* Update current pointer to new process */
    xchgl %ecx,current
    
    /* 
     * Construct Far Jump Target to execute Hardware Task Switch.
     * The far jump reloads TR (Task Register) with the new process's TSS Selector.
     */
    ljmp *tmp_tss_target
    
1:  popl %ebx
    popl %edx
    popl %ecx
    popl %ebp
    ret

```

---

## Primary Historical Sources & Classical References

1. **Torvalds, L.** (1991, August 25). *"What would you like to see most in minix?"* Usenet post to `comp.os.minix`.
2. **Tanenbaum, A. S., & Torvalds, L.** (1992, January). *"LINUX is obsolete."* Usenet Debate Archives, `comp.os.minix`.
3. **Free Software Foundation.** (1991). *GNU General Public License, Version 2.* FSF Legal Docs.
4. **Torvalds, L.** (2005). *"Re: Git release notes / design goals."* Linux Kernel Mailing List (`lkml`).
5. **Molnar, I.** (2007). "
$$PATCH$$


 Completely Fair Scheduler (CFS)." Linux Kernel Mailing List (`lkml`).
6. **Bovet, D. P., & Cesati, M.** (2005). *Understanding the Linux Kernel (3rd ed.).* O'Reilly Media.