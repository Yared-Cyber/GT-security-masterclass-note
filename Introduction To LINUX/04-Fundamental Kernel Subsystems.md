# 04-Fundamental Kernel Subsystems

---

## 1. Process Management & CPU Scheduling

### Process Abstractions & Data Structures

In the Linux kernel, every thread of execution is represented by an instance of `struct task_struct` (defined in `<linux/sched.h>`). Linux implements a 1:1 threading model (NPTL - Native POSIX Thread Library), where both process leaders and individual threads are represented as distinct `task_struct` instances possessing unique Thread IDs (`PID` in kernel space) while sharing a common Thread Group ID (`TGID`, which corresponds to `PID` in POSIX user space).

```
                  +-----------------------------------+
                  |        struct task_struct         |
                  +-----------------------------------+
                  | volatile long state               |
                  | void *stack (thread_info)         |
                  | struct mm_struct *mm              |
                  | struct mm_struct *active_mm       |
                  | pid_t pid (Thread ID)             |
                  | pid_t tgid (Process ID)           |
                  | const struct cred __rcu *cred     |
                  | struct files_struct *files        |
                  | struct fs_struct *fs              |
                  | struct signal_struct *signal      |
                  | struct sighand_struct *sighand    |
                  | struct sched_entity se            |
                  | struct sched_rt_entity rt         |
                  | struct sched_dl_entity dl         |
                  | struct task_struct __rcu *parent  |
                  | struct list_head children         |
                  +-----------------------------------+

```

#### Key `task_struct` Members

* **`state` / `__state**`: Bitmask representing execution status (`TASK_RUNNING`, `TASK_INTERRUPTIBLE`, `TASK_UNINTERRUPTIBLE`, `__TASK_STOPPED`, `TASK_TRACED`).
* **`stack`**: Pointer to the kernel-mode stack (typically 16KB on x86-64, allocated via the Page Allocator). On modern x86-64 kernels with `CONFIG_THREAD_INFO_IN_TASK=y`, `struct thread_info` sits at the top of `task_struct` rather than at the bottom of the kernel stack, containing low-level architecture flags (`TIF_NEED_RESCHED`, `TIF_SIGPENDING`).
* **`mm` / `active_mm**`: Pointer to `struct mm_struct` defining the process virtual address space. Kernel threads have `mm == NULL` and temporarily borrow the active address space of the previously running process via `active_mm` during context switches to avoid unnecessary TLB flushes.
* **Credentials (`cred`)**: RCUs-protected pointer to `struct cred` containing real/effective/saved UID/GID and POSIX capabilities (`cap_inheritable`, `cap_permitted`, `cap_effective`).
* **Subsystem Pointers**:
* `files_struct *files`: File descriptor table (`fdtable`).
* `fs_struct *fs`: Filesystem context (root and working directory dentries/vfsmounts).
* `signal_struct *signal` & `sighand_struct *sighand`: Process-wide signal handling definitions, pending signal queues, and disposition actions.



#### Creation Primitives: `fork()`, `vfork()`, and `clone()`

All process creation primitives route internally to `kernel_clone()` (which replaced `_do_fork()` in recent kernels) located in `kernel/fork.c`.

```
sys_fork()     ---> kernel_clone(SIGCHLD, 0, ...)
sys_vfork()    ---> kernel_clone(CLONE_VFORK | CLONE_VM | SIGCHLD, 0, ...)
sys_clone()    ---> kernel_clone(flags, stack_start, ...)
sys_clone3()   ---> kernel_clone(&clone_args)

```

##### Flag Semantics

* **`CLONE_VM`**: Parent and child share the same address space (`mm_struct`). Used for POSIX threads.
* **`CLONE_FS`**: Parent and child share filesystem information (root directory, current working directory, and umask).
* **`CLONE_FILES`**: Parent and child share the file descriptor table (`struct files_struct`). Open/close events and file offsets reflect across both.
* **`CLONE_SIGHAND`**: Parent and child share signal handlers and dispositions. Requires `CLONE_VM`.
* **`CLONE_THREAD`**: Places the child in the same thread group as the parent (`tgid` is duplicated). Requires `CLONE_SIGHAND`.
* **`CLONE_VFORK`**: Suspends the parent execution until the child releases the virtual memory space via `execve()` or `_exit()`.

```c
/* Simplified internals of kernel_clone() */
pid_t kernel_clone(struct kernel_clone_args *args) {
    struct task_struct *p;
    
    /* Copy task structure and apply copy-on-write semantics */
    p = copy_process(NULL, 0, args->script_param, args);
    if (IS_ERR(p))
        return PTR_ERR(p);

    struct pid *pid = get_task_pid(p, PIDTYPE_PID);
    pid_t nr = pid_vnr(pid);

    if (args->flags & CLONE_VFORK)
        init_completion(&vfork);

    /* Enqueue child to runqueue */
    wake_up_new_task(p);

    if (args->flags & CLONE_VFORK)
        wait_for_vfork_done(p, &vfork); /* Parent sleeps */

    return nr;
}

```

---

### Kernel Schedulers

#### Evolution

1. **$O(n)$ Scheduler (Linux 2.4 and earlier)**: Iterated over all runnable tasks in a single global list per schedule call. Scaled poorly with high task counts and produced heavy global lock contention.
2. **$O(1)$ Scheduler (Ingo Molnar, Linux 2.6.0 - 2.6.22)**: Introduced dual runqueue arrays (`active` and `expired`) per CPU with priority bitmapping. Ensured constant time execution regardless of task count, but relied on complex interactive heuristics to calculate dynamic priorities, causing latency regressions for desktop workloads.
3. **Completely Fair Scheduler (CFS, Linux 2.6.23 - 6.5)**: Replaced priority heuristics with the concept of Red-Black tree virtual runtime tracking.
4. **EEVDF Scheduler (Earliest Eligible Virtual Deadline First, Peter Zijlstra, Linux 6.6+)**: Replaced CFS heuristics to address latency lag and strict slice guarantee issues.

#### CFS Mechanics (`kernel/sched/fair.c`)

CFS models an "ideal multi-tasking CPU" on hardware by tracking the normalized execution time spent by each process, stored in `se.vruntime` within `struct sched_entity`.

##### Virtual Runtime Formulation

When a task runs for physical time $\Delta \text{exec\_time}$, its `vruntime` advances inversely proportional to its nice weight:

$$\Delta \text{vruntime} = \Delta \text{exec\_time} \times \frac{\text{NICE\_0\_LOAD}}{\text{weight}}$$

Where `NICE_0_LOAD` is 1024. The mapping from `nice` values (-20 to +19) to weights is governed by the `sched_prio_to_weight` array:

```c
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};

```

##### Red-Black Tree Organization

Runnable CFS tasks are placed in a per-CPU self-balancing Red-Black tree (`struct cfs_rq`), keyed strictly on `vruntime`. The leftmost node (`rb_leftmost`) contains the task with the smallest `vruntime` (the most CPU-starved process). The scheduler always picks `rb_leftmost` to run next.

```
                    cfs_rq
                      |
                 [rb_root]
                     |
                (vruntime=120)
                /            \
        (vruntime=105)     (vruntime=135)
            /
    [rb_leftmost] 
    (vruntime=98)  <-- Next task selected by pick_next_task_fair()

```

#### Modern EEVDF Scheduler Mechanics

EEVDF removes CFS heuristics like "latency target" and "sleeper fairness" by assigning explicit **eligible times** ($e_i$) and **virtual deadlines** ($d_i$) to tasks based on requested time slices ($q$).

* **Lag Calculation**: $\text{Lag}_i = V - v_i$, where $V$ is the virtual time of the runqueue and $v_i$ is the task's virtual runtime. A task is *eligible* to execute if $\text{Lag}_i \ge 0$.
* **Virtual Deadline**: $d_i = e_i + \frac{q_i}{w_i}$.
* **Selection Policy**: Out of all *eligible* tasks, EEVDF selects the task with the earliest virtual deadline ($d_i$).

#### Real-Time & Deadline Scheduling Policies

Linux real-time tasks preempt CFS/EEVDF tasks unconditionally.

1. **`SCHED_FIFO`**: First-In, First-Out real-time policy. Runs until it yields, blocks on I/O, or is preempted by a higher-priority `SCHED_FIFO` or `SCHED_RR` task (priorities 1 to 99).
2. **`SCHED_RR`**: Round-Robin real-time policy. Same as `SCHED_FIFO`, but assigned a fixed time quantum (`rr_interval`). When the quantum expires, the task is moved to the tail of its priority runqueue.
3. **`SCHED_DEADLINE`**: Implements the **Earliest Deadline First (EDF)** algorithm coupled with the **Constant Bandwidth Server (CBS)**. Parameterized by three values: `Period`, `Runtime`, and `Deadline`.

$$\text{Guarantee: Task receives } \text{Runtime} \text{ time every } \text{Period} \text{ before } \text{Deadline}$$

```c
struct sched_attr attr = {
    .size     = sizeof(attr),
    .sched_policy   = SCHED_DEADLINE,
    .sched_runtime  = 10 * 1000 * 1000, // 10ms
    .sched_deadline = 20 * 1000 * 1000, // 20ms
    .sched_period   = 30 * 1000 * 1000, // 30ms
};
sched_setattr(0, &attr, 0);

```

---

## 2. Memory Management Subsystem (MM)

### Virtual Memory Architecture

#### x86-64 4-Level and 5-Level Page Table Hierarchy

x86-64 translates Virtual Addresses (VA) to Physical Addresses (PA) using page tables managed by CR3 control register pointers.

```
x86-64 4-Level Page Table Translation Path (48-bit Canonical Virtual Address)

 63        47        38        29        20        11          0
+----------+---------+---------+---------+---------+-----------+
| Sign Ext | PML4    | PDPT    | PD      | PT      | Offset    |
| (16 bits)| (9 bits)| (9 bits)| (9 bits)| (9 bits)| (12 bits) |
+----------+---------+---------+---------+---------+-----------+
                 |         |         |         |         |
 CR3 Register    |         |         |         |         |
   |             |         |         |         |         |
   v             v         |         |         |         |
+-------+      +-------+   |         |         |         |
| PML4  |----->| PDPT  |   |         |         |         |
+-------+      +-------+   v         |         |         |
               | PDPT  |----->+-------+        |         |
               +-------+      |  PD   |        |         |
                              +-------+        v         |
                              |  PD   |----->+-------+   |
                              +-------+      |  PT   |   |
                                             +-------+   |
                                             |  PTE  |---|---> [Physical Frame] + Offset
                                             +-------+

```

* **PML4 (Page Map Level 4)**: Index extracted from bits. Points to Page Directory Pointer Table (PDPT).
* **PDPT (Page Directory Pointer Table)**: Index extracted from bits. Points to Page Directory (PD) or maps a 1GB Huge Page directly.
* **PD (Page Directory)**: Index extracted from bits. Points to Page Table (PT) or maps a 2MB Huge Page directly.
* **PT (Page Table)**: Index extracted from bits. Contains Page Table Entries (PTE) pointing to physical 4KB frames.
* **5-Level Paging (Paging57)**: Adds a Level 5 Page Table (P4D) expanding virtual address space from 48-bit (256 TB) to 57-bit (128 PB), using bits.

#### TLB Flushing & Hardware Page Fault Handling (`do_page_fault`)

1. Memory Reference encounters TLB Miss; hardware MMU reads page tables from physical memory.
2. If PTE `Present` bit is clear, hardware fires Vector 14 Fault (`#PF`) and saves the faulting address to register `CR2`.
3. Architecture-independent kernel handler `do_page_fault()` (in `arch/x86/mm/fault.c`) is invoked.

```
Hardware #PF Interrupt
       |
       v
  CR2 -> read faulting address
       |
       v
do_page_fault()
       |
       +---> Find VMA (find_vma)
       |       |
       |       +-- No VMA / Invalid Access Flags? -> SIGSEGV
       |
       +---> PTE Not Present?
               |
               +-- Is Anonymous Page? -> do_anonymous_page() [Zero Fill]
               +-- Is Swap Entry?      -> do_swap_page()      [I/O Read from Swap]
               +-- Is File Backend?   -> filemap_fault()     [Page Cache Read]

```

4. **TLB Flushing**: When page entries change, TLBs must be invalidated.
* Single-core: `invlpg [address]` instruction.
* Multi-core: Inter-Processor Interrupt (IPI) trigger **TLB Shootdown** across remote execution cores.



---

### Physical Memory Allocation

#### Buddy Allocator (`mm/page_alloc.c`)

Physical RAM is tracked in contiguous blocks of $2^{\text{order}}$ pages ($4\text{KB} \times 2^n$), spanning orders $0$ through `MAX_ORDER` (typically Order 10 = 4MB).

```
Free Lists per Zone (e.g., ZONE_NORMAL)

Order 0 (4KB)  : [Page] <-> [Page] <-> [Page]
Order 1 (8KB)  : [ Block ] <-> [ Block ]
Order 2 (16KB) : [    Block 2    ]
Order 3 (32KB) : Empty

```

##### Allocation Logic

To allocate Order $N$:

1. Query free list for Order $N$. If a free block exists, return it immediately.
2. If Order $N$ is empty, scan higher orders $N+1, N+2, \dots$ until a free block is located.
3. Split the larger block in half: one half satisfies the allocation request; the second half (the "buddy") is placed into the lower-order free list.

##### Coalescing (Freeing) Logic

When freeing a block of Order $N$, the allocator computes the physical address of its buddy:

$$\text{Buddy\_Address} = \text{Base\_Address} \oplus (2^N \times \text{PAGE\_SIZE})$$

If the computed buddy is also free and resides at Order $N$, they merge into a single Order $N+1$ block. This process recurses up to `MAX_ORDER`.

#### Slab/Slub/Slob Allocators (`mm/slub.c`)

The Buddy Allocator manages coarse 4KB page frames. Kernel data structures require fine-grained allocation (bytes to kilobytes). The **SLUB Allocator** satisfies this by slicing Buddy-allocated pages into fixed-size object caches.

```
kmem_cache (e.g., "task_struct", "filp")
  |
  +--> cpu_slab (Per-CPU fast path: Lockless CPU cache)
  |      |
  |      +--> page ---> [Obj 1 (Free)] -> [Obj 2 (Allocated)] -> [Obj 3]
  |
  +--> node_slabs (NUMA Node Slow Path)
         |
         +--> partial (List of slabs with free and used objects)
         +--> full    (List of fully allocated slabs)

```

* **`kmem_cache_create()`**: Allocates object cache templates.
* **`kmem_cache_alloc()` / `kmem_cache_free()**`: High-performance object acquisition and return paths.
* **`kmalloc()`**: Built on top of pre-created sets of standard power-of-two `kmem_cache` instances (`kmalloc-8`, `kmalloc-16`, ..., `kmalloc-8192`).

---

### Memory Reclaim & Swapping

#### Anonymous vs. File-backed Pages

* **File-backed Pages**: Pages mapped from files on disk (executables, shared libraries, page cache). Reclaimable by writing back dirty modifications to the storage device or dropping clean frames outright.
* **Anonymous Pages**: Process heap, stack, `brk`, and `mmap(MAP_ANONYMOUS)` mappings. Have no filesystem backing store; must be written to designated Swap space to reclaim physical RAM.

#### Dual-LRU Active/Inactive List Engine (`mm/swap.c`)

Pages are placed on two split doubly linked lists protected by `spin_lock`: the **Active List** and the **Inactive List**.

```
[ Unreferenced ]                           [ Referenced Twice ]
       |                                           |
       v                                           v
[ Inactive List ] --(Second Chance / Ref Pass)--> [ Active List ]
       |                                           |
       +<-------(Evicted when Active Shrinks)<-----+
       |
       v
[ Reclaimed / Swapped ]

```

1. New pages enter the Inactive List.
2. If referenced again while on the Inactive List, the page's `PG_referenced` bit is set, promoting it to the Active List.
3. When memory pressure triggers kernel reclaim (`kswapd` thread or direct reclaim path), `shrink_active_list()` demotes unreferenced pages from the Active List back to the Inactive List.
4. `shrink_inactive_list()` evicts candidates from the tail of the Inactive List.

#### OOM Killer Mechanics (`mm/oom_kill.c`)

When physical memory and swap are exhausted and reclaim fails, the kernel triggers `out_of_memory()`.

```c
/* Pseudocode representation of badness scoring */
unsigned long oom_badness(struct task_struct *p, unsigned long totalpages) {
    long points;
    long adj;

    if (oom_task_origin(p))
        return ULONG_MAX;

    /* Do not kill init or kernel threads */
    if (p->flags & PF_KTHREAD || is_global_init(p))
        return 0;

    /* Read task memory usage: RSS + swap usage + page tables */
    points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS) +
             atomic_long_read(&p->mm->nr_ptes);

    /* Normalize score relative to total system memory */
    adj = (long)p->signal->oom_score_adj;
    if (adj == OOM_SCORE_ADJ_MIN) // -1000 disables OOM killing
        return 0;

    points += adj * (long)totalpages / 1000;
    return points > 0 ? points : 1;
}

```

The process with the highest calculated `oom_badness` score receives a `SIGKILL` signal, and its `mm_struct` is flagged with `TIF_MEMDIE` to expedite memory freeing via the asynchronous OOM reaper thread.

---

## 3. Virtual File System (VFS) & Storage I/O

### VFS Abstraction Layer

The VFS defines an object-oriented kernel interface abstracting underlying filesystems (ext4, btrfs, xfs, procfs, nfs).

```
                      +-------------------+
                      |   struct file     | (Open File Description)
                      +-------------------+
                        |               |
       f_pos, f_flags   |               v f_op
                        |     +-----------------------+
                        |     | struct file_operations|
                        |     +-----------------------+
                        |     | read, write, ioctl    |
                        |     +-----------------------+
                        v
                      +-------------------+
                      |   struct dentry   | (Directory Hierarchy Node)
                      +-------------------+
                        |               |
             d_name     |               v d_op
                        |     +-----------------------+
                        |     |struct dentry_operations
                        |     +-----------------------+
                        v
                      +-------------------+
                      |   struct inode    | (On-Disk Metadata Copy)
                      +-------------------+
                        |               |
    i_ino, i_size, i_mode|               v i_op
                        |     +-----------------------+
                        |     |struct inode_operations|
                        |     +-----------------------+
                        |     | create, lookup, mkdir |
                        |     +-----------------------+
                        v
                      +-------------------+
                      | struct superblock | (Filesystem Mount Metadata)
                      +-------------------+
                                        | s_op
                                        v
                              +----------------------------+
                              | struct super_operations    |
                              +----------------------------+
                              | alloc_inode, destroy_inode |
                              +----------------------------+

```

#### Primary VFS Objects

##### `struct superblock` (`<linux/fs.h>`)

Represents an entire mounted filesystem instance. Contains global metadata (block size, allocation maps, device references) and the dispatch table `struct super_operations`.

```c
struct super_operations {
    struct inode *(*alloc_inode)(struct super_block *sb);
    void (*destroy_inode)(struct inode *);
    void (*dirty_inode) (struct inode *, int flags);
    int (*write_inode) (struct inode *, struct writeback_control *wbc);
    int (*drop_inode) (struct inode *);
    void (*evict_inode) (struct inode *);
    void (*put_super) (struct super_block *);
    int (*sync_fs)(struct super_block *sb, int wait);
    int (*statfs) (struct dentry *, struct kstatfs *);
    int (*remount_fs) (struct super_block *, int *, char *);
};

```

##### `struct inode`

Represents the metadata of an individual filesystem object (file, directory, symbolic link, FIFO, socket). Key fields include `i_ino` (inode number), `i_mode` (permissions and type bitmask), `i_size` (length in bytes), `i_mapping` (address space mapping for page cache operations), and pointer table `struct inode_operations`.

```c
struct inode_operations {
    struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
    int (*create) (struct user_namespace *, struct inode *,struct dentry *, umode_t, bool);
    int (*link) (struct dentry *,struct inode *,struct dentry *);
    int (*unlink) (struct inode *,struct dentry *);
    int (*mkdir) (struct user_namespace *, struct inode *,struct dentry *, umode_t);
    int (*rmdir) (struct inode *,struct dentry *);
    int (*rename) (struct user_namespace *, struct inode *, struct dentry *,
                   struct inode *, struct dentry *, unsigned int);
};

```

##### `struct dentry`

Represents directory structure hierarchy components connecting pathnames to target inodes. Dentries exist exclusively in memory and form trees to speed up path lookups. Pointer table: `struct dentry_operations`.

##### `struct file`

Represents an open file instance created by `openat()`. It is ephemeral, unique to the calling process context (referenced in `struct files_struct` array), and tracks open flags (`f_flags`), access permissions, internal cursor position (`f_pos`), and operations `struct file_operations`.

```c
struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    __poll_t (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    int (*flush) (struct file *, fl_owner_t id);
    int (*release) (struct inode *, struct file *);
    int (*fsync) (struct file *, loff_t, loff_t, int datasync);
};

```

#### Dcache & Inode Cache Mechanics

* **Dcache (`dentry_hash`)**: Dynamic lookup table using an RCU-protected hash table mapping `(parent_dentry, string_name) -> dentry`. Avoids expensive underlying filesystem directory reads during path resolution (`fs/namei.c:walk_component()`).
* **Inode Cache (`inode_hashtable`)**: Holds in-memory active `struct inode` structures indexed by filesystem superblock pointer and inode number. Unreferenced inodes sit on a LRU cache for rapid re-acquisition before garbage collection.

---

### Block I/O Layer & Storage Drivers

#### The `bio` Structure (`<linux/blk_types.h>`)

The `struct bio` represents an active block I/O operation in vector form.

```c
struct bio {
    struct bio          *bi_next;       /* Request queue link */
    struct block_device *bi_bdev;       /* Target block device */
    unsigned short      bi_flags;      /* BIO status flags */
    unsigned short      bi_ioprio;     /* I/O priority */
    struct bvec_iter     bi_iter;       /* Tracks current sector & offset */
    unsigned int        bi_vcnt;       /* Number of bio_vec elements */
    struct bio_vec      *bi_io_vec;     /* Memory page segment vector */
    bio_end_io_t        *bi_end_io;     /* Completion callback */
};

struct bio_vec {
    struct page *bv_page;   /* Pointer to physical page frame */
    unsigned int bv_len;    /* Segment length in bytes */
    unsigned int bv_offset; /* Offset within page */
};

```

Instead of requiring contiguous physical memory, a `bio` contains an array of `bio_vec` segments scattered across disparate memory pages (scatter-gather I/O).

#### Multi-Queue Block I/O Layer (`blk-mq`)

Modern block storage engines utilize `blk-mq` architecture to scale across multi-core systems, avoiding global request queue locking bottlenecks.

```
+-------------------------------------------------------------+
|                     Software Staging Layer                  |
|  [ CPU 0 Queue ]   [ CPU 1 Queue ]  ...   [ CPU N Queue ]   |
+-------------------------------------------------------------+
                           |  (Hardware Mapping)
                           v
+-------------------------------------------------------------+
|                     Hardware Submission Layer               |
|  [ Hardware Dispatch Queue 0 ]   [ Hardware Dispatch Queue M]|
+-------------------------------------------------------------+
                           |
                           v
              [ NVMe Controller / Block Device ]

```

* **Software Staging Queues (`blk_mq_ctx`)**: Per-CPU request queues where processes submit I/O requests without lock contention.
* **Hardware Dispatch Queues (`blk_mq_hw_ctx`)**: Hardware mapping channels corresponding directly to physical NVMe controller submission queues.

#### I/O Schedulers

Operating on software queues prior to hardware dispatch:

1. **BFQ (Budget Fair Queueing)**: Allocates disk time bandwidth allocations based on process priorities. Optimized for interactive desktop desktop responsiveness and slow rotational media.
2. **Kyber**: Simple scheduler designed for low-latency fast storage (NVMe SSDs). Uses self-tuning target read/write completion latencies.
3. **mq-deadline**: Adaptation of the traditional Deadline scheduler for `blk-mq`. Retains explicit expiration timers on read/write requests (reads prioritized over writes) to prevent I/O starvation.
4. **None**: Direct pass-through mode bypasses software queuing entirely. Used for low-latency NVMe drives where hardware queues process I/O fast enough that software scheduling overhead hurts performance.

---

## 4. Inter-Process Communication (IPC) & Network Subsystem

### IPC Mechanisms

#### System V vs. POSIX IPC Comparison

| Mechanism | System V API | POSIX API | Key Characteristics |
| --- | --- | --- | --- |
| **Shared Memory** | `shmget()`, `shmat()`, `shmdt()` | `shm_open()`, `mmap()` | Direct physical memory mapping between processes. Fast, but requires explicit external synchronization (e.g., mutexes or semaphores). |
| **Message Queues** | `msgget()`, `msgsnd()`, `msgrcv()` | `mq_open()`, `mq_send()`, `mq_receive()` | Structured messaging. POSIX provides message priority queues and async event notification via signals or threads. |
| **Semaphores** | `semget()`, `semop()` | `sem_open()`, `sem_wait()`, `sem_post()` | Counter-based process locking. System V allows atomic array modifications; POSIX provides simpler, lighter primitives. |

#### UNIX Domain Sockets & Pipes

* **Anonymous Pipes (`pipe()`)**: Implemented internally as a single VFS virtual inode pointing to a circular kernel memory buffer (`struct pipe_inode_info`).
* One end is restricted strictly to write operations (`O_WRONLY`); the opposite end is restricted to read operations (`O_RDONLY`).
* Writes block when the circular pipe ring buffer fills (default size: 64KB, adjustable via `fcntl(F_SETPIPE_SZ)`).


* **Named Pipes (`mkfifo()`)**: Identical internal `struct pipe_inode_info` structure as anonymous pipes, but linked into the VFS namespace as a FIFO node, allowing unrelated processes to discover and open it.
* **UNIX Domain Sockets (`AF_UNIX`)**: Provides standard BSD socket semantics locally without network stack overhead. Skips IP checksum calculation, routing table evaluation, and packet serialization. Supports passing open file descriptors between distinct processes via `sendmsg()` and `recvmsg()` carrying `SCM_RIGHTS` control message structures (`struct cmsghdr`).

#### Signaling Extensions (`eventfd`, `signalfd`)

* **`eventfd()`**: Creates an event notification object represented as a single VFS file descriptor containing an 8-byte (64-bit) unsigned integer counter in kernel space. `write()` adds to the counter; `read()` retrieves and clears the counter, blocking if it is zero. Efficient for user-kernel or inter-thread event loops (`epoll`).
* **`signalfd()`**: Delivers standard and real-time Linux signals through an open file descriptor. Eliminates the need for asynchronous signal handler contexts (`sigaction`), allowing processes to read incoming signals via `poll()`, `select()`, or `epoll()`.

---

### Kernel Network Stack Pipeline

#### The `sk_buff` (Socket Buffer) Architecture (`<linux/skbuff.h>`)

`struct sk_buff` is the fundamental structure used to track network packet data as it traverses the kernel network stack.

```
+---------------------------------------------------+
|                  struct sk_buff                   |
|  head ---------> +-----------------------------+   |
|  data ---------> | Headroom (Reserved space)   |   |
|                  +-----------------------------+   |
|                  | Link Layer Header (MAC)     |   |
|                  +-----------------------------+   |
|                  | Network Layer Header (IP)   |   |
|                  +-----------------------------+   |
|                  | Transport Layer (TCP/UDP)   |   |
|  tail ---------> +-----------------------------+   |
|                  | Tailroom (Appended payload) |   |
|  end ----------> +-----------------------------+   |
+---------------------------------------------------+

```

##### Pointer Manipulation Helpers

* **`skb_reserve(skb, len)`**: Moves both `data` and `tail` pointers down, creating headroom for headers.
* **`skb_push(skb, len)`**: Decrements the `data` pointer (moving up into headroom) to prepending protocol headers (e.g., adding an IP header before transmission).
* **`skb_pull(skb, len)`**: Increments the `data` pointer down to strip protocol headers as the packet moves up the network stack.
* **`skb_put(skb, len)`**: Advances the `tail` pointer down to expand the payload payload region.

#### Complete Packet Ingress Traversal Lifecycle

```
[ Network Interface Card (NIC) ]
               |
               | 1. DMA Transfer to Rx Ring Buffer
               v
     [ Rx Descriptor Ring ]
               |
               | 2. Raise Hardware Interrupt (IRQ)
               v
[ CPU Execution Core Interrupt Handler ]
               |
               | 3. Disable NIC IRQ line & Schedule SoftIRQ
               v
      [ NET_RX_SOFTIRQ ]
               |
               | 4. Invoke NAPI Polling Routine (napi_poll)
               v
        [ napi_poll() ]
               |
               | 5. Driver extracts frame, allocates sk_buff
               v
     [ netif_receive_skb() ]
               |
               | 6. Route to registered Packet Type Handler
               v
       [ ip_rcv() (IPv4) ]
               |
               | 7. Checksum validation, Netfilter PREROUTING
               v
     [ ip_local_deliver() ]
               |
               | 8. Reassemble IP fragments, Netfilter INPUT
               v
      [ tcp_v4_rcv() (TCP) ]
               |
               | 9. Validate TCP checksum, check connection state
               v
     [ tcp_data_queue() ]
               |
               | 10. Append sk_buff to target Socket Receive Queue
               v
    [ sk->sk_receive_queue ]
               |
               | 11. Wake up blocking process (sk_data_ready)
               v
  [ User Space Application ]
   (read / recvmsg unblocks)

```

1. **Physical Reception**: The Network Interface Card (NIC) receives an optical/electrical frame and executes a Direct Memory Access (DMA) transfer to copy the frame into a pre-allocated host kernel memory buffer in the Rx Ring Descriptor Array.
2. **Hardware Interrupt**: The NIC raises a hardware IRQ. The CPU executes the top-half device driver interrupt handler, which disables the hardware interrupt line on that NIC to prevent interrupt storms, and schedules a bottom-half `NET_RX_SOFTIRQ`.
3. **NAPI Polling Loop**: The kernel execution thread processes the `NET_RX_SOFTIRQ`, executing the driver's registered NAPI poll routine (`napi_poll()`). The driver polls packets directly out of the Rx Descriptor Ring in batches up to a fixed quota (default budget: 64 packets).
4. **`sk_buff` Allocation**: The driver wraps the raw frame in a `struct sk_buff` wrapper and passes it to the generic network layer via `netif_receive_skb()`.
5. **Network Protocol Layer Parsing (`ip_rcv`)**:
* Evaluates link-layer fields, validates IP header checksums, and processes Netfilter hooks (`NF_INET_PRE_ROUTING`).
* Passes packet to `ip_local_deliver()`, which handles IP defragmentation and passes it through `NF_INET_LOCAL_IN` hooks.


6. **Transport Protocol Layer Parsing (`tcp_v4_rcv`)**:
* Looks up the corresponding socket (`struct sock`) using the 4-tuple: `(Source IP, Source Port, Destination IP, Destination Port)`.
* Evaluates TCP sequence numbers, manages window sizes, updates TCP state machines, and constructs acknowledgment frames (ACKs).


7. **Socket Queue Enqueue**: `tcp_data_queue()` appends the `sk_buff` to `sk->sk_receive_queue`.
8. **User Space Context Switch**: The kernel invokes the socket's data notification callback (`sk_data_ready()`), which wakes processes sleeping on `sys_read()`, `sys_recv()`, `select()`, `poll()`, or `epoll_wait()`. The user thread copies the payload from `sk_buff` into application memory buffers and frees the socket buffer via `kfree_skb()`.

---

## 5. Practical Implementation: LKM Process Tree Traversal

The following complete Loadable Kernel Module (LKM) demonstrates how to safely traverse the kernel process tree starting from the current process context (`current`), navigating up to the root `init_task`, and using RCU lock wrappers to iterate over child task nodes using internal kernel macros (`for_each_process`).

### `sys_tree_walker.c`

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/sched/signal.h>
#include <linux/rcupdate.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Principal Systems Engineer");
MODULE_DESCRIPTION("LKM demonstrating safe process tree and system task traversal");
MODULE_VERSION("1.0");

static void walk_parent_hierarchy(struct task_struct *start_task)
{
    struct task_struct *curr = start_task;
    int depth = 0;

    pr_info("[Tree Walker] Starting Ancestry Walk from: %s [PID: %d]\n",
            curr->comm, curr->pid);

    /* Read-side critical section to safely traverse parent pointers */
    rcu_read_lock();
    while (curr && curr->pid != 0) {
        pr_info("  --> [%d] PID: %5d | TGID: %5d | Task Name: %-16s | State: 0x%lx\n",
                depth, curr->pid, curr->tgid, curr->comm, (unsigned long)curr->__state);

        if (curr->pid == 1) {
            pr_info("  --> Reached Init Process (PID 1)\n");
            break;
        }

        /* Safely move to parent process */
        curr = rcu_dereference(curr->real_parent);
        depth++;
    }
    rcu_read_unlock();
}

static void iterate_system_processes(void)
{
    struct task_struct *task;
    unsigned int process_count = 0;
    unsigned int thread_count = 0;

    pr_info("[Tree Walker] Beginning Full System Task Traversal via for_each_process()\n");

    rcu_read_lock();
    for_each_process(task) {
        struct task_struct *thread;
        process_count++;

        /* Iterate over all threads within the thread group */
        for_each_thread(task, thread) {
            thread_count++;
        }
    }
    rcu_read_unlock();

    pr_info("[Tree Walker] Traversal Summary: Active Processes (TGIDs): %u | Total Threads (PIDs): %u\n",
            process_count, thread_count);
}

static int __init tree_walker_init(void)
{
    pr_info("========================================================\n");
    pr_info("[Tree Walker] Kernel Module Loaded Successfully\n");
    pr_info("========================================================\n");

    /* 1. Walk parents from current task context */
    walk_parent_hierarchy(current);

    /* 2. Traverse all processes on the system */
    iterate_system_processes();

    return 0;
}

static void __exit tree_walker_exit(void)
{
    pr_info("[Tree Walker] Cleaning up and unloading kernel module.\n");
}

module_init(tree_walker_init);
module_exit(tree_walker_exit);

```

### `Makefile`

```makefile
obj-m += sys_tree_walker.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

```