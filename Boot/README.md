### **Phase 0: Environment & Bootstrapping**

#### **Step 1: Setting Up the Development Environment**

First, we need a cross-compiler. This is a special version of GCC that runs on our host machine (e.g., Linux) but compiles code for our target machine (a bare-metal `x86_64-elf` system). We can't use our system's default compiler because it links against the host OS's libraries, which won't exist on PyKOS.

We also need an assembler (NASM), an emulator (QEMU), and tools to create a bootable ISO (GRUB and `xorriso`).

**On a Debian/Ubuntu-based system, you would run:**
```bash
sudo apt update
sudo apt install build-essential nasm xorriso qemu-system-x86 grub-pc-bin
```
*(Building a full cross-compiler toolchain is a complex topic. For this educational project, we can often get by with the system's `gcc` and careful linker flags, but a dedicated `x86_64-elf-gcc` is the professional standard.)*

#### **Step 2: Project Directory Structure**

Let's organize our project cleanly. Create a root directory (e.g., `pykos/`) and set it up as follows:

```
pykos/
├── src/
│   ├── arch/x86_64/
│   │   └── boot.asm        # The assembly bootloader stub
│   └── kernel/
│       ├── kernel.cpp      # The main C++ kernel entry point
│       └── vga.hpp         # Header for our VGA text mode driver
├── build/                      # Will hold compiled object files and the final kernel binary
└── iso/                        # Will be used to construct the bootable ISO
    └── boot/
        └── grub/
            └── grub.cfg      # GRUB configuration file
```

#### **Step 3: The Bootloader Stub (`boot.asm`)**

This is the very first piece of our code that will run. Its job is to set up the 64-bit CPU environment (Long Mode) and call our C++ kernel. We'll use the GRUB Multiboot2 specification.

**File:** `src/arch/x86_64/boot.asm`
```nasm
section .boot
align 8

; Multiboot2 Header - tells GRUB we are a compliant kernel
header_start:
    dd 0xE85250D6       ; Multiboot2 magic number
    dd 0                ; Architecture (x86)
    dd header_end - header_start ; Header length
    dd -(0xE85250D6 + 0 + (header_end - header_start)) ; Checksum

    ; Required end tag
    dw 0, 0
    dd 8
header_end:

; Set up a stack for our C++ kernel
section .bss
align 16
stack_bottom:
    resb 4096 * 4       ; Reserve 16KB for the stack
stack_top:

; The kernel's entry point
section .text
bits 32
global start
extern kmain            ; This is our C++ kernel's main function

start:
    ; GRUB has already put us in 32-bit protected mode.
    ; For simplicity in this phase, we'll stay in 32-bit mode
    ; and just call our C++ kernel. Phase 1 will handle the full 64-bit setup.
    
    ; Set up the stack
    mov esp, stack_top

    ; Push the multiboot information pointer and magic number, as required
    push eax
    push ebx
    
    ; Call our C++ kernel's main function
    call kmain

    ; If kmain ever returns, hang the system
    cli
.hang:
    hlt
    jmp .hang
```
*Note: This is a simplified 32-bit entry. A full 64-bit long mode transition is more complex and will be a key part of Phase 1.*

#### **Step 4: The C++ Kernel (`kernel.cpp` and `vga.hpp`)**

This is where the main logic begins. We'll create a simple VGA driver to print to the screen.

**File:** `src/kernel/vga.hpp`
```cpp
#pragma once
#include <stddef.h>
#include <stdint.h>

// A simple class to handle VGA text mode output
class VGATerminal {
public:
    VGATerminal();
    void put_char(char c);
    void write_string(const char* str);
    void set_color(uint8_t color);
    void clear_screen();

private:
    void scroll();

    size_t row;
    size_t column;
    uint8_t color;
    volatile uint16_t* buffer;

    static const size_t VGA_WIDTH = 80;
    static const size_t VGA_HEIGHT = 25;
};
```

**File:** `src/kernel/kernel.cpp`
```cpp
#include "vga.hpp"

// Define the VGA color attribute (light grey on black)
#define VGA_COLOR(fg, bg) (fg | bg << 4)
#define VGA_COLOR_LIGHT_GREY 7
#define VGA_COLOR_BLACK 0

// Constructor for our VGA terminal
VGATerminal::VGATerminal() : row(0), column(0) {
    color = VGA_COLOR(VGA_COLOR_LIGHT_GREY, VGA_COLOR_BLACK);
    buffer = (volatile uint16_t*)0xB8000; // The memory-mapped address for VGA text
    clear_screen();
}

void VGATerminal::clear_screen() {
    for (size_t y = 0; y < VGA_HEIGHT; y++) {
        for (size_t x = 0; x < VGA_WIDTH; x++) {
            const size_t index = y * VGA_WIDTH + x;
            buffer[index] = (uint16_t)color << 8 | ' ';
        }
    }
    row = 0;
    column = 0;
}

void VGATerminal::put_char(char c) {
    if (c == '\n') {
        column = 0;
        row++;
    } else {
        const size_t index = row * VGA_WIDTH + column;
        buffer[index] = (uint16_t)color << 8 | c;
        column++;
    }

    if (column >= VGA_WIDTH) {
        column = 0;
        row++;
    }
    
    if (row >= VGA_HEIGHT) {
        // For now, we'll just wrap around. Scrolling will be added later.
        row = 0; 
    }
}

void VGATerminal::write_string(const char* str) {
    for (size_t i = 0; str[i] != '\0'; i++) {
        put_char(str[i]);
    }
}

// The main entry point for our C++ kernel
// Must be declared 'extern "C"' to prevent C++ name mangling
extern "C" void kmain(void* multiboot_structure, unsigned int magic_number) {
    VGATerminal terminal;
    terminal.write_string("Welcome to PyKOS!\n");
    terminal.write_string("Phase 0: Bootstrapping complete.");

    // The assembly code will hang the system after this function returns.
}
```

#### **Step 5: The Linker Script and GRUB Config**

These files tell the build tools how to assemble our final kernel binary and how GRUB should load it.

**File:** `linker.ld`
```ld
ENTRY(start)

SECTIONS {
    /* Load the kernel at 1MB */
    . = 1M;
    
    .text : {
        *(.boot)
        *(.text)
    }
    
    .rodata : {
        *(.rodata)
    }
    
    .data : {
        *(.data)
    }
    
    .bss : {
        *(.bss)
    }
}
```

**File:** `iso/boot/grub/grub.cfg`
```
set timeout=0
set default=0

menuentry "PyKOS" {
    multiboot2 /boot/pykos.kernel
    boot
}
```

#### **Step 6: The Build Process (`Makefile`)**

This Makefile automates the entire compilation, linking, and ISO creation process.

**File:** `Makefile`
```makefile
# Define the cross-compiler prefix
# If you built a full cross-compiler, this would be 'x86_64-elf-'
CROSS_PREFIX ?= 

# Compiler and Assembler
AS = nasm
CC = $(CROSS_PREFIX)g++
LD = $(CROSS_PREFIX)ld

# Compiler flags
CFLAGS = -ffreestanding -O2 -Wall -Wextra -fno-exceptions -fno-rtti -std=c++17 -c
ASFLAGS = -f elf64

# Project structure
SRC_DIR = src
BUILD_DIR = build
ISO_DIR = iso
KERNEL = $(BUILD_DIR)/pykos.kernel

# Source files
ASM_SOURCES = $(wildcard $(SRC_DIR)/arch/x86_64/*.asm)
C_SOURCES = $(wildcard $(SRC_DIR)/kernel/*.cpp)
OBJECTS = $(patsubst $(SRC_DIR)/%, $(BUILD_DIR)/%, $(ASM_SOURCES:.asm=.o)) \
          $(patsubst $(SRC_DIR)/%, $(BUILD_DIR)/%, $(C_SOURCES:.cpp=.o))

.PHONY: all clean run iso

all: $(KERNEL)

# Rule to compile C++ files
$(BUILD_DIR)/kernel/%.o: $(SRC_DIR)/kernel/%.cpp
	@mkdir -p $(dir $@)
	$(CC) $(CFLAGS) $< -o $@

# Rule to assemble ASM files
$(BUILD_DIR)/arch/x86_64/%.o: $(SRC_DIR)/arch/x86_64/%.asm
	@mkdir -p $(dir $@)
	$(AS) $(ASFLAGS) $< -o $@

# Rule to link the kernel
$(KERNEL): $(OBJECTS) linker.ld
	$(LD) -n -T linker.ld -o $(KERNEL) $(OBJECTS)

# Rule to create the bootable ISO
iso: $(KERNEL)
	@mkdir -p $(ISO_DIR)/boot
	cp $(KERNEL) $(ISO_DIR)/boot/pykos.kernel
	grub-mkrescue -o pykos.iso $(ISO_DIR)

# Rule to run in QEMU
run: iso
	qemu-system-x86_64 -cdrom pykos.iso

# Rule to clean up build files
clean:
	rm -rf $(BUILD_DIR) pykos.iso
```
*Note: You might need to adjust the `ASFLAGS` to `-f elf32` if your linker has trouble with the mixed 32/64-bit objects initially.*

#### **Step 7: Build and Run!**

Now, from your `pykos/` root directory, you can simply run:

```bash
make run
```

If everything is set up correctly, QEMU will launch, boot from the generated `pykos.iso`, and you will see a black screen with the following text:

```
Welcome to PyKOS!
Phase 0: Bootstrapping complete.
```


### **Phase 1: Core Kernel Services (Memory & Interrupts)**

#### **Objective:**
To take full control of the CPU and memory. We will replace the temporary 32-bit environment with a proper 64-bit Long Mode setup, enable interrupts to respond to hardware and software events, and implement a virtual memory system using paging to isolate the kernel from future user-space programs.

#### **Step 1: The Transition to 64-bit Long Mode**

Our `boot.asm` was a simplification. A proper x86-64 OS must transition the CPU to 64-bit Long Mode. This involves setting up new page tables.

**File:** `src/arch/x86_64/long_mode_init.asm` (New file)
```nasm
section .bootstrap_page_tables
align 4096
pml4_table:
    ; Map the first 1GB of physical memory to the same virtual address
    dq pml3_table + 0x3 ; Present, Writeable, Supervisor
    times 511 dq 0
pml3_table:
    ; Each entry here maps 1GB. We'll map 1GB.
    dq pml2_table + 0x3 ; Present, Writeable, Supervisor
    times 511 dq 0
pml2_table:
    ; Each entry here maps 2MB. We'll use 2MB pages for simplicity.
    ; This maps the first 1GB of physical memory.
    %assign i 0
    %rep 512
        dq (i * 0x200000) + 0x83 ; Present, Writeable, Supervisor, 2MB page
    %assign i i+1
    %endrep

section .text
bits 32
global start64
extern kmain64 ; Our new 64-bit kernel entry point

start64:
    ; 1. Load our new page table directory
    mov eax, pml4_table
    mov cr3, eax

    ; 2. Enable PAE (Physical Address Extension)
    mov eax, cr4
    or eax, 1 << 5
    mov cr4, eax

    ; 3. Enable Long Mode
    mov ecx, 0xC0000080 ; EFER MSR
    rdmsr
    or eax, 1 << 8     ; Set LME bit
    wrmsr

    ; 4. Enable Paging
    mov eax, cr0
    or eax, 1 << 31    ; Set PG bit
    mov cr0, eax

    ; 5. Load a 64-bit GDT (Global Descriptor Table)
    lgdt [gdt64_pointer]

    ; 6. Jump to our 64-bit code segment
    jmp gdt64_code_segment:.long_mode

bits 64
.long_mode:
    ; We are now in 64-bit Long Mode!
    ; Set up segment registers
    mov ax, 0
    mov ss, ax
    mov ds, ax
    mov es, ax
    
    ; Set up the stack
    mov rsp, stack_top
    
    ; Clear SSE registers just in case
    xorps xmm0, xmm0 

    ; Call our 64-bit C++ kernel
    call kmain64

    ; Hang if it returns
    cli
.hang:
    hlt
    jmp .hang

; 64-bit GDT definition
gdt64:
    dq 0 ; Null segment
gdt64_code_segment: equ $ - gdt64
    dq 0x00209A0000000000 ; 64-bit code segment
gdt64_data_segment: equ $ - gdt64
    dq 0x0000920000000000 ; 64-bit data segment
gdt64_end:

gdt64_pointer:
    dw gdt64_end - gdt64 - 1
    dq gdt64
```
*We will need to update our `boot.asm` to call `start64` instead of `kmain` directly, and our linker script will need to accommodate the 64-bit code.*

#### **Step 2: Interrupt Handling (GDT, IDT, and ISRs)**

This is the nervous system of the OS. The CPU needs a way to tell us when something happens (a key is pressed, a timer ticks, an error occurs).

1.  **Global Descriptor Table (GDT):** We've defined a minimal one in assembly. A C++ version will allow us to add more segments later (e.g., for user mode).

2.  **Interrupt Descriptor Table (IDT):** This is a table of 256 entries. Each entry points to a handler for a specific interrupt.

**File:** `src/kernel/idt.hpp` (New file)
```cpp
#pragma once
#include <stdint.h>

// Defines an entry in the IDT
struct IDTEntry {
    uint16_t base_low;
    uint16_t selector;    // Kernel code segment selector
    uint8_t  ist;         // Interrupt Stack Table
    uint8_t  flags;       // Type and attributes
    uint16_t base_mid;
    uint32_t base_high;
    uint32_t reserved;
} __attribute__((packed));

// A pointer structure for the 'lidt' instruction
struct IDTPointer {
    uint16_t limit;
    uint64_t base;
} __attribute__((packed));

void init_idt();
```

The C++ code in `idt.cpp` will create an array of 256 `IDTEntry` structs and a function `load_idt()` which uses an inline assembly `lidt` instruction to tell the CPU where our IDT is.

3.  **Interrupt Service Routines (ISRs):** These are the functions that actually *handle* the interrupts.

**File:** `src/arch/x86_64/isrs.asm` (New file)
```nasm
extern isr_handler ; A C++ function we will call

; Macro to create a stub for each of the first 32 CPU exceptions
%macro ISR_NOERR 1
global isr%1
isr%1:
    cli
    push 0      ; Push a dummy error code
    push %1     ; Push the interrupt number
    jmp isr_common_stub
%endmacro

; Common stub that saves registers and calls C++
isr_common_stub:
    push rax
    push rbx
    ; ... push all other general-purpose registers ...
    
    mov rdi, rsp ; The first argument to isr_handler is a pointer to the stack
    call isr_handler
    
    ; ... pop all registers ...
    
    add rsp, 16 ; Clean up interrupt number and error code
    sti
    iretq       ; Return from interrupt
```
We would then use this macro to generate handlers for `isr0`, `isr1`, etc. The C++ `isr_handler` function will receive a pointer to the saved registers and can then decide what to do based on the interrupt number.

**Example `isr_handler` in C++:**
```cpp
// In a new file, e.g., src/kernel/interrupts.cpp
#include "vga.hpp"

extern "C" void isr_handler(Registers* regs) {
    VGATerminal term;
    term.write_string("Caught Interrupt: ");
    // Convert regs->int_no to a string and print it
    
    // For now, hang on any exception
    asm("cli; hlt");
}
```

#### **Step 3: Memory Management**

Now we build the core of our memory system.

1.  **Physical Page Frame Allocator:** We need a way to track which 4KB pages of physical RAM are free or used. A bitmap is a simple and effective way to do this.

**File:** `src/kernel/pmm.hpp` (Physical Memory Manager)
```cpp
#pragma once
#include <stdint.h>
#include <stddef.h>

class PageFrameAllocator {
public:
    void init(MemoryMap* mem_map); // Takes memory map from bootloader
    void* alloc_page();
    void free_page(void* page);
private:
    uint8_t* bitmap;
    size_t total_pages;
    size_t last_alloc_index;
};
```
The `init` function will iterate through the memory map provided by GRUB, find the largest available RAM segment, and place its bitmap there. `alloc_page` finds the first clear bit, sets it, and returns the corresponding physical address.

2.  **Virtual Memory Manager (VMM):** This manages the page tables that translate virtual addresses to physical addresses.

**File:** `src/kernel/vmm.hpp`
```cpp
#pragma once
#include <stdint.h>

// A virtual address space (PML4 table)
using PageMapLevel4 = uint64_t[512];

class AddressSpace {
public:
    AddressSpace(); // Creates a new, empty address space
    void map_page(void* virt, void* phys, uint64_t flags);
    void* get_phys_addr(void* virt);
    PageMapLevel4* get_pml4();
private:
    PageMapLevel4* pml4;
};

void switch_address_space(AddressSpace* space);
```
The `map_page` function will be the workhorse. It will navigate the four levels of page tables (PML4, PDPT, PD, PT), allocating new tables from our `PageFrameAllocator` as needed, until it can set the final entry in the Page Table (PT) to map a virtual page to a physical one.

**Integration into Kernel Boot:**

Our new `kmain64` function will look something like this:
```cpp
// In kernel.cpp
#include "idt.hpp"
#include "pmm.hpp"
#include "vmm.hpp"

PageFrameAllocator g_page_allocator;
AddressSpace g_kernel_space;

extern "C" void kmain64(MultibootInfo* mb_info) {
    VGATerminal terminal;
    terminal.write_string("PyKOS 64-bit Kernel Initializing...\n");

    // 1. Initialize Interrupts
    init_idt();
    terminal.write_string("[OK] IDT Initialized.\n");
    
    // 2. Initialize Physical Memory Manager
    g_page_allocator.init(mb_info->get_memory_map());
    terminal.write_string("[OK] PMM Initialized.\n");

    // 3. Set up the Kernel's own higher-half address space
    // Here we'd create a new AddressSpace, map the kernel's code/data
    // to a high virtual address (e.g., 0xFFFFFFFF80000000), and map
    // all physical memory to another region for easy access.
    // This is a complex step involving careful mapping.
    // For now, we assume we're running in the boot page tables.
    
    terminal.write_string("Triggering a test interrupt (Divide by Zero)...\n");
    asm volatile("div %0" :: "r"(0)); // This should trigger ISR 0

    // We should never get here.
    terminal.write_string("ERROR: System did not halt on exception!");
}
```

#### **Step 4: Build and Run**

We need to update our `Makefile` to compile our new 64-bit source files and link them correctly. The linker script (`linker.ld`) will also need to be updated to handle 64-bit addresses and place the kernel in a "higher half" (a high memory address).

After running `make run`, you should see:
```
Welcome to PyKOS 64-bit Kernel Initializing...
[OK] IDT Initialized.
[OK] PMM Initialized.
Triggering a test interrupt (Divide by Zero)...
Caught Interrupt: 0
```
...and then the system will hang inside our `isr_handler`, as intended.

---

### **Phase 2: Multitasking & Scheduling**

#### **Objective:**
To implement a preemptive multitasking scheduler. This involves creating a data structure to represent a process, building a scheduler to switch between processes, and using a hardware timer to interrupt the CPU periodically, forcing context switches.

#### **Step 1: The Process Control Block (PCB) and Kernel Threads**

First, we need a way to represent a "task" or "process." In a UNIX-like OS, this is the Process Control Block (PCB). For PyKOS, we'll create a C++ class for this. Initially, we'll implement *kernel threads*—threads of execution that run exclusively in Ring 0.

**File:** `src/kernel/process.hpp` (New file)
```cpp
#pragma once
#include <stdint.h>
#include "vmm.hpp" // Include our Virtual Memory Manager header

// This structure defines the registers we need to save for a context switch.
// Order is important for our assembly context switch routine.
struct [[gnu::packed]] CPUState {
    uint64_t rax, rbx, rcx, rdx;
    uint64_t rsi, rdi, rbp;
    uint64_t r8, r9, r10, r11, r12, r13, r14, r15;
    uint64_t rip; // Instruction pointer
    uint64_t rflags;
    uint64_t rsp; // Stack pointer
};

enum class ProcessState {
    Ready,
    Running,
    Blocked,
    Terminated
};

class Process {
public:
    // Constructor for a new kernel thread
    Process(void* entry_point);
    
    // Destructor to clean up resources
    ~Process();

    // The saved state of the CPU when this process is not running
    CPUState* cpu_state;

    // The kernel stack for this process
    void* kernel_stack;

    // The virtual address space for this process
    AddressSpace* address_space;

    // The current state of the process
    ProcessState state;
    
    // Process ID
    uint64_t pid;
    
    // Pointer for scheduler's linked list
    Process* next; 

private:
    static uint64_t next_pid;
};
```

**Implementation in `process.cpp`:**
The constructor for `Process` will be crucial. It needs to:
1.  Allocate a new kernel stack for the thread (using our `PageFrameAllocator`).
2.  Set up an initial `CPUState` on this new stack. This is a "fake" context that makes it look like the thread was interrupted just before it started.
3.  The `rip` (instruction pointer) in this fake state is set to the thread's entry point function.
4.  The `rsp` (stack pointer) is set to the top of the newly allocated kernel stack.
5.  Set `rflags` to enable interrupts (`0x202`).
6.  Assign a unique PID.

#### **Step 2: The Scheduler**

We'll implement a simple round-robin scheduler. It will maintain a linked list of all `Ready` processes and switch to the next one on each timer tick.

**File:** `src/kernel/scheduler.hpp` (New file)
```cpp
#pragma once
#include "process.hpp"

class Scheduler {
public:
    Scheduler();
    void add_process(Process* proc);
    void schedule(CPUState* current_cpu_state); // The main scheduling function

    static Process* current_process;

private:
    Process* ready_queue_head;
};

// This is the function called by the timer interrupt handler.
extern "C" void schedule(CPUState* cpu_state);
```

**File:** `src/kernel/scheduler.cpp`
```cpp
#include "scheduler.hpp"

Scheduler g_scheduler;
Process* Scheduler::current_process = nullptr;

// ... constructor and add_process ...

void Scheduler::schedule(CPUState* current_cpu_state) {
    if (current_process) {
        // Save the state of the currently running process
        current_process->cpu_state = current_cpu_state;
        current_process->state = ProcessState::Ready;
        // Add it to the end of the ready queue (simplified)
        add_process(current_process); 
    }

    // Get the next process from the front of the ready queue
    Process* next_process = ready_queue_head;
    if (!next_process) {
        // No other processes to run, just return.
        // This happens if we only have one thread that went to sleep.
        // A real OS would switch to an idle thread here.
        return;
    }
    
    ready_queue_head = next_process->next;
    next_process->next = nullptr; // De-link it

    // Switch to the next process
    current_process = next_process;
    current_process->state = ProcessState::Running;

    // The magic happens here: context_switch will load the new process's
    // cpu_state and will NOT return to this function. It will return
    // to wherever the new process was interrupted.
    context_switch(current_process->cpu_state);
}

// C-style function to be callable from assembly
extern "C" void schedule(CPUState* cpu_state) {
    g_scheduler.schedule(cpu_state);
}
```

#### **Step 3: The Context Switch and Timer Interrupt**

The heart of multitasking is the **context switch**. This low-level routine, written in assembly, saves the current process's state and loads the next one's. We also need to set up a hardware timer to trigger the scheduler.

1.  **Context Switch Assembly:**

**File:** `src/arch/x86_64/context.asm` (New file)
```nasm
global context_switch
extern Scheduler_current_process ; We need to access the C++ current_process pointer

section .text
context_switch:
    ; The argument (in RDI) is a pointer to the *next* process's CPUState.
    ; The *current* process's state has already been saved on the stack
    ; by the interrupt handler.
    
    mov [Scheduler_current_process], rdi ; Update current_process pointer in C++
    
    ; Load the new process's stack pointer from the saved state
    mov rsp, [rdi + 16*8] ; RSP is the last member of CPUState

    ; Pop all the registers for the new process
    pop r15
    ; ... pop all other registers in reverse order of how they were pushed ...
    pop rax

    add rsp, 16 ; Skip over the interrupt number and error code
    sti         ; Re-enable interrupts
    iretq       ; Return from interrupt, loading new RIP, RFLAGS, RSP
```

2.  **Timer Interrupt:** We need to program a hardware timer (like the PIT) to fire regularly (e.g., 100 times per second).

**File:** `src/kernel/timer.cpp` (New file)
```cpp
#include "idt.hpp"
#include "scheduler.hpp"

// This is the C++ handler for the timer interrupt (IRQ0)
extern "C" void timer_handler(CPUState* cpu_state) {
    // Send End-of-Interrupt signal to the PIC
    outb(0x20, 0x20); 

    // Call the scheduler
    schedule(cpu_state);
}

void init_timer(uint32_t frequency) {
    // ... code to program the PIT to fire at the desired frequency ...
    
    // Register our handler for IRQ0 (the timer interrupt)
    register_interrupt_handler(32, timer_handler);
}
```
*We also need to update our `isrs.asm` to handle IRQs (Interrupt Requests from hardware), which is slightly different from handling CPU exceptions.* The IRQ handlers will need to call their respective C++ handlers, like `timer_handler`.

#### **Step 4: Integration into the Kernel**

Now we put all the pieces together in our main kernel function.

**Updated `kmain64` in `kernel.cpp`:**
```cpp
// ... includes for timer, process, scheduler ...

void task_a_func() {
    VGATerminal term;
    while (true) {
        term.put_char('A');
        // Simple delay loop
        for (volatile int i = 0; i < 15000000; i++);
    }
}

void task_b_func() {
    VGATerminal term;
    while (true) {
        term.put_char('B');
        // Simple delay loop
        for (volatile int i = 0; i < 15000000; i++);
    }
}

extern "C" void kmain64(MultibootInfo* mb_info) {
    VGATerminal terminal;
    terminal.write_string("PyKOS 64-bit Kernel - Phase 2: Multitasking\n");

    init_idt();
    // ... init PMM ...
    
    terminal.write_string("[OK] Kernel core services initialized.\n");

    // Create our first two kernel threads
    Process* task_a = new Process((void*)task_a_func);
    Process* task_b = new Process((void*)task_b_func);

    g_scheduler.add_process(task_a);
    g_scheduler.add_process(task_b);
    
    terminal.write_string("[OK] Processes created and added to scheduler.\n");

    // Initialize the timer, which will start the scheduling
    init_timer(100); // 100 Hz
    
    terminal.write_string("[OK] Timer initialized. Starting multitasking...\n");
    
    // Enable interrupts to start the timer ticks
    asm volatile("sti");
    
    // We are now the "idle" thread. If there are no other tasks,
    // the scheduler will return here. We just hang.
    while (true) {
        asm("hlt");
    }
}
```

#### **Step 5: Build and Run**

Update the `Makefile` to include the new source files (`process.cpp`, `scheduler.cpp`, `timer.cpp`, `context.asm`, etc.). Then, run:

```bash
make run
```

If everything is correct, you will see the initial boot messages, and then you will see a rapid stream of `A` and `B` characters being printed to the screen, interleaved. This demonstrates that the two tasks are running concurrently, with the timer interrupt forcing the CPU to switch between them.



### **Phase 3: The Python Bridge (User Mode & Syscalls)**

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

