# Memory-Mapped Files and mmap

Everything we've discussed in this section comes together in `mmap()` — one of the most versatile system calls in Unix. It maps files (or anonymous memory) directly into a process's virtual address space. Instead of reading and writing files with `read()` and `write()` system calls, you access the file through regular memory loads and stores. The OS handles the rest through the page fault mechanism we've already studied.

## How mmap() Works

When you call `mmap()` on a file, the kernel sets up page table entries for a region of the process's virtual address space, pointing to the file's contents. The pages aren't loaded immediately — they're marked as "not present." When the process first accesses a mapped address, a page fault occurs, and the kernel loads the corresponding file block from disk into a physical frame.

```
  Process calls: mmap(NULL, 16384, PROT_READ, MAP_PRIVATE, fd, 0)
  
  "Map 16 KB of this file into my address space, read-only, private copy"

  Before access:
  +----------------------------------+
  | Virtual Address Space            |
  | ...                              |
  | 0x7F00_0000: [not present] pg 0  |  <-- page table entries exist
  | 0x7F00_1000: [not present] pg 1  |      but pages NOT loaded
  | 0x7F00_2000: [not present] pg 2  |
  | 0x7F00_3000: [not present] pg 3  |
  | ...                              |
  +----------------------------------+

  Process reads 0x7F00_1050:
  --> Page fault! Kernel loads file offset 0x1000-0x1FFF into a frame
  --> Page table updated: 0x7F00_1000 now points to the frame
  --> Access completes, returns the byte at file offset 0x1050
```

This is demand paging applied to file I/O. The kernel's page cache handles caching, so repeated reads of the same region hit RAM, not disk.

## mmap() vs read()/write()

### Traditional I/O (read/write)

```
  Application          Kernel             Disk
  buffer               page cache
  +--------+          +--------+         +--------+
  |        | <--copy--| cached | <--DMA--| file   |
  | data   |          | data   |         | data   |
  +--------+          +--------+         +--------+
  User space           Kernel space

  Two copies: disk -> kernel buffer -> user buffer
  (or one with O_DIRECT, but you lose caching)
```

### mmap I/O

```
  Application                             Disk
  virtual address
  points directly
  to page cache
  +--------+                              +--------+
  |        |------> same physical <--DMA--| file   |
  | access |        frame as              | data   |
  +--------+        page cache            +--------+

  One copy: disk -> page cache (which IS the application's memory)
```

### Comparison

| Aspect | read()/write() | mmap() |
|--------|---------------|--------|
| Data copies | 2 (disk->kernel->user) or 1 with O_DIRECT | 1 (disk->page cache, mapped to user) |
| System calls | One per read/write operation | None after initial mmap (just memory access) |
| Random access | Requires lseek() + read() | Just use pointer arithmetic |
| Sequential access | Kernel can optimize read-ahead | Works but read-ahead less effective |
| Error handling | Per-call errno on read/write | SIGBUS on I/O error (harder to handle) |
| File size changes | Handled naturally | Must remap or handle SIGBUS |
| Network filesystems | Works reliably | Can cause hangs (NFS, CIFS) |
| Sharing | Requires explicit IPC | MAP_SHARED makes file visible to other processes |

## MAP_SHARED vs MAP_PRIVATE

The `flags` argument to mmap() determines how writes behave:

### MAP_SHARED

Writes go through to the underlying file. Multiple processes mapping the same file with MAP_SHARED see each other's changes. The kernel writes dirty pages back to the file (at some point — not immediately).

```
  Process A                    Process B
  mmap(MAP_SHARED, file)       mmap(MAP_SHARED, file)
       |                            |
       v                            v
  +----------+                 +----------+
  | Virtual  |                 | Virtual  |
  | Address  |                 | Address  |
  +-----+----+                 +----+-----+
        |                           |
        +-------> Same <-----------+
                  Physical Frame
                  (page cache)
                       |
                       v
                    Disk File

  A writes to mapped region --> B sees the change
  Changes eventually written to file by kernel
```

This is a powerful IPC mechanism — processes can communicate through a shared memory-mapped file without any system calls after the initial mmap.

### MAP_PRIVATE (Copy-on-Write)

The process gets its own private copy. Writes don't affect the file or other processes. Implemented using copy-on-write:

1. Initially, the mapping points to the same physical frames as the page cache
2. When the process writes, a page fault triggers
3. The kernel copies the page to a new frame, maps the private copy
4. The write goes to the private copy — the original file is unchanged

```
  Process with MAP_PRIVATE:

  Before write:
  Virtual page --> page cache frame (shared, read-only)

  After write to page:
  Virtual page --> NEW frame (private copy, read-write)
                   page cache frame (unchanged)
```

MAP_PRIVATE is used by the dynamic linker when loading shared libraries: the code sections are MAP_PRIVATE (they shouldn't be modified, but if some runtime patching is needed, the process gets its own copy).

## Anonymous mmap

mmap doesn't require a file. **Anonymous mmap** (`MAP_ANONYMOUS`) allocates memory that isn't backed by any file — it's backed by swap space (or nothing, until written to).

```
  void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE,
                   MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
```

This allocates `size` bytes of virtual memory, zero-initialized, not backed by a file. It's essentially a memory allocation.

### How malloc Uses mmap

The C library's `malloc()` uses two strategies internally:

```
  malloc(size):
    if size < threshold (~128 KB, configurable):
        Use brk()/sbrk() to extend the heap
        +------------------+
        | Heap grows -->   |  brk() moves the break point
        +------------------+
        - Fast (just moves a pointer)
        - But freed memory may not return to OS (fragmentation within heap)

    if size >= threshold:
        Use mmap(MAP_ANONYMOUS)
        +------------------+
        | Separate mapping |  Independent virtual memory region
        +------------------+
        - Each large allocation is independent
        - munmap() immediately returns memory to OS
        - Avoids fragmenting the main heap
```

This is why `free()` on a small allocation doesn't necessarily reduce your process's RSS (Resident Set Size) — the heap block is freed for reuse by malloc but not returned to the OS. Large allocations freed via `munmap()` are immediately returned.

```
  Process memory layout:

  +------------------+ High addresses
  |     Stack        |
  +------------------+
  |       |          |
  |       v          |
  |  (grows down)    |
  |                  |
  | mmap'd region    | <-- Large malloc (anonymous mmap)
  |                  |
  | mmap'd region    | <-- Shared library (file-backed mmap)
  |                  |
  | mmap'd region    | <-- Another large malloc
  |                  |
  |  (grows up)      |
  |       ^          |
  |       |          |
  +------------------+
  |     Heap         | <-- Small malloc (brk/sbrk)
  +------------------+
  |     BSS          |
  +------------------+
  |     Data         |
  +------------------+
  |     Code         |
  +------------------+ Low addresses
```

## Memory-Mapped I/O vs Port-Mapped I/O

A brief but important distinction. "Memory-mapped I/O" can refer to two different things:

**1. Memory-mapped files** (what this file is about): mapping file contents into virtual address space using mmap().

**2. Memory-mapped device I/O** (hardware concept): mapping device registers into the physical address space so the CPU can communicate with hardware devices using regular load/store instructions instead of special I/O instructions.

```
  Memory-Mapped I/O (devices):
  Physical Address Space:
  +------------------+ 0x0000_0000
  |     RAM          |
  +------------------+ 0x8000_0000
  | GPU registers    |  <-- Writing to 0x8000_0000 sends
  +------------------+      commands to the GPU
  | NIC registers    |  <-- Reading from here checks
  +------------------+      network card status
  | Disk controller  |
  +------------------+ 0xFFFF_FFFF

  Port-Mapped I/O (x86 legacy):
  Uses special in/out instructions to a separate I/O address space
  Not part of the memory address space
  Still used on x86 for legacy devices (keyboard, serial ports)
```

Modern systems predominantly use memory-mapped device I/O because it works with the same instructions (and MMU protections) as regular memory access. Port-mapped I/O is a legacy x86 concept.

## Real-World Connection

- **How databases use mmap**: MongoDB historically used mmap for its storage engine (MMAPv1). SQLite can use mmap for read-only access to the database file. The advantage is simplicity — the OS handles all the caching via the page cache. The disadvantage is less control over eviction, write ordering, and error handling (SIGBUS on I/O errors is hard to handle gracefully). WiredTiger (MongoDB's current engine) and PostgreSQL use their own buffer management instead of mmap for these reasons.

- **Dynamic linker (ld.so)**: when your program uses shared libraries (libc.so, libpthread.so), the dynamic linker maps them into your address space using mmap. Code sections are MAP_PRIVATE + PROT_READ|PROT_EXEC. Data sections are MAP_PRIVATE + PROT_READ|PROT_WRITE. Multiple processes sharing the same library share the same physical frames for the code pages (saving memory).

- **How malloc works internally**: as described above, glibc's malloc uses brk() for small allocations (fast, within the heap region) and anonymous mmap for large allocations (clean, independently freeable). The threshold is tunable via `mallopt(M_MMAP_THRESHOLD, size)`. Understanding this explains why a process's RSS can stay high even after freeing most of its memory — the small-allocation heap doesn't shrink.

- **Log file processing**: tools that process large log files benefit from mmap. Instead of reading the entire file into a buffer, mmap it and access it like an array. The kernel handles paging in only the portions you actually read, and the page cache prevents re-reading the same sections.

- **Shared memory IPC via mmap**: two processes can communicate by both mapping the same file with MAP_SHARED. Writes by one process are visible to the other through the shared page cache. Combined with synchronization primitives (mutexes in the shared region), this is a high-performance IPC mechanism. On Linux, `/dev/shm` is a tmpfs filesystem specifically designed for this purpose.

## Interview Angle

**Q: What is mmap() and how does it work?**

mmap() maps a file (or anonymous memory) directly into a process's virtual address space. The kernel sets up page table entries for the mapped region. Pages are loaded on demand via page faults — when the process first accesses a mapped address, the kernel reads the corresponding file block from disk into a physical frame and updates the page table. After that, the process accesses the file through regular memory operations, with no system calls needed. The kernel handles writing dirty pages back to the file.

**Q: What's the difference between MAP_SHARED and MAP_PRIVATE?**

MAP_SHARED means writes are visible to other processes that map the same file and are eventually written back to the file on disk. Multiple processes share the same physical frames. MAP_PRIVATE uses copy-on-write — the process initially shares frames with the page cache, but writes create private copies that don't affect the file or other processes. MAP_PRIVATE is used for things like shared library code, where each process might need to patch relocations without affecting others.

**Q: How does malloc work internally?**

glibc's malloc uses two strategies. For small allocations (typically < 128 KB), it uses brk()/sbrk() to extend the heap — a contiguous region of memory managed by the allocator with bins, free lists, and coalescing. For large allocations, it uses anonymous mmap to create independent memory regions. The mmap approach is slower per-allocation but avoids fragmenting the heap and returns memory to the OS immediately on free (via munmap). This dual strategy is why a process's memory usage might not decrease after freeing small allocations.

**Q: Why might a database choose NOT to use mmap for its storage engine?**

While mmap is simpler (the OS handles caching via the page cache), it gives the database less control over critical behaviors. The database can't control eviction order (important for transaction guarantees), can't control when dirty pages are written to disk (needed for write-ahead logging), and gets SIGBUS on I/O errors instead of a clean error return from read(). Databases like PostgreSQL manage their own buffer pool for these reasons, giving them precise control over page lifecycle, write ordering, and error handling.
