# Wasm Runtimes and the OS Boundary

Solomon Hykes, co-founder of Docker, wrote in 2019: "If WASM+WASI existed in 2008, we wouldn't have needed to create Docker." That is a strong statement from the person who popularized containers. It hints at something significant: WebAssembly is not just a browser technology. It is becoming a new abstraction layer that sits where the OS boundary used to be -- providing isolation, portability, and security in a fundamentally different way than containers.

---

## What Is WebAssembly?

**WebAssembly (Wasm)** is a portable binary instruction format. Originally designed to run near-native-speed code in web browsers, it has evolved into a general-purpose compilation target.

Key properties:
- **Binary format**: compact, fast to decode (not text like JavaScript)
- **Sandboxed execution**: cannot access memory outside its linear memory, cannot make syscalls directly
- **Architecture-independent**: the same `.wasm` file runs on x86, ARM, RISC-V
- **Language-agnostic**: compile from C, C++, Rust, Go, Python, and more

```
Source Code                    Wasm Module                  Execution
+----------+                  +------------+               +----------+
| Rust     |                  |            |               |          |
| C/C++    +--- compile ----->| .wasm file +--- runtime -->| Sandboxed|
| Go       |   (to Wasm)      | (bytecode) |   (Wasmtime, | Execution|
| Python   |                  |            |   Wasmer...)  |          |
+----------+                  +------------+               +----------+
                              Architecture-                 Memory
                              independent                   isolated
```

---

## WASI: The System Interface

In a browser, Wasm modules interact with the outside world through JavaScript. On a server, there is no JavaScript. **WASI** (WebAssembly System Interface) provides a standardized set of syscall-like APIs that Wasm modules can use to interact with the host system.

```
Traditional Process:                 Wasm Module:

Application                          Application (Wasm)
    |                                    |
    | syscall                            | WASI call
    v                                    v
Linux Kernel                         Wasm Runtime (Wasmtime)
    |                                    |
    | hardware access                    | controlled host access
    v                                    v
Hardware                             Host OS (selectively)
```

WASI provides:
- **File I/O**: read/write files (only directories explicitly granted)
- **Environment variables**: access specific env vars
- **Clocks**: time functions
- **Random numbers**: secure random generation
- **Networking** (emerging): sockets, HTTP (still maturing)

The critical design principle: **capability-based security**. A Wasm module cannot access any file on the system -- it can only access directories the host explicitly grants. There is no ambient authority.

```
Capability-Based Security:

Container:                           Wasm (WASI):
+-----------------------+            +-----------------------+
| Process can access    |            | Module can ONLY       |
| any file the user     |            | access resources      |
| can access            |            | explicitly granted    |
| (ambient authority)   |            | (capability tokens)   |
|                       |            |                       |
| seccomp/AppArmor      |            | Built into the        |
| adds restrictions     |            | execution model       |
| (deny list)           |            | (allow list)          |
+-----------------------+            +-----------------------+
```

---

## Wasm as a New "OS Boundary"

Traditionally, the OS boundary is the syscall interface: user-mode code crosses into kernel mode to access resources. Containers add a namespace layer on top. Wasm introduces a different boundary entirely.

```
Isolation Layers:

Native Process:    Container:         Wasm Module:

+----------+       +----------+       +----------+
| App Code |       | App Code |       | App Code |
+----+-----+       +----+-----+       +----+-----+
     |                  |                  |
     v                  v                  v
+---------+       +-----------+       +----------+
| syscall |       | namespace |       | Wasm     |
| (kernel |       | + cgroup  |       | Runtime  |
| boundary|       | + syscall |       | (sandbox)|
| )       |       | (kernel)  |       +----+-----+
+---------+       +-----------+            |
     |                  |                  v
     v                  v             +---------+
  Kernel              Kernel          | Host OS |
                                      | (only   |
                                      | granted |
                                      | caps)   |
                                      +---------+
```

The Wasm runtime provides:
- **Memory isolation**: each module has its own linear memory; it cannot read or write host memory
- **Control flow integrity**: the module can only call exported functions; no arbitrary jumps
- **Capability-based I/O**: no ambient access to files, network, or environment
- **Deterministic execution**: same inputs produce same outputs (useful for reproducible builds)

---

## Wasm vs Containers

| Aspect | Container (Docker) | Wasm Module |
|---|---|---|
| **Startup time** | ~1 second | Microseconds |
| **Image size** | 10s-100s MB | KBs to low MBs |
| **Isolation mechanism** | OS namespaces + cgroups | Language-level sandbox |
| **Portability** | Requires matching kernel (Linux) | Architecture-independent |
| **Security model** | Deny-list (seccomp, AppArmor) | Allow-list (capabilities) |
| **Multi-language** | Any (runs a full process) | Compile-to-Wasm languages |
| **Ecosystem maturity** | Mature (Docker, K8s, etc.) | Early but growing fast |
| **Networking** | Full TCP/IP stack | Limited (WASI networking still evolving) |
| **File system** | Full (mount namespace) | Capability-granted directories only |
| **Debugging** | Familiar tools | Improving but limited |
| **Best for** | General workloads, microservices | Edge, plugins, serverless, functions |

---

## Wasm Runtimes

| Runtime | Focus | Notable Feature |
|---|---|---|
| **Wasmtime** | Server-side, WASI reference | Bytecode Alliance, Cranelift JIT compiler |
| **Wasmer** | Universal Wasm runtime | Supports multiple backends (LLVM, Cranelift, Singlepass) |
| **WasmEdge** | Edge and cloud-native | Optimized for cloud-native workloads, CNCF project |
| **V8** | Browser + server (Node/Deno) | Highly optimized JIT, used by Cloudflare Workers |
| **wazero** | Go-native Wasm runtime | Zero dependencies, no CGo, pure Go |

---

## The Component Model: Composing Wasm Modules

The **Wasm Component Model** is an emerging standard for composing Wasm modules -- connecting them together with typed interfaces, like microservices but running in the same process.

```
Traditional Microservices:              Wasm Component Model:

+-------+  HTTP   +-------+            +---------------------------+
| Svc A +-------->+ Svc B |            | +-------+     +-------+  |
+-------+         +-------+            | | Comp  | fn  | Comp  |  |
    |                 |                 | |   A   +---->+   B   |  |
    | HTTP            | HTTP            | +-------+     +-------+  |
    v                 v                 |     |                    |
+-------+         +-------+            |     | fn call            |
| Svc C |         | Svc D |            |     v                    |
+-------+         +-------+            | +-------+                |
                                        | | Comp  |                |
Network calls (ms latency)             | |   C   |                |
Serialization overhead                  | +-------+                |
Process isolation                       +---------------------------+
                                         Function calls (ns latency)
                                         No serialization
                                         Sandbox isolation
```

Components define interfaces using **WIT** (Wasm Interface Types) -- a typed IDL that allows components written in different languages to interoperate. A Rust component can call a Go component's function directly, in the same process, with the safety guarantees of Wasm sandboxing.

---

## Current Limitations

Wasm outside the browser is powerful but still maturing:

| Limitation | Status | Impact |
|---|---|---|
| **Networking** | WASI sockets in development | Cannot yet build a general-purpose web server as easily as with containers |
| **Threads** | Wasm threads proposal in progress | Limited parallelism for compute-heavy workloads |
| **File system** | Basic WASI FS available | No advanced FS features (inotify, mmap) |
| **Ecosystem** | Growing rapidly | Not all libraries compile to Wasm; some require POSIX features |
| **Debugging** | Basic source maps, DWARF | Not as mature as native debugging tools |
| **Garbage collection** | Wasm GC proposal shipping | Languages with GC (Java, C#, Python) compile less efficiently |

These limitations are why Wasm has not replaced containers. But they are being actively addressed, and for specific workloads (edge functions, plugin systems, serverless), Wasm is already production-ready.

---

## Comparison: Native Process vs Container vs Wasm

| Aspect | Native Process | Container | Wasm Module |
|---|---|---|---|
| **Abstraction** | OS process | Namespaced process | Sandboxed bytecode |
| **Isolation** | Process boundary | Namespace + cgroup | Linear memory sandbox |
| **Startup** | Milliseconds | ~1 second | Microseconds |
| **Image size** | Application binary | 10s-100s MB | KBs-MBs |
| **Portability** | Recompile per arch | Recompile per OS | Run anywhere |
| **Security** | User permissions | seccomp + namespaces | Capability-based |
| **Performance** | Native | Near-native | Near-native (JIT) |
| **File access** | Ambient | Namespace-filtered | Explicitly granted |
| **Best fit** | System services | Microservices | Edge, plugins, serverless |

---

## Real-World Connection

**Cloudflare Workers**: Runs customer code on Cloudflare's edge network using V8 isolates (which support Wasm). Cold start is under 5 milliseconds. They run thousands of tenants on each machine -- possible because Wasm isolates are far cheaper than containers.

**Fermyon Spin**: A framework for building serverless applications with Wasm. Each HTTP request handler is a Wasm component. Startup time is microseconds, enabling true "scale to zero" without cold start penalties.

**Shopify Functions**: Shopify lets merchants write custom logic (discounts, shipping rules) as Wasm modules. The sandbox ensures merchant code cannot access other merchants' data or the host system -- capability-based security in action.

**Plugin systems**: Many applications (databases, editors, game engines) are adopting Wasm as a plugin format. Plugins get sandboxed execution, memory isolation, and cross-platform portability without the risk of loading arbitrary native code.

**Docker + Wasm**: Docker Desktop now supports running Wasm containers alongside traditional Linux containers. You can use `docker run --runtime=io.containerd.wasmedge.v1` to run a Wasm module through Docker's interface -- a sign of convergence.

---

## Interview Angle

**Q: What is WASI and why does it matter?**

A: WASI (WebAssembly System Interface) is a standardized set of APIs that allow Wasm modules to interact with the host system -- file I/O, environment variables, clocks, networking. It matters because it enables Wasm to run outside the browser as a general-purpose execution environment. The critical design choice is capability-based security: a Wasm module has zero ambient authority. It cannot access any file, network socket, or environment variable unless the host explicitly grants that capability. This is the inverse of the traditional POSIX model where a process can access anything its user can. WASI makes the security boundary the default, not an afterthought.

**Q: Why might Wasm replace containers for some workloads?**

A: Three reasons: startup time (microseconds vs seconds -- eliminates the serverless cold start problem), image size (KBs vs MBs -- faster deployment, more tenants per machine), and security model (capability-based allow-list vs deny-list). For edge computing (Cloudflare Workers), plugin systems (Shopify Functions), and serverless functions, these advantages are significant. However, Wasm will not replace containers for general-purpose microservices anytime soon because WASI networking is still maturing, not all languages compile to Wasm efficiently, and the debugging experience is less mature. The future is likely both: containers for general workloads, Wasm for edge and functions.

**Q: Compare the security models of containers and Wasm.**

A: Containers use a deny-list approach. A process in a container can, by default, make any syscall the kernel allows. You then restrict it with seccomp profiles (blocking dangerous syscalls) and AppArmor/SELinux policies. If you forget to apply a profile, the process has broad access. Wasm uses an allow-list approach built into the execution model. A Wasm module starts with zero capabilities -- it cannot access any file, any network socket, anything. The host must explicitly grant each capability (this directory, this env var, this socket). A forgotten configuration means the module has less access, not more. The Wasm model is more secure by default, but also more restrictive, which is why it suits sandboxed workloads (plugins, functions) better than general-purpose services.

---

Next: [OS for Distributed Systems](05-os-for-distributed-systems.md)
