# 2. OSI & TCP/IP Models

## The OSI Model (Open Systems Interconnection)

Developed by **ISO** in 1984 — a **conceptual framework** with **7 layers** that standardizes how different systems communicate.

### Mnemonic to Remember (Top → Bottom)
> **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing
>
> Application → Presentation → Session → Transport → Network → Data Link → Physical

### Mnemonic (Bottom → Top)
> **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

---

## Layer-by-Layer Breakdown

### Layer 7 — Application Layer
- **Closest to the end user**
- Provides network services directly to applications
- **Protocols**: HTTP, HTTPS, FTP, SMTP, DNS, DHCP, SNMP, Telnet, SSH
- **PDU**: Data
- **Example**: Web browser requesting a page

### Layer 6 — Presentation Layer
- **Translation, encryption, compression**
- Converts data formats (e.g., EBCDIC → ASCII)
- Handles **encryption/decryption** (SSL/TLS operates here)
- Data **compression** for efficient transmission
- **Protocols/Formats**: JPEG, GIF, PNG, MPEG, SSL/TLS, ASCII, Unicode
- **PDU**: Data

### Layer 5 — Session Layer
- **Establishes, manages, terminates sessions**
- Handles **authentication & authorization**
- Session **checkpointing & recovery** (if connection drops, resume from checkpoint)
- **Synchronization** using sync points
- **Protocols**: NetBIOS, PPTP, RPC, SCP
- **PDU**: Data

### Layer 4 — Transport Layer
- **End-to-end delivery** & reliability
- **Segmentation & reassembly** of data
- **Flow control** and **error control**
- Multiplexing using **port numbers**
- **Protocols**: TCP (reliable), UDP (fast, unreliable)
- **PDU**: **Segment** (TCP) / **Datagram** (UDP)
- **Devices**: Gateway, Load Balancer

### Layer 3 — Network Layer
- **Logical addressing (IP)** and **routing**
- Determines **best path** from source to destination
- **Packet forwarding** and **fragmentation**
- **Protocols**: IP (IPv4/IPv6), ICMP, ARP, RARP, IGMP, OSPF, BGP, RIP
- **PDU**: **Packet**
- **Devices**: **Router**, Layer 3 Switch

### Layer 2 — Data Link Layer
- **Physical addressing (MAC)** and **framing**
- Error detection (not correction at this level usually)
- Flow control at link level
- **Two sublayers**:
  - **LLC (Logical Link Control)** — error/flow control, multiplexing
  - **MAC (Media Access Control)** — physical addressing, channel access
- **Protocols**: Ethernet (IEEE 802.3), Wi-Fi (IEEE 802.11), PPP, HDLC
- **PDU**: **Frame**
- **Devices**: **Switch**, Bridge

### Layer 1 — Physical Layer
- **Actual transmission of raw bits** over the medium
- Defines electrical, mechanical, and timing specifications
- Deals with cables, connectors, voltages, data rates
- **PDU**: **Bits**
- **Devices**: Hub, Repeater, Modem, Cables

---

## PDU at Each Layer

```
Layer 7 (Application)    → Data
Layer 6 (Presentation)   → Data
Layer 5 (Session)        → Data
Layer 4 (Transport)      → Segment / Datagram
Layer 3 (Network)        → Packet
Layer 2 (Data Link)      → Frame
Layer 1 (Physical)       → Bits
```

### Encapsulation Process (Sending)
```
[Data] 
  → + App/Pres/Sess headers → [Data]
  → + Transport header      → [Segment]
  → + Network header (IP)   → [Packet]
  → + DLL header + trailer  → [Frame]
  → Converted to            → [Bits]
```

### Decapsulation Process (Receiving)
Reverse of encapsulation — headers stripped at each layer.

---

## The TCP/IP Model

Developed by **DARPA** — practical, protocol-based model with **4 layers** (sometimes shown as 5).

### 4-Layer TCP/IP Model

| Layer | Name | OSI Equivalent |
|-------|------|----------------|
| 4 | **Application** | Application + Presentation + Session |
| 3 | **Transport** | Transport |
| 2 | **Internet** | Network |
| 1 | **Network Access** (Host-to-Network) | Data Link + Physical |

### 5-Layer TCP/IP Model (Updated)

| Layer | Name | OSI Equivalent |
|-------|------|----------------|
| 5 | Application | Application + Presentation + Session |
| 4 | Transport | Transport |
| 3 | Network (Internet) | Network |
| 2 | Data Link | Data Link |
| 1 | Physical | Physical |

---

## OSI vs TCP/IP — Comparison

| Feature | OSI Model | TCP/IP Model |
|---------|-----------|--------------|
| **Layers** | 7 | 4 (or 5) |
| **Developed by** | ISO | DARPA/DoD |
| **Approach** | Theoretical / Conceptual | Practical / Implementation-based |
| **Protocols** | Protocol-independent | Protocol-dependent (built around TCP/IP) |
| **Session & Presentation** | Separate layers | Merged into Application |
| **Transport** | Connection-oriented only | Both connection-oriented & connectionless |
| **Network Layer** | Connection + Connectionless | Connectionless (IP) |
| **Reliability** | Defined at Transport layer | End-to-end (TCP) or best-effort (UDP) |
| **Usage** | Reference model | Actual Internet model |
| **Header size** | Variable | Minimum 20 bytes (TCP), 8 bytes (UDP) |

---

## Protocol Stack at Each Layer

```
┌─────────────────────────────────────────────┐
│  APPLICATION    HTTP  FTP  SMTP  DNS  DHCP  │
├─────────────────────────────────────────────┤
│  TRANSPORT      TCP          UDP            │
├─────────────────────────────────────────────┤
│  NETWORK        IP    ICMP   ARP   OSPF    │
├─────────────────────────────────────────────┤
│  DATA LINK      Ethernet   Wi-Fi   PPP     │
├─────────────────────────────────────────────┤
│  PHYSICAL       Cables  Signals  Connectors│
└─────────────────────────────────────────────┘
```

---

## Data Flow Between Layers (How Communication Works)

```
Host A                                    Host B
┌──────────┐                        ┌──────────┐
│ App      │  ← Logical Link →     │ App      │
│ Pres     │  ← Logical Link →     │ Pres     │
│ Session  │  ← Logical Link →     │ Session  │
│ Transport│  ← Logical Link →     │ Transport│
│ Network  │  ← Logical Link →     │ Network  │
│ Data Link│  ← Logical Link →     │ Data Link│
│ Physical │  ←== Actual Link ==→  │ Physical │
└──────────┘                        └──────────┘
```

- Only **Physical Layer** has an actual, direct connection
- All other layers communicate **logically** through their peer layers
- Intermediate devices (routers, switches) only process up to certain layers

---

## How a Router and Switch Process Data

```
Switch operates at Layer 2:
  Frame arrives → Read MAC address → Forward to correct port

Router operates at Layer 3:
  Frame arrives → Strip to Packet → Read IP address → Route → Re-encapsulate into Frame
```

---

## Important Exam Questions

1. **Compare OSI and TCP/IP models** (table format)
2. **Explain the function of each OSI layer with protocols**
3. **What is encapsulation? Explain with diagram**
4. **What are PDUs at each layer?**
5. **Why is OSI called a reference model?**
6. **Explain how data flows from source to destination through the layers**

---

## Quick Revision Checklist

- [ ] All 7 OSI layers — name, function, protocols, PDU, devices
- [ ] TCP/IP model — 4 layers and mapping to OSI
- [ ] Encapsulation & Decapsulation process
- [ ] OSI vs TCP/IP comparison table
- [ ] PDUs: Data → Segment → Packet → Frame → Bits
- [ ] Which devices operate at which layers
- [ ] Peer-to-peer logical communication between layers
