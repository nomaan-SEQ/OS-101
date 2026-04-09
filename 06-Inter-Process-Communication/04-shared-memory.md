# Shared Memory

## The Fastest IPC Mechanism

Every other IPC mechanism copies data through the kernel: the sender writes data into a kernel buffer, and the receiver reads it out. Shared memory eliminates this entirely. Two processes map the **same physical memory** into their virtual address spaces, then read and write directly. After the initial setup, the kernel is not involved at all.

```
   Normal IPC (pipes, message queues):

   Process A           Kernel            Process B
   +---------+     +-----------+      +---------+
   | buffer  |---->| copy data |----->| buffer  |
   +---------+     +-----------+      +---------+
                   Two copies: A->kernel, kernel->B


   Shared Memory:

   Process A                              Process B
   +---------+                           +---------+
   | virtual |                           | virtual |
   | addr    |                           | addr    |
   | space   |                           | space   |
   |  0x7f.. |---+                   +---|  0x7f.. |
   +---------+   |                   |   +---------+
                 |   Physical RAM    |
                 +-->+-----------+<--+
                     | shared    |
                     | memory    |
                     | region    |
                     +-----------+
                     No kernel copy!
```

This makes shared memory ideal for **high-throughput, low-latency** data sharing. But there's a cost: since the kernel doesn't mediate access, **you must handle synchronization yourself**.

## How It Works

The lifecycle of shared memory:

```
1. CREATE    Process A creates a shared memory region
                |
2. ATTACH    Process A maps the region into its address space
                |
3. SHARE     Process B opens and maps the same region
                |
4. USE       Both processes read/write directly (with synchronization!)
                |
5. DETACH    Each process unmaps the region when done
                |
6. DESTROY   One process removes the shared memory object
```

## System V Shared Memory

```c
#include <sys/shm.h>

// 1. Create a shared memory segment
int shmid = shmget(key, size, IPC_CREAT | 0666);
//   key:  identifier (use ftok() or IPC_PRIVATE)
//   size: segment size in bytes

// 2. Attach: map into this process's address space
void *ptr = shmat(shmid, NULL, 0);
//   Returns pointer to the shared region
//   NULL = let kernel choose the address

// 3. Use it — read/write through the pointer
sprintf(ptr, "Hello from Process A");   // write
printf("Got: %s\n", (char *)ptr);       // read

// 4. Detach: unmap from this process
shmdt(ptr);

// 5. Destroy: remove the segment (when no longer needed)
shmctl(shmid, IPC_RMID, NULL);
```

## POSIX Shared Memory

The modern, cleaner API that works through the filesystem (`/dev/shm` on Linux).

```c
#include <sys/mman.h>
#include <fcntl.h>

// 1. Create/open a shared memory object
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
//   Name must start with /
//   Creates a file in /dev/shm/

// 2. Set the size
ftruncate(fd, 4096);

// 3. Map into this process's address space
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd, 0);

// 4. Use it — just like System V
sprintf(ptr, "Hello from Process A");

// 5. Unmap
munmap(ptr, 4096);

// 6. Close the file descriptor
close(fd);

// 7. Remove the shared memory object (when done)
shm_unlink("/myshm");
```

```bash
# Inspect POSIX shared memory on Linux
ls -la /dev/shm/

# Inspect System V shared memory
ipcs -m
```

## The Synchronization Problem

Shared memory gives you raw speed, but the kernel provides **no protection** against concurrent access. You need explicit synchronization:

```
  Process A                    Process B
  writes "Hello"               reads shared memory
       |                            |
       +--- What if B reads while A is mid-write? ---+
       |                                              |
       v                                              v
  You get torn reads, corrupted data, race conditions.
```

**Common synchronization approaches for shared memory:**

| Approach | How It Works | Tradeoff |
|----------|-------------|----------|
| **POSIX semaphore** (named) | `sem_open()`, `sem_wait()`, `sem_post()` | Simple, works across processes |
| **Mutex in shared memory** | Place `pthread_mutex_t` in the shared region with `PTHREAD_PROCESS_SHARED` | Lowest overhead, but setup is more involved |
| **System V semaphore** | `semget()`, `semop()` | Legacy, more complex API |
| **Atomic operations** | `__atomic_*` builtins, C11 `_Atomic` | Lock-free, but limited to simple operations |
| **File locking** | `flock()` or `fcntl()` locks | Simple but coarse-grained |

A typical pattern uses a POSIX named semaphore:

```c
sem_t *sem = sem_open("/mysem", O_CREAT, 0666, 1);

// Before accessing shared memory:
sem_wait(sem);          // lock
// ... read/write shared memory ...
sem_post(sem);          // unlock
```

## mmap with MAP_SHARED

You don't always need `shmget` or `shm_open`. The `mmap()` system call with `MAP_SHARED` can create shared mappings in two ways:

### Anonymous Shared Mapping (Related Processes)

```c
// Parent creates mapping before fork
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_SHARED | MAP_ANONYMOUS, -1, 0);

pid_t pid = fork();
// Both parent and child now share this memory region
// (child inherits the mapping through fork)
```

### Memory-Mapped Files (Any Processes)

```c
int fd = open("/tmp/shared_data", O_RDWR | O_CREAT, 0666);
ftruncate(fd, 4096);

void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_SHARED, fd, 0);

// Any process that maps the same file shares the memory
// Changes are eventually written back to the file (persistence!)
```

Memory-mapped files are powerful because:
- Any process can participate (just open the same file)
- Data persists across process restarts
- The kernel handles the file I/O (page cache integration)
- This is how databases implement their buffer pools

```
  Memory-Mapped File IPC:

  Process A          Process B          File on Disk
  +--------+        +--------+        +----------+
  | mmap   |------->|        |        |          |
  | region |   same | mmap   |        | data.bin |
  |        |<-------| region |<------>|          |
  +--------+ phys   +--------+  page  +----------+
              mem                cache
              pages             sync
```

## Comparison of Shared Memory APIs

| Feature | System V (`shmget`) | POSIX (`shm_open`) | `mmap` (file-backed) | `mmap` (anonymous) |
|---------|--------------------|--------------------|---------------------|-------------------|
| **Naming** | Integer key | String (starts with /) | File path | None (related processes only) |
| **Persistence** | Until `shmctl(IPC_RMID)` | Until `shm_unlink()` | File on disk | Until processes exit |
| **Unrelated processes** | Yes | Yes | Yes | No (parent-child only) |
| **File system visible** | `ipcs -m` | `/dev/shm/` | Normal file | Not visible |
| **Portability** | Very broad | Most modern Unix | Universal | Universal |
| **Best for** | Legacy systems | New POSIX-compliant code | Persistent shared data | Simple parent-child sharing |

## Real-World Connection

- **PostgreSQL** uses System V shared memory for its shared buffer pool — the cache of database pages that all backend processes access. This is why `shmmax` kernel parameter tuning was historically important for PostgreSQL.
- **Chrome/Chromium** uses shared memory between the browser process and renderer processes for compositing and transferring rendered content. Each tab is a separate process, and shared memory avoids copying large bitmaps.
- **Redis** with the `fork()` for background saves uses copy-on-write semantics of shared memory pages — the child process shares the parent's memory until pages are modified.
- **tmpfs** (`/dev/shm` on Linux) is a filesystem backed entirely by RAM/swap, making it the natural home for POSIX shared memory objects.
- **DPDK** (Data Plane Development Kit) uses huge-page shared memory to share packet buffers between processes, achieving millions of packets per second.
- **Numpy/PyTorch** multiprocessing uses shared memory to share large tensors between worker processes without copying.

## Interview Angle

**Q: Why is shared memory the fastest IPC mechanism?**

A: Because after the initial setup (creating and mapping the region), there is no kernel involvement. Both processes read and write directly to the same physical memory pages. All other IPC mechanisms (pipes, message queues, sockets) require the kernel to copy data between address spaces — at least one copy in, one copy out. Shared memory eliminates both copies.

**Q: If shared memory is the fastest, why doesn't everyone use it?**

A: Because it shifts the synchronization burden entirely to the programmer. The kernel provides no mediation — if two processes read and write simultaneously without locking, you get data corruption. This makes shared memory harder to use correctly and harder to debug. For most applications, the simplicity of pipes or sockets outweighs the performance advantage. Shared memory is worth the complexity only when throughput is critical and the data being shared is large.

**Q: What's the difference between `shm_open` + `mmap` and just using `mmap` on a regular file?**

A: `shm_open` creates an object in a RAM-backed filesystem (`/dev/shm` on Linux), so there's no disk I/O involved — it's purely in-memory. `mmap` on a regular file uses the page cache and may involve disk I/O (reads on first access, writes on flush). Use `shm_open` when you want pure in-memory sharing. Use file-backed `mmap` when you also want the data to persist on disk, or when you're using the file as the rendezvous point between processes.

---

**Next:** [Sockets for IPC](05-sockets-for-ipc.md) — Unix domain sockets, the flexible workhorse of modern local IPC.
