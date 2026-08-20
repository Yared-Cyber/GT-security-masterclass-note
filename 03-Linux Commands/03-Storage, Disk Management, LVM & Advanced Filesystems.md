## SECTION 1: Disk Usage & Monitoring

---

### Command: `df -hT`

#### 1. Fundamental Purpose & Historical Evolution

`df` (Disk Free) originated in early AT&T UNIX to summarize storage space availability across mounted filesystems. Historically, applications queried static sector counters or parsed mount tables directly, which caused race conditions and inconsistencies across disparate disk structures.

As UNIX evolved toward abstracting storage architectures, the Virtual Filesystem (VFS) layer introduced unified filesystem statistics APIs (`statfs`/`statvfs`). This allowed user space to query uniform metadata regardless of the underlying storage hardware, network protocols, or block structures.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
df Binary -> Open /proc/self/mountinfo -> Iterate Mounts -> statvfs() Syscall
                                                                |
+---------------------------------------------------------------+
| VFS Layer: sys_statvfs() / vfs_statfs()
|   -> Fetch struct super_block
|   -> Invoke sb->s_op->statfs(dentry, &statbuf)
+---------------------------------------------------------------+
                                |
        +-----------------------+-----------------------+
        v                                               v
Ext4 Driver: ext4_statfs()                      XFS Driver: xfs_fs_statfs()
  Reads s_blocks_count, s_r_blocks_count          Reads agi/agf counters
  Calculates free/reserved blocks                 Summarizes allocation groups

```

When `df -hT` executes:

1. **Mount Point Enumeration:** The process opens and reads `/proc/self/mountinfo` (or `/proc/mounts`). This presents a point-in-time view of the process mount namespace, identifying mount points, options, devnames, and filesystem types.
2. **System Call Invocation:** For each enumerated mount point, `df` invokes the `statvfs()` library function, which wraps the `statfs()` or `statvfs64()` system call:

$$\text{sys\_statfs(const char *path, struct statfs *buf)}$$


3. **VFS Kernel Traversal:**
* The kernel resolves `path` to a `struct dentry` and extracts the associated `struct super_block`.
* The VFS calls the filesystem-specific operation pointer `sb->s_op->statfs()`.
* **Ext4 Implementation (`ext4_statfs()`):** Queries the in-memory superblock (`struct ext4_sb_info`). Free block counts are derived by reading group descriptors and deducting reserved blocks (`s_r_blocks_count`, typically 5% allocated for root/daemon protection against total space exhaustion).
* **XFS Implementation (`xfs_fs_statfs()`):** Queries the XFS mount structure (`struct xfs_mount`). It aggregates free data block counters across all Allocation Groups (AGs) via AG free block headers (`xfs_agf_t`) and subtracts space reserved for internal B-trees and log buffers.


4. **Human-Readable Binary Math:** The binary transforms raw block counts into powers of $1024$ ($2^{10} = \text{KiB}$, $2^{20} = \text{MiB}$, $2^{30} = \text{GiB}$, $2^{40} = \text{TiB}$).

#### 3. Disk Structure & On-Disk Layout Dynamics

Filesystem block accounting relies directly on superblock structures defined on physical disk:

```
Ext4 Superblock Layout (Offset 1024 bytes, Size 1024 bytes)
+-------------------------------------------------------------------+
| Magic Bytes: 0xEF53 (Offset 0x38)                                 |
| s_blocks_count_lo / s_blocks_count_hi (64-bit Total Block Count) |
| s_r_blocks_count_lo / s_r_blocks_count_hi (Reserved Block Count)  |
| s_free_blocks_count_lo / s_free_blocks_count_hi (Free Blocks)     |
+-------------------------------------------------------------------+

```

* **Ext4:** The superblock sits at byte offset `1024` from the start of the partition. It contains fields like `s_blocks_count_lo/hi` and `s_free_blocks_count_lo/hi`. Total capacity equals $(\text{s\_blocks\_count} \times \text{block\_size})$. Available non-root space equals:

$$\text{Free} = (\text{s\_free\_blocks\_count} - \text{s\_r\_blocks\_count}) \times \text{block\_size}$$


* **XFS:** The primary superblock resides at Sector 0 of Allocation Group 0 (AG 0). It maintains `sb_dblocks` (total data blocks), `sb_fdblocks` (free data blocks), and `sb_agblocks` (blocks per AG).

#### 4. Line-by-Line Flag & Syntax Breakdown

* `-h` (`--human-readable`): Modifies output formatting by dividing raw block counts by powers of $1024$ ($2^{10}, 2^{20}, 2^{30}$) rather than displaying default 1 KiB block totals.
* `-T` (`--print-type`): Directs `df` to extract and output the string representation of `statvfs.f_btype` / raw mountinfo filesystem driver names (e.g., `ext4`, `xfs`, `zfs`, `btrfs`, `tmpfs`).

#### 5. Exhaustive Output Anatomy

```
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/nvme0n1p2 ext4      468G  120G  325G  27% /
devtmpfs       devtmpfs  16G      0   16G   0% /dev
tmpfs          tmpfs     16G   250M   16G   2% /dev/shm
/dev/nvme0n1p1 vfat      511M  6.1M  505M   2% /boot/efi
/dev/sda1      xfs       1.8T  850G  980G  47% /mnt/data

```

* `Filesystem`: Physical block device path (`/dev/nvme0n1p2`), virtual device node, or remote network path.
* `Type`: Kernel filesystem driver identifier (`ext4`, `xfs`, `vfat`, `tmpfs`).
* `Size`: Total usable capacity calculation:

$$\text{Capacity} = \text{f\_blocks} \times \text{f\_frsize}$$


* `Used`: Total allocated block space:

$$\text{Used} = (\text{f\_blocks} - \text{f\_bfree}) \times \text{f\_frsize}$$


* `Avail`: Space available to non-privileged processes:

$$\text{Avail} = \text{f\_bavail} \times \text{f\_frsize}$$



*Note:* Because $\text{f\_bavail} = \text{f\_bfree} - \text{reserved\_blocks}$, $\text{Size} \neq \text{Used} + \text{Avail}$.
* `Use%`: Percentage calculation:

$$\text{Use\%} = \frac{\text{Used}}{\text{Used} + \text{Avail}} \times 100$$


* `Mounted on`: VFS namespace mounting directory path where the root inode of this filesystem is attached.

---

### Command: `du -sh *`

#### 1. Fundamental Purpose & Historical Evolution

`du` (Disk Usage) measures the space consumed by files, directories, and subtrees. While `df` queries global filesystem metadata, `du` walks hierarchical directory structures to aggregate allocated space on a per-inode level.

Historically, user tools calculated file size by taking the byte length reported in directory structures. This introduced bugs with sparse files (where unallocated zero-regions do not consume physical disk blocks) and hard links (which caused double-counting). Modern `du` addresses these edge cases by querying actual allocated block counts (`st_blocks`) and tracking unique inode numbers in memory.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
du -sh * Initiated
   |
   +--> openat(AT_FDCWD, target_dir, O_RDONLY|O_DIRECTORY)
   |
   +--> loop: getdents64() -> returns linux_dirent64 buffer
         |
         +--> fstatx() / fstatat(dfd, filename, &statbuf, AT_SYMLINK_NOFOLLOW)
               |
               +--> VFS vfs_fstatat()
                     |
                     +--> Extract statbuf.st_ino, statbuf.st_dev, statbuf.st_blocks
                     |
                     +--> Inode Deduplication Check (Hash Table lookup: [st_dev, st_ino])
                           |
                           +-- NEW: Record in hash table; Add (st_blocks * 512 bytes)
                           +-- SEEN: Skip addition (prevents double-counting hard links)

```

1. **Directory Traversal:** `du` uses `openat()` to open directory file descriptors and calls `getdents64()` to parse `struct linux_dirent64` records in chunks.
2. **Stat System Calls:** For every entry, it issues `fstatat()` or `statx()` passing `AT_SYMLINK_NOFOLLOW` so symlinks are not dereferenced:

$$\text{sys\_statx(int dfd, const char *filename, int flags, unsigned int mask, struct statx *statxbuf)}$$


3. **Hard Link Inode Tracking:** `du` maintains an in-memory red-black tree or hash table storing tuple pairs `(st_dev, st_ino)`.
* If an inode's device and inode number are already in the table, `du` ignores its size for the aggregate counter.
* If missing, it adds `(st_dev, st_ino)` to the map and accumulates `st_blocks`.


4. **Block Allocation Math vs File Length:** `statx.st_blocks` represents space in **512-byte units**, regardless of the underlying filesystem block size (e.g., 4096 bytes).

$$\text{Physical Size} = \text{st\_blocks} \times 512$$


* **Sparse Files:** A $10\text{ GiB}$ sparse file with only $1\text{ MiB}$ of written data reports `st_size = 10737418240`, but `st_blocks = 2048`. `du` correctly calculates space as $2048 \times 512 = 1\text{ MiB}$.
* **Tail Packing / Small Files:** Small files inside a $4\text{ KiB}$ block filesystem consume a minimum of 8 blocks ($8 \times 512 = 4096\text{ bytes}$) unless inline data or tail-packing mechanics apply.



#### 3. Disk Structure & On-Disk Layout Dynamics

`du` extracts block information directly from on-disk inode structs via the kernel VFS layer:

```
Ext4 On-Disk Inode Structure (128/256 bytes)
+-----------------------------------------------------------------------+
| i_mode (16 bits) | i_uid (16 bits) | i_size_lo / i_size_high (64 bits)|
| i_blocks_lo (32 bits) | i_blocks_high (16 bits) -> Total 512B Blocks  |
| i_block[15] -> Extent Tree Root / Block Pointers                      |
+-----------------------------------------------------------------------+

```

When files use extents (e.g., Ext4 `i_block` pointing to `ext4_extent_header`), `i_blocks` counts all allocated data blocks plus intermediate extent tree allocation blocks.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `-s` (`--summarize`): Suppresses recursive tree printing for every child directory. It outputs only the aggregate total for each specified argument.
* `-h` (`--human-readable`): Formats output values using base-1024 unit suffixes (K, M, G, T).
* `*`: Shell glob expansion that expands to all non-hidden paths in the current working directory. `du` executes iteratively against each expanded entry.

#### 5. Exhaustive Output Anatomy

```
4.0K    /var/log/alternatives.log
1.2G    /var/log/journal
12M     /var/log/syslog
1.3G    total

```

* `4.0K`, `1.2G`, `1.3G`: Total actual physical allocation ($\text{st\_blocks} \times 512$) converted to human-readable base-2 scales.
* Path string: The file or directory path passed via shell globbing or manual input arguments.

---

### Tool: `ncdu`

#### 1. Fundamental Purpose & Historical Evolution

`ncdu` (NCurses Disk Usage) replaces traditional `du` and custom shell parsing scripts with an interactive terminal viewer. As storage capacity grew into tens of terabytes with millions of files, standard `du` output became difficult to navigate efficiently. `ncdu` pairs depth-first directory scanning with an ncurses TUI, displaying interactive, sorted directory views directly in the terminal.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
ncdu Execution Pipeline
  |
  +-- 1. Initialize ncurses (initscr, raw, noecho)
  |
  +-- 2. Traverse Directory Hierarchy (Depth-First Search)
  |      +-- openat() -> getdents64() -> statx()
  |      +-- Dynamically allocate in-memory nodes (struct dir / struct file)
  |
  +-- 3. In-Memory Directory Tree Construction
  |      +-- Aggregate physical size (st_blocks * 512) upward to parents
  |      +-- Build double-linked list pointers for child navigation
  |
  +-- 4. Interactive Rendering Loop
         +-- Process keybindings (j/k, Enter, d)
         +-- Refresh ncurses buffer -> write to stdout TTY

```

1. **Terminal Initialization:** Invokes `initscr()`, `raw()`, `noecho()`, and sets up signal handlers (`SIGWINCH` for terminal window resizes, `SIGINT` for clean exits).
2. **Directory Traversal Loop:** `ncdu` uses an optimized depth-first search (DFS). It combines `openat()`, `getdents64()`, and `fstatat()`/`statx()` calls, storing the metadata in an in-memory tree:
```c
struct node {
    char *name;
    uint64_t size;   /* st_size */
    uint64_t asize;  /* st_blocks * 512 */
    ino_t ino;
    dev_t dev;
    struct node *parent;
    struct node *line_next;
    struct node *sub;
    uint16_t flags;
};

```


3. **Aggregating Subtree Totals:** When traversing back up the file tree, `ncdu` adds the `asize` and `size` fields of child nodes to their parent nodes. Hard links are deduplicated using an in-memory hash table.
4. **TUI Render Cycle:** `ncdu` translates the structured node tree into curses primitives (`waddstr`, `wprintw`), drawing a progress bar and item list that can be sorted by usage, size, file count, or name.

#### 3. Disk Structure & On-Disk Layout Dynamics

`ncdu` reads directory entry metadata via VFS system calls without altering physical disk layouts. However, when a user issues a delete command (`d`) within `ncdu`, it executes `unlinkat()` system calls. This decrements the target inode's `i_nlink` field and frees underlying extents or data blocks back to the filesystem's allocation maps (e.g., Ext4 block bitmaps or XFS free space B-trees).

#### 4. Line-by-Line Flag & Syntax Breakdown

* Executing `ncdu [path]` without flags starts an interactive scan of the specified target path (defaulting to current working directory if omitted).
* Key operational flags:
* `-x`: Restricts directory scanning to the single filesystem mount point where the scan originated (checks `statx.st_dev` and skips child directories on different device IDs).
* `-r`: Read-only mode. Disables file deletion (`d`) and modification capabilities within the TUI interface.



#### 5. Exhaustive Output Anatomy

```
ncdu 1.16 ~ Use the arrow keys to navigate, press ? for help
--- /var/log -------------------------------------------------------------------------------------------------------------------------------------------------------
  1.2 GiB [==========] /journal
 12.4 MiB [          ]  syslog
  4.1 MiB [          ] /installer
400.0 KiB [          ]  dpkg.log
  4.0 KiB [          ] /alternatives.log

```

* Header Bar: Current version, keymap hints, and absolute path of the highlighted directory scope (`/var/log`).
* Column 1 (`1.2 GiB`, `12.4 MiB`): Calculated aggregated disk usage ($\sum \text{st\_blocks} \times 512$) for that item or directory tree.
* Column 2 (`[==========]`): Visual percentage graph representing the item's relative footprint compared to its parent directory's total usage.
* Column 3 (`/journal`, `syslog`): Item name. Leading `/` indicates a directory containing child items.

---

### Command: `lsblk -f`

#### 1. Fundamental Purpose & Historical Evolution

Legacy systems identified storage hardware using static device nodes in `/dev` (e.g., `/dev/hda`, `/dev/sda`). As hot-pluggable storage (NVMe, USB, iSCSI, SANs) evolved, static device mapping caused non-deterministic naming upon reboot.

`lsblk` (List Block Devices) was developed as part of `util-linux` to parse kernel block layer structures and `udev` database metadata. It presents parent-child storage hierarchies—from physical PCI controllers down through partition maps, Device Mapper targets, LVM volume groups, and mounted filesystems—in a unified tree view.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
lsblk Process
  |
  +-- Read /sys/block/ and /sys/dev/block/
  |     |
  |     +-- Traverse parent/child sysfs directory relationships
  |         (e.g., /sys/block/nvme0n1/nvme0n1p2/holders)
  |
  +-- Query libudev Database (/run/udev/data/b<major>:<minor>)
  |     |
  |     +-- Fetch cached disk properties, labels, UUIDs, model strings
  |
  +-- Fallback / Direct Inspection via libblkid
        |
        +-- open("/dev/nvme0n1p2", O_RDONLY|O_CLOEXEC)
        +-- Match magic signatures at offset tables

```

1. **Sysfs Hierarchy Parsing:** `lsblk` scans `/sys/dev/block/`, which contains symbolic links named by major and minor device numbers (`<major>:<minor>`) pointing to block device entries in `/sys/devices/`.
2. **Parent-Child Relationship Tree:** `lsblk` determines device trees by parsing sysfs relationship directories. For example, `/sys/block/sda/sda1/` indicates that partition `sda1` is a child of disk `sda`. For Device Mapper devices, `lsblk` scans `/sys/block/dm-0/slaves/` and `/sys/block/sda1/holders/` to discover underlying dependencies.
3. **`libudev` & `libblkid` Integration:**
* `lsblk -f` uses `libudev` to read properties cached by the udev daemon (`udevd`) in `/run/udev/data/b<major>:<minor>`.
* If `udev` data is unavailable or missing signatures, `lsblk` falls back to `libblkid`. `libblkid` opens the target block device in read-only mode and scans for superblock magic byte offset signatures.


4. **Mount Point Correlation:** Cross-references major/minor IDs and block paths with entries parsed from `/proc/self/mountinfo`.

#### 3. Disk Structure & On-Disk Layout Dynamics

`lsblk -f` inspects filesystem superblock magic bytes directly at known disk offsets:

| Filesystem / Architecture | Byte Offset Location | Magic Byte Signature |
| --- | --- | --- |
| **Ext2 / Ext3 / Ext4** | Offset `1024` (0x400) | `0xEF53` |
| **XFS** | Offset `0` (0x000) | `0x58465342` ("XFSB") |
| **Btrfs** | Offset `65536` (0x10000) | `0x5F42545246735F` ("*btrfs*") |
| **LUKS Encrypted** | Offset `0` (0x000) | `0x4C554B53002D` ("LUKS\0-") |
| **Swap Partition** | Offset `4086` (Page Size - 10) | `0x53574150535041434532` ("SWAPSPACE2") |

#### 4. Line-by-Line Flag & Syntax Breakdown

* `-f` (`--fs`): Directs `lsblk` to output filesystem metadata columns: `FSTYPE` (filesystem type), `LABEL` (filesystem label), `UUID` (Universally Unique Identifier), `FSAVAIL` (filesystem space available), `FSUSE%` (filesystem capacity used percentage), and `MOUNTPOINTS` (current VFS attach points).

#### 5. Exhaustive Output Anatomy

```
NAME          FSTYPE      LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                 
├─sda1        vfat              B8A2-3F1C                             504.9M     1% /boot/efi
├─sda2        ext4              e4f923a1-12c4-4b92-8e2b-1199a0d898ef  324.8G    27% /
└─sda3        crypto_LUKS       9f8d1c2b-3a4f-5e6d-7c8b-9a0b1c2d3e4f                
  └─secure_lv ext4              c1a2b3c4-d5e6-7f8a-9b0c-1d2e3f4a5b6c  912.4G    10% /mnt/secure

```

* `NAME`: Device node basename formatted hierarchically. Tree branches display parent/child block device relationships (e.g., `sda3` parent of `secure_lv`).
* `FSTYPE`: Filesystem driver or block layer format detected via magic byte probing (`vfat`, `ext4`, `crypto_LUKS`).
* `LABEL`: On-disk volume label string read from superblock headers.
* `UUID`: $128$-bit hexadecimal UUID string assigned during volume formatting.
* `FSAVAIL`: Space available to non-privileged users via `statvfs()`.
* `FSUSE%`: Capacity utilization percentage ($\text{Used} / \text{Total}$).
* `MOUNTPOINTS`: Active mount paths in the process mount namespace.

---

### Command: `blkid`

#### 1. Fundamental Purpose & Historical Evolution

`blkid` provides a targeted user-space interface to query block device attributes like UUIDs, LABELs, and probe filesystem signatures.

Early Linux startup scripts depended on static drive references (`/dev/sda1`). This caused system failure if drive boot ordering changed. `blkid` enabled persistent block device naming (such as `UUID=...` or `LABEL=...`) in `/etc/fstab` by scanning block device headers and maintaining a persistent cache (`/etc/blk_cache` or `/run/blkid/blkid.tab`).

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
blkid Execution Flow
  |
  +-- Read /etc/blk_cache (or /run/blkid/blkid.tab)
  |     |
  |     +-- Validate cached entries against dev_t major:minor timestamps
  |
  +-- Cache Miss or Direct Device Scan -> libblkid Probing
        |
        +-- open("/dev/sda1", O_RDONLY|O_CLOEXEC)
        |
        +-- Issue ioctl(fd, BLKGETSIZE64, &size) -> Fetch physical block device size
        |
        +-- Direct Read at Probing Offsets -> libblkid Topology Engine
              |
              +-- Read Offset 0x000 -> Check XFS, LUKS
              +-- Read Offset 0x400 -> Check Ext2/3/4
              +-- Read Offset 0x10000 -> Check Btrfs

```

1. **Cache Loading:** `blkid` reads the binary/text file `/run/blkid/blkid.tab` to quickly load previously discovered block devices and signatures.
2. **Cache Validation:** For each cached device node, `blkid` calls `stat()` to compare the active device's major/minor numbers and modification timestamp (`st_mtime`) with the cached metadata. If the cache is stale, `blkid` re-probes the device.
3. **`libblkid` Probing Engine:**
* Opens the block device with `O_RDONLY | O_CLOEXEC`.
* Executes an `ioctl()` call to retrieve total device size:

$$\text{ioctl(fd, BLKGETSIZE64, uint64\_t *bytes)}$$


* Runs a series of probing routines across known magic byte offsets.
* When a magic signature matches, `blkid` extracts structural fields (e.g., UUID bytes, Volume Label strings, Block Size parameters).



#### 3. Disk Structure & On-Disk Layout Dynamics

`blkid` parses specific bytes directly from superblock and header structures:

```
Ext4 Superblock Header Mapping (Offset 1024)
+-------------------------------------------------------------------------+
| Offset 0x38 (2 bytes): Magic Number (0xEF53)                            |
| Offset 0x60 (16 bytes): s_uuid (128-bit UUID field)                      |
| Offset 0x70 (16 bytes): s_volume_name (ASCII Volume Label String)       |
+-------------------------------------------------------------------------+

```

```
LUKS1/2 Superblock Header Mapping (Offset 0)
+-------------------------------------------------------------------------+
| Offset 0x00 (6 bytes): Magic Bytes ("LUKS\xba\xbe" / "LUKS\x00\x2d")    |
| Offset 0x08 (2 bytes): Version (1 or 2)                                 |
| Offset 0x28 (40 bytes): UUID String (e.g. "9f8d1c2b-...")              |
+-------------------------------------------------------------------------+

```

#### 4. Line-by-Line Flag & Syntax Breakdown

* Executing `blkid` without flags scans all devices listed in `/proc/partitions` and prints their detected tags.
* Common operational flags:
* `-s <tag>`: Restricts output to specified tags (e.g., `blkid -s UUID /dev/sda1` returns only the device UUID).
* `-o value`: Prints only the raw tag value without variable key identifiers, making it ideal for shell scripting automation.
* `-c <file>`: Specifies an alternate cache file location instead of `/run/blkid/blkid.tab`.



#### 5. Exhaustive Output Anatomy

```
/dev/sda1: UUID="B8A2-3F1C" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="c1f2e3d4-01"
/dev/sda2: UUID="e4f923a1-12c4-4b92-8e2b-1199a0d898ef" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c1f2e3d4-02"
/dev/sda3: UUID="9f8d1c2b-3a4f-5e6d-7c8b-9a0b1c2d3e4f" TYPE="crypto_LUKS" PARTUUID="c1f2e3d4-03"

```

* Device Node Path (`/dev/sda2`): Kernel block device node path.
* `UUID="..."`: Filesystem or volume-level identifier stored inside the superblock data structure.
* `BLOCK_SIZE="..."`: Minimum operational allocation block size defined inside the filesystem superblock (e.g., `4096` bytes).
* `TYPE="..."`: Detected driver type based on matching magic signature routines.
* `PARTLABEL="..."`: Partition name string stored inside GPT partition entry metadata arrays (if present).
* `PARTUUID="..."`: Partition UUID defined in the GPT partition array entry, which remains distinct from the internal filesystem `UUID`.

---

---

## SECTION 2: Partitioning & Formatting

---

### Tool: `fdisk /dev/sdX`

#### 1. Fundamental Purpose & Historical Evolution

`fdisk` is a foundational utility for manipulating Master Boot Record (MBR) partition tables. Created during early PC DOS development, MBR uses 32-bit Logical Block Addressing (LBA) and Cylinder-Head-Sector (CHS) geometry maps.

Because MBR uses 32-bit LBA sector addressing paired with 512-byte sector sizes, it cannot address disk space beyond $2^{32} \times 512 = 2.19\text{ TB}$. Modern `fdisk` supports both MBR and GPT, though it remains the primary reference implementation for managing MBR partition structures.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
fdisk /dev/sdX Operations
  |
  +-- open("/dev/sdX", O_RDWR | O_EXCL)
  |     |
  |     +-- O_EXCL prevents concurrent modifications by other utilities
  |
  +-- Read Sector 0 (First 512 bytes) -> Parse MBR Layout
  |
  +-- In-Memory Partition Table Modifications
  |
  +-- User Issues 'w' (Write Changes to Disk)
        |
        +-- pwrite(fd, mbr_buffer, 512, 0) -> Commit Sector 0
        |
        +-- ioctl(fd, BLKPG, &blkpg_arg) -> Inform Kernel of Table Changes
              |
              +-- Kernel block layer: block/ioctl.c -> blkpg_ioctl()
              +-- Deletes removed partition nodes, instantiates new partition nodes
              +-- Re-reads partition table without requiring a reboot

```

1. **Exclusive Device Opening:** `fdisk` opens `/dev/sdX` using `open()` with `O_RDWR | O_EXCL`. `O_EXCL` acquires an exclusive lock on the block device node, preventing competing processes from altering disk structures concurrently.
2. **Reading Sector 0:** Issues `read()` or `pread()` to load physical Sector 0 ($512\text{ bytes}$) into memory. It verifies the final two bytes for the boot record magic signature `0x55AA`.
3. **Partition Table Manipulation:** Changes made within `fdisk` exist only in user-space memory buffers until explicitly written.
4. **Writing and Notifying the Kernel:** When the user enters `w`:
* `fdisk` writes the updated 512-byte MBR sector to LBA 0 using `pwrite()`.
* It issues `ioctl(fd, BLKPG, struct blkpg_ioctl_arg *arg)` calls to update the kernel partition table in memory.
* The kernel parses the structure and calls `add_partition()` or `del_partition()` in `block/partitions/core.c`. This dynamically creates or removes device nodes (e.g., `/dev/sda1`) in sysfs and `udev` without requiring a system reboot.



#### 3. Disk Structure & On-Disk Layout Dynamics

```
Master Boot Record (MBR) - Sector 0 (512 Bytes)
+-------------------------------------------------------------------------+
| Offsets 0x000 - 0x1BD (446 bytes): Bootloader Code (Stage 1 / GRUB)     |
+-------------------------------------------------------------------------+
| Offset 0x1BE (16 bytes): Partition Entry #1                             |
| Offset 0x1CE (16 bytes): Partition Entry #2                             |
| Offset 0x1DE (16 bytes): Partition Entry #3                             |
| Offset 0x1EE (16 bytes): Partition Entry #4                             |
+-------------------------------------------------------------------------+
| Offset 0x1FE (2 bytes): Magic Boot Signature (0x55 0xAA)                |
+-------------------------------------------------------------------------+

```

An individual 16-byte MBR Partition Entry structure consists of:

```
MBR 16-Byte Partition Entry Map
Byte 0      : Boot Indicator (0x80 = Active/Bootable, 0x00 = Inactive)
Bytes 1-3   : Starting CHS Address (Legacy Geometry Format)
Byte 4      : Partition Type Flag (e.g., 0x83 = Linux Native, 0x8e = LVM, 0x05 = Extended)
Bytes 5-7   : Ending CHS Address
Bytes 8-11  : Starting LBA Sector Address (32-bit Unsigned Little-Endian Integer)
Bytes 12-15 : Total Sectors in Partition (32-bit Unsigned Little-Endian Integer)

```

* **Extended & Logical Partitions:** To bypass the 4-primary-partition limit, an entry can use type `0x05` or `0x0F` (Extended Partition). This points to an Extended Boot Record (EBR) sector elsewhere on disk. The EBR contains its own 16-byte partition entry and a link pointer to the next EBR, creating a singly-linked list of logical partitions.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `/dev/sdX`: Path specifying the target raw block device.
* Interactive Commands inside `fdisk`:
* `p`: Parses the in-memory MBR structure and prints the active partition configuration.
* `n`: Allocates a new partition entry. Asks for primary vs extended selection, partition number ($1$-$4$), starting sector, and ending sector offset.
* `t`: Modifies byte `0` of the 16-byte entry (Partition Type flag, e.g., changing from `0x83` Linux native to `0x8E` LVM).
* `w`: Writes the modified 512-byte buffer back to LBA 0 via `pwrite()` and triggers kernel table updates via `ioctl(BLKPG)`.



#### 5. Exhaustive Output Anatomy

```
Disk /dev/sda: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: ST2000DM008-2FR1
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x7c9a2b10

Device     Boot   Start        End    Sectors   Size Id Type
/dev/sda1  *       2048    2099199    2097152     1G 83 Linux
/dev/sda2       2099200 3907028991 3904929792  1.8T 8e Linux LVM

```

* `Disklabel type: dos`: Indicates the drive uses an MBR partition layout.
* `Disk identifier: 0x7c9a2b10`: 4-byte unique MBR signature stored at offset `0x1B8` in Sector 0.
* `Device`: Generated kernel device path.
* `Boot`: `*` indicates byte `0` of that entry is set to `0x80` (active/bootable).
* `Start / End`: The starting and ending 32-bit LBA sector addresses. Sector `2048` aligns the partition starting edge to $1\text{ MiB}$ ($2048 \times 512\text{ bytes} = 1048576\text{ bytes}$), preventing misaligned I/O operations on Advanced Format 4KiB physical sector drives.
* `Id`: Partition type byte in hexadecimal (`83` = Standard Linux Filesystem, `8e` = LVM physical volume).

---

### Tool: `gdisk /dev/sdX`

#### 1. Fundamental Purpose & Historical Evolution

`gdisk` (GPT fdisk) was written to manage GUID Partition Tables (GPT), which replaced the legacy MBR format. Unified Extensible Firmware Interface (UEFI) specifications defined GPT to overcome structural limitations in MBR:

* Addresses up to $9.4\text{ ZB}$ ($9.4 \times 10^{21}\text{ bytes}$) using 64-bit LBA sector counts.
* Supports 128 primary partition entries by default (eliminating the need for extended/logical partition chains).
* Improves data integrity by storing primary and backup headers, using 128-bit GUIDs to uniquely identify partitions, and protecting header structures with CRC32 checksums.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
gdisk Partition Table Processing
  |
  +-- Read LBA 0 (Protective MBR) -> Confirm 0xEE Type Byte
  |
  +-- Read LBA 1 (Primary GPT Header)
  |     |
  |     +-- Calculate CRC32 of LBA 1 -> Compare against Header CRC32 Field
  |
  +-- Read LBA 2-33 (Primary Partition Entry Array)
  |     |
  |     +-- Calculate CRC32 of Entry Array -> Compare against Header Field
  |
  +-- [If Corrupted]: Read Backup GPT Header from Last LBA Sector on Disk
  |
  +-- In-Memory Partition Editing
  |
  +-- Save Operations ('w')
        |
        +-- Write Primary GPT Header & Array (LBA 1-33)
        +-- Write Backup GPT Array & Header (Tail LBAs)
        +-- Issue ioctl(BLKPG) to notify block layer of updated layout

```

1. **Protective MBR Verification:** `gdisk` reads LBA 0 to inspect the Protective MBR. It expects a single partition entry of type `0xEE` spanning the entire size of the drive. This prevents legacy MBR utilities from misidentifying the disk as unpartitioned and overwriting GPT data structures.
2. **GPT Header Reading & CRC32 Verification:**
* Reads LBA 1 into a buffer containing the Primary GPT Header structure.
* Calculates the CRC32 checksum of the header bytes (with the CRC field zeroed during calculation) and compares it to the stored `HeaderCRC32` value.
* Reads LBA 2 through 33 (which hold the 128 partition entries) and verifies their integrity against `PartitionArrayCRC32`.


3. **Backup Recovery Mechanics:** If the primary header is corrupted or CRC validation fails, `gdisk` reads the Backup GPT Header located at the final sector of the physical disk ($LBA_{max}$). It then uses this backup data to restore the primary structures.
4. **Kernel Commit:** When changes are saved (`w`), `gdisk` updates both the primary (LBA 1–33) and backup GPT regions, then notifies the kernel using `ioctl(BLKPG)`.

#### 3. Disk Structure & On-Disk Layout Dynamics

```
GPT On-Disk Sector Layout Map
+--------------------------------------------------------------------------+
| LBA 0        : Protective MBR (0xEE Partition Entry spanning disk)       |
+--------------------------------------------------------------------------+
| LBA 1        : Primary GPT Header Structure (512 Bytes)                  |
+--------------------------------------------------------------------------+
| LBA 2 - 33   : Primary Partition Array (128 entries * 128 bytes each)   |
+--------------------------------------------------------------------------+
| LBA 34...    : User Allocation & File System Partition Space             |
+--------------------------------------------------------------------------+
| LBA Max - 32 : Backup Partition Array (128 entries * 128 bytes each)     |
+--------------------------------------------------------------------------+
| LBA Max      : Backup GPT Header Structure                               |
+--------------------------------------------------------------------------+

```

An individual **128-byte GPT Partition Entry** consists of:

```
GPT 128-Byte Partition Entry Map
Bytes 0 - 15   : Partition Type GUID (e.g. 0FC63DA8-8483-4772-8E79-3D69D8477DE4 for Linux FS)
Bytes 16 - 31  : Unique Partition GUID (Randomly generated 128-bit UUID)
Bytes 32 - 39  : Starting LBA Sector (64-bit Unsigned Little-Endian Integer)
Bytes 40 - 47  : Ending LBA Sector (64-bit Unsigned Little-Endian Integer)
Bytes 48 - 55  : Attribute Flags (Bit 0: System Partition, Bit 60: Read-Only, Bit 63: No Auto-Mount)
Bytes 56 - 127 : Partition Name String (36 characters in UTF-16LE Encoding)

```

#### 4. Line-by-Line Flag & Syntax Breakdown

* Executing `gdisk /dev/sdX` opens the target block device in interactive edit mode.
* Key interactive commands:
* `p`: Prints the validated GPT table layout, including GUIDs and sector alignments.
* `n`: Prompts for partition number ($1$-$128$), starting LBA sector (defaults to $2048$ for 1MiB alignment), ending sector, and Type Code (e.g., `8300` for Linux filesystem, `8e00` for LVM).
* `c`: Changes the UTF-16LE partition label string stored in bytes 56–127 of the partition entry.
* `w`: Synchronizes primary and backup GPT structures to disk and issues `ioctl(BLKPG)` to update the kernel block device mappings.



#### 5. Exhaustive Output Anatomy

```
GPT fdisk (gdisk) version 1.0.8

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT items, updating in-memory bytes.

Command (? for help): p
Disk /dev/sda: 3907029168 sectors, 1.8 TiB
Model: ST2000DM008-2FR1
Sector size (logical/physical): 512/4096 bytes
Disk identifier (GUID): 6F9A2B10-3C4D-5E6F-7A8B-9C0D1E2F3A4B
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 3907029134

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2099199   1000.0 MiB  EF00  EFI System Partition
   2         2099200      3907028991   1.8 TiB     8300  Linux filesystem

```

* `MBR: protective`: Indicates LBA 0 contains a valid `0xEE` Protective MBR.
* `Disk identifier (GUID)`: 128-bit GUID assigned to the disk array header at LBA 1 (Offset `56`).
* `First/Last usable sector`: Usable LBA boundary offsets, bound by the primary array ending at LBA 33 and the backup array starting at $\text{LBA}_{max} - 33$.
* `Code`: `gdisk` shortcut map for GUID definitions (`EF00` maps to the EFI System Partition GUID `C12A7328-F81F-11D2-BA4B-00A0C93EC93B`).

---

### Tool: `parted /dev/sdX`

#### 1. Fundamental Purpose & Historical Evolution

`parted` (GNU Partitioned) was created to provide a scriptable partition management tool across multiple disk label formats (MBR, GPT, BSD, Sun, MAC, DASD).

Unlike interactive tools such as `fdisk` or `gdisk`, `parted` exposes a non-interactive CLI interface via `libparted`. This design allows drive partition tables to be created, resized, aligned, and destroyed using single shell commands, making it ideal for automated OS installation scripts and cloud provisioning pipelines.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
parted Scriptable Execution Path
  |
  +-- Parse Command-Line Arguments (e.g. mkpart primary ext4 1MiB 100%)
  |
  +-- libparted Initialization
  |     |
  |     +-- Open target device with O_RDWR
  |     +-- Probe and auto-detect existing label type (ped_disk_probe)
  |
  +-- Sector Alignment Calculation Engine
  |     |
  |     +-- Query physical sector size (ioctl BLKPBSZGET)
  |     +-- Calculate alignment boundary (Grain size: 2048 sectors / 1 MiB)
  |
  +-- Write Partition Table Data Directly to Disk LBAs
  |
  +-- Issue Partition Refresh System Calls
        |
        +-- Loop: ioctl(fd, BLKPG_DEL_PARTITION)
        +-- Loop: ioctl(fd, BLKPG_ADD_PARTITION)
        +-- udev daemon receives KOBJ_CHANGE netlink event -> Generates /dev nodes

```

1. **`libparted` Abstraction:** `parted` serves as a CLI front-end for `libparted`. When invoked, `libparted` opens the target block device node and detects the existing label format using signature probing (`ped_disk_probe()`).
2. **Alignment Offset Logic:** `parted` calculates physical sector boundaries to ensure logical partition starts align with underlying physical hardware blocks:
* Queries minimum and optimal I/O sizes using `ioctl(fd, BLKIOOPT)` and `ioctl(fd, BLKPBSZGET)`.
* On Advanced Format drives ($4\text{ KiB}$ physical sectors), `parted` aligns starting sectors to multiples of 2048 ($1\text{ MiB}$). This prevents misaligned writes that would otherwise cause expensive read-modify-write performance penalties.


3. **Immediate On-Disk Commits:** Unlike `fdisk` (which holds changes in memory until `w` is issued), commands executed in `parted` modify disk structures immediately.
4. **Kernel Boundary Synchronization:** Calls `ioctl(fd, BLKPG)` to add or remove partitions in the kernel block layer. This triggers a Netlink event (`KOBJ_CHANGE`), instructing `udevd` to update node entries under `/dev/` and `/sys/class/block/`.

#### 3. Disk Structure & On-Disk Layout Dynamics

When `parted` creates a partition table using `mklabel gpt`, it zeroes LBA 0 through 33 and constructs the Protective MBR and primary GPT headers. When `mklabel msdos` is issued, it zeroes LBA 0, writes the boot record magic signature `0x55AA` to offsets `510–511`, and sets up four empty 16-byte partition entries starting at offset `0x1BE`.

#### 4. Line-by-Line Flag & Syntax Breakdown

* Command Syntax: `parted -a optimal /dev/sda mkpart primary ext4 1MiB 100%`
* `-a optimal`: Configures the partition alignment engine to align partition start and end points to optimal physical I/O boundaries (typically $1\text{ MiB}$ / 2048 sectors).
* `/dev/sda`: Target block device path.
* `mkpart`: Issues the sub-command to allocate a new partition.
* `primary`: Partition type parameter (required for MBR compatibility; ignored on GPT labels).
* `ext4`: Sets the filesystem type code hint in the partition entry table (sets the partition type GUID on GPT).
* `1MiB`: Starting sector offset. Using explicit unit suffixes (`MiB`, `GiB`) ensures exact boundary alignment.
* `100%`: Ending offset instructed to extend to the maximum available capacity on the physical disk.



#### 5. Exhaustive Output Anatomy

```
Model: ATA ST2000DM008-2FR1 (scsi)
Disk /dev/sda: 2000GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name  Flags
 1      1048kB  1074MB  1073MB  fat32              boot, esp
 2      1074MB  2000GB  1999GB  ext4

```

* `Sector size (logical/physical): 512B/4096B`: Logical sectors are $512\text{ bytes}$ for software compatibility, while physical sectors are $4096\text{ bytes}$ ($4\text{ KiB}$ Advanced Format).
* `Partition Table: gpt`: Label framework in use.
* `Flags: boot, esp`: Set attributes in the GPT Partition Entry. Bit `0` is configured to identify the partition as an EFI System Partition (type GUID `C12A7328-F81F-11D2-BA4B-00A0C93EC93B`).

---

### Commands: `mkfs.ext4 /dev/sdX1` / `mkfs.xfs /dev/sdX1`

#### 1. Fundamental Purpose & Historical Evolution

`mkfs` tools format block devices by constructing valid filesystem structures on disk.

* **Ext4 (Fourth Extended Filesystem):** Evolved from Ext2 and Ext3. It introduced 48-bit block addressing (supporting volumes up to $1\text{ EiB}$), extents (replacing indirect block mapping to reduce metadata overhead), delayed allocation (`delalloc`), multiblock allocation (`mballoc`), and fast journal checksumming (`jbd2`).
* **XFS:** Originally developed by Silicon Graphics (SGI) for IRIX, XFS was designed for scalable parallel I/O performance. It uses Allocation Groups (AGs), B-tree-indexed metadata structures, dynamic inode allocation, and extent-based block mapping.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

##### `mkfs.ext4 /dev/sdX1`:

```
mkfs.ext4 Initialization Pipeline
  |
  +-- 1. Open device O_RDWR | O_EXCL -> Verify block size via ioctl(BLKGETSIZE64)
  |
  +-- 2. Compute Layout (Flex_BG grouping, Inode counts, Journal size)
  |
  +-- 3. Write Primary Superblock at Offset 1024 (0x400)
  |
  +-- 4. Allocate and Write Block Group Descriptor Table (GDT)
  |
  +-- 5. Zero-fill and Initialize Inode Tables & Bitmaps for each Block Group
  |
  +-- 6. Construct Inode 8 (Journal Inode) -> Initialize JBD2 Journal Blocks

```

1. Opens `/dev/sdX1` with `O_RDWR | O_EXCL` and queries device geometry using `ioctl(BLKGETSIZE64)`.
2. Computes the layout for Block Groups, Flexible Block Groups (`flex_bg`), and inode ratio allocations.
3. Writes the Primary Superblock at byte offset `1024` with magic byte `0xEF53`.
4. Initializes the Group Descriptor Table (GDT), defining block bitmap, inode bitmap, and inode table locations for all block groups.
5. Constructs special system inodes:
* Inode 1: Bad block inode.
* Inode 2: Root directory (`/`).
* Inode 8: Journal inode (allocates continuous extents for the `jbd2` journal engine).



##### `mkfs.xfs /dev/sdX1`:

```
mkfs.xfs Initialization Pipeline
  |
  +-- 1. Query topology parameters (ioctl BLKALIGNOFF, BLKIOOPT)
  |
  +-- 2. Divide physical disk space into equal-sized Allocation Groups (AGs)
  |
  +-- 3. For each Allocation Group (AG 0 to AG N):
  |      +-- Write AG Superblock (AGF, AGI, AGFL headers) at Sector 0 of AG
  |      +-- Initialize Root Node for Free Space B-Trees (bnobt & cntbt)
  |      +-- Initialize Root Node for Inode Allocation B-Tree (inobt)
  |
  +-- 4. Construct Inode 128 (Root Directory) & Allocate Log AG Sub-region

```

1. Queries physical disk geometry and allocation alignment using `ioctl(BLKALIGNOFF)` and `ioctl(BLKIOOPT)`.
2. Divides the storage volume into independent Allocation Groups (AGs), typically allocating 4 to 64 AGs based on total size.
3. Initializes four core header structures at Sector 0 of **every** AG:
* `XFS_SB`: Superblock metadata.
* `XFS_AGF`: Free space management header.
* `XFS_AGI`: Inode allocation header.
* `XFS_AGFL`: Allocation Group Free List header.


4. Constructs the root nodes for the free space B-trees (`bnobt` sorted by block number, `cntbt` sorted by block count) and the inode B-tree (`inobt`).
5. Reserves internal log blocks (`xlog`) inside an AG or assigns an external log block device.

#### 3. Disk Structure & On-Disk Layout Dynamics

##### Ext4 Block Group Structural Layout:

```
+-----------------------------------------------------------------------------------+
| Boot Block | Superblock | Group Descriptors | Block Bitmap | Inode Bitmap | Inode |
| (1024 B)   | (1024 B)   | Table (GDT)       | (1 Block)    | (1 Block)    | Table |
+-----------------------------------------------------------------------------------+
| <------------------------------ Block Group 0 ----------------------------------> |

```

* Flexible Block Groups (`flex_bg`): Combines block bitmaps, inode bitmaps, and inode tables from multiple block groups into a single contiguous memory block. This reduces disk head movement and improves parallel I/O throughput.

##### XFS Allocation Group Structural Layout:

```
+-----------------------------------------------------------------------------------+
| AG Header (SB, AGF, AGI, AGFL) | Free Space B-Trees | Inode B-Tree | Dynamic Data |
| (First 4 Sectors of AG)        | (bnobt, cntbt)     | (inobt)      | Blocks       |
+-----------------------------------------------------------------------------------+
| <------------------------------ Allocation Group N -----------------------------> |

```

* Each AG operates as an independent metadata zone. Because each AG manages its own allocation locks, XFS can process concurrent write operations across multiple threads without lock contention.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `mkfs.ext4 -b 4096 -E lazy_itable_init=0 -O metadata_csum /dev/sda1`
* `-b 4096`: Explicitly sets the filesystem block allocation unit size to $4096\text{ bytes}$ ($4\text{ KiB}$).
* `-E lazy_itable_init=0`: Forces `mkfs.ext4` to zero-fill the inode table during formatting rather than relying on a background `kthread` (`ext4lazyinit`) after mounting.
* `-O metadata_csum`: Enables metadata checksumming (CRC32c) across superblocks, group descriptors, inodes, and extent trees.


* `mkfs.xfs -b size=4096 -d agcount=16 -f /dev/sdb1`
* `-b size=4096`: Sets the logical block allocation size to $4096\text{ bytes}$.
* `-d agcount=16`: Forces the filesystem layout to split into 16 independent Allocation Groups.
* `-f`: Forces formatting, overwriting any existing filesystem magic signatures on the block device.



#### 5. Exhaustive Output Anatomy

##### `mkfs.ext4` Output:

```
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 488378646 4k blocks and 122101760 inodes
Filesystem UUID: a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d
Superblock backups stored on blocks: 
    32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (262144 blocks): done
Writing superblocks and filesystem accounting information: done

```

* `488378646 4k blocks`: Total addressable block count ($488378646 \times 4096\text{ bytes} \approx 1.82\text{ TiB}$).
* `Superblock backups`: Duplicate backup superblocks written at sparse block locations (sparse_super mechanism, block groups 0, 1, and powers of 3, 5, 7) for disaster recovery.
* `Creating journal (262144 blocks)`: Allocates Inode 8 and assigns $262144 \times 4096 = 1\text{ GiB}$ for the JBD2 transaction log.

##### `mkfs.xfs` Output:

```
meta-data=/dev/sdb1              isize=512    agcount=16, agsize=30523665 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, spinodes=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=488378640, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=238466, version=2
blocks   =none                   bsize=4096   blocks=0, rtextents=0

```

* `isize=512`: Allocates $512\text{ bytes}$ per inode structure to accommodate extended attributes (`xattrs`) and inline data.
* `agcount=16`: Storage volume split across 16 independent Allocation Groups.
* `crc=1`: Enables CRC32c metadata verification across all AG structures.
* `finobt=1`: Free Inode B-Tree enabled to accelerate inode allocation tracking.
* `reflink=1`: Enables Copy-on-Write (CoW) extent sharing for instant file cloning (`cp --reflink`).

---

---

## SECTION 3: Mounting & Advanced Storage Technologies

---

### Command: `mount -o ro /dev/sdX1 /mnt`

#### 1. Fundamental Purpose & Historical Evolution

Mounting attaches a block device's filesystem hierarchy to a target directory in the VFS namespace. Early UNIX systems maintained a global mount table. Modern Linux uses per-process mount namespaces (`CLONE_NEWNS`), allowing isolated mount views for containers and service sandboxes.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
mount -o ro /dev/sdX1 /mnt
  |
  +-- Invokes move_mount / fsopen / fsmount OR mount() Syscall
  |     |
  |     +-- sys_mount("/dev/sdX1", "/mnt", "ext4", MS_RDONLY, NULL)
  |
  +-- VFS Path Resolution (kern_path) -> Resolve "/mnt" to struct path
  |
  +-- Kernel Filesystem Driver lookup (get_fs_type("ext4"))
  |
  +-- Superblock Reading & Initialization
  |     |
  |     +-- alloc_super() -> Allocates struct super_block
  |     +-- sb->s_op->fill_super() -> Reads sector 1024 into memory page
  |     +-- Validates magic number 0xEF53
  |
  +-- Read-Only Flag Enforcement
  |     |
  |     +-- Sets SB_RDONLY in sb->s_flags
  |     +-- Sets MNT_READONLY on struct vfsmount
  |     +-- Rejects subsequent write(), unlink(), or truncate() with -EROFS
  |
  +-- Mount Topology Update
        |
        +-- Attach vfsmount structure to target directory dentry in process mount namespace

```

1. **System Call Interface:** `mount` calls `sys_mount()` or the modern Mount API (`fsopen()`, `fsconfig()`, `fsmount()`, `move_mount()`):

$$\text{sys\_mount(const char *dev\_name, const char *dir\_name, const char *type, unsigned long flags, const void *data)}$$


2. **VFS Path Resolution:** The kernel resolves `dir_name` (`/mnt`) to a `struct path` containing a `struct dentry` and `struct vfsmount`.
3. **Driver Superblock Initialization:**
* Looks up the driver via `get_fs_type("ext4")`.
* Calls `alloc_super()` to allocate a `struct super_block` in kernel memory.
* Invokes `fill_super()` (`ext4_fill_super()`), which reads byte offset 1024 from the block device into the Page Cache and verifies the `0xEF53` magic byte signature.


4. **Read-Only Enforcements:**
* Passing `MS_RDONLY` sets the `SB_RDONLY` flag on `super_block->s_flags` and `MNT_READONLY` on `vfsmount->mnt_flags`.
* Any write path (`vfs_write()`, `vfs_unlink()`, `vfs_truncate()`) checks `IS_RDONLY(inode)` or `mnt_want_write()`. If set, the kernel immediately aborts the operation and returns `-EROFS` (Read-only file system).


5. **Namespace Binding:** The kernel binds the newly instantiated `vfsmount` structure to the target dentry within the calling process's mount namespace (`current->nsproxy->mnt_ns`).

#### 3. Disk Structure & On-Disk Layout Dynamics

During a read-only mount, the kernel does not replay pending journal transactions to disk (such as `jbd2` for Ext4 or `xlog` for XFS). If the journal is dirty, the mount may fail unless the `-o ro,noload` options are supplied. This instructs the kernel to mount the filesystem without processing the transaction log or writing recovery data to disk.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `-o ro`: Maps to the `MS_RDONLY` system call flag, mounting the filesystem in read-only mode.
* `/dev/sdX1`: Source block device node.
* `/mnt`: Target directory mount point within the VFS hierarchy.

#### 5. Output Anatomy

Executing `mount` produces no output on success ($0$ exit code). Inspecting `/proc/self/mountinfo` reveals:

```
28 1 8:1 / /mnt rw,nosuid,nodev,relatime ro,attr2,inode64,logbufs=8,noquota - xfs /dev/sda1 rw,seclabel

```

* `28`: Unique mount ID.
* `1`: Parent mount ID.
* `8:1`: Major and minor device numbers ($8 = \text{sd block driver}$, $1 = \text{partition 1}$).
* `/mnt`: VFS mount point location.
* `ro`: Confirms read-only operation enforcement across VFS layer execution routes.

---

### Command: `umount /mnt`

#### 1. Fundamental Purpose & Historical Evolution

`umount` detaches a mounted filesystem from the VFS hierarchy. It flushes dirty page cache buffers to disk, completes pending journal transactions, and unregisters the filesystem's `super_block` structure from kernel memory.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
umount /mnt Processing Path
  |
  +-- sys_umount2(target="/mnt", flags=0)
  |
  +-- Lookup vfsmount & Validate Busy Status
  |     |
  |     +-- Check mnt_count / active open file descriptors
  |     +-- [If Open Handles Exist]: Return -EBUSY (Device or resource busy)
  |
  +-- Flush Dirty Page Cache Pages (sync_filesystem)
  |     |
  |     +-- Write out dirty pages -> Flush metadata blocks -> Commit journal
  |
  +-- VFS Mount Point Unlinking
  |     |
  |     +-- Detach vfsmount from namespace parent tree
  |     +-- Call sb->s_op->put_super() to release filesystem-specific resources
  |
  +-- Deallocate super_block Memory Structure

```

1. **System Call:** Invokes `sys_umount2(const char *target, int flags)`.
2. **Busy Reference Check:**
* Checks the reference counter (`mnt_count`) on the `vfsmount` structure.
* If a process holds an open file descriptor (`sys_open()`), working directory (`chdir()`), or active memory mapping (`mmap()`) pointing to a file on that mount, the kernel aborts the unmount operation and sets `errno` to `EBUSY`.


3. **Data Flushing:**
* Calls `sync_filesystem()`, forcing all dirty page cache buffers associated with the filesystem to write back to physical disk.
* Ext4 calls `jbd2_journal_flush()` to commit all pending transactions and clear the dirty bit on the journal superblock.


4. **VFS Detachment & Superblock Teardown:**
* Unlinks the `vfsmount` structure from the process mount namespace.
* Invokes `sb->s_op->put_super()`, freeing kernel memory buffers allocated for the filesystem's metadata structures.



#### 3. Disk Structure & On-Disk Layout Dynamics

During clean unmounts, the kernel writes state flags directly to disk. For Ext4, it sets `s_state = EXT4_VALID_FS` (0x0001) in the superblock header at byte offset 1024. On subsequent boots, the filesystem driver reads this flag to confirm the previous session unmounted cleanly, allowing it to skip a full `e2fsck` consistency check.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `/mnt`: Mount point path or device node to unmount.
* Advanced System Call Flags (via `umount2`):
* `MNT_FORCE` (`umount -f`): Forces unmounting on unresponsive network filesystems (NFS).
* `MNT_DETACH` (`umount -l`): Performs a "lazy" unmount. It immediately hides the mount point from the VFS namespace while keeping the underlying filesystem active until all open file descriptors are closed.



#### 5. Output Anatomy

Executes silently on success ($0$ exit code). If open file handles exist on the target mount point, it displays:

```
umount: /mnt: target is busy.

```

---

### Commands: `pvcreate`, `vgcreate`, `lvcreate`

#### 1. Fundamental Purpose & Historical Evolution

Logical Volume Management (LVM2) decouples virtual storage mappings from physical storage hardware using the kernel's Device Mapper (`dm-core`) framework.

* Traditional partitioning binds filesystems directly to fixed physical disk sectors.
* LVM abstracts physical drives into Physical Volumes (PVs), aggregates PVs into flexible storage pools called Volume Groups (VGs), and allocates Logical Volumes (LVs) from these pools. This abstraction enables transparent online volume resizing, spanning multiple drives, live data migration, and instant Copy-on-Write snapshots.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
LVM Subsystem Abstraction Architecture

+-----------------------------------------------------------------------+
| User Space Tool: lvcreate / vgcreate / pvcreate                       |
|   1. Computes PE allocation map                                       |
|   2. Writes ASCII Volume Group Metadata Area (MDA) header to disk     |
|   3. Calls libdevmapper -> ioctl(DM_DEV_CREATE) & ioctl(DM_TABLE_LOAD) |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| Kernel Block Layer: Device Mapper Core (dm-core)                      |
|                                                                       |
| /dev/mapper/vg0-lv0  [Virtual Block Device Node]                      |
|   |                                                                   |
|   +--> Targets Mapping Array (dm-linear)                              |
|          |                                                            |
|          +-- Sector Range 0..2097151     -> Mapping -> /dev/sda1 (PEs) |
|          +-- Sector Range 2097152..4194303 -> Mapping -> /dev/sdb1 (PEs) |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| Physical Block Devices: /dev/sda1, /dev/sdb1 (LVM PV Metadata)        |
+-----------------------------------------------------------------------+

```

1. **`pvcreate /dev/sdX1`:**
* Opens `/dev/sdX1` with `O_RDWR | O_EXCL`.
* Writes the LVM Label Header at Sector 1 (byte offset `512`), which includes the PV UUID and physical extent alignment details.
* Constructs the Volume Group Metadata Area (MDA) circular buffer, which stores LVM configuration details in an ASCII descriptor format.


2. **`vgcreate vg0 /dev/sdX1 /dev/sdY1`:**
* Reads the PV headers on `/dev/sdX1` and `/dev/sdY1`.
* Groups both devices into a unified pool, defining a uniform Physical Extent size (PE size, default $4\text{ MiB}$).
* Writes the updated VG metadata descriptor to the MDA circular buffers on both physical volumes.


3. **`lvcreate -L 10G -n lv0 vg0`:**
* Calculates how many Logical Extents (LEs) are needed ($10\text{ GiB} / 4\text{ MiB} = 2560\text{ LEs}$) and maps them to available physical extents (PEs) across the PVs.
* Uses `libdevmapper` to send `ioctl(DM_DEV_CREATE)` to `/dev/mapper/control`, creating the block device node `/dev/mapper/vg0-lv0` (or major/minor `dm-N`).
* Sends an `ioctl(DM_TABLE_LOAD)` call passing a mapping table that defines how the virtual sector range maps to the underlying physical drive sectors:
`0 20971520 linear /dev/sda1 2048`
* Calls `ioctl(DM_DEV_SUSPEND)` and `ioctl(DM_DEV_RESUME)` to atomically activate the new device-mapper block target.



#### 3. Disk Structure & On-Disk Layout Dynamics

```
Physical Volume (PV) On-Disk Sector Layout
+-----------------------------------------------------------------------+
| Sector 0          : MBR / Partition Table Header space (or unused)     |
| Sector 1          : LVM Label Header ("LVM2 001", PV UUID, MDA Offsets)|
| Sectors 2 - 255   : Volume Group Metadata Area 1 (MDA Circular Buffer) |
| Sector 2048+      : Data Area Boundary (Physical Extent 0 Start)      |
+-----------------------------------------------------------------------+

```

##### ASCII Metadata Format (Inside MDA):

```hcl
vg0 {
    id = "3a1b2c-4d5e-6f7a-8b9c-0d1e2f3a4b5c"
    seqno = 2
    extent_size = 8192 # Sector count (4 MiB)
    
    physical_volumes {
        pv0 {
            id = "f1e2d3-c4b5-a698-7766-554433221100"
            device = "/dev/sda1"
            dev_size = 20971520
            pe_start = 2048
        }
    }
    logical_volumes {
        lv0 {
            id = "001122-3344-5566-7788-99aabbccddeeff"
            segment1 {
                start_extent = 0
                extent_count = 2560
                type = "striped"
                stripe_count = 1
                stripes = ["pv0", 0]
            }
        }
    }
}

```

#### 4. Line-by-Line Flag & Syntax Breakdown

* `pvcreate /dev/sda1`: Writes LVM labels and an empty MDA area to physical partition `/dev/sda1`.
* `vgcreate vg0 /dev/sda1 /dev/sdb1`: Merges physical volumes `/dev/sda1` and `/dev/sdb1` into a logical storage pool named `vg0`.
* `lvcreate -L 10G -n lv0 vg0`:
* `-L 10G`: Defines the volume allocation size in binary units ($10\text{ GiB}$).
* `-n lv0`: Assigns the logical volume name (`lv0`).
* `vg0`: Identifies the target Volume Group pool from which free extents will be allocated.



#### 5. Output Anatomy

```
  Physical volume "/dev/sda1" successfully created.
  Volume group "vg0" successfully created
  Logical volume "lv0" created.

```

Subsystem validation using `dmsetup table /dev/mapper/vg0-lv0`:

```
0 20971520 linear 8:1 2048

```

* `0`: Starting virtual sector on the `dm-0` target device.
* `20971520`: Total sector length ($20971520 \times 512 = 10.73\text{ GB}$).
* `linear`: Target driver module mapped inside kernel space (`dm-linear.ko`).
* `8:1`: Major and minor number of physical backing device (`/dev/sda1`).
* `2048`: Starting physical offset sector on physical device `/dev/sda1`.

---

### Command: `lvextend -r -L +10G /dev/vg0/lv0`

#### 1. Fundamental Purpose & Historical Evolution

`lvextend` increases the size of an LVM logical volume and its underlying filesystem online, without interrupting running applications or unmounting the filesystem.

Historically, expanding volume capacity required taking the filesystem offline, unmounting it, extending the partition boundary, running disk geometry checks, and re-mounting the volume. The combination of Device Mapper dynamic mapping and filesystem online resizing capabilities (`EXT4_IOC_RESIZE_FS`, `xfs_growfs`) allows storage capacity to be scaled live on production systems.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
lvextend -r Live Execution Sequence

1. LVM Allocates Unused Physical Extents (PEs)
2. Load Updated Device Mapper Mapping Table into Memory via ioctl(DM_TABLE_LOAD)
3. Issue ioctl(DM_DEV_SUSPEND) -> Freeze Active I/O Operations on Target LV
4. Issue ioctl(DM_DEV_RESUME)  -> Atomically Swap Active Mapping Table & Resume I/O
5. Exec Flags '-r' Triggers Online Filesystem Expansion Syscall:
   + Ext4: ioctl(EXT4_IOC_RESIZE_FS, &new_block_count)
   + XFS : ioctl(XFS_IOC_FSGROWFSDATA, &xfs_growfs_data)

```

1. **LVM Unallocated PE Search:** Scans the target Volume Group (`vg0`) descriptor map for unallocated Physical Extents (PEs).
2. **Device Mapper Table Construction:** Generates an updated mapping table appending the newly assigned PEs to the existing device layout.
3. **Atomic Table Swap (`dm_suspend` / `dm_resume`):**
* Issues `ioctl(DM_TABLE_LOAD)` to load the newly constructed extent mapping table into a passive kernel memory slot.
* Issues `ioctl(DM_DEV_SUSPEND)`. The `dm-core` layer temporarily pauses incoming I/O requests for `/dev/mapper/vg0-lv0` and queues them in system memory.
* Atomically updates the active device mapper execution pointer to reference the new linear target table.
* Issues `ioctl(DM_DEV_RESUME)`, unfreezing queued I/O requests and resuming normal disk operations.


4. **Filesystem Online Expansion (`-r` flag invocation):**
* `lvextend` inspects the volume's superblock to identify the active filesystem type.
* **Ext4 Resizing:** Issues `ioctl(fd, EXT4_IOC_RESIZE_FS, &new_block_count)`. The kernel `ext4` driver dynamically adds new block group descriptors, allocates new block bitmaps, updates the superblock's `s_blocks_count` field, and expands memory structures on the fly.
* **XFS Resizing:** Issues `ioctl(fd, XFS_IOC_FSGROWFSDATA, struct xfs_growfs_data *in)`. The kernel `xfs` driver constructs new Allocation Groups (AGs) over the newly added sectors, initializes their header structures (`AGF`, `AGI`, `AGFL`), and links the new capacity into the active storage pool.



#### 3. Disk Structure & On-Disk Layout Dynamics

During online expansion, the LVM volume group metadata descriptor (MDA) updates its ASCII sequence number (`seqno = seqno + 1`) and appends an additional segment descriptor:

```hcl
segment2 {
    start_extent = 2560
    extent_count = 2560
    type = "striped"
    stripe_count = 1
    stripes = ["pv0", 2560]
}

```

On the filesystem layer:

* **Ext4:** Generates new Block Group Descriptor entries in memory and commits them to disk, updating the free block bitmaps without changing existing data extents.
* **XFS:** Appends new Allocation Groups (AGs) to the end of the filesystem, keeping existing AG structures intact while making the new space immediately available for allocation.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `lvextend`: Main executable binary for logical volume expansion.
* `-r` (`--resizefs`): Instructs LVM to automatically expand the underlying filesystem after increasing the logical volume size.
* `-L +10G`: Specifies the capacity increment ($+10\text{ GiB}$) to append to the volume's current capacity.
* `/dev/vg0/lv0`: Target logical volume path.

#### 5. Output Anatomy

```
  Size of logical volume vg0/lv0 changed from 10.00 GiB (2560 extents) to 20.00 GiB (5120 extents).
  Logical volume vg0/lv0 successfully resized.
meta-data=/dev/mapper/vg0-lv0    isize=512    agcount=4, agsize=655360 blks
         =                       sectsz=4096  attr=2, projid32bit=1
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
blocks   =none                   bsize=4096   blocks=0, rtextents=0
data blocks changed from 2621440 to 5242880

```

* `agcount`: Reflects updated filesystem AG structures or block group counts.
* `data blocks changed from 2621440 to 5242880`: Confirms $10\text{ GiB}$ ($2621440 \times 4096\text{ bytes}$) of online capacity was appended to the active filesystem block allocator.

---

### Command: `lvconvert --merge /dev/vg0/snap`

#### 1. Fundamental Purpose & Historical Evolution

`lvconvert --merge` reverts an origin Logical Volume to a previous point-in-time state using an LVM Copy-on-Write (CoW) snapshot.

When a classic LVM snapshot is created, it tracks changes made to the origin volume. When data on the origin volume is modified, LVM copies the original sector contents to the snapshot volume before overwriting them. Restoring an origin volume traditionally required taking the storage offline and running raw block-copy operations (`dd`). Merging allows the snapshot changes to be merged back into the origin volume live or upon the next activation, streamlining disaster recovery operations.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
LVM Snapshot Merging Architecture
  |
  +-- Execute lvconvert --merge /dev/vg0/snap
  |
  +-- Instantiates kernel Target Driver: dm-snapshot-merge
  |
  +-- Background Kernel Thread (kcopyd) Iterates Chunk Exception Map:
  |     +-- Read allocated historical chunk from Snapshot Exception Store
  |     +-- Write chunk back to physical offset on Origin Volume
  |     +-- Free mapped exception entries from Snapshot Metadata Table
  |
  +-- Concurrent I/O Handling during Merge:
  |     +-- Read Request  -> Serviced from Snapshot if exception exists, else from Origin
  |     +-- Write Request -> Writes directly to Origin; invalidates matching exception entry
  |
  +-- All Exception Chunks Copied -> Destroy Snapshot Target -> Restore dm-linear Target

```

1. **Target Conversion:** Transforms the Device Mapper target for `/dev/vg0/snap` into a `dm-snapshot-merge` kernel device mapper module.
2. **Background Copy Engine (`kcopyd`):** The kernel spawns asynchronous `kcopyd` background threads that iterate through the snapshot exception store.
3. **Exception Table Processing:**
* Reads historical storage chunks from the snapshot volume and writes them back to their original physical offsets on the origin volume.
* Once a chunk is copied back to the origin volume, its entry is removed from the snapshot exception map.


4. **Transparent Concurrent I/O:**
* **Read Requests:** If an I/O request targets an offset that has not yet been merged, `dm-snapshot-merge` reads the historical block from the snapshot target. Otherwise, it reads directly from the origin volume.
* **Write Requests:** Incoming writes are written directly to the origin volume, and any matching exception entry in the snapshot table is removed.


5. **Teardown:** Once all exception chunks are copied back to the origin, `dm-snapshot-merge` destroys the snapshot metadata store and converts the origin volume back to a standard `dm-linear` mapping table. If the origin volume is active and cannot be merged immediately, the merge operation is deferred and finishes automatically during the next volume activation (`vgchange -ay`).

#### 3. Disk Structure & On-Disk Layout Dynamics

```
Snapshot CoW Exception Table Layout
+-------------------------------------------------------------------------+
| Header: Magic Bytes "SNAP", Version, Chunk Size (e.g. 64 KiB), Flags     |
+-------------------------------------------------------------------------+
| Map Entry 0 : [Origin Chunk Offset #1204] -> [Snapshot Sector Offset #64]|
| Map Entry 1 : [Origin Chunk Offset #4092] -> [Snapshot Sector Offset #128|
+-------------------------------------------------------------------------+

```

During a snapshot merge, `kcopyd` reads exception pairs starting from the last entry in the list and writes them back to the origin volume. As each chunk copy completes, the metadata table header decrements its active chunk counter until no exceptions remain.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `lvconvert`: Main LVM volume state transition and conversion utility.
* `--merge`: Directs LVM to merge the specified snapshot logical volume back into its parent origin volume.
* `/dev/vg0/snap`: Target snapshot logical volume path.

#### 5. Output Anatomy

```
  Merging of volume vg0/snap started.
  vg0/snap: Merged: 42.4%
  vg0/snap: Merged: 100.0%
  Merge of snapshot logical volume vg0/snap completed.
  Logical volume "snap" successfully removed.

```

* `Merged: 42.4%`: Real-time progress updates reported by `dm-snapshot-merge` as `kcopyd` processes chunks from the exception store.
* `Logical volume "snap" successfully removed`: Indicates all exception chunks were restored to the origin volume, and the snapshot target has been safely decommissioned.

---

### Commands: `zpool`, `zfs`

#### 1. Fundamental Purpose & Historical Evolution

ZFS (Zettabyte File System) was designed by Sun Microsystems to eliminate traditional storage abstractions (partition tables, volume managers, formatting tools).

Traditional storage stacks layer independent subsystems on top of each other—filesystems sit on top of logical volume managers, which sit on top of hardware partitions. This separation creates synchronization challenges (such as the "RAID write hole" and silent data corruption due to bit rot). ZFS replaces this layered approach with an integrated architecture that combines the Storage Pool Allocator (SPA), Data Management Unit (DMU), and ZFS POSIX Layer (ZPL).

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
ZFS Layered System Architecture

+-----------------------------------------------------------------------+
| ZPL (ZFS POSIX Layer) - Exposes VFS interfaces (files, directories)   |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| DMU (Data Management Unit) - Transactional CoW Object Engine         |
|   - Handles Datasets, Snapshots, Clones, & LZ4/ZSTD Compression       |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| SPA (Storage Pool Allocator) - Virtual Devices (vdevs) Management     |
|   - Calculates RAID-Z Parity & Manages Hardware Disk Interfaces       |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
| ZIL (ZFS Intent Log) & SLOG       | L2ARC (SSD Read Cache)            |
+-----------------------------------------------------------------------+

```

1. **Transaction Groups (TXGs):** ZFS aggregates write operations into global Transaction Groups (TXGs) in memory. Every 5 seconds (or when dirtied memory thresholds are reached), the SPA flushes the active TXG to disk using Copy-on-Write (CoW).
2. **Copy-on-Write (CoW) Allocation Mechanics:** ZFS never overwrites data blocks in place. When a block is modified, ZFS allocates a new block elsewhere in the pool and writes the modified data to it. Parent block pointers are then updated recursively up the tree to the root node (the Uberblock).
3. **End-to-End Integrity Verification (Merkle Trees):**
* Every block pointer (`blkptr_t`) stores a 256-bit checksum (such as Fletcher4 or SHA-256) of the data block it points to.
* When data is read from disk, ZFS calculates its checksum and compares it against the expected checksum value stored in its parent block pointer.
* If the checksums do not match (indicating bit rot or silent data corruption), ZFS uses redundant RAID-Z or mirror parity blocks to automatically repair the corrupted data and write the corrected block back to disk.



#### 3. Disk Structure & On-Disk Layout Dynamics

##### `blkptr_t` (ZFS Block Pointer Structural Mapping):

```
ZFS 128-Byte Block Pointer Layout
+-------------------------------------------------------------------------+
| DVA[0] (Data Virtual Address: vdev_id, offset, asize)                   |
| DVA[1] (Data Virtual Address: Mirror/Redundant Replica Offset)          |
| DVA[2] (Data Virtual Address: Tertiary Mirror Replica Offset)           |
| Props: Compres, Endianness, Type, LSIZE (Logical), PSIZE (Physical)     |
| Transaction Group Counter (TXG Generation Number)                       |
| 256-Bit Checksum (4 x 64-bit SHA256 / Fletcher4 Hash Values)             |
+-------------------------------------------------------------------------+

```

##### ZFS Uberblock Ring Buffer:

```
Pool Label Sector (First 256 KiB of Disk Vdev)
+-------------------------------------------------------------------------+
| Boot Header (8 KiB) | Vdev Configuration Tree (112 KiB)                 |
| Uberblock Ring Buffer (128 KiB: Array of 128 x 1 KiB Uberblock Structs) |
+-------------------------------------------------------------------------+

```

* The **Uberblock** serves as the absolute root of the ZFS Merkle tree structure. During a transaction commit, ZFS writes a new Uberblock entry with an incremented TXG generation counter to the ring buffer. A transaction group commit becomes atomic the moment its updated Uberblock is written to disk.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `zpool create -f -o ashift=12 tank raidz1 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1`:
* `create`: Initializes a new storage pool.
* `-f`: Forces pool creation, overriding existing partition or filesystem signatures.
* `-o ashift=12`: Sets the vdev block allocation size to $2^{12} = 4096\text{ bytes}$ ($4\text{ KiB}$ alignment for Advanced Format drives).
* `tank`: Name assigned to the root zpool.
* `raidz1`: Configures single-parity RAID-Z across the member drives (similar to RAID-5, but avoids the write hole by using variable stripe widths).
* `/dev/nvme0n1 ...`: Target block devices assigned to the top-level vdev array.


* `zfs create -o compression=lz4 tank/data`:
* `create`: Instantiates a new ZFS dataset under the `tank` pool.
* `-o compression=lz4`: Enables transparent LZ4 compression for all block writes within the dataset.



#### 5. Output Anatomy

##### `zpool status`:

```
  pool: tank
 state: ONLINE
  scan: scrub repaired 0B in 00:02:14 with 0 errors on Thu Aug 20 10:00:00 2026
config:

	NAME        STATE     READ WRITE CKSUM
	tank        ONLINE       0     0     0
	  raidz1-0  ONLINE       0     0     0
	    nvme0n1 ONLINE       0     0     0
	    nvme1n1 ONLINE       0     0     0
	    nvme2n1 ONLINE       0     0     0

errors: No known data errors

```

* `raidz1-0`: Virtual device (vdev) group providing single-parity data protection.
* `READ / WRITE / CKSUM`: Displays I/O errors and checksum validation failures detected during read/write operations or background pool scrubs.

---

### Command: `btrfs subvolume snapshot / /snapshots/root-bak`

#### 1. Fundamental Purpose & Historical Evolution

Btrfs (B-Tree Filesystem) was created to bring modern Copy-on-Write capabilities, online scaling, dynamic subvolume management, and self-healing storage functionality natively to Linux filesystems.

Traditional Linux filesystems rely on fixed inode tables and linear block bitmaps, making features like snapshots difficult to implement efficiently. Btrfs addresses this by structuring all filesystem metadata—including files, extents, devices, and subvolumes—into dynamic, searchable B-Trees.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
Btrfs Copy-on-Write Subvolume Snapshotting

Root Tree Array Pointer
  |
  +-- [ Subvolume Root Tree Node A (Generation 100) ]
        |
        +-- Extent Item Reference #4029 ---> Physical Data Block Extent #8812
        |
        +-- Extent Item Reference #4030 ---> Physical Data Block Extent #9914

[ Execute btrfs subvolume snapshot / /snapshots/root-bak ]
  |
  +-- Instant Copy-on-Write Duplication:
        |
        +-- [ Snapshot Subvolume Root Node B (Generation 101) ]
              |
              +-- Points to SAME Extent #8812 (Ref Count = 2)
              +-- Points to SAME Extent #9914 (Ref Count = 2)

[ Subsequent Write to File in Snapshot B ]:
  |
  +-- Allocate NEW Extent #10412 -> Point Snapshot Node B to Extent #10412
  +-- Original Subvolume Root Node A remains pointing to Extent #8812

```

1. **Subvolume Metadata Structure:** In Btrfs, a subvolume is an independent, dynamically allocated tree structure (Root Tree ID) with its own inode and object indexing spaces.
2. **Instant Snapshotting Engine:**
* When `btrfs subvolume snapshot` is executed, the kernel duplicates the **root node** of the source subvolume tree (`/`).
* It assigns a new Root Object ID to the snapshot subvolume tree (`/snapshots/root-bak`).
* The operation does not copy any underlying file data blocks. Instead, the newly created snapshot tree points directly to the existing extents referenced by the source subvolume.
* The kernel increments the reference counter (`btrfs_extent_item`) for all shared extents in the Extent B-Tree.


3. **Copy-on-Write Isolation:**
* When a file in either the source subvolume or the snapshot is modified, Btrfs allocates a new extent on disk for the updated data blocks.
* The modified subvolume tree is updated to point to the newly allocated extent, while the other subvolume tree continues pointing to the original, unreferenced extent.



#### 3. Disk Structure & On-Disk Layout Dynamics

##### Btrfs B-Tree Structural Layout:

```
Btrfs B-Tree Generic Node Structural Map
+-------------------------------------------------------------------------+
| Header: Checksum (CSUM 32B), FS UUID, Block Number, Generation (transid)|
| Key Item 0: [Object ID | Type Flag | Offset Descriptor]                |
| Key Item 1: [Object ID | Type Flag | Offset Descriptor]                |
| Data Offset Pointer Array (Growing downward toward header)             |
+-------------------------------------------------------------------------+

```

##### Core Global B-Trees in Btrfs Architecture:

* **Root Tree:** Stores reference pointers to all active subvolumes, snapshots, and operational trees.
* **Extent Tree:** Tracks space allocations and manages block extent reference counters across all subvolumes.
* **Chunk Tree:** Maps logical filesystem addresses to physical storage device offsets across attached drives.
* **FS Tree:** Stores directory structures, inode descriptors, and extended attributes for a specific subvolume.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `btrfs subvolume snapshot`: Command to create an instant Copy-on-Write snapshot of a target subvolume.
* `/`: Target source subvolume path to snapshot.
* `/snapshots/root-bak`: Destination directory path where the new snapshot subvolume tree will be created.
* Optional Flag: `-r`: Creates a read-only snapshot. Sets the `BTRFS_SUBVOL_RDONLY` flag on the root item descriptor in the Root Tree, preventing any subsequent modifications to the snapshot's extents.

#### 5. Output Anatomy

```
Create a snapshot of '/' in '/snapshots/root-bak'

```

System verification using `btrfs subvolume list /`:

```
ID 256 gen 1024 top level 5 path root
ID 280 gen 1024 top level 5 path snapshots/root-bak

```

* `ID 280`: Unique 64-bit object identifier assigned to the new snapshot subvolume tree.
* `gen 1024`: Transaction generation counter (`transid`) when the snapshot was created.

---

### Command: `losetup -fP image.img`

#### 1. Fundamental Purpose & Historical Evolution

The loop device subsystem (`loop.ko`) allows standard disk image files stored on a filesystem to be mounted and accessed as raw block devices (`/dev/loopN`).

Historically, testing partition schemes, mounting disk image backups, or analyzing disk dumps required flashing the image onto a physical drive. The loop driver abstracts virtual disk files into block devices, allowing the kernel block layer to parse partition tables, create virtual partition nodes, and format or mount file-backed images directly.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
Loop Virtualization Driver Architecture

VFS Application Request -> /dev/loop0p1 (Partition Target)
                               |
                               v
Kernel Block Layer: loop.ko Driver Thread (loop_kthread)
  |
  +-- Translates Partition LBA Offset (e.g. Sector 2048 * 512 = 1048576)
  +-- Invokes VFS File System Layer: filp_open() on target file
  |
  +-- Translates I/O to vfs_read() / vfs_write() operations
        |
        v
Underlying Hardware File System (e.g. /home/user/image.img on Ext4)

```

1. **Virtual Block Device Allocation:** `losetup` opens `/dev/loop-control` and issues `ioctl(LOOP_CTL_GET_FREE)` to find an unused loop device node (e.g., `/dev/loop0`).
2. **File Binding:** Opens `image.img` and binds it to the allocated loop device using `ioctl(loop_fd, LOOP_SET_FD, image_fd)`.
3. **Kernel Thread Processing (`loop_kthread`):** The driver spawns a dedicated kernel thread (`loop_kthread`). Incoming I/O requests (`submit_bio`) sent to `/dev/loop0` are converted into underlying file reads and writes using internal `vfs_read()` and `vfs_write()` calls against the backing file.
4. **Partition Scanning (`-P` flag):**
* Issues `ioctl(loop_fd, BLKRRPART)`.
* The kernel block layer scans the backing file's partition table (such as MBR or GPT) and creates partition device nodes (e.g., `/dev/loop0p1`, `/dev/loop0p2`) using `ioctl(BLKPG)`.



#### 3. Disk Structure & On-Disk Layout Dynamics

The loop device maps physical disk structures directly to byte offsets within the backing image file:

* Offset `0`: Represents Sector 0 (MBR or Protective MBR) of the virtual disk.
* Offset `1048576` ($1\text{ MiB}$): Sector 2048, marking the start of partition 1 (`/dev/loop0p1`).
* Reads or writes to `/dev/loop0p1` are shifted by byte offset `1048576` before being written to `image.img`.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `losetup`: Loop block device management utility.
* `-f` (`--find`): Requests the next available loop device from `/dev/loop-control`.
* `-P` (`--partscan`): Forces the kernel to scan the backing image file for partition tables and automatically create partition device nodes (`/dev/loopXpN`).
* `image.img`: Path to target raw disk file.

#### 5. Output Anatomy

Executing `losetup -fP image.img` executes silently ($0$ exit status). Inspecting system bindings via `losetup -a`:

```
/dev/loop0: [2050]:2097152 (/home/user/image.img) back-size=10737418240 offset=0

```

Inspect partition node creation via `ls -l /dev/loop0*`:

```
brw-rw---- 1 root disk 7, 0 Aug 20 10:30 /dev/loop0
brw-rw---- 1 root disk 259, 1 Aug 20 10:30 /dev/loop0p1
brw-rw---- 1 root disk 259, 2 Aug 20 10:30 /dev/loop0p2

```

* `7, 0`: Major device number `7` (standard Linux loop block driver) and minor device number `0`.
* `259, 1`: Dynamically assigned block major/minor numbers for virtual partition nodes created via kernel partition scanning.

---

### Command: `cryptsetup luksOpen /dev/sdX1 secure_volume`

#### 1. Fundamental Purpose & Historical Evolution

LUKS (Linux Unified Key Setup) provides a standardized, implementation-independent on-disk format for block-level encryption using the `dm-crypt` kernel module.

Legacy disk encryption tools stored raw encrypted data without standardized headers or key management metadata. LUKS introduced a structured header containing key slot metadata, digest algorithms, and Anti-Forensic Split (AF-splitter) masks. This layout supports multiple user passphrases, seamless key revocation, and Argon2id Key Derivation Function (KDF) hardness hardening to mitigate brute-force attacks.

#### 2. Under-the-Hood Execution & Kernel Mechanisms

```
cryptsetup luksOpen Execution Path

1. User Space: Cryptsetup reads LUKS Header at Offset 0 of /dev/sdX1
2. Extract Active Keyslot Metadata (Argon2id KDF Params, Salt, Iterations)
3. Prompt for Passphrase -> Run Argon2id KDF -> Derive Key-Encrypting-Key (KEK)
4. Decrypt Master Key Payload from Key Slot Offset
5. Verify Master Key Digest against Stored Header Hash
6. Pass Unwrapped Master Key to Kernel Keyring API (keyctl)
7. Invoke libdevmapper -> Issue ioctl(DM_DEV_CREATE) & ioctl(DM_TABLE_LOAD)
   + Pass crypto mapping string: "crypt aes-xts-plain64 [MasterKey] 0 /dev/sdX1 4096"
8. Creates decrypted virtual block device: /dev/mapper/secure_volume

```

1. **Header Parsing:** `cryptsetup` opens `/dev/sdX1` and reads the LUKS header at byte offset `0`.
2. **Key Derivation Function (KDF):** Reads the active keyslot metadata (Argon2id params, salt, memory limit, time cost) and passes the user passphrase to the Argon2id KDF pipeline. This derives a intermediate Key-Encrypting Key (KEK).
3. **Master Key Unwrapping:** Uses the derived KEK to decrypt the master key payload stored in the designated keyslot region.
4. **Digest Verification:** Calculates the cryptographic digest of the unwrapped master key and compares it to the stored `hdr_digest` string. If they match, authentication succeeds.
5. **Kernel Keyring Registration:** Loads the decrypted master key into the Linux Kernel Keyring subsystem (`keyctl`). This prevents the master key from remaining in user-space application memory.
6. **Device Mapper Target Activation:**
* Issues `ioctl(DM_DEV_CREATE)` to create `/dev/mapper/secure_volume`.
* Sends `ioctl(DM_TABLE_LOAD)` passing the `dm-crypt` target specification:
`0 20971520 crypt aes-xts-plain64 :64:logon:cryptsetup:... 0 /dev/sdX1 4096`


7. **Transparent Encryption Pipeline:** Sector reads/writes sent to `/dev/mapper/secure_volume` are processed through the `dm-crypt` kernel thread. The thread uses the Linux Crypto API (`crypto_alloc_skcipher`) to execute symmetric AES-XTS encryption/decryption operations on the fly before passing I/O requests down to the underlying physical storage controller.

#### 3. Disk Structure & On-Disk Layout Dynamics

##### LUKS2 Header Structural Layout:

```
+-------------------------------------------------------------------------+
| Offsets 0x00 - 0x05 (6 bytes)  : Magic Bytes ("LUKS\xba\xbe")          |
| Offsets 0x08 - 0x09 (2 bytes)  : Version Identifier (2)                 |
| Offsets 0x0C - 0x1B (16 bytes) : Header Size & Primary Sequence Count   |
| Offsets 0x28 - 0x4F (40 bytes) : Volume UUID String                     |
| Offsets 0x50 - 0x80            : Sub-Header Checksum (SHA256)           |
| Offset 0x1000                  : JSON Metadata Area (Keyslots, Segments)|
| Offset 0x8000                  : Binary Key Material Slots Area         |
| Offset Payload Start (e.g. 16M): Encrypted User Storage Data            |
+-------------------------------------------------------------------------+

```

##### JSON Metadata Area Structure:

```json
{
  "keyslots": {
    "0": {
      "type": "luks2",
      "kdf": {
        "type": "argon2id",
        "time": 4,
        "memory": 1048576,
        "cpus": 4,
        "salt": "base64_encoded_salt..."
      },
      "af": {
        "type": "stripes",
        "stripes": 4000
      },
      "area": {
        "type": "raw",
        "offset": "32768",
        "size": "262144"
      }
    }
  }
}

```

* **Anti-Forensic Splitter (AF-Splitter):** The master key payload is split across thousands of stripes (e.g., 4000 stripes) using XOR transformations before being written to disk. This ensures that if even a single sector in the keyslot region is damaged or corrupted, the entire master key becomes unrecoverable, preventing partial key recovery attacks.

#### 4. Line-by-Line Flag & Syntax Breakdown

* `cryptsetup`: Master management binary for LUKS storage targets.
* `luksOpen`: Command action to unlock an encrypted volume and instantiate a decrypted Device Mapper block target.
* `/dev/sdX1`: Source encrypted physical partition path.
* `secure_volume`: Virtual mapped device name created under `/dev/mapper/secure_volume`.

#### 5. Output Anatomy

```
Enter passphrase for /dev/sdX1: 

```

Upon entering the correct passphrase, the process executes silently ($0$ exit status).

Inspect active crypt mapping using `cryptsetup status secure_volume`:

```
/dev/mapper/secure_volume is active.
  type:    LUKS2
  cipher:  aes-xts-plain64
  keysize: 512 bits
  key location: keyring
  device:  /dev/sdX1
  sector size: 4096 bytes
  offset:  32768 sectors
  size:    20971520 sectors
  mode:    read/write

```

* `type: LUKS2`: Validates LUKS version 2 structure.
* `cipher: aes-xts-plain64`: Symmetric cipher configuration. Uses Advanced Encryption Standard (AES) in XTS block cipher mode, using 64-bit plain initialization vectors (IVs) derived from physical sector numbers.
* `keysize: 512 bits`: Key size parameter ($2 \times 256$-bit keys used for AES-XTS cipher operations).
* `key location: keyring`: Confirms the decrypted master key is stored securely in kernel memory using the Linux keyring API rather than user-space memory buffers.
* `offset: 32768 sectors`: Data payload offset ($32768 \times 512 = 16.77\text{ MB}$). This space is reserved at the start of the drive for the primary LUKS header, secondary backup header, JSON metadata, and binary keyslots.