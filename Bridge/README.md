## **Phase 3: The Python Bridge (User Mode & Syscalls)**

#### **Objective:**
To prepare and load a Python interpreter as the first user-space process (`init`). This requires creating an initial ramdisk (`initrd`), implementing the `exec` logic to load an ELF binary, designing the system call Application Binary Interface (ABI), and enforcing memory protection (sandboxing) using our virtual memory system.

#### **Step 1: Preparing the Python User Space (The `initrd`)**

We cannot simply copy a standard Python interpreter. We need one that is:
1.  **Statically Linked:** It must not have any dependencies on external libraries (like `libc.so`), as they don't exist in PyKOS.
2.  **Compiled for a "Bare-Metal" Target:** It should be compiled with flags that assume no underlying OS.
3.  **Minimal:** We only need the core interpreter and a small subset of the standard library.

For an educational OS, the easiest approach is to use a minimal, embeddable Python implementation like **MicroPython**, or to build a standard CPython from source with a custom cross-compiler and flags to create a static binary.

**Creating the `initrd.tar`:**

This is done on the host machine as part of the build process.

1.  **Acquire Python:** Let's assume we have a statically compiled `python` ELF binary.
2.  **Create Directory Structure:**
    ```bash
    mkdir -p initrd_contents/bin
    mkdir -p initrd_contents/lib/python3.x
    mkdir -p initrd_contents/etc

    cp /path/to/static/python initrd_contents/bin/
    cp -r /path/to/minimal/stdlib/* initrd_contents/lib/python3.x/
    ```
3.  **Write the Initial Python Script (`init.py`):**
    This will be the first user program our OS runs. It will be our future shell.

    **File:** `initrd_contents/etc/init.py`
    ```python
    # This is a placeholder for our future shell.
    # For now, it just proves we can execute Python code.
    
    # We will need a way to make syscalls. Let's imagine a C extension
    # or a ctypes-like interface provided by the kernel.
    import _pykos_syscall as syscalls

    # Syscall numbers (these must match the kernel's definitions)
    SYS_WRITE = 1
    STDOUT_FD = 1 # File descriptor for standard output

    def kprint(message):
        """A simple print function using our write syscall."""
        message_bytes = message.encode('utf-8')
        syscalls.syscall(SYS_WRITE, STDOUT_FD, message_bytes, len(message_bytes))

    kprint("--- Hello from Python User Space! ---\n")
    kprint("PyKOS Phase 3: The Python Bridge is operational.\n")
    
    # After this script finishes, the process should terminate.
    # We will need an exit syscall.
    ```
4.  **Package the `initrd`:**
    ```bash
    # In the Makefile
    tar -cf build/initrd.tar -C initrd_contents .
    ```
5.  **Update GRUB:** We tell GRUB to load this file into memory alongside our kernel.

    **File:** `iso/boot/grub/grub.cfg` (Updated)
    ```
    menuentry "PyKOS" {
        multiboot2 /boot/pykos.kernel
        module2 /boot/initrd.tar
        boot
    }
    ```
    GRUB will now pass the memory address and size of `initrd.tar` to our kernel.

#### **Step 2: Kernel Logic for Loading the `init` Process**

Our kernel needs to find, parse, and execute the Python interpreter from the `initrd`.

1.  **Parsing the `initrd`:**

    **File:** `src/kernel/initrd.cpp` (New file)
    ```cpp
    #include "initrd.hpp"
    #include "vmm.hpp" // For memory management

    // A simple (and insecure) TAR header parser
    struct TarHeader { /* ... fields for name, size, type, etc. ... */ };

    void parse_initrd(uintptr_t address) {
        TarHeader* header = (TarHeader*)address;
        while (header->name[0] != '\0') {
            size_t size = octal_to_int(header->size);
            // Store file location and size in a simple map or list
            g_initrd_files[header->name] = { (uintptr_t)header + 512, size };
            
            // Move to the next header
            address += 512 + ((size + 511) / 512) * 512;
            header = (TarHeader*)address;
        }
    }
    ```
    This function will be called early in `kmain64` with the address passed by GRUB.

2.  **The `exec` Logic:** This is the heart of the bridge. It's a complex function that will be part of our process management code.

    **File:** `src/kernel/process.cpp` (Updated `Process` class and new `exec` function)
    ```cpp
    // In Process class...
    // New constructor for user processes
    Process::Process() {
        // ... allocate PID, etc. ...
        this->address_space = new AddressSpace(); // A NEW, EMPTY address space
        this->state = ProcessState::Ready;
    }

    // High-level logic for exec
    void exec(const char* path, const char* argv[]) {
        // 1. Find the ELF binary in the initrd
        FileData elf_data = g_initrd_files[path];
        if (!elf_data.address) { /* handle error */ return; }

        // 2. Create a new process and address space
        Process* new_proc = new Process();
        
        // 3. Load the ELF into the new address space
        // This involves reading ELF headers, allocating physical pages for
        // .text and .data sections, copying the data, and mapping them
        // into the new process's virtual memory with correct permissions.
        uintptr_t entry_point = load_elf(new_proc->address_space, elf_data);

        // 4. Allocate and set up a user-mode stack
        void* user_stack_ptr = setup_user_stack(new_proc->address_space, argv);
        
        // 5. Prepare the CPU state for the jump to user mode
        new_proc->cpu_state->rip = entry_point;
        new_proc->cpu_state->rsp = (uint64_t)user_stack_ptr;
        // Set segment registers (CS, SS) to user-mode values from the GDT
        // Set RFLAGS to enable interrupts
        
        // 6. Add to scheduler
        g_scheduler.add_process(new_proc);
    }
    ```

#### **Step 3: The System Call ABI and Sandboxing**

1.  **Syscall Handler:** We need to create the bridge for our Python script.

    **File:** `src/arch/x86_64/syscall.asm` (New file)
    ```nasm
    extern syscall_dispatcher

    global syscall_entry
    syscall_entry:
        ; The syscall instruction lands here.
        ; User RIP is in RCX, RFLAGS in R11.
        ; Other registers are preserved.
        
        ; Save user context that syscall/sysret doesn't
        push rcx
        push r11
        ; ... save other registers ...
        
        mov rdi, rsp ; Pass pointer to the saved registers to C++
        call syscall_dispatcher
        
        ; ... restore registers ...
        
        sysretq ; Return to user mode
    ```
    The CPU must be configured via MSRs to jump to `syscall_entry` when `syscall` is executed.

    **File:** `src/kernel/syscall.cpp` (New file)
    ```cpp
    #include "syscall.hpp"
    // ... include headers for file I/O, process management, etc.

    // A simple lookup table for syscall handlers
    void* syscall_handlers[MAX_SYSCALLS];

    // The main dispatcher
    extern "C" void syscall_dispatcher(CPUState* regs) {
        uint64_t syscall_num = regs->rax;
        if (syscall_num < MAX_SYSCALLS && syscall_handlers[syscall_num]) {
            // Call the handler function
            using SyscallHandler = uint64_t (*)(uint64_t, uint64_t, ...);
            auto handler = (SyscallHandler)syscall_handlers[syscall_num];
            
            // Pass arguments from registers and store return value in rax
            regs->rax = handler(regs->rdi, regs->rsi, regs->rdx, ...);
        } else {
            regs->rax = -1; // Error
        }
    }

    // Example handler implementation
    uint64_t sys_write(uint64_t fd, uint64_t buffer_ptr, uint64_t count) {
        // 1. SECURITY CHECK: Verify buffer_ptr is a valid user-space address!
        // The VMM must check that the entire buffer is in user-accessible memory.
        if (!is_valid_user_pointer((void*)buffer_ptr, count)) {
            return -1;
        }
        
        // 2. Write to the console (for now)
        char* buf = (char*)buffer_ptr;
        for (uint64_t i = 0; i < count; i++) {
            g_serial_port.write(buf[i]); // Or VGA terminal
        }
        return count;
    }

    void init_syscalls() {
        // Register the sys_write handler
        syscall_handlers[SYS_WRITE] = (void*)sys_write;
        // ... register other handlers ...
    }
    ```

2.  **Sandboxing (Memory Protection):** This is the most crucial security feature.
    *   **Paging:** The kernel's `exec` logic ensures that a user process *only* has page table entries for its own code, data, and stack. It has no mappings for the kernel's private memory or for any other process.
    *   **CPU Enforcement:** If the Python process tries to access a memory address it doesn't have a mapping for, the CPU will automatically trigger a **Page Fault exception**.
    *   **Page Fault Handler:** Our ISR for Page Faults (interrupt 14) must be robust. It checks if the faulting address belongs to the currently running process. If it's a legitimate fault (e.g., stack growth, copy-on-write), it can fix it. If it's an invalid access (a segmentation fault), the handler will **terminate the user process** without harming the kernel or other processes.

#### **Step 4: Final Integration**

Our `kmain64` is now the **kernel's initialization thread**. Its final job is to start the first user-space process.

**Updated `kmain64` in `kernel.cpp`:**
```cpp
// ... all previous initializations ...

extern "C" void kmain64(MultibootInfo* mb_info) {
    // ... init terminal, IDT, PMM ...
    
    // Find and parse the initrd
    Module* initrd_mod = mb_info->find_module("/boot/initrd.tar");
    parse_initrd(initrd_mod->address);
    terminal.write_string("[OK] Initrd parsed.\n");
    
    init_syscalls();
    terminal.write_string("[OK] System calls initialized.\n");
    
    // Start the first user process!
    terminal.write_string("Executing /bin/python with /etc/init.py...\n");
    const char* argv[] = { "/etc/init.py", nullptr };
    exec("/bin/python", argv);
    
    // Initialize the timer to start multitasking
    init_timer(100);
    asm volatile("sti");

    // Kernel's main thread now becomes the idle loop
    while (true) { asm("hlt"); }
}
```

#### **Step 5: Build and Run**

After updating the `Makefile` and build scripts, running `make run` should now produce output on the QEMU serial console (as QEMU's VGA is slow and our `sys_write` targets the serial port):
```
PyKOS 64-bit Kernel - Phase 3...
[OK] Initrd parsed.
[OK] System calls initialized.
Executing /bin/python with /etc/init.py...
--- Hello from Python User Space! ---
PyKOS Phase 3: The Python Bridge is operational.
```

