# 8. Network Devices

## Overview

Network devices operate at different layers of the OSI model and serve specific purposes in data communication.

```
Layer 7 (Application)  → Gateway (Application-level)
Layer 4 (Transport)    → Gateway, Load Balancer
Layer 3 (Network)      → Router, Layer 3 Switch
Layer 2 (Data Link)    → Switch, Bridge
Layer 1 (Physical)     → Hub, Repeater, Modem
```

---

## Layer 1 Devices (Physical Layer)

### Hub
- **Broadcasts** incoming data to **all ports** (except the source port)
- Does **not inspect** frames or addresses
- **No intelligence** — cannot filter or forward selectively
- Creates a **single collision domain** (all ports share bandwidth)
- Creates a **single broadcast domain**
- **Types**:
  - **Passive Hub**: Just a connector, no signal regeneration
  - **Active Hub**: Regenerates/amplifies signal
  - **Intelligent Hub**: Includes management features
- **Largely obsolete** — replaced by switches
- **Half-duplex** operation

### Repeater
- **Regenerates and amplifies** weak signals
- Extends the **distance** of a network segment
- Operates on **bits** — does not understand frames or addresses
- **Two-port device** (input → amplify → output)
- Does not filter traffic
- **Use**: Extending cable runs beyond maximum distance

### Modem (Modulator-Demodulator)
- Converts **digital signals ↔ analog signals**
- **Modulator**: Digital → Analog (for transmission over phone lines)
- **Demodulator**: Analog → Digital (for receiving)
- **Types**: DSL modem, cable modem, fiber modem (ONT)
- Required for Internet access over telephone/cable lines

---

## Layer 2 Devices (Data Link Layer)

### Bridge
- Connects **two or more LAN segments**
- Operates on **MAC addresses**
- Maintains a **MAC address table** (learns which devices are on which segment)
- **Filters traffic** — only forwards frames destined for another segment
- **Divides collision domains** (each port = separate collision domain)
- Single **broadcast domain** (broadcasts still forwarded to all segments)
- **Two ports** typically
- Largely replaced by switches

### Switch (Layer 2 Switch)
- **Multi-port bridge** — much smarter and faster
- Operates on **MAC addresses**
- Maintains a **MAC address table (CAM table)**
- **Forwards frames** only to the specific destination port
- **Each port = separate collision domain**
- **All ports = single broadcast domain** (unless VLANs used)
- **Full-duplex** operation
- **Microsecond latency**

#### How a Switch Works
```
1. Frame arrives on port 1 with:
   Source MAC: AA:AA:AA:AA:AA:AA
   Dest MAC:   BB:BB:BB:BB:BB:BB

2. Switch learns: "AA is on port 1" → adds to MAC table

3. Switch looks up BB in MAC table:
   - Found on port 3 → forward to port 3 ONLY
   - Not found → FLOOD to all ports (except port 1)

4. When BB replies, switch learns "BB is on port 3"
```

#### Switch Forwarding Methods
| Method | Description | Speed | Reliability |
|--------|-------------|-------|-------------|
| **Store-and-Forward** | Receives entire frame, checks CRC, then forwards | Slowest | Highest |
| **Cut-Through** | Reads only dest MAC (first 6 bytes), forwards immediately | Fastest | Lowest |
| **Fragment-Free** | Reads first 64 bytes (min frame size), then forwards | Medium | Medium |

#### VLANs (Virtual LANs)
- **Logically segment** a physical switch into multiple broadcast domains
- Devices in different VLANs **cannot communicate** without a router (Layer 3)
- **Benefits**: Security, broadcast control, flexibility
- Configured on **managed switches**
- **VLAN Tagging** (IEEE 802.1Q): Adds 4-byte tag to frame for VLAN identification

---

## Layer 3 Devices (Network Layer)

### Router
- Operates on **IP addresses** (Layer 3)
- Connects **different networks** (e.g., LAN to WAN, LAN to LAN)
- Makes forwarding decisions using **routing table**
- **Each port = separate collision domain AND broadcast domain**
- Performs **NAT**, **DHCP**, **firewall** functions in many cases
- Uses **routing protocols** (RIP, OSPF, BGP) to build routing tables

#### How a Router Works
```
1. Packet arrives on interface eth0
2. Router strips Layer 2 header (frame)
3. Reads destination IP address
4. Looks up routing table for best match
5. Determines next hop and outgoing interface
6. Decrements TTL (if TTL=0, discard + send ICMP Time Exceeded)
7. Re-encapsulates with new Layer 2 header (new source/dest MAC)
8. Forwards out the appropriate interface
```

#### Routing Table Entry
```
Destination Network | Subnet Mask | Next Hop | Interface | Metric
192.168.1.0        | /24         | 10.0.0.1 | eth0      | 10
0.0.0.0            | /0          | 10.0.0.1 | eth0      | 1    (default route)
```

### Layer 3 Switch (Multilayer Switch)
- Combines **switch (Layer 2) + router (Layer 3)** functionality
- Can route between VLANs **without an external router**
- **Hardware-based routing** → faster than traditional routers
- Used in **enterprise networks** for inter-VLAN routing

---

## Layer 4+ Devices

### Gateway
- Operates at **multiple layers** (up to Layer 7)
- Translates between **different protocols** or **network architectures**
- Examples:
  - Email gateway (SMTP ↔ X.400)
  - VoIP gateway (analog ↔ digital)
  - API gateway
- **"Default Gateway"** in networking = the router that connects your network to others

### Load Balancer
- Distributes incoming traffic across **multiple servers**
- Operates at **Layer 4** (TCP/UDP) or **Layer 7** (HTTP)
- **Algorithms**: Round Robin, Least Connections, Weighted, IP Hash
- Improves **availability, scalability, performance**

### Firewall
- **Filters traffic** based on security rules
- Can operate at multiple layers:
  - **Packet Filter** (Layer 3/4): IP addresses, ports
  - **Stateful Inspection** (Layer 4): Tracks connection state
  - **Application Firewall** (Layer 7): Inspects application data
- **Types**: Hardware firewall, Software firewall, Next-Gen Firewall (NGFW)

### Access Point (AP)
- Provides **wireless connectivity** (Wi-Fi) to a wired network
- Operates at **Layer 2**
- Acts as a **bridge** between wireless devices and the wired LAN

---

## Collision Domain vs Broadcast Domain

### Collision Domain
- A network segment where **data collisions can occur**
- Two devices transmitting simultaneously → collision
- Devices: Hub = 1 collision domain; Switch/Bridge = 1 per port; Router = 1 per port

### Broadcast Domain
- A network segment where a **broadcast reaches all devices**
- Devices: Hub/Switch = 1 broadcast domain; Router = 1 per port (blocks broadcasts)

### Summary Table

| Device | Collision Domains | Broadcast Domains | Operates at |
|--------|-------------------|-------------------|-------------|
| **Hub** | 1 (shared) | 1 | Layer 1 |
| **Repeater** | 1 (shared) | 1 | Layer 1 |
| **Bridge** | 1 per port | 1 | Layer 2 |
| **Switch** | 1 per port | 1 (or per VLAN) | Layer 2 |
| **Router** | 1 per port | 1 per port | Layer 3 |

### Example Calculation
```
Network: 1 Router with 3 ports → 3 Switches → each Switch has 4 hosts

Collision Domains:
  Each switch port = separate collision domain
  3 switches × 4 host ports + 3 router ports = 12 + 3 = 15

Broadcast Domains:
  Router separates broadcast domains
  = 3 (one per router port/interface)
```

---

## Complete Device Comparison

| Feature | Hub | Switch | Router |
|---------|-----|--------|--------|
| **OSI Layer** | 1 | 2 | 3 |
| **Address** | None | MAC | IP |
| **Intelligence** | None | MAC table | Routing table |
| **Collision Domain** | 1 (all ports) | 1 per port | 1 per port |
| **Broadcast Domain** | 1 | 1 | 1 per port |
| **Speed** | Shared | Dedicated | Dedicated |
| **Duplex** | Half | Full | Full |
| **Filtering** | No | Yes (MAC) | Yes (IP) |
| **Cost** | Low | Medium | High |
| **Use** | Obsolete | LAN | Internetworking |

---

## Quick Revision Checklist

- [ ] Hub: Layer 1, broadcasts to all, single collision domain, obsolete
- [ ] Repeater: Layer 1, regenerates signal, extends distance
- [ ] Bridge: Layer 2, connects LAN segments, MAC-based, 2 ports
- [ ] Switch: Layer 2, multi-port bridge, MAC table, per-port collision domain
- [ ] Switch forwarding: Store-and-Forward vs Cut-Through vs Fragment-Free
- [ ] VLANs: logical segmentation, 802.1Q tagging
- [ ] Router: Layer 3, IP-based, separate broadcast domains, routing table
- [ ] Layer 3 Switch: Switch + Router, inter-VLAN routing
- [ ] Gateway: protocol translator, multi-layer
- [ ] Firewall: packet filter, stateful, application-level
- [ ] Collision domain vs Broadcast domain with calculations
- [ ] Hub vs Switch vs Router comparison table
