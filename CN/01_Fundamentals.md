# 1. Fundamentals of Computer Networks

## What is a Computer Network?

A **computer network** is a collection of interconnected devices (computers, servers, routers, etc.) that can communicate and share resources with each other using a set of rules called **protocols**.

### Why Do We Need Networks?
- **Resource Sharing** — Share printers, files, databases, software
- **Communication** — Email, video conferencing, instant messaging
- **Data Centralization** — Central backup, easier management
- **Cost Efficiency** — Shared resources reduce hardware costs
- **Reliability** — Data replication across multiple machines

---

## Types of Networks

| Type | Full Form | Range | Speed | Example |
|------|-----------|-------|-------|---------|
| **PAN** | Personal Area Network | ~10 meters | Low–Medium | Bluetooth, USB |
| **LAN** | Local Area Network | Building / Campus | High (1 Gbps+) | Office network, College lab |
| **MAN** | Metropolitan Area Network | City-wide | Medium–High | Cable TV network |
| **WAN** | Wide Area Network | Country / Global | Varies | The Internet |

### Other Notable Types
- **CAN (Campus Area Network)** — Multiple LANs within a campus
- **SAN (Storage Area Network)** — High-speed network for storage devices
- **VPN (Virtual Private Network)** — Secure tunnel over a public network

---

## Network Topologies

Topology = the **physical or logical arrangement** of devices in a network.

### 1. Bus Topology
```
[A] --- [B] --- [C] --- [D]
         |
    (Single backbone cable)
```
- All devices share a **single communication line**
- **Advantages**: Simple, low cost, easy to extend
- **Disadvantages**: Single point of failure (backbone), limited cable length, collisions
- **Used in**: Early Ethernet (10BASE2, 10BASE5)

### 2. Star Topology
```
       [A]
        |
[B] -- [HUB/SWITCH] -- [C]
        |
       [D]
```
- All devices connected to a **central hub/switch**
- **Advantages**: Easy to add/remove devices, fault isolation, no collisions (with switch)
- **Disadvantages**: Central device failure = entire network down, more cabling
- **Most commonly used** in modern LANs

### 3. Ring Topology
```
[A] → [B] → [C] → [D] → [A]
```
- Each device connects to **exactly two neighbors**, forming a circle
- Data travels in **one direction** (unidirectional) or both (dual ring)
- **Advantages**: Equal access, predictable performance
- **Disadvantages**: Single break = network failure, difficult to add devices
- **Used in**: Token Ring (IEEE 802.5), FDDI

### 4. Mesh Topology
```
[A] --- [B]
| \   / |
|  \ /  |
|  / \  |
| /   \ |
[C] --- [D]
```
- **Full Mesh**: Every device connected to every other device
  - Links needed = **n(n-1)/2**
- **Partial Mesh**: Some devices connected to all, others to a few
- **Advantages**: High redundancy, fault tolerant, reliable
- **Disadvantages**: Expensive, complex wiring
- **Used in**: WANs, backbone networks

### 5. Hybrid Topology
- Combination of two or more topologies
- Example: Star-Bus, Star-Ring
- **Advantages**: Flexible, scalable
- **Disadvantages**: Complex design, higher cost

### Topology Comparison Table

| Topology | Cost | Reliability | Scalability | Failure Impact |
|----------|------|-------------|-------------|----------------|
| Bus | Low | Low | Limited | High |
| Star | Medium | Medium | Easy | Central device |
| Ring | Medium | Low | Difficult | High |
| Mesh | High | Very High | Difficult | Very Low |
| Hybrid | High | High | Easy | Varies |

---

## Transmission Modes

How data flows between two connected devices.

### 1. Simplex
```
[Sender] ──────→ [Receiver]
```
- **One-direction only** — no reverse communication
- Example: Keyboard → CPU, TV broadcast

### 2. Half-Duplex
```
[Device A] ←────→ [Device B]  (one at a time)
```
- **Both directions**, but **not simultaneously**
- Example: Walkie-talkie

### 3. Full-Duplex
```
[Device A] ←────→ [Device B]  (simultaneously)
```
- **Both directions at the same time**
- Example: Telephone, modern Ethernet

| Mode | Direction | Simultaneous? | Example |
|------|-----------|---------------|---------|
| Simplex | One-way | N/A | Keyboard, TV |
| Half-Duplex | Both ways | No | Walkie-talkie |
| Full-Duplex | Both ways | Yes | Phone, Ethernet |

---

## Key Terminologies

| Term | Meaning |
|------|---------|
| **Node** | Any device on the network (PC, router, printer) |
| **Host** | A device that sends/receives data (end device) |
| **Protocol** | Set of rules for communication (e.g., TCP, HTTP) |
| **Bandwidth** | Maximum data transfer rate (measured in bps) |
| **Throughput** | Actual data transfer rate achieved |
| **Latency** | Time delay for data to travel from source to destination |
| **Jitter** | Variation in latency over time |
| **Packet** | A unit of data transmitted over a network |
| **Port** | A logical endpoint for communication (e.g., port 80 for HTTP) |

---

## Quick Revision Checklist

- [ ] Define computer network and its need
- [ ] PAN vs LAN vs MAN vs WAN — range, speed, examples
- [ ] All 5 topologies — diagrams, pros, cons
- [ ] Bus: single backbone, collisions
- [ ] Star: central hub/switch, most common
- [ ] Ring: unidirectional, token-based
- [ ] Mesh: n(n-1)/2 links, high redundancy
- [ ] Simplex vs Half-Duplex vs Full-Duplex with examples
- [ ] Key terms: bandwidth vs throughput vs latency
