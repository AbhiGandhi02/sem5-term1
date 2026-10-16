# 5. Network Layer

## Role of the Network Layer

- **Layer 3** of the OSI model
- Responsible for **source-to-destination delivery** across multiple networks
- Functions: **Logical addressing (IP), Routing, Fragmentation, Packet forwarding**
- PDU: **Packet**
- Devices: **Router**, Layer 3 Switch

---

## IP Addressing

### IPv4 Address
- **32 bits** (4 bytes), written in **dotted decimal** notation
- Example: `192.168.1.1` → `11000000.10101000.00000001.00000001`
- Total addresses: **2^32 = ~4.3 billion**

### Classful Addressing

| Class | First Octet Range | Default Subnet Mask | Network Bits | Host Bits | # Networks | # Hosts/Network |
|-------|-------------------|---------------------|-------------|-----------|------------|-----------------|
| **A** | 1–126 | 255.0.0.0 (/8) | 8 | 24 | 126 | 16,777,214 |
| **B** | 128–191 | 255.255.0.0 (/16) | 16 | 16 | 16,384 | 65,534 |
| **C** | 192–223 | 255.255.255.0 (/24) | 24 | 8 | 2,097,152 | 254 |
| **D** | 224–239 | — | — | — | Multicast | — |
| **E** | 240–255 | — | — | — | Reserved/Experimental | — |

> **Note**: 127.x.x.x is reserved for **loopback** (127.0.0.1 = localhost)

### Special IP Addresses

| Address | Purpose |
|---------|---------|
| `0.0.0.0` | Default route / "this network" |
| `127.0.0.1` | Loopback (localhost) |
| `255.255.255.255` | Limited broadcast |
| `x.x.x.0` | Network address (host bits all 0) |
| `x.x.x.255` | Broadcast address (host bits all 1) |

### Private IP Ranges (RFC 1918)

| Class | Range | CIDR |
|-------|-------|------|
| A | 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 |
| B | 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 |
| C | 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 |

---

## Subnetting

Dividing a large network into **smaller sub-networks** for better management and security.

### Key Concepts
- **Subnet Mask**: Identifies network vs host portion
- **CIDR Notation**: `/n` where n = number of network bits (e.g., /24)
- **Subnet bits** are borrowed from the **host portion**

### Subnetting Formulas
```
Number of subnets     = 2^s  (s = subnet bits borrowed)
Number of hosts/subnet = 2^h - 2  (h = remaining host bits; subtract network & broadcast)
Block size (jump)     = 256 - subnet mask octet value
```

### Subnetting Example
```
Given: 192.168.1.0/24, need 4 subnets

Subnet bits needed: 2 (since 2^2 = 4)
New mask: /26 → 255.255.255.192
Hosts per subnet: 2^6 - 2 = 62
Block size: 256 - 192 = 64

Subnets:
  192.168.1.0/26    → Hosts: .1 to .62,   Broadcast: .63
  192.168.1.64/26   → Hosts: .65 to .126, Broadcast: .127
  192.168.1.128/26  → Hosts: .129 to .190, Broadcast: .191
  192.168.1.192/26  → Hosts: .193 to .254, Broadcast: .255
```

---

## CIDR (Classless Inter-Domain Routing)

- Replaces classful addressing
- Uses **variable-length subnet masking (VLSM)**
- Notation: `IP/prefix` (e.g., `10.0.0.0/12`)
- Enables **route aggregation (supernetting)** — combining multiple networks into one route

### Supernetting
- **Combining** multiple smaller networks into a single larger one
- Used for **route summarization**
```
Example: Combine 4 Class C networks
  192.168.0.0/24
  192.168.1.0/24
  192.168.2.0/24
  192.168.3.0/24
  
Supernet: 192.168.0.0/22
```

---

## IPv6

- **128-bit** addresses (vs 32-bit in IPv4)
- Written in **hexadecimal**, separated by colons: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- **Simplified** by removing leading zeros and replacing consecutive zero groups with `::`
  - `2001:db8:85a3::8a2e:370:7334`
- Total addresses: **2^128 ≈ 3.4 × 10^38**

### IPv4 vs IPv6

| Feature | IPv4 | IPv6 |
|---------|------|------|
| **Address Size** | 32 bits | 128 bits |
| **Notation** | Dotted decimal | Hexadecimal, colon-separated |
| **Address Space** | ~4.3 billion | ~3.4 × 10^38 |
| **Header Size** | 20–60 bytes | Fixed 40 bytes |
| **Checksum** | Yes | No (handled by upper layers) |
| **Fragmentation** | Routers & sender | Only sender |
| **Broadcast** | Yes | No (uses multicast) |
| **ARP** | Yes | Replaced by NDP |
| **NAT** | Common | Not needed |
| **IPSec** | Optional | Built-in |

---

## Routing

### Types of Routing
1. **Static Routing**: Routes manually configured by admin
   - Simple, low overhead, but doesn't adapt to changes
2. **Dynamic Routing**: Routes learned automatically via routing protocols
   - Adapts to topology changes, more overhead
3. **Default Routing**: All traffic not matching specific routes sent to a default gateway

### Routing Algorithms

#### 1. Distance Vector Routing (Bellman-Ford Algorithm)
- Each router maintains a **distance table** (destination, cost, next hop)
- Routers share tables with **direct neighbors** periodically
- **Convergence**: Slow — information spreads hop by hop
- **Problems**:
  - **Count to Infinity** — routing loops cause costs to increment indefinitely
  - **Solutions**: Split Horizon, Poison Reverse, Hold-down timers
- **Protocol**: **RIP**

#### 2. Link State Routing (Dijkstra's Algorithm)
- Each router discovers **all network topology**
- Builds a **complete map** of the network
- Computes **shortest path** using Dijkstra's algorithm
- **Steps**:
  1. Discover neighbors (Hello packets)
  2. Measure cost to each neighbor
  3. Broadcast **LSA (Link State Advertisement)** to all routers
  4. Build **topology database** from all LSAs
  5. Run **Dijkstra's SPF** algorithm to compute shortest paths
- **Protocol**: **OSPF**

#### 3. Path Vector Routing
- Each router maintains **full path** (list of ASes) to each destination
- Prevents loops by checking if own AS appears in path
- **Protocol**: **BGP**

### Comparison of Routing Algorithms

| Feature | Distance Vector | Link State | Path Vector |
|---------|----------------|------------|-------------|
| **Algorithm** | Bellman-Ford | Dijkstra | Path-based |
| **Knowledge** | Neighbors only | Full topology | Full AS path |
| **Updates** | Periodic, full table | Event-driven, LSAs | Incremental |
| **Convergence** | Slow | Fast | Medium |
| **Scalability** | Low | Medium | High |
| **Loop Prevention** | Split Horizon | SPF tree | Path check |
| **Protocol** | RIP | OSPF | BGP |

---

## Routing Protocols

### RIP (Routing Information Protocol)
- **Type**: Distance Vector
- **Metric**: Hop count (max = 15; 16 = unreachable)
- **Updates**: Every 30 seconds (full routing table)
- **Algorithm**: Bellman-Ford
- **Administrative Distance**: 120
- **Versions**: RIPv1 (classful), RIPv2 (classless, supports VLSM)

### OSPF (Open Shortest Path First)
- **Type**: Link State
- **Metric**: Cost (based on bandwidth: Cost = 10^8 / bandwidth)
- **Updates**: Event-driven (only when topology changes)
- **Algorithm**: Dijkstra (SPF)
- **Administrative Distance**: 110
- Uses **areas** for hierarchical design (Area 0 = backbone)
- Supports **VLSM** and **CIDR**
- Uses **Hello packets** for neighbor discovery (every 10 sec)

### BGP (Border Gateway Protocol)
- **Type**: Path Vector
- **Purpose**: Inter-AS (Autonomous System) routing — **the Internet routing protocol**
- **Metric**: Path attributes (AS path, next hop, etc.)
- Uses **TCP port 179**
- **Types**: eBGP (between ASes), iBGP (within AS)
- **Administrative Distance**: eBGP = 20, iBGP = 200

### Comparison

| Feature | RIP | OSPF | BGP |
|---------|-----|------|-----|
| **Type** | Distance Vector | Link State | Path Vector |
| **Metric** | Hop count | Cost (bandwidth) | AS path + attributes |
| **Max hops** | 15 | None | None |
| **Convergence** | Slow | Fast | Slow |
| **Scope** | Intra-AS (small) | Intra-AS (large) | Inter-AS |
| **Transport** | UDP 520 | IP Protocol 89 | TCP 179 |
| **AD** | 120 | 110 | 20/200 |

---

## Supporting Protocols

### ARP (Address Resolution Protocol)
- Maps **IP address → MAC address**
- Broadcasts ARP request, unicast ARP reply
- Maintains **ARP cache/table**

### RARP (Reverse ARP)
- Maps **MAC address → IP address**
- Used by diskless workstations (obsolete, replaced by DHCP)

### ICMP (Internet Control Message Protocol)
- Used for **error reporting and diagnostics**
- **Not used for data transfer**
- **Types**:
  - Type 0: Echo Reply (ping response)
  - Type 3: Destination Unreachable
  - Type 5: Redirect
  - Type 8: Echo Request (ping)
  - Type 11: Time Exceeded (TTL expired — used by traceroute)
- **Tools**: `ping` (uses Echo Request/Reply), `traceroute` (uses TTL expiry)

### NAT (Network Address Translation)
- Translates **private IP ↔ public IP** at router
- Allows multiple devices to share a **single public IP**
- **Types**:
  - **Static NAT**: 1:1 mapping (private ↔ public)
  - **Dynamic NAT**: Pool of public IPs
  - **PAT (Port Address Translation) / NAT Overload**: Multiple private IPs mapped to one public IP using different port numbers (most common)
- **Advantages**: Conserves public IPs, adds security layer
- **Disadvantages**: Breaks end-to-end connectivity, issues with some protocols

---

## IPv4 Header (20–60 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |    DSCP   |ECN|         Total Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|     Fragment Offset     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    TTL    |    Protocol   |       Header Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source IP Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination IP Address                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if any)                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Important Fields
| Field | Size | Description |
|-------|------|-------------|
| Version | 4 bits | IPv4 = 4, IPv6 = 6 |
| IHL | 4 bits | Header length in 32-bit words (min 5 = 20 bytes) |
| Total Length | 16 bits | Total packet size (header + data), max 65535 bytes |
| TTL | 8 bits | Time To Live — decremented by 1 at each router |
| Protocol | 8 bits | Upper layer protocol (6=TCP, 17=UDP, 1=ICMP) |
| Source/Dest IP | 32 bits each | Source and destination addresses |
| Flags | 3 bits | DF (Don't Fragment), MF (More Fragments) |
| Fragment Offset | 13 bits | Position of fragment in original packet (in 8-byte units) |

---

## Quick Revision Checklist

- [ ] IPv4: 32-bit, dotted decimal, classful classes A–E
- [ ] Private IP ranges: 10.x, 172.16–31.x, 192.168.x
- [ ] Subnetting: formulas, example calculation
- [ ] CIDR & VLSM & Supernetting
- [ ] IPv6: 128-bit, hex notation, key differences from IPv4
- [ ] Distance Vector (Bellman-Ford) → RIP
- [ ] Link State (Dijkstra) → OSPF
- [ ] Path Vector → BGP
- [ ] RIP vs OSPF vs BGP comparison
- [ ] ARP, RARP, ICMP, NAT (and PAT)
- [ ] IPv4 header fields (especially TTL, Protocol, Flags, Fragment Offset)
- [ ] Count to Infinity problem and solutions
