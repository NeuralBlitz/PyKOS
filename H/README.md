
---

### **Phase 8: Networking Stack**

#### **Objective:**
To connect PyKOS to the outside world. This is a highly complex undertaking, so we will start with a minimal implementation of the TCP/IP stack to support basic network communication.

#### **Step 1: The Network Device Driver**

We need a driver for a virtual network card. The **AMD PCnet AM79C973** (PCnet-PCI II) is an excellent choice as it's well-documented and widely supported by emulators like QEMU.

*   **Design: `pcnet` Driver:**
    1.  **PCI Bus Enumeration:** The kernel needs a simple PCI bus scanner to detect devices. It will iterate through the PCI configuration space to find a device with the correct Vendor and Device ID for the PCnet card.
    2.  **Driver Initialization:** The driver will claim the device's I/O ports and memory-mapped regions, enable bus mastering, and set up circular buffers (receive and transmit rings) in memory for DMA (Direct Memory Access).
    3.  **Interrupt Handling:** The driver will register an IRQ handler. When the network card receives a packet, it will trigger an interrupt. The handler will process the receive ring and pass the packet data up to the network stack.

#### **Step 2: The Network Stack (Layers 2-4)**

We will implement a minimal version of the core networking protocols in the C++ kernel.

*   **Design: Layered Network Stack:**
    1.  **Layer 2 (Data Link):**
        *   **ARP (Address Resolution Protocol):** Implement ARP to resolve IP addresses to MAC addresses on the local network. The kernel will maintain an ARP cache.
        *   **Ethernet Frame Handling:** Code to construct and parse Ethernet II frames.
    2.  **Layer 3 (Network):**
        *   **IP (Internet Protocol):** Implement IP packet handling, including header creation, checksum calculation, and basic routing (e.g., sending to a gateway if the destination is not local).
        *   **ICMP (Internet Control Message Protocol):** Implement handlers for ICMP echo requests ("ping") to test network connectivity.
    3.  **Layer 4 (Transport):**
        *   **UDP (User Datagram Protocol):** A simple, connectionless protocol. This is the easiest to implement first.
        *   **TCP (Transmission Control Protocol):** A much more complex, stateful protocol. Implementing TCP involves managing connection states (SYN, ACK, FIN), sequence numbers, windowing for flow control, and retransmission timers.

#### **Step 3: The Socket API**

This is the VFS-like abstraction that exposes networking to the Python user space.

*   **Design: Socket Syscalls:**
    1.  **`socket()` Syscall:** Creates a new communication endpoint. The kernel creates a `Socket` object in memory and returns a file descriptor for it.
    2.  **`bind()`, `listen()`, `accept()` Syscalls:** For server-side operations.
    3.  **`connect()` Syscall:** For client-side operations.
    4.  **`send()`/`recv()` Syscalls:** These are similar to `write`/`read` but operate on socket file descriptors. The kernel will route the data through the TCP/IP stack and out the network driver.
*   **Impact:** Our Python user space can now use a familiar `socket` library (a custom `_pykos_socket.py` wrapper around the new syscalls) to create network clients and servers. We could write a simple web server or an IRC client in Python!

---

### **Phase 9: A Simple Graphical User Interface (GUI)**

#### **Objective:**
To move beyond VGA text mode and implement a basic graphical environment. We will not build a full-fledged windowing system like X11, but a simpler framebuffer-based GUI.

#### **Step 1: The Framebuffer Driver**

Modern bootloaders like GRUB can set up a linear framebuffer (LFB) for the kernel, which is a simple, flat region of memory that directly corresponds to the pixels on the screen.

*   **Design: `lfb` Driver:**
    1.  **Multiboot Information:** The kernel will query the Multiboot2 info structure provided by GRUB to get the address, resolution (width/height), and pixel format of the framebuffer.
    2.  **Graphics Context:** The kernel will create a `GraphicsContext` class that provides high-level drawing primitives, such as `draw_pixel(x, y, color)`, `draw_rect(...)`, and `draw_char(...)` (using a bitmapped font).

#### **Step 2: A Minimal Window Manager & Compositor**

This kernel module will manage "windows," which will be simple rectangular regions of the screen.

*   **Design: `WindowManager`:**
    1.  **Window Struct:** A `Window` struct will store its position, size, and a pointer to its own off-screen buffer.
    2.  **Compositing:** The `WindowManager` will periodically "compose" the final screen image by copying the contents of each window's buffer to the main framebuffer. This prevents applications from drawing over each other.
    3.  **Event Handling:** The keyboard and (a new) mouse driver will send their input events to the `WindowManager`. It will determine which window is "in focus" and route the events to the corresponding process.

#### **Step 3: The GUI Syscall Interface**

*   **Design: GUI Syscalls:**
    1.  **`create_window()`:** A process can request a new window. The kernel returns a file descriptor for it.
    2.  **`get_window_buffer()`:** Maps the window's off-screen buffer into the process's virtual address space, allowing the user-space program to draw directly into it.
    3.  **`poll_event()`:** A blocking syscall that waits for a GUI event (like a key press or mouse click) to be delivered to the process.
*   **Impact:** We can now write simple graphical applications in Python! A `py_tkinter.py` library could be created to wrap these syscalls, allowing a user to write a basic text editor or a simple game.

---

### **Phase 10: Expanding Hardware Support**

#### **Objective:**
To make PyKOS run on more than just the most basic emulated hardware.

*   **Design:**
    1.  **USB Driver Stack:** This is a major undertaking. It involves writing a host controller driver (e.g., for UHCI or EHCI), a USB hub driver, and then class drivers for specific devices like USB keyboards, mice, and mass storage devices.
    2.  **AHCI SATA Driver:** To support modern SATA hard disks, replacing the legacy ATA PATA driver.
    3.  **APIC & SMP Support:** Move from the old PIC to the modern APIC (Advanced Programmable Interrupt Controller) to support multiple CPU cores (Symmetric Multiprocessing). This requires significant changes to the scheduler to make it SMP-safe (using locks) and to distribute processes across cores.

This roadmap outlines a path from a functional command-line OS to a basic graphical, networked system. Each phase introduces fundamental OS concepts and presents significant but achievable engineering challenges, perfectly aligning with the project's educational philosophy.
