# 3. Physical Layer

## Role of the Physical Layer

- **Layer 1** of the OSI model
- Responsible for the **transmission of raw bits** (0s and 1s) over a physical medium
- Defines **electrical, mechanical, functional, and procedural** specifications
- Deals with: cables, connectors, voltages, bit rates, signal encoding

---

## Transmission Media

The physical path between transmitter and receiver.

### A. Guided (Wired) Media

#### 1. Twisted Pair Cable
- Two insulated copper wires **twisted together** to reduce electromagnetic interference (crosstalk)
- **Types**:
  - **UTP (Unshielded Twisted Pair)** — no shielding, cheaper, used in LANs
    - Categories: Cat5 (100 Mbps), Cat5e (1 Gbps), Cat6 (10 Gbps), Cat6a, Cat7
  - **STP (Shielded Twisted Pair)** — metallic shield, better noise protection, costlier
- **Connectors**: RJ-45 (Ethernet), RJ-11 (Telephone)
- **Max distance**: ~100 meters (for Ethernet)
- **Use**: LANs, telephone lines

#### 2. Coaxial Cable
- Central copper conductor surrounded by insulation, metallic shield, and outer jacket
- **Better shielding** than twisted pair
- **Types**:
  - **Thin coax (10BASE2)** — 185m, flexible
  - **Thick coax (10BASE5)** — 500m, rigid
- **Connectors**: BNC (Bayonet Neill-Concelman)
- **Bandwidth**: Higher than twisted pair
- **Use**: Cable TV, older Ethernet, broadband internet

#### 3. Fiber Optic Cable
- Transmits data as **light pulses** through glass/plastic fibers
- **Types**:
  - **Single-mode fiber (SMF)** — narrow core (~9μm), laser source, long distance (up to 100 km), higher bandwidth
  - **Multi-mode fiber (MMF)** — wider core (~50-62.5μm), LED source, shorter distance (~2 km), cheaper
- **Connectors**: SC, ST, LC
- **Advantages**: Very high bandwidth, immune to EMI, secure, long distance
- **Disadvantages**: Expensive, fragile, difficult to install
- **Use**: Backbone networks, undersea cables, ISPs

#### Comparison of Guided Media

| Feature | Twisted Pair | Coaxial | Fiber Optic |
|---------|-------------|---------|-------------|
| **Cost** | Cheapest | Medium | Expensive |
| **Bandwidth** | Low–Medium | Medium | Very High |
| **Distance** | ~100m | ~500m | 2–100+ km |
| **EMI Immunity** | Low | Medium | Complete |
| **Security** | Low | Medium | High |
| **Installation** | Easy | Moderate | Difficult |

### B. Unguided (Wireless) Media

#### 1. Radio Waves
- **Frequency**: 3 KHz – 1 GHz
- **Omnidirectional** — broadcast in all directions
- Can penetrate walls
- **Use**: FM radio, TV, Wi-Fi, Bluetooth
- Prone to interference

#### 2. Microwaves
- **Frequency**: 1 GHz – 300 GHz
- **Unidirectional** — line-of-sight transmission
- **Terrestrial**: Tower-to-tower, requires line of sight
- **Satellite**: Ground → Satellite → Ground (geosynchronous orbit ~36,000 km)
- **Use**: Long-distance telephone, satellite TV, backbone links
- **Disadvantage**: Weather-sensitive, expensive setup

#### 3. Infrared
- **Frequency**: 300 GHz – 400 THz
- **Short range**, cannot penetrate walls
- Line-of-sight only
- **Use**: TV remotes, short-range device communication
- Not used for long-distance networking

#### Comparison of Unguided Media

| Feature | Radio Waves | Microwaves | Infrared |
|---------|------------|------------|----------|
| **Direction** | Omnidirectional | Unidirectional | Line-of-sight |
| **Range** | Long | Very Long | Short |
| **Penetration** | Through walls | Line-of-sight | Cannot penetrate |
| **Use** | Wi-Fi, Radio | Satellite | Remotes |

---

## Multiplexing

Technique to **combine multiple signals** and transmit them over a **single channel**.

### 1. FDM (Frequency Division Multiplexing)
```
Channel: |--f1--|--f2--|--f3--|--f4--|
          Guard bands between frequencies
```
- Divides bandwidth into **separate frequency bands**
- Each signal gets its own frequency range
- **Guard bands** prevent interference
- **Analog** technique
- **Example**: Radio broadcasting, cable TV

### 2. TDM (Time Division Multiplexing)
```
Time:  |--T1--|--T2--|--T3--|--T1--|--T2--|--T3--|
        Slot1   Slot2  Slot3  Slot1   Slot2  Slot3
```
- Divides time into **slots**, each user gets a slot
- **Types**:
  - **Synchronous TDM** — fixed time slots, even if no data (wastes bandwidth)
  - **Statistical (Asynchronous) TDM** — slots assigned on demand (more efficient)
- **Digital** technique
- **Example**: Telephone networks, T1 lines

### 3. WDM (Wavelength Division Multiplexing)
- Similar to FDM but uses **light wavelengths** in fiber optics
- Each signal transmitted on a **different wavelength (color) of light**
- **DWDM (Dense WDM)** — very closely spaced wavelengths, more channels
- **Example**: Long-haul fiber optic communication

### 4. CDM (Code Division Multiplexing)
- Each user gets a **unique code** to encode data
- All users transmit simultaneously on the **same frequency**
- Receiver uses the code to extract the correct signal
- **Used in**: CDMA mobile networks (3G)

| Technique | Divides | Type | Medium | Example |
|-----------|---------|------|--------|---------|
| FDM | Frequency | Analog | Any | Radio, Cable TV |
| TDM | Time | Digital | Any | Telephone |
| WDM | Wavelength | Optical | Fiber | Fiber networks |
| CDM | Code | Digital | Wireless | CDMA, 3G |

---

## Switching Techniques

How data is moved from source to destination across a network.

### 1. Circuit Switching
```
[A] ═══════════════════════ [B]
     Dedicated path established
     before communication
```
- **Dedicated communication path** established for the entire duration
- **Three phases**: Connection establishment → Data transfer → Disconnection
- **Advantages**: Guaranteed bandwidth, no delay once established, ordered delivery
- **Disadvantages**: Inefficient (channel idle during silence), setup delay, no alternate path if link fails
- **Example**: Traditional telephone network (PSTN)

### 2. Packet Switching
Each message divided into **packets** that travel independently.

#### a) Datagram Approach (Connectionless)
```
Packet 1 → Route A → [Destination]
Packet 2 → Route B → [Destination]
Packet 3 → Route A → [Destination]
```
- Each packet routed **independently** — different paths possible
- **No connection setup** required
- Packets may arrive **out of order** — reassembled at destination
- **Example**: IP (Internet Protocol)

#### b) Virtual Circuit Approach (Connection-oriented)
```
[A] --VC1-→ [R1] --VC1-→ [R2] --VC1-→ [B]
     Path decided before data transfer
```
- **Logical path established** before transmission (not dedicated physical path)
- All packets follow the **same route** and arrive **in order**
- **Types**:
  - **SVC (Switched Virtual Circuit)** — temporary, like a phone call
  - **PVC (Permanent Virtual Circuit)** — permanent, like a leased line
- **Example**: ATM, Frame Relay, MPLS

### 3. Message Switching
```
[A] → [R1] stores full message → [R2] stores full message → [B]
       (store-and-forward)
```
- **Entire message** stored at each intermediate node before forwarding
- **No real-time** communication — store-and-forward at each hop
- **No dedicated path** or circuit
- **Advantages**: More efficient channel usage, message priority possible
- **Disadvantages**: High delay, large storage needed at nodes, not suitable for real-time
- **Example**: Email systems, early telegraph

### Comparison of Switching Techniques

| Feature | Circuit | Packet (Datagram) | Packet (VC) | Message |
|---------|---------|-------------------|-------------|---------|
| **Path** | Dedicated | No fixed path | Logical path | No fixed path |
| **Setup** | Required | None | Required | None |
| **Delay** | Setup delay | Variable | Setup + low | Very high |
| **Order** | Guaranteed | Not guaranteed | Guaranteed | N/A |
| **Efficiency** | Low | High | Medium–High | Medium |
| **Real-time** | Yes | Possible | Yes | No |
| **Example** | PSTN | IP/Internet | ATM, MPLS | Email |

---

## Signal Encoding & Modulation (Brief Overview)

### Digital-to-Digital Encoding
- **NRZ (Non-Return to Zero)**: High=1, Low=0 (simple, sync issues)
- **Manchester**: Transition in middle of bit (used in Ethernet)
- **Differential Manchester**: Transition at start = 0, no transition = 1 (used in Token Ring)

### Digital-to-Analog Modulation
- **ASK (Amplitude Shift Keying)**: Amplitude changes for 0/1
- **FSK (Frequency Shift Keying)**: Frequency changes for 0/1
- **PSK (Phase Shift Keying)**: Phase changes for 0/1
- **QAM (Quadrature Amplitude Modulation)**: Combination of ASK + PSK

### Analog-to-Digital Conversion
- **PCM (Pulse Code Modulation)**: Sampling → Quantization → Encoding
- **Nyquist Theorem**: Sampling rate ≥ 2 × max frequency

---

## Important Formulas

| Formula | Description |
|---------|-------------|
| **Nyquist Rate = 2 × B** | Minimum sampling rate (B = bandwidth) |
| **Nyquist Bit Rate = 2 × B × log₂(L)** | Max bit rate for noiseless channel (L = signal levels) |
| **Shannon Capacity = B × log₂(1 + SNR)** | Max bit rate for noisy channel |
| **SNR(dB) = 10 × log₁₀(SNR)** | Signal-to-Noise ratio in decibels |

---

## Quick Revision Checklist

- [ ] Guided media: Twisted pair (UTP/STP), Coaxial, Fiber (SMF/MMF)
- [ ] Unguided media: Radio, Microwave, Infrared
- [ ] Multiplexing: FDM, TDM (sync/async), WDM, CDM
- [ ] Circuit switching: dedicated path, 3 phases, PSTN
- [ ] Packet switching: Datagram (connectionless) vs Virtual Circuit (connection-oriented)
- [ ] Message switching: store-and-forward, high delay
- [ ] Signal encoding: NRZ, Manchester, ASK/FSK/PSK/QAM
- [ ] Nyquist & Shannon formulas
