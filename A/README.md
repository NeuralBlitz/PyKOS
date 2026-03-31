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



