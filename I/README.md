
---

### **Phase 11: User and Group Management & Security Context**

#### **Objective:**
To transition PyKOS from a single-user system to a true multi-user operating system. This is a foundational step for security and resource isolation.

#### **Step 1: Kernel Implementation of Users and Groups**

The kernel needs to be aware of user identities to enforce permissions.

*   **Design: `User` and `Group` Structures:**
    1.  **Process Context:** The `Process` class will be extended to include a `uid` (User ID) and a `gid` (Group ID). The `init` process (PID 1) will run as `root` (`uid=0`, `gid=0`).
    2.  **Authentication Database:** A simple, plaintext file like `/etc/passwd` will be created in PyFS. The kernel will parse this file at boot time to load the user database into memory.
        *   `/etc/passwd` format: `username:password_hash:uid:gid:user_info:home_dir:shell`
        *   `/etc/group` format: `groupname:gid:user1,user2,...`
    3.  **Privilege Model:** The kernel will hard-code the concept of a "superuser" (`uid=0`), which is allowed to bypass certain permission checks.

#### **Step 2: Login Service and Session Management**

We need a way for users to log in. The initial shell should no longer run as root by default.

*   **Design: The `login` Program:**
    1.  **New `init` Process:** The kernel will now start a new system service, `/sbin/init`, as PID 1.
    2.  **`getty` and `login`:** The `init` service will spawn a `getty` process for each terminal (e.g., `/dev/tty`). `getty` will print a "login:" prompt.
    3.  When a username is entered, `getty` will `exec` a new Python program, `/bin/login`.
    4.  The `login` program will ask for a password, hash it, and compare it to the hash in `/etc/passwd`.
    5.  **`setuid()`/`setgid()` Syscalls:** If authentication is successful, `login` will call two new syscalls, `sys_setuid(uid)` and `sys_setgid(gid)`, to change its own user and group identity. This is a privileged operation that the kernel will only allow if the calling process is `root` (which `login` is, as it was spawned by `init`).
    6.  After dropping privileges, `login` will `exec` the user's specified shell (e.g., `/bin/sh.py`). This new shell process now runs as the logged-in user, not as root.

#### **Step 3: VFS Permission Enforcement**

All file-related syscalls must now rigorously enforce permissions.

*   **Design: `check_access()` Kernel Function:**
    *   Before any `sys_read`, `sys_write`, or `sys_open` operation proceeds, the VFS will call an internal `check_access()` function.
    *   This function takes the process's `uid`/`gid` and the target `VFSNode`'s inode information (owner, group, permission bits).
    *   It performs the standard UNIX permission check:
        1.  If process `uid` matches inode `uid`, check user permissions (`rwx---`).
        2.  Else, if process `gid` matches inode `gid`, check group permissions (`-rwx--`).
        3.  Otherwise, check "other" permissions (`---rwx`).
        4.  (Superuser `uid=0` always passes).
    *   If the check fails, the syscall returns an `EPERM` (Permission Denied) error to the user-space process.

---

### **Phase 12: Building a Software Ecosystem**

#### **Objective:**
To create a minimal but functional software ecosystem for PyKOS, including a package manager and a C library to allow non-Python applications to run.

#### **Step 1: The PyKOS C Library (`libpk`)**

To run programs written in C/C++, we need a C standard library that interfaces with our kernel's syscalls.

*   **Design: `libpk.a`:**
    1.  **Syscall Wrappers:** Create a set of assembly files (`syscalls.S`) that implement the `syscall` instruction interface for each of PyKOS's syscalls.
    2.  **Standard Functions:** Implement core `libc` functions (`printf`, `malloc`, `fopen`, `fread`, etc.) in C. These functions will internally call the syscall wrappers.
        *   `malloc()` will use a new `sbrk()` or `mmap()` syscall to request memory from the kernel.
        *   `printf()` and `fopen()` will use `write()` and `open()`.
    3.  **Static Library:** Compile this code into a static library (`libpk.a`). Now, C programs can be compiled with our cross-compiler and statically linked against `libpk.a` to produce a runnable PyKOS binary.

#### **Step 2: A Simple Package Manager (`pypkg`)**

A way to install new software is essential.

*   **Design: `pypkg` (written in Python):**
    1.  **Package Format:** Define a simple package format, perhaps a `.tar.gz` archive containing a `package.json` manifest file. The manifest lists the files to be installed and their destinations (e.g., `/bin`, `/lib`).
    2.  **Repository:** A simple HTTP server on the host machine can serve as the package repository.
    3.  **`pypkg` Logic:**
        *   `pypkg install <package>`: Downloads the package archive using the new networking stack.
        *   It verifies the package's checksum and signature (for security).
        *   It extracts the files into the PyFS filesystem according to the manifest.
*   **Impact:** Users can now easily extend the functionality of their PyKOS system by installing new command-line tools, libraries, or even graphical applications.

---

### **Phase 13: System Boot and Service Management**

#### **Objective:**
To replace our simple `init` process with a more robust System V-style or `systemd`-like service manager.

*   **Design: `init` as a Service Manager:**
    1.  **Runlevels/Targets:** Define different system states, like `single-user.target` (for maintenance) and `multi-user.target` (the default).
    2.  **Service Scripts:** Create a directory like `/etc/init.d/` containing simple shell scripts (or Python scripts) for starting, stopping, and checking the status of system daemons (like `loggerd`).
    3.  **`init` Logic:**
        *   On boot, `init` (PID 1) determines the target runlevel.
        *   It executes the scripts in `/etc/init.d/` in a defined order to start system services.
        *   It then starts the `getty` processes for user logins.
        *   It adopts "orphaned" processes (processes whose parent has exited) and is responsible for `wait`ing on them to prevent "zombies."
*   **Impact:** This provides a structured, configurable boot process and robust management of background services, bringing PyKOS much closer to the architecture of a modern Linux distribution.

This extended roadmap transforms PyKOS into a highly capable and recognizable multi-user, networked operating system. It provides a solid foundation for endless further exploration, from implementing new filesystems to building more complex GUI toolkits or even a web browser.
