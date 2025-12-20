
### **Project: PyKOS (The Python Kernel Operating System)**

**Core Philosophy:** To build a simple, clean, and well-documented x86-64 operating system that serves as a transparent educational tool. PyKOS demystifies core OS concepts by implementing them in modern C++ and exposing them to a flexible, high-level Python user space. Its hybrid design highlights the crucial boundary and interaction between kernel and user modes, demonstrating how a low-level, high-performance core can empower a dynamic, high-level environment.

---

### **High-Level Architecture Diagram**

This diagram illustrates the fundamental separation of privilege and the key components of PyKOS.

```mermaid
graph TD
    subgraph "Hardware (Physical or Emulated)"
        CPU
        RAM
        Disk
        Keyboard
        Serial
    end

    subgraph "PyKOS Kernel (C++ | Ring 0)"
        A[Bootloader - GRUB] --> B(Kernel Entry);
        B --> C{Core Initialization};
        C --> D[GDT & IDT Setup];
        D --> E[Interrupt & Exception Handlers];
        C --> F[Physical & Virtual Memory Mgmt];
        F --> G[Paging & Page Frame Allocator];
        C --> H[Scheduler & Process Mgmt];
        H --> I[Process Table];
        C --> J[Device Drivers];
        J --> J1(Keyboard);
        J --> J2(Serial/TTY);
        J --> J3(VGA Text);
        J --> J4(ATA Block Device);
        C --> K[VFS Layer];
        K --> L[devfs];
        K --> M[PyFS Filesystem];
        M --> J4;
        L --> J1;
        L --> J2;
        L --> J3;
        C --> N[IPC Manager - Pipes];
        C --> O[Syscall Handler];
    end

    subgraph "User Space (Python | Ring 3)"
        P[Python Interpreter & StdLib];
        P --> Q[Python Shell (init.py)];
        Q --"fork() / exec()"--> R(User Scripts);
        R --"open/read/write"--> O;
        Q --"open/read/write"--> O;
        Q --"pipe()"--> O;
    end

    O -- "Syscall/Sysret ABI" -- P;
    
    style CPU fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#bbf,stroke:#333,stroke-width:2px
    style O fill:#ff9,stroke:#333,stroke-width:4px
    style P fill:#9f9,stroke:#333,stroke-width:2px
```

---

## **Project Roadmap: A Phased Approach**

This project is broken down into manageable phases, each building upon the last.

#### **Phase 0: Environment & Bootstrapping (The "Hello, Kernel!")**
*   **Objective:** Set up a cross-compiler toolchain and boot a minimal C++ kernel.
*   **Tasks:**
    1.  Install dependencies: `build-essential`, `nasm`, `xorriso`, `qemu`.
    2.  Build a cross-compiler toolchain (e.g., `gcc`, `binutils`) targeting `x86_64-elf`.
    3.  Create a linker script to correctly place the kernel in memory.
    4.  Write a minimal bootloader assembly stub (`boot.asm`) that sets up a 64-bit long mode environment.
    5.  Configure GRUB to load the kernel binary.
    6.  Write a minimal C++ kernel (`kernel.cpp`) that clears the screen and prints a message using VGA text mode.
    7.  Package into a bootable ISO image and run in QEMU.
*   **Outcome:** A bootable OS that displays "Welcome to PyKOS!" on the screen.

#### **Phase 1: Core Kernel Services (Memory & Interrupts)**
*   **Objective:** Establish fundamental memory management and interrupt handling.
*   **Tasks:**
    1.  **GDT & IDT:** Set up the Global Descriptor Table and Interrupt Descriptor Table.
    2.  **ISRs:** Implement Interrupt Service Routines for CPU exceptions (e.g., Page Fault, General Protection Fault) and hardware interrupts (IRQs).
    3.  **PIC/APIC:** Program the Programmable Interrupt Controller to route hardware IRQs.
    4.  **Memory Management:** Implement a physical page frame allocator (e.g., using a bitmap) to track used and free RAM.
    5.  **Virtual Memory (Paging):** Implement a paging system (PML4, PDPT, PD, PT) that maps kernel code and data to a high memory address (e.g., the higher half) and identity-maps physical memory for now.
*   **Outcome:** A kernel that can handle exceptions gracefully and has full control over the machine's memory.

#### **Phase 2: Multitasking & Scheduling**
*   **Objective:** Introduce processes and a preemptive scheduler.
*   **Tasks:**
    1.  **Process Control Block (PCB):** Design a `Process` C++ class to store state (registers, stack, page directory).
    2.  **Scheduler:** Implement a simple, round-robin preemptive scheduler.
    3.  **Timer Interrupt:** Set up the PIT (Programmable Interval Timer) or HPET to fire periodically, triggering a context switch.
    4.  **`fork()`:** Implement a basic `fork()` mechanism that duplicates a process and its memory space.
*   **Outcome:** The kernel can run multiple "tasks" (e.g., functions that print different characters to the screen) concurrently.

#### **Phase 3: The Python Bridge (User Mode & Syscalls)**
*   **Objective:** Boot into a Python user-space environment.
*   **Tasks:**
    1.  **Initrd:** Create a TAR archive (`initrd.tar`) containing the Python interpreter ELF binary, its standard library (`.py` files), and an initial shell script (`init.py`).
    2.  **Kernel Loading:** The kernel learns to parse the `initrd` passed by GRUB.
    3.  **`exec()`:** Implement the `exec()` logic: create a new user-mode process, set up its virtual address space, load the Python ELF binary from the `initrd`, set up a user stack, and jump to its entry point.
    4.  **System Call ABI:** Design and implement the syscall handler. This involves using the `syscall`/`sysret` instructions and a defined register convention (`rax` for syscall number, `rdi`, `rsi`, etc. for args).
    5.  **Sandboxing:** Enforce memory protection using paging. The user process has its own page tables and cannot access kernel memory or other processes' memory. Privileged instructions will cause a General Protection Fault.
*   **Outcome:** PyKOS boots, the kernel initializes, and then starts a single Python process which runs a simple script (e.g., `print("Hello from Python user space!")`) using a syscall to write to the serial console.

#### **Phase 4: I/O and Device Abstraction (VFS)**
*   **Objective:** Create a unified interface for device I/O.
*   **Tasks:**
    1.  **Device Drivers:** Refactor the keyboard and serial port code into formal `DeviceDriver` classes.
    2.  **VFS Layer:** Design the abstract VFS classes (`VFSNode`, `FileSystem`).
    3.  **`devfs`:** Implement a `devfs` filesystem where files like `/dev/tty` and `/dev/kbd` are special `VFSNode`s that point directly to the driver instances.
    4.  **Syscalls:** Implement `open`, `read`, `write`, `close` syscalls that operate on file descriptors and route through the VFS.
*   **Outcome:** Python scripts in user space can now interact with devices using standard file operations (e.g., `kbd_file = open('/dev/kbd', 'r'); char = kbd_file.read(1);`).

#### **Phase 5: Persistent Storage (Filesystem)**
*   **Objective:** Implement a simple, persistent on-disk filesystem.
*   **Tasks:**
    1.  **Block Device Driver:** Write a driver for a simple block device (e.g., ATA PATA for QEMU's default virtual disk).
    2.  **PyFS Design:** Design the on-disk layout for "PyFS": superblock, inode table, data/inode bitmaps, and directory entries.
    3.  **Filesystem Implementation:** Write the C++ code to read/write inodes and data blocks, manage allocations, and traverse directories.
    4.  **`mount()`:** Implement a `mount` syscall and have the kernel mount a PyFS-formatted virtual disk at the VFS root (`/`).
*   **Outcome:** The Python user space can create files and directories that persist after the OS is shut down and rebooted.

#### **Phase 6: The Interactive Shell**
*   **Objective:** Create a functional user shell in Python.
*   **Tasks:**
    1.  **Shell Script:** Write `init.py` as an interactive REPL.
    2.  **Process Lifecycle:** Implement logic in the shell to `fork` and `exec` other Python scripts. Add support for foreground (waiting for child to exit) and background (`&`) processes.
    3.  **IPC:** Implement pipes in the kernel and expose them via a `pipe()` syscall. The shell will use this for command pipelines (`|`).
    4.  **I/O Redirection:** The shell will parse `>` and `<` and use `open()` and `dup2()` syscalls to redirect `stdout`/`stdin` before calling `exec`.
*   **Outcome:** A fully interactive PyKOS system where a user can navigate the filesystem, run scripts, and pipe commands together, all from a Python shell.

---

## **Detailed Technical Blueprint**

### **1. C++ Kernel & Python User Space Integration**

*   **Initrd:** The `initrd.tar` archive is the key. It's a simple, uncompressed TAR file. The kernel will contain a minimal TAR parser.
    *   **Structure:**
        ```
        initrd.tar
        ├── bin/
        │   └── python  (Statically linked x86-64 ELF)
        ├── lib/
        │   └── python3.x/ (The standard library .py files)
        └── etc/
            └── init.py (The initial shell script)
        ```
*   **Kernel's `exec` Syscall Logic:**
    1.  A user process (e.g., the shell) calls `sys_exec("/bin/python", ["init.py"])`.
    2.  The kernel finds `/bin/python` in the `initrd`.
    3.  It creates a new, empty virtual address space for the new process. This involves allocating and setting up a new PML4 table.
    4.  The kernel's own memory region (the higher half) is mapped into this new address space to allow seamless transitions back to the kernel via syscalls or interrupts.
    5.  The kernel reads the Python ELF header to identify program segments (`.text`, `.data`).
    6.  For each segment, it allocates physical pages, copies the segment data from the `initrd` into them, and maps these pages into the new process's virtual address space with the correct permissions (Read/Execute for `.text`, Read/Write for `.data`).
    7.  It allocates pages for a user-mode stack and maps them into the address space.
    8.  It prepares a trap frame on the kernel stack with the target registers for user mode: `RIP` points to the Python ELF entry point, `RSP` points to the new user stack, and segment registers (`CS`, `SS`) are set to user-mode values.
    9.  The kernel executes an `IRETQ` instruction, which pops the values from the trap frame into the CPU's registers, atomically switching the CPU from Ring 0 to Ring 3 and jumping to the Python interpreter's first instruction.

### **2. System Call ABI Design**

*   **Instruction:** `syscall` for user->kernel, `sysret` for kernel->user.
*   **Register Convention (x86-64):**
    *   `RAX`: System call number (e.g., 1 for `read`, 2 for `write`).
    *   `RDI`, `RSI`, `RDX`, `R10`, `R8`, `R9`: Arguments 1 through 6.
    *   `RAX`: Return value from the kernel.
*   **Kernel-Side Handler:** The `syscall` instruction triggers an interrupt. The IDT points to a C++-callable assembly stub that saves all user registers (the trap frame), switches to the kernel stack, and calls a main C++ `syscall_dispatcher(trap_frame*)` function. This function uses a lookup table based on `trap_frame->rax` to call the appropriate kernel function (e.g., `sys_read`).
*   **Python-Side Wrapper:** A simple Python library using `ctypes` or a custom C extension will provide functions like `os_read(fd, count)` that internally place the arguments in the correct registers and execute the `syscall` instruction.

### **3. Inter-Process Communication (IPC): Pipes**

*   **Kernel Data Structure (`Pipe`):**
    ```cpp
    class Pipe {
        char buffer[PIPE_SIZE];
        size_t read_ptr;
        size_t write_ptr;
        size_t count;
        // Wait queues for processes blocked on read/write
        WaitQueue readers_waiting;
        WaitQueue writers_waiting;
    };
    ```
*   **`pipe()` Syscall:** Creates a `Pipe` object in kernel memory. Creates two file descriptors for the calling process: one for reading (`fd[0]`), one for writing (`fd[1]`). Both point to the same `Pipe` object but with different access flags.
*   **`read()`/`write()` on Pipe:**
    *   `write()`: If the buffer is full, the writing process is added to `writers_waiting` and the scheduler is invoked. Otherwise, data is copied to the buffer and any waiting readers are woken up.
    *   `read()`: If the buffer is empty, the reading process is added to `readers_waiting` and the scheduler is invoked. Otherwise, data is copied to the user buffer and any waiting writers are woken up.
*   **`ls | grep .py` Example:**
    1.  The shell calls `pipe()`, getting `pipe_fds[0]` (read) and `pipe_fds[1]` (write).
    2.  The shell calls `fork()` twice, creating two children (for `ls` and `grep`).
    3.  **`ls` Process:** Closes `pipe_fds[0]`. Calls `dup2(pipe_fds[1], 1)`, redirecting its standard output to the pipe's write end. Closes the original `pipe_fds[1]`. Calls `exec("ls")`.
    4.  **`grep` Process:** Closes `pipe_fds[1]`. Calls `dup2(pipe_fds[0], 0)`, redirecting its standard input to the pipe's read end. Closes the original `pipe_fds[0]`. Calls `exec("grep", [".py"])`.
    5.  The shell closes both `pipe_fds` and waits for both children to complete.

### **4. Virtual File System (VFS) and `devfs`**

*   **VFS Abstractions (C++):**
    ```cpp
    class VFSNode {
    public:
        virtual ~VFSNode() {}
        virtual ssize_t read(void* buffer, size_t count) = 0;
        virtual ssize_t write(const void* buffer, size_t count) = 0;
        // ... other methods like open, close, readdir
    };

    class DeviceDriver : public VFSNode { ... };
    class TTYDriver : public DeviceDriver { ... };
    ```
*   **`devfs` Implementation:**
    *   `devfs` is a simple, in-memory `FileSystem` implementation.
    *   At kernel boot, it's populated with `VFSNode`s that are actually instances of driver classes. For example, `devfs->create_node("/dev/tty", new TTYDriver())`.
*   **Syscall Routing:**
    *   A process calls `sys_open("/dev/tty", ...)`
    *   The kernel's `sys_open` function traverses the VFS tree starting from the root to find the `/dev/tty` node.
    *   It finds the `TTYDriver` instance.
    *   It creates a new file descriptor in the process's descriptor table and points it to this `TTYDriver` instance.
    *   A subsequent `sys_read(fd, ...)` on that descriptor will call `TTYDriver->read()`.

### **5. Persistent Filesystem: "PyFS"**

*   **On-Disk Layout (Example for a 1MB disk):**
    *   **Block 0: Superblock:** Magic number (`0x50794653`), total blocks, total inodes, block size (e.g., 4096 bytes).
    *   **Block 1: Inode Bitmap:** One bit per inode, marking it as free or used.
    *   **Block 2: Data Block Bitmap:** One bit per data block, marking it as free or used.
    *   **Blocks 3-10: Inode Table:** An array of `Inode` structs.
        ```c
        struct Inode {
            uint16_t mode; // File type (file/dir), permissions
            uint32_t size;
            uint32_t direct_pointers[12]; // Pointers to data blocks
            uint32_t indirect_pointer;
        };
        ```
    *   **Blocks 11-255: Data Blocks:** Where actual file content and directory entries are stored.
*   **Directory Structure:** A directory is a file whose data blocks contain a list of `DirectoryEntry` structs.
    ```c
    struct DirectoryEntry {
        uint32_t inode_number;
        char name[252];
    };
    ```
*   **Kernel Implementation:** The `PyFS` implementation will have functions like `pyfs_read_inode(ino)`, `pyfs_alloc_block()`, `pyfs_read_block(block_num, buffer)`. These will use the `ATADriver` to perform the underlying PIO reads/writes to the virtual disk. The `mount` syscall will register an instance of this `PyFS` class with the VFS.

### **6. The Python Shell (`init.py`)**

This script ties everything together, using the syscalls exposed by the kernel.

```python
import os_api # A custom library wrapping syscalls

def main():
    while True:
        try:
            cwd = os_api.getcwd()
            command_line = input(f"PyKOS:{cwd}$ ")
            if not command_line:
                continue

            # Basic parsing for redirection and pipelines
            parts = command_line.split()
            program = parts[0]
            args = parts[1:]

            # Simple foreground/background
            is_background = False
            if args and args[-1] == '&':
                is_background = True
                args.pop()

            pid = os_api.fork()
            if pid == 0:
                # --- In Child Process ---
                # Handle I/O redirection and pipelines here
                # e.g., if '>' in args: setup_redirection(args)
                
                # Execute the program
                os_api.exec(program, args)
                # exec should not return; if it does, it's an error
                print(f"Error: command not found: {program}")
                os_api.exit(1)
            else:
                # --- In Parent Process (Shell) ---
                if not is_background:
                    # Wait for the child process to complete
                    status = os_api.waitpid(pid)
                    print(f"[Process {pid} exited with status {status}]")

        except KeyboardInterrupt:
            print() # Handle Ctrl+C gracefully
        except EOFError:
            break # Handle Ctrl+D to exit

if __name__ == "__main__":
    main()
```

This comprehensive blueprint outlines a clear, phased path to creating PyKOS. Each step builds logically on the previous one, culminating in a simple yet powerful educational operating system that beautifully demonstrates the synergy between a low-level C++ kernel and a high-level Python user space.
educational operating system


