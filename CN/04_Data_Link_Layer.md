# 4. Data Link Layer

## Role of the Data Link Layer

- **Layer 2** of the OSI model
- Provides **node-to-node (hop-to-hop) delivery** of data
- Responsible for: **Framing, Error Control, Flow Control, Access Control**
- PDU: **Frame**
- Addressing: **MAC Address** (48-bit, hardware address)

### Two Sublayers
1. **LLC (Logical Link Control)** — Upper sublayer
   - Multiplexing protocols, flow control, error control
   - Defined by IEEE 802.2
2. **MAC (Media Access Control)** — Lower sublayer
   - Physical addressing (MAC address), channel access control
   - Defined by IEEE 802.3 (Ethernet), 802.11 (Wi-Fi), etc.

---

## Framing

Process of **encapsulating network layer packets** into frames by adding header and trailer.

### Frame Structure
```
┌──────────┬──────────┬─────────┬───────────┬─────────┐
│ Preamble │  Header  │ Payload │  Trailer  │   IFG   │
│          │ (Dest MAC│ (Data)  │ (CRC/FCS) │         │
│          │  Src MAC │         │           │         │
│          │  Type)   │         │           │         │
└──────────┴──────────┴─────────┴───────────┘─────────┘
```

### Types of Framing
1. **Character/Byte Count** — First field in frame specifies total bytes
   - Problem: If count corrupted, synchronization lost
2. **Character/Byte Stuffing** — Special flag bytes mark start/end
   - Flag: `01111110` — if flag appears in data, escape character inserted
3. **Bit Stuffing** — Flag pattern `01111110` marks boundaries
   - If five consecutive 1s in data, a `0` is stuffed after them
   - Receiver removes the stuffed bit
4. **Physical Layer Coding Violations** — Use invalid signal patterns as delimiters

---

## Error Detection & Correction

### Types of Errors
- **Single-bit error**: Only 1 bit changed
- **Burst error**: Multiple consecutive bits changed

### Error Detection Methods

#### 1. Parity Check
- **Simple (1D) Parity**: Add 1 extra bit to make total 1s even (even parity) or odd (odd parity)
  - Detects **odd number** of bit errors only
  - Cannot detect even number of errors
- **2D Parity**: Arrange data in matrix, add parity for rows AND columns
  - Can detect AND **correct single-bit errors**
  - Can detect (but not correct) burst errors

#### 2. Checksum
- Data divided into **fixed-size segments** (usually 16-bit)
- All segments **added together** using 1's complement arithmetic
- **Complement of the sum** = checksum
- Receiver adds all segments + checksum → result should be all 1s
- **Used in**: IP, TCP, UDP headers
- Can detect errors but **cannot correct** them

#### 3. CRC (Cyclic Redundancy Check) ⭐
- Most powerful error detection method at Data Link Layer
- **Steps**:
  1. Agree on a **generator polynomial** (divisor) of degree `r`
  2. Append `r` zero bits to the data
  3. Perform **binary division** (XOR) of (data + zeros) by the generator
  4. **Remainder** = CRC bits → append to original data
  5. Receiver divides received frame by generator — if remainder = 0, no error
- **Generator Polynomials** (common):
  - CRC-8, CRC-16, CRC-32 (used in Ethernet), CRC-CCITT
- Can detect: single-bit, double-bit, odd number of errors, burst errors ≤ r bits

#### CRC Example
```
Data:       1101011011
Generator:  10011 (degree 4, so r=4)

Step 1: Append 4 zeros → 11010110110000
Step 2: Divide by 10011 using XOR
Step 3: Remainder = 1110
Step 4: Transmitted frame = 11010110111110
Step 5: Receiver divides by 10011, remainder = 0 → No error
```

### Error Correction

#### Hamming Code
- Can **detect and correct single-bit errors**, detect double-bit errors
- **Redundant bits** placed at positions that are powers of 2 (1, 2, 4, 8, ...)
- **Number of redundant bits `r`**: 2^r ≥ m + r + 1 (where m = data bits)
- Each parity bit covers specific bit positions based on binary representation

#### Hamming Code Steps
```
Data bits (m=4): D3 D2 D1 D0

Position:  1   2   3   4   5   6   7
           P1  P2  D3  P4  D2  D1  D0
                        
P1 checks: positions with bit 0 set (1,3,5,7) → 1,3,5,7
P2 checks: positions with bit 1 set (2,3,6,7) → 2,3,6,7
P4 checks: positions with bit 2 set (4,5,6,7) → 4,5,6,7
```

- **Syndrome** = binary number formed by parity check results
- If syndrome = 0 → no error; otherwise syndrome = position of error bit

#### Hamming Distance
- Number of **bit positions** where two codewords differ
- **Minimum Hamming distance `d`**:
  - To **detect** up to `k` errors: d ≥ k + 1
  - To **correct** up to `k` errors: d ≥ 2k + 1

---

## Flow Control

Mechanism to **prevent sender from overwhelming the receiver**.

### 1. Stop-and-Wait
```
Sender              Receiver
  |--- Frame 0 ---→|
  |←---- ACK -------|
  |--- Frame 1 ---→|
  |←---- ACK -------|
```
- Send **one frame**, wait for **ACK**, then send next
- **Simple** but **very inefficient** (low utilization)
- **Utilization** = 1 / (1 + 2a), where a = propagation delay / transmission delay
- Uses **sequence numbers 0 and 1** (alternating)

### 2. Sliding Window Protocol
- Allows **multiple frames** to be in transit before receiving ACK
- **Window size** = number of frames that can be sent without waiting for ACK

#### a) Go-Back-N (GBN)
```
Window size (sender) = 2^n - 1 (where n = number of sequence bits)
Window size (receiver) = 1
```
- Sender can send up to **N frames** without waiting
- Receiver only accepts frames **in order**
- If error in frame `i`:
  - Receiver **discards frame i and all subsequent frames**
  - Sender **retransmits frame i and all subsequent frames** (go back to i)
- Uses **cumulative ACK**
- **Timer**: One timer for oldest unACKed frame

#### b) Selective Repeat (SR)
```
Window size (sender) = 2^(n-1)
Window size (receiver) = 2^(n-1)
```
- Sender retransmits **only the erroneous frame**
- Receiver accepts **out-of-order frames** and buffers them
- Uses **individual ACK** for each frame
- More efficient than GBN but more **complex**
- Receiver needs **buffer** to store out-of-order frames

### Comparison

| Feature | Stop-and-Wait | Go-Back-N | Selective Repeat |
|---------|--------------|-----------|-----------------|
| **Sender Window** | 1 | 2^n - 1 | 2^(n-1) |
| **Receiver Window** | 1 | 1 | 2^(n-1) |
| **ACK Type** | Individual | Cumulative | Individual |
| **Retransmission** | Single frame | Frame + all after | Only errored frame |
| **Efficiency** | Low | Medium | High |
| **Complexity** | Low | Medium | High |
| **Buffer at Receiver** | No | No | Yes |

### Efficiency Formulas
```
a = Tp / Tt  (propagation time / transmission time)

Stop-and-Wait:    η = 1 / (1 + 2a)
Go-Back-N:        η = N / (1 + 2a)       if N < (1 + 2a)
                  η = 1                   if N ≥ (1 + 2a)
Selective Repeat: η = N / (1 + 2a)       if N < (1 + 2a)
                  η = 1                   if N ≥ (1 + 2a)
```

---

## MAC (Media Access Control) Protocols

How multiple devices **share a common communication channel**.

### Classification
```
MAC Protocols
├── Random Access (Contention)
│   ├── ALOHA (Pure & Slotted)
│   ├── CSMA
│   ├── CSMA/CD
│   └── CSMA/CA
├── Controlled Access
│   ├── Polling
│   ├── Token Passing
│   └── Reservation
└── Channelization
    ├── FDMA
    ├── TDMA
    └── CDMA
```

### Random Access Protocols

#### 1. Pure ALOHA
- Station transmits **whenever it has data**
- If **collision** → wait random time → retransmit
- **Vulnerable period**: 2 × Frame time
- **Maximum throughput**: **18.4%** (S = G × e^(-2G), max at G = 0.5)

#### 2. Slotted ALOHA
- Time divided into **equal slots**; transmission only at **slot boundaries**
- **Vulnerable period**: 1 × Frame time
- **Maximum throughput**: **36.8%** (S = G × e^(-G), max at G = 1)
- **Double** the efficiency of Pure ALOHA

#### 3. CSMA (Carrier Sense Multiple Access)
- **Listen before transmitting** ("sense the carrier")
- **Types**:
  - **1-Persistent**: If idle → transmit immediately. If busy → wait and transmit as soon as idle
    - Most aggressive, highest collision probability
  - **Non-Persistent**: If idle → transmit. If busy → wait random time, then sense again
    - Less collision but more delay
  - **P-Persistent**: If idle → transmit with probability `p`, defer with probability `1-p`
    - Compromise between the two

#### 4. CSMA/CD (Collision Detection) — Wired (Ethernet)
- **Listen + Transmit + Detect collisions**
- If collision detected during transmission:
  1. Stop transmitting
  2. Send **jam signal** (48 bits)
  3. Wait **random backoff** time (Binary Exponential Backoff)
  4. Retry
- **Minimum frame size** = 2 × Propagation delay × Bandwidth
  - For Ethernet: 64 bytes (512 bits at 10 Mbps)
- **Used in**: IEEE 802.3 (Ethernet)
- **Cannot be used in wireless** (hidden terminal problem)

#### 5. CSMA/CA (Collision Avoidance) — Wireless (Wi-Fi)
- Collisions **cannot be detected** in wireless → try to **avoid** them
- **Mechanism**:
  1. Sense channel — if idle, wait **DIFS** (Distributed Inter-Frame Space)
  2. If still idle → transmit
  3. If busy → random backoff
  4. Receiver sends **ACK** after **SIFS** (Short Inter-Frame Space)
- **Optional RTS/CTS** (Request to Send / Clear to Send) to handle hidden terminal
- **Used in**: IEEE 802.11 (Wi-Fi)

### CSMA/CD vs CSMA/CA

| Feature | CSMA/CD | CSMA/CA |
|---------|---------|---------|
| **Full Form** | Collision Detection | Collision Avoidance |
| **Medium** | Wired (Ethernet) | Wireless (Wi-Fi) |
| **Standard** | IEEE 802.3 | IEEE 802.11 |
| **Collision** | Detected during TX | Avoided before TX |
| **ACK** | Not used | Required |
| **RTS/CTS** | Not used | Optional |

---

## Ethernet (IEEE 802.3)

### Ethernet Frame Format
```
┌──────────┬─────┬──────────┬──────────┬──────┬─────────┬─────┐
│ Preamble │ SFD │ Dest MAC │ Src MAC  │ Type │  Data   │ FCS │
│ 7 bytes  │ 1B  │ 6 bytes  │ 6 bytes  │ 2B   │46-1500B │ 4B  │
└──────────┴─────┴──────────┴──────────┴──────┴─────────┴─────┘
```
- **Preamble**: 7 bytes of `10101010` — synchronization
- **SFD**: `10101011` — marks start of frame
- **Dest/Src MAC**: 48-bit (6 bytes) hardware addresses
- **Type/Length**: Protocol type (e.g., 0x0800 = IPv4)
- **Data**: 46–1500 bytes (min 46, max 1500 = MTU)
- **FCS**: Frame Check Sequence (CRC-32)
- **Minimum frame size**: 64 bytes (excluding preamble and SFD)

### MAC Address
- **48 bits** (6 bytes), written as `AA:BB:CC:DD:EE:FF`
- First 24 bits = **OUI** (Organizationally Unique Identifier) — manufacturer
- Last 24 bits = **Device-specific**
- **Broadcast MAC**: `FF:FF:FF:FF:FF:FF`
- **Burned into NIC** (Network Interface Card)

### Ethernet Standards

| Standard | Speed | Cable | Max Distance |
|----------|-------|-------|-------------|
| 10BASE-T | 10 Mbps | UTP Cat3 | 100m |
| 100BASE-TX | 100 Mbps | UTP Cat5 | 100m |
| 1000BASE-T | 1 Gbps | UTP Cat5e/6 | 100m |
| 10GBASE-T | 10 Gbps | UTP Cat6a/7 | 100m |
| 1000BASE-SX | 1 Gbps | Multimode Fiber | 550m |
| 1000BASE-LX | 1 Gbps | Single-mode Fiber | 5 km |

---

## Quick Revision Checklist

- [ ] DLL responsibilities: Framing, Error control, Flow control, Access control
- [ ] LLC vs MAC sublayers
- [ ] Framing: Byte stuffing vs Bit stuffing
- [ ] Error Detection: Parity, Checksum, CRC (steps with example)
- [ ] Hamming Code: redundant bit positions, syndrome calculation
- [ ] Hamming Distance: detect (d ≥ k+1), correct (d ≥ 2k+1)
- [ ] Stop-and-Wait: simple, η = 1/(1+2a)
- [ ] Go-Back-N: window = 2^n - 1, cumulative ACK, retransmit all from error
- [ ] Selective Repeat: window = 2^(n-1), individual ACK, retransmit only errored
- [ ] ALOHA: Pure (18.4%) vs Slotted (36.8%)
- [ ] CSMA types: 1-persistent, non-persistent, p-persistent
- [ ] CSMA/CD (Ethernet, wired) vs CSMA/CA (Wi-Fi, wireless)
- [ ] Ethernet frame format, MAC address structure
- [ ] Minimum frame size = 64 bytes
