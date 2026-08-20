## SECTION 1: Standard POSIX Permissions

---

### 1. `chmod 755 script.sh` / `chmod u+x script.sh`

#### Fundamental Purpose & Historical Evolution

In early Research Unix (v1–v6), file security relied on a monolithic 16-bit mode word in the on-disk inode structure. As Unix expanded into multi-user environments (UNIX System V and BSD), the need arose for a consistent method to grant discrete permissions across three user classes: the file's owner, the file's primary group, and all other authenticated system accounts.

The `chmod` utility and its underlying system calls (`chmod()`, `fchmod()`, `fchmodat()`) were introduced to manipulate these DAC (Discretionary Access Control) permission bits without altering file content or ownership.

Before fine-grained access control models existed, the security boundary relied entirely on standard POSIX mode bits (UGO: User, Group, Other). The primary design constraint addressed by `chmod` was preventing unauthorized execution, read exposure, or modification of administrative scripts and system binaries by non-privileged users.

```
       16-bit Inode st_mode Structure
+-----------------------------------------------+
| File Type | Special Bits | User  | Group | Other |
| (4 bits)  |   (3 bits)   | (rwx) | (rwx) | (rwx) |
+-----------------------------------------------+
  15  ..  12  11   ..   9   8 . 6   5 . 3   2 . 0

```

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ User Space ]
  chmod("script.sh", 0755)  OR  chmod("script.sh", "u+x")
            |
            v  (glibc wrapper)
[ System Call Interface ]
  sys_fchmodat(AT_FDCWD, "script.sh", 0755, 0)
            |
            v
[ VFS Path Resolution ]
  user_path_at() ---> filename_lookup()
            |
            +---> Resolves path to `struct path` (vfsmount + dentry)
            |
[ Kernel Authorization Check ]
  chmod_common()
            |
            +---> lookup_fast() / lookup_slow() fetches `struct inode`
            |
            +---> inode_owner_or_capable(&init_user_ns, inode)
                  |
                  +-- Checks: current_fsuid() == inode->i_uid
                  |           OR capable_wrt_inode_uid(inode, CAP_FOWNER)
                  |
                  +-- Returns 0 -> OK | -EPERM -> Denied
            |
[ Filesystem Inode Modification ]
  notify_change()
            |
            +---> iattr.ia_valid = ATTR_MODE
            +---> iattr.ia_mode  = (inode->i_mode & S_IFMT) | new_mode
            |
            +---> inode->i_op->setattr()  (e.g., ext4_setattr)
                  |
                  +-- Starts Journal Handle (jbd2)
                  +-- Updates raw disk inode (`st_mode`)
                  +-- Marks inode dirty: mark_inode_dirty()
                  +-- Commits journal entry

```

1. **System Call Handling**: `chmod 755 script.sh` invokes glibc's `chmod()`, which maps to `sys_fchmodat(AT_FDCWD, "script.sh", 0755, 0)`.
2. **Parsing Symbolic Strings vs. Octal Bitmasks**:
* **Octal (`0755`)**: Parsed directly into an integer bitmask (`S_IRWXU | S_IRGRP | S_IXGRP | S_IROTH | S_IXOTH`).
* **Symbolic (`u+x`)**: The command reads the existing mode via `stat()`, modifies the user bitmask by applying `existing_mode | S_IXUSR`, and submits the calculated mode to `fchmodat()`.


3. **VFS Path Resolution**: `sys_fchmodat()` invokes `user_path_at()` to resolve `"script.sh"` relative to the current working directory (`AT_FDCWD`), returning a pointer to the associated `struct dentry` and `struct inode`.
4. **Authorization Verification**: The kernel calls `chmod_common()`, which enforces DAC ownership rules via `inode_owner_or_capable()`:
```c
bool inode_owner_or_capable(struct user_namespace *mnt_userns, const struct inode *inode)
{
    if (uid_eq(current_fsuid(), inode->i_uid))
        return true;
    if (ns_capable(mnt_userns, CAP_FOWNER))
        return true;
    return false;
}

```


If `current_fsuid()` matches `inode->i_uid`, or if the process holds `CAP_FOWNER` in its user namespace, access is granted. Otherwise, the call aborts with `-EPERM`.
5. **Inode Modification & Metadata Persistence**:
* `chmod_common()` calls `notify_change()`, passing a `struct iattr` with `ATTR_MODE`.
* The filesystem-specific operation (e.g., `ext4_setattr()`) locks the inode in-memory (`inode_lock(inode)`).
* Preserves the file type bitmask (`inode->i_mode & S_IFMT`) while replacing permission bits with the target mode.
* Updates `inode->i_ctime` to current system time.
* Marks the inode dirty (`mark_inode_dirty()`) and commits the transaction via the journaling block device (`jbd2`).



#### Bitmask, Data Structure & Inode Field Dynamics

`st_mode` is a 16-bit integer field inside `struct inode` (and mirrored in `struct stat`).

```
Mode Bit Structure (16 bits):
TTTT SSS RWE RWE RWE
|||| ||| ||| ||| ||+-- Others Execute (00001)
|||| ||| ||| ||| +---- Others Write   (00002)
|||| ||| ||| ||+------ Others Read    (00004)
|||| ||| ||| |+------- Group Execute  (00010)
|||| ||| ||| +-------- Group Write    (00020)
|||| ||| ||+---------- Group Read     (00040)
|||| ||| |+----------- User Execute   (00100)
|||| ||| +------------ User Write     (00200)
|||| ||+-------------- User Read      (00400)
|||| |+--------------- Sticky Bit     (01000) (S_ISVTX)
|||| +---------------- SGID Bit       (02000) (S_ISGID)
|||+------------------ SUID Bit       (04000) (S_ISUID)
++++------------------ File Type Mask (0170000) (S_IFMT)

```

**Octal Bitmask Lookup Table**:

* `S_IFMT`   `0170000`: Bitmask for file type field
* `S_IFREG`  `0100000`: Regular file
* `S_IFDIR`  `0040000`: Directory
* `S_ISUID`  `0004000`: Set-user-ID bit
* `S_ISGID`  `0002000`: Set-group-ID bit
* `S_ISVTX`  `0001000`: Sticky bit
* `S_IRWXU`  `0000700`: RWX mask for Owner
* `S_IRUSR`  `0000400`: Owner Read
* `S_IWUSR`  `0000200`: Owner Write
* `S_IXUSR`  `0000100`: Owner Execute
* `S_IRWXG`  `0000070`: RWX mask for Group
* `S_IRGRP`  `0000040`: Group Read
* `S_IWGRP`  `0000020`: Group Write
* `S_IXGRP`  `0000010`: Group Execute
* `S_IRWXO`  `0000007`: RWX mask for Other
* `S_IROTH`  `0000004`: Other Read
* `S_IWOTH`  `0000002`: Other Write
* `S_IXOTH`  `0000001`: Other Execute

#### Line-by-Line Flag & Syntax Breakdown

* `chmod`: Executable binary calling `fchmodat()`.
* `755`: Absolute octal definition:
* `7` (`4+2+1`): `S_IRUSR | S_IWUSR | S_IXUSR` (Owner: rwx)
* `5` (`4+0+1`): `S_IRGRP | S_IXGRP` (Group: r-x)
* `5` (`4+0+1`): `S_IROTH | S_IXOTH` (Other: r-x)


* `u+x`: Relative mode expression:
* `u`: Directs operation solely to user (owner) bits (`S_IRWXU`).
* `+`: Applies bitwise OR (`mode |= S_IXUSR`).
* `x`: Selects the executable bit (`S_IXUSR`, `00100`).


* `script.sh`: Path passed to path resolution functions.

#### Exhaustive Output Anatomy

Command:

```bash
ls -l script.sh

```

Output:

```text
-rwxr-xr-x 1 Alice Security 4096 Aug 20 10:00 script.sh

```

* `-`: File type character (`S_IFREG`, regular file).
* `rwx`: Owner permissions (`S_IRUSR | S_IWUSR | S_IXUSR` -> `0700`). Read, Write, Execute enabled.
* `r-x`: Group permissions (`S_IRGRP | S_IXGRP` -> `0050`). Read, Execute enabled; Write disabled.
* `r-x`: Other permissions (`S_IROTH | S_IXOTH` -> `0005`). Read, Execute enabled; Write disabled.
* `1`: Link count (`st_nlink`).
* `Alice`: Owner username resolved from `st_uid`.
* `Security`: Group name resolved from `st_gid`.
* `4096`: File size in bytes (`st_size`).
* `Aug 20 10:00`: Timestamp of last modification (`st_mtime`).
* `script.sh`: Filename entry stored in parent directory dentry.

---

### 2. `chown -R user:group dir`

#### Fundamental Purpose & Historical Evolution

`chown` (Change Ownership) exists to re-assign file/directory ownership boundaries (`st_uid` and `st_gid` in the on-disk inode). In multi-tenant environments, redistributing access rights or offloading project maintenance requires shifting file owner attributes without rewriting file contents.

Historically, unrestricted `chown` allowed non-privileged users to "give away" files. This created security vulnerabilities: users could bypass quota limits by assigning large files to other accounts, or set up malicious SUID binaries on systems where non-root `chown` preserved privilege bits.

Modern Linux restricts `chown` operations to processes bearing `CAP_CHOWN` (typically restricted to `EUID == 0`). System adjustments also clear SUID/SGID bits during `chown` calls made by non-root users to eliminate privilege elevation vulnerabilities.

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ chown -R user:group dir ]
         |
         v
  Lookup user/group strings via NSS (getpwnam / getgrnam)
  Obtain numeric UID=1001, GID=1002
         |
         v
  sys_fchownat(AT_FDCWD, "dir", 1001, 1002, 0)
         |
         v
  VFS lookup -> dentry -> inode
         |
         v
  check_chown_permission()
    |-- Checks if current process has CAP_CHOWN in current user namespace.
    +-- If not root -> return -EPERM.
         |
         v
  chown_common()
    |-- inode_lock(inode)
    |-- Checks if SUID/SGID bits are set:
    |     If set AND not root -> mode &= ~(S_ISUID | S_ISGID)  [CAP_FSETID check]
    |-- inode->i_uid = make_kuid(current_user_ns(), 1001)
    |-- inode->i_gid = make_kgid(current_user_ns(), 1002)
    |-- inode->i_ctime = current_time(inode)
    |-- mark_inode_dirty(inode)
    +-- inode_unlock(inode)
         |
         v
  [ Traversal Engine: -R Flag ]
  openat(AT_FDCWD, "dir", O_RDONLY|O_CLOEXEC|O_DIRECTORY)
         |
         v
  Loop: sys_getdents64()
    +-- Reads linux_dirent64 entries
    +-- Recurses into subdirectories using openat(parent_fd, entry_name, ...)
    +-- Calls sys_fchownat() on each directory and file descriptor

```

1. **User/Group Name Resolution**: The user-space `chown` binary calls `getpwnam("user")` and `getgrnam("group")`. These query glibc's Name Service Switch (`nsswitch.conf`), parsing local files (`/etc/passwd`, `/etc/group`) or remote directory services (LDAP/SSSD) to return numeric target `UID` and `GID` values.
2. **Recursive Traversal Mechanism (`-R`)**:
* `chown` utilizes `fts` or `openat()` with `getdents64()` to walk the filesystem tree recursively.
* It maintains file descriptors safely across nested directories, avoiding path-max overflows and symlink race conditions (`TOCTOU`).


3. **Kernel System Call**: For every encountered inode, the kernel invokes `sys_fchownat(dfd, filename, uid, gid, flags)`.
4. **Kernel Capability Check**: Inside `chown_common()`, the kernel runs `check_chown_permission()`:
```c
static int chown_common(const struct path *path, uid_t user, gid_t group)
{
    struct inode *inode = path->dentry->d_inode;
    // ...
    if (!capable_wrt_inode_uid(inode, CAP_CHOWN))
        return -EPERM;
    // ...
}

```


Unless the caller holds `CAP_CHOWN` inside the active user namespace, the kernel rejects the change with `-EPERM`.
5. **Clear-on-Chown Security Mechanism**:
* If an inode has SUID (`S_ISUID`) or SGID (`S_ISGID`) bits configured, changing the user or group owner clears these bits automatically unless the executing process possesses `CAP_FSETID`.
* This prevents a user from creating a malicious SUID binary and changing its owner to another account to execute commands under that targeted context.



#### Bitmask, Data Structure & Inode Field Dynamics

In-memory `struct inode` structures track ownership using explicit kernel structures:

```c
struct inode {
    kuid_t i_uid; /* Numeric Owner UID */
    kgid_t i_gid; /* Numeric Owner GID */
    umode_t i_mode; /* Type and permissions bitmask */
    /* ... */
};

```

On disk (e.g., ext4 file systems), these map to numeric 32-bit fields:

* `i_uid_low` / `i_uid_high`: 32-bit combined unique user identifier.
* `i_gid_low` / `i_gid_high`: 32-bit combined unique group identifier.

When `chown` alters ownership, `st_mode` undergoes automatic bit-clearing if `S_ISUID` or `S_ISGID` are set:


$$\text{new\_mode} = \text{st\_mode} \ \& \ \sim(\text{S\_ISUID} \ \vert{} \ \text{S\_ISGID})$$

#### Line-by-Line Flag & Syntax Breakdown

* `chown`: System administration binary calling `fchownat()`.
* `-R`: Recursive flag. Triggers directory iteration via `getdents64()` to apply modifications across subdirectories and contained files.
* `user`: Target username string (resolved to numeric `UID`).
* `:`: Delimiter separating target user owner from group owner.
* `group`: Target group name string (resolved to numeric `GID`).
* `dir`: Top-level path provided for tree traversal.

#### Exhaustive Output Anatomy

Before execution:

```bash
ls -ld dir ; ls -l dir/script.sh

```

```text
drwxr-xr-x 2 root root 4096 Aug 20 10:00 dir
-rwsr-xr-x 1 root root  256 Aug 20 10:00 dir/script.sh

```

Command:

```bash
chown -R Alice:Security dir

```

After execution (viewed as non-root user execution):

```text
drwxr-xr-x 2 Alice Security 4096 Aug 20 10:00 dir
-rwxr-xr-x 1 Alice Security  256 Aug 20 10:00 dir/script.sh

```

* `drwxr-xr-x`: Directory indicator (`d`) and mode mask (`0755`).
* `Alice`: Directory owner updated from `root` to `Alice` (`st_uid = 1001`).
* `Security`: Group ownership updated from `root` to `Security` (`st_gid = 1002`).
* `-rwxr-xr-x`: Notice that `script.sh` lost its SUID bit (`s` reverted to `x`) because non-root execution cleared `S_ISUID` (`04000`) via `CAP_FSETID` sanitization rules.

---

### 3. `umask 022`

#### Fundamental Purpose & Historical Evolution

The user file-creation mode mask (`umask`) was introduced to enforce systemic, baseline access limits for newly created files and directories. Rather than requiring applications to explicitly redact access rights after file creation (creating race conditions during creation phases), `umask` operates as an environment constraint inherited across process states.

When processes call creation primitives (`open(O_CREAT)`, `mkdir()`, `mknod()`), they pass a requested permission mode (typically `0666` for regular files and `0777` for directories). The kernel applies the calling process's active `umask` mask to mask off specified permissions before creating the file on disk. This design ensures that sensitive files are never exposed, even if an application requests overly permissive defaults.

```
       Default Base Mode           Process umask
     (Files: 0666, Dirs: 0777)        (0022)
            |                            |
            +-------------[ AND NOT ]----+
                            |
                            v
                   Effective File Mode

```

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ Shell / Process ]
  umask(0022)
      |
      v
[ System Call ]
  sys_umask(mode)
      |
      +---> current->fs->umask = mode & 0777
      +---> Returns old umask value
      
[ File Creation Flow: open("file.txt", O_CREAT|O_WRONLY, 0666) ]
  sys_openat()
      |
      v
  do_filp_open() -> path_openat() -> lookup_open()
      |
      v
  atomic_open() -> vfs_create()
      |
      v
  mode = mode & ~current->fs->umask
      |
      +---> Requested: 0666 (110 110 110)
      +---> umask:     0022 (000 010 010)
      +---> ~umask:    0755 (111 101 101)
      +---> Effective: 0644 (110 100 100)
      |
      v
  inode_operations->create(dir_inode, dentry, effective_mode, true)

```

1. **System Call Implementation**: Calling `umask(022)` executes the `sys_umask()` kernel function:
```c
SYSCALL_DEFINE1(umask, int, mask)
{
    int old = current->fs->umask;
    current->fs->umask = mask & S_IRWXUGO;
    return old;
}

```


2. **Process Storage Scope**: The integer mask value is saved directly inside `current->fs->umask`, hosted in the process's `struct fs_struct`.
3. **Inheritance Model**:
* `fork()`: The child process inherits a copy of the parent's `fs_struct` (or references it if `CLONE_FS` is set).
* `execve()`: The `umask` setting survives process image replacements, guaranteeing that spawned execution binaries preserve security defaults.


4. **VFS Integration During Creation**:
* When `open(path, O_CREAT, 0666)` or `mkdir(path, 0777)` is called, processing passes through `path_openat()` to `lookup_open()`.
* The kernel applies bitwise logic to calculate the final inode mode mask before invoking filesystem-level write handlers:

$$\text{mode} = \text{requested\_mode} \ \& \ \sim\text{current->fs->umask}$$





#### Bitmask, Data Structure & Inode Field Dynamics

Because execution flags (`+x`) carry security risks on standard files, typical user space applications request base permissions of `0666` (`rw-rw-rw-`) when creating regular files, and `0777` (`rwxrwxrwx`) when creating directories.

**File Creation Dynamics (`mode = 0666 & ~0022`)**:

$$\begin{array}{rcccl} \text{Requested Mode (0666):} & 110 & 110 & 110 & \text{(rw-rw-rw-)} \\ \text{umask Value (0022):} & 000 & 010 & 010 & \text{(----w--w-)} \\ \text{Bitwise NOT umask (\textasciitilde 0022):} & 111 & 101 & 101 & \text{(rwxr-xr-x)} \\ \hline \text{Final Inode Mode (0644):} & 110 & 100 & 100 & \text{(rw-r--r--)} \end{array}$$

**Directory Creation Dynamics (`mode = 0777 & ~0022`)**:

$$\begin{array}{rcccl} \text{Requested Mode (0777):} & 111 & 111 & 111 & \text{(rwxrwxrwx)} \\ \text{umask Value (0022):} & 000 & 010 & 010 & \text{(----w--w-)} \\ \text{Bitwise NOT umask (\textasciitilde 0022):} & 111 & 101 & 101 & \text{(rwxr-xr-x)} \\ \hline \text{Final Inode Mode (0755):} & 111 & 101 & 101 & \text{(rwxr-xr-x)} \end{array}$$

#### Line-by-Line Flag & Syntax Breakdown

* `umask`: Shell built-in command wrapping the `sys_umask` system call interface.
* `022`: Octal permission bitmask definition:
* `0` (Owner): Clears no bit modifications for file owner permissions.
* `2` (Group): Strips write bit (`S_IWGRP`, bit `00020`) from created resources.
* `2` (Other): Strips write bit (`S_IWOTH`, bit `00002`) from created resources.



#### Exhaustive Output Anatomy

Command:

```bash
umask 022
touch standard_file.txt
mkdir standard_dir
ls -ld standard_file.txt standard_dir

```

Output:

```text
drwxr-xr-x 2 Alice Security 4096 Aug 20 10:00 standard_dir
-rw-r--r-- 1 Alice Security    0 Aug 20 10:00 standard_file.txt

```

* `drwxr-xr-x`: Directory permissions computed as `0777 & ~0022 = 0755`. Execute bits are preserved so users can traverse the directory.
* `-rw-r--r--`: Regular file permissions computed as `0666 & ~0022 = 0644`. Group and Other write bits are masked off.

---

## SECTION 2: Advanced Security Controls

---

### 1. `chmod +s /usr/bin/binary`

#### Fundamental Purpose & Historical Evolution

The Set-User-ID (SUID) and Set-Group-ID (SGID) permission mechanics were invented by Dennis Ritchie in 1972 to solve a fundamental access control challenge: non-privileged users occasionally need to modify privileged system files without gaining full administrative access.

For example, to change their account password, a standard user must update `/etc/shadow`—a file restricted to `root`. Placing the SUID bit on the `/usr/bin/passwd` binary instructs the kernel to execute the process using the target file owner's privileges (e.g., `root`), rather than the privileges of the calling user.

```
+-------------------------------------------------------------------+
|                       Process Credentials                         |
|  RUID: 1001 (User)   EUID: 0 (Root via SUID)   SUID: 0 (Saved)    |
+-------------------------------------------------------------------+
                                  ^
                                  |
               Executes file with SUID Bit (04000)
                                  |
+-------------------------------------------------------------------+
|                        On-Disk Inode                              |
|  Path: /usr/bin/passwd   Owner: root (0)   Mode: -rwsr-xr-x       |
+-------------------------------------------------------------------+

```

Because SUID binaries grant elevated privilege, they present significant attack surface if improperly secured. Vulnerabilities such as buffer overflows, environment variable injection (`LD_PRELOAD`), or race conditions in SUID executables can lead to full system compromise. Modern Linux mitigates these vectors through mechanisms like `securebits`, `/proc/sys/fs/suid_dumpable`, and the `NO_NEW_PRIVS` flag.

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ User Application ]
  execve("/usr/bin/binary", argv, envp)
            |
            v
[ System Call Handler: sys_execve ]
  do_execve() -> do_execveat_common() -> bprm_execve()
            |
            v
[ Load Binary Image ]
  exec_binprm() -> search_binary_handler() -> load_elf_binary()
            |
            v
[ Kernel Privilege Assessment ]
  begin_new_exec()
            |
            +---> prepare_binprm()
                  |
                  +-- Checks inode mode for S_ISUID / S_ISGID
                  +-- Checks mount point flags (e.g., MS_NOSUID / MNT_NOSUID)
                  +-- Checks task flags: current->no_new_privs
                  |
            +---> setup_new_exec()
                  |
                  +-- If S_ISUID is present AND allowed:
                      new_cred->euid = inode->i_uid
                      new_cred->fsuid = new_cred->euid
                  +-- If S_ISGID is present AND allowed:
                      new_cred->egid = inode->i_gid
                      new_cred->fsgid = new_cred->egid
                  |
            +---> commit_creds(new_cred)
                  |
                  +-- Replaces running process credential block

```

1. **System Call Pipeline**: Calling `execve()` initiates image replacement via `do_execveat_common()`.
2. **Binary Preparation (`prepare_binprm()`)**: The kernel inspects the target file's inode attributes. If `S_ISUID` (`04000`) is active, special privilege transformation rules apply.
3. **Privilege Elevation Restrictions**: SUID elevation is blocked under any of these conditions:
* The underlying filesystem partition is mounted with the `nosuid` (`MNT_NOSUID`) flag.
* The calling thread has `task_struct->no_new_privs` set to `1` (commonly applied via `prctl(PR_SET_NO_NEW_PRIVS)`).
* The executing process is being traced via `ptrace` without appropriate administrative capabilities.


4. **Credential Modification**: If elevation is validated, `setup_new_exec()` updates the process credential structure (`struct cred`):
```c
if ((bprm->file->f_path.mnt->mnt_flags & MNT_NOSUID) == 0) {
    if (mode & S_ISUID)
        new_cred->euid = inode->i_uid;
    if (mode & S_ISGID)
        new_cred->egid = inode->i_gid;
}

```


The process's Real UID (`RUID`) remains that of the invoking user, while its Effective UID (`EUID`) and Saved Set-UID (`SUID`) are set to the file owner's UID.
5. **Core Dump Safeguards**: If a process raises its privileges via SUID, the kernel marks it non-dumpable (`/proc/sys/fs/suid_dumpable = 0`). This prevents unprivileged users from generating core dumps of privileged processes to extract sensitive kernel memory or data structures.

#### Bitmask, Data Structure & Inode Field Dynamics

The SUID and SGID properties occupy the high order bits of the permission structure within `st_mode`:

```
Bit Representation:
S_ISUID = 0004000 (Bit 11: Set-User-ID)
S_ISGID = 0002000 (Bit 10: Set-Group-ID)
S_ISVTX = 0001000 (Bit 9:  Sticky Bit)

```

**Interaction Matrix with Execution Bits**:

* SUID Enabled + User Executable (`S_ISUID | S_IXUSR`): Represented as lower-case `s` in output strings (`-rwsr-xr-x`).
* SUID Enabled + User Execution Disabled (`S_ISUID` without `S_IXUSR`): Represented as upper-case `S` in output strings (`-rwSr-xr-x`), indicating a configuration anomaly where SUID is set without execution rights.

**Process Credential Structure (`struct cred`) State Transitions**:

```
Before Executing SUID Binary (Owner: root):
+----------------------------------------------------------------+
| RUID: 1001  | EUID: 1001  | SUID: 1001  | FSUID: 1001          |
+----------------------------------------------------------------+

After Executing SUID Binary:
+----------------------------------------------------------------+
| RUID: 1001  | EUID: 0     | SUID: 0     | FSUID: 0             |
+----------------------------------------------------------------+

```

#### Line-by-Line Flag & Syntax Breakdown

* `chmod`: Command modification utility binary.
* `+s`: Symbolic modifier flag. Applies bitwise OR operations to assign both SUID (`04000`) and SGID (`02000`) bits based on selected targeted classes (defaulting to both `u` and `g` when no target class is explicitly defined).
* `/usr/bin/binary`: Target executable file path.

#### Exhaustive Output Anatomy

Command:

```bash
ls -l /usr/bin/passwd

```

Output:

```text
-rwsr-xr-x 1 root root 68248 Aug 20 10:00 /usr/bin/passwd

```

* `-`: File type (`S_IFREG`, regular file).
* `rws`: User permission set. The `s` character replaces `x`, indicating both `S_IXUSR` (`00100`) and `S_ISUID` (`04000`) are active.
* `r-x`: Group permissions (`00050`). Read and execute enabled.
* `r-x`: Other permissions (`00005`). Read and execute enabled.

---

### 2. `chmod +t /tmp`

#### Fundamental Purpose & Historical Evolution

The Sticky Bit (`S_ISVTX`) was introduced in early Unix versions to instruct the kernel to retain an executable's text segment in swap memory after process termination (hence "sticky"), speeding up subsequent executions of frequently used programs.

In modern Linux, memory paging systems have rendered this original swap-retention behavior obsolete. Instead, modern systems repurpose the sticky bit on directories to secure shared multi-user write environments like `/tmp` and `/var/tmp`.

```
Without Sticky Bit on /tmp (Mode 0777):
User Bob can delete /tmp/alice_file even if Bob does NOT own alice_file!
(Because Bob has write permissions on the parent directory /tmp)

With Sticky Bit on /tmp (Mode 01777):
Kernel intercepts unlink() -> may_delete() check:
User Bob CANNOT delete /tmp/alice_file unless:
  1. Bob owns alice_file (UID == fsuid), OR
  2. Bob owns /tmp directory, OR
  3. Bob possesses CAP_FOWNER capability.

```

Without the sticky bit, any user with write access to a directory can delete or rename files within it, regardless of who owns those files. In shared directories like `/tmp`, this allows malicious users to perform deletion attacks, replace target sockets, or execute symlink exploitation attacks. Setting the sticky bit restricts file deletion and renaming so that only the file owner, directory owner, or privileged account (`CAP_FOWNER`) can remove or rename contained resources.

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ User Application ]
  unlink("/tmp/target_file") OR rename("/tmp/file1", "/tmp/file2")
            |
            v
[ System Call Handler: sys_unlinkat ]
  do_unlinkat()
            |
            v
[ VFS Path Traversal ]
  filename_lookup() -> obtains parent dentry/inode AND target dentry/inode
            |
            v
[ Permission Verification: may_delete() ]
  vfs_unlink() -> may_delete(dir_inode, victim_dentry, is_dir)
            |
            +---> Checks standard DAC write access to directory
            |
            +---> Checks sticky bit condition:
                  if (dir_inode->i_mode & S_ISVTX) {
                      check: current_fsuid() == victim_inode->i_uid
                          OR current_fsuid() == dir_inode->i_uid
                          OR capable_wrt_inode_uid(victim_inode, CAP_FOWNER)
                  }
            |
            +---> If checks match -> Permit deletion/rename
            +---> If checks fail  -> Return -EPERM

```

1. **Deletion Interception Path**: When a process attempts to delete or rename a file using `unlinkat()` or `renameat()`, the kernel routes the request to `may_delete()` inside the VFS layer (`fs/namei.c`).
2. **Sticky Bit Validation Routine**:
```c
static int check_sticky(struct user_namespace *mnt_userns,
                        struct inode *dir, struct inode *inode)
{
    kuid_t fsuid = current_fsuid();
    if (uid_eq(fsuid, inode->i_uid))
        return 0;
    if (uid_eq(fsuid, dir->i_uid))
        return 0;
    if (capable_wrt_inode_uid(inode, CAP_FOWNER))
        return 0;
    return -EPERM;
}

```


3. **Hardlink and FIFO Security Extensions**: Modern Linux kernels build upon sticky bit protections through sysctl settings managed by the VFS layer:
* `/proc/sys/fs/protected_symlinks`: Restricts symlink traversal in world-writable, sticky directories to prevent symlink spoofing attacks.
* `/proc/sys/fs/protected_hardlinks`: Prevents users from creating hard links to files they do not own, blocking unauthorized link creation to sensitive resources.
* `/proc/sys/fs/protected_fifos` / `protected_regular`: Restricts how named pipes and regular files can be opened in sticky world-writable directories, mitigating temporary file hijacking vectors.



#### Bitmask, Data Structure & Inode Field Dynamics

The sticky bit occupies bit 9 of the `st_mode` mask:

```
S_ISVTX = 0001000 (Bit 9: Sticky Bit / Restricted Deletion Flag)

```

**Bitmask Calculations**:

* Base Directory Permissions (`0777`): `S_IRWXU | S_IRWXG | S_IRWXO`
* Sticky Bit (`01000`): `S_ISVTX`
* Combined Bitmask (`01777`): `S_ISVTX | S_IRWXU | S_IRWXG | S_IRWXO`

**String Representation Rules**:

* Sticky Bit Enabled + Other Execute (`S_ISVTX | S_IXOTH`): Displayed as a lower-case `t` in the permissions string (`drwxrwxrwt`).
* Sticky Bit Enabled + Other Execute Disabled (`S_ISVTX` without `S_IXOTH`): Displayed as an upper-case `T` (`drwxrwxrwT`), indicating sticky bit activation on a directory that lacks global execution access.

#### Line-by-Line Flag & Syntax Breakdown

* `chmod`: Control utility adjusting file/directory mode masks.
* `+t`: Symbolic flag targeting bit assignment for `S_ISVTX` (`01000`), leaving standard UGO execution bits unchanged.
* `/tmp`: Target directory path.

#### Exhaustive Output Anatomy

Command:

```bash
ls -ld /tmp

```

Output:

```text
drwxrwxrwt 15 root root 4096 Aug 20 10:00 /tmp

```

* `d`: File type (`S_IFDIR`, directory).
* `rwx`: Owner permissions (`0700`). Read, write, and directory traversal enabled.
* `rwx`: Group permissions (`0070`). Read, write, and directory traversal enabled.
* `rwt`: Other permissions. The trailing `t` indicates that world execution (`S_IXOTH`, `00001`) and the sticky bit (`S_ISVTX`, `01000`) are both active.

---

### 3. `getfacl file` / `setfacl -m u:username:rwx file`

#### Fundamental Purpose & Historical Evolution

Traditional POSIX permissions use a simple UGO model (User, Group, Other), which limits fine-grained access control. For example, standard permissions cannot grant read access to a specific secondary user without either making that user the file's primary owner or adding them to the file's primary group.

```
POSIX DAC Limitations (UGO Only):
  Owner: Alice (rw-)
  Group: Developers (r--)
  Other: (---)
  Problem: How to give Bob *only* (rw-) without changing Owner or Group?

POSIX.1e ACL Extended Attribute Architecture:
  Owner: Alice (rw-)
  ACL User (Bob): (rw-) <--- Explicitly added via extended attributes!
  Group: Developers (r--)
  ACL Mask: (rw-)
  Other: (---)

```

To address these limitations, the POSIX.1e working group drafted a fine-grained Access Control List (ACL) specification. Although the draft standard was never formally ratified, its model was adopted by Solaris, FreeBSD, and Linux. Linux introduced POSIX ACL support in the 2.6 kernel line using Extended Attributes (`xattrs`), storing ACL rules in dedicated `system.*` namespaces (`system.posix_acl_access` and `system.posix_acl_default`).

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ User Application: setfacl -m u:bob:rwx file ]
                       |
                       v
  Generates binary struct posix_acl structure payload
                       |
                       v
[ System Call: setxattr() ]
  sys_fsetxattr(fd, "system.posix_acl_access", value, size, flags)
                       |
                       v
[ VFS xattr Layer ]
  vfs_setxattr() -> xattr_permission()
                       |
                       +-- Checks write access & CAP_FOWNER capability
                       |
[ Filesystem Driver (e.g., ext4) ]
  ext4_xattr_set() -> ext4_set_acl() -> posix_acl_from_xattr()
                       |
                       v
[ Inode Struct Updates ]
  Validates ACL structures
  Recalculates ACL_MASK bitmask
  Parses entries into internal `struct posix_acl`
  Stores payload in disk xattr block
                       |
                       v
[ Inode Mode Bit Synchronization ]
  Syncs group mode bits (st_mode) with calculated ACL_MASK

```

1. **Extended Attribute Namespace Mapping**: POSIX ACLs are backed by VFS Extended Attributes (`xattrs`):
* Access ACLs (`system.posix_acl_access`): Enforce active file/directory access policies.
* Default ACLs (`system.posix_acl_default`): Define inherited access rules applied automatically to newly created subdirectories and files.


2. **Kernel System Call Routing**: `getfacl` and `setfacl` interact with extended attributes using the `getxattr()` and `setxattr()` system calls:
```c
sys_getxattr(path, "system.posix_acl_access", value, size);
sys_setxattr(path, "system.posix_acl_access", value, size, flags);

```


3. **The ACL Evaluation Engine (`posix_acl_permission`)**: When a process attempts to access a file, the kernel evaluates permissions in strict hierarchical order:

```
[ Access Evaluation Request ]
              |
              v
     Is Process UID == File Owner UID?
      /                             \
    (Yes)                           (No)
    /                                 \
Evaluate ACL_USER_OBJ          Search Specific ACL_USER Entries
(Base Owner Permissions)        Matching Process UID
                                     /                   \
                              (Match Found)         (No Match Found)
                                   /                       \
                         Apply Mask Filtering        Search Process Groups in
                         (Entry & ACL_MASK)         ACL_GROUP_OBJ & ACL_GROUP
                                                            /              \
                                                     (Match Found)    (No Match Found)
                                                          /                  \
                                                Apply Mask Filtering       Evaluate ACL_OTHER
                                                (Entry & ACL_MASK)         (Base World Perms)

```

4. **Dynamic Mask Recalculation**: Adding named user or named group entries generates an `ACL_MASK` entry. The `ACL_MASK` sets the maximum permissions allowable for all named users, the group owner, and named groups. If an entry grants `rwx` permissions but the `ACL_MASK` is set to `r--`, the effective permission resolves to `r--`.

#### Bitmask, Data Structure & Inode Field Dynamics

In memory, POSIX ACL entries are represented as array structures containing entry tags, permission bitmasks, and qualification IDs:

```c
struct posix_acl_entry {
    short e_tag;   /* ACL_USER_OBJ, ACL_USER, ACL_GROUP_OBJ, ACL_GROUP, ACL_MASK, ACL_OTHER */
    unsigned short e_perm; /* Read (04), Write (02), Execute (01) */
    kuid_t e_uid;  /* KUID for ACL_USER entry types */
    kgid_t e_gid;  /* KGID for ACL_GROUP entry types */
};

struct posix_acl {
    refcount_t a_refcount;
    unsigned int a_count;
    struct posix_acl_entry a_entries[];
};

```

**On-Disk Extended Attribute Header Layout (`system.posix_acl_access`)**:

```
+-------------------------------------------------------------------+
| Version Header (4 bytes): 0x00020000                              |
+-------------------------------------------------------------------+
| Header Entry 1: e_tag (2b) | e_perm (2b) | e_id (4b)               |
+-------------------------------------------------------------------+
| Header Entry 2: e_tag (2b) | e_perm (2b) | e_id (4b)               |
+-------------------------------------------------------------------+
| ... Additional ACL Entries ...                                    |
+-------------------------------------------------------------------+

```

**POSIX ACL Tag Identifiers**:

* `ACL_USER_OBJ`   (`0x01`): Corresponds to traditional file owner permissions.
* `ACL_USER`       (`0x02`): Specifies permissions for an explicit user UID.
* `ACL_GROUP_OBJ`  (`0x04`): Corresponds to traditional group permissions.
* `ACL_GROUP`      (`0x08`): Specifies permissions for an explicit group GID.
* `ACL_MASK`       (`0x10`): Sets the maximum allowed access for named users, group owner, and named groups.
* `ACL_OTHER`      (`0x20`): Corresponds to traditional world/other permissions.

#### Line-by-Line Flag & Syntax Breakdown

* `getfacl`: Utility command that queries and displays file ACL configurations via `getxattr()`.
* `setfacl`: Administrative utility that modifies file ACL configurations via `setxattr()`.
* `-m`: Modify option flag. Updates or adds one or more ACL rules.
* `u:`: Target entry tag type (`ACL_USER`).
* `username`: Target account identifier, resolved to numeric UID.
* `:rwx`: Bitmask sequence defining explicit permissions: Read (`04`), Write (`02`), and Execute (`01`).

#### Exhaustive Output Anatomy

Command:

```bash
setfacl -m u:Bob:rwx file.txt
getfacl file.txt

```

Output:

```text
# file: file.txt
# owner: Alice
# group: Security
user::rw-
user:Bob:rwx
group::r--
mask::rwx
other::r--

```

* `# file: file.txt`: Filename parsed from the input path.
* `# owner: Alice`: File owner resolved from `st_uid`.
* `# group: Security`: Group owner resolved from `st_gid`.
* `user::rw-`: `ACL_USER_OBJ` entry representing the file owner's permissions (`Alice`).
* `user:Bob:rwx`: Extended `ACL_USER` entry granting explicit permissions to user `Bob` (`UID 1002`).
* `group::r--`: `ACL_GROUP_OBJ` entry showing file group permissions (`Security`).
* `mask::rwx`: `ACL_MASK` entry defining the permission cap applied to named users, group owner, and named groups.
* `other::r--`: `ACL_OTHER` entry showing traditional permissions for unlisted users.

---

### 4. `chattr +i file`

#### Fundamental Purpose & Historical Evolution

Traditional POSIX DAC attributes (mode bits and ownership) govern access based on user identity. However, these checks are bypassed by privileged processes (`EUID == 0` or processes with `CAP_DAC_OVERRIDE`). As a result, standard permissions cannot protect critical configuration files from accidental modification or malicious deletion by an elevated account.

To address this limitation, Extended Filesystem attributes (`i_flags`) were introduced, starting with ext2 and now supported across ext4, XFS, and Btrfs. These attributes include flags like Immutability (`+i`) and Append-Only (`+a`).

When the immutable attribute is applied, the kernel enforces write restrictions at the VFS layer, blocking modifications, unlinking, renaming, and link creation regardless of process permissions—even for `root`.

```
                  VFS Open Request (O_RDWR or O_TRUNC)
                                   |
                                   v
                   Is EXT4_IMMUTABLE_FL Flag Set?
                      /                         \
                    (Yes)                       (No)
                    /                             \
         Block Operation                Perform Standard DAC/MAC Checks
         Return -EPERM                  (Owner, Mode, Capabilities)
     (Overrides EUID == 0!)

```

#### Under-the-Under Execution & Kernel Mechanisms

```
[ User Space ]
  chattr +i /etc/shadow
       |
       v
  fd = open("/etc/shadow", O_RDONLY)
  ioctl(fd, FS_IOC_GETFLAGS, &flags)
  flags |= FS_IMMUTABLE_FL
  ioctl(fd, FS_IOC_SETFLAGS, &flags)
       |
       v
[ VFS Inode Driver: ext4_ioctl ]
  ext4_ioctl() -> ext4_ioctl_setflags()
       |
       +---> Check administrative privilege:
       |     capable(CAP_LINUX_IMMUTABLE)
       |     If missing -> return -EPERM
       |
       +---> Lock Inode: inode_lock(inode)
       +---> Update Inode Flags:
       |     ei->i_flags |= EXT4_IMMUTABLE_FL
       +---> Synchronize Inode Attributes:
       |     ext4_set_inode_flags(inode)
       |     inode->i_flags |= S_IMMUTABLE
       +---> Write Changes: ext4_mark_inode_dirty()
       +---> Unlock Inode: inode_unlock(inode)

```

1. **System Call Interface**: `chattr` opens a file descriptor to the target path and issues an `ioctl()` call with the `FS_IOC_SETFLAGS` command.
2. **Capability Verification**: Modifying filesystem attribute flags requires administrative capabilities. The kernel checks whether the calling process holds `CAP_LINUX_IMMUTABLE`:
```c
if (!capable(CAP_LINUX_IMMUTABLE))
    return -EPERM;

```


3. **Kernel Inode Flag Synchronization**:
* The filesystem driver updates its internal flags (e.g., `EXT4_IMMUTABLE_FL`).
* It then syncs these changes to the generic VFS inode representation via `ext4_set_inode_flags()`, applying `S_IMMUTABLE` to `inode->i_flags`.


4. **VFS Enforcement Layer**: Once `S_IMMUTABLE` is set on an inode, VFS operations enforce write restrictions across all system calls:
* File Open (`open(O_WRONLY)` / `open(O_TRUNC)`): `may_open()` checks for `S_IMMUTABLE` and aborts with `-EPERM`.
* Unlink (`unlink()`): `may_delete()` checks `IS_IMMUTABLE(inode)` and rejects file removal attempts.
* Rename (`rename()`): `vfs_rename()` checks both source and target inodes, blocking rename operations if either has `S_IMMUTABLE` set.
* Link Creation (`link()`): `vfs_link()` rejects hard link creation for immutable files.
* Timestamp updates are suspended, preventing modifications to `i_atime` and `i_mtime`.



#### Bitmask, Data Structure & Inode Field Dynamics

Extended attribute flags are stored within filesystem-specific inode structures:

```c
struct ext4_inode {
    __le16 i_mode;        /* File mode mask */
    __le16 i_uid;         /* Owner UID */
    __le32 i_size_lo;     /* File size */
    __le32 i_atime;       /* Access time */
    __le32 i_ctime;       /* Creation/Change time */
    __le32 i_mtime;       /* Modification time */
    __le32 i_dtime;       /* Deletion time */
    __le16 i_gid;         /* Owner GID */
    /* ... */
    __le32 i_flags;       /* Filesystem Attribute Flags Mask */
    /* ... */
};

```

**Common Ext4 Inode Flag Bitmasks**:

* `EXT4_SECRM_FL`        `0x00000001`: Secure deletion (zero block overwrite).
* `EXT4_UNRM_FL`         `0x00000002`: Undelete support reserved bit.
* `EXT4_COMPR_FL`        `0x00000004`: File compression active.
* `EXT4_SYNC_FL`         `0x00000008`: Synchronous updates (`O_SYNC` enforcement).
* `EXT4_IMMUTABLE_FL`    `0x00000010`: Immutable file flag (`+i`).
* `EXT4_APPEND_FL`       `0x00000020`: Append-only file flag (`+a`).
* `EXT4_NODUMP_FL`       `0x00000040`: Omit from dump utility backups.
* `EXT4_NOATIME_FL`      `0x00000080`: Disable access time (`atime`) updates.

#### Line-by-Line Flag & Syntax Breakdown

* `chattr`: Utility binary that sets extended attributes via `ioctl()`.
* `+i`: Applies the immutable flag (`EXT4_IMMUTABLE_FL`, `0x00000010`), setting the `S_IMMUTABLE` flag in VFS `inode->i_flags`.
* `file`: Target file path.

#### Exhaustive Output Anatomy

Command:

```bash
chattr +i secure_config.cfg
rm -f secure_config.cfg

```

Output:

```text
rm: cannot remove 'secure_config.cfg': Operation not permitted

```

Even when executed by `root` (`EUID == 0`), the deletion attempt fails:

```text
[ VFS: may_delete() Check ]
IS_IMMUTABLE(inode) == True -> Access Denied (-EPERM)

```

---

### 5. `lsattr file`

#### Fundamental Purpose & Historical Evolution

Because low-level extended attributes (`i_flags`) are stored outside standard POSIX permission bitmasks (`st_mode`), standard file listing utilities like `ls` do not display them. The `lsattr` command was created alongside `chattr` to query and display these filesystem-level attribute flags.

`lsattr` reads extended flags directly from the filesystem by invoking the `FS_IOC_GETFLAGS` ioctl interface. It then converts the returned 32-bit bitmask into a human-readable text string, providing visibility into attributes like immutability, append-only status, and synchronous I/O settings.

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ User Application: lsattr file ]
              |
              v
    open("file", O_RDONLY | O_NONBLOCK)
              |
              v
    ioctl(fd, FS_IOC_GETFLAGS, &flags)
              |
              v
[ VFS Handler: ext4_ioctl ]
    ext4_ioctl(..., FS_IOC_GETFLAGS, ...)
              |
              +---> Accesses internal inode: EXT4_I(inode)->i_flags
              +---> Copies 32-bit flags value to user space memory
              |
[ User Space Formatting ]
    Bitwise evaluation of flag bitmask
    Translates active bits to character representations
    Outputs formatted attribute string

```

1. **Path Opening**: `lsattr` opens the file descriptor using non-blocking read-only mode (`O_RDONLY | O_NONBLOCK`), allowing attribute queries even on special files or FIFO pipes.
2. **Ioctl Execution**: It issues the `FS_IOC_GETFLAGS` ioctl call to request the raw 32-bit attribute mask from the underlying filesystem:
```c
unsigned long flags;
ioctl(fd, FS_IOC_GETFLAGS, &flags);

```


3. **Kernel Retrieval**: The filesystem driver (e.g., `ext4_ioctl`) processes the request, reads `i_flags` from the target inode, and returns the 32-bit mask to user space.
4. **Character String Conversion**: `lsattr` processes the returned 32-bit mask using bitwise AND operations (`flags & MASK`), translating active flags into a fixed-width character representation.

#### Bitmask, Data Structure & Inode Field Dynamics

`lsattr` decodes individual flag bits from the 32-bit integer mask returned by `FS_IOC_GETFLAGS`:

```
Bitmask Position Lookup Table:
Bit 0  (0x00000001) -> 's' (Secure deletion)
Bit 1  (0x00000002) -> 'u' (Undelete)
Bit 2  (0x00000004) -> 'c' (Compressed)
Bit 3  (0x00000008) -> 'S' (Synchronous updates)
Bit 4  (0x00000010) -> 'i' (Immutable)
Bit 5  (0x00000020) -> 'a' (Append-only)
Bit 6  (0x00000040) -> 'd' (No dump)
Bit 7  (0x00000080) -> 'A' (No atime updates)
Bit 11 (0x00000800) -> 'E' (Encrypted)
Bit 12 (0x00001000) -> 'j' (Data journalling)
Bit 19 (0x00080000) -> 'e' (Extent format)
Bit 21 (0x00200000) -> 'V' (Verity structure)

```

#### Line-by-Line Flag & Syntax Breakdown

* `lsattr`: Utility executable that reads and formats inode flags obtained via `FS_IOC_GETFLAGS`.
* `file`: Path to the target file.

#### Exhaustive Output Anatomy

Command:

```bash
lsattr file.txt

```

Output:

```text
----i---------e---- file.txt

```

* Position 5 (`i`): `EXT4_IMMUTABLE_FL` (`0x00000010`) is active. The file is immutable.
* Position 15 (`e`): `EXT4_EXTENTS_FL` (`0x000080000`) is active. The file uses extents for block mapping on disk.
* Hyphons (`-`): Represent inactive attribute flag positions in the bitmask output layout.

---

### 6. `getcap /path/to/binary` / `setcap cap_net_bind_service=+ep /path/to/binary`

#### Fundamental Purpose & Historical Evolution

Traditionally, Unix access control followed a binary model: a process had either standard user privileges or full administrative access (`root`, UID 0). This all-or-nothing approach meant that processes needing a single privileged operation—such as binding to a low network port (<1024)—had to run with full root privileges, exposing the entire system to risk if the application was compromised.

```
Traditional SUID Model (All-or-Nothing):
  Binary Executable (SUID Root)
              |
              v
  Process acquires full root privilege (UID 0)
  All system calls permit full control (Device write, Reboot, Tracing)

Linux Capabilities Model (Fine-Grained Privileges):
  Binary Executable (security.capability xattr)
              |
              v
  Process acquires *only* CAP_NET_BIND_SERVICE
  Can bind to privileged ports (<1024)
  All other administrative operations remain restricted

```

To provide fine-grained privilege management, POSIX.1e proposed dividing monolithic root privileges into distinct units called **Capabilities**. Linux implemented this framework starting in the 2.2 kernel line and expanded it in kernel 2.6.24 by introducing File Capabilities.

File capabilities are stored in the `security.capability` extended attribute namespace. They allow executables to be granted specific privileges—such as `CAP_NET_BIND_SERVICE`—without running as SUID `root` or operating with full administrative access.

#### Under-the-Hood Execution & Kernel Mechanisms

```
[ Executive Binary Launch: execve() ]
                |
                v
  do_open_exec() -> vfs_getxattr(dentry, "security.capability")
                |
                v
  Reads binary structure: struct vfs_ns_cap_data
                |
                v
  Evaluates Linux Capability Transformation Equations:
    P'(permitted)   = (P(inheritable) & F(inheritable)) | 
                      (F(permitted) & P(bounding)) | P'(ambient)
    P'(effective)   = F(effective) ? P'(permitted) : P'(ambient)
    P'(inheritable) = P(inheritable)
                |
                v
  Updates process `struct cred`:
    cred->cap_permitted = P'(permitted)
    cred->cap_effective = P'(effective)
                |
                v
[ Execution Phase: sys_bind() on Port 80 ]
  Kernel checks cap_raised(current_cred()->cap_effective, CAP_NET_BIND_SERVICE)
    If set   -> Permit binding to port < 1024
    If clear -> Deny operation (-EACCES)

```

1. **Extended Attribute Namespace**: File capabilities are stored in the `security.capability` extended attribute. Setting capabilities calls `setxattr()`, which requires the process to hold `CAP_SETFCAP`.
2. **Capability Set Definitions**:
* **Permitted ($P_{permitted}$)**: The maximum set of capabilities the process may assume.
* **Effective ($P_{effective}$)**: The active capabilities used by the kernel to perform permission checks during system calls.
* **Inheritable ($P_{inheritable}$)**: Capabilities preserved across an `execve()` call when the executed binary also has corresponding inheritable file capabilities set.
* **Bounding ($P_{bset}$)**: A system-wide or per-process cap that restricts the capabilities a process can acquire via `execve()`.
* **Ambient ($P_{ambient}$)**: Capabilities preserved across `execve()` for non-SUID executables, simplifying capability inheritance in unprivileged programs.


3. **Capability Transformation Equations (`execve`)**: During an `execve()` call, the kernel computes the process's new capability sets using the following formulas:

$$P'_{(\text{permitted})} = (P_{(\text{inheritable})} \ \& \ F_{(\text{inheritable})}) \ | \ (F_{(\text{permitted})} \ \& \ P_{(\text{bounding})}) \ | \ P'_{(\text{ambient})}$$

$$P'_{(\text{effective})} = F_{(\text{effective})} \ ? \ P'_{(\text{permitted})} \ : \ P'_{(\text{ambient})}$$

$$P'_{(\text{inheritable})} = P_{(\text{inheritable})}$$

Where:

* $P$ denotes the process capability set prior to execution.
* $P'$ denotes the recalculated process capability set after execution.
* $F$ denotes the file capability sets read from `security.capability`.

#### Bitmask, Data Structure & Inode Field Dynamics

File capabilities are stored on disk as binary structures inside the `security.capability` extended attribute:

```c
/* Version 2 file capability structure (VFS representation) */
struct vfs_ns_cap_data {
    __le32 magic_etc; /* Version flag and Effective Bit indicator */
    struct {
        __le32 permitted;   /* Lower 32 bits of Permitted capability mask */
        __le32 inheritable; /* Lower 32 bits of Inheritable capability mask */
    } data[2];              /* Array index 0 = Bits 0..31, Index 1 = Bits 32..63 */
};

```

`magic_etc` Field Bit Layout:

* `VFS_CAP_REVISION_2` (`0x02000000`): 64-bit capability vector structure version.
* `VFS_CAP_FLAGS_EFFECTIVE` (`0x00000001`): Effective bit flag. When set, the kernel automatically enables all permitted capabilities in the effective capability set upon execution ($P'_{effective} = P'_{permitted}$).

**In-Kernel Process Credentials Capability Vectors (`struct cred`)**:

```c
struct cred {
    /* ... */
    kernel_cap_t cap_inheritable; /* Inheritable capabilities mask */
    kernel_cap_t cap_permitted;   /* Permitted capabilities mask */
    kernel_cap_t cap_effective;   /* Effective capabilities mask */
    kernel_cap_t cap_bset;        /* Bounding capabilities mask */
    kernel_cap_t cap_ambient;     /* Ambient capabilities mask */
    /* ... */
};

```

Each `kernel_cap_t` field is a 64-bit bitmask where each bit maps to a specific capability defined in `<linux/capability.h>`:

* `CAP_CHOWN`              (Bit 0): Bypasses owner checks during `chown()`.
* `CAP_DAC_OVERRIDE`       (Bit 1): Bypasses read, write, and execute DAC permission checks.
* `CAP_FOWNER`             (Bit 3): Bypasses permission checks on operations that require matching file ownership.
* `CAP_NET_BIND_SERVICE`   (Bit 10): Allows binding sockets to privileged ports (port numbers below 1024).
* `CAP_SYS_ADMIN`          (Bit 21): Grants extensive administrative capabilities (mounting, swap management, tracing).
* `CAP_LINUX_IMMUTABLE`    (Bit 9): Allows modifying `i_flags` on inodes (e.g., setting `EXT4_IMMUTABLE_FL`).

#### Line-by-Line Flag & Syntax Breakdown

* `getcap`: Utility command that queries and displays file capabilities stored in the `security.capability` extended attribute via `getxattr()`.
* `setcap`: Utility command that updates or writes file capabilities using `setxattr()`.
* `cap_net_bind_service`: Target capability flag name, resolving to Bit 10 (`0x00000400`).
* `=`: Assignment operator that replaces the existing file capability sets.
* `+e`: Enables the Effective flag (`VFS_CAP_FLAGS_EFFECTIVE`). This instructs the kernel to automatically elevate permitted capabilities to the effective capability set upon execution.
* `+p`: Sets the Permitted bit mask ($F_{permitted}$), authorizing the binary to acquire the specified capability.
* `/path/to/binary`: Target executable file path.

#### Exhaustive Output Anatomy

Command:

```bash
setcap cap_net_bind_service=+ep /usr/bin/custom_server
getcap /usr/bin/custom_server

```

Output:

```text
/usr/bin/custom_server cap_net_bind_service=ep

```

When queried via `getxattr()` and formatted by `getcap`:

* `/usr/bin/custom_server`: Path to target executable file.
* `cap_net_bind_service`: Indicates that Capability Bit 10 is present in the file's capability structure.
* `e`: The effective flag bit (`VFS_CAP_FLAGS_EFFECTIVE`) is set in `magic_etc`.
* `p`: The capability is set in the file's permitted mask ($F_{permitted}$).

When this binary executes, the kernel calculates its effective capabilities as follows:

$$P'_{(\text{permitted})} = \text{CAP\_NET\_BIND\_SERVICE} \quad (\text{Bit 10 set})$$

$$P'_{(\text{effective})} = \text{CAP\_NET\_BIND\_SERVICE} \quad (\text{Because Effective Bit } e \text{ is enabled})$$

This configuration allows the application to bind to port numbers below 1024 without running as `root` or requiring full administrative privileges.

---

## Technical Summary Matrix

| Security Layer | Data Structure / Storage | Enforcing Kernel Subsystem | Primary System Call Interfaces | Overriding Capability / Flag |
| --- | --- | --- | --- | --- |
| **POSIX Permissions** | `inode->i_mode` (Bits 0–8) | VFS DAC (`generic_permission`) | `chmod()`, `fchmodat()`, `open()` | `CAP_DAC_OVERRIDE` |
| **SUID / SGID** | `inode->i_mode` (Bits 10–11) | Executable Image Handler (`execve`) | `chmod()`, `execve()` | Checked against `MNT_NOSUID`, `NO_NEW_PRIVS` |
| **Sticky Bit** | `inode->i_mode` (Bit 9) | VFS Unlink/Rename (`may_delete`) | `chmod()`, `unlinkat()`, `renameat()` | `CAP_FOWNER` |
| **POSIX ACLs** | `system.posix_acl_access` xattr | ACL Evaluation Engine | `getxattr()`, `setxattr()` | `CAP_DAC_OVERRIDE` |
| **Inode Attributes** | Ext4 `i_flags` field | Filesystem Drivers & VFS hooks | `ioctl(FS_IOC_SETFLAGS)` | `CAP_LINUX_IMMUTABLE` |
| **Capabilities** | `security.capability` xattr | Credential Transformation Subsystem | `execve()`, `capget()`, `capset()` | Checked in specific kernel operations |