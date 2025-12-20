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
