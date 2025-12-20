### **Phase 6: The Interactive Shell**

#### **Objective:**
To write a Python shell (`init.py`) that provides a read-eval-print loop (REPL). This shell must be able to launch other programs (Python scripts), manage them as foreground or background processes, and handle I/O redirection (`<`, `>`) and command pipelines (`|`) using the system calls we've implemented.

#### **Step 1: Expanding the System Call Interface**

Our shell requires more powerful syscalls than just basic file I/O. We need to implement process management and more advanced file descriptor manipulation in the C++ kernel.

**New Syscalls to be added to `syscall.cpp`:**

1.  **`fork()` -> `sys_fork()`:**
    *   **Kernel Logic:** This is a complex but crucial syscall.
        1.  Create a new `Process` object for the child.
        2.  Duplicate the parent's `CPUState` (the trap frame from the syscall).
        3.  **Crucially, duplicate the parent's virtual address space.** This involves creating a new PML4 and copying all of the parent's page table entries. For efficiency, this should be implemented as **copy-on-write (COW)**. Initially, both processes' page tables point to the same physical pages, but the pages are marked as read-only. If either process tries to write to a page, a Page Fault occurs, and only then does the kernel duplicate that specific page.
        4.  Duplicate the parent's file descriptor table.
        5.  Add the child process to the scheduler's ready queue.
        6.  In the parent's context, `sys_fork` returns the child's PID.
        7.  In the child's context, `sys_fork` returns `0`. This is achieved by modifying the `rax` register in the child's saved `CPUState` before it ever runs.

2.  **`exec(path, argv)` -> `sys_exec()`:**
    *   **Kernel Logic:** We've already designed this. `sys_exec` will replace the *current* process's memory image with a new program loaded from the filesystem (either the `initrd` or PyFS). It reuses the same PID but gets a completely new address space and stack.

3.  **`waitpid(pid)` -> `sys_waitpid()`:**
    *   **Kernel Logic:** If the specified `pid` is a child of the current process and has terminated, this syscall returns the child's exit status. If the child is still running, the parent process's state is set to `Blocked`, and it is removed from the scheduler's ready queue. When the child process eventually calls `exit`, the kernel will find the waiting parent, set its state back to `Ready`, and place it back in the scheduler's queue.

4.  **`pipe(fds)` -> `sys_pipe()`:**
    *   **Kernel Logic:** We designed this in the blueprint. It creates a `Pipe` kernel object and returns two file descriptors.

5.  **`dup2(oldfd, newfd)` -> `sys_dup2()`:**
    *   **Kernel Logic:** This manipulates the process's file descriptor table. It makes `newfd` point to the same underlying `FileDescriptor` object as `oldfd`. If `newfd` was already open, it is closed first. This is the key to I/O redirection.

6.  **`exit(status)` -> `sys_exit()`:**
    *   **Kernel Logic:** Marks the current process as `Terminated`. It cleans up its resources (memory, file descriptors). If the parent process is waiting (`sys_waitpid`), the kernel wakes it up and provides the exit `status`.

#### **Step 2: The Python `os_api.py` Library**

We need to create a user-space Python library that makes these new syscalls easy to use. This library would be part of our `initrd`.

**File:** `initrd_contents/lib/os_api.py`
```python
import _pykos_syscall as syscalls

# Syscall numbers
SYS_EXIT = 0
SYS_WRITE = 1
SYS_READ = 2
SYS_OPEN = 3
SYS_CLOSE = 4
SYS_FORK = 5
SYS_EXEC = 6
SYS_WAITPID = 7
SYS_PIPE = 8
SYS_DUP2 = 9
# ... and so on

# File open flags
O_RDONLY = 0
O_WRONLY = 1
O_RDWR = 2
O_CREAT = 4

def exit(status):
    syscalls.syscall(SYS_EXIT, status)

def fork():
    return syscalls.syscall(SYS_FORK)

def exec(path, args):
    # The kernel needs a null-terminated list of string pointers.
    # This function would need to carefully pack 'path' and 'args'
    # into a memory structure the kernel can understand.
    return syscalls.syscall(SYS_EXEC, path, args)

def waitpid(pid):
    return syscalls.syscall(SYS_WAITPID, pid)
    
def pipe():
    # Kernel will write two integers into a buffer we provide.
    fds_buffer = bytearray(8) # Space for two 64-bit integers
    syscalls.syscall(SYS_PIPE, fds_buffer)
    read_fd = int.from_bytes(fds_buffer[0:4], 'little')
    write_fd = int.from_bytes(fds_buffer[4:8], 'little')
    return read_fd, write_fd

def dup2(oldfd, newfd):
    return syscalls.syscall(SYS_DUP2, oldfd, newfd)
    
# ... wrappers for open, read, write, close ...
```

#### **Step 3: The Interactive Shell (`init.py`)**

Now we write the full-featured shell. It will be the `init` process (PID 1).

**File:** `initrd_contents/etc/init.py` (Final Version)
```python
import os_api

STDOUT_FD = 1
STDIN_FD = 0

def kprint(msg):
    os_api.write(STDOUT_FD, msg.encode())

def main():
    kprint("PyKOS v1.0 Shell Initialized.\n")

    while True:
        try:
            # 1. Read-Eval-Print Loop (REPL)
            kprint("$ ")
            command_line = os_api.readline(STDIN_FD).strip() # New syscall we'd need
            if not command_line:
                continue

            # 2. Command Parsing
            parts = command_line.split()
            program = parts[0]
            args = parts[1:]

            is_background = '&' in args
            if is_background:
                args.remove('&')

            # 3. Process Creation
            pid = os_api.fork()

            if pid == 0:
                # --- CHILD PROCESS ---
                # This code runs in the new child process.
                
                # 4. I/O Redirection & Pipelines
                # This is a simplified parser. A real shell's is much more complex.
                if '|' in args:
                    # Handle pipeline: e.g., "ls | grep py"
                    pipe_read_fd, pipe_write_fd = os_api.pipe()
                    
                    cmd1_parts = args[0:args.index('|')]
                    cmd2_parts = args[args.index('|') + 1:]
                    
                    child_pid = os_api.fork()
                    if child_pid == 0:
                        # Grandchild process (left side of pipe, e.g., 'ls')
                        os_api.close(pipe_read_fd)
                        os_api.dup2(pipe_write_fd, STDOUT_FD) # stdout -> pipe
                        os_api.exec(cmd1_parts[0], cmd1_parts)
                    else:
                        # Original child process (right side of pipe, e.g., 'grep')
                        os_api.close(pipe_write_fd)
                        os_api.dup2(pipe_read_fd, STDIN_FD) # stdin <- pipe
                        os_api.exec(cmd2_parts[0], cmd2_parts)

                elif '>' in args:
                    # Handle output redirection: e.g., "echo hello > /hello.txt"
                    file_path_index = args.index('>') + 1
                    file_path = args[file_path_index]
                    cmd_parts = args[0:args.index('>')]
                    
                    fd = os_api.open(file_path, os_api.O_WRONLY | os_api.O_CREAT)
                    os_api.dup2(fd, STDOUT_FD) # stdout -> file
                    os_api.close(fd)
                    os_api.exec(cmd_parts[0], cmd_parts)
                
                else:
                    # Simple command execution
                    os_api.exec(program, args)
                
                # If exec fails, we'll get here.
                kprint(f"Error: Command '{program}' not found.\n")
                os_api.exit(127)

            else:
                # --- PARENT PROCESS (SHELL) ---
                if not is_background:
                    # 5. Process Lifecycle Management
                    # Wait for the foreground child to finish
                    status = os_api.waitpid(pid)
                    kprint(f"[{pid} exited with status {status}]\n")
                else:
                    # For a background process, just print its PID and continue
                    kprint(f"[{pid} started in background]\n")

        except EOFError:
            kprint("logout\n")
            break # Exit shell on Ctrl+D
        except Exception as e:
            kprint(f"Shell error: {e}\n")

if __name__ == "__main__":
    # Before starting the shell, we need to ensure stdin/stdout/stderr exist.
    # The kernel should set up file descriptors 0, 1, 2 to point to /dev/tty
    # for the init process.
    main()

```

#### **Step 4: Final Polish and Build**

1.  **Kernel Update:** The kernel's `exec` logic for the very first process needs to be updated to automatically create file descriptors 0, 1, and 2, all pointing to `/dev/tty`. This gives the shell its standard I/O channels.
2.  **Sample User Programs:** To make the shell useful, we'd add other simple Python scripts to our `initrd` in a `/bin` directory, like `ls.py`, `cat.py`, `echo.py`. These scripts would use the `os_api` to interact with the VFS and print their results.
3.  **Final `Makefile` Run:** The final `make run` command will build the updated kernel, package the new `init.py` and other scripts into the `initrd`, create the `pykos.iso`, and boot it in QEMU.

**The Final User Experience:**

When you boot PyKOS, you will see the kernel messages, and then:

```
PyKOS v1.0 Shell Initialized.
$ 
```

You can now type commands:

```
$ echo "Hello, PyKOS!" > /greeting.txt
[10 exited with status 0]
$ cat /greeting.txt
Hello, PyKOS!
[11 exited with status 0]
$ ls | grep greeting
greeting.txt
[13 exited with status 0]
[12 exited with status 0]
$ 
```

