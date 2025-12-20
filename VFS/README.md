## **Phase 4: I/O and Device Abstraction (VFS)**

#### **Objective:**
To create a unified, abstract interface for all I/O operations, modeled after the UNIX Virtual File System (VFS). This will allow our Python user space to interact with hardware devices (like the keyboard and console) using the same standard `open`, `read`, and `write` syscalls it will later use for files. We will implement a `devfs` to expose these devices as files.

#### **Step 1: Refactoring Device Drivers**

Our existing, ad-hoc code for the serial port and keyboard needs to be formalized into a consistent driver model that inherits from a common VFS interface.

**File:** `src/fs/vfs.hpp` (New file - defines the core abstractions)
```cpp
#pragma once
#include <stddef.h>
#include <stdint.h>

// Represents any "file" or "device" in the system.
// This is an abstract base class.
class VFSNode {
public:
    enum class NodeType { File, Directory, Device };

    VFSNode(const char* name, NodeType type);
    virtual ~VFSNode() {}

    // Core file operations
    virtual int read(void* buffer, size_t size, size_t offset) = 0;
    virtual int write(const void* buffer, size_t size, size_t offset) = 0;

    // Directory operations (to be implemented later)
    // virtual VFSNode* find_child(const char* name) { return nullptr; }

    const char* name;
    NodeType type;
};

// Represents an open file handle
struct FileDescriptor {
    VFSNode* node;
    size_t offset;
    // ... flags like read/write permissions ...
};
```

**File:** `src/drivers/tty.hpp` (Updated for the new model)
```cpp
#pragma once
#include "fs/vfs.hpp"
#include "drivers/serial.hpp"

// Our TTY driver now inherits from VFSNode
class TTYDriver : public VFSNode {
public:
    TTYDriver();
    int read(void* buffer, size_t size, size_t offset) override;
    int write(const void* buffer, size_t size, size_t offset) override;

private:
    SerialPort serial;
};
```
The `TTYDriver::write` method will simply loop through the buffer and call `serial.write_char()`. The `read` method will be more complex, likely blocking the calling process until a character is received from the serial port.

#### **Step 2: Implementing `devfs`**

`devfs` isn't a real filesystem on a disk; it's a virtual, in-memory tree of `VFSNode` objects that represent devices.

**File:** `src/fs/devfs.cpp` (New file)
```cpp
#include "vfs.hpp"
#include "drivers/tty.hpp"
#include "drivers/keyboard.hpp"
#include <map> // For simplicity, a map can represent the directory structure

// A global map to hold our device files
std::map<const char*, VFSNode*> devfs_nodes;

void init_devfs() {
    // Create and register our device nodes
    devfs_nodes["/dev/tty"] = new TTYDriver();
    devfs_nodes["/dev/kbd"] = new KeyboardDriver();
    
    // ... we could also have /dev/null, /dev/zero, etc.
}

VFSNode* devfs_find(const char* path) {
    if (devfs_nodes.count(path)) {
        return devfs_nodes[path];
    }
    return nullptr;
}
```
`init_devfs()` will be called during kernel initialization.

#### **Step 3: Implementing the File-Related Syscalls**

These new syscalls are the interface between our Python user space and the VFS.

**File:** `src/kernel/syscall.cpp` (Updated)
```cpp
// ... previous syscalls ...

// int open(const char* path, int flags);
uint64_t sys_open(uint64_t path_ptr, uint64_t flags) {
    const char* path = (const char*)path_ptr;
    // SECURITY: Verify user-space pointer 'path'
    
    // For now, only search devfs
    VFSNode* node = devfs_find(path);
    if (!node) {
        return -1; // File not found
    }

    // Get the current process
    Process* current = Scheduler::current_process;
    
    // Find a free file descriptor
    int fd_num = find_free_fd(current);
    if (fd_num < 0) {
        return -1; // No free FDs
    }

    // Create the file descriptor object
    current->fds[fd_num] = { node, 0 /* offset */, flags };
    return fd_num;
}

// int read(int fd, void* buf, size_t count);
uint64_t sys_read(uint64_t fd, uint64_t buf_ptr, uint64_t count) {
    Process* current = Scheduler::current_process;
    // SECURITY: Check fd is valid and buf_ptr is a valid user pointer
    
    FileDescriptor& desc = current->fds[fd];
    int bytes_read = desc.node->read((void*)buf_ptr, count, desc.offset);
    
    if (bytes_read > 0) {
        desc.offset += bytes_read;
    }
    return bytes_read;
}

// ... sys_write and sys_close would be implemented similarly ...
```

Now, Python scripts can open `/dev/tty` and write to it, which will appear on our serial console.

---

### **Phase 5: Persistent Storage (Filesystem)**

#### **Objective:**
To create a simple, inode-based filesystem named **PyFS** and a driver to read/write it from a virtual hard disk. This will allow data to persist across reboots.

#### **Step 1: The Block Device Driver (ATA PATA)**

We need to communicate with the hard disk controller. For QEMU's default `ide-hd`, this is a PATA (IDE) controller.

**File:** `src/drivers/ata.hpp` (New file)
```cpp
#pragma once
#include <stdint.h>

class ATADriver {
public:
    ATADriver();
    // Read 'sector_count' sectors starting from LBA 'sector' into 'buffer'
    void read_sectors(uint64_t sector, uint8_t sector_count, void* buffer);
    // Write 'sector_count' sectors
    void write_sectors(uint64_t sector, uint8_t sector_count, const void* buffer);
};
```
The implementation in `ata.cpp` will be complex, involving I/O port communication with the IDE controller, polling for status bits, and handling data transfers. This is classic, low-level device driver programming. For simplicity, we'll assume a single primary master drive.

#### **Step 2: The PyFS Filesystem Implementation**

This module contains the logic for interpreting the on-disk structures we designed.

**File:** `src/fs/pyfs.hpp` (New file)
```cpp
#pragma once
#include "vfs.hpp"
#include "drivers/ata.hpp"

// On-disk structures (matching our design blueprint)
struct PyFS_Superblock { /* ... */ };
struct PyFS_Inode { /* ... */ };
struct PyFS_DirEntry { /* ... */ };

class PyFS : public VFSNode { // A filesystem is itself a VFS node (the root dir)
public:
    PyFS(ATADriver* device);
    VFSNode* find_child(const char* name); // Main function for path traversal
    // ... functions to create files/dirs, etc.

private:
    ATADriver* device;
    PyFS_Superblock superblock;
    // ... caching for bitmaps and inodes ...
};
```

**Core Logic in `pyfs.cpp`:**
*   **`mount()`:** The kernel will call a `PyFS::mount` function. This function reads Block 0 into the `superblock` struct, checks the magic number, and loads the root inode (usually inode #2). It returns a `PyFS*` object representing the root of the filesystem.
*   **Path Traversal:** When a syscall like `open("/home/user/file.txt")` is made, the VFS will start at the PyFS root object.
    1.  It calls `pyfs_root->find_child("home")`.
    2.  `find_child` reads the data blocks of the root inode, which contain directory entries. It finds the "home" entry and gets its inode number.
    3.  It loads the "home" inode and calls `find_child("user")` on it.
    4.  This process repeats until `file.txt` is found, and a `VFSNode` representing that file is returned.

#### **Step 3: Integration with VFS**

The final step is to make the VFS aware of our new filesystem.

**Updated `sys_open` logic in `syscall.cpp`:**
```cpp
uint64_t sys_open(uint64_t path_ptr, uint64_t flags) {
    const char* path = (const char*)path_ptr;
    // ... security checks ...
    
    VFSNode* node = nullptr;
    if (strncmp(path, "/dev/", 5) == 0) {
        node = devfs_find(path);
    } else {
        // Path is in the root filesystem
        node = g_root_fs->find_path(path); // A new VFS helper function
    }

    if (!node) {
        // Here we could add logic to create the file if O_CREAT flag is set
        return -1; // Not found
    }

    // ... create file descriptor as before ...
}
```
And in `kmain64`, after initializing drivers:
```cpp
// in kmain64
ATADriver* ata_primary = new ATADriver();
PyFS* root_fs = new PyFS(ata_primary);
g_vfs->mount("/", root_fs); // Mount our new filesystem at the root
terminal.write_string("[OK] PyFS mounted as root.\n");
```

#### **Step 4: Build and Run**

We need a way to create a formatted virtual disk image. A simple host-side tool (`mkpyfs`) can be written to create a blank disk image and write the PyFS superblock and root directory structure to it.

`qemu-system-x86_64 -cdrom pykos.iso -hda pykos.img`

After this, our Python `init.py` script can start using file I/O on a persistent disk:
```python
# in init.py
kprint("Attempting to create a file...\n")
try:
    fd = syscalls.syscall(SYS_OPEN, "/hello.txt", O_CREAT | O_RDWR)
    if fd >= 0:
        message = "Persistence is working!"
        syscalls.syscall(SYS_WRITE, fd, message.encode(), len(message))
        syscalls.syscall(SYS_CLOSE, fd)
        kprint("File '/hello.txt' created and written to.\n")
except Exception as e:
    kprint(f"File operation failed: {e}\n")
```
If you reboot PyKOS and run another script to read `/hello.txt`, the message will still be there.
