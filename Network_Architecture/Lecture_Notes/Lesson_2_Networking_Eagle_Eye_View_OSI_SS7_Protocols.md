# 📓 Lesson 2 — Computer Networking, Eagle Eye View
### *CN at Scaler — Session 2* · OSI · SS7 · BBR · TCP · UDP · QUIC · SMTP · POP · FTP

> *"Seven syscalls make a server. Seven layers of nesting carry eleven bytes. Today we open both ends."*

**Legend:** 🧠 = my addition, *not on the slides* · 💡 **Simple words mein** = an easy (sometimes Hinglish) summary. Read it first if a section feels heavy.

---

## ⭐ What this lecture is about (and why it matters)

Lesson 1 built a TCP server from raw system calls. Lesson 2 zooms *out* and shows what sits under that socket: the OSI layers (over copper, Wi-Fi radio, even a Starlink satellite 550 km up), a separate "second internet" for phones (**SS7**, Signalling System No. 7), the life of a TCP connection (handshake, close, SYN cookies, **BBR** congestion control), the faster options (**UDP**, **QUIC**), and three text protocols — **SMTP**, **POP3**, **FTP** — that you will type *by hand* in `telnet`.

One idea shows up at every stop:

```
 ┌──────────────────────────────────────────────────────────┐
 │   ONE LAYER'S HEADERS ARE ANOTHER LAYER'S BODY.          │
 │   (Ethernet ⊃ IP ⊃ TCP ⊃ HTTP  ==  MTP2 ⊃ MTP3 ⊃ … ⊃ SMS) │
 └──────────────────────────────────────────────────────────┘
```

Why it matters: most networking interview questions, most "why is my connection slow / stuck / refused" bugs, and every protocol you design come down to four ideas — **encapsulation** (wrapping), **framing**, **where reliability lives**, and **keeping control separate from data**.

### The agenda (five stops)

| # | Stop | One-line hook from the slide |
|---|---|---|
| 01 | Sockets | Six syscalls and you have a server. The seventh is where the bugs live. |
| 02 | OSI | Seven layers — over copper, over radio, and over a satellite 550 km up. |
| 03 | SS7 | A second internet, older than TCP/IP, and why an SMS is 140 bytes. |
| 04 | TCP · UDP · QUIC | Handshake, teardown, SYN cookies, and the round-trip tax. |
| 05 | Text protocols | SMTP, POP and FTP — typed by hand into telnet. |

---

## 🧱 0 · Prerequisites from scratch

If any of these feel weak, read them first — the slides assume all of them.

### 🧠 0.1 Bits, bytes, octets, hex

- A **bit** is a 0 or 1. A **byte** is 8 bits. Network people say **octet** for "exactly 8 bits" (some old machines had 6- or 9-bit "bytes", so standards say octet to be clear). A **septet** is 7 bits — needed for SMS.
- **Hex** (base 16) writes one byte as two digits: `0x4D` = `0100 1101` = 77 decimal. Each nibble (4 bits) is one hex digit.

### 🧠 0.2 What is a packet?

Data is cut into chunks. Each chunk has a **header** (address + control info, like the writing on an envelope) and a **payload** (the real content, the letter inside). Each layer calls the chunk by a different name:

| Layer | Name of the chunk |
|---|---|
| L2 (Ethernet) | **frame** |
| L3 (IP) | **packet** |
| L4 (TCP) | **segment** · (UDP) **datagram** |
| L7 (app) | **message** / data |

### 🧠 0.3 Endianness (byte order)

A 16-bit number like port **2026** = `0x07EA` needs two bytes. Which byte goes first?

```
 big-endian    (network byte order):  07 EA   ← "most significant byte first"
 little-endian (x86, ARM-as-used):    EA 07
```
The wire is **always big-endian**. Your laptop is almost surely little-endian. `htons()` = **h**ost **to** **n**etwork **s**hort (16-bit) does the swap. (`htonl` does 32-bit.)

### 🧠 0.4 File descriptor and socket

A **file descriptor (fd)** is a small number the kernel gives your process as a handle to "something you can read/write" — a file, a pipe, or a **socket** (a network endpoint). fd 0/1/2 are stdin/stdout/stderr, so your first socket is usually fd 3.

### 🧠 0.5 IP address, port, 4-tuple

- **IP address** = which machine. **Port** = which program on that machine (16 bits, 0–65535).
- A TCP connection is uniquely named by the **4-tuple** `(src IP, src port, dst IP, dst port)`. An **ephemeral port** is a temporary high source port the kernel picks for a client by itself.

### 🧠 0.6 RTT, bandwidth, latency

- **Latency** = time for one bit to get there. **RTT** (round-trip time) = there and back. **Bandwidth** = bits per second the pipe carries.
- Analogy: bandwidth is how *wide* the highway is; RTT is how *long* it is. More lanes don't make the drive shorter.
- **BDP** (bandwidth-delay product) = bandwidth × RTT = how much data is "in flight" in the pipe at once. Important for BBR.

### 🧠 0.7 Buffers and queues

If a router or switch gets packets faster than it can send them, it keeps the extras in a **buffer** (a queue). A buffer handles short bursts, but a *full* buffer means (a) extra delay for everything waiting and (b) new packets get **dropped**.

### 🧠 0.8 Checksum and CRC

Both are small numbers calculated from the data so the receiver can spot damage. A **checksum** (TCP/IP) is a simple sum; a **CRC** (cyclic redundancy check — Ethernet's CRC-32, SS7's CRC-16) uses polynomial maths and catches many more error types. Neither is security — anyone can recalculate them.

### 🧠 0.9 Control plane vs data plane

- **Data plane** = moving the real user traffic (voice, video, file bytes). **Control plane** = the signalling *about* the traffic (set up a call, route this, end it).
Keeping them separate is one of the lecture's four big takeaways.

### 🧠 0.10 telnet and netcat

`telnet host port` (and `nc host port`) open a raw TCP connection and connect your keyboard to it. What you type is sent as bytes; what comes back is printed. For a **text** protocol, this means *you* can act as the client.

---

## 1 · Sockets — recap and the two new details

> *"Everything above this is a convention. Everything below is somebody else's problem."*

### 1.1 The server: seven calls

**Plain English:** a server is a program that takes a port, waits for someone to knock, and talks to them.

```c
/* echo_server.c */
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(2026);
    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 1);

    while (1) {
        int client_fd = accept(server_fd, NULL, NULL);
        char buf[4096];
        int n = read(client_fd, buf, sizeof(buf));
        write(client_fd, buf, n);
        close(client_fd);
    }
}
```

```
 socket → bind → listen → accept → read → write → close
   │        │       │        │       │       │       │
 make fd  claim  become   take one  get   send   release
          port   passive  client    bytes bytes
```

The slide's point: **"Everything else in this file is address bookkeeping."** Two details on the slide come back later:

| Detail | What the slide says | Why you care |
|---|---|---|
| `listen(fd, 1)` | The `1` is the **accept-queue depth, not the connection limit.** | Comes back in the **SYN flood** section (§4.2). |
| `htons(2026)` | Host to network short. **The wire is big-endian; your laptop probably isn't.** | Comes back in the **FTP 227 reply** (§5.8), where you do `htons` by hand. |

> 🧠 **"Six syscalls … the seventh is where the bugs live."** The agenda says six-plus-one. Simple reading: the first six give you a working server; the seventh, `close()`, is where the tricky bugs are — forgotten closes (`CLOSE_WAIT`), `TIME_WAIT` and "address already in use", RST vs FIN. All of these come later in this lecture.

**Gotchas**
- `listen(fd, 1)` does **not** mean "only one client ever". It means at most ~1 fully-connected client may wait for `accept()` at a time. 🧠 Linux also keeps a separate **SYN queue** for half-open connections (`net.ipv4.tcp_max_syn_backlog`) and limits the backlog to `net.core.somaxconn`.
- No return values are checked (slide code is kept short to read).

### 1.2 The client: the same calls, mirrored

```c
/* client.c */
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    addr.sin_port = htons(8080);
    connect(fd, (struct sockaddr*)&addr, sizeof(addr));

    write(fd, "hello", 5);

    struct linger l = {1, 0}; // abortive close: sends RST instead of FIN
    setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
    close(fd);
    return 0;
}
```

**No bind, no listen.** The kernel picks an **ephemeral source port** for you. **`connect()` is what triggers the 3-way handshake.**

```
   SERVER                         CLIENT
   socket()                       socket()
   bind()      ◀── not needed ──
   listen()    ◀── not needed ──
   accept()  ◀════ handshake ════ connect()
   read()    ◀─────────────────── write()
   write()   ───────────────────▶ read()
   close()                        close()
```

> ⚠️ **Gotcha on the slides:** the server listens on **2026**, but the client connects to **8080**. They are two separate examples — to test them together, change one port to match the other.

### 1.3 `SO_LINGER {1, 0}` — FIN vs RST

**Plain English:** there are two ways to end a phone call — say "bye" and hang up politely, or pull the cable out of the wall.

| | Normal `close()` | `SO_LINGER {1, 0}` then `close()` |
|---|---|---|
| Sends | **FIN** — a polite "I'm done sending" | **RST** — reset, abort |
| Unsent data | delivered | **discarded** |
| Afterwards | socket sits in **TIME_WAIT for 2 MSL** | **no TIME_WAIT**, torn down immediately |
| Speed | slower | fast |
| Use | always, in production | **only to demonstrate the difference** |

- `struct linger { int l_onoff; int l_linger; }` → `{1, 0}` = "linger is ON, with a timeout of 0 seconds" = abort.
- 🧠 **MSL** = Maximum Segment Lifetime, the longest a stray segment may live in the network. 2 MSL is ~60 s on Linux (`TIME_WAIT` is fixed at 60 s there), up to 4 minutes by the original RFC. The peer that gets an RST sees `ECONNRESET` on its next read/write. 🧠

**Why does this exist?** Sometimes you *want* to drop a connection at once (a badly behaving client, a load balancer health check). But "fast, also lossy" — any bytes still in the send buffer are lost.

---

## 2 · OSI — seven layers, each refusing to care about the others

> *"A 1984 committee model that has outlived almost everything built on top of it."*

### 2.1 The seven layers

**Plain English:** the OSI (Open Systems Interconnection) model splits "send data across a network" into seven stacked jobs. Each layer talks only to the layer just above and below it, and ignores the rest. Like the post: the letter writer doesn't care if the mail goes by truck or plane.

> 💡 **Simple words mein:** Courier parcel socho — gift (HTTP) ko box (TCP) mein, box ko courier bag (IP) mein, bag ko truck (Ethernet) mein. Har level sirf apna label padhta hai, andar kya hai usse matlab nahi. Yahi encapsulation hai.

```
 ┌────┬──────────────┬───────────────────────────────────────┐
 │ L7 │ APPLICATION  │ HTTP · FTP · DNS · SMTP               │
 │ L6 │ PRESENTATION │ Encryption, compression, encoding     │
 │ L5 │ SESSION      │ Session management, authentication    │
 │ L4 │ TRANSPORT    │ TCP · UDP                             │
 │ L3 │ NETWORK      │ IP · ICMP · routing                   │
 │ L2 │ DATA LINK    │ Ethernet · MAC · ARP · switches       │
 │ L1 │ PHYSICAL     │ Cables, signals, hardware             │
 └────┴──────────────┴───────────────────────────────────────┘
   🧠 mnemonic (top→bottom): "All People Seem To Need Data Processing"
```

Acronyms: **HTTP** HyperText Transfer Protocol · **FTP** File Transfer Protocol · **DNS** Domain Name System · **SMTP** Simple Mail Transfer Protocol · **IP** Internet Protocol · **ICMP** Internet Control Message Protocol (ping) · **MAC** Media Access Control address (hardware address of a network card) · **ARP** Address Resolution Protocol (finds the MAC for an IP on the local network).

### 2.2 TCP vs UDP (both at L4)

| | **TCP** (Transmission Control Protocol) | **UDP** (User Datagram Protocol) |
|---|---|---|
| Connection | ✔ Connection-oriented | ✔ Connectionless |
| Delivery | ✔ Reliable, ordered | ✘ No delivery guarantee, ✘ no ordering |
| Control | ✔ Flow and congestion control, ✔ error recovery | ✔ Fast, low overhead |
| Latency | ⏱ Higher | ⏱ Low |
| Used by | 📦 HTTP, FTP, SMTP, SSH | 📦 DNS, VoIP, video calls, gaming |

🧠 **Gotcha:** the real internet's TCP/IP model has only ~4–5 layers; L5 and L6 have no separate protocols — TLS and app framing are "folded into the app". The slides say the same in the SS7-vs-TCP/IP table (§3.2).

### 2.3 L1/L2 on a wire vs on radio: CSMA/CD vs CSMA/CA

> *"On a wire you detect collisions. On radio you can only avoid them."* Same L3 and above. Completely different L1 and L2 problem.

**Plain English:** when many devices share one medium, two may talk at once — a **collision**. On a wire you can *hear* a collision while talking (like noticing someone interrupting you on a call). On radio you can't — your own voice is so loud you can't hear anyone else.

- **CSMA/CD** = Carrier Sense Multiple Access with **Collision Detection** — classic Ethernet.
- **CSMA/CA** = Carrier Sense Multiple Access with **Collision Avoidance** — Wi-Fi (**IEEE 802.11**).

```
 CSMA/CA (802.11)
 ────────────────
   Want to transmit?
          │
          ▼
   Listen to channel  (carrier sense)
          │
   Is someone transmitting? ──YES──▶ wait / defer, listen again ─┐
          │ NO                                                    │
          ▼                       ◀──────────────────────────────┘
   Random backoff  (contention window)
          │
          ▼
   Transmit  (frame goes out over radio)
          │
   ACK received? ──NO──▶ back off and retransmit
          │ YES
          ▼
       Success
```

**Why not CSMA/CD on Wi-Fi?** A radio **cannot listen while it transmits** — its own signal hides everything else. So Wi-Fi can't detect a collision; it **avoids** one instead: sense, back off, transmit, and **ask for an ACK for every single frame**.

**Where does the time go?** **≈2 ms of contention delay** (waiting for a turn) before a frame even leaves (slide figure), and **every frame is ACKed one by one at L2** — long before TCP at L4 knows anything happened.

🧠 Extra background:
- The **random backoff** picks a random number of time slots from a "contention window" that doubles after each failure (exponential backoff), so two stations that collided are unlikely to collide again.
- Modern switched, full-duplex Ethernet almost never has collisions (each cable is a private two-way link), so CSMA/CD is mostly history. Wi-Fi is still shared.
- **Gotcha:** Wi-Fi resends at L2, so a "lossy" Wi-Fi link often looks to TCP like a *slow, jittery* link, not a lossy one.

### 2.4 L1 can be a satellite — and nothing above L2 notices

> *"Starlink is a physical layer. That is the entire point of having layers."*

```
 💻 Your laptop
   │  Ethernet / Wi-Fi        (local network)
 📶 Router
   │
 🛰 Starlink terminal         (phased-array dish)
   │  RF uplink               (Ku / Ka-band)
 🛸 LEO satellite             (~550 km · ~20 ms)
   │  🔴 Laser crosslink      (optional, sat-to-sat)
 🏭 Ground gateway            (terrestrial handoff)
   │
 ☁  The internet              (same IP packet)
```

**The whole case for layering:** nine physical hops, one of them 550 km straight up, and the IP header your laptop wrote arrives **unchanged — except TTL, checksum and any NAT rewrite**. L3 and above never learn that space was involved.

Acronyms: **LEO** Low Earth Orbit · **RF** Radio Frequency · **Ku/Ka-band** radio frequency ranges (🧠 roughly 12–18 GHz and 26–40 GHz) · **TTL** Time To Live · **NAT** Network Address Translation.

🧠 Why those three fields change:
- **TTL** goes down by 1 at every router (so looping packets finally die). Because TTL changed, the IPv4 **header checksum** must be recalculated.
- **NAT** rewrites source IP/port when your private address (192.168.x.x) is swapped for a public one.

🧠 **Phased-array dish** = an antenna that steers its beam electronically (no moving parts) to follow fast-moving satellites overhead.

**Gotcha:** "Satellite internet is slow (high latency)" was true for *geostationary* satellites (~36,000 km, ~600 ms RTT). LEO at ~550 km is ~20 ms — that's the whole Starlink trick.

---

## 3 · SS7 — the second internet

> *"There is a second internet. You used it this morning and you have never seen a packet from it."*

**SS7** = Signalling System No. 7. It's the worldwide network phone operators use to set up calls, deliver SMS, and let your phone roam abroad. It is older than TCP/IP's rise and still runs under the mobile network.

### 3.1 The one big idea: get signalling out of the voice channel (1975)

**Before — in-band signalling.** One channel carried *both* the talk and the control:

```
 ┌─────────── ONE CHANNEL ─────────────────────────────────────┐
 │  Voice        2600 Hz tone          Voice      Dialled digits│
 │  what you say "this trunk is idle"  what you say  MF tones   │
 └─────────────────────────────────────────────────────────────┘
   Anyone who can MAKE THE SOUND can MAKE THE DECISION.
```

🧠 This is how phone phreaks (phone hackers) broke the system: a whistle or "**blue box**" playing 2600 Hz told the switch the line was idle, and then **MF (multi-frequency) tones** let you dial anywhere, free. The famous story: a Cap'n Crunch cereal-box whistle that happened to hit 2600 Hz.

**After — out-of-band signalling (SS7, 1975):**

```
  ════════ Voice bearer — 64 kbps DS0 ═════════════════  the conversation, and nothing else

  ──────── Signalling network — MTP / SCCP / TCAP / MAP / ISUP ────
           a separate packet-switched network the caller never touches
```

> *"Same separation of concerns as a control plane and a data plane. Fifty years old, and every network you build still does it."*

🧠 **DS0** = "Digital Signal 0", one 64 kbps channel = 8,000 samples/second × 8 bits — exactly one digital phone call.

**Why it exists:** security (users can no longer send fake control tones) and efficiency (signalling no longer blocks a voice line while a call is being set up).

> 💡 **Simple words mein:** Pehle baat-cheet aur control ek hi line pe the — koi bhi seeti bajake switch ko "line free hai" bol ke free call kar sakta tha. SS7 ne control ko alag "staff only" network pe daal diya; ab caller usse chhoo hi nahi sakta.

### 3.2 SS7 stack vs TCP/IP stack

> *"Same seven jobs, different names, very different trust model."*

| OSI | SS7 stack | what it does | TCP/IP stack | what it does |
|---|---|---|---|---|
| **L7** | MAP · INAP · CAP · ISUP | Call setup, roaming, SMS | HTTP · DNS · SMTP · SIP | Application protocols |
| **L5–L6** | TCAP | Dialogues, components, ASN.1 BER | TLS / app framing | No distinct layer — folded into the app |
| **L4** | SCCP (classes 0–3) | Subsystem numbers, Global Title Translation | TCP / UDP | Ports, byte-stream sequencing, windowing |
| **L3** | MTP3 | Point-code routing, SLS load sharing | IP | Best-effort datagrams, fragmentation, TTL |
| **L2** | MTP2 | Signal units, FSN/BSN, CRC-16, retransmission | Ethernet · PPP · HDLC | Framing, MAC addressing, FCS |
| **L1** | MTP1 | 64 kbps DS0 on E1/T1 | Copper · fibre · radio | Any bit-carrying medium |

> *SS7 is a whole stack; TCP is one protocol at L4. The fair comparison is stack to stack.*

**Acronym decoder** (🧠 expansions):

| Acronym | Stands for | Rough TCP/IP equivalent |
|---|---|---|
| **MTP1/2/3** | Message Transfer Part levels 1/2/3 | physical / link / IP |
| **SCCP** | Signalling Connection Control Part | something like ports + DNS-ish name translation |
| **TCAP** | Transaction Capabilities Application Part | RPC framework: "dialogues" of request/response "components" |
| **MAP** | Mobile Application Part | the mobile "API": SMS, roaming, location |
| **INAP** | Intelligent Network Application Part | toll-free numbers, prepaid logic |
| **CAP** | CAMEL Application Part | prepaid / intelligent services while roaming |
| **ISUP** | ISDN User Part | call setup/teardown between switches |
| **ASN.1 BER** | Abstract Syntax Notation One, Basic Encoding Rules | a binary tag-length-value encoding (like protobuf's ancestor) |
| **Point Code** | address of a signalling node | IP address |
| **SSN** | SubSystem Number | port number |
| **Global Title (GT)** | a phone-number-style address translated to point code + SSN | hostname → IP lookup |
| **SLS** | Signalling Link Selection | a hash that spreads traffic across parallel links |
| **E1 / T1** | 2.048 Mbps (32×DS0, Europe/India) / 1.544 Mbps (24×DS0, USA) trunks | |
| **PPP / HDLC / FCS** | Point-to-Point Protocol / High-level Data Link Control / Frame Check Sequence | |
| **SIP** | Session Initiation Protocol | (it *is* TCP/IP) |

### 3.3 Packet shape: one layer's headers are another layer's body (TCP/IP side)

The slide's example: a browser on `192.168.1.42` sending `GET /` to `example.com`. Reading top to bottom = the receiver peeling the onion.

```
L2 — ETHERNET II FRAME
┌─────────┬─────────┬───────────┬──────────────────────────────────┬──────────┐
│ Dst MAC │ Src MAC │ EtherType │ Payload — the entire IP packet   │   FCS    │
│   6 B   │   6 B   │ 2B 0x0800 │ 46–1500 B (MTU)                  │ 4B CRC-32│
└─────────┴─────────┴───────────┴──────────────────────────────────┴──────────┘
                                 │
L3 — IPv4 PACKET (20 B header)   ▼
┌───────┬─────────┬─────┬───────┬────────┬────────┬────────┬────────────────────┐
│Ver/IHL│Total Len│ TTL │ Proto │Checksum│ Src IP │ Dst IP │ Payload — TCP seg. │
│ 4b+4b │   2B    │ 1B  │6 = TCP│   2B   │   4B   │   4B   │                    │
└───────┴─────────┴─────┴───────┴────────┴────────┴────────┴────────────────────┘
                                                             │
L4 — TCP SEGMENT (20 B header)                               ▼
┌──────────┬──────────┬─────┬─────┬─────────────┬────────┬──────────────────────┐
│ Src Port │ Dst Port │ Seq │ Ack │    Flags    │ Window │ Payload — app data   │
│    2B    │    2B    │ 4B  │ 4B  │ SYN/ACK/FIN │   2B   │ ≤ MSS, typ. 1460 B   │
└──────────┴──────────┴─────┴─────┴─────────────┴────────┴──────────────────────┘
                                                             │
L7 — APPLICATION DATA                                        ▼
   GET / HTTP/1.1 · Host: example.com
   — an opaque byte stream TCP neither knows nor cares about
   (HTTP request · TLS record · SMTP command · SIP INVITE · anything)
```

**The byte budget (memorise this):**
```
  1500 B MTU − 20 B IP − 20 B TCP = 1460 B MSS
  + 14 B Ethernet header + 4 B FCS  →  1518 B wire frame
```

Definitions:
- **EtherType** says what's inside: `0x0800` = IPv4 (🧠 `0x86DD` = IPv6, `0x0806` = ARP).
- **MTU** = Maximum Transmission Unit, the biggest L2 payload (1500 B on Ethernet).
- **MSS** = Maximum Segment Size, the biggest TCP *payload* per segment.
- **Ver/IHL** = IP version (4) + Internet Header Length (in 32-bit words; 5 → 20 bytes).
- **Proto = 6** means "the payload is TCP" (🧠 17 = UDP, 1 = ICMP).
- **FCS** = Frame Check Sequence, a CRC-32 over the frame.

🧠 Notes / gotchas:
- The slide shows only the key IPv4 fields. The full 20-byte header also has Type of Service/DSCP, Identification, Flags and Fragment Offset.
- Why 46 B minimum payload? Classic Ethernet needed frames ≥ 64 B for collision detection to work (14 + 46 + 4 = 64). Short payloads get padding.
- On the real copper there are also 8 bytes of preamble and a 12-byte gap between frames, not counted in the 1518.
- Each layer reads *only its own header*, then passes the rest up. That's the "onion".

### 3.4 SS7 encapsulation: seven layers of nesting to carry eleven bytes

The slide's example: a GSM phone sending **"Hello World!"** — an **MO-ForwardSM** (Mobile-Originated Forward Short Message), all the way down to the octets.

```
1 · MTP2   MSU   Flag · BSN+BIB · FSN+FIB · LI · [ SIO+SIF ] · CRC-16 · Flag   — SIF hard cap 273 octets
 └ 2 · MTP3     SIO 0x83 · DPC 2065 · OPC 2050 · SLS 5                    → payload 129 octets
    └ 3 · SCCP   UDT  Msg 0x09 · class 1 · Called GT +447700900999
    │                         · Calling GT +447700900111                   → 99 octets
       └ 4 · TCAP  BEGIN  Tag 0x62 · OTID 0x00000001 · Dialogue AARQ
       │                  ACN = shortMsgMO-Relay v3
          └ 5 · Invoke  Tag 0xA1 · Invoke ID 1 · opCode 46 (0x2E) = mo-ForwardSM  → 47 octets
             └ 6 · MAP  MO-ForwardSM-Arg  sm-RP-DA +447700900999
             │                             · sm-RP-OA +447700900456 · sm-RP-UI = 25 octets
                └ 7 · SMS-SUBMIT TPDU  MTI · TP-MR · TP-DA · TP-PID
                │                       · TP-DCS 0x00 (GSM-7) · TP-VP · TP-UDL 0x0C
                   └ 8 · TP-UD
                        C8 32 9B FD 06 5D DF 72 36 39 04
                        → "Hello World!" · 12 chars · 84 bits · 11 octets
```

> *"Identical picture to Ethernet ⊃ IP ⊃ TCP ⊃ HTTP. Different names, same onion. The payload just caps at 272 octets instead of 1460 B."*

**Field decoder** (🧠 expansions — the slide just lists the names):

| Field | Meaning |
|---|---|
| **MSU** | Message Signal Unit — the MTP2 frame that carries real data |
| **Flag** | a fixed bit pattern (`0x7E`) marking the start/end of a frame (HDLC-style) |
| **FSN / FIB** | Forward Sequence Number / Forward Indicator Bit — "this is frame #n" |
| **BSN / BIB** | Backward Sequence Number / Backward Indicator Bit — "I've got up to #n" (piggy-backed ACK) |
| **LI** | Length Indicator |
| **SIO** | Service Information Octet — what's inside (🧠 `0x83`: low nibble 3 = SCCP, high bits = network indicator) |
| **SIF** | Signalling Information Field — the payload |
| **DPC / OPC** | Destination / Origin Point Code — the "IP addresses" |
| **SLS** | Signalling Link Selection — load-sharing key |
| **UDT** | Unitdata — SCCP's connectionless message type (`0x09`) |
| **class 1** | SCCP connectionless, in-sequence delivery (🧠 class 0 = basic connectionless; 2/3 = connection-oriented) |
| **Called/Calling GT** | Global Titles — phone-number-style addresses (here SMSC and the sending switch) |
| **BEGIN / OTID** | TCAP opens a dialogue; Originating Transaction ID |
| **AARQ / ACN** | Association request / Application Context Name — "we're about to speak shortMsgMO-Relay version 3" |
| **Invoke / opCode 46** | "please run operation 46" = mo-ForwardSM (a remote procedure call) |
| **sm-RP-DA / sm-RP-OA / sm-RP-UI** | Short Message Relay Protocol Destination Address / Originating Address / User Information |
| **TPDU** | Transfer Protocol Data Unit — the actual SMS |
| **MTI, TP-MR, TP-DA, TP-PID, TP-DCS, TP-VP, TP-UDL** | Message Type Indicator, Message Reference, Destination Address, Protocol ID, Data Coding Scheme (`0x00` = GSM-7), Validity Period, User Data Length (`0x0C` = 12 *septets*) |
| **TP-UD** | User Data — the text |

🧠 Note the tags `0x62`, `0xA1`: these are ASN.1 BER tags — every TCAP/MAP element is **tag, length, value**. The same TLV idea from building binary protocols in Lesson 1.

### 3.5 Why an SMS is 140 bytes

> *"An SMS is 140 bytes because a 1980s signalling packet caps at 272."* Three separate size limits, stacked. The smallest one wins, and it shapes the product.

```
 ┌───────────────────────────┐
 │ 272 octets — SIF          │  MTP2's hard ceiling: physical limit of one Message Signal Unit
 │  ┌──────────────────────┐ │
 │  │ ~200 octets          │ │  sm-RP-UI: what MAP is willing to carry as opaque user data
 │  │  ┌────────────────┐  │ │
 │  │  │ 140 octets     │  │ │  TP-UD: the SMS itself — chosen so the WHOLE MAP
 │  │  │ TP-UD          │  │ │  operation always fits in ONE MSU
 │  │  └────────────────┘  │ │
 │  └──────────────────────┘ │
 └───────────────────────────┘
```

**No splitting, no joining back.** The 140-octet limit exists so that **one text message is always exactly one signalling packet**. Nothing to split, nothing to reorder, nothing to remember — that's why SMS ran on 1980s hardware, and **why it still works when the data network is down**.

🧠 **272 vs 273?** The encapsulation slide says "SIF hard cap 273 octets"; this slide says 272. Both are right: in ITU-T Q.703 the SIF is at most **272** octets, and **SIO + SIF together** is at most **273**. The framing slide later writes "SIF ≤ 273" — read it as SIO+SIF.

### 3.6 GSM-7: 160 characters in 140 bytes

> *"I said 140 bytes, but you can send 160."* The gap is explained by a packing trick from mobile engineers.

```
  140 octets  ×8  = 1120 bits  ÷7  = 160 characters
  (TP-UD)     bits/octet          bits/char (GSM 7-bit alphabet, TP-DCS = 0x00)
```

**The packing trick.** Each character is **7 bits, not 8** (plain English text never uses the 8th bit — "ASCII wastes a bit"). The septets are joined into one long bit stream and cut again into octets — so after the first character, **character edges no longer line up with byte edges**.

```
 12 chars × 7 bits = 84 bits  →  11 octets (88 bits), with 4 bits of zero padding
```

**And when it breaks:** switch TP-DCS to **UCS-2** (a 16-bit Unicode encoding) for emoji or any non-Latin script, and it becomes **140 ÷ 2 = 70 characters**. Same 140-octet field either way. So **one emoji can cut your message length by more than half — and quietly cost you a second SMS.**

> 💡 **Simple words mein:** Har dabba (byte) mein 8 seat hain, par har character ko sirf 7 chahiye. Toh khaali seat pe agle character ka tukda bitha do — 8 characters 7 dabbon mein fit. Emoji aaya toh har character ko 16 seat chahiye, sirf 70 fit honge.

🧠 How the packing works (GSM 03.38 / 23.038 rule): fill each octet from its **lowest bit upward**. Character 1's 7 bits go into the low 7 bits of octet 1; the lowest bit of character 2 fills octet 1's top bit; the other 6 bits of character 2 go into the low 6 bits of octet 2; and so on. Every 8 characters fit in exactly 7 octets.

```
 octet 1:  [c2 bit0 | c1 bits 6..0]
 octet 2:  [c3 bits 1..0 | c2 bits 6..1]
 octet 3:  [c4 bits 2..0 | c3 bits 6..2]
 ...       (shift grows by 1 each octet; after 7 octets it resets)
```
🧠 For letters, digits, space and `!`, GSM-7 codes are the same as ASCII codes (`H` = 0x48). Some symbols differ (e.g. `@` is 0x00 in GSM-7). Unpacking "Hello World!" by hand is homework — see §7.

🧠 Long SMS ("concatenated SMS") uses a few bytes of each 140 for a User Data Header saying "part 2 of 3", so each part holds 153 GSM-7 chars (or 67 UCS-2 chars).

### 3.7 Sending a text is a database lookup wrapped in a queue

**Plain English:** your phone doesn't send the SMS *to* your friend. It gives it to a post office (the SMSC), which finds out where your friend is right now and delivers it when it can.

```
 Phone A      VMSC/VLR        SMSC            HLR            MSC B        Phone B
 (handset)  (A's serving    (store &      (subscriber     (B's serving  (handset)
             switch)         forward)      database)       switch)
    │             │              │               │              │            │
    │─SMS-SUBMIT─▶│  (TPDU, 140 octets max)      │              │            │
    │             │─MAP mo-ForwardSM (opCode 46)▶│              │            │
    │             │◀─TCAP END — accepted, queued─│              │            │
    │             │              │  STORED. Delivery is now     │            │
    │             │              │  the SMSC's problem          │            │
    │             │              │─MAP sendRoutingInfoForSM────▶│            │
    │             │              │     (MSISDN)  │              │            │
    │             │              │◀─IMSI + serving MSC address─ │            │
    │             │              │─MAP mt-ForwardSM (SMS-DELIVER)──────────▶ │
    │             │              │               │              │─SMS-DELIVER▶ over the air
    │             │              │               │              │◀─delivery ack
    │             │              │◀────────mt-ForwardSM result──│            │
    │◀────────────│ SMS-STATUS-REPORT (only if you asked for one)            │
```

Acronyms (🧠 expansions):
- **MSC** = Mobile Switching Centre (the telephone exchange for mobiles); **VMSC** = the *visited* MSC serving you now.
- **VLR** = Visitor Location Register (temporary list of phones now in this MSC's area).
- **SMSC** = Short Message Service Centre (the store-and-forward post office).
- **HLR** = Home Location Register (your operator's main subscriber database: who you are, where you are now).
- **MSISDN** = your phone number. **IMSI** = International Mobile Subscriber Identity (the SIM's internal ID). **MO** = Mobile Originated (from the phone), **MT** = Mobile Terminated (to the phone).

**Key insight:** once the SMSC replies "accepted, queued", **your** phone's job is done. That's why you can SMS a phone that's switched off — the SMSC keeps it and tries again.

### 3.8 A phone call: signalling on one network, voice on another (ISUP)

```
 Phone A            MSC A                        MSC B              Phone B
 (caller)     (originating switch)        (terminating switch)     (callee)
   │ off-hook + dialled digits ▶│                    │                  │
   │                            │──IAM────────────▶  │                  │
   │                            │  Initial Address Message:            │
   │                            │  called number, bearer request ─ring─▶│
   │                            │◀─ACM─ Address Complete → caller hears ringback
   │                            │                    │◀── B answers ────│
   │                            │◀─ANM─ Answer Message → voice path cut through
   │═══ conversation: 64 kbps bearer circuit, NO SS7 involvement at all ═══│
   │ on-hook ▶                  │──REL (Release)──▶  │                  │
   │                            │◀─RLC (Release Complete) → circuit freed
```

> *"Four messages set up a call. During the conversation itself the signalling network is completely idle — it did its job at the edges."*

🧠 "Ringback" = the "trring-trring" tone you hear while the other phone rings. "Cut through" = the switches join the voice circuit end-to-end.

### 3.9 In 5G the same call is HTTP-shaped text over IP (IMS · SIP)

```
 Alice                      Proxy / SBC                       Bob
 (UA / softphone)           (IMS core)                 (UA / softphone)
   │─INVITE sip:bob@scaler.com (SDP offer)─▶│                    │
   │                            │──INVITE forwarded──────────────▶│
   │                            │◀─100 Trying · 180 Ringing───────│
   │◀────────180 Ringing────────│                                 │
   │                            │◀────200 OK (SDP answer)─────────│
   │◀──────────200 OK───────────│                                 │
   │───────────ACK — end to end, past the proxy──────────────────▶│
   │═════ RTP / SRTP media flows DIRECTLY between endpoints, UDP ══│
   │───────────BYE────────────────────────────────────────────────▶│
   │◀──────────200 OK──────────────────────────────────────────────│
```

> *"100, 180, 200 — SIP borrowed HTTP's status codes wholesale. Signalling is text; media is RTP over UDP. Control and data stay on separate paths."*

Acronyms: **IMS** IP Multimedia Subsystem · **SIP** Session Initiation Protocol · **UA** User Agent · **SBC** Session Border Controller (🧠 a SIP-aware firewall/proxy at the operator's edge) · **SDP** Session Description Protocol (🧠 a text blob saying "send me audio to this IP:port using these codecs") · **RTP** Real-time Transport Protocol · **SRTP** Secure RTP.

🧠 A SIP INVITE looks a lot like an HTTP request:
```
INVITE sip:bob@scaler.com SIP/2.0
Via: SIP/2.0/UDP 10.0.0.5:5060;branch=z9hG4bK776asdhds
From: Alice <sip:alice@scaler.com>;tag=1928301774
To: Bob <sip:bob@scaler.com>
Call-ID: a84b4c76e66710@10.0.0.5
CSeq: 1 INVITE
Content-Type: application/sdp
Content-Length: 142

v=0
...
```

**Try it yourself (from the slide):** Asterisk or FreeSWITCH, a softphone (Linphone, Zoiper), and Wireshark with the SIP dissector (decoder) on.

**The pattern:** ISUP (signalling) vs 64 kbps circuit (voice) → SIP (signalling) vs RTP (media). Fifty years later, still kept apart.

### 3.10 SS7 vs TCP: who is reliable, and where?

> *"SS7 makes every hop reliable. TCP makes only the endpoints reliable."*

| Function | SS7 | TCP |
|---|---|---|
| Sequencing | FSN — 7 bits, per signal unit, wraps at 128 | Sequence number — 32 bits, **per byte** |
| Acknowledgement | BSN + BIB, piggybacked on every SU | ACK number + flag, cumulative (+ SACK) |
| Retransmission | BIB toggle; PCR on long / satellite links | RTO timers, fast retransmit on 3 dup-ACKs |
| Flow control | SIB in an LSSU — a blunt stop/go signal | Sliding window advertised in every segment |
| Error detection | CRC-16 per signal unit | 16-bit ones-complement checksum |
| Addressing | Point Code (OPC/DPC) + SSN + Global Title | IP address + port number |
| Connection setup | Connectionless; SCCP class 2/3 if needed | 3-way handshake (SYN, SYN-ACK, ACK) |
| Congestion control | Hop-by-hop via MTP3 transfer-controlled | End-to-end (Reno, CUBIC, BBR) |
| Scope of reliability | **Per hop**, between adjacent signalling points | **End to end**, across arbitrary hops |
| Max payload | 272 octets (SIF) | ~1460 B typical, 64 KB with window scaling |
| Security | **None — trust by point code** | None natively; TLS layered above |

> *"Both stacks solve the same eleven problems. The only real disagreement is which layer gets to be responsible."*

🧠 Decoder:
- **SU** = Signal Unit. **LSSU** = Link Status Signal Unit. **SIB** = Status Indication "Busy".
- **PCR** = Preventive Cyclic Retransmission — on long-delay links (satellite), MTP2 keeps resending un-ACKed units on its own instead of waiting for a NACK.
- **RTO** = Retransmission TimeOut. **SACK** = Selective ACK ("I have bytes 1–1000 and 2001–3000, missing the middle"). **Fast retransmit**: if the sender sees 3 duplicate ACKs for the same number, it assumes a segment was lost and resends without waiting for the timer.
- **"Trust by point code"**: any node on the SS7 network is believed. Once operators began selling SS7 access to others, attackers could ask for a phone's location or redirect its SMS (that's how SMS-based 2FA codes have been stolen). 🧠

**Interview gotcha:** "Why doesn't the internet do per-hop reliability like SS7?" → the end-to-end principle: routers stay simple and remember nothing, endpoints handle reliability. It scales to billions of devices and lets apps that *don't want* reliability (voice, games) skip it.

### 3.11 BBR: stop treating packet loss as the congestion signal (2016)

> *"Loss-based TCP fills the buffer until something drops. On a fat, long, or lossy link that is exactly wrong."*

**Plain English:** classic TCP speeds up until something breaks (a packet is lost), then panics and halves its speed. BBR instead *measures* how wide and how long the road is, and drives at exactly that speed.

```
 In-flight data over time        (sketch of the slide's chart)

       /|      /|      /|      /|        ← Loss-based (Reno / CUBIC): sawtooth
      / |  ~~~/~|~~~~~/~|~~~~~/~|~~~     ← BBR: flat-ish line at bandwidth × RTT
     /  |    /  |    /  |    /  |
    /   |   /   |   /   |   /   |
   /    |__/    |__/    |__/    |
  ──────────────────────────────────▶ time
       drop!   drop!   drop!   drop!
```

**The sawtooth is the bug.** Reno and CUBIC grow until a packet is dropped, then halve. So they **need loss to move forward** — and on a link that drops packets for reasons *other than* congestion (Wi-Fi noise, a weak long-distance link), they crawl.

**What BBR measures instead:** **B**ottleneck **B**andwidth × **R**ound-trip propagation time. Send at the rate the path can really carry, keeping the queue almost empty. Google reports **up to 2,000× throughput on some benchmarks** — and much lower latency.

> 💡 **Simple words mein:** CUBIC = gaadi tab tak tez chalao jab tak takkar na ho, phir brake maar ke speed aadhi. BBR = pehle dekho sadak kitni chaudi aur lambi hai, phir bilkul utni speed pe chalao — na jam, na takkar.

Links from the slide: `youtu.be/VIX45zMMZG8` · cloud.google.com/blog — "TCP BBR comes to GCP" · bufferbloat, 2011: `youtu.be/qbIozKVz73g`.

🧠 Background:
- **Congestion window (cwnd)** = how many un-ACKed bytes TCP allows itself in flight. Reno = add a little on success, cut by a factor on loss (AIMD). CUBIC (Linux default) grows along a cubic curve but still roughly halves on loss.
- **Bufferbloat** (2011): routers got huge buffers. Loss-based TCP keeps filling them before any drop, so every packet waits in a long queue → seconds of delay on a "fast" link. BBR keeps the queue empty, so delay stays low.
- BBR cycles through phases (Startup → Drain → ProbeBW, with periodic ProbeRTT) to keep its two estimates fresh.
- On Linux: `sysctl net.ipv4.tcp_congestion_control` shows the current algorithm; `bbr` can be enabled. Congestion control is **sender-side only** — the receiver doesn't change. The 2,000× figure is a best case (long, lossy path vs CUBIC); usual gains are far smaller.

---

## 4 · TCP · UDP · QUIC

> *"No class with 'computer networks' in the title is complete without the handshake."*

### 4.1 Opening: the 3-way handshake

**Plain English:** each side picks a random starting number to count its bytes, and each side must hear the other confirm *its* number. Two numbers, two confirmations — the middle message carries one confirmation plus the second number, so three messages.

```
        Client                                              Server
 CLOSED — app calls connect()
          │── SYN  seq=1000  MSS 1460, SACK permitted, wscale 7 ──▶│
          │                                                        │ SYN RECEIVED — half-open
          │                                                        │ entry in backlog
          │◀─ SYN ACK  seq=5000 ack=1001  MSS 1460, wscale 7 ──────│
 ESTABLISHED — both ISNs agreed
          │── ACK  seq=1001 ack=5001  (no payload required) ──────▶│
          │                                                        │ ESTABLISHED — accept() returns
          │── PSH ACK  seq=1001  first payload bytes ─────────────▶│
```

**Why 3?**
- **Segment 1** announces the client's ISN (Initial Sequence Number). **Segment 2** ACKs it *and* announces the server's. **Segment 3** ACKs the server's.
- **The server's SYN is what forces the third leg** — someone has to confirm it.

> 💡 **Simple words mein:** Phone pe: "Hello, sun rahe ho?" (SYN) → "Haan sun raha hoon, tum mujhe sun rahe ho?" (SYN-ACK) → "Haan!" (ACK). Dono ko pakka pata ho ki doosra sun raha hai — tab baat shuru.

**SYN and FIN each use up one sequence number.** That's why `ack=1001` answers `seq=1000` even though the SYN had no payload — otherwise a bare SYN couldn't be told apart from a resend.

**The options in the SYN** (🧠 explained):
- **MSS 1460** — "don't send me segments bigger than 1460 bytes of payload."
- **SACK permitted** — "I understand selective ACKs."
- **wscale 7** — window scale: multiply the 16-bit window field by 2⁷ = 128, so windows can be bigger than 64 KB. Options are agreed only in SYNs — remember this for SYN cookies.
- **PSH** = Push flag: "give this to the application now."

**Random ISN.** Early stacks picked ISNs in a guessable way. Guess the next ISN and you can inject fake segments blind — **the Mitnick attack** (🧠 Kevin Mitnick, 1994, faked a trusted host by predicting sequence numbers). **RFC 6528:** ISN = keyed hash of the 4-tuple + timer.

**Gotchas**
- "The 3-way handshake exists to exchange data" — no, it's to **sync sequence numbers** (that's literally what SYN stands for).
- The server kernel finishes the handshake *before* your app calls `accept()`. `accept()` just takes an already-connected client off the queue.

### 4.2 Defence: SYN cookies (1996)

> *"Keep no state, and there is nothing to exhaust."* Between segment 1 and segment 3, the server holds an entry in a limited backlog. **That backlog is the target.**

**The attack (SYN flood):** send SYNs from **spoofed** (faked) source addresses and never finish the handshake. Each costs the attacker one packet and costs the server a half-open entry that stays until it times out. Fill the backlog and **real clients are refused**. *Remember `listen(server_fd, 1)` from the first slide.*

```
 Attacker (spoofed IPs)              Server backlog
   SYN from 1.2.3.4  ───────────▶   [half-open][half-open][half-open] FULL
   SYN from 5.6.7.8  ───────────▶      SYN-ACKs go to addresses that never reply
   SYN from 9.9.9.9  ───────────▶   Real client's SYN ──▶ ✘ dropped / refused
```

**The defence:** **don't store anything.** Pack the connection details into the server's own ISN — a **keyed hash of the 4-tuple, a rough timestamp and the MSS**. The client's ACK brings that number back, plus one. The server recalculates, checks it, and **rebuilds the whole connection from a packet it has never seen before.**

> 💡 **Simple words mein:** Sarkari office mein register mein naam likhne ki jagah, clerk aapko ek signed token de deta hai aur sab bhool jaata hai. Aap token wapas laaye, signature sahi hai? Kaam shuru. Register bhar hi nahi sakta, toh flood bekaar.

```
 Client                                   Server (stateless)
   │── SYN seq=x ─────────────────────▶│  ISN_s = hash(secret, 4-tuple, time) ⊕ MSS bits
   │◀─ SYN-ACK seq=ISN_s ack=x+1 ──────│  ... and FORGETS everything
   │── ACK ack=ISN_s+1 ───────────────▶│  recompute hash; if (ack-1) matches → valid!
   │                                    │  rebuild the connection, put it in the accept queue
```

**The cost:** most TCP options are lost — there are only **32 bits of ISN** to use, so **window scale, SACK and timestamps are mostly lost** for that connection. That's why cookies **turn on only under pressure**, not always.

🧠 On Linux: `sysctl net.ipv4.tcp_syncookies` (default 1 = "use only when the SYN queue overflows"). Modern Linux can hide some options in the TCP timestamp field when timestamps are on.

**Homework (two questions on the slide):**
1. TCP SYN cookies are 1996. HTTP cookies are… which came first, and did either name influence the other?
2. Where else have you used this exact trick — put the state into a token the client hands back? (Hints in §7.)

### 4.3 Closing: four segments, not three

```
        Client                                           Server
  close() — the active close
          │── FIN ACK  seq=2000 ack=6000 ───────────────▶│
          │                                              │ CLOSE_WAIT — app may still have data
          │◀─ ACK  ack=2001 ─────────────────────────────│
  FIN_WAIT_2 — half-closed, still receiving              │
          │◀─ …any remaining data the server still owes ─│
          │◀─ FIN ACK  seq=6000 ack=2001 ────────────────│ (server app called close())
          │── ACK  ack=6001 ────────────────────────────▶│
  TIME_WAIT — 2 MSL, then CLOSED                         CLOSED
```

**Half-close — not an error, a real state.** After the first FIN and its ACK the connection is **one-way**. The client sends no more; the server keeps sending until it closes its own side.

**SYN+ACK can be combined. FIN+ACK cannot** (usually). Why? 🧠 On open, the server has nothing to wait for — it can ACK and SYN together. On close, the server's kernel ACKs the FIN *at once*, but its own FIN must wait until **its application** calls `close()`, maybe much later. (If the app closes instantly, the kernel can sometimes merge them, giving a 3-segment close.)

**Sockets stuck in CLOSE_WAIT:** almost always **an app that forgot to call `close()`**. Not a network problem. *"Look at your code, not your switch."*

**TIME_WAIT** is why you get **"address already in use"** on server restart — and why **`SO_REUSEADDR`** exists.

🧠 Why TIME_WAIT exists: (1) if the final ACK is lost, the peer resends its FIN and we must still be there to ACK it again; (2) old late segments from this connection must die out before a new connection reuses the same 4-tuple. Only the side that closes **first** (the active closer) enters TIME_WAIT.

🧠 State cheat-sheet for `ss -tan` / `netstat`:

| State | Who | Meaning |
|---|---|---|
| LISTEN | server | waiting for SYNs |
| SYN-SENT | client | sent SYN, waiting |
| SYN-RECV | server | half-open |
| ESTABLISHED | both | open |
| FIN-WAIT-1/2 | active closer | sent FIN / got its ACK |
| CLOSE-WAIT | passive closer | got FIN, **app hasn't closed yet** |
| LAST-ACK | passive closer | sent its FIN, waiting for ACK |
| TIME-WAIT | active closer | 2 MSL cool-down |

### 4.4 The round-trip tax

> *"Every round trip before your first byte is a tax, and everything modern is an attack on it."*

```
 Protocol         RTTs before first app byte
 ───────────────────────────────────────────────────────────────────────────
 TCP              [RTT]                 One RTT before a byte of app data can move.
 TCP + TLS 1.2    [RTT][RTT][RTT]       Handshake, then ClientHello/ServerHello, then key exchange.
 TCP + TLS 1.3    [RTT][RTT]            One round trip of crypto instead of two.
 TCP Fast Open    [RTT]                 A returning client puts data in the SYN itself.
 QUIC / HTTP3     [RTT]                 Transport and crypto setup folded into a single round trip.
```

> *"Bandwidth you can buy. Round trips you cannot — Mumbai to Virginia is ~190 ms and no amount of money changes the speed of light."*

Worked example: a page fetch Mumbai → Virginia over TCP + TLS 1.2 = 3 × 190 ms = **570 ms before the first byte of HTTP**. With QUIC, ~190 ms. That gap is the whole reason QUIC exists.

🧠 Notes:
- **TLS** = Transport Layer Security (the "S" in HTTPS). QUIC and TLS 1.3 can even do **0-RTT** resumption for returning clients (data in the very first packet), at the cost of replay risk (an attacker resending it).
- **TCP Fast Open (TFO)**: on the first visit the server gives the client a cookie (yes, another cookie); next time the client sends that cookie *and request data* in the SYN, so the server can start work before the handshake ends.

### 4.5 UDP: TCP with every promise removed

> *"The packets look very similar. The handshake is missing and the retries are not as rigorous."*

```
TCP — 20 B header, minimum
┌────────┬────────┬─────┬─────┬───────┬────────┬──────────┬────────┬─────────┬─────────┐
│Src Port│Dst Port│ Seq │ Ack │ Flags │ Window │ Checksum │ Urgent │ Options │ Payload │
│   2B   │   2B   │ 4B  │ 4B  │  2B   │   2B   │    2B    │   2B   │ 0–40 B  │  ≤ MSS  │
└────────┴────────┴─────┴─────┴───────┴────────┴──────────┴────────┴─────────┴─────────┘

UDP — 8 B header, that's all of it
┌──────────┬──────────┬────────┬──────────┬─────────────────────┐
│ Src Port │ Dst Port │ Length │ Checksum │ Payload             │
│    2B    │    2B    │   2B   │    2B    │ up to 65,507 B      │
└──────────┴──────────┴────────┴──────────┴─────────────────────┘
```

🧠 Where 65,507 comes from: the IPv4 Total Length field is 16 bits → max 65,535 B packet − 20 B IP header − 8 B UDP header = **65,507 B**. (In practice anything over ~1472 B gets split into IP fragments on Ethernet, which you usually want to avoid.)

🧠 The TCP "Flags 2B" box really packs the Data Offset (header length, 4 bits), reserved bits and the flag bits (CWR, ECE, URG, ACK, PSH, RST, SYN, FIN) into those 16 bits.

**Who really uses it:** Zoom, WebRTC, game servers, DNS, and every real-time streaming protocol. **A resent audio frame arrives too late to play. Dropping it is the right thing.**

**The broadcast promise** (the instructor's own view): *"I liked UDP for broadcast — easy system discovery on a LAN, no registry, no config. In practice people just hard-coded the IPs they had already allocated. Some popular systems did use it. It never became what I expected."* (🧠 examples of UDP broadcast/multicast discovery: DHCP, mDNS/Bonjour, SSDP.)

**Homework:** draw the UDP packet from memory, then capture a real one and compare it with your drawing. Then answer: **with only length and checksum, what must an application rebuild itself?** (Hints in §7.)

**Gotcha:** "UDP is unreliable so it's bad" — no. It's a *blank canvas*. Real-time apps want to choose for themselves what to retry; QUIC builds a full reliable transport on top of it.

### 4.6 QUIC rebuilds TCP inside UDP, in userspace

```
 HTTP/1.1 · HTTP/2 over TCP               HTTP/3 over QUIC
 ┌────────┬───────────────────────────┐   ┌────────┬─────────────────────────────────┐
 │ HTTP/2 │ streams, multiplexed      │   │ HTTP/3 │ same streams, mapped onto QUIC's│
 ├────────┼───────────────────────────┤   ├────────┼─────────────────────────────────┤
 │ TLS    │ separate handshake,       │   │ QUIC   │ independent streams + TLS 1.3,  │
 │        │ separate round trips      │   │        │ one handshake                   │
 ├────────┼───────────────────────────┤   ├────────┼─────────────────────────────────┤
 │ TCP    │ one ordered byte stream — │   │ UDP    │ 8 bytes the kernel and the      │
 │        │ one lost packet stalls    │   │        │ middleboxes leave alone         │
 │        │ every stream              │   │        │                                 │
 ├────────┼───────────────────────────┤   ├────────┼─────────────────────────────────┤
 │ IP     │                           │   │ IP     │                                 │
 └────────┴───────────────────────────┘   └────────┴─────────────────────────────────┘
```

**Head-of-line blocking, gone.** HTTP/2 sends many streams together over one TCP connection — but TCP still delivers **one ordered byte stream**, so one lost packet stops *all* of them. QUIC gives **each stream its own numbering**. Lose a packet, only that stream waits.

> 💡 **Simple words mein:** TCP = ek hi railway ticket counter ki line; aage wale ka kaam atka toh sab ruk gaye. QUIC = har stream ki apni alag line; ek atki toh baaki chalti rahengi.

```
 HTTP/2 over TCP:   [img1][css][img1][js]  ← packet with "css" lost
                    js and img1 are sitting in the kernel, but TCP won't release
                    them until css is retransmitted. EVERYONE waits.

 HTTP/3 over QUIC:  only the css stream waits; img1 and js are delivered now.
```

**Why userspace matters.** TCP lives in the kernel — changing it means **shipping an OS update to the whole planet** and hoping no **middlebox** complains. QUIC ships inside the browser. *"Google can deploy a congestion-control change on Tuesday."*

🧠 **Middlebox** = anything in the path that checks/changes packets: firewalls, NATs, load balancers. Many drop TCP packets with unknown options, which "ossified" (froze) TCP. QUIC encrypts almost everything, so middleboxes can't interfere — they just see UDP.

**Where it really is (instructor's view):** people are still adopting HTTP/2. HTTP/3 is not yet QUIC-everywhere. *"My read: media over QUIC becomes widespread in the next two to five years."*

🧠 Bonus QUIC features: **connection IDs** (your connection survives a switch from Wi-Fi to 4G, as it isn't tied to the 4-tuple), and **0-RTT** resumption.

---

## 5 · Speak it by hand — SMTP, POP3, IMAP, FTP

> *"Last time we built binary protocols. These three are text — so you can be the client."*

### 5.1 SMTP: the S is Simple — it has never stood for Secure

Sending mail is a conversation, and **both sides speak human-readable ASCII**.

| Port | Name | Use |
|---|---|---|
| **25** | SMTP relay | server to server, plaintext |
| **587** | Submission | client to server, **STARTTLS — use this one** |
| **465** | SMTPS | implicit TLS from the first byte |
| **110 / 143** | POP3 / IMAP | retrieval (coming up next) |

🧠 **STARTTLS** = start in plaintext, then the client sends the `STARTTLS` verb and both sides upgrade the same connection to TLS. **Implicit TLS** = TLS starts at once, before any SMTP is spoken.

**Why it stayed text:** every relay in between must be able to **read, rewrite and add headers**. A text protocol with a fixed delimiter is easy to extend — add a header nobody understands and nothing breaks. That's why forty years of extensions fitted in cleanly: STARTTLS, SPF, DKIM, DMARC "are all just headers and verbs."

**Why it stayed insecure:** **`MAIL FROM` is whatever you type.** The base protocol has no authentication at all — you'll see this in the demo when you send mail as anyone you like. SPF, DKIM and DMARC are all later patches trying to answer a question SMTP never asked: *is this sender allowed to say that?*

🧠 The three patches:
- **SPF** (Sender Policy Framework): the domain lists, in DNS, which IP addresses may send its mail.
- **DKIM** (DomainKeys Identified Mail): the sending server signs the message; the signature goes in a `DKIM-Signature:` header and the public key lives in DNS.
- **DMARC** (Domain-based Message Authentication, Reporting and Conformance): the domain's DNS policy saying "if SPF/DKIM fail, reject/quarantine, and send me reports." (Exactly: SPF and DMARC *policies* live in DNS; their *results* show up as headers like `Authentication-Results:`.)

**Try it (from the slide):** open a real message in Gmail → "Show original" and read the headers before writing any yourself.

### 5.2 MIME + Base64: images travel as text, at a 33% tax

> *"A text protocol cannot carry arbitrary bytes. So the bytes stop being arbitrary."*

**Plain English:** SMTP lines can safely carry only printable characters. So binary files are *re-spelled* using just 64 safe characters. Every 3 bytes become 4 characters.

```
 3 BYTES IN  ──▶  4 CHARACTERS OUT

   'M'        'a'        'n'
   0x4D       0x61       0x6E
 01001101   01100001   01101110          24 bits
   │
   ▼ regroup as 6-bit chunks
 010011   010110   000101   101110
   19       22        5       46
   T        W         F       u          → "TWFu"
```

**24 bits regrouped as 6 — and 6 bits maps onto 64 safe printable characters.**

- **The 64 characters:** `A–Z a–z 0–9 + /` and `=` for padding. Picked because every mail system, gateway and 7-bit link ever built passes them through unchanged.
- **4/3 size on the wire** → ≈ **33% overhead, always**.
- **76 characters per line** — the RFC 2045 line-length limit.

🧠 Index table: 0–25 = `A`–`Z`, 26–51 = `a`–`z`, 52–61 = `0`–`9`, 62 = `+`, 63 = `/`. If the input isn't a multiple of 3 bytes, the output is padded with one or two `=`.

**How the parts are separated (MIME multipart):**
```
Content-Type: multipart/mixed; boundary="=_a7f3c91"

--=_a7f3c91
Content-Type: text/plain

…the message…
--=_a7f3c91
Content-Type: image/png
Content-Transfer-Encoding: base64

iVBORw0KGgoAAAANSUhEUg…
--=_a7f3c91--
```
- `--boundary` starts each part; `--boundary--` (two trailing dashes) ends the whole thing.
- *"Same delimitation idea we used for binary protocols last session — pick a boundary that cannot appear in the body."*
- 🧠 **MIME** = Multipurpose Internet Mail Extensions. Fun fact: `iVBORw0KGgo` is what *every* PNG starts with in base64 (it's the PNG magic bytes `89 50 4E 47 0D 0A 1A 0A`).

**Gotcha:** base64 is **encoding, not encryption**. Anyone can decode it.

### 5.3 Demo setup: a whole mail server in a container

```bash
# mailpit — SMTP + web UI
podman run -d \
  --name mailpit \
  -p 1025:1025 \
  -p 8025:8025 \
  axllent/mailpit
# SMTP on 1025, web UI on 8025
# Check mail at: http://localhost:8025

# greenmail — SMTP + POP3
podman run -d \
  --name greenmail \
  -e 'GREENMAIL_OPTS=-Dgreenmail.setup.test.all
      -Dgreenmail.hostname=0.0.0.0
      -Dgreenmail.users=bob:secret@localhost
      -Dgreenmail.verbose' \
  -p 3025:3025 \
  -p 3110:3110 \
  docker.io/greenmail/standalone:2.1.12
# SMTP on 3025, POP3 on 3110, user bob / secret
```

| | mailpit | greenmail |
|---|---|---|
| What | Accepts anything, delivers nothing, shows every message in a browser | A real mailbox with a real user (`bob` / `secret`) |
| Ports | SMTP 1025, web UI 8025 | SMTP 3025, POP3 3110 |
| Best for | watching what you just sent | **the one we type into** (and fetch back via POP3) |

- Docker works the same — swap `podman` for `docker`. Neither container touches the internet; nothing you send goes anywhere.
- 🧠 `-p 3025:3025` = "map host port 3025 to container port 3025". `-d` = run in background. Greenmail's `setup.test.all` uses the "test" ports (standard port + 3000: 25→3025, 110→3110), so it doesn't need root.

### 5.4 Demo: be the mail client (SMTP by hand)

```
$ telnet localhost 3025
EHLO scaler-demo
MAIL FROM:<jeet@scaler.com>
RCPT TO:<kshitij@scaler.com>
DATA
From: Jeet <jeet@scaler.com>
To: Kshitij <kshitij@scaler.com>
Subject: SMTP is surprisingly simple

Hello K,

We just sent this email by manually speaking SMTP.
.
QUIT
```

🧠 What the server says back (typical — exact text varies by server):
```
220 ... ESMTP ready                      ← greeting on connect
EHLO scaler-demo
250-... Hello scaler-demo                ← lists extensions it supports
250 ...
MAIL FROM:<jeet@scaler.com>
250 OK
RCPT TO:<kshitij@scaler.com>
250 OK
DATA
354 Start mail input; end with <CRLF>.<CRLF>
...headers, blank line, body...
.
250 OK                                   ← message accepted
QUIT
221 Bye
```
🧠 Reply codes: **2xx** = success, **3xx** = "go on, send more", **4xx** = temporary failure (try later), **5xx** = permanent failure. `EHLO` = Extended HELLO (the modern version of `HELO`).

**The envelope vs the letter**
```
 ┌─────────── ENVELOPE (SMTP commands) ─────────────┐
 │ MAIL FROM:<jeet@scaler.com>                      │  decides where it GOES
 │ RCPT TO:<kshitij@scaler.com>                     │  (and where bounces return)
 │   ┌──────── LETTER (inside DATA) ─────────────┐  │
 │   │ From: Jeet <jeet@scaler.com>              │  │  just text printed on the letter —
 │   │ To: Kshitij <kshitij@scaler.com>          │  │  what your mail client SHOWS you
 │   │ Subject: ...                              │  │
 │   │                                           │  │
 │   │ body                                      │  │
 │   └───────────────────────────────────────────┘  │
 └──────────────────────────────────────────────────┘
   They do NOT have to match. NOTHING CHECKS.
```
🧠 That's exactly how BCC works (recipient in the envelope, not on the letter) — and how spoofing (faking the sender) works.

> 💡 **Simple words mein:** Courier wala sirf lifafe ke bahar ka address (envelope) dekhta hai; andar ki chitthi pe "From: PM sahab" likh do, koi check nahi karega. Isliye fake mail itna aasaan hai.

**The lone dot.** A single `.` on its own line ends the message. That's the delimiter — and why **a body line starting with a dot gets an extra dot added** (dot-stuffing; the receiver removes it). **Blank line between headers and body:** miss it and your subject becomes body text.

🧠 Gotchas when typing it:
- SMTP lines formally end with CRLF (`\r\n`). `telnet` sends CRLF; `nc` sends only LF unless you use `nc -C` (on some versions `--crlf`). Some strict servers reject bare LF. If you mistype, Backspace may send odd bytes over telnet; just retype the line.

### 5.5 POP3: four verbs you can hold in your head

> *"IMAP is more popular. POP is easier to understand — so learn this one first."*

```
$ telnet localhost 3110
USER bob
PASS secret
STAT
LIST
RETR 1
DELE 1
QUIT
```

| Verb | Meaning |
|---|---|
| `STAT` | how many messages, how many bytes |
| `LIST` | one line per message: number and size |
| `RETR n` | send me message *n*, in full |
| `DELE n` | mark *n* for deletion — **applied at QUIT** |

**Plaintext password:** `USER` and `PASS` travel over the wire unencrypted. **POP3S on 995** exists for exactly this reason.

🧠 Expected replies: every response starts with `+OK` or `-ERR`. `STAT` → `+OK 1 512` (1 message, 512 bytes). `RETR 1` → `+OK`, then the whole message, then a line with just `.` (the same dot delimiter as SMTP, with dot-stuffing). If you disconnect without `QUIT`, the `DELE` is *not* applied (🧠 deletions only happen in the "UPDATE" state that QUIT starts).

**The exercise from the slide:** send the message from the SMTP slide to **`bob@localhost`** via port **3025**, then fetch it back on **3110**. *"Same bytes, two protocols."* (So in the SMTP demo, use `RCPT TO:<bob@localhost>`.)

### 5.6 IMAP vs POP3

> *"IMAP fixes a lot of problems with POP — you only feel them once you have used POP."*

| | **POP3 · port 110** | **IMAP · port 143** |
|---|---|---|
| Where the mail lives | On your device. The server is a spool to be emptied. | On the server. The client holds a cache. |
| Second device | Sees nothing. The first client already took it. | Same mailbox, same state, same moment. |
| Read / unread | Local only. Nobody else knows what you opened. | A server-side flag: `\Seen`, `\Answered`, `\Flagged`. |
| Folders | There is one. It is called the inbox. | Arbitrary hierarchy, created and renamed over the wire. |
| Search | Download everything first, then search locally. | `SEARCH` runs on the server. `IDLE` pushes new mail to you. |
| Partial fetch | No. `RETR` gives you the whole message, attachments and all. | `FETCH BODY[1]` — one MIME part, headers only, whatever you want. |

> *"The trade: POP puts the state on your machine, IMAP puts it on the server. Everything else follows from that one decision."*

🧠 **IMAP** = Internet Message Access Protocol. IMAPS = port 993. Each IMAP command starts with a tag the client picks, so replies can be matched: `a1 LOGIN bob secret`, `a2 SELECT INBOX`, `a3 FETCH 1 BODY[HEADER]`.

### 5.7 Framing: length or delimiter (or both)

> *"Every protocol today picked one, and some picked both."*

**Plain English:** a receiver reading a stream of bytes must know where one message ends. Either you agree on an "end marker" (delimiter), or you say the size first (length).

| | **DELIMITER** | **LENGTH** | **BOTH** |
|---|---|---|---|
| Idea | Agree on a byte sequence that ends the message. | Put the size in front and read exactly that many bytes. | Delimit the frame, then length-prefix the payload inside it. |
| Examples | **SMTP** a lone `.` on its own line · **HTTP/1.1 headers** CRLF CRLF · **MIME** `--=_boundary--` · **Redis** CRLF between parts | **SMS TPDU** TP-UDL = 12 septets · **HTTP** `Content-Length: 4096` · **SS7 MTP2** LI, and SIF ≤ 273 · **Protobuf** varint length prefix | **IMAP** literals: `{310}` then 310 bytes · **HTTP/1.1** CRLF headers + Content-Length · **HTTP/2** binary framing over a stream · **POP3** `+OK` line, then a dotted body |
| GOOD | Cheap to write, human-readable, streams without knowing the size in advance. | Content can never be confused for protocol. No escaping, no scanning. | Human-readable control, safe binary payload. The pragmatic answer. |
| COST | You must **escape** the delimiter if it appears in the payload — **dot-stuffing**. Get that wrong and content becomes protocol. | You must know the size before you start sending — which is exactly what **chunked transfer encoding** exists to avoid. | Two parsers, two failure modes, and **request smuggling** lives in the gap between them. |

🧠 Explanations:
- **CRLF** = Carriage Return + Line Feed = `\r\n`. HTTP headers end at the first empty line (`\r\n\r\n`).
- **Varint** = a variable-length integer: small numbers take 1 byte, bigger ones more.
- **Chunked transfer encoding** (HTTP/1.1): send the body as `size\r\n data\r\n` chunks, ending with a `0` chunk — length-prefix piece by piece, for when you don't know the total size yet.
- **Request smuggling**: if a front proxy decides where a request ends by one rule (say `Content-Length`) and the backend uses another (`Transfer-Encoding: chunked`), an attacker can hide a second request inside the first.
- 🧠 Redis's protocol (RESP) also uses lengths for bulk strings (`$5\r\nhello\r\n`), so you could call it "both" too.

### 5.8 FTP: refuses to mix commands and data at all

> *"One connection for the conversation. A second, brand-new connection for every file."*

**Active mode — the original design, 1971:**
```
  Client                                         Server
    │── control connection ──▶ port 21  (USER, PASS, LIST, RETR) ──▶│
    │◀── data connection ── FROM port 20, INTO the client ───────────│
```
**The server calls back into you. Every NAT and every firewall on earth now blocks this.**

🧠 In active mode the client first sends `PORT h1,h2,h3,h4,p1,p2` to tell the server where to connect. Behind a home router (NAT), the client's private address can't be reached from outside, so this fails.

**Passive mode — added later, now effectively mandatory:**
```
  Client                                         Server
    │── control connection ──▶ port 21 · PASV ─────────────────────▶│
    │◀── 227 Entering Passive Mode (h,h,h,h,p1,p2) ──────────────────│
    │── data connection ──▶ the port the server just named ─────────▶│
```
**In passive mode the client always opens both connections — the only shape NAT was ever going to allow.**

**Why it exists:** a clean split of control and data — the third example in this lecture (after SS7 and SIP).

### 5.9 Demo setup: a throwaway FTP server

```bash
mkdir -p ftp-demo && echo "hello from ftp" > ftp-demo/hello.txt

podman run --rm -it \
  --name ftp-demo \
  -p 2121:2121 \
  -p 30000-30009:30000-30009 \
  -v "$PWD/ftp-demo:/srv" \
  docker.io/library/python:3.13-slim \
  sh -lc 'pip install -q pyftpdlib &&
    python -m pyftpdlib \
      --interface=0.0.0.0 \
      --port=2121 \
      --directory=/srv \
      --username=scaler \
      --password=demo \
      --write \
      --range=30000-30009 \
      --nat-address=127.0.0.1 \
      --debug'
```
(The PDF-to-text copy added stray spaces like `--interface =0.0.0.0` and `library/ python`; the version above is the correct, runnable one.)

| Flag | Why |
|---|---|
| `--range=30000-30009` | The passive-mode data ports, **fixed to a small range so we can publish them** (`-p 30000-30009:...`). A real server does the same so the firewall rule stays small. |
| `--nat-address=127.0.0.1` | The address the server should **announce in its 227 reply**. Inside a container it doesn't know its own reachable address — so you tell it. |
| `--debug` | Leave this on. The server log is the other half of the conversation you're about to have. |
| `--write` | 🧠 allow uploads/modifications (otherwise read-only). |
| `-v "$PWD/ftp-demo:/srv"` | 🧠 mount your local folder into the container as the FTP root. |

### 5.10 Demo: watch a file arrive on a socket you opened yourself

Two terminals. One for control, one for data.

```
 terminal 1 — control                       terminal 2 — data
 ─────────────────────                      ─────────────────────
 $ telnet localhost 2121
 USER scaler            → 331 ...
 PASS demo              → 230 Login successful
 PWD                    → 257 "/" ...
 PASV
 → 227 Entering passive mode (127,0,0,1,117,48).
                                            $ telnet localhost 30000
                                            # nothing yet — the socket is just open
 LIST                                       # listing appears HERE
                                            (connection closes after the listing)
 PASV                   # a new port for every transfer
 → 227 ... (127,0,0,1,117,49)
                                            $ telnet localhost 30001
 RETR hello.txt                             hello from ftp      # file body appears HERE
```
(🧠 The reply codes on the left are typical FTP codes — 331 "need password", 230 logged in, 257 path, 150/226 transfer start/complete; exact text varies.)

**Decode the 227 yourself:** `(127,0,0,1, 117, 48)`
- The first four numbers are the **IP address**: `127.0.0.1`.
- The last two are the **port, as a big-endian pair**:
```
 port = 117 × 256 + 48 = 29952 + 48 = 30000
 next: (117, 49) → 30001
```

**Why a port is split into two numbers:** the reply is **text**, and in 1971 nobody wanted a client to read a number bigger than 255 out of a comma-separated line. So a 16-bit port is sent as **two 8-bit halves, high byte first — network byte order, written out in ASCII.** *The same `htons()` from the very first slide, done by hand.*

```
 30000 = 0x7530  →  high byte 0x75 = 117,  low byte 0x30 = 48
```

🧠 Gotchas:
- You must connect to the data port **before** (or right after) sending `LIST`/`RETR`, and each passive port works for **one** transfer only.
- The `LIST` output format isn't standard (often just `ls -l` output) — 🧠 that's why `MLSD` was added later.
- Plain FTP sends `PASS` unencrypted, like POP3. 🧠 Use SFTP (over SSH) or FTPS (FTP over TLS) in real life.

---

## 6 · Hands-on: run it yourself

> You're on macOS. Where Linux and macOS differ, both are shown. Install containers with `brew install podman` (then `podman machine init && podman machine start`) or use Docker Desktop.

### 6.1 Sockets + TCP states

```bash
# Compile and run the Lesson-1 echo server (port 2026)
gcc -o echo_server echo_server.c && ./echo_server &
nc localhost 2026          # type a line; it echoes back once, then closes

# See connection states
ss -tan | grep 2026                 # Linux
netstat -an -p tcp | grep 2026      # macOS
lsof -iTCP:2026                     # macOS/Linux: which process owns it
```
**What you'll see:** `LISTEN` for the server; after a client leaves, `TIME_WAIT` on whichever side closed first. Restart the server at once and (without `SO_REUSEADDR`) you may get **`bind: Address already in use`**.

**FIN vs RST experiment:** run the slide's `client.c` (change the port to match your server) while capturing:
```bash
sudo tcpdump -i lo0 -n 'tcp port 2026'     # macOS loopback is lo0; Linux is lo
```
With `SO_LINGER {1,0}` you'll see a segment flagged `[R]` (RST). Remove those two lines and you'll see `[F.]` (FIN) followed by an orderly close.

### 6.2 Watch the 3-way handshake

```bash
sudo tcpdump -i en0 -n -S 'tcp port 80 and host example.com'
curl -s http://example.com > /dev/null
```
Expected (abbreviated):
```
IP me.54321 > ex.80: Flags [S],  seq 1000, win 65535, options [mss 1460,...,wscale 6,...,sackOK]
IP ex.80 > me.54321: Flags [S.], seq 5000, ack 1001, ...
IP me.54321 > ex.80: Flags [.],  ack 5001
IP me.54321 > ex.80: Flags [P.], seq 1001:1076, ack 5001     ← GET / ...
...
IP me.54321 > ex.80: Flags [F.], ...
```
`-S` prints absolute sequence numbers so you can verify **ack = seq + 1** for the SYN. (Flags: `S` SYN, `.` ACK, `P` PSH, `F` FIN, `R` RST.)

### 6.3 SYN cookies and congestion control (Linux only)

```bash
sysctl net.ipv4.tcp_syncookies              # 1 = use when the SYN queue overflows
sysctl net.ipv4.tcp_congestion_control      # usually cubic
sysctl net.ipv4.tcp_available_congestion_control
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr   # try BBR (needs the tcp_bbr module)
ss -ti                                      # per-connection cwnd, rtt, and the algorithm in use
```

### 6.4 Capture a UDP packet (homework #1)

```bash
sudo tcpdump -i en0 -n -X 'udp port 53' &
dig @1.1.1.1 example.com
```
`-X` shows hex. After the 20-byte IP header you'll find the 8-byte UDP header: src port (random), dst port `0035` (= 53), length, checksum, then the DNS query. Or open it in Wireshark for a field-by-field breakdown.

### 6.5 The round-trip tax, measured

```bash
curl -o /dev/null -s -w 'tcp: %{time_connect}s  tls: %{time_appconnect}s  first-byte: %{time_starttransfer}s\n' https://example.com
ping -c 5 example.com          # one RTT
```
`time_appconnect − time_connect` ≈ TLS cost (1 RTT for TLS 1.3). Try a far-away server to feel the speed-of-light limit.
🧠 If your curl was built with HTTP/3: `curl --http3 -w ... https://cloudflare.com`.

### 6.6 Base64 by hand

```bash
printf 'Man' | base64                 # TWFu
printf 'Ma'  | base64                 # TWE=   (one pad)
printf 'M'   | base64                 # TQ==   (two pads)
head -c 3000 /dev/urandom | base64 | wc -c    # ≈ 4000+ chars → the 33% tax (plus newlines)
```

### 6.7 SMTP → POP3 round trip (greenmail)

```bash
podman run -d --name greenmail \
  -e 'GREENMAIL_OPTS=-Dgreenmail.setup.test.all -Dgreenmail.hostname=0.0.0.0 -Dgreenmail.users=bob:secret@localhost -Dgreenmail.verbose' \
  -p 3025:3025 -p 3110:3110 docker.io/greenmail/standalone:2.1.12

telnet localhost 3025     # EHLO, MAIL FROM, RCPT TO:<bob@localhost>, DATA, ..., ., QUIT
telnet localhost 3110     # USER bob, PASS secret, STAT, LIST, RETR 1, DELE 1, QUIT
```
🧠 macOS no longer ships `telnet`: `brew install telnet`, or use `nc -c localhost 3025` (BSD nc's `-c` sends CRLF). Try putting a line starting with `.` in your body and see it come back intact after RETR — dot-stuffing at work. Try `MAIL FROM:<ceo@anything.com>` — nothing stops you.

For the mailpit variant, send to port 1025 instead and watch it appear at `http://localhost:8025`.

### 6.8 FTP by hand

Run the pyftpdlib container (§5.9), then follow §5.10 with two terminals. Watch the `--debug` log in the server terminal as you type.

### 6.9 SIP call capture (homework #4)

Install Asterisk (or FreeSWITCH), register two softphones (Linphone/Zoiper), place a call while Wireshark captures. Filter `sip` to see INVITE/100/180/200/ACK/BYE; filter `rtp` for media. **Telephony → VoIP Calls** in Wireshark will list the call and play its audio.

---

## 7 · Homework (four things to go and find out)

### HW1 · Draw the UDP packet from memory
**Task:** draw it, then capture a real one and compare with your drawing. Note what an application must rebuild for itself with only length and checksum.

**Approach / hints:**
- 4 fields × 2 bytes: src port, dst port, length (header + data), checksum. Then payload.
- Capture with §6.4. Check: is `length` = 8 + payload size? Is the checksum non-zero? (🧠 In IPv4 a zero checksum means "not calculated"; it's required in IPv6.)
- What's missing vs TCP, which the *app* must rebuild if it needs it: sequence numbers (ordering, duplicate detection), ACKs and retransmission (reliability), connection setup/teardown, flow control, congestion control, message fragmentation beyond one datagram, and security. Think about what DNS does (a query ID + timeout + retry) and what QUIC does (all of it).

### HW2 · SYN cookies (1996) vs HTTP cookies
**Task:** which came first, did either name influence the other, and list three other places you have pushed state into a token the client hands back.

<details><summary>Hints (open after trying)</summary>

- 🧠 HTTP cookies came first: Lou Montulli at Netscape, 1994. SYN cookies: D. J. Bernstein (with Eric Schenk), 1996. Verify the names' origin yourself — both are usually traced to the older programmer term **"magic cookie"** (an opaque token passed back and forth, e.g. X11's MIT-MAGIC-COOKIE).
- Places the "state-in-a-token" trick shows up: **JWTs** / signed session cookies, **CSRF tokens**, pagination **cursors** ("next_page_token"), OAuth `state` parameters, signed download URLs (S3 presigned URLs), TCP Fast Open cookies, QUIC **Retry tokens**, DNS cookies.
- The common pattern: server signs/hashes the state with a secret → client keeps it → server checks it on return, so the server remembers nothing.
</details>

### HW3 · Unpack "Hello World!" by hand
**Task:** `C8 32 9B FD 06 5D DF 72 36 39 04` — eleven octets, twelve characters. Recover the first three by hand, then write the unpacker.

**Approach:**
1. Write each octet in binary.
2. Character 1 = the **low 7 bits** of octet 1.
3. Character 2 = the **top 1 bit** of octet 1 (as its lowest bit) + the **low 6 bits** of octet 2 (shifted up by 1).
4. Character 3 = the top 2 bits of octet 2 + the low 5 bits of octet 3 (shifted up by 2)… and so on.
5. Map each 7-bit value through the GSM-7 table (for these characters = ASCII).

<details><summary>Check your first three characters</summary>

```
C8 = 1100 1000   → low 7 bits: 100 1000 = 0x48 = 'H'
                   leftover top bit: 1
32 = 0011 0010   → low 6 bits: 11 0010, shifted left 1, plus leftover 1
                   = 110 0101 = 0x65 = 'e'
                   leftover top 2 bits: 00
9B = 1001 1011   → low 5 bits: 1 1011, shifted left 2, plus leftover 00
                   = 110 1100 = 0x6C = 'l'
```
</details>

**Unpacker hint (pseudocode):**
```
carry = 0; carry_bits = 0; out = []
for each octet b:
    value = ((b << carry_bits) | carry) & 0x7F
    out.append(value)
    carry = b >> (7 - carry_bits)
    carry_bits += 1
    if carry_bits == 7:          # a whole extra septet is sitting in carry
        out.append(carry & 0x7F)
        carry = 0; carry_bits = 0
stop after TP-UDL (= 12) characters
```
Test it: the output must be exactly 12 characters and the last one is `!`. Then try the reverse (a packer) and check you get the same 11 bytes.

### HW4 · Run Asterisk, place a SIP call, capture it
**Task:** two softphones and Wireshark. Telephony → VoIP Calls will play the audio straight out of your own capture. Draw your own conclusions about RTP.

**Hints:** see §6.9. Things to notice: SIP messages are readable text; the SDP body names the IP:port for media; RTP flows **directly between the phones** (not through Asterisk, unless it's set to relay media); RTP (unless SRTP) is **unencrypted**, which is why Wireshark can play it back. 🧠 Also look at RTP sequence numbers and timestamps — RTP adds back a bit of what UDP lacks (ordering, timing) but not resending.

---

## 8 · What to keep (the four takeaways)

| Idea | The slide's line |
|---|---|
| **Encapsulation** | One layer's headers are another layer's body. Ethernet ⊃ IP ⊃ TCP ⊃ HTTP is the same picture as MTP2 ⊃ MTP3 ⊃ SCCP ⊃ TCAP ⊃ MAP ⊃ TPDU. Two committees, fifty years apart, one idea. |
| **Framing** | Length or delimiter. SMS chose length, SMTP chose a dot, IMAP chose both. There is no third option. |
| **Reliability has an address** | SS7 put it at L2, per hop. TCP put it at L4, end to end. QUIC put it in userspace. All three were right for their network. |
| **Control and data want separating** | Blue boxes died when SS7 split them. SIP keeps media off the proxy. FTP opens a second socket. Same instinct, three eras. |

---

## 9 · Summary cheat-sheet

| Term | One-line meaning |
|---|---|
| `htons` | Convert a 16-bit number to big-endian (network byte order). |
| `listen(fd, n)` | Make a socket passive; `n` = accept-queue depth, not a connection limit. |
| `connect()` | Client call that triggers the 3-way handshake; kernel picks an ephemeral port. |
| FIN | Polite close: "I'm done sending." Leads to TIME_WAIT on the active closer. |
| RST | Abort: tear down now, discard unsent data. `SO_LINGER {1,0}` forces it. |
| TIME_WAIT | 2 MSL cool-down after active close; causes "address already in use". |
| SO_REUSEADDR | Lets a server rebind a port with connections still in TIME_WAIT. |
| CLOSE_WAIT | Peer closed, your app hasn't — almost always a missing `close()`. |
| OSI | 7-layer reference model (1984): Physical, Data link, Network, Transport, Session, Presentation, Application. |
| CSMA/CD | Ethernet: detect collisions while transmitting. |
| CSMA/CA | Wi-Fi (802.11): can't detect, so sense, random backoff, transmit, per-frame ACK. |
| LEO / Starlink | ~550 km, ~20 ms; just another L1 — IP header unchanged except TTL/checksum/NAT. |
| MTU / MSS | 1500 B max L2 payload / 1460 B max TCP payload (1500 − 20 − 20). |
| 1518 B | Full Ethernet frame: 1500 + 14 header + 4 FCS. |
| EtherType 0x0800 | "Payload is IPv4." |
| SS7 | Out-of-band signalling network for phones (1975). |
| In-band signalling | Control tones (2600 Hz, MF) in the voice channel → blue-box fraud. |
| MTP1/2/3 | SS7 physical / link (reliable per hop, CRC-16) / routing (point codes). |
| SCCP | SS7 L4-ish: subsystem numbers, Global Title Translation. |
| TCAP | SS7 dialogues/transactions, ASN.1 BER encoded. |
| MAP | Mobile Application Part: SMS, roaming, location (e.g. opCode 46 mo-ForwardSM). |
| ISUP | Call setup between switches: IAM, ACM, ANM, REL, RLC. |
| SIF 272 | Max SS7 payload per MSU → caps MAP → SMS = 140 octets. |
| GSM-7 | 7-bit alphabet: 140 octets × 8 ÷ 7 = 160 chars. UCS-2 → 70 chars. |
| SMSC | Store-and-forward SMS centre. HLR = subscriber DB. |
| SIP / SDP / RTP | Text call signalling / media description / media transport over UDP. |
| 3-way handshake | SYN, SYN-ACK, ACK — each side's ISN confirmed. SYN & FIN consume a seq number. |
| ISN | Initial Sequence Number; random (RFC 6528) to stop blind injection (Mitnick). |
| SYN flood | Spoofed SYNs fill the half-open backlog. |
| SYN cookie | Encode connection state in the server ISN; store nothing. Loses most options. |
| 4-segment close | FIN, ACK, (data), FIN, ACK; half-close is a real state. |
| RTT tax | TCP 1, TCP+TLS1.2 3, TCP+TLS1.3 2, TFO 1, QUIC 1 RTT before first byte. |
| TFO | TCP Fast Open: returning client sends data in the SYN. |
| BBR | Congestion control using bottleneck bandwidth × RTT, not loss. |
| Reno / CUBIC | Loss-based congestion control: grow until drop, then cut (sawtooth). |
| Bufferbloat | Oversized buffers filled by loss-based TCP → huge latency. |
| UDP | 8-byte header: ports, length, checksum. No handshake, no retries. Max 65,507 B payload. |
| QUIC | Reliable, encrypted, multi-stream transport over UDP, in userspace. |
| Head-of-line blocking | One lost TCP packet stalls every HTTP/2 stream; QUIC fixes it per stream. |
| SMTP ports | 25 relay, 587 submission (STARTTLS), 465 implicit TLS. |
| Envelope vs letter | `MAIL FROM`/`RCPT TO` route; `From:`/`To:` are just displayed text. |
| Lone dot | Ends SMTP DATA / POP3 multi-line replies; dot-stuffing escapes it. |
| SPF / DKIM / DMARC | Later patches for sender authentication (DNS policy, signatures, policy+reports). |
| Base64 | 3 bytes → 4 chars from 64 safe chars; 33% overhead; 76-char lines. |
| MIME boundary | `--boundary` separates parts; `--boundary--` ends. |
| POP3 | USER/PASS, STAT, LIST, RETR, DELE (applied at QUIT); port 110, POP3S 995. |
| IMAP | Server-side state: flags, folders, SEARCH, IDLE, partial FETCH; port 143. |
| Framing | Delimiter (SMTP dot), length (Content-Length, TP-UDL), or both (IMAP literals). |
| Request smuggling | Attack exploiting two parsers disagreeing on message length. |
| FTP active / passive | Server dials client from port 20 (NAT-hostile) / client dials port named in 227. |
| 227 (h,h,h,h,p1,p2) | IP + port = p1 × 256 + p2 (htons in ASCII). |

---

## 10 · Self-check questions

**1. In `listen(server_fd, 1)`, what does the `1` mean, and why does the lecture say to remember it?**
<details><summary>Answer</summary>

It's the accept-queue depth (backlog) — how many connections may wait for `accept()` — not a limit on total connections. It comes back in the SYN-flood part: a small, limited backlog of half-open entries is exactly what a SYN flood fills up, and SYN cookies defend against that.
</details>

**2. What's the difference between closing with FIN and closing with `SO_LINGER {1, 0}`?**
<details><summary>Answer</summary>

Normal `close()` sends FIN: remaining data is delivered, and the active closer waits in TIME_WAIT for 2 MSL. Linger with timeout 0 sends RST: the connection is killed at once, unsent data is thrown away, no TIME_WAIT. Fast but lossy — for demos, not production.
</details>

**3. Why does Wi-Fi use CSMA/CA instead of Ethernet's CSMA/CD?**
<details><summary>Answer</summary>

A radio can't listen while it transmits — its own signal hides everything else — so it can't detect a collision. So it avoids them: carrier sense, random backoff, transmit, and ask for an L2 ACK for every frame (resending if none comes).
</details>

**4. Your packet goes via Starlink to a satellite 550 km up. Which IP header fields change on the way, and why doesn't TCP care about the satellite?**
<details><summary>Answer</summary>

Only TTL (minus 1 per router), the header checksum (recalculated because TTL changed), and any NAT rewrite of addresses/ports. TCP and everything above only see IP packets; the satellite is just a different L1/L2 — that's the point of layering.
</details>

**5. Compute the MSS and full Ethernet frame size for a 1500-byte MTU.**
<details><summary>Answer</summary>

MSS = 1500 − 20 (IPv4 header) − 20 (TCP header) = 1460 B. Frame = 1500 + 14 (Ethernet header: 6+6+2) + 4 (FCS) = 1518 B.
</details>

**6. Why is an SMS limited to 140 bytes, yet you can type 160 characters?**
<details><summary>Answer</summary>

An SS7 MSU's SIF is capped at 272 octets; MAP carries ~200 octets of user data; 140 octets for TP-UD makes sure the whole MAP operation always fits in one MSU — no splitting or joining. GSM-7 uses 7-bit characters packed into one continuous bit stream: 140 × 8 / 7 = 160. With UCS-2 (emoji, non-Latin), it's 140 / 2 = 70.
</details>

**7. What was the security problem with in-band signalling, and what design principle fixed it?**
<details><summary>Answer</summary>

Control tones (2600 Hz, MF digits) went in the same channel as voice, so anyone who could make the sound (blue box, whistle) could control the switch. SS7 (1975) moved signalling to a separate out-of-band network — the control-plane/data-plane split.
</details>

**8. SS7 and TCP both provide reliability. Where does each put it, and what's one downside of SS7's trust model?**
<details><summary>Answer</summary>

SS7 makes every hop reliable at L2 (MTP2: FSN/BSN, CRC-16, retransmission between adjacent nodes). TCP makes only the endpoints reliable, at L4, across arbitrary hops. SS7 has no security — it trusts any node by its point code — so anyone with SS7 access can look up locations or steal SMS.
</details>

**9. Why does the 3-way handshake need exactly three messages, and why does `ack=1001` answer `seq=1000`?**
<details><summary>Answer</summary>

Each direction has its own ISN, and each ISN must be acknowledged: SYN (client ISN), SYN-ACK (ack client + server ISN), ACK (ack server). The server's SYN forces the third leg. SYN uses up one sequence number even without payload, so the ACK is ISN+1 — otherwise a bare SYN couldn't be told apart from a resend.
</details>

**10. How do SYN cookies let a server survive a SYN flood, and what does it cost?**
<details><summary>Answer</summary>

The server stores nothing for a half-open connection. It packs a keyed hash of the 4-tuple, a rough timestamp, and the MSS into its own ISN. The client's final ACK returns ISN+1; the server recalculates and checks the hash, and rebuilds the connection. Cost: only 32 bits are available, so most TCP options (window scale, SACK, timestamps) are mostly lost — so cookies turn on only under pressure.
</details>

**11. Your server has thousands of sockets in CLOSE_WAIT. Network problem or code problem? And what about "address already in use" on restart?**
<details><summary>Answer</summary>

CLOSE_WAIT = the peer sent FIN but your app never called `close()` — a code bug (leaked sockets). "Address already in use" comes from TIME_WAIT left by the active close; `SO_REUSEADDR` allows rebinding.
</details>

**12. Explain head-of-line blocking in HTTP/2 and how QUIC fixes it. Why was QUIC built in userspace over UDP instead of changing TCP?**
<details><summary>Answer</summary>

HTTP/2 sends many streams over one TCP connection, but TCP delivers one ordered byte stream, so one lost packet stops all streams until it's resent. QUIC gives each stream its own numbering, so only the affected stream waits. TCP lives in OS kernels and is frozen in place by middleboxes; changing it means updating every OS. QUIC runs over UDP (which middleboxes leave alone) and ships in the browser/app, so it can be updated fast; it also folds the TLS 1.3 handshake into one RTT.
</details>

**13. In the SMTP demo, what's the difference between `MAIL FROM` and the `From:` header? Why does that make spoofing easy?**
<details><summary>Answer</summary>

`MAIL FROM`/`RCPT TO` are the envelope — they decide where mail goes. `From:`/`To:` inside DATA are just text on the letter that the mail client shows. Nothing in base SMTP checks either, or that they match; SPF, DKIM and DMARC were added later to answer "is this sender allowed to say that?"
</details>

**14. Decode `227 Entering Passive Mode (10,0,0,5,195,80)`. Why is the port written this way?**
<details><summary>Answer</summary>

IP = 10.0.0.5; port = 195 × 256 + 80 = 49920 + 80 = 50000. The reply is text, and in 1971 the designers didn't want clients reading numbers > 255 out of a comma list, so the 16-bit port is sent as two bytes, high byte first — network byte order (htons) written out in ASCII.
</details>

**15. Delimiter vs length framing: give one example of each from the lecture and the classic failure mode of each.**
<details><summary>Answer</summary>

Delimiter: SMTP's lone dot (also HTTP's CRLF CRLF, MIME boundary). Failure: if the delimiter appears in the payload it must be escaped (dot-stuffing), or content becomes protocol. Length: HTTP Content-Length, SMS TP-UDL, MTP2 LI, protobuf varints. Failure: you must know the size before sending (hence chunked encoding). Using both (IMAP literals, HTTP/1.1) gives two parsers that can disagree → request smuggling.
</details>

**16. Why does FTP use a separate connection for data, and why did passive mode become mandatory?**
<details><summary>Answer</summary>

FTP strictly keeps control (commands on port 21) apart from data (a new connection per transfer) — the same control/data split as SS7 and SIP. In active mode the server connects *back* to the client from port 20, which NATs and firewalls block. In passive mode the client opens both connections (to the port named in the 227 reply), the only shape NAT allows.
</details>
