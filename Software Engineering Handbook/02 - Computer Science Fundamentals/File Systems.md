# File Systems

> **In one line —** the layer that turns a disk full of numbered blocks into names, folders and files that humans and programs can work with.

| | |
|---|---|
| **Category** | System Software |
| **Architectural Layer** | Kernel Space |
| **Related notes** | [SSD and HDD](SSD%20and%20HDD.md) · [Kernel](Kernel.md) · [System Calls](System%20Calls.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) · [Backup Strategy](../14%20-%20Scalability%20and%20Reliability/Backup%20Strategy.md) |

---

## 1. Short Definition

*What is it?*

A file system is the organisational scheme that a [kernel](Kernel.md) imposes on raw storage. A disk only understands "block 4,829,110"; a file system provides `/home/ivan/notes.md`, permissions, timestamps and directories.

---

## 2. Purpose

*What is its main purpose?*

To give storage **structure, names and rules**. Without it, every program would have to remember which block numbers held its data and hope no other program used the same ones.

---

## 3. Problem

*What engineering problem does it solve?*

```text
Raw disk                       File system

block 0, block 1, ...          /var/log/app.log
no names                       owner, permissions, size, dates
no ownership                   directories, hierarchy
no free-space tracking         allocation and free-space management
```

It also solves **crash consistency**: if the power dies mid-write, the file system must not become unreadable.

---

## 4. Architecture Position

```text
Application
    ↓  open() / read() / write()
System calls
    ↓
Virtual File System (VFS)     ← one interface for all file system types
    ↓
ext4 / XFS / NTFS / APFS / overlayfs
    ↓
Block layer  →  driver
    ↓
SSD / HDD
```

> [!IMPORTANT]
> The **VFS** is why your code does not care whether the file is on ext4, NTFS, a network share or inside a Docker layer. One API, many implementations.

---

## 5. Real World Example

- **[Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md)** images are stacked read-only layers combined by **overlayfs**. That is why images share storage and why writing inside a container copies a file up into a writable layer.
- **Databases** often bypass much of the file system's helpfulness — PostgreSQL manages its own page layout and uses `fsync` explicitly, because it cannot afford the file system to reorder its writes.
- **Log rotation** exists because a file system with a full disk breaks everything on the machine at once.

---

## 6. Input

*What does it receive as input?*

Path-based operations from user space: open, read, write, rename, delete, list a directory, change permissions.

---

## 7. Processing

*What happens inside it?*

```text
open("/home/ivan/notes.md")
    ↓
Walk the path: /  →  home  →  ivan  →  notes.md
    ↓
Read the inode  (metadata: owner, permissions, size, block pointers)
    ↓
Check permissions against the calling process
    ↓
Return a file descriptor
```

An **inode** holds everything about a file except its name — the name lives in the directory entry that points to the inode. That separation is why hard links work, and why deleting a file that a process still has open does not free the space until the process closes it.

---

## 8. Output

*What does it return?*

File descriptors, bytes, directory listings, metadata — and error codes such as `ENOENT` (no such file), `EACCES` (permission denied), `ENOSPC` (disk full).

---

## 9. Internal Idea

*How does it work internally?*

Three structures do most of the work:

- **Superblock** — describes the file system as a whole
- **Inodes** — one per file, holding metadata and pointers to data blocks
- **Directories** — files whose contents are a list of `name → inode` mappings

Modern file systems add a **journal**: before changing anything important, they write down what they are about to do. After a crash, the journal is replayed so the file system is never left half-modified.

---

## 10. Communication

- **[System calls](System%20Calls.md)** from user space above
- **Page cache in [RAM](RAM.md)** — most reads never reach the disk
- **Block layer and drivers** below
- **Network** for NFS, SMB and similar remote file systems

---

## 11. Dependencies

- A block device ([SSD or HDD](SSD%20and%20HDD.md)), or a network equivalent
- Kernel support for the specific file system type
- Free space and free inodes — **you can run out of inodes while disk space remains**, which produces a very confusing "disk full" error

---

## 12. Alternatives

```text
ext4        Linux default; reliable, well understood
XFS         excellent with large files and parallel I/O
Btrfs/ZFS   snapshots, checksums, built-in RAID; heavier
NTFS/APFS   Windows and macOS defaults
overlayfs   layered, used by containers
NFS/SMB     network file systems
S3/MinIO    object storage — not a file system at all, but often the right answer
```

> [!TIP]
> For user uploads in a web application, **object storage is almost always the right choice** over a file system. It scales, it is shared between servers, and it does not fill up your disk.

---

## 13. When To Use

> [!TIP]
> Use the file system for configuration, logs, temporary working files and anything local to one machine's lifetime.

---

## 14. When NOT To Use

> [!CAUTION]
> - **Do not store user uploads on an application server's local disk.** Add a second server and half your requests will not find the file.
> - **Do not use the file system as a database.** No transactions across files, no queries, no concurrency control.
> - **Do not use it as a queue.** Directory-scanning "queues" break under concurrency in subtle, data-losing ways.

---

## 15. Advantages

- Universal, simple API that every language supports
- Fast for local access, with automatic RAM caching by the kernel
- Permissions and ownership built in
- Journaling protects structural integrity across crashes

---

## 16. Disadvantages

- Local to one machine unless you add a network file system
- No transactions spanning multiple files
- Directories with millions of entries perform badly
- Full disks and exhausted inodes cause confusing, wide-reaching failures
- File locking across processes is notoriously unreliable, especially over NFS

---

## 17. Performance Impact

| Resource | Impact |
|---|---|
| **Speed** | Cached reads are RAM-fast; uncached reads pay full disk latency |
| **Memory** | The kernel uses free RAM as a page cache — high "used memory" here is healthy |
| **CPU** | Low, except for checksumming file systems like ZFS |
| **GPU** | Only relevant when slow file reads starve a training pipeline |
| **Network** | Network file systems add latency to every metadata operation, not just reads |

---

## 18. Security Considerations

> [!CAUTION]
> **Path traversal** is one of the most common web vulnerabilities: a user-supplied filename containing `../../etc/passwd` reaching a file operation. Never build a path from unvalidated user input.

- **Permissions** — the classic `rwx` for owner, group and others; do not solve problems with `chmod 777`
- **Symlink attacks** — a symlink swapped between check and use lets an attacker redirect a privileged write
- **Deleted is not erased** — see [SSD and HDD](SSD%20and%20HDD.md)
- **Encryption at rest** (LUKS, BitLocker) protects a stolen or discarded disk
- **Temporary files** created with predictable names in shared directories are a long-standing vulnerability pattern

---

## 19. Mental Model

> [!NOTE]
> **A file system is a library's catalogue.**
>
> The shelves hold numbered boxes ([disk blocks](SSD%20and%20HDD.md)). The catalogue maps a title you can remember to the box that holds it, records who may borrow it, and keeps a log of pending changes so an interrupted reshelving can be finished later.

---

## 20. Mini Architecture Diagram

```text
Application
    ↓
System calls
    ↓
VFS
    ↓
ext4 / XFS / overlayfs
    ↓
Page cache (RAM)  ←── most reads stop here
    ↓
Block layer → driver → disk
```

---

## 21. Complete Request Flow

```text
Application: open("/var/app/config.json")
    ↓
Syscall into the kernel
    ↓
VFS resolves the path component by component
    ↓
Permission check against the process's user and group
    ↓
Inode read (often already cached)
    ↓
File descriptor returned
    ↓
read() → is the block in the page cache?
         YES → memcpy, ~100 ns
         NO  → disk read (~100 µs), then cached
    ↓
Bytes delivered to the application
```

---

## 22. Key Takeaway

> [!IMPORTANT]
> A file system turns numbered blocks into named, permissioned, crash-resistant files — but it is local to one machine, which is why distributed systems reach for object storage instead.

---

## 23. Common Mistakes

- **Storing uploads locally** in an application that will ever have more than one server
- **Building paths from user input** without validation — path traversal
- **Using files as a database or a queue**
- **Never monitoring free space or inodes** — a full disk is a top cause of production incidents
- **Assuming `write()` means "on disk"** — it means "in the page cache"; only `fsync` guarantees persistence
- **Millions of files in one directory**

---

## 24. Open Source Technologies

- **ext4**, **XFS**, **Btrfs**, **ZFS** — Linux file systems
- **overlayfs** — the layering behind container images
- **FUSE** — file systems implemented in user space
- **MinIO**, **Ceph** — object and distributed storage, the usual right answer for user data
- **inotify** — watch the file system for changes

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **To research later:**

---

## 26. Workbook Exercise

- [ ] Check free disk space *and* free inodes on a machine you use (`df -h` and `df -i`).
- [ ] Find any place in your code where a file path is built from user input, and validate it.
- [ ] Decide where user uploads in your project should live, and justify the choice in two sentences.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Application
    ↓
System calls → VFS → file system
    ↓
Page cache (RAM)
    ↓
Disk
```

## 2. Request Flow

```text
Input       a path and an operation
    ↓
Processing  path resolution → inode → permission check → cached or physical read
    ↓
Output      a file descriptor, bytes, or an error code
```

## 3. Real-World Usage

**Docker** builds every image out of stacked overlayfs layers. Sharing unchanged layers between images is why pulling a second image based on the same base is nearly instant — a file system feature turned into the core of a product.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The kernel layer that turns disk blocks into named files and directories |
| **Why does it exist?** | Because raw block storage has no names, ownership or crash safety |
| **Where does it belong?** | Inside the kernel, between system calls and the block layer |
| **When should I use it?** | For local, single-machine data — use object storage for anything shared |
