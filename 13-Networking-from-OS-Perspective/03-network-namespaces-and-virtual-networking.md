# Network Namespaces and Virtual Networking

Every container you run gets its own network stack. It has its own IP address, its own routing table, its own iptables rules, and its own set of sockets — completely isolated from the host and other containers. This isolation is not magic. It is built from a small set of kernel primitives: network namespaces, virtual ethernet pairs, software bridges, and packet encapsulation. Understanding these primitives is understanding how Docker, Kubernetes, and cloud networking actually work.

## Network Namespaces (Deep Dive)

We introduced namespaces in section 12. Here we go deeper into the network namespace specifically.

A network namespace provides an isolated copy of the entire network stack:

| Resource | Per-Namespace? | Meaning |
|----------|---------------|---------|
| Network interfaces | Yes | Each namespace has its own `eth0`, `lo`, etc. |
| IP addresses | Yes | Same IP can exist in different namespaces |
| Routing table | Yes | Each namespace routes independently |
| iptables/nftables rules | Yes | Firewall rules are namespace-scoped |
| Socket table | Yes | Sockets in one namespace are invisible to another |
| `/proc/net` | Yes | Network stats are per-namespace |
| ARP table | Yes | Each namespace resolves MACs independently |

### Creating and Using Network Namespaces

```bash
# Create a namespace
ip netns add container1

# Run a command inside it
ip netns exec container1 ip addr
# Output: only loopback (lo), no other interfaces

# Programmatically (what Docker does):
clone(CLONE_NEWNET | CLONE_NEWPID | ...)
# Child process starts in a fresh network namespace
```

A new network namespace starts empty — only a loopback interface (down by default). To communicate with the outside world, you need to create virtual interfaces and connect them.

## Virtual Ethernet Pairs (veth)

A veth pair is like an Ethernet cable with two ends. Anything sent into one end comes out the other. The key feature: each end can be placed in a different network namespace.

```bash
# Create a veth pair
ip link add veth0 type veth peer name veth1

# Move one end into the container namespace
ip link set veth1 netns container1

# Configure the host end
ip addr add 172.17.0.1/24 dev veth0
ip link set veth0 up

# Configure the container end
ip netns exec container1 ip addr add 172.17.0.2/24 dev veth1
ip netns exec container1 ip link set veth1 up
ip netns exec container1 ip link set lo up
```

Now the host and container can communicate:

```
┌─────────────────────┐         ┌─────────────────────┐
│   Host Namespace    │         │  Container Namespace │
│                     │         │                      │
│  veth0              │         │  veth1               │
│  172.17.0.1    ─────┼─────────┼──── 172.17.0.2      │
│                     │  veth   │                      │
│                     │  pair   │                      │
│  eth0 (real NIC)    │         │  (no external        │
│  10.0.0.5           │         │   access yet)        │
└─────────────────────┘         └──────────────────────┘
```

But the container still cannot reach the internet. For that, we need a bridge and NAT.

## Linux Bridge: The Software Switch

A Linux bridge operates like a physical Ethernet switch — it connects multiple interfaces at Layer 2 and forwards frames based on MAC addresses.

```bash
# Create a bridge
ip link add br0 type bridge
ip link set br0 up
ip addr add 172.17.0.1/24 dev br0

# Attach container veth endpoints to the bridge
ip link set veth0 master br0
ip link set veth2 master br0  # another container's veth
```

```
┌────────────────────────────────────────────────────────┐
│                    Host Namespace                       │
│                                                        │
│   ┌──────────┐                                         │
│   │  br0     │  172.17.0.1 (bridge, acts as gateway)   │
│   │ (bridge) │                                         │
│   └──┬───┬───┘                                         │
│      │   │                                             │
│  veth0   veth2         eth0                            │
│   │       │          10.0.0.5 (real NIC)               │
└───┼───────┼──────────────┼─────────────────────────────┘
    │       │              │
    │       │              │ (NAT/masquerade to internet)
    │       │
┌───┴────┐ ┌┴─────────┐
│  C1    │ │   C2     │
│ .0.2   │ │  .0.3    │
│ veth1  │ │  veth3   │
└────────┘ └──────────┘
```

**How containers talk to each other**: Container C1 sends a frame to 172.17.0.3 (C2). The bridge sees the destination MAC, looks it up in its forwarding table, and sends the frame out the correct veth port. Pure Layer 2 switching — no routing needed.

**How containers reach the internet**: Container C1 sends a packet to 8.8.8.8. It goes to the bridge (C1's default gateway is 172.17.0.1). The host routes it to eth0 with an iptables MASQUERADE rule that replaces the source IP (172.17.0.2) with the host's IP (10.0.0.5). The reply comes back, conntrack translates the destination back, and the packet reaches C1.

## Docker Bridge Networking: The Full Picture

When Docker starts with the default bridge network, here is exactly what it creates:

```
                        External Network
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                     Host (10.0.0.5)                      │
│                                                          │
│   eth0 ──── 10.0.0.5                                    │
│     │                                                    │
│     │  iptables MASQUERADE:                              │
│     │  -t nat -A POSTROUTING -s 172.17.0.0/16           │
│     │       -o eth0 -j MASQUERADE                        │
│     │                                                    │
│     │  iptables DNAT (port publish):                     │
│     │  -t nat -A PREROUTING -p tcp --dport 8080          │
│     │       -j DNAT --to 172.17.0.2:80                   │
│     │                                                    │
│   ┌─┴──────────┐                                         │
│   │  docker0    │  172.17.0.1/16  (Linux bridge)         │
│   │  (bridge)   │                                        │
│   └──┬──────┬───┘                                        │
│   vethXXX  vethYYY                                       │
│      │        │                                          │
└──────┼────────┼──────────────────────────────────────────┘
       │        │
  ┌────┴───┐  ┌─┴───────┐
  │ nginx  │  │  redis   │
  │ :80    │  │  :6379   │
  │172.17  │  │ 172.17   │
  │ .0.2   │  │  .0.3    │
  └────────┘  └──────────┘
```

The packet path for an external client reaching nginx (published as host port 8080):

1. Packet arrives at host eth0, dest port 8080
2. PREROUTING chain: DNAT changes dest to 172.17.0.2:80
3. Routing: forward to docker0 bridge
4. Bridge: sends to vethXXX (nginx's veth)
5. vethXXX delivers to container namespace
6. nginx receives on port 80

Return path:
1. nginx replies, source 172.17.0.2:80
2. conntrack translates source back to 10.0.0.5:8080
3. POSTROUTING: MASQUERADE applied
4. Packet exits eth0

## Overlay Networks: Cross-Host Container Networking

Bridges work on a single host. When containers on different hosts need to communicate (as in a cluster), you need overlay networks.

### VXLAN (Virtual Extensible LAN)

VXLAN encapsulates Layer 2 Ethernet frames inside UDP packets, allowing a virtual L2 network to span across L3 boundaries:

```
Container A (Host 1)                    Container B (Host 2)
172.18.0.2                              172.18.0.3
    │                                       ▲
    ▼                                       │
┌───────────┐                          ┌───────────┐
│  Bridge   │                          │  Bridge   │
└─────┬─────┘                          └─────┬─────┘
      │                                      │
┌─────┴──────────┐                    ┌──────┴─────────┐
│  VXLAN device  │                    │  VXLAN device  │
│  Encapsulate:  │                    │  Decapsulate:  │
│  Original L2   │                    │  Strip outer   │
│  frame inside  │                    │  UDP/IP,       │
│  UDP packet    │                    │  deliver inner │
└─────┬──────────┘                    └──────┬─────────┘
      │                                      │
      ▼                                      │
┌──────────────── Physical Network ──────────────────┐
│  Host 1 (10.0.1.1) ──── UDP:4789 ──── Host 2      │
│                                     (10.0.1.2)     │
└────────────────────────────────────────────────────┘
```

The packet format:

```
┌──────────────────────────────────────────────────┐
│ Outer Ethernet │ Outer IP │ Outer UDP │ VXLAN    │
│ (host MACs)    │ 10.0.1.1 │ :4789    │ Header   │
│                │→10.0.1.2 │          │ (VNI)    │
├──────────────────────────────────────────────────┤
│ Inner Ethernet │ Inner IP  │ TCP/UDP  │ Payload  │
│ (container     │ 172.18.0.2│          │          │
│  MACs)         │→172.18.0.3│          │          │
└──────────────────────────────────────────────────┘
```

**Used by**: Docker Swarm overlay driver, Flannel (VXLAN mode), Weave Net.

Overhead: ~50 bytes per packet (outer headers + VXLAN header). This reduces the effective MTU inside the overlay — a common source of mysterious "connection hangs" when large packets hit the MTU limit and Path MTU Discovery is broken.

## Macvlan and IPvlan

Sometimes you want containers to appear as real hosts on the physical network — no bridge, no NAT.

| Mode | How It Works | Use Case |
|------|-------------|----------|
| **Macvlan** | Each container gets its own MAC address on the host NIC | Container needs to be a first-class network citizen |
| **IPvlan L2** | Containers share the host's MAC but get unique IPs | When the switch limits MAC addresses per port |
| **IPvlan L3** | Like L2 but routing-based instead of bridging | Large-scale, routed container deployments |

```
Macvlan:
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Host    │  │  C1      │  │  C2      │
│ MAC: AA  │  │ MAC: BB  │  │ MAC: CC  │
│ IP: .10  │  │ IP: .11  │  │ IP: .12  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     └──────────┬──┴──────────┬──┘
                │  Physical NIC│
                │  (all MACs)  │
                └──────────────┘
```

No bridge, no NAT, no iptables overhead. Packets go directly to/from the physical NIC. The trade-off: host-to-container communication requires special routing (the host NIC cannot send to its own macvlan sub-interfaces directly).

## eBPF for Networking

eBPF (extended Berkeley Packet Filter) lets you run sandboxed programs in the kernel at various hook points — including network hooks. It is rapidly replacing iptables for high-performance networking.

**Why replace iptables?**
- iptables rules are O(n) — evaluated sequentially for every packet
- Modifying iptables rules requires replacing entire chains
- conntrack adds overhead even when not needed
- No programmability — fixed set of actions (ACCEPT, DROP, DNAT, etc.)

**eBPF advantages**:
- Programs compiled to kernel bytecode, JIT-compiled to native instructions
- O(1) lookups using eBPF maps (hash tables)
- Attach at multiple hook points: XDP (driver level), TC (traffic control), socket level
- Full programmability — implement custom load balancing, policy, observability

### Cilium: Kubernetes CNI Using eBPF

Cilium is a Kubernetes network plugin (CNI) that replaces kube-proxy and iptables with eBPF programs:

```
Traditional Kubernetes Networking:

  Pod → veth → bridge → iptables (DNAT to Service IP)
       → iptables (select backend pod)
       → routing → veth → destination Pod

  Hundreds of iptables rules for services
  O(n) per packet

Cilium with eBPF:

  Pod → veth → eBPF program (TC hook)
       → O(1) map lookup for Service → Pod mapping
       → direct routing to destination Pod

  No iptables, no conntrack overhead
  O(1) per packet
```

Cilium also provides:
- **Network policy enforcement**: eBPF programs drop unauthorized packets at the source
- **L7 visibility**: Inspect HTTP, gRPC, Kafka headers without a sidecar proxy
- **Transparent encryption**: WireGuard-based encryption between nodes

## Real-World Connection

**Kubernetes pod networking (CNI)**: Every Kubernetes pod gets its own network namespace with a unique IP address. The CNI (Container Network Interface) plugin is responsible for creating the veth pair, attaching it to the right bridge or routing table, and ensuring pods across nodes can reach each other. Different CNIs choose different approaches: Flannel uses VXLAN overlays, Calico uses BGP routing (no encapsulation overhead), and Cilium uses eBPF for both routing and policy.

**Service mesh and packet path**: Istio injects an Envoy sidecar proxy into each pod. Traffic is redirected through the sidecar using iptables rules (REDIRECT in the PREROUTING chain). Every request goes: app → iptables → Envoy → iptables → kernel → network → kernel → iptables → Envoy → iptables → destination app. That is a lot of iptables traversals — this is why Cilium's eBPF-based service mesh is gaining traction. It can implement some mesh features at the kernel level, skipping the sidecar entirely for L4 policy.

**Docker Compose networking**: When you run `docker compose up`, Docker creates a dedicated bridge network for the compose project. All services in the compose file are attached to this bridge. DNS resolution is provided by Docker's embedded DNS server — services can reach each other by name (e.g., `web` can connect to `db:5432`). This is all network namespaces + veth pairs + a bridge + iptables NAT, created automatically.

**MTU issues with overlay networks**: VXLAN adds ~50 bytes of overhead. If the physical network MTU is 1500, the effective container MTU is ~1450. If a container sends a 1500-byte packet, the outer encapsulated packet exceeds the physical MTU and must be fragmented — or dropped if DF (Don't Fragment) is set. This causes symptoms like: TCP connections establish but hang when transferring data, small requests work but large ones time out. The fix: set container MTU to account for encapsulation overhead.

## Interview Angle

**Q: How do two Docker containers on the same host communicate?**

A: Each container runs in its own network namespace. A veth pair connects each container to the `docker0` bridge — one end is in the container namespace (visible as `eth0` inside the container), the other end is attached to the bridge on the host. When Container A sends a packet to Container B's IP, it goes through A's veth to the bridge. The bridge operates as a Layer 2 switch — it looks up the destination MAC in its forwarding table and sends the frame out the correct veth port to Container B. No routing or NAT is needed for same-host, same-bridge traffic.

**Q: What is a veth pair and why is it needed?**

A: A veth pair is a virtual Ethernet cable — two endpoints where packets sent to one end appear at the other. It is needed because network namespaces are fully isolated; there is no way for packets to cross namespace boundaries without an explicit connection. The veth pair provides that connection. One end goes inside the container namespace, the other stays in the host namespace (typically attached to a bridge). This is the fundamental building block of container networking.

**Q: How does cross-host container networking work?**

A: Overlay networks like VXLAN. When Container A on Host 1 sends a packet to Container B on Host 2, the VXLAN device on Host 1 encapsulates the entire L2 frame (including container MAC and IP headers) inside a UDP packet addressed to Host 2. Host 2's VXLAN device strips the outer headers and delivers the inner frame to the bridge, which forwards it to Container B. From the containers' perspective, they are on the same L2 network — the physical network between hosts is invisible.

---

Next: [Zero-Copy I/O and Kernel Bypass](04-zero-copy-io-and-kernel-bypass.md)
