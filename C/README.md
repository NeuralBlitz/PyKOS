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



