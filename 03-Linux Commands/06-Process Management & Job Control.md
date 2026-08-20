### SECTION 1: HISTORICAL ARCHITECTURE & FOUNDATIONAL PARADIGMS

#### 1. Unix Historical Context

Unix process management and job control evolved through three primary historic branches before reaching modern Linux and POSIX standardization:

```
[Unix V7 (1979)]
  │
  ├─► [BSD (4.1b / 4.2BSD)] ──► Job Control, csh, SIGTSTP/SIGCONT, process groups
  │
  ├─► [System V (SysV)]     ──► /proc (SVR4 via AT&T), semaphores, IPC, init runlevels
  │
  └─► [POSIX.1 (IEEE 1003.1)] ──► Standardized PID, PGID, SID, sigaction, tcsetpgrp
        │
        └─► [Linux Kernel] ──────► /proc filesystem (Linux 1.0+ via procfs), NPTL, clone()

```

* **Unix Version 7 (V7, 1979):** V7 established the fundamental process model: `fork()` to duplicate an execution context and `execve()` to replace the process image. Processes were isolated execution domains with simple linear execution models. Signaling was primitive and un-reliable (signals could be missed or race against handlers).
* **BSD Branch (4.1b / 4.2BSD):** BSD introduced robust signal handling (`sigvec`, later `sigaction`) and job control mechanics driven by Jim Kulp at IIASA. BSD added the concepts of Process Groups (`setpgrp()`), Controlling Terminals (`/dev/tty`), and signals like `SIGTSTP`, `SIGSTOP`, `SIGTTIN`, and `SIGTTOU`, allowing users to suspend foreground execution and manage background execution blocks via shell mechanics (`csh`).
* **System V (SysV - AT&T):** SysV prioritized enterprise scalability, inter-process communication (SysV IPC: semaphores, shared memory, message queues), and rigid operational init runlevels (`/etc/inittab`). Crucially, SVR4 (System V Release 4) introduced the virtual `/proc` filesystem (originally designed by Tom J. Killian for Unix Version 8 and implemented in SVR4 by Roger Faulkner and Ron Gomes), mapping live processes into file system representations where debuggers could manipulate process memory via file operations.
* **POSIX Standardization (POSIX.1 / IEEE 1003.1):** POSIX unified BSD job control and SysV process attributes. It formalized the strict hierarchical relationship between **Process IDs (PIDs)**, **Process Group IDs (PGIDs)**, and **Session IDs (SIDs)**, along with standard functions (`setpgid()`, `setsid()`, `tcsetpgrp()`) and reliable signal sets (`sigset_t`, `sigprocmask()`).
* **Linux Evolution:** Early Linux (0.01 to 1.0) adopted POSIX mechanics directly. Linux adopted a `/proc` filesystem layout inspired by SVR4, but expanded it extensively to expose deep kernel data structures as readable text files (`/proc/[pid]/stat`, `/proc/[pid]/status`, `/proc/meminfo`). Linux deviated from traditional Unix by implementing light-weight processes (LWPs) via the unified `clone()` system call, eventually standardizing thread implementation around the Native POSIX Thread Library (NPTL).

---

#### 2. The Process Abstraction

##### Threads vs. Processes in Linux

In the Linux kernel, there is **no strict architectural distinction** between a thread and a process. The Linux scheduler (`kernel/sched/`) schedules entities known as `task_struct` instances (often referred to as *tasks*).

* **Process:** A `task_struct` that owns its execution context exclusively (unique memory space, file descriptor table, signal handlers).
* **Thread:** A `task_struct` that shares its execution context (address space, file descriptors, signal table) with other tasks in the same thread group.

This is governed entirely by the execution flags passed to the underlying `clone()` system call (`sys_clone` or `clone3`):

```c
int clone(int (*fn)(void *), void *child_stack, int flags, void *arg, ...);

```

| Flag | Impact on `task_struct` Shared Resources |
| --- | --- |
| `CLONE_VM` | Shares the Virtual Memory address space (`mm_struct`). The child and parent execute in the same memory pages. Changes to memory via one thread are immediately visible to the other. |
| `CLONE_FILES` | Shares the File Descriptor Table (`files_struct`). If the child opens/closes a file descriptor (`fd`), the parent's table reflects the exact same state modification. |
| `CLONE_SIGHAND` | Shares Signal Handlers (`sighand_struct`). Both parent and thread share the disposition array mapping signal numbers to actions. (Requires `CLONE_VM`). |
| `CLONE_THREAD` | Places the child into the same **Thread Group** as the parent. The child shares the parent's Process ID (PID) as its Thread Group ID (TGID). |
| `CLONE_SYSVSEM` | Shares System V Semaphore undo values (`sysvsem`). |

```
Standard Process (fork()):
[Parent task_struct] ──► Copies mm_struct (Copy-On-Write), files_struct, sighand_struct
                             │
                             ▼
                     [Child task_struct] (Unique PID, Unique TGID)

POSIX Thread (pthread_create() via clone(CLONE_VM|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD)):
[Leader task_struct] ──┬──► Shares mm_struct ──────┐
                     ├──► Shares files_struct ────┼──► [Thread task_struct]
                     └──► Shares sighand_struct ──┘    (Unique TID, Same TGID)

```

##### The Task Struct (`struct task_struct`)

The `task_struct` (defined in `<linux/sched.h>`) is the central kernel data structure representing an execution context. It consumes ~2-4 KB of memory (depending on kernel configuration) and holds all meta-information required by the CPU scheduler, memory manager, file subsystem, and security mechanisms.

```
                    ┌──────────────────────────────────────────┐
                    │            struct task_struct            │
                    ├──────────────────────────────────────────┤
                    │ volatile long state                      │  <-- Execution state flags
                    │ void *stack                              │  <-- Pointer to kernel stack
                    │ refcount_t usage                         │
                    │ int flags                                │
                    │                                          │
                    │ int prio, static_prio, normal_prio       │  <-- Scheduler priorities
                    │ unsigned int rt_priority                 │
                    │ const struct sched_class *sched_class    │  <-- CFS / RT / Deadline ops
                    │ struct sched_entity se                   │  <-- CFS runtime accounting
                    │                                          │
                    │ struct mm_struct *mm                     │  <-- Memory map details
                    │ struct mm_struct *active_mm              │
                    │                                          │
                    │ pid_t pid                                │  <-- Kernel Thread ID (TID)
                    │ pid_t tgid                               │  <-- POSIX Process ID (PID)
                    │ struct task_struct __rcu *real_parent    │  <-- Parent process pointer
                    │ struct list_head children                │  <-- Head of child list
                    │ struct list_head sibling                 │  <-- Sibling linked list
                    │                                          │
                    │ struct files_struct *files               │  <-- Open file descriptors
                    │ struct signal_struct *signal             │  <-- Shared signal state
                    │ struct sighand_struct *sighand           │  <-- Signal action handlers
                    │ sigpending_t pending                     │  <-- Per-thread pending signals
                    └──────────────────────────────────────────┘

```

Primary Process State Flags:

* `TASK_RUNNING` (`0x00000000`): The task is either currently executing on a CPU core or sitting in a CPU runqueue waiting to be dispatched by the scheduler.
* `TASK_INTERRUPTIBLE` (`0x00000001`): The task is blocked (sleeping) waiting for an event or resource (e.g., socket read, disk I/O, semaphore lock). It will wake up prematurely if it receives a hardware interrupt or a kernel signal.
* `TASK_UNINTERRUPTIBLE` (`0x00000002`): The task is sleeping on a critical system resource (typically direct block I/O or page faults). It **will not handle signals** while in this state. It cannot be killed via `kill -9`.
* `TASK_STOPPED` (`0x00000004`): Execution has been halted explicitly via a job control signal (`SIGSTOP`, `SIGTSTP`, `SIGTTIN`, `SIGTTOU`).
* `TASK_TRACED` (`0x00000008`): The task's execution has been paused by a tracing tool like `ptrace` (e.g., `gdb` breakpoint or `strace`).
* `EXIT_ZOMBIE` / `TASK_ZOMBIE` (`0x00000020`): The task has terminated execution via `exit()`. Its memory mapping, file descriptors, and CPU contexts have been freed, but its metadata entry in the task table remains allocated so the parent can read its exit code via `waitpid()`.

##### PID, PPID, PGID, and SID Relationships

Linux arranges processes in a strict 4-level structural hierarchy for process isolation, terminal control, and signal broadcasting:

* **TID (Thread ID):** Unique identifier of a kernel execution thread (`task->pid`).
* **PID (Process ID):** Identifier of the process thread group leader (`task->tgid`). For single-threaded applications, `PID == TID`.
* **PPID (Parent Process ID):** The PID of the process that spawned this task using `fork()` or `clone()`.
* **PGID (Process Group ID):** The ID of a collection of one or more processes associated together (typically formed via shell pipes like `cat log.txt | grep error | wc -l`). The PGID equals the PID of the *Process Group Leader*. Signals sent to a negative PID (e.g., `kill -9 -1234`) are broadcast to all processes within PGID `1234`.
* **SID (Session ID):** A collection of process groups tied to a single controlling terminal (tty/pty). The SID equals the PID of the *Session Leader* (typically the login shell).

```
System Session Hierarchy (Terminal / Dev / tty1)
──────────────────────────────────────────────────────────────────────────────────────────
Session (SID: 1000) [Session Leader: bash (PID 1000)]
 │
 ├── Foreground Process Group (PGID: 2000)
 │     ├── Process 1 (PID: 2000, PPID: 1000) [cat log.txt]
 │     ├── Process 2 (PID: 2001, PPID: 1000) [grep "ERROR"]
 │     └── Process 3 (PID: 2002, PPID: 1000) [wc -l]
 │
 └── Background Process Group (PGID: 3000)
       ├── Process 4 (PID: 3000, PPID: 1000) [python worker.py &]
       └── Process 5 (PID: 3001, PPID: 3000) [worker_child]

```

---

#### 3. Init & Process Inheritance

##### PID 1 Evolution

When the Linux kernel finishes booting, it spawns user-space execution via `kernel_init()`, which executes the primary init binary assigned PID 1.

```
+---------------------------------------------------------------------------------------+
| SYSTEM V INIT                                                                         |
| - Imperative shell scripts executed sequentially via runlevels (/etc/rc.d/).          |
| - High latency, non-deterministic order, zero process supervision/recovery tracking.   |
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| UPSTART (Ubuntu / Event-driven)                                                       |
| - Asynchronous event loops listening for hardware/network state transitions.          |
| - Partial parallelization, but difficult state tracking for complex dependency graphs.|
+---------------------------------------------------------------------------------------+
                                           │
                                           ▼
+---------------------------------------------------------------------------------------+
| SYSTEMD                                                                               |
| - Declarative unit files (.service, .socket, .target).                               |
| - Explicit cgroup tracking (avoids process loss on fork bombs).                       |
| - Concurrent service execution via socket-activation (FD passing).                   |
+---------------------------------------------------------------------------------------+

```

##### Roles of PID 1:

1. **System Initialization & Service Management:** Boots and orchestrates background user-space daemons.
2. **Orphan Adoption:** When a parent process terminates before its child processes, those children become *orphans*. The kernel automatically re-parents orphaned processes to PID 1 (or a sub-reaper designated via `prctl(PR_SET_CHILD_SUBREAPER)`).
3. **Process Reaping:** When a child process terminates, it transitions to `TASK_ZOMBIE`. PID 1 executes an infinite event loop issuing `waitpid(-1, &status, WNOHANG)` to consume exit statuses and free remaining kernel `task_struct` slots.

##### Zombie Creation Mechanics (`exit()` vs `waitpid()`)

1. **Termination:** A process terminates by executing the `exit_group()` system call (or returning from `main()`).
2. **Kernel Cleanup (`do_exit()`):** The kernel sets `task->exit_code`, frees page tables (`exit_mm()`), closes file descriptors (`exit_files()`), and changes state to `EXIT_ZOMBIE`.
3. **Parent Notification:** The kernel sends a `SIGCHLD` signal to the parent process.
4. **Reaping (`wait4()` / `waitpid()`):** The parent catches `SIGCHLD` and executes `waitpid(child_pid, &status, 0)`. The kernel copies the exit code to the parent and frees the `task_struct` memory block entirely.

```
Parent                     Child               Kernel
  │                          │                   │
  │── fork() ───────────────►│                   │
  │                          │                   │
  │                          │── exit(0) ───────►│
  │                          │ [Terminates]      │ Releases mm_struct, FDs
  │                          │                   │ Sets state = EXIT_ZOMBIE
  │                          │                   │ Sends SIGCHLD to Parent
  │                          │                   │
  │◄── SIGCHLD ──────────────┼───────────────────│
  │                          │                   │
  │── waitpid() ─────────────┼──────────────────►│ Reads exit_code
  │                          │                   │ Deallocates task_struct
  ▼                          ▼                   ▼ [Zombie Reaped]

```

##### Degradation from Lingering Zombies

Zombies hold no user-space memory or open files. However, each zombie holds its entry inside the kernel's process table (a fixed-capacity data structure governed by `/proc/sys/kernel/pid_max`). If a leaky parent continually forks children that exit without a `waitpid()` call:

* The kernel exhausts available PIDs (`pid_max`).
* Subsequent attempts by any system user to issue `fork()`, spawn a subshell, or run commands fail immediately with `EAGAIN` ("Resource temporarily unavailable").

---

### SECTION 2: DEEP DIVE: PROCESS MONITORING & INTERNALS

#### 1. `ps aux` vs `ps -ef`

##### Parser Behavior: BSD Syntax vs. System V Syntax

The `ps` utility in Linux (from `procps-ng`) features a multi-syntax parser engineered to preserve backward compatibility across disparate Unix standards:

* **BSD Syntax (`ps aux`):** **No leading dashes**. Modifies the output columns and behavior according to traditional BSD conventions. `a` displays processes of all users, `u` selects user-oriented format (CPU/MEM percentages, start time, user name), and `x` includes processes without a controlling terminal (daemons).
* **System V Syntax (`ps -ef`):** **Requires leading dashes**. ` -e` selects all processes (equivalent to `-A`), and `-f` requests full-format listing (UID, PID, PPID, C, STIME, TTY, TIME, CMD).

Mixing dash conventions yields different parser states within `procps-ng`. For instance, `ps -aux` warns the user because `ps` interprets `-a` as System V "all processes excluding session leaders" and `-u` as a user selection flag expecting an argument, rather than BSD layout control!

##### `STAT` Column Symbol Matrix

The `STAT` column encodes the internal state of a process using single-letter codes:

| Symbol | Kernel State Encoding | Semantic Meaning |
| --- | --- | --- |
| **R** | `TASK_RUNNING` | Executing on CPU or sitting in scheduler runqueue. |
| **S** | `TASK_INTERRUPTIBLE` | Interruptible sleep; waiting for an event/signal. |
| **D** | `TASK_UNINTERRUPTIBLE` | Uninterruptible sleep; waiting for direct block I/O. |
| **Z** | `EXIT_ZOMBIE` | Dead process; awaiting parent `waitpid()` reaping. |
| **T** | `TASK_STOPPED` | Stopped by job control signal (`SIGSTOP`/`SIGTSTP`). |
| **t** | `TASK_TRACED` | Paused under active tracer inspection (`ptrace`). |
| **<** | Priority Flag | High priority (nice level less than 0). |
| **N** | Priority Flag | Low priority (nice level greater than 0). |
| **L** | Memory Flag | Has pages locked directly into RAM memory (`mlock`). |
| **s** | Session Flag | Is a Session Leader (PID == SID). |
| **l** | Threading Flag | Is multi-threaded (uses multiple `task_struct` in thread group). |
| **+** | Foreground Flag | In the foreground Process Group of controlling terminal. |

##### `/proc/[pid]/` Parsing Breakdown

To construct output, `ps` traverses the `/proc` directory, scanning numeric subdirectories (`/proc/1/`, `/proc/2/`, etc.) and parsing specific pseudo-files:

```
           ┌──────────────────────────────────────────────┐
           │                  ps Engine                   │
           └──────────────────────────────────────────────┘
              ▲            ▲              ▲             ▲
              │            │              │             │
 Parsing:     │            │              │             │
    ┌─────────┴──────┐ ┌───┴──────────┐ ┌─┴───────────┐ ┌┴───────────┐
    │ /proc/[pid]/   │ │ /proc/[pid]/ │ │ /proc/[pid]/│ │ /proc/[pid]/│
    │     stat       │ │   cmdline    │ │   status    │ │    statm    │
    └────────────────┘ └──────────────┘ └─────────────┘ └─────────────┘

```

1. `/proc/[pid]/stat`: Formatted as a single-line space-separated metric list. Provides instantaneous process metrics parsed directly by `ps`:
* Field 1: `pid`
* Field 2: `tcomm` (executable path enclosed in parentheses)
* Field 3: `state` (`R`, `S`, `D`, `Z`, etc.)
* Field 4: `ppid`
* Field 5: `pgrp` (Process Group ID)
* Field 6: `session` (Session ID)
* Field 14: `utime` (CPU time spent in user mode, in clock ticks)
* Field 15: `stime` (CPU time spent in kernel mode, in clock ticks)
* Field 18: `priority` (Kernel scheduling priority)
* Field 19: `nice` (Nice value, -20 to 19)
* Field 20: `num_threads` (Count of active execution threads)
* Field 22: `starttime` (Time process started after system boot)


2. `/proc/[pid]/cmdline`: Contains the full command-line arguments used to launch the process, stored as a null-byte (`\0`) separated sequence. `ps` converts null bytes to spaces for printing. If empty, the task is a kernel thread (displayed in brackets, e.g., `[kworker/0:1H]`).
3. `/proc/[pid]/status`: human-readable plain text representation of `stat`, providing `Uid` (real, effective, saved, filesystem UID) and `Gid` metadata along with human-readable memory footprints (`VmSize`, `VmRSS`, `SigPnd`, `SigBlk`).
4. `/proc/[pid]/statm`: Provides memory metrics in terms of system memory pages (typically 4096 bytes):
* Field 1: `size` (Total virtual memory size)
* Field 2: `resident` (Resident Set Size - RSS)
* Field 3: `shared` (Shared memory pages)



---

#### 2. `top` / `htop` / `btop`

##### Interactive Metric Calculation Mechanics

Tools like `top`, `htop`, and `btop` calculate non-blocking delta metrics by executing non-destructive polling loops over discrete time intervals $\Delta t$ (typically defaulting to 1.5 or 3.0 seconds).

```
Sample Interval T1                                        Sample Interval T2
┌────────────────────────────────┐                        ┌────────────────────────────────┐
│ Read /proc/stat -> Total CPU   │                        │ Read /proc/stat -> Total CPU   │
│ Read /proc/[pid]/stat -> utime │ ── Wait Δt Seconds ──► │ Read /proc/[pid]/stat -> utime │
│                          stime │                        │                          stime │
└────────────────────────────────┘                        └────────────────────────────────┘
                                                                           │
                                                                           ▼
                                                    Compute ΔTotal_CPU and ΔProcess_Ticks
                                                    CPU % = (ΔProcess_Ticks / ΔTotal_CPU) * 100

```

1. **System-Wide CPU Sampling:** Reads `/proc/stat`, which exposes total system clock ticks spent across execution states: `user`, `nice`, `system`, `idle`, `iowait`, `irq`, `softirq`, `steal`. Summing these gives $T_{\text{total\_cpu\_ticks}}(t_1)$.
2. **Process CPU Sampling:** Reads `/proc/[pid]/stat` to fetch $u_{\text{time}}(t_1)$ and $s_{\text{time}}(t_1)$ for every process.
3. **Delta Calculation:** At time $t_2$, it repeats the step.

$$\Delta T_{\text{total}} = T_{\text{total\_cpu\_ticks}}(t_2) - T_{\text{total\_cpu\_ticks}}(t_1)$$


$$\Delta T_{\text{proc}} = \left(u_{\text{time}}(t_2) + s_{\text{time}}(t_2)\right) - \left(u_{\text{time}}(t_1) + s_{\text{time}}(t_1)\right)$$


$$\text{CPU Usage \%} = \left( \frac{\Delta T_{\text{proc}}}{\Delta T_{\text{total}}} \right) \times 100 \times N_{\text{cores}}$$



This calculation uses lightweight integer arithmetic, avoiding significant overhead during process table iteration.

##### System Load Averages

System Load Averages (exposed via `/proc/loadavg` and rendered in `top` headers) represent the average number of tasks in a runnable or queuing state over 1, 5, and 15-minute windows.

Linux calculates this using an **exponentially damped moving average** algorithm updated once every 5 seconds.

$$\text{Load}(t) = \text{Load}(t - \Delta t) \cdot e^{-\Delta t / \tau} + n \cdot \left(1 - e^{-\Delta t / \tau}\right)$$

Where $\tau$ is the decay constant (1, 5, or 15 minutes) and $n$ is the instantaneous count of active tasks.

Unlike traditional Unix variants (which only count tasks in `TASK_RUNNING`), **Linux includes tasks in the `TASK_UNINTERRUPTIBLE` (`D` state) in load average calculations**.

* **Why?** In Linux, tasks blocked on critical disk/NVMe hardware queues or network filesystem lock-ups (`NFS`) represent real physical demand on system hardware.
* **Inflation Consequence:** A process stuck indefinitely in `D` state (due to a unresponsive SAN or dead storage array) adds `1.0` to the load average calculation indefinitely, even though actual CPU usage remains at 0.0%.

##### Terminal Rendering Engine Differences

* `top`: Uses standard `termios` library and ANSI terminal escape codes to re-render lines sequentially, minimizing memory footprints for headless server environments.
* `htop`: Employs `ncurses` (new curses) window manipulation library. Allocates double-buffered terminal window screens in memory, enabling sub-window scrolling, mouse click interception (via xterm mouse mode escapes), and fine-grained colored string attributes.
* `btop`: Uses C++ custom low-level TUI rendering frameworks utilizing direct UTF-8 vector canvas drawing (using Braille and block characters for micro-graphs) and direct non-blocking input event loops polling `/proc` and `/sys` directly via native platform bindings.

---

#### 3. `pstree -p`

##### Tree Traversal Algorithm

`pstree` parses all numeric directories under `/proc` once, constructing an in-memory directed tree structure using node objects.

```
       [PID 1: systemd]
       ├── [PID 800: NetworkManager]
       └── [PID 1200: sshd]
             └── [PID 2500: sshd (user)]
                   └── [PID 2501: bash]
                         ├── [PID 3100: ps]
                         └── [PID 3101: pstree]

```

1. **Scanning:** Iterates over `/proc/[pid]/stat`. Reads `pid` (Field 1) and `ppid` (Field 4).
2. **Node Allocation:** Creates a dynamically allocated node:
```c
struct proc_node {
    pid_t pid;
    pid_t ppid;
    char comm[256];
    struct proc_node *parent;
    struct proc_node *first_child;
    struct proc_node *next_sibling;
};

```


3. **Linking Phase:** It resolves parent pointers. For a process with PID $P$ and PPID $Q$:
* It finds Node $Q$. Sets Node $P$'s `parent` pointer to Node $Q$.
* Appends Node $P$ to Node $Q$'s linked list of children via `first_child` and `next_sibling` pointers.


4. **DFS Tree Printing:** Performs a **Depth-First Search (DFS)** starting from Root (PID 1). Reconstructs branch characters (`├─`, `└─`) dynamically based on sibling existence. `-p` enables printing the numeric `pid` value in trailing parentheses next to the executable node name.

---

#### 4. `pgrep -l process_name`

##### Internal Traversal Engine vs Signal Handling

`pgrep` does not interact with the signal system; it is a search engine for process metadata. It reads directory handles via `opendir("/proc")`, looping through entries with `readdir()`.

##### `/proc/[pid]/cmdline` vs `/proc/[pid]/comm`

When matching process names, `pgrep` behaves differently based on target sources:

```
Process Execution: "python3 /usr/bin/ansible-playbook site.yml"

  /proc/[pid]/comm    ──► "python3" (Truncated to TASK_COMM_LEN: 16 bytes)
  /proc/[pid]/cmdline ──► "python3\0/usr/bin/ansible-playbook\0site.yml"

```

* **Default Behavior (`pgrep process_name`):** Reads `/proc/[pid]/comm`. The kernel limits `comm` to 16 characters (`TASK_COMM_LEN`). Long process names (or scripts executed via interpreters) are truncated here. `pgrep` matches against this short process name string.
* **Full Command Line Matching (`pgrep -f process_name`):** Forces `pgrep` to read `/proc/[pid]/cmdline`. This allows matching against full invocation parameters, pathnames, or interpreter scripts (e.g., finding `ansible-playbook` when the binary executed is `python3`).
* **Output Flag (`-l`):** Prints the process name alongside the matching PID.

---

### SECTION 3: JOB CONTROL & TERMINAL SIGNALS

#### 1. POSIX Job Control Mechanics

##### The Controlling Terminal (tty/pty)

A POSIX Controlling Terminal (represented by an OS character device file like `/dev/pts/2`) manages input/output distribution between physical/virtual serial ports and software process groups.

* A session can have at most **one** controlling terminal.
* The controlling terminal tracks a single process group as the **Foreground Process Group**. All other process groups within the session are in the **Background**.

```
Controlling Terminal (/dev/pts/1)
  │
  ├─ Foreground Process Group (PGID 5000) ◄── Reads STDIN, Receives ^C (SIGINT)
  │    └── [vim file.txt]
  │
  └─ Background Process Group (PGID 5001) ◄── Suspended on STDIN read (SIGTTIN)
       └── [build_job.sh]

```

##### Roles of `SIGTTIN` and `SIGTTOU`

To prevent background jobs from corrupting terminal states or interleaving output:

1. **`SIGTTIN` (Signal Terminal Input):** If a background process attempts to read from its controlling terminal standard input (`read(STDIN_FILENO)`), the kernel terminal driver intercepts the read operation, suspends the background process, and dispatches a `SIGTTIN` signal to its process group. The process transitions to `TASK_STOPPED`.
2. **`SIGTTOU` (Signal Terminal Output):** If the terminal `termios` flag `TOSTOP` is set, any attempt by a background process to write to standard output (`write(STDOUT_FILENO)`) triggers the terminal driver to interrupt the execution with a `SIGTTOU` signal, halting the background process.

---

#### 2. Command Mechanics

##### `jobs`

The shell manages an internal array structure mapping small integer handles (`%1`, `%2`) to Process Group IDs:

```c
typedef struct job {
    int job_index;       /* %1, %2 */
    pid_t pgid;          /* Process group ID */
    char *command_line;  /* String representation */
    int state;           /* RUNNING, STOPPED, DONE */
} job_t;

```

When a user issues `jobs`, the shell iterates over its internal job list, issuing non-blocking checks (`waitpid(-1, &status, WNOHANG|WUNTRACED|WCONTINUED)`) to synchronize and print the status of each managed process group.

##### `bg %1` and `fg %1`

* `bg %1`:
1. Looks up `%1` in the job table to get its target PGID (e.g., `4500`).
2. Executes the system call `kill(-4500, SIGCONT)`.
3. Updates its internal job state to `RUNNING`. The process group executes in the background.


* `fg %1`:
1. Retrieves target PGID for `%1`.
2. Calls `tcsetpgrp(tty_fd, pgid)` via system call `ioctl(tty_fd, TIOCSPGRP, &pgid)`. This transfers foreground ownership of `/dev/pts/X` to target PGID.
3. Issues `kill(-pgid, SIGCONT)` to resume execution if suspended.
4. Calls `waitpid(-pgid, &status, WUNTRACED)` — the shell blocks itself until the foreground process group suspends or terminates.



```
       [Shell (PGID 1000)]               [Background Job (PGID 2000)]
                │                                    │
 fg %1          │                                    │
 ─────────────► │ 1. tcsetpgrp(tty_fd, 2000)         │
                │ ─────────────────────────────────► │ Terminal gives FD access
                │ 2. kill(-2000, SIGCONT)            │
                │ ─────────────────────────────────► │ Resumes execution
                │ 3. waitpid(-2000, WUNTRACED)       │
                │ [Shell Sleeps]                     │ [Executes in Foreground]

```

##### `nohup command &`

When a controlling terminal closes or disconnects, the kernel sends a `SIGHUP` (Signal Hangup) to the session leader, which broadcasts `SIGHUP` to all child process groups, causing their termination.

`nohup` modifies this process lifecycle:

1. **Signal Disposition Adjustment:** Wraps `command` by setting its handling of `SIGHUP` to `SIG_IGN` (Ignore) via `sigaction()`.
2. **I/O Redirection:** Checks if `stdout` is connected to a terminal. If so, redirects file descriptor 1 to append to `nohup.out` (or `$HOME/nohup.out`). Does the same for `stderr` (fd 2).
3. **Terminal Detachment:** Execs the command. When the terminal closes, the process ignores the broadcast `SIGHUP` and continues execution, adopted by PID 1 when its parent shell exits.

---

### SECTION 4: PROCESS SIGNALING MECHANICS

#### 1. Kernel Signal Subsystem

##### Signal Representation Inside Kernel

Signals are asynchronous notifications processed by the kernel on behalf of tasks. Within `task_struct`, signals are tracked via several fields:

```c
struct task_struct {
    ...
    sigpending_t pending;              /* Per-thread pending signals */
    struct signal_struct *signal;      /* Shared thread-group pending signals */
    struct sighand_struct *sighand;    /* Signal action definitions */
    sigset_t blocked;                  /* Signal mask (blocked signals) */
    ...
};

```

* `sigset_t`: Implemented as a bitmask array (64 bits on 64-bit Linux) where each bit maps to a specific signal number (1 to 64). Bit 1 = `SIGHUP`, Bit 9 = `SIGKILL`, etc.
* **Signal Queues:** Standard signals (1-31) cannot be queued; sending a duplicate signal that is already pending drops the secondary signal. Real-time signals (32-64) are queued using dynamically allocated structures (`struct sigqueue`).

##### Signal Delivery Execution Cycle

Signals are delivered when a task returns from a kernel boundary (system call execution or hardware interrupt) back to user space:

```
[User Mode Execution]
       │  System Call / Hardware Interrupt
       ▼
[Kernel Mode Processing]
       │
       ▼
[Signal Delivery Check] ──► Reads pending bitmask & (~blocked bitmask)
       │
       ├─► Signal Found?
       │     ├─► Default Action (e.g., SIGKILL / SIGSTOP) ──► Kernel handles directly
       │     └─► Custom User Handler Hook
       │           │
       │           ├─► Modifies Kernel User Stack Frame (rt_sigframe)
       │           ├─► Sets Instruction Pointer (EIP/RIP) to User Signal Handler Address
       │           └─► Sets Return Address to sa_restorer (sys_rt_sigreturn)
       │
       ▼
[Return to User Mode] ────► Executes Custom Handler ──► sys_rt_sigreturn ──► Resumes normal code

```

---

#### 2. Command Breakdown

##### `kill -9 PID` (`SIGKILL` vs `SIGTERM`)

* `SIGTERM` (Signal 15): Requests process termination. The target process can catch the signal, invoke cleanup routines (flushing file buffers, removing temporary locks, closing network sockets), or choose to ignore it entirely.
* `SIGKILL` (Signal 9): **Forces process termination**.

```
                           SIGTERM (15) vs SIGKILL (9)
                       
   SIGTERM (15)                                    SIGKILL (9)
   ┌──────────┐                                    ┌──────────┐
   │ Process  │                                    │ Process  │
   └────┬─────┘                                    └────┬─────┘
        │ Catches signal                                │ CANNOT be caught,
        ▼                                               │ blocked, or ignored.
   ┌───────────────────────┐                            │
   │ Executes cleanup code │                            ▼
   └────┬──────────────────┘                       ┌───────────────────────┐
        │                                          │ Kernel invokes        │
        ▼                                          │ do_group_exit()       │
   ┌───────────────────────┐                       │ Forces teardown       │
   │ Exits gracefully      │                       └───────────────────────┘
   └───────────────────────┘

```

When `SIGKILL` is delivered, the kernel bypasses user space handlers completely. The kernel's signal handling loop checks the bitmask, identifies bit 9 (`SIGKILL`), and calls `do_group_exit()`. The task is forced into termination immediately.

* **Exceptions:** `SIGKILL` cannot kill a process locked in `TASK_UNINTERRUPTIBLE` (`D` state) because the task cannot check for pending signals until its block device I/O operation returns.

##### `killall process_name` vs `pkill -u username`

Both utilities search `/proc` to build execution candidate lists before issuing signals via the `kill()` syscall.

```c
int kill(pid_t pid, int sig);

```

* `killall process_name`: Matches targets by process name against `/proc/[pid]/comm` or command lines.
* `pkill -u username`: Resolves `username` to a numerical UID via `/etc/passwd` parsing (`getpwnam()`), scanning `/proc/[pid]/status` for lines starting with `Uid:` to issue signals to matching processes.

##### Race Conditions & Fork Bombs

When running `killall` or `pkill` during an active fork bomb (a process rapidly spawning sub-processes):

1. `killall` reads `/proc` sequentially.
2. While `killall` iterates through PIDs 1000–2000, the fork bomb rapidly creates new child processes at PIDs 2001–5000.
3. By the time `killall` completes its scan, new processes exist that were missed during the initial `/proc` iteration, allowing the fork bomb to survive.

##### `pidof service`

`pidof` resolves binary names to running Process IDs:

1. Iterates over `/proc/[pid]/`.
2. Follows the executable symbolic link at `/proc/[pid]/exe` using `readlink()`.
3. Compares the resolved path to the target binary path provided in the arguments (e.g., verifying `/proc/1234/exe` points to `/usr/sbin/sshd`).
4. Returns a space-separated string of all matching PIDs.

---

### SECTION 5: SCHEDULING, PRIORITY, AFFINITY & RESOURCE LIMITS

#### 1. Linux Scheduler Internals

##### The CFS and EEVDF Schedulers

Linux historically used the **Completely Fair Scheduler (CFS)**, replaced in modern kernels (6.6+) by the **Earliest Eligible Virtual Deadline First (EEVDF)** scheduler.

```
       Red-Black Tree Runqueue (rb_node)
                      
                     [vruntime = 105]
                        /        \
                       /          \
            [vruntime = 98]     [vruntime = 112]
               /      \
              /        \
     [vruntime = 90]  [vruntime = 102]
         ▲
         │
  Leftmost Node
  (Next to execute on CPU)

```

Both schedulers balance CPU execution time across tasks using a metric called **Virtual Runtime (`vruntime`)**:

* `vruntime` measures the normalized amount of CPU time consumed by a task.
* Tasks are tracked in a time-ordered **Red-Black Tree** (`rb_node` structures).
* The scheduler selects the leftmost node of the red-black tree (the task with the smallest `vruntime`) to run next.

##### Nice Values & Virtual Runtime Scaling

The **Nice Value** (ranging from `-20` to `19`) acts as a scaling weight on `vruntime` progression.

The kernel maps nice values to scheduling weight values using an internal lookup array (`sched_prio_to_weight`):

$$\text{vruntime}_{\text{delta}} = \text{physical\_time}_{\text{delta}} \times \left( \frac{\text{NICE\_0\_LOAD}}{\text{task\_weight}} \right)$$

| Nice Level | Weight Value | `vruntime` Accumulation Rate | CPU Allocation Allocation (vs Nice 0) |
| --- | --- | --- | --- |
| **-20** (High Priority) | 88761 | ~1/88x (Extremely Slow) | ~88x more CPU time |
| **0** (Default) | 1024 | 1x (Normal) | Baseline |
| **19** (Low Priority) | 15 | ~68x (Extremely Fast) | ~1/68th CPU time |

Tasks with a **lower nice value** (higher priority) have a higher execution weight, meaning their `vruntime` grows more slowly. As a result, they remain on the left side of the red-black tree longer and receive more CPU execution time.

---

#### 2. Priority Management

##### `nice` vs `renice` System Call Interaction

* `nice -n -10 command`: Spawns a **new** process with an adjusted nice value. Executes `nice()` or `setpriority(PRIO_PROCESS, 0, -10)` before executing `execve()`.
* `renice -n 5 -p PID`: Alters the priority of an **already running** process. Issues `setpriority(PRIO_PROCESS, PID, 5)`. The kernel updates the task's dynamic weight and recalculates its position in the scheduler's red-black tree.

##### Capability Restrictions (`CAP_SYS_NICE`)

Unprivileged users can only **increase** the nice value of their processes (lowering priority, making them "nicer" to other users).

Setting a negative nice value (increasing CPU priority) requires the POSIX capability **`CAP_SYS_NICE`** (held by `root` by default). This prevents unprivileged users from starving critical system processes of CPU time.

---

#### 3. Real-Time Scheduling

##### Real-Time Scheduling Policies

Linux supports real-time scheduling policies defined by POSIX.1-2001, which take precedence over normal scheduling policies (`SCHED_OTHER`, `SCHED_BATCH`, `SCHED_IDLE`):

```
Priority Tier Overview:
+-------------------------------------------------------------------------------+
| Real-Time Policies (Priorities 1 - 99, Highest)                               |
| - SCHED_DEADLINE : Earliest deadline first execution guarantee.               |
| - SCHED_FIFO     : First-In, First-Out (runs until preempted or yields).       |
| - SCHED_RR       : Round-Robin (time-sliced real-time execution).             |
+-------------------------------------------------------------------------------+
                                       │
                                       ▼
+-------------------------------------------------------------------------------+
| Normal / Standard Policies (Priority 0, Lowest)                               |
| - SCHED_OTHER    : Standard CFS/EEVDF time-sharing scheduler.                 |
+-------------------------------------------------------------------------------+

```

* `SCHED_FIFO` (First-In, First-Out): Runs until it explicitly yields CPU execution (`sched_yield()`), blocks on I/O, or is preempted by a higher-priority real-time task. It does not use time-slicing.
* `SCHED_RR` (Round-Robin): Similar to `SCHED_FIFO`, but tasks at the same priority level share CPU time using fixed time-slices.
* `SCHED_DEADLINE`: Uses Earliest Deadline First (EDF) scheduling based on three parameters: Runtime, Deadline, and Period.

##### Commands and Kernel Preemption

`chrt -f -p 99 PID` sets process `PID` to policy `SCHED_FIFO` (`-f`) with real-time priority `99` using the system call:

```c
sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);

```

When a task with a real-time policy becomes runnable, it **preempts** any running standard task (`SCHED_OTHER`) immediately, regardless of that task's nice value.

---

#### 4. CPU Affinity

##### Bitmask Mechanics

CPU affinity pins a process to execution on specific logical CPU cores. Kernel task affinity is represented by a bitmask data structure (`cpu_set_t`).

```
CPU Core Mapping:  Core 3 | Core 2 | Core 1 | Core 0
Bitmask Value:       0        0        1        1     = 0x03 (Hex)

```

Executing `taskset -cp 0,1 PID` pins process `PID` to CPU cores 0 and 1 by calling:

```c
int sched_setaffinity(pid_t pid, size_t cpusetsize, const cpu_set_t *mask);

```

##### NUMA Architecture & Cache Invalidation Implications

```
NUMA Node 0                               NUMA Node 1
┌───────────────────────────────┐         ┌───────────────────────────────┐
│ [CPU Core 0]   [CPU Core 1]   │         │ [CPU Core 2]   [CPU Core 3]   │
│   └── L1/L2 Cache ──┘         │         │   └── L1/L2 Cache ──┘         │
│ ┌───────────────────────────┐ │ Inter-  │ ┌───────────────────────────┐ │
│ │ Local RAM (Socket 0)      │ │ connect │ │ Local RAM (Socket 1)      │ │
│ └───────────────────────────┘ │◄───────►│ └───────────────────────────┘ │
└───────────────────────────────┘         └───────────────────────────────┘

```

On Non-Uniform Memory Access (NUMA) systems:

* **Cache Invalidation:** Bouncing a process across CPU cores on different physical sockets invalidates L1 and L2 processor caches, forcing costly cache line reloads.
* **Cross-Node Memory Latency:** If a process running on NUMA Node 1 accesses memory allocated on NUMA Node 0, memory requests must travel over an interconnect (e.g., AMD Infinity Fabric, Intel UPI), increasing memory latency.

Configuring CPU affinity using `taskset` keeps processes localized to specific cores and NUMA nodes, improving cache hits and memory access performance for performance-critical applications.

---

#### 5. Dynamic Resource Limits

##### Soft vs. Hard Limits (`struct rlimit`)

Process resource limits govern constraints like maximum open files, stack sizes, and CPU execution limits. They are enforced via the `rlimit` structure:

```c
struct rlimit {
    rlim_t rlim_cur;  /* Soft limit: Soft operational ceiling */
    rlim_t rlim_max;  /* Hard limit: Hard ceiling (ceiling for soft limit) */
};

```

```
0 ─────────────────────► Soft Limit ────────────────────► Hard Limit ──► ∞
                         (Warning/Error raised)           (Privileged escalation required)

```

* **Soft Limit (`rlim_cur`):** The current effective limit for the process. When a process exceeds this limit, the kernel issues an error (e.g., returning `EMFILE` on `open()` when hitting `RLIMIT_NOFILE`). A process can increase its soft limit up to its hard limit.
* **Hard Limit (`rlim_max`):** The absolute maximum threshold for the soft limit. Non-privileged processes can lower their hard limit, but **cannot increase it** without the `CAP_SYS_RESOURCE` capability.

##### Modifying Limits at Runtime (`prlimit`)

Historically, updating limits required configuring limits in `/etc/security/limits.conf` (parsed by `pam_limits.so` during session creation), which required restarting the service to take effect.

Modern Linux systems use the `prlimit64` system call to view and update resource limits for **live, running processes** dynamically:

```c
int prlimit(pid_t pid, int resource, const struct rlimit *new_limit, struct rlimit *old_limit);

```

Executing `prlimit --pid=1234 --nofile=4096` updates the `RLIMIT_NOFILE` entry in process 1234's `task_struct` immediately, expanding its maximum open file descriptor limit without requiring a service restart.