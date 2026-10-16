# 6. Transport Layer

## Role of the Transport Layer

- **Layer 4** of the OSI model
- Provides **end-to-end (process-to-process) delivery**
- Uses **port numbers** to identify applications
- Functions: **Segmentation, Reassembly, Flow Control, Error Control, Congestion Control**
- PDU: **Segment** (TCP) / **Datagram** (UDP)

---

## Port Numbers

| Range | Name | Description | Examples |
|-------|------|-------------|---------|
| 0–1023 | **Well-Known** | Reserved for standard services | HTTP(80), HTTPS(443), FTP(21), SSH(22), DNS(53), SMTP(25) |
| 1024–49151 | **Registered** | Registered with IANA for specific apps | MySQL(3306), PostgreSQL(5432) |
| 49152–65535 | **Dynamic/Private** | Temporary, assigned by OS | Client-side ports |

### Common Port Numbers

| Port | Protocol | Service |
|------|----------|---------|
| 20 | TCP | FTP Data |
| 21 | TCP | FTP Control |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 143 | TCP | IMAP |
| 443 | TCP | HTTPS |

### Socket
- A **socket** = **IP address + Port number**
- Example: `192.168.1.5:80`
- Uniquely identifies a process on a network
- A **connection** is defined by a pair of sockets: (source socket, destination socket)

---

## TCP (Transmission Control Protocol)

### Characteristics
- **Connection-oriented** — establishes connection before data transfer
- **Reliable** — guaranteed delivery, error checking, retransmission
- **Ordered** — data arrives in the correct sequence
- **Flow control** — sliding window mechanism
- **Congestion control** — adjusts sending rate based on network conditions
- **Full-duplex** — both sides can send/receive simultaneously
- **Byte-stream** — treats data as a continuous stream of bytes

### TCP Header (20–60 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset| Rsv |U|A|P|R|S|F|           Window Size              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |        Urgent Pointer         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if any)                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Important TCP Header Fields

| Field | Size | Description |
|-------|------|-------------|
| Source/Dest Port | 16 bits each | Process identification |
| Sequence Number | 32 bits | Byte number of first byte in segment |
| ACK Number | 32 bits | Next byte expected from the other side |
| Data Offset | 4 bits | Header length (in 32-bit words) |
| **Flags** | 6 bits | URG, ACK, PSH, RST, SYN, FIN |
| Window Size | 16 bits | Receiver's buffer space (for flow control) |
| Checksum | 16 bits | Error detection |
| Urgent Pointer | 16 bits | Points to urgent data (when URG flag set) |

### TCP Flags

| Flag | Name | Purpose |
|------|------|---------|
| **SYN** | Synchronize | Initiate connection |
| **ACK** | Acknowledge | Acknowledge received data |
| **FIN** | Finish | Terminate connection |
| **RST** | Reset | Abort connection |
| **PSH** | Push | Push data immediately to application |
| **URG** | Urgent | Data is urgent (use urgent pointer) |

---

## TCP 3-Way Handshake (Connection Establishment)

```
    Client                          Server
      |                                |
      |---- SYN (seq=x) ------------->|    Step 1: Client sends SYN
      |                                |
      |<--- SYN+ACK (seq=y, ack=x+1) -|    Step 2: Server responds SYN+ACK
      |                                |
      |---- ACK (seq=x+1, ack=y+1) -->|    Step 3: Client sends ACK
      |                                |
      |===== CONNECTION ESTABLISHED ===|
```

1. **SYN**: Client → Server (seq = x, SYN flag set)
2. **SYN-ACK**: Server → Client (seq = y, ack = x+1, SYN+ACK flags set)
3. **ACK**: Client → Server (seq = x+1, ack = y+1, ACK flag set)

---

## TCP Connection Termination (4-Way Handshake)

```
    Client                          Server
      |                                |
      |---- FIN (seq=u) ------------->|    Step 1: Client sends FIN
      |                                |
      |<--- ACK (ack=u+1) ------------|    Step 2: Server ACKs
      |                                |
      |<--- FIN (seq=v) --------------|    Step 3: Server sends FIN
      |                                |
      |---- ACK (ack=v+1) ----------->|    Step 4: Client ACKs
      |                                |
      |== TIME_WAIT (2×MSL) → CLOSED =|
```

- Either side can initiate termination
- **TIME_WAIT**: Client waits for **2 × MSL (Maximum Segment Lifetime)** before closing
  - Ensures all segments are delivered and ACK of final FIN reaches server

---

## TCP Flow Control (Sliding Window)

- Receiver advertises **window size** (rwnd) — how much data it can accept
- Sender limits unACKed data to ≤ window size
- Prevents **receiver buffer overflow**

```
Example: Window size = 4 segments

Sender: [1][2][3][4]  [5][6][7]...
         ↑ sent but    ↑ not yet
         unACKed       sent

After ACK for 1,2:
Sender: [1][2] [3][4][5][6]  [7][8]...
         ↑ ACKed  ↑ window     ↑ not sent
                  slides right
```

### Silly Window Syndrome
- Receiver advertises very small window → sender sends tiny segments → inefficient
- **Solutions**:
  - **Nagle's Algorithm** (sender side): Buffer small segments, send when either ACK received or buffer full
  - **Clark's Solution** (receiver side): Don't advertise small windows; wait until buffer is half empty or MSS available

---

## TCP Congestion Control

Controls how much data the sender injects into the network to **avoid congestion**.

### Key Variables
- **cwnd (Congestion Window)**: Sender's limit on unACKed data
- **ssthresh (Slow Start Threshold)**: Threshold between slow start and congestion avoidance
- **Effective window** = min(cwnd, rwnd)

### Phases

#### 1. Slow Start
```
cwnd starts at 1 MSS
Each ACK → cwnd doubles (exponential growth)

Round 1: cwnd = 1 MSS → send 1 segment
Round 2: cwnd = 2 MSS → send 2 segments
Round 3: cwnd = 4 MSS → send 4 segments
Round 4: cwnd = 8 MSS → send 8 segments
...continues until cwnd reaches ssthresh
```

#### 2. Congestion Avoidance
```
cwnd increases by 1 MSS per RTT (linear growth: additive increase)
Continues until congestion detected (packet loss)
```

#### 3. On Congestion Detection

**TCP Tahoe** (older):
```
On ANY loss (timeout or 3 duplicate ACKs):
  → ssthresh = cwnd / 2
  → cwnd = 1 MSS
  → Go to Slow Start
```

**TCP Reno** (improved):
```
On Timeout:
  → ssthresh = cwnd / 2
  → cwnd = 1 MSS
  → Go to Slow Start

On 3 Duplicate ACKs (Fast Retransmit + Fast Recovery):
  → ssthresh = cwnd / 2
  → cwnd = ssthresh (NOT 1)
  → Go to Congestion Avoidance (skip slow start)
```

### AIMD (Additive Increase, Multiplicative Decrease)
- **Additive Increase**: cwnd increases by 1 MSS each RTT (linear) during congestion avoidance
- **Multiplicative Decrease**: cwnd halved on congestion detection
- Creates a **sawtooth pattern** in cwnd over time

### Congestion Control Summary

```
┌─────────────────────────────────────────┐
│         cwnd                            │
│          ↑     /\    /\    /\           │
│          │    /  \  /  \  /  \          │
│ ssthresh │---/----\/----\/----\---       │
│          │  / (congestion        \      │
│          │ /   avoidance)         \     │
│          │/ (slow                       │
│          │   start)                     │
│          └───────────────────────→ time  │
└─────────────────────────────────────────┘
```

---

## UDP (User Datagram Protocol)

### Characteristics
- **Connectionless** — no handshake, no connection setup
- **Unreliable** — no guarantee of delivery, no retransmission
- **Unordered** — packets may arrive out of order
- **No flow control** or **congestion control**
- **Lightweight** — minimal overhead (8-byte header)
- **Fast** — suitable for real-time applications

### UDP Header (8 bytes — fixed)

```
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Size | Description |
|-------|------|-------------|
| Source Port | 16 bits | Sender's port |
| Destination Port | 16 bits | Receiver's port |
| Length | 16 bits | Total datagram size (header + data) |
| Checksum | 16 bits | Error detection (optional in IPv4, mandatory in IPv6) |

---

## TCP vs UDP

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented | Connectionless |
| **Reliability** | Reliable (ACKs, retransmission) | Unreliable |
| **Ordering** | Ordered | Unordered |
| **Speed** | Slower | Faster |
| **Header Size** | 20–60 bytes | 8 bytes |
| **Flow Control** | Yes (sliding window) | No |
| **Congestion Control** | Yes (slow start, AIMD) | No |
| **Broadcasting** | No | Yes |
| **Use Cases** | HTTP, FTP, Email, SSH | DNS, DHCP, VoIP, Streaming, Gaming |
| **Overhead** | High | Low |

---

## Socket Programming (Basics)

### TCP Socket Flow
```
Server:                          Client:
  socket()                        socket()
  bind()                          
  listen()                        
  accept() ← blocks               connect() → 3-way handshake
  read()/write()                  write()/read()
  close()                         close()
```

### UDP Socket Flow
```
Server:                          Client:
  socket()                        socket()
  bind()                          
  recvfrom() ← blocks             sendto()
  sendto()                        recvfrom()
  close()                         close()
```

- TCP uses **streams** (connected sockets)
- UDP uses **datagrams** (`sendto` / `recvfrom` with addresses)

---

## Quick Revision Checklist

- [ ] Transport layer: process-to-process delivery, port numbers, sockets
- [ ] Well-known ports: HTTP(80), HTTPS(443), FTP(21), SSH(22), DNS(53), SMTP(25)
- [ ] TCP characteristics: reliable, ordered, connection-oriented
- [ ] TCP header: Sequence number, ACK number, Flags (SYN/ACK/FIN/RST)
- [ ] 3-Way Handshake: SYN → SYN-ACK → ACK
- [ ] 4-Way Termination: FIN → ACK → FIN → ACK + TIME_WAIT
- [ ] TCP Flow Control: Sliding window, rwnd, Silly Window Syndrome
- [ ] TCP Congestion Control: Slow Start, Congestion Avoidance, AIMD
- [ ] TCP Tahoe vs TCP Reno (3 dup ACKs handling)
- [ ] cwnd, ssthresh, sawtooth pattern
- [ ] UDP characteristics: connectionless, unreliable, fast, 8-byte header
- [ ] TCP vs UDP comparison table
- [ ] Socket programming: TCP (stream) vs UDP (datagram) flow
