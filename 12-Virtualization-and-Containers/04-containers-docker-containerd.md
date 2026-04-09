# Containers: Docker and containerd

## What Is a Container, Really?

Strip away the marketing and a container is: **a regular Linux process (or group of processes) that has been isolated using namespaces and resource-limited using cgroups, running against a specific root filesystem.**

There is no "container" kernel object. There's no container syscall. The kernel doesn't know what a container is. It just sees a process with certain namespace memberships and cgroup assignments. The magic is in how the userspace tooling orchestrates these kernel primitives.

## Container Images: Layered Filesystem Snapshots

A container needs a root filesystem — the `/bin`, `/lib`, `/etc` that the process sees when it starts. This filesystem comes from a **container image**, which is a stack of read-only layers.

### Layers and the Union Filesystem

```
Container Image Layers:

┌─────────────────────────────────┐
│  Layer 3: COPY app.py /app/     │  ← Your application code
├─────────────────────────────────┤
│  Layer 2: RUN pip install flask │  ← Installed dependencies
├─────────────────────────────────┤
│  Layer 1: RUN apt-get update    │  ← System packages
├─────────────────────────────────┤
│  Layer 0: Ubuntu 22.04 base     │  ← Base OS filesystem
└─────────────────────────────────┘
   All layers are READ-ONLY

When a container runs:
┌─────────────────────────────────┐
│  Thin R/W Layer (container)     │  ← Writes go here (ephemeral)
├─────────────────────────────────┤
│  Layer 3: app.py                │  ┐
├─────────────────────────────────┤  │
│  Layer 2: flask packages        │  ├── Read-only image layers
├─────────────────────────────────┤  │
│  Layer 1: system packages       │  │
├─────────────────────────────────┤  │
│  Layer 0: Ubuntu base           │  ┘
└─────────────────────────────────┘
```

An **overlay filesystem** (OverlayFS on Linux) merges these layers into a single coherent view. The container sees a normal filesystem tree — it doesn't know about layers. When the container writes a file:

- **New file**: written to the thin read/write layer on top
- **Modify existing file**: the file is **copied up** from the read-only layer to the R/W layer, then modified there (copy-on-write)
- **Delete file**: a "whiteout" marker is placed in the R/W layer, hiding the file from lower layers

This is why container images are efficient:
- **Shared layers**: 100 containers from the same image share all read-only layers. Only the thin R/W layer is unique per container.
- **Fast startup**: no copying needed — just add a new R/W layer on top of existing layers.
- **Small deltas**: updating an image only transfers the changed layers.

### OCI Image Specification

Container images follow the **OCI (Open Container Initiative) Image Spec**, an industry standard. An OCI image is:

1. **A manifest** — JSON listing all layers and their digests (SHA256 hashes)
2. **A config** — JSON with metadata (environment variables, entrypoint command, exposed ports)
3. **Layer tarballs** — each layer is a compressed tar archive of filesystem changes

This standardization means images built with Docker work with containerd, Podman, CRI-O, and any OCI-compliant runtime.

## The Container Runtime Stack

"Docker" is not a single program. It's a stack of components, each with a specific role:

```
Container Runtime Hierarchy:

  ┌──────────────────────────────────────────────────┐
  │  docker CLI                                      │
  │  (user runs: docker run nginx)                   │
  └──────────────────┬───────────────────────────────┘
                     │ REST API
                     ▼
  ┌──────────────────────────────────────────────────┐
  │  dockerd (Docker daemon)                         │
  │  Manages images, volumes, networks               │
  │  Exposes the Docker API                          │
  └──────────────────┬───────────────────────────────┘
                     │ gRPC
                     ▼
  ┌──────────────────────────────────────────────────┐
  │  containerd                                      │
  │  Container lifecycle: create, start, stop, delete│
  │  Image pull/push, snapshot management            │
  └──────────────────┬───────────────────────────────┘
                     │ OCI Runtime Spec
                     ▼
  ┌──────────────────────────────────────────────────┐
  │  runc                                            │
  │  Actually creates the container:                 │
  │  - Sets up namespaces (clone/unshare)            │
  │  - Configures cgroups                            │
  │  - Calls pivot_root for filesystem               │
  │  - Drops privileges                              │
  │  - exec()s the container process                 │
  └──────────────────┬───────────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────────────────┐
  │  Container Process (e.g., nginx)                 │
  │  Just a regular Linux process now.               │
  │  Namespaced, cgroup-limited, filesystem-isolated │
  └──────────────────────────────────────────────────┘
```

| Component | Role | Written In |
|---|---|---|
| **docker CLI** | User interface — parses commands, talks to dockerd | Go |
| **dockerd** | High-level management — images, volumes, networks, build | Go |
| **containerd** | Mid-level runtime — container lifecycle, image management | Go |
| **runc** | Low-level runtime — creates the actual container using kernel primitives | Go |

**runc** does the real work. When it creates a container, it:
1. Creates new namespaces via `clone()` with namespace flags
2. Sets up cgroup limits by writing to `/sys/fs/cgroup/`
3. Mounts the overlay filesystem and calls `pivot_root` to make it the container's `/`
4. Applies security profiles (seccomp, AppArmor/SELinux)
5. Drops capabilities and `exec()`s the container's entrypoint

After `exec()`, runc exits. The container process is just a normal process on the host, managed by containerd.

## Dockerfile to Running Container

The lifecycle from source code to running container:

```
Dockerfile → Image → Container:

┌─────────────────────────┐
│  Dockerfile             │     docker build
│  FROM ubuntu:22.04      │  ──────────────►  Image
│  RUN apt-get install .. │                  (stored in
│  COPY app.py /app/      │                   registry)
│  CMD ["python", "app"]  │
└─────────────────────────┘
                                               │
                                  docker run   │
                                               ▼
                              ┌──────────────────────┐
                              │  Running Container   │
                              │  (namespaced process  │
                              │   with cgroup limits) │
                              └──────────────────────┘
```

Each `RUN` instruction in a Dockerfile creates a new layer. This is why instruction order matters — put frequently-changing steps (like `COPY . /app`) last so that earlier layers can be cached.

## Docker Networking

When containers need to communicate, Docker provides several network modes:

| Mode | How It Works | Use Case |
|---|---|---|
| **bridge** (default) | Containers connect to a virtual bridge (`docker0`). Each gets a veth pair and private IP (172.17.x.x). NAT for external access. | Single-host container communication |
| **host** | Container shares the host's network namespace. No network isolation. Container binds directly to host ports. | Maximum network performance, no isolation needed |
| **none** | No network interfaces (except loopback). Complete network isolation. | Security-sensitive workloads |
| **overlay** | VXLAN-based network spanning multiple Docker hosts. Containers on different machines can communicate as if on the same LAN. **VXLAN (Virtual Extensible LAN)** works by encapsulating Layer 2 Ethernet frames inside UDP packets, allowing containers on different physical hosts to behave as if they are on the same local network -- essentially creating a "virtual cable" across machines over the existing IP network. | Multi-host Docker Swarm or manual multi-host setups |

```
Bridge Networking:

  Host Network Namespace
  ┌─────────────────────────────────────────┐
  │  eth0: 10.0.0.5 (real NIC)             │
  │                                         │
  │  docker0: 172.17.0.1 (bridge)          │
  │      │           │                      │
  │    veth1       veth2                    │
  └──────┼───────────┼─────────────────────┘
         │           │
    ┌────┴────┐ ┌────┴────┐
    │Container│ │Container│
    │   A     │ │   B     │
    │eth0:    │ │eth0:    │
    │172.17.  │ │172.17.  │
    │0.2      │ │0.3      │
    └─────────┘ └─────────┘

  A and B can talk to each other via the bridge.
  External traffic goes through NAT on the host.
```

## Docker Storage Drivers

The storage driver handles how image layers are stacked and how the container's writable layer works:

| Driver | Filesystem | Notes |
|---|---|---|
| **overlay2** | OverlayFS | Default on modern Linux. Fast, efficient, well-supported |
| **devicemapper** | Device mapper thin provisioning | Older, used on CentOS/RHEL before overlay2 support |
| **btrfs** | Btrfs | Uses native Btrfs snapshots for layers |
| **zfs** | ZFS | Uses ZFS clones for layers |

overlay2 is the standard choice today. It's in the mainline kernel, requires no special filesystem setup, and performs well.

## Alternative Container Runtimes

runc is the reference implementation, but it's not the only option:

| Runtime | What It Does Differently |
|---|---|
| **crun** | Written in C instead of Go. Faster startup, lower memory usage. Drop-in replacement for runc. |
| **gVisor (runsc)** | Implements a user-space kernel that intercepts system calls. The container never talks to the real kernel directly. Stronger isolation but with a performance cost and limited syscall compatibility. Used by Google Cloud Run. |
| **Kata Containers** | Each container runs inside a lightweight VM. Container API on the outside, VM isolation underneath. Combines the convenience of containers with the security of VMs. |
| **Firecracker** | MicroVM runtime from AWS. Boots a minimal VM in ~125ms. Powers AWS Lambda and Fargate. Not a standard OCI runtime — it's a VMM optimized for container-like use cases. |

```
The Isolation Spectrum:

  Weaker isolation                              Stronger isolation
  Faster startup                                Slower startup
  ◄──────────────────────────────────────────────────────────►

  runc/crun         gVisor              Kata           Full VM
  (namespaces       (user-space         Containers     (separate
   + cgroups)        kernel)            (lightweight    kernel,
                                         VM per         hypervisor)
                                         container)
```

## Docker Alternatives

Docker isn't the only way to build and run containers:

| Tool | Role | Key Difference |
|---|---|---|
| **Podman** | Drop-in Docker replacement | Daemonless — no root daemon needed. Each container is a child of the calling process. Better systemd integration. |
| **Buildah** | Image building | Builds OCI images without running a daemon. Scriptable. |
| **nerdctl** | CLI for containerd | Docker-compatible CLI that talks directly to containerd (no dockerd needed) |
| **CRI-O** | Kubernetes container runtime | Minimal runtime designed specifically for Kubernetes. Implements the CRI (Container Runtime Interface) without Docker's extras. |

### Why Docker Isn't Needed on Kubernetes Anymore

Kubernetes removed Docker (dockershim) support in v1.24. Why?

```
Old Kubernetes path:                New Kubernetes path:
kubelet                             kubelet
  │ CRI                               │ CRI
  ▼                                    ▼
dockershim (translation layer)      containerd (directly)
  │                                    │
  ▼                                    ▼
dockerd (Docker daemon)             runc
  │
  ▼
containerd
  │
  ▼
runc

Docker added two unnecessary layers. Kubernetes only needs
containerd + runc. The Docker CLI, daemon, and API are
irrelevant for production container orchestration.
```

Kubernetes only needs something that implements the Container Runtime Interface (CRI) to manage containers. containerd and CRI-O both do this directly. Docker added overhead (dockershim translation, dockerd daemon) without benefit in a Kubernetes context. Your Docker images still work — they're OCI standard — but the Docker daemon is no longer part of the runtime chain.

## Real-World Connection

When you run `docker run -p 8080:80 nginx`:

1. **docker CLI** parses the command and sends a request to **dockerd**
2. **dockerd** checks if the nginx image exists locally; if not, pulls it from Docker Hub
3. **dockerd** tells **containerd** to create a container from the image
4. **containerd** calls **runc** with an OCI runtime spec
5. **runc** creates namespaces, sets up cgroups, mounts the overlay filesystem, and `exec`s nginx
6. **runc** exits; the nginx process is now running, managed by containerd
7. **dockerd** sets up the bridge network: creates a veth pair, assigns an IP, adds iptables rules for port mapping (host 8080 → container 80)

The whole thing takes about a second. Compare that to booting a VM.

When you push an image to a registry, only the layers that don't already exist at the destination are uploaded. If your base image (ubuntu:22.04) is already there, only your application layers transfer — often just a few MB.

## Interview Angle

**Q: What is a container image and how are layers used?**
A: A container image is a stack of read-only filesystem layers, each representing a set of changes (added/modified/deleted files). They follow the OCI Image Spec. When a container runs, a thin read/write layer is added on top using an overlay filesystem (typically OverlayFS). Writes use copy-on-write — modified files are copied up to the R/W layer. Multiple containers from the same image share all read-only layers, making them space-efficient and fast to start.

**Q: What is the relationship between Docker, containerd, and runc?**
A: They form a stack. Docker (dockerd) is the high-level daemon providing the user-facing API, image building, and networking. containerd is the mid-level runtime managing container lifecycle and image storage. runc is the low-level runtime that actually creates containers using kernel primitives (namespaces, cgroups, pivot_root). Kubernetes talks directly to containerd, bypassing Docker entirely.

**Q: What does runc actually do when creating a container?**
A: runc creates new namespaces (PID, network, mount, etc.) via clone/unshare syscalls, sets up cgroup resource limits, mounts the overlay filesystem and calls pivot_root to make it the container's root, applies security profiles (seccomp, AppArmor), drops unnecessary capabilities, and finally exec()s the container's entrypoint process. After exec, runc exits — the container process is just a regular namespaced process on the host.

**Q: Why was Docker removed from Kubernetes?**
A: Kubernetes only needs a Container Runtime Interface (CRI) implementation to manage containers. Docker added unnecessary layers — the dockershim translation layer and the dockerd daemon — between kubelet and containerd. By talking directly to containerd (or CRI-O), Kubernetes eliminates that overhead. Docker images still work because they follow the OCI standard; it's only the Docker daemon that's no longer in the path.

---

Next: [VM vs Container Tradeoffs](05-vm-vs-container-tradeoffs.md)
