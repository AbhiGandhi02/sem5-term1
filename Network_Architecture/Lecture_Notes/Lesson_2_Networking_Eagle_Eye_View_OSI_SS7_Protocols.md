# 📓 Lesson 2 — Computer Networking, Eagle Eye View
### *CN at Scaler — Session 2* · OSI · SS7 · BBR · TCP · UDP · QUIC · SMTP · POP · FTP

> *"Seven syscalls make a server. Seven layers of nesting carry eleven bytes. Today we open both ends."*

Five stops, one idea running through all of them:

```
 ┌──────────────────────────────────────────────────────────┐
 │   ONE LAYER'S HEADERS ARE ANOTHER LAYER'S BODY —         │
 │   in Ethernet, in SS7, in email, in FTP.                 │
 └──────────────────────────────────────────────────────────┘
```

| # | Stop | Hook |
|---|---|---|
| 01 | Sockets | Six syscalls and you have a server. The seventh is where the bugs live. |
| 02 | OSI | Seven layers — over copper, over radio, and over a satellite 550 km up. |
| 03 | SS7 | A second internet, older than TCP/IP, and why an SMS is 140 bytes. |
| 04 | TCP · UDP · QUIC | Handshake, teardown, SYN cookies, and the round-trip tax. |
| 05 | Text protocols | SMTP, POP and FTP — typed by hand into telnet. |

### Contents
1 Sockets · 2 OSI · 3 SS7 · 4 TCP · UDP · QUIC · 5 Speak it by hand (SMTP, POP3, IMAP, FTP) · 6 Homework solutions · 7 What to keep · 8 Cheat-sheet · 9 MCQs

---

## 1 · Sockets

> *"Everything above this is a convention. Everything below is somebody else's problem."*

### 1.1 The server: a TCP server is seven system calls

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
```

Everything else in this file is address bookkeeping. Two details come back later:

| Detail | What the slide says |
|---|---|
| `listen(fd, 1)` | The `1` is the **accept-queue depth, not the connection limit.** Remember it — it comes back with **SYN floods** (§4.2). |
| `htons(2026)` | **H**ost **to** **n**etwork **s**hort. **The wire is big-endian; your laptop probably isn't.** Comes back in the **FTP 227 reply** (§5.9). |

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

- **No bind, no listen.** The kernel picks an **ephemeral source port** for you.
- **`connect()` is what triggers the handshake.**

### 1.3 `SO_LINGER {1, 0}` — FIN vs RST

| | Normal `close()` | `SO_LINGER {1, 0}` then `close()` |
|---|---|---|
| Sends | **FIN** — a polite "I'm done sending" | **RST** instead |
| Unsent data | delivered | **discarded** |
| Afterwards | socket sits in **TIME_WAIT for 2 MSL** | torn down immediately, **no TIME_WAIT** |
| Verdict | — | Fast. Also lossy. Use it to **demonstrate the difference, not in production.** |

---

## 2 · OSI

> *"A 1984 committee model that has outlived almost everything built on top of it."*

### 2.1 Seven layers, each one refusing to care about the others

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
```

> 💡 **Simple words mein:** Courier parcel socho — gift (HTTP) ko box (TCP) mein, box ko courier bag (IP) mein, bag ko truck (Ethernet) mein. Har level sirf apna label padhta hai, andar kya hai usse matlab nahi.

### 2.2 TCP vs UDP (both at L4)

| | **TCP** | **UDP** |
|---|---|---|
| Connection | ✔ Connection-oriented | ✔ Connectionless |
| Delivery | ✔ Reliable, ordered delivery | ✘ No delivery guarantee, ✘ no ordering |
| Control | ✔ Flow and congestion control, ✔ error recovery | ✔ Fast, low overhead |
| Latency | ⏱ Higher | ⏱ Low |
| Used by | 📦 HTTP, FTP, SMTP, SSH | 📦 DNS, VoIP, video calls, gaming |

### 2.3 L1/L2: on a wire you detect collisions, on radio you can only avoid them

Same L3 and above. Completely different L1 and L2 problem.

```
 CSMA/CA (802.11)
   Want to transmit?
          ▼
   Listen to channel  (carrier sense)
          ▼
   Is someone transmitting? ──YES──▶ wait / defer, listen again
          │ NO
          ▼
   Random backoff  (contention window)
          ▼
   Transmit  (frame goes out over radio)
          ▼
   ACK received? ──NO──▶ back off and retransmit
          │ YES
          ▼
       Success
```

- **Why not CSMA/CD:** a radio **cannot listen while it transmits** — its own signal drowns everything else out. So Wi-Fi cannot detect a collision the way Ethernet can. It **avoids** instead: sense, back off, transmit, and demand an **explicit ACK for every single frame**.
- **Where the time goes:** **≈2 ms of contention delay** before a frame even leaves. Every frame is ACKed **individually at L2** — long before TCP at L4 has any idea anything happened.

### 2.4 Your packet can go to space and back, and nothing above L2 notices

> *"Starlink is a physical layer. That is the entire point of having layers."*

```
 💻 Your laptop
   │  Ethernet / Wi-Fi        (local network)
 🛰 Starlink terminal         (phased-array dish)
   │  RF uplink               (Ku / Ka-band)
 🛸 LEO satellite             (~550 km · ~20 ms)
   │  🔴 Laser crosslink      (optional, sat-to-sat)
 🏭 Ground gateway            (terrestrial handoff)
   │
 ☁  The internet              (same IP packet)
```

**The whole argument for layering:** nine physical hops, one of them 550 km straight up, and the IP header written by your laptop arrives **unchanged — except TTL, checksum and any NAT rewrite**. L3 and above never learn that space was involved.

---

## 3 · SS7

> *"There is a second internet. You used it this morning and you have never seen a packet from it."*

### 3.1 SS7's one big idea (1975): get the signalling out of the voice channel

**Before — in-band signalling.** One channel carries both the conversation and the control:

```
 ┌─────────── ONE CHANNEL ─────────────────────────────────────┐
 │  Voice        2600 Hz tone          Voice      Dialled digits│
 │  what you say "this trunk is idle"  what you say  MF tones   │
 └─────────────────────────────────────────────────────────────┘
   Anyone who can MAKE THE SOUND can MAKE THE DECISION.
```

**After — out-of-band signalling (SS7, 1975):**

```
  ════════ Voice bearer — 64 kbps DS0 ═════════════  the conversation, and nothing else
  ──────── Signalling network — MTP / SCCP / TCAP / MAP / ISUP ────
           a separate packet-switched network the caller never touches
```

> *"Same separation of concerns as a control plane and a data plane. Fifty years old, and every network you build still does it."*

> 💡 **Simple words mein:** Pehle baat-cheet aur control ek hi line pe the — jo bhi sahi awaaz (2600 Hz) bana sake, woh switch ko control kar sakta tha. SS7 ne control ko alag network pe daal diya; caller usse chhoo hi nahi sakta.

### 3.2 SS7 vs TCP/IP: same seven jobs, different names, very different trust model

| OSI | SS7 stack | what it does | TCP/IP stack | what it does |
|---|---|---|---|---|
| **L7** | MAP · INAP · CAP · ISUP | Call setup, roaming, SMS | HTTP · DNS · SMTP · SIP | Application protocols |
| **L5–L6** | TCAP | Dialogues, components, ASN.1 BER | TLS / app framing | No distinct layer — folded into the app |
| **L4** | SCCP (classes 0–3) | Subsystem numbers, Global Title Translation | TCP / UDP | Ports, byte-stream sequencing, windowing |
| **L3** | MTP3 | Point-code routing, SLS load sharing | IP | Best-effort datagrams, fragmentation, TTL |
| **L2** | MTP2 | Signal units, FSN/BSN, CRC-16, retransmission | Ethernet · PPP · HDLC | Framing, MAC addressing, FCS |
| **L1** | MTP1 | 64 kbps DS0 on E1 / T1 | Copper · fibre · radio | Any bit-carrying medium |

> *SS7 is a whole stack; TCP is one protocol at L4. The fair comparison is stack to stack.*

### 3.3 Packet shape: one layer's headers are another layer's body

A browser on `192.168.1.42` sending `GET /` to `example.com`. Read top to bottom = peeling the onion at the receiver.

```
L2 — ETHERNET II FRAME
┌─────────┬─────────┬───────────┬──────────────────────────────────┬──────────┐
│ Dst MAC │ Src MAC │ EtherType │ Payload — the entire IP packet   │   FCS    │
│   6 B   │   6 B   │ 2B 0x0800 │ 46–1500 B (MTU)                  │ 4B CRC-32│
└─────────┴─────────┴───────────┴──────────────────────────────────┴──────────┘
L3 — IPv4 PACKET (20 B header)
┌───────┬─────────┬─────┬───────┬────────┬────────┬────────┬────────────────────┐
│Ver/IHL│Total Len│ TTL │ Proto │Checksum│ Src IP │ Dst IP │ Payload — TCP seg. │
│ 4b+4b │   2B    │ 1B  │6 = TCP│   2B   │   4B   │   4B   │                    │
└───────┴─────────┴─────┴───────┴────────┴────────┴────────┴────────────────────┘
L4 — TCP SEGMENT (20 B header)
┌──────────┬──────────┬─────┬─────┬─────────────┬────────┬──────────────────────┐
│ Src Port │ Dst Port │ Seq │ Ack │    Flags    │ Window │ Payload — app data   │
│    2B    │    2B    │ 4B  │ 4B  │ SYN/ACK/FIN │   2B   │ ≤ MSS, typ. 1460 B   │
└──────────┴──────────┴─────┴─────┴─────────────┴────────┴──────────────────────┘
L7 — APPLICATION DATA
   GET / HTTP/1.1 · Host: example.com
   — an opaque byte stream TCP neither knows nor cares about
   (HTTP request · TLS record · SMTP command · SIP INVITE · anything)
```

**Byte budget:**
```
  1500 B MTU − 20 B IP − 20 B TCP = 1460 B MSS
  + 14 B Ethernet + 4 B FCS  →  1518 B wire frame
```

### 3.4 SS7 encapsulation: seven layers of nesting to carry eleven bytes

A GSM handset sending **"Hello World!"** — **MO-ForwardSM**, all the way down to the octets.

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

### 3.5 An SMS is 140 bytes because a 1980s signalling packet caps at 272

Three independent length limits, stacked. **The smallest one wins, and it defines the product.**

```
 ┌───────────────────────────┐
 │ 272 octets — SIF          │  MTP2's hard ceiling: the physical limit of a single Message Signal Unit
 │  ┌──────────────────────┐ │
 │  │ ~200 octets sm-RP-UI │ │  what MAP is willing to carry as opaque user data
 │  │  ┌────────────────┐  │ │
 │  │  │ 140 octets     │  │ │  TP-UD, the SMS itself: chosen so the whole MAP
 │  │  │ TP-UD          │  │ │  operation always fits in one MSU
 │  │  └────────────────┘  │ │
 │  └──────────────────────┘ │
 └───────────────────────────┘
```

**No segmentation, no reassembly.** One text message is always exactly one signalling packet. Nothing to fragment, nothing to reorder, nothing to hold state for — which is why SMS ran on 1980s hardware, and **why it still works when the data network is down**.

### 3.6 GSM-7: 160 characters in 140 bytes — because ASCII wastes a bit

> *"I said 140 bytes, but you can send 160."* The gap is a packing trick from mobile engineers.

```
  140 octets in TP-UD  ×8 bits/octet  = 1120 bits  ÷7 bits/char  = 160 characters
                                                    (GSM 7-bit alphabet, TP-DCS = 0x00)
```

- Each character is **7 bits, not 8**. The septets are pushed into one continuous bit stream and re-sliced into octets — so after the first character, **character boundaries stop lining up with byte boundaries**.
- **12 chars × 7 bits = 84 bits → 11 octets (88 bits), with 4 bits of zero padding.**
- **When it breaks:** switch TP-DCS to **UCS-2** for emoji or any non-Latin script → **140 ÷ 2 = 70 characters**. Same 140-octet field. One emoji can more than halve your message length — and silently cost you a second SMS.

> 💡 **Simple words mein:** Har byte mein 8 seat hain, par har character ko sirf 7 chahiye. Khaali seat pe agle character ka tukda bitha do — isliye 140 bytes mein 160 characters. Emoji aaya (UCS-2) toh sirf 70.

### 3.7 Sending a text is a database lookup wrapped in a queue

```
 Phone A      VMSC/VLR        SMSC            HLR            MSC B        Phone B
 (handset)  (A's serving    (store &      (subscriber     (B's serving  (handset)
             switch)         forward)      database)       switch)
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

### 3.8 A phone call is signalling on one network, voice on another (ISUP)

```
 Phone A            MSC A                        MSC B              Phone B
 (caller)     (originating switch)        (terminating switch)     (callee)
   │ off-hook + dialled digits ▶│                    │                  │
   │                            │──IAM────────────▶  │ (Initial Address Message:
   │                            │                    │  called number, bearer request) ─ring─▶│
   │                            │◀─ACM─ Address Complete → caller hears ringback
   │                            │                    │◀── B answers ────│
   │                            │◀─ANM─ Answer Message → voice path cut through
   │═══ conversation: 64 kbps bearer circuit, NO SS7 involvement at all ═══│
   │ on-hook ▶                  │──REL (Release)──▶  │                  │
   │                            │◀─RLC (Release Complete) → circuit freed
```

> *"Four messages set up a call. During the conversation itself the signalling network is completely idle — it did its job at the edges."*

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

Try it yourself: Asterisk or FreeSWITCH, a softphone (Linphone, Zoiper), and Wireshark with the SIP dissector on (→ HW4).

### 3.10 SS7 makes every hop reliable. TCP makes only the endpoints reliable.

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

### 3.11 BBR (2016): stop treating packet loss as the signal for congestion

> *"Loss-based TCP fills the buffer until something drops. On a fat, long, or lossy link that is exactly wrong."*

- **The sawtooth is the bug.** Reno and CUBIC grow until a packet is dropped, then **halve**. So they **need loss to make progress** — and on a link that drops packets for reasons other than congestion, they crawl.
- **What BBR measures instead:** **B**ottleneck **B**andwidth × **R**ound-trip propagation time. Send at the rate the path can actually carry, keeping the queue **near empty**. Google reports **up to 2,000× throughput on some benchmarks** — and much lower latency.

> 💡 **Simple words mein:** CUBIC = tab tak tez chalao jab tak takkar (loss) na ho, phir speed aadhi. BBR = pehle naapo sadak kitni chaudi aur lambi hai, phir bilkul utni speed pe chalao.

---

## 4 · TCP · UDP · QUIC

> *"No class with 'computer networks' in the title is complete without the handshake."*

### 4.1 Opening: three messages, because each direction needs its own sequence number confirmed

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

- **Why 3:** Segment 1 announces the client's ISN. Segment 2 acknowledges it and announces the server's. Segment 3 acknowledges the server's. **The server's SYN is what forces the third leg.**
- **SYN and FIN each consume a sequence number.** That's why `ack=1001` answers `seq=1000` even though the SYN carried no payload — otherwise a bare SYN would look the same as a retransmission.
- **Random ISN:** early stacks counted predictably. Guess the next ISN and you can inject forged segments blind — **the Mitnick attack**. **RFC 6528:** keyed hash of the 4-tuple + timer.

> 💡 **Simple words mein:** "Hello, sun rahe ho?" (SYN) → "Haan, tum mujhe sun rahe ho?" (SYN-ACK) → "Haan!" (ACK). Dono ke number confirm — tab baat shuru.

### 4.2 Defence: SYN cookies (1996) — keep no state, and there is nothing to exhaust

Between segment 1 and segment 3, the server holds an entry in a **finite backlog**. **That backlog is the target.**

- **The attack:** send SYNs from **spoofed** source addresses and never complete the handshake. Each costs the attacker one packet and costs the server a half-open entry that lingers until it times out. Fill the backlog and legitimate connections are refused. *Remember `listen(server_fd, 1)` from the first slide.*
- **The defence:** **do not store anything.** Encode the connection parameters into the server's own ISN — a **keyed hash of the 4-tuple, a coarse timestamp and the MSS**. The client's ACK carries that number back, incremented. The server recomputes, validates, and rebuilds the whole connection from a packet it has never seen before.
- **The cost:** most TCP options do not survive — only **32 bits of ISN**, so **window scale, SACK and timestamps are mostly lost** for that connection. That's why cookies **switch on under pressure** rather than always.

> 💡 **Simple words mein:** Register mein entry likhne ki jagah server ek signed token (ISN) de deta hai aur sab bhool jaata hai. Token wapas aaya aur sahi hai? Connection bana do. Register hi nahi, toh flood bekaar.

➡️ Homework: see **HW2 in §6**.

### 4.3 Closing takes four segments, not three

```
        Client                                           Server
  close() — the active close
          │── FIN ACK  seq=2000 ack=6000 ───────────────▶│
          │                                              │ CLOSE_WAIT — app may still have data
          │◀─ ACK  ack=2001 ─────────────────────────────│
  FIN_WAIT_2 — half-closed, still receiving              │
          │◀─ …any remaining data the server still owes ─│
          │◀─ FIN ACK  seq=6000 ack=2001 ────────────────│
          │── ACK  ack=6001 ────────────────────────────▶│
  TIME_WAIT — 2 MSL, then CLOSED
```

- **Half-close — not an error, a real state.** After the first FIN and its ACK the connection is **one-directional**: the client sends no more; the server keeps sending until it closes its own side.
- **SYN+ACK can be combined. FIN+ACK cannot.**
- **Sockets stuck in CLOSE_WAIT:** almost always an application that **forgot to call `close()`**. Not a network problem — *look at your code, not your switch.*
- **TIME_WAIT** is why you get **"address already in use"** on server restart — and why **`SO_REUSEADDR`** exists.

### 4.4 The round-trip tax

> *"Every round trip before your first byte is a tax, and everything modern is an attack on it."*

| Protocol | RTTs before first app byte | Why |
|---|---|---|
| TCP | 1 | One RTT before a single byte of app data can move. |
| TCP + TLS 1.2 | 3 | Handshake, then ClientHello / ServerHello, then key exchange. |
| TCP + TLS 1.3 | 2 | One round trip of crypto instead of two. |
| TCP Fast Open | 1 | A returning client puts data in the SYN itself. |
| QUIC / HTTP3 | 1 | Transport and crypto setup folded into a single round trip. |

> *"Bandwidth you can buy. Round trips you cannot — Mumbai to Virginia is ~190 ms and no amount of money changes the speed of light."*

### 4.5 UDP is TCP with every promise removed

The packets look very similar. The handshake is missing and the retries are not as rigorous.

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

- **Who actually uses it:** Zoom, WebRTC, game servers, DNS, and every real-time streaming protocol. **A retransmitted audio frame arrives too late to play. Dropping it is the correct behaviour.**
- **The broadcast promise (instructor):** UDP looked great for broadcast — easy system discovery on a LAN, no registry, no config. In practice people hard-coded the IPs they had already allocated; it never became what he expected.
- Homework: see **HW1 in §6**.

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

- **Head-of-line blocking, gone.** HTTP/2 multiplexes streams over one TCP connection — but TCP still delivers **one ordered byte stream**, so a single lost packet stalls all of them. QUIC gives **each stream its own sequence space**. Lose a packet, only that stream waits.
- **Why userspace matters.** TCP lives in the kernel — changing it means shipping an OS update to the entire planet and hoping no middlebox objects. QUIC ships in the browser. *"Google can deploy a congestion-control change on Tuesday."*
- **Where it actually is:** people are still adopting HTTP/2; HTTP/3 is not yet QUIC-everywhere. Instructor's read: media over QUIC becomes widespread in the next two to five years.

> 💡 **Simple words mein:** TCP = ek hi line; aage wala atka toh sab ruk gaye. QUIC = har stream ki apni line; ek atki toh baaki chalti rahengi.

---

## 5 · Speak it by hand — SMTP, POP3, IMAP, FTP

> *"Last time we built binary protocols. These three are text — so you can be the client."*

### 5.1 SMTP: the S is Simple — it has never stood for Secure

A mail transfer is a conversation, and both halves of it are human-readable ASCII.

| Port | Name | Use |
|---|---|---|
| **25** | SMTP relay | server to server, plaintext |
| **587** | Submission | client to server, **STARTTLS — use this one** |
| **465** | SMTPS | implicit TLS from the first byte |
| **110 / 143** | POP3 / IMAP | retrieval |

- **Why it stayed text:** every relay in between must be able to **read, rewrite and add headers**. A text protocol with a fixed delimiter is trivially extensible — add a header nobody understands and nothing breaks. That's why forty years of extensions bolted on cleanly: STARTTLS, SPF, DKIM, DMARC are all just headers and verbs.
- **Why it stayed insecure:** **`MAIL FROM` is whatever you type.** No authentication in the base protocol at all. SPF, DKIM and DMARC are later patches trying to answer a question SMTP never asked: *is this sender allowed to say that?*
- Try: open a real message in Gmail → "Show original" and read the headers.

### 5.2 MIME: images travel as text, and it costs you 33%

> *"A text protocol cannot carry arbitrary bytes. So the bytes stop being arbitrary."*

```
 3 BYTES IN  ──▶  4 CHARACTERS OUT
   'M'        'a'        'n'
   0x4D       0x61       0x6E
 01001101   01100001   01101110          24 bits
   ▼ regroup as 6-bit chunks
 010011   010110   000101   101110
   T        W         F       u          → "TWFu"
```

- **24 bits regrouped as 6** — and 6 bits maps onto **64 safe printable characters**: `A–Z a–z 0–9 + /`, and `=` for padding. Chosen because every mail system, gateway and 7-bit link ever built passes them through unchanged.
- **4/3 size on the wire → ≈ 33% overhead, always.**
- **76 characters per line** — RFC 2045 line-length limit.

**How the parts are separated:**
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
Same delimitation idea as the binary protocols last session — **pick a boundary that cannot appear in the body.**

### 5.3 Demo setup: one container gives you a whole mail server

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

| mailpit | greenmail |
|---|---|
| Accepts anything, delivers nothing, shows every message in a browser. Best for watching what you just sent. | A real mailbox with a real user, and POP3 on 3110 to fetch the message back. **The one we type into.** |

Docker works identically — swap `podman` for `docker`. Neither container touches the internet.

### 5.4 Demo: now be the mail client

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

- **Envelope vs letter:** `MAIL FROM` and `RCPT TO` are the **envelope** — they decide where it goes. `From:` and `To:` inside DATA are just text printed on the **letter**; your mail client shows you the letter. **They do not have to match. Nothing checks.**
- **The lone dot:** a single `.` on its own line ends the message. That's the delimiter — and why a body line starting with a dot gets an extra one added.
- **Blank line between headers and body.** Miss it and your subject becomes body text.

> 💡 **Simple words mein:** Courier wala sirf lifafe ka address (envelope) dekhta hai; andar ki chitthi pe "From:" kuch bhi likh do, koi check nahi karta.

### 5.5 POP3: four verbs, and you can hold all of it in your head

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

- **Plaintext password:** `USER` and `PASS` go over the wire in the clear. **POP3S on 995** exists for exactly this reason.
- Exercise: send the SMTP message to **`bob@localhost`** via port **3025**, then fetch it back on **3110**. *"Same bytes, two protocols."*

### 5.6 IMAP fixes a lot of problems with POP — you only feel them once you've used POP

| | **POP3 · port 110** | **IMAP · port 143** |
|---|---|---|
| Where the mail lives | On your device. The server is a spool to be emptied. | On the server. The client holds a cache. |
| Second device | Sees nothing. The first client already took it. | Same mailbox, same state, same moment. |
| Read / unread | Local only. Nobody else knows what you opened. | A server-side flag: `\Seen`, `\Answered`, `\Flagged`. |
| Folders | There is one. It is called the inbox. | Arbitrary hierarchy, created and renamed over the wire. |
| Search | Download everything first, then search locally. | `SEARCH` runs on the server. `IDLE` pushes new mail to you. |
| Partial fetch | No. `RETR` gives you the whole message, attachments and all. | `FETCH BODY[1]` — one MIME part, headers only, whatever you want. |

> *"The trade: POP puts the state on your machine, IMAP puts it on the server. Everything else follows from that one decision."*

### 5.7 Framing: length or delimiter — every protocol picked one, some picked both

| | **DELIMITER** | **LENGTH** | **BOTH** |
|---|---|---|---|
| Idea | Agree on a byte sequence that ends the message. | Put the size in front and read exactly that many bytes. | Delimit the frame, then length-prefix the payload inside it. |
| Examples | **SMTP** a lone `.` on its own line · **HTTP/1.1 headers** CRLF CRLF · **MIME** `--=_boundary--` · **Redis** CRLF between parts | **SMS TPDU** TP-UDL = 12 septets · **HTTP** `Content-Length: 4096` · **SS7 MTP2** LI, and SIF ≤ 273 · **Protobuf** varint length prefix | **IMAP** literals: `{310}` then 310 bytes · **HTTP/1.1** CRLF headers + Content-Length · **HTTP/2** binary framing over a stream · **POP3** `+OK` line, then a dotted body |
| GOOD | Cheap to write, human-readable, streams without knowing the size in advance. | Content can never be confused for protocol. No escaping, no scanning. | Human-readable control, safe binary payload. The pragmatic answer. |
| COST | You must **escape** the delimiter if it appears in the payload — **dot-stuffing**. Get that wrong and content becomes protocol. | You must know the size before you start sending — which is exactly what **chunked transfer encoding** exists to avoid. | Two parsers, two failure modes, and **request smuggling** lives in the gap between them. |

### 5.8 FTP refuses to mix commands and data at all

> *"One connection for the conversation. A second, brand-new connection for every file."*

**Active mode — the original design, 1971:**
```
  Client                                         Server
    │── control connection ──▶ port 21  (USER, PASS, LIST, RETR) ──▶│
    │◀── data connection ── FROM port 20, INTO the client ───────────│
```
The server dials back into you. **Every NAT and every firewall on earth now blocks this.**

**Passive mode — added later, now effectively mandatory:**
```
  Client                                         Server
    │── control connection ──▶ port 21 · PASV ─────────────────────▶│
    │◀── 227 Entering Passive Mode (h,h,h,h,p1,p2) ──────────────────│
    │── data connection ──▶ the port the server just named ─────────▶│
```
In passive mode **the client always opens both connections** — the only shape NAT was ever going to allow.

### 5.9 Demo: a throwaway FTP server, and watching a file arrive

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

| Flag | Why |
|---|---|
| `--range=30000-30009` | Passive-mode data ports, pinned to a small range so we can publish them. A real server does the same so the firewall rule stays finite. |
| `--nat-address=127.0.0.1` | The address the server should advertise in its **227 reply**. Inside a container it doesn't know its own reachable address — so you tell it. |
| `--debug` | Leave this on. The server log is the other half of the conversation. |

```
 terminal 1 — control                       terminal 2 — data
 $ telnet localhost 2121
 USER scaler
 PASS demo
 PWD
 PASV
 → 227 Entering passive mode (127,0,0,1,117,48).
                                            $ telnet localhost 30000
                                            # nothing yet — the socket is just open
 LIST                                       # listing appears HERE
 PASV                   # a new port for every transfer
                                            $ telnet localhost 30001
 RETR hello.txt                             # file body appears HERE
```

**Decode the 227 yourself:** `(127,0,0,1, 117, 48)` — first four = IP `127.0.0.1`; last two = the port as a **big-endian pair**:
```
 port = 117 × 256 + 48 = 30000
```
**Why split the port:** the reply is text, and in 1971 nobody wanted a client to parse a number larger than 255 out of a comma-separated line. So a 16-bit port ships as **two 8-bit halves, high byte first — network byte order, spelled out in ASCII.** The same `htons()` from the very first slide, by hand.

---

## 6 · Homework — four things to go and find out (✅ solutions)

### HW1 · Draw the UDP packet from memory
**Task:** draw it, capture a real one and diff; note what an app must rebuild with only length and checksum.

**✅ Solution**
```
┌──────────┬──────────┬────────┬──────────┬──────────────┐
│ Src Port │ Dst Port │ Length │ Checksum │ Payload      │   8-byte header
│    2B    │    2B    │   2B   │    2B    │ ≤ 65,507 B   │
└──────────┴──────────┴────────┴──────────┴──────────────┘
```
Capture one (a DNS query is the easiest UDP packet):
```bash
sudo tcpdump -i en0 -n -X 'udp port 53'      # terminal 1 (Linux: -i eth0 or -i any)
dig @1.1.1.1 example.com                     # terminal 2
```
In the hex, right after the 20-byte IP header you see 8 bytes: random src port, dst port `0035` (= 53), length (= 8 + DNS payload size), checksum — then the DNS query. Matches the drawing.

**What the app must rebuild itself** (everything TCP gave for free): ordering/sequence numbers, ACKs and retransmission, duplicate detection, connection setup/teardown, flow control, congestion control, splitting big messages. (DNS does the minimum: a query ID + timeout + retry. QUIC rebuilds all of it.)

### HW2 · TCP SYN cookies are 1996. HTTP cookies are…?
**Task:** which came first, did either name influence the other, and three other places you've pushed state into a token the client hands back.

**✅ Solution**
- **HTTP cookies came first** — Lou Montulli at Netscape, **1994**. SYN cookies — D. J. Bernstein, **1996**.
- Names: both borrow the older programmer term **"magic cookie"** (an opaque token you get and hand back unchanged). The SYN cookie is named in the same spirit, after HTTP cookies were already famous.
- Three more "state in a token" examples: **JWT / signed session cookie** (login state), **pagination cursor** (`next_page_token`), **pre-signed download URL** (e.g. S3), also CSRF tokens and OAuth `state`.
- Common pattern: server signs the state with a secret → client stores it → server verifies on return and remembers nothing.

### HW3 · Unpack "Hello World!" by hand
**Task:** `C8 32 9B FD 06 5D DF 72 36 39 04` — 11 octets, 12 characters. Recover the first three by hand, then write the unpacker.

**✅ Solution — by hand** (fill each octet from its lowest bit up):
```
C8 = 1100 1000 → low 7 bits 100 1000 = 0x48 = 'H'     carry top 1 bit: 1
32 = 0011 0010 → low 6 bits 11 0010, shift left 1, add carry 1
               = 110 0101 = 0x65 = 'e'                 carry top 2 bits: 00
9B = 1001 1011 → low 5 bits 1 1011, shift left 2, add carry 00
               = 110 1100 = 0x6C = 'l'
```
(For these characters the GSM-7 code = the ASCII code.)

**✅ Solution — unpacker (Python, tested):** treat the bytes as one little-endian number and slice 7 bits at a time.
```python
def unpack(data, n):                          # n = TP-UDL (number of septets)
    bits = int.from_bytes(data, "little")     # octet 1 = lowest bits
    return "".join(chr((bits >> (7 * i)) & 0x7F) for i in range(n))

def pack(text):                               # reverse check
    bits = sum(ord(c) << (7 * i) for i, c in enumerate(text))
    return bits.to_bytes((7 * len(text) + 7) // 8, "little")

raw = bytes.fromhex("C8329BFD065DDF72363904")
print(unpack(raw, 12))                        # Hello World!
print(pack("Hello World!").hex(" ").upper())  # C8 32 9B FD 06 5D DF 72 36 39 04
```

### HW4 · Run Asterisk, place a SIP call, capture it
**Task:** two softphones and Wireshark; Telephony → VoIP Calls plays the audio from your capture. Draw conclusions about RTP.

**✅ Solution (easiest path, a Linux box/VM):**
1. `sudo apt install asterisk wireshark`
2. Add two users to `/etc/asterisk/pjsip.conf` (repeat the 3 blocks with `6002`):
   ```ini
   [transport-udp]
   type=transport
   protocol=udp
   bind=0.0.0.0:5060

   [6001]
   type=endpoint
   context=demo
   disallow=all
   allow=ulaw
   auth=6001
   aors=6001

   [6001]
   type=auth
   auth_type=userpass
   username=6001
   password=pass6001

   [6001]
   type=aor
   max_contacts=1
   ```
3. In `/etc/asterisk/extensions.conf`:
   ```ini
   [demo]
   exten => _600X,1,Dial(PJSIP/${EXTEN})
   ```
4. `sudo systemctl restart asterisk`. Register Linphone/Zoiper as `6001` and `6002` (server = Asterisk's IP, UDP 5060).
5. Start Wireshark on that interface, call `6002` from `6001`, talk, hang up.
6. Filter `sip` → you see INVITE (with SDP), 100 Trying, 180 Ringing, 200 OK, ACK, BYE, 200 OK — the §3.9 ladder. Filter `rtp` → the voice. Telephony → VoIP Calls → select call → **Play Streams**.

**Conclusions about RTP:** signalling (SIP) is readable text; media is RTP over **UDP** on the IP:port named in SDP, on a separate path from SIP (control vs data again); plain RTP is **unencrypted** — that's why Wireshark can play it (SRTP fixes this); lost RTP packets are not resent.

---

## 7 · What to keep

| Idea | The slide's line |
|---|---|
| **Encapsulation** | One layer's headers are another layer's body. Ethernet ⊃ IP ⊃ TCP ⊃ HTTP is the same picture as MTP2 ⊃ MTP3 ⊃ SCCP ⊃ TCAP ⊃ MAP ⊃ TPDU. Two committees, fifty years apart, one idea. |
| **Framing** | Length or delimiter. SMS chose length, SMTP chose a dot, IMAP chose both. There is no third option. |
| **Reliability has an address** | SS7 put it at L2, per hop. TCP put it at L4, end to end. QUIC put it in userspace. All three were right for their network. |
| **Control and data want separating** | Blue boxes died when SS7 split them. SIP keeps media off the proxy. FTP opens a second socket. Same instinct, three eras. |

---

## 8 · Summary cheat-sheet

| Term | One-line meaning |
|---|---|
| Server calls | socket → bind → listen → accept → read → write → close. |
| `listen(fd, 1)` | `1` = accept-queue depth, not the connection limit. |
| `htons` | Host to network short; the wire is big-endian. |
| Client | No bind/listen; kernel picks ephemeral port; `connect()` triggers the handshake. |
| FIN vs RST | `close()` → FIN, TIME_WAIT 2 MSL. `SO_LINGER {1,0}` → RST, unsent data discarded, no TIME_WAIT. |
| CSMA/CA | Wi-Fi: radio can't listen while sending → sense, backoff, transmit, per-frame L2 ACK; ≈2 ms contention. |
| Starlink | Just L1 (~550 km, ~20 ms); IP header unchanged except TTL, checksum, NAT. |
| MSS / frame | 1500 − 20 − 20 = 1460 B; + 14 + 4 = 1518 B. |
| SS7 (1975) | Out-of-band signalling; ended "make the sound, make the decision" (2600 Hz). |
| SS7 stack | MTP1/2/3 (L1–L3) · SCCP (L4) · TCAP (L5–6) · MAP/INAP/CAP/ISUP (L7). |
| SMS limits | SIF 272 > sm-RP-UI ~200 > TP-UD 140; one SMS = one MSU. |
| GSM-7 | 140 × 8 ÷ 7 = 160 chars; UCS-2 → 70. |
| SMS path | SMS-SUBMIT → mo-ForwardSM → SMSC stores → HLR lookup → mt-ForwardSM. |
| ISUP | IAM, ACM (ringback), ANM, REL, RLC; SS7 idle during the talk. |
| SIP | Text, HTTP-style codes 100/180/200; media = RTP over UDP, direct. |
| SS7 vs TCP | Per-hop at L2 vs end-to-end at L4; SS7 trusts by point code. |
| BBR (2016) | Bottleneck bandwidth × RTT, queue near empty; Reno/CUBIC need loss (sawtooth). |
| Handshake | SYN, SYN-ACK, ACK; SYN & FIN consume a seq number; random ISN (RFC 6528) vs Mitnick. |
| SYN cookies | State in the ISN; loses window scale/SACK/timestamps → only under pressure. |
| Close | 4 segments; half-close; CLOSE_WAIT = forgot `close()`; TIME_WAIT → SO_REUSEADDR. |
| RTT tax | TCP 1 · TLS 1.2 3 · TLS 1.3 2 · TFO 1 · QUIC 1. |
| UDP | 8 B header (ports, length, checksum), ≤ 65,507 B payload. |
| QUIC | Over UDP in userspace; per-stream sequence space kills head-of-line blocking. |
| SMTP ports | 25 relay · 587 submission (STARTTLS) · 465 implicit TLS. |
| Envelope vs letter | MAIL FROM/RCPT TO route; From:/To: just displayed; nothing checks. |
| Base64 | 3 bytes → 4 chars, 33% overhead, 76 chars/line. |
| POP3 | STAT, LIST, RETR, DELE (at QUIT); 110, POP3S 995. |
| IMAP | State on server: flags, folders, SEARCH, IDLE, partial FETCH; 143. |
| Framing | Delimiter (dot-stuffing) · length (chunked avoids it) · both (request smuggling). |
| FTP | Active: server dials back from 20 (NAT blocks). Passive: client opens both; 227 port = p1 × 256 + p2. |

---

## 9 · MCQs (mid → advanced)

**Q1.** A server calls `listen(server_fd, 1)`. Which statement is correct?
- A) The server can only ever serve one client in its lifetime
- B) At most one connection can wait in the accept queue; this small backlog is what a SYN flood targets
- C) The `1` sets the number of worker threads
- D) The `1` is the TCP window size in segments

> ✅ **Answer: B.** The argument is the accept-queue depth, not a connection limit — a finite backlog is exactly the target of a SYN flood. A is the common misreading.

**Q2.** A client writes 4 KB, sets `SO_LINGER {1, 0}` and immediately calls `close()` while most of the data is still unsent. What happens?
- A) A FIN is sent after all 4 KB are delivered, then TIME_WAIT
- B) The kernel waits 1 second before closing
- C) The data is delivered, but TIME_WAIT is skipped
- D) An RST is sent, the unsent data is discarded, and there is no TIME_WAIT

> ✅ **Answer: D.** Linger on with a zero timeout means abortive close: RST, unsent data thrown away, no TIME_WAIT. A describes a normal `close()`.

**Q3.** You kill and instantly restart your server and get "address already in use". The most likely cause and fix?
- A) Sockets in TIME_WAIT from the previous run; use `SO_REUSEADDR`
- B) Sockets stuck in CLOSE_WAIT; reboot the switch
- C) The backlog of `listen()` is full; raise it
- D) The client is using the same ephemeral port

> ✅ **Answer: A.** The slides say TIME_WAIT is why you get "address already in use" on restart, and why `SO_REUSEADDR` exists.

**Q4.** A production server shows thousands of sockets in CLOSE_WAIT. Where should you look first?
- A) The network switch configuration
- B) The Wi-Fi contention window
- C) The application code — it is not calling `close()` after the peer closed
- D) The TCP congestion control algorithm

> ✅ **Answer: C.** CLOSE_WAIT means the peer sent FIN but your app never closed its side — "look at your code, not your switch."

**Q5.** Why does Wi-Fi (802.11) use CSMA/CA instead of Ethernet's CSMA/CD?
- A) A radio cannot listen while it transmits, so it cannot detect collisions and must avoid them
- B) Wi-Fi frames are too small to collide
- C) CSMA/CD needs IP addresses, which Wi-Fi doesn't have at L2
- D) Wi-Fi relies on TCP to detect collisions

> ✅ **Answer: A.** Its own signal drowns everything out, so Wi-Fi senses, backs off, transmits and demands an ACK for every frame. D is wrong — the L2 ACKs happen long before TCP knows anything.

**Q6.** A Wi-Fi frame carrying a TCP segment is lost in a collision. Which layer notices and retransmits first?
- A) TCP at L4, via its RTO timer
- B) IP at L3, via the TTL
- C) The 802.11 link layer at L2, because no per-frame ACK arrived
- D) The application, via a timeout

> ✅ **Answer: C.** Every Wi-Fi frame is individually acknowledged at L2; a missing ACK triggers backoff and retransmit long before TCP has any idea.

**Q7.** Your packet goes laptop → Starlink terminal → LEO satellite (550 km) → ground gateway → internet. Which IP header fields may change on the way?
- A) Source IP only
- B) Only TTL, checksum, and any NAT rewrite
- C) Every field, because the physical medium changed
- D) None at all

> ✅ **Answer: B.** That's the whole argument for layering: the satellite is just L1, and the header arrives unchanged except TTL, checksum and NAT. D forgets TTL/checksum.

**Q8.** Ethernet MTU is 1500 B, with 20 B IPv4 and 20 B TCP headers. What are the MSS and the full wire frame size (header + FCS)?
- A) 1500 B and 1518 B
- B) 1480 B and 1500 B
- C) 1460 B and 1500 B
- D) 1460 B and 1518 B

> ✅ **Answer: D.** 1500 − 20 − 20 = 1460 B MSS; 1500 + 14 B Ethernet + 4 B FCS = 1518 B.

**Q9.** Before 1975, a "blue box" playing 2600 Hz could make a phone trunk look idle. What design flaw allowed this, and what fixed it?
- A) In-band signalling — control and voice shared one channel; SS7 moved signalling out-of-band
- B) Weak encryption of voice; fixed by TLS
- C) No CRC on voice frames; fixed by MTP2's CRC-16
- D) Too few trunks; fixed by E1/T1 multiplexing

> ✅ **Answer: A.** "Anyone who can make the sound can make the decision." SS7 put signalling on a separate packet network the caller never touches.

**Q10.** In the SS7 stack, which layer does point-code routing with SLS load sharing — the rough equivalent of IP?
- A) SCCP
- B) TCAP
- C) MTP2
- D) MTP3

> ✅ **Answer: D.** MTP3 is SS7's L3 (point-code routing, SLS). SCCP (L4) does subsystem numbers and Global Title Translation; MTP2 is the reliable link layer.

**Q11.** Why is the SMS user data capped at 140 octets?
- A) Because GSM-7 needs exactly 160 characters
- B) So the whole MAP operation always fits in one MTP2 MSU (SIF ≤ 272) — no segmentation or reassembly
- C) Because TCP's MSS on early phones was 140 bytes
- D) Because SMSC databases stored fixed 140-byte rows

> ✅ **Answer: B.** Three stacked limits (272 SIF → ~200 sm-RP-UI → 140 TP-UD); 140 guarantees one SMS = one signalling packet. A reverses cause and effect.

**Q12.** A 100-character message in plain English fits in one SMS. You add a single emoji. What happens?
- A) Nothing — emoji take one septet
- B) The encoding switches to UCS-2 (70 chars per SMS), so the message now needs a second SMS
- C) The SMSC drops the emoji
- D) The limit rises to 280 octets

> ✅ **Answer: B.** UCS-2 gives 140 ÷ 2 = 70 characters in the same 140-octet field, so 101 characters no longer fit — "one emoji can silently cost you a second SMS."

**Q13.** Using GSM-7 packing, how many octets does a 20-character message occupy in TP-UD?
- A) 18
- B) 20
- C) 17
- D) 23

> ✅ **Answer: A.** 20 × 7 = 140 bits → round up to 18 octets (144 bits, 4 bits of padding), the same rule as 12 chars → 84 bits → 11 octets. 20 would be 8-bit ASCII.

**Q14.** Phone A sends an SMS while Phone B is switched off. What happens, per the SMS flow?
- A) Phone A keeps retrying until B is on
- B) The MSC rejects the SMS immediately
- C) The SMSC accepts and queues it (TCAP END); delivery is now the SMSC's problem
- D) The HLR stores the text

> ✅ **Answer: C.** Sending a text is "a database lookup wrapped in a queue": once the SMSC replies "accepted, queued", A's job is done. The HLR only gives back IMSI + serving MSC.

**Q15.** In an ISUP call, at which message does the caller start hearing ringback?
- A) IAM
- B) ANM
- C) REL
- D) ACM

> ✅ **Answer: D.** IAM carries the called number; ACM (Address Complete) gives the caller ringback; ANM cuts the voice path through when B answers.

**Q16.** During a 10-minute ISUP phone call, how busy is the SS7 signalling network for that call?
- A) It carries every voice sample
- B) It is completely idle — voice goes on the 64 kbps bearer circuit
- C) It sends an ACK every second
- D) It re-sends IAM every minute

> ✅ **Answer: B.** "Four messages set up a call. During the conversation itself the signalling network is completely idle."

**Q17.** In a 5G IMS/SIP call between Alice and Bob through a proxy/SBC, how does the voice (RTP) travel?
- A) Directly between the endpoints over UDP, while SIP signalling goes via the proxy
- B) Through the proxy over TCP, inside SIP messages
- C) Over SS7 MTP links
- D) Inside the SDP body of the INVITE

> ✅ **Answer: A.** Signalling is text via the proxy; media is RTP/SRTP over UDP directly between endpoints — control and data on separate paths. SDP only *describes* the media.

**Q18.** Which comparison of SS7 and TCP reliability is correct?
- A) Both are end-to-end
- B) SS7 is end-to-end at L4; TCP is per-hop at L2
- C) Neither retransmits
- D) SS7 is per-hop at L2 (MTP2); TCP is end-to-end at L4

> ✅ **Answer: D.** "SS7 makes every hop reliable. TCP makes only the endpoints reliable." B swaps them.

**Q19.** SS7's FSN is 7 bits per signal unit; TCP's sequence number is 32 bits. What does each count?
- A) Both count bytes
- B) Both count packets
- C) FSN counts signal units (wraps at 128); TCP counts bytes
- D) FSN counts bytes; TCP counts segments

> ✅ **Answer: C.** FSN is per signal unit and wraps at 128; TCP's 32-bit sequence number is per byte.

**Q20.** A long-distance link randomly drops 1% of packets due to noise, not congestion. Why does CUBIC crawl while BBR does well?
- A) CUBIC encrypts every packet
- B) CUBIC halves on every loss, treating noise as congestion; BBR paces at bottleneck bandwidth × RTT
- C) BBR retransmits at L2
- D) BBR ignores RTT

> ✅ **Answer: B.** Loss-based algorithms need loss to make progress and halve on every drop; BBR measures bottleneck bandwidth × round-trip time and keeps the queue near empty.

**Q21.** A client sends `SYN seq=7000`. The server replies `SYN ACK seq=3000`. What ack number must the server's SYN-ACK carry, and what does the client's final ACK carry?
- A) ack=7000, then ack=3000
- B) ack=7001, then ack=3000
- C) ack=7001, then ack=3001
- D) ack=7000, then ack=3001

> ✅ **Answer: C.** SYN consumes one sequence number, so each SYN is acknowledged with ISN + 1 — same as `ack=1001` for `seq=1000` on the slide.

**Q22.** A server enables SYN cookies permanently. What is the downside?
- A) Window scale, SACK and timestamps are mostly lost, since only 32 bits of ISN are available
- B) The server must store twice as many half-open entries
- C) The handshake becomes 4 messages
- D) Clients must send data in the SYN

> ✅ **Answer: A.** State lives only in the 32-bit ISN, so most options don't survive — which is why cookies switch on only under pressure. B is the opposite: cookies store nothing.

**Q23.** Mumbai to Virginia is ~190 ms RTT. Roughly how long before the first HTTP byte can move over TCP + TLS 1.2, versus QUIC?
- A) 190 ms vs 190 ms
- B) 380 ms vs 0 ms
- C) 760 ms vs 380 ms
- D) 570 ms vs 190 ms

> ✅ **Answer: D.** TCP + TLS 1.2 costs 3 RTTs (3 × 190 = 570 ms); QUIC folds transport and crypto into 1 RTT (190 ms). Bandwidth can't fix this.

**Q24.** A page loads 3 resources over HTTP/2 on one TCP connection. One packet of the CSS stream is lost. What happens, and how does HTTP/3 over QUIC differ?
- A) Only CSS waits in both cases
- B) All streams stall over TCP (one ordered byte stream); with QUIC only the CSS stream waits
- C) TCP drops the whole connection; QUIC drops the packet silently
- D) Both stall all streams

> ✅ **Answer: B.** TCP delivers one ordered byte stream, so a lost packet blocks everyone (head-of-line blocking). QUIC gives each stream its own sequence space.

**Q25.** You connect to an FTP server, send `PASV`, and get `227 Entering Passive Mode (10,0,0,5,195,80)`. Which data connection do you open?
- A) The client opens a connection to 10.0.0.5 port 50000
- B) The server connects back to the client on port 20
- C) The client connects to 10.0.0.5 port 19580
- D) The client connects to 195.80.0.0 port 21

> ✅ **Answer: A.** Port = 195 × 256 + 80 = 50000 (big-endian pair, htons in ASCII), and in passive mode the client opens both connections. B is active mode, which NAT blocks; C just glues the digits.
