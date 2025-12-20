

---

### **Phase 7: System Hardening & Feature Expansion**

#### **Objective:**
To mature PyKOS from an educational prototype into a more stable and capable operating system. This involves improving memory management, adding more complex scheduling, expanding the filesystem's capabilities, and enriching the user-space environment.

#### **Step 1: Advanced Memory Management**

Our current `fork()` implementation with full memory duplication is simple but inefficient. The standard solution is **Copy-on-Write (COW)**.

*   **Design: Implementing Copy-on-Write (COW):**
    1.  **Modify `sys_fork()`:** When `fork()` is called, instead of duplicating all the parent's memory pages, the kernel will:
        *   Create a new address space for the child.
        *   Map the *same* physical pages from the parent into the child's address space.
        *   Mark the corresponding Page Table Entries (PTEs) for both the parent and the child as **read-only**.
    2.  **Enhance the Page Fault Handler (ISR 14):**
        *   When a process (either parent or child) attempts to *write* to a COW page, the CPU will trigger a Page Fault because the page is marked read-only.
        *   Our Page Fault handler will check a special flag in the PTE to see if this is a COW fault.
        *   If it is, the handler will:
            a.  Allocate a new, fresh physical page.
            b.  Copy the contents of the original (shared) page to the new page.
            c.  Update the faulting process's page table to map the virtual address to this new physical page, now with **read-write** permissions.
            d.  If the original page now only has one reference (the other process), its PTE can also be marked as read-write.
*   **Impact:** This dramatically speeds up `fork()` and reduces memory consumption, as memory pages are only duplicated when they are actually modified.

#### **Step 2: A More Sophisticated Scheduler**

Round-robin is fair but inefficient. A **priority-based scheduler** would allow more important processes (like interactive shells) to be more responsive than background tasks.

*   **Design: Multi-Level Feedback Queue Scheduler:**
    1.  **Process Priority:** Add a `priority` field to the `Process` struct.
    2.  **Ready Queues:** Instead of one `ready_queue`, the scheduler will maintain multiple queues (e.g., `high_priority_q`, `normal_priority_q`, `low_priority_q`).
    3.  **Scheduling Logic:** The scheduler always picks a process from the highest-priority queue that is not empty.
    4.  **Dynamic Priority Adjustment:**
        *   Processes that use their full time slice without blocking (e.g., for I/O) are likely CPU-bound. Their priority is lowered, and they are moved to a lower-priority queue.
        *   Processes that block frequently (e.g., an interactive shell waiting for keyboard input) are likely I/O-bound. Their priority is boosted when they become ready again.
*   **Impact:** The interactive shell will feel much more responsive, even when a CPU-intensive script is running in the background.

#### **Step 3: Filesystem Enhancements (Permissions & Symlinks)**

Our PyFS is functional but lacks basic UNIX-like features.

*   **Design: Adding Permissions and Ownership:**
    1.  **Modify `PyFS_Inode`:** Add `uid` (user ID), `gid` (group ID), and `mode` (permission bits like `rwx`) fields to the on-disk inode structure.
    2.  **Modify `Process`:** Add `uid` and `gid` to the `Process` class in the kernel.
    3.  **Enforce Permissions:** Update all VFS syscalls (`sys_open`, `sys_read`, `sys_write`) to check the process's `uid`/`gid` against the inode's permissions before allowing access.
*   **Design: Adding Symbolic Links (Symlinks):**
    1.  **New Inode Type:** Add a new file type to the `mode` field in the inode for symlinks.
    2.  **Store Target Path:** For a symlink inode, instead of data block pointers, its data block will store the target path as a string.
    3.  **Modify Path Traversal:** When the VFS traversal encounters a symlink, it will read the target path from the symlink's data block and restart the traversal from that new path. The kernel must also keep track of a recursion depth to prevent infinite symlink loops.

#### **Step 4: Expanding the User-Space Environment**

A bare shell is limited. We need more standard command-line utilities written in Python.

*   **New Python Scripts for `/bin`:**
    *   **`ls.py`:** Would use new syscalls (`readdir`, `stat`) to read directory entries and display file metadata (like permissions and size).
    *   **`mkdir.py`, `touch.py`, `rm.py`:** Use syscalls for creating/deleting files and directories.
    *   **`ps.py`:** A new `/proc` virtual filesystem (`procfs`) could be created, similar to `devfs`. The kernel would populate it with directories for each running process. `ps.py` would then read files like `/proc/1/status` to get information about process 1.
    *   **`sh.py`:** Our `init.py` could be renamed to `sh.py` (for "shell"), and a new, simpler `init.py` could be created to act as a system initializer (e.g., mounting filesystems, starting background services before launching the shell).

#### **Step 5: System Services (Daemons)**

With a stable multitasking environment, we can introduce the concept of system services or daemons.

*   **Example: A Logger Daemon (`loggerd.py`):**
    1.  **Create a Named Pipe:** The kernel could be extended to support named pipes (FIFOs), which appear as special files in the filesystem (e.g., `/dev/log`).
    2.  **Launch at Boot:** The `init.py` script would launch `loggerd.py` in the background.
    3.  **Logger Logic:** `loggerd.py` would open `/dev/log` for reading and enter a loop. It reads any messages sent to the pipe and writes them to a persistent log file (e.g., `/var/log/system.log`).
    4.  **System-wide Logging:** Any other process (including the kernel via a special VFS call) can now open `/dev/log` for writing and send log messages to a centralized service.

---
