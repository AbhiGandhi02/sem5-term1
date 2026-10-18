# Networking for DevOps

> **DevOps – Lecture 3 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 1 – Linux](01-Linux-Permissions-and-Commands.md) (`ssh`, `scp`) and [Lecture 2 – Shell Scripting](02-Shell-Scripting.md) (you can script every command here).
>
> **Practice:** [Foundations MCQs](Foundations-MCQs.md).

---

## Table of Contents
1. [The Big Picture: How Data Travels](#1-the-big-picture-how-data-travels)
2. [IP Addresses](#2-ip-addresses)
3. [Ports](#3-ports)
4. [Subnetting & CIDR](#4-subnetting--cidr)
5. [DHCP: Automatic IP Assignment](#5-dhcp-automatic-ip-assignment)
6. [NAT: Sharing One Public IP](#6-nat-sharing-one-public-ip)
7. [DNS: The Internet's Phonebook](#7-dns-the-internets-phonebook)
8. [The OSI Model](#8-the-osi-model)
9. [TCP vs. UDP](#9-tcp-vs-udp)
10. [HTTP, HTTPS & TLS](#10-http-https--tls)
11. [Networking Commands & Tools](#11-networking-commands--tools)
12. [Network Security Basics](#12-network-security-basics)
13. [Putting It All Together: What Happens When You Open a Website](#13-putting-it-all-together-what-happens-when-you-open-a-website)
14. [Hands-on Practice Lab](#14-hands-on-practice-lab)
15. [Quick Revision Cheat Sheet](#15-quick-revision-cheat-sheet)
16. [Likely Exam / Interview Questions](#16-likely-exam--interview-questions)

---

## 1. The Big Picture: How Data Travels

When your laptop talks to a server, several systems work together. **Analogy: sending a parcel.**

| Real world | Networking | Section |
|------------|-----------|---------|
| Street address of a building | **IP address** | 2 |
| Flat/apartment number in the building | **Port** | 3 |
| Neighbourhood / postal zone | **Subnet** | 4 |
| Housing society assigning flat numbers to new tenants | **DHCP** | 5 |
| Society reception: all residents' outgoing mail uses the society's address | **NAT** | 6 |
| Phonebook: name → address | **DNS** | 7 |
| Courier rules: tracked & signed vs. drop-and-go | **TCP vs. UDP** | 9 |
| Sealed, tamper-proof envelope | **HTTPS / TLS** | 10 |
| Security guard at the gate | **Firewall** | 12 |

**Why DevOps cares:** you'll design **VPCs and subnets** on AWS, open **ports** in security groups, point **domains** at servers with DNS, set up **HTTPS** certificates, and debug "**the app can't connect**" problems. All of that is networking.

---

## 2. IP Addresses

An **IP (Internet Protocol) address** is a unique number that **identifies a device** on a network and tells other devices **where to send data**.

### IPv4

- **32 bits**, written as **4 numbers (octets) separated by dots**: `192.168.1.10`
- Each octet is 8 bits, so each number ranges from **0 to 255**.
- Total addresses: 2³² ≈ **4.3 billion**. That isn't enough for every device on Earth, which is why **NAT** and **IPv6** exist.

```
   192   .   168   .    1    .   10
11000000 . 10101000 . 00000001 . 00001010
└─ 8 bits ┘
└──────────────── 32 bits ────────────────┘
```

### IPv6

- **128 bits**, written as **8 groups of 4 hex digits** separated by colons:
  `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- Shortening rules: drop leading zeros in each group, and replace **one** run of all-zero groups with `::`
  → `2001:db8:85a3::8a2e:370:7334`
- 2¹²⁸ ≈ 3.4 × 10³⁸ addresses, effectively unlimited.

| | IPv4 | IPv6 |
|-|------|------|
| Size | 32-bit | 128-bit |
| Format | Decimal, dotted: `10.0.0.1` | Hex, colons: `fe80::1` |
| Count | ~4.3 billion | ~3.4 × 10³⁸ |
| Needs NAT? | Usually | Not needed |

### ➕ Extra: public vs. private IPs

**Private** IPs are used **inside** a home, office, or cloud VPC. They are **not routable** on the public internet. Anyone can reuse them in their own network.

| Private range | CIDR | Typical use |
|---------------|------|-------------|
| `10.0.0.0` – `10.255.255.255` | `10.0.0.0/8` | AWS VPCs, large companies |
| `172.16.0.0` – `172.31.255.255` | `172.16.0.0/12` | Docker default networks, AWS default VPC |
| `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | Home Wi-Fi routers |

**Public** IPs are globally unique and reachable from the internet (e.g., your EC2 instance's public IP, or `8.8.8.8`).

**Special addresses:**

| Address | Meaning |
|---------|---------|
| `127.0.0.1` (`localhost`) | **Loopback**: "this machine itself" |
| `0.0.0.0` | On a server: "listen on **all** interfaces". In a firewall rule: `0.0.0.0/0` = **anywhere** |
| `169.254.x.x` | Link-local. You get this when **DHCP failed** to give you an address |
| `255.255.255.255` | Broadcast to everyone on the local network |

---

## 3. Ports

➕ An IP address gets data to the right **machine**. A **port** gets it to the right **program** on that machine.

- A port is a number from **0 to 65535**.
- Connection endpoint = **IP:port**, e.g., `13.234.10.5:443`

| Range | Name | Notes |
|-------|------|-------|
| 0 – 1023 | Well-known | Need root to listen on them in Linux |
| 1024 – 49151 | Registered | Apps such as databases |
| 49152 – 65535 | Ephemeral | Temporary ports your OS picks for outgoing connections |

### Ports to memorize

| Port | Protocol | Use |
|------|----------|-----|
| **20/21** | FTP | File transfer (insecure) |
| **22** | **SSH** / SCP | Secure remote login (Lecture 1) |
| **23** | **Telnet** | Remote login, **unencrypted** |
| **25** | SMTP | Sending email |
| **53** | **DNS** | Name resolution (UDP, and TCP for large responses) |
| **67/68** | **DHCP** | Server / client |
| **80** | **HTTP** | Web |
| **443** | **HTTPS** | Secure web |
| 3306 | MySQL | Database |
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache |
| 27017 | MongoDB | Database |
| 3000 / 5000 / 8000 / 8080 | – | Common dev-server ports (Node, Flask, Django, Tomcat) |

---

## 4. Subnetting & CIDR

**This is the most calculation-heavy topic. Practise the examples until they're automatic.**

### 4.1 The idea

Every IPv4 address has two parts:

```
         Network part          |  Host part
   (which network/street?)     |  (which house on that street?)
```

**Subnetting** = dividing one big network into **smaller sub-networks**. Reasons:
- **Organisation:** web servers in one subnet, databases in another
- **Security:** a **public** subnet (internet-facing) and a **private** subnet (databases, hidden)
- **Efficiency:** smaller broadcast domains mean less noise
- **Address planning:** don't waste IPs

The **subnet mask** tells you where the network part ends and the host part begins.

### 4.2 Subnet mask & CIDR notation

A subnet mask is 32 bits: **1s for the network part, 0s for the host part.**

```
IP:    192.168.1.10   = 11000000.10101000.00000001.00001010
Mask:  255.255.255.0  = 11111111.11111111.11111111.00000000
                        └──────── 24 ones ───────┘└ 8 zeros┘
                             network part           host part
```

**CIDR (Classless Inter-Domain Routing) notation** writes the mask as just **the number of 1-bits**:

> `192.168.1.10/24` = IP `192.168.1.10` with mask `255.255.255.0`

**Why "classless"?** In the old **classful** system, networks came in only three fixed sizes:

| Class | First octet | Default mask | Hosts |
|-------|-------------|--------------|-------|
| A | 1 – 126 | /8 | ~16.7 million |
| B | 128 – 191 | /16 | ~65,000 |
| C | 192 – 223 | /24 | 254 |

This wasted huge numbers of addresses: a company needing 500 hosts had to take a whole Class B (65,000). **CIDR** allows **any prefix length** (`/22`, `/27`, …), so networks can be sized exactly. This is the "flexible hierarchical allocation" from class.

### 4.3 The key formulas

With prefix **/n**, the number of **host bits** is **h = 32 − n**.

| Quantity | Formula |
|----------|---------|
| Total addresses in the subnet | **2ʰ** |
| **Usable** host addresses | **2ʰ − 2** |
| Block size (how much the network address jumps per subnet) | 2ʰ, applied in the "interesting" octet |

**Why −2?** Two addresses in every subnet are reserved:
- **Network address**: first address, all host bits = 0. It identifies the subnet itself.
- **Broadcast address**: last address, all host bits = 1. It means "send to everyone in this subnet".

> ➕ **AWS note:** AWS reserves **5** IPs per subnet (network, router `.1`, DNS `.2`, future use `.3`, broadcast). So a `/24` gives **251** usable IPs on AWS, not 254.

### 4.4 CIDR reference table

| CIDR | Subnet mask | Total IPs | Usable hosts |
|------|-------------|-----------|--------------|
| /8  | 255.0.0.0       | 16,777,216 | 16,777,214 |
| /16 | 255.255.0.0     | 65,536 | 65,534 |
| /20 | 255.255.240.0   | 4,096 | 4,094 |
| /22 | 255.255.252.0   | 1,024 | 1,022 |
| /23 | 255.255.254.0   | 512 | 510 |
| **/24** | **255.255.255.0** | **256** | **254** |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /29 | 255.255.255.248 | 8 | 6 |
| /30 | 255.255.255.252 | 4 | 2 |
| /32 | 255.255.255.255 | 1 | 1 (a single specific host) |

**Mask octet values to memorize** (the number of 1-bits in one octet → decimal):

```
bits:   1    2    3    4    5    6    7    8
value: 128  192  224  240  248  252  254  255
```
Each one = previous + the next power of two (128 + 64 = 192, + 32 = 224, …).

### 4.5 Step-by-step method (no binary needed)

Given an IP with a CIDR prefix, find the network address, broadcast address, host range, and **highest host**:

1. **Find the "interesting" octet**: the one where the mask isn't 255 or 0.
   (/1–8 → 1st, /9–16 → 2nd, /17–24 → 3rd, /25–32 → 4th)
2. **Block size** = 256 − (mask value in that octet).
3. **Network address** = the largest multiple of the block size that is ≤ the IP's value in that octet. Octets after it become 0.
4. **Broadcast** = next network address − 1. Octets after it become 255.
5. **First host** = network + 1. **Last (highest) host** = broadcast − 1.

### 4.6 Worked examples

#### Example 1: `192.168.10.75/26`
1. /26 → 4th octet is interesting. Mask = `255.255.255.192`
2. Block size = 256 − 192 = **64** → subnets start at .0, .64, .128, .192
3. 75 is in the block starting at **64** → network = `192.168.10.64`
4. Next block = 128 → broadcast = `192.168.10.127`
5. Hosts: `192.168.10.65` – **`192.168.10.126`**

| Network | First host | **Highest host** | Broadcast | Usable |
|---------|-----------|-----------------|-----------|--------|
| 192.168.10.64 | .65 | **.126** | .127 | 2⁶ − 2 = 62 |

#### Example 2: `172.16.45.200/20`
1. /20 → 3rd octet (16 + 4 bits). Mask = `255.255.240.0`
2. Block size = 256 − 240 = **16** → 3rd octet blocks: 0, 16, 32, **48**, …
3. 45 falls in the block starting at **32** → network = `172.16.32.0`
4. Next block = 48 → broadcast = `172.16.47.255`
5. Hosts: `172.16.32.1` – **`172.16.47.254`**; usable = 2¹² − 2 = **4,094**

#### Example 3: `10.0.0.130/25`
- Mask `255.255.255.128`, block 128 → blocks .0 and .128
- Network `10.0.0.128`, broadcast `10.0.0.255`
- Highest host **`10.0.0.254`**, usable 126

#### Example 4: splitting a network (design question)
> *"You have `192.168.1.0/24`. Split it into 4 equal subnets."*

- 4 subnets = 2² → borrow **2 bits** → /24 + 2 = **/26**
- Block size 64:

| Subnet | Network | Host range | Broadcast |
|--------|---------|-----------|-----------|
| 1 | 192.168.1.0/26 | .1 – .62 | .63 |
| 2 | 192.168.1.64/26 | .65 – .126 | .127 |
| 3 | 192.168.1.128/26 | .129 – .190 | .191 |
| 4 | 192.168.1.192/26 | .193 – .254 | .255 |

#### Example 5: sizing for hosts
> *"Smallest subnet that fits 50 hosts?"*

You need 2ʰ − 2 ≥ 50. h = 5 gives 30 (too small); **h = 6 gives 62** ✅ → **/26** (32 − 6).

#### ➕ Example 6: a typical AWS VPC layout
```
VPC: 10.0.0.0/16  (65,536 IPs)
├── Public subnet  A: 10.0.1.0/24   → load balancer, bastion host
├── Public subnet  B: 10.0.2.0/24
├── Private subnet A: 10.0.11.0/24  → app servers
└── Private subnet B: 10.0.12.0/24  → databases
```

### 4.7 Quick check: are two IPs in the same subnet?
Compute the network address of both with the same mask. If the results match, they are in the same subnet and can talk directly. Otherwise traffic must go through a **router (gateway)**.
> `192.168.10.60/26` → network .0 · `192.168.10.70/26` → network .64 → **different subnets**.

---

## 5. DHCP: Automatic IP Assignment

**DHCP (Dynamic Host Configuration Protocol)** automatically gives a device everything it needs to join a network:
- an **IP address**
- the **subnet mask**
- the **default gateway** (the router's IP)
- **DNS server** addresses
- a **lease time** (how long it can keep the IP)

Without DHCP, you'd have to type all of these into every phone, laptop, and server by hand.

### ➕ How it works: **DORA**

```
 Client (new laptop)                         DHCP Server (router)
        │                                            │
        │── 1. DISCOVER (broadcast) ────────────────>│  "Is there a DHCP server? I need an IP."
        │<─ 2. OFFER ────────────────────────────────│  "You can have 192.168.1.23"
        │── 3. REQUEST ─────────────────────────────>│  "I'll take 192.168.1.23, please"
        │<─ 4. ACKNOWLEDGE ──────────────────────────│  "Confirmed. It's yours for 24 hours"
```
- It uses **UDP**: port **67** (server) and **68** (client).
- The device renews the lease before it expires. If it isn't renewed, the IP goes back to the pool.
- **Static IP vs. DHCP:** servers often get **fixed** IPs (or DHCP *reservations*) so their address never changes. Laptops and phones use dynamic IPs.

> On AWS, the VPC's built-in DHCP gives each EC2 instance its **private IP** automatically.

---

## 6. NAT: Sharing One Public IP

**NAT (Network Address Translation)** lets **many devices with private IPs share one public IP** when talking to the internet. This **saves IPv4 addresses**, which is the main reason it exists.

```
   Home network (private)                Router (NAT)              Internet
 ┌───────────────────────┐          ┌──────────────────┐
 │ Laptop 192.168.1.10   │──┐       │ private  ↔ public│
 │ Phone  192.168.1.11   │──┼──────>│ 192.168.1.1      │──────>  google.com
 │ TV     192.168.1.12   │──┘       │ public: 49.36.x.x│        (sees only 49.36.x.x)
 └───────────────────────┘          └──────────────────┘
```

**How the router keeps track:** it rewrites the **source IP and port** of each outgoing packet and stores the mapping in a **NAT table**. Replies come back to the right device using that table.

| Inside (private) | Outside (public) |
|------------------|------------------|
| 192.168.1.10:51000 | 49.36.x.x:**40001** |
| 192.168.1.11:51000 | 49.36.x.x:**40002** |

(Using ports like this is called **PAT** or **NAT overload**. It's the form your home router uses.)

**Side effect:** devices behind NAT **can't be reached from outside** unless you set up **port forwarding**. That adds some security.

> ➕ **AWS:** instances in a **private subnet** use a **NAT Gateway** to reach the internet for updates (`apt update`), while staying unreachable from the internet.

---

## 7. DNS: The Internet's Phonebook

**DNS (Domain Name System)** translates **human-friendly names** (`google.com`) into **IP addresses** (`142.250.183.14`). Computers connect using IPs; people remember names.

### ➕ How a DNS lookup works

```
You type www.example.com
        │
        ▼
1. Browser cache → 2. OS cache & /etc/hosts → found? done ✅
        │ (not found)
        ▼
3. Recursive resolver (your ISP, or 8.8.8.8 / 1.1.1.1)
        │
        ├──> 4. Root server (.)            "Ask the .com servers"
        ├──> 5. TLD server (.com)          "Ask example.com's nameservers"
        └──> 6. Authoritative server       "www.example.com = 93.184.215.14"
        │
        ▼
7. Resolver caches the answer (for its TTL) and returns it to you
```

- **TTL (Time To Live):** how long (in seconds) an answer may be cached. This is why DNS changes can take time to spread ("propagation").
- **`/etc/hosts`** lets you override DNS locally: `127.0.0.1  myapp.local`
- DNS mostly uses **UDP port 53**.

### ➕ DNS record types

| Record | Maps | Example |
|--------|------|---------|
| **A** | Name → **IPv4** | `example.com → 93.184.215.14` |
| **AAAA** | Name → **IPv6** | `example.com → 2606:2800::1` |
| **CNAME** | Name → **another name** (alias) | `www.example.com → example.com` |
| **MX** | Mail server for the domain | `example.com → mail.example.com` |
| **NS** | Which nameservers are authoritative | `example.com → ns1.provider.com` |
| **TXT** | Arbitrary text (domain verification, SPF) | `"v=spf1 include:_spf.google.com ~all"` |
| **PTR** | **Reverse**: IP → name | `14.215.184.93.in-addr.arpa → example.com` |

> In DevOps, you'll create **A/CNAME records** (e.g., in AWS **Route 53**) to point `api.myapp.com` at a load balancer or server.

---

## 8. The OSI Model

The **OSI (Open Systems Interconnection) model** splits network communication into **7 layers**. Each layer has one job and relies on the layer below it. It's a **conceptual model** for understanding and **troubleshooting** networks.

### Mnemonics
- **Top → bottom (7 → 1):** **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing
- **Bottom → top (1 → 7):** **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

| # | Layer | Job (plain English) | Data unit | Protocols / examples | Devices |
|---|-------|--------------------|-----------|---------------------|---------|
| 7 | **Application** | What the user's app speaks | Data | **HTTP, HTTPS, DNS, SSH, FTP, SMTP** | – |
| 6 | **Presentation** | Format, encrypt, compress | Data | **TLS/SSL**, JPEG, JSON, UTF-8 | – |
| 5 | **Session** | Open, manage, and close conversations | Data | Sessions, RPC, sockets | – |
| 4 | **Transport** | End-to-end delivery between **programs** (ports), reliability | **Segment** (TCP) / Datagram (UDP) | **TCP, UDP** | – |
| 3 | **Network** | Routing between networks (**IP addresses**) | **Packet** | **IP, ICMP** (ping), routing | **Router** |
| 2 | **Data Link** | Delivery within the local network (**MAC addresses**) | **Frame** | Ethernet, Wi-Fi, ARP | **Switch** |
| 1 | **Physical** | Bits as signals | **Bits** | Cables, fibre, radio | Hub, cable |

### Encapsulation: how the layers wrap data
When you send data, each layer **adds its own header** on the way down. The receiver **removes** them on the way up.

```
L7  [ HTTP data ]
L4  [TCP hdr | HTTP data ]                          ← Segment (ports)
L3  [IP hdr | TCP hdr | HTTP data ]                 ← Packet  (IPs)
L2  [Eth hdr | IP hdr | TCP hdr | HTTP data | FCS]  ← Frame   (MACs)
L1  0101101001010...                                 ← Bits
```

### ➕ OSI vs. TCP/IP model
The real internet runs on the simpler **4-layer TCP/IP model**:

| TCP/IP layer | OSI layers |
|-------------|-----------|
| Application | 7 + 6 + 5 |
| Transport | 4 |
| Internet | 3 |
| Network Access / Link | 2 + 1 |

### Troubleshooting with layers (a DevOps habit)
Work **bottom-up**:
1. **L1/L2:** Is the interface up? (`ip link`)
2. **L3:** Do I have an IP? Can I reach the host? (`ip addr`, `ping`, `traceroute`)
3. **L4:** Is the port open? (`telnet host port`, `nc -zv host port`)
4. **L7:** Does the app respond correctly? (`curl -v`)

---

## 9. TCP vs. UDP

Both are **Transport layer (L4)** protocols that carry data between programs using **ports**.

| Feature | **TCP** (Transmission Control Protocol) | **UDP** (User Datagram Protocol) |
|---------|----------------------------------------|----------------------------------|
| Connection | **Connection-oriented** (handshake first) | **Connectionless** (just send) |
| Reliability | **Guaranteed delivery**: lost packets are resent | **No guarantee**: lost packets are gone |
| Order | **In order** | May arrive out of order |
| Error checking | Checksum + acknowledgements + retransmission | Checksum only |
| Speed / overhead | Slower, more overhead | **Faster**, lightweight |
| Header size | 20+ bytes | 8 bytes |
| Analogy | Registered post with tracking and signature | Throwing flyers over a wall |
| Used by | **HTTP/HTTPS, SSH, FTP, SMTP, databases** | **DNS, DHCP, video streaming, VoIP, online gaming** |

**Rule of thumb:** use TCP when **every byte matters** (web pages, files, payments). Use UDP when **speed matters more than perfection** (a dropped video frame is better than a frozen call).

### ➕ TCP three-way handshake

```
 Client                         Server
   │──── SYN ──────────────────>│   "Can we talk?"
   │<─── SYN-ACK ───────────────│   "Yes, can you hear me?"
   │──── ACK ──────────────────>│   "Yes. Starting now."
   │══════ data flows ══════════│
```
Closing uses **FIN / ACK** messages. If a port has nothing listening on it, the server replies with **RST**, which you see as "**Connection refused**".

---

## 10. HTTP, HTTPS & TLS

### HTTP (HyperText Transfer Protocol)
The **application-layer** protocol of the web. It works by **request/response**: the client asks and the server answers. It runs on **TCP port 80**.

```
GET /index.html HTTP/1.1          ← request line: METHOD  PATH  VERSION
Host: example.com                 ← headers
User-Agent: curl/8.0

HTTP/1.1 200 OK                   ← response status
Content-Type: text/html
<html>...</html>                  ← body
```

**➕ Common methods:** `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (delete)

**➕ Status codes:**

| Range | Meaning | Common ones |
|-------|---------|-------------|
| 1xx | Informational | 101 Switching Protocols |
| **2xx** | **Success** | **200 OK**, 201 Created, 204 No Content |
| **3xx** | **Redirect** | **301** Moved Permanently, **302** Found, 304 Not Modified |
| **4xx** | **Client error** (your fault) | 400 Bad Request, **401** Unauthorized, **403** Forbidden, **404** Not Found, 429 Too Many Requests |
| **5xx** | **Server error** (server's fault) | **500** Internal Error, **502** Bad Gateway, **503** Unavailable, **504** Gateway Timeout |

> **502/504** are very common in DevOps. They usually mean the **reverse proxy or load balancer** (e.g., Nginx) couldn't reach the **backend app**, because it's down, on the wrong port, or too slow.

### HTTPS = HTTP + TLS
**HTTPS** is HTTP sent **inside an encrypted TLS tunnel**. It runs on **TCP port 443**.

Plain HTTP is **plaintext**, so anyone on the path (public Wi-Fi, ISP) can **read and modify** it, passwords included. HTTPS provides three guarantees:

| Guarantee | Meaning |
|-----------|---------|
| 🔒 **Confidentiality** | Data is encrypted, so eavesdroppers see gibberish |
| ✅ **Integrity** | Data can't be changed in transit without detection |
| 🪪 **Authentication** | You're really talking to `bank.com`, not an impostor (proved by its **certificate**) |

### TLS certificates
A **TLS (formerly SSL) certificate** is a digital ID card for a website. It contains:
- the **domain name(s)** it's valid for
- the server's **public key**
- the **issuer**: a trusted **Certificate Authority (CA)** such as Let's Encrypt or DigiCert
- **validity dates** (expiry)

Your browser/OS ships with a list of trusted CAs. If the certificate was signed by one of them, matches the domain, and hasn't expired, you see the 🔒 padlock. Otherwise you get "**Your connection is not private**".

### ➕ Simplified TLS handshake
```
1. Client Hello  → "I support these TLS versions and ciphers" + random value
2. Server Hello  ← picks the cipher + sends its CERTIFICATE
3. Client verifies the certificate (trusted CA? right domain? not expired?)
4. Key exchange  → both sides derive the same secret SESSION KEY
                   (asymmetric crypto is used only for this step)
5. Encrypted HTTP traffic using fast SYMMETRIC encryption with the session key 🔒
```

> **DevOps tip:** get free, auto-renewing certificates with **Let's Encrypt + Certbot** (`sudo certbot --nginx`), or **AWS Certificate Manager (ACM)** for load balancers. **Expired certificates** are a classic production outage.

---

## 11. Networking Commands & Tools

### 11.1 `ifconfig` / `ip` / `ipconfig`: see your IP and interfaces

| OS | Command |
|----|---------|
| Linux (old) | `ifconfig` |
| Linux (modern) ➕ | **`ip addr`** (or `ip a`) |
| macOS | `ifconfig` |
| Windows | `ipconfig` (`ipconfig /all` for details) |

```bash
$ ip addr
2: eth0: <BROADCAST,MULTICAST,UP> mtu 9001
    link/ether 0a:1b:2c:3d:4e:5f            ← MAC address (L2)
    inet 172.31.5.10/20 brd 172.31.15.255   ← IPv4 / CIDR, broadcast (L3)
    inet6 fe80::81b:2cff:fe3d:4e5f/64       ← IPv6
```
- `lo` = loopback (`127.0.0.1`); `eth0` / `ens5` = Ethernet; `wlan0` / `en0` = Wi-Fi
- On Ubuntu, `ifconfig` may need `sudo apt install net-tools`. `ip` is the modern replacement.

**➕ Related:**
```bash
ip route              # routing table; "default via X" = your gateway
hostname -I           # just your IPs
curl ifconfig.me      # your PUBLIC IP (as the internet sees you, after NAT)
ipconfig /release && ipconfig /renew   # Windows: ask DHCP for a new IP
```

### 11.2 `ping`: is the host reachable?
Sends **ICMP Echo Request** packets (Layer 3) and measures the **round-trip time (RTT)**.

```bash
$ ping -c 4 google.com          # -c 4 = send 4 packets (Linux/Mac loops forever otherwise)
64 bytes from 142.250.183.14: icmp_seq=1 ttl=117 time=12.4 ms
...
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max = 11.9/12.3/12.8 ms
```

| Output | Meaning |
|--------|---------|
| `time=12.4 ms` | Round-trip latency |
| `ttl=117` | Hops remaining (each router subtracts 1) |
| `packet loss` | Percentage of packets that never returned |
| `Request timeout` / 100% loss | Host down, **or ICMP blocked by a firewall** |
| `unknown host` | **DNS** failed to resolve the name |

> ⚠️ **No ping reply ≠ server down.** AWS security groups block ICMP by default, so an EC2 instance can be perfectly healthy and still not answer ping.

### 11.3 `traceroute` / `tracert`: which path does traffic take?
Shows every **router (hop)** between you and the destination, plus the latency at each hop.

| OS | Command |
|----|---------|
| Linux / macOS | `traceroute google.com` |
| Windows | `tracert google.com` |

```
 1  192.168.1.1     1.2 ms    ← your home router (gateway)
 2  10.20.0.1       5.4 ms    ← ISP
 3  * * *                     ← router not answering (normal, often filtered)
 4  142.250.x.x    12.1 ms    ← Google
```

**How it works (➕):** it sends packets with **TTL = 1, 2, 3, …**. Each router that decrements a TTL to 0 sends back an "ICMP Time Exceeded" message, which reveals that router's address.

**Use it to find where** latency jumps or the connection breaks (your network, your ISP, or the destination).

### 11.4 `telnet`: test whether a port is open
Telnet is an old **remote-login protocol** (TCP **port 23**). It sends **everything in plaintext**, including passwords, so **never use it for logins. Use SSH (port 22) instead.**

**Its useful DevOps role today: checking whether a TCP port is reachable.**
```bash
$ telnet 13.234.10.5 80
Connected to 13.234.10.5.          ✅ port open, something is listening
Escape character is '^]'.          (Ctrl + ] then type quit to exit)

$ telnet 13.234.10.5 5432
Connection refused                 ❌ host reachable, but nothing listening on that port
...hangs, then times out...        ❌ a firewall/security group is silently dropping traffic
```

> **"Refused" vs. "timeout" is an important distinction:**
> - **Refused** → the server is reachable and the port is closed. Check whether the **app** is running and on which port.
> - **Timeout** → packets are being **dropped**. Check the **firewall / security group / NACL**.

**➕ Modern alternative:** `nc -zv host port` (netcat), e.g., `nc -zv google.com 443`

### 11.5 `nslookup` / `dig`: query DNS
```bash
$ nslookup google.com
Server:   8.8.8.8                 ← the DNS resolver used
Address:  142.250.183.14          ← answer (A record)

nslookup -type=MX gmail.com       # mail servers
nslookup google.com 1.1.1.1       # ask a specific DNS server
```

**➕ `dig`** gives more detail and is preferred on Linux:
```bash
dig google.com              # full answer, including TTL
dig +short google.com       # just the IP
dig CNAME www.github.com
dig +trace example.com      # show the full root → TLD → authoritative chain
```

### 11.6 ➕ More tools you'll use constantly

| Command | What it does | Example |
|---------|--------------|---------|
| `curl` | Make HTTP requests from the terminal | `curl -I https://google.com` (headers only), `curl -v` (verbose, shows the TLS handshake) |
| `wget` | Download files | `wget https://example.com/file.zip` |
| `ss` / `netstat` | Which ports are **listening** on this machine, and by which process | `sudo ss -tulnp` |
| `arp -a` / `ip neigh` | IP → MAC table for the local network | `ip neigh` |
| `whois` | Who owns a domain or IP | `whois google.com` |

`ss -tulnp` flags: **t**cp, **u**dp, **l**istening, **n**umeric (don't resolve names), **p**rocess.
```bash
$ sudo ss -tulnp
tcp LISTEN 0 511 0.0.0.0:80    users:(("nginx",pid=812))      ← nginx on all interfaces, port 80
tcp LISTEN 0 128 127.0.0.1:5432 users:(("postgres",pid=640)) ← postgres only on localhost!
```
> If an app listens on **`127.0.0.1`**, it's reachable **only from the same machine**. To accept outside connections it must bind to **`0.0.0.0`**. This is a very common reason for "it works on the server but not from my laptop".

### 11.7 Troubleshooting flowchart: "I can't reach my app"
```
Can you resolve the name?          nslookup app.com     ── no ──> DNS problem (records, TTL)
        │ yes
Can you reach the host?            ping / traceroute    ── no ──> routing / host down / ICMP blocked
        │ yes
Is the port reachable?             telnet host 80       ── timeout ─> firewall / security group
        │ yes                                           ── refused ─> app not running / wrong port
Is the app listening correctly?    ss -tulnp (on server) ─ 127.0.0.1 only? ─> bind to 0.0.0.0
        │ yes
Does the app respond properly?     curl -v http://host  ── 5xx ──> check app logs (/var/log/...)
```

---

## 12. Network Security Basics

### Prefer HTTPS over HTTP
Always serve production traffic over HTTPS, and **redirect HTTP (80) → HTTPS (443)**. Without it, logins, cookies, and API keys travel in **plaintext**.

### Firewalls: allow only what's needed
A **firewall** filters traffic using rules based on **IP, port, and protocol**. Best practice is the **principle of least privilege**: deny everything by default, then allow only what's required.

**Example: a typical web server**

| Port | Source | Why |
|------|--------|-----|
| 22 (SSH) | **Your IP only** (`203.0.113.7/32`) | Admin access. **Never** `0.0.0.0/0` |
| 80 (HTTP) | `0.0.0.0/0` | Public, redirects to HTTPS |
| 443 (HTTPS) | `0.0.0.0/0` | Public website |
| 5432 (Postgres) | App server subnet only | **Never** expose a database publicly |

**➕ On Linux, using `ufw`:**
```bash
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 80,443/tcp
sudo ufw enable
sudo ufw status
```

**➕ On AWS:**

| | **Security Group** | **Network ACL (NACL)** |
|-|--------------------|------------------------|
| Applies to | An instance (its network interface) | A whole subnet |
| Rules | **Allow only** | Allow **and** deny |
| State | **Stateful**: return traffic is allowed automatically | **Stateless**: you must allow return traffic explicitly |

### VPNs: secure remote access
A **VPN (Virtual Private Network)** creates an **encrypted tunnel** over the public internet, so a remote user (or office) behaves as if it were **inside** the private network.
- Employees reach internal tools without exposing those tools to the internet.
- Traffic on untrusted Wi-Fi is protected.
- ➕ Examples: OpenVPN, WireGuard, AWS Client VPN / Site-to-Site VPN.
- ➕ A **bastion host** (jump box) is a related pattern: one hardened public server you SSH into first, then hop to private servers.

---

## 13. Putting It All Together: What Happens When You Open a Website

A classic interview question. Here's how every concept above connects when you type `https://example.com`:

```
 1. DHCP   Your laptop already got an IP, gateway, and DNS server when it joined Wi-Fi.
 2. DNS    Browser → cache → resolver → root → .com → authoritative
           → "example.com = 93.184.215.14"
 3. ARP    Laptop finds the MAC address of its gateway (router) on the local network.
 4. TCP    3-way handshake with 93.184.215.14:443  (SYN → SYN-ACK → ACK)
 5. TLS    Handshake: server sends its certificate, browser verifies it, session key agreed 🔒
 6. HTTP   Browser sends:  GET / HTTP/1.1   Host: example.com
 7. NAT    Your router swaps 192.168.1.10 for your public IP on the way out (and back).
 8. Routing Packets hop across routers (what traceroute shows) to the server's network.
 9. Server Firewall allows 443 → load balancer → Nginx → app → response "200 OK" + HTML
10. Render Browser decrypts, parses HTML, fetches CSS/JS/images (repeating steps 2–9 as needed).
```

---

## 14. Hands-on Practice Lab

Run these on your **EC2 instance** and on your **own laptop**, and compare the results.

```bash
# --- Your identity on the network ---
ip addr                     # (mac: ifconfig) → find your private IP and CIDR
ip route                    # find your default gateway
curl ifconfig.me            # your public IP → different from the above? That's NAT!

# --- DNS ---
nslookup google.com
dig +short github.com
dig +trace example.com      # watch the root → TLD → authoritative chain
nslookup -type=MX gmail.com
cat /etc/hosts

# --- Reachability & path ---
ping -c 4 8.8.8.8           # IP works but ping google.com fails? → DNS problem
traceroute google.com       # (Windows: tracert) count the hops

# --- Ports ---
telnet google.com 443       # should connect (Ctrl+] then quit)
telnet google.com 25        # likely times out (blocked)
nc -zv google.com 80
sudo ss -tulnp              # what's listening on YOUR machine?

# --- HTTP / HTTPS ---
curl -I http://google.com   # look for a 301 redirect to https
curl -vI https://google.com # find the TLS version and certificate issuer in the output
```

### Mini-project: script it (ties to Lecture 2)
Write `netcheck.sh <host> <port>`. It should:
1. Resolve the host with `dig +short` and exit 1 if DNS fails.
2. `ping -c 2` it and report reachable or not (don't exit, since ICMP may be blocked).
3. Test the port with `nc -zv -w 3` and report open or closed.

<details>
<summary><b>Solution</b></summary>

```bash
#!/bin/bash
HOST="$1"; PORT="${2:-80}"
[ -z "$HOST" ] && { echo "Usage: $0 <host> [port]"; exit 1; }

IP=$(dig +short "$HOST" | tail -1)
if [ -z "$IP" ]; then echo "❌ DNS: cannot resolve $HOST"; exit 1; fi
echo "✅ DNS: $HOST → $IP"

if ping -c 2 -W 2 "$HOST" > /dev/null 2>&1; then
    echo "✅ PING: reachable"
else
    echo "⚠️  PING: no reply (may be blocked by firewall)"
fi

if nc -zv -w 3 "$HOST" "$PORT" > /dev/null 2>&1; then
    echo "✅ PORT $PORT: open"
else
    echo "❌ PORT $PORT: closed or filtered"; exit 1
fi
```
</details>

### Subnetting drills (answers hidden)
<details><summary><b>1. <code>192.168.5.200/27</code>: network, broadcast, highest host, usable count?</b></summary>

Block 32 → blocks …, 160, **192**, 224. Network `192.168.5.192`, broadcast `.223`, highest host **`.222`**, usable 30.
</details>
<details><summary><b>2. <code>10.1.1.17/29</code>: network and host range?</b></summary>

Block 8 → 16, 24. Network `10.1.1.16`, hosts `.17`–`.22`, broadcast `.23`.
</details>
<details><summary><b>3. <code>172.20.99.10/22</code>: network and broadcast?</b></summary>

3rd octet, block 4 → 96, 100. Network `172.20.96.0`, broadcast `172.20.99.255`, highest host `172.20.99.254`, usable 1,022.
</details>
<details><summary><b>4. How many /28 subnets fit in a /24?</b></summary>

2^(28−24) = **16** subnets, each with 14 usable hosts.
</details>
<details><summary><b>5. Smallest prefix for 500 hosts?</b></summary>

2⁹ − 2 = 510 ≥ 500 → 9 host bits → **/23**.
</details>
<details><summary><b>6. Are <code>10.0.0.100/25</code> and <code>10.0.0.130/25</code> in the same subnet?</b></summary>

No. .100 is in `10.0.0.0/25` and .130 is in `10.0.0.128/25`.
</details>

---

## 15. Quick Revision Cheat Sheet

```
IPv4          32-bit, 4 octets (0-255), ~4.3B        IPv6  128-bit hex, a:b::c
PRIVATE       10.0.0.0/8   172.16.0.0/12   192.168.0.0/16      LOOPBACK 127.0.0.1
ANYWHERE      0.0.0.0/0

SUBNET        host bits h = 32 - prefix     total = 2^h     usable = 2^h - 2
              block = 256 - mask octet      network = multiple of block ≤ IP
              broadcast = next network - 1  highest host = broadcast - 1
MASK OCTETS   128 192 224 240 248 252 254 255
COMMON        /24=254  /25=126  /26=62  /27=30  /28=14  /29=6  /30=2   (AWS: −5 not −2)

DHCP          auto IP, mask, gateway, DNS, lease     DORA     UDP 67/68
NAT           many private IPs → one public IP (saves IPv4); NAT table tracks ports
DNS           name → IP     A, AAAA, CNAME, MX, NS, TXT, PTR     TTL     UDP 53
              browser → OS → resolver → root → TLD → authoritative

OSI (7→1)     Application  Presentation  Session  Transport  Network  Data-link  Physical
              "All People Seem To Need Data Processing"
              L4 = TCP/UDP + ports   L3 = IP + routers   L2 = MAC + switches

TCP           reliable, ordered, handshake (SYN, SYN-ACK, ACK): HTTP, SSH, DB
UDP           fast, no guarantees: DNS, DHCP, video, games, VoIP

HTTP 80       plaintext                 HTTPS 443 = HTTP + TLS (encrypt + integrity + identity)
STATUS        2xx ok  3xx redirect  4xx client error  5xx server error (502/504 = backend issue)
CERT          domain + public key + CA signature + expiry

PORTS         22 SSH  23 Telnet  53 DNS  80 HTTP  443 HTTPS  3306 MySQL  5432 Postgres

TOOLS         ip addr / ifconfig / ipconfig     ping -c 4 host     traceroute / tracert
              nslookup / dig +short             telnet host port  /  nc -zv host port
              curl -vI url                      ss -tulnp
DEBUG         refused = app/port issue          timeout = firewall issue
              127.0.0.1 bind = local only       0.0.0.0 bind = all interfaces

SECURITY      HTTPS everywhere · firewall least privilege · SSH from your IP only · VPN / bastion
```

---

## 16. Likely Exam / Interview Questions

<details>
<summary><b>Q1. What is the difference between IPv4 and IPv6?</b></summary>

IPv4 is 32-bit (~4.3 billion addresses), written in dotted decimal. IPv6 is 128-bit (~3.4×10³⁸), written in hex with colons. IPv6 was created because IPv4 addresses ran out, and it removes the need for NAT.
</details>

<details>
<summary><b>Q2. What does <code>/24</code> mean in <code>192.168.1.0/24</code>?</b></summary>

The first 24 bits are the network part (mask 255.255.255.0) and the remaining 8 bits are for hosts: 256 addresses, 254 usable.
</details>

<details>
<summary><b>Q3. Find the highest usable host in <code>192.168.10.75/26</code>.</b></summary>

Block size 64 → network .64, broadcast .127 → highest host **192.168.10.126**.
</details>

<details>
<summary><b>Q4. Why do we subtract 2 when counting usable hosts?</b></summary>

The first address is the network address and the last is the broadcast address. Neither can be assigned to a host.
</details>

<details>
<summary><b>Q5. What problem does CIDR solve?</b></summary>

Classful addressing allowed only /8, /16, or /24 networks, which wasted many addresses. CIDR allows any prefix length, so networks can be sized precisely and routes can be aggregated.
</details>

<details>
<summary><b>Q6. Explain DHCP and the DORA process.</b></summary>

DHCP automatically assigns IP, mask, gateway, and DNS settings. Discover (the client broadcasts) → Offer (the server proposes an IP) → Request (the client accepts) → Acknowledge (the server confirms and starts the lease).
</details>

<details>
<summary><b>Q7. What is NAT and why is it used?</b></summary>

NAT translates private IPs to a public IP (and back), so many devices can share one public address. This conserves IPv4 addresses and hides internal hosts from the internet.
</details>

<details>
<summary><b>Q8. How does DNS resolution work? Name some record types.</b></summary>

Browser/OS cache → recursive resolver → root → TLD → authoritative server, then the answer is cached for its TTL. Records: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), NS (nameserver), TXT, PTR (reverse).
</details>

<details>
<summary><b>Q9. Name the 7 OSI layers and give one protocol for each.</b></summary>

Application (HTTP), Presentation (TLS), Session (RPC), Transport (TCP), Network (IP), Data Link (Ethernet), Physical (cables/fibre).
</details>

<details>
<summary><b>Q10. TCP vs. UDP: when would you use each?</b></summary>

TCP for reliable, ordered delivery (web, SSH, file transfer, databases). UDP for low latency where some loss is acceptable (DNS, streaming, gaming, VoIP).
</details>

<details>
<summary><b>Q11. What is the difference between HTTP and HTTPS?</b></summary>

HTTPS is HTTP inside a TLS-encrypted connection (port 443 instead of 80). It adds confidentiality, integrity, and server authentication through a CA-signed certificate.
</details>

<details>
<summary><b>Q12. What does a TLS certificate contain, and who issues it?</b></summary>

The domain name, the server's public key, validity dates, and a signature from a Certificate Authority (e.g., Let's Encrypt). Browsers trust certificates signed by CAs in their trust store.
</details>

<details>
<summary><b>Q13. <code>ping</code> to an EC2 instance fails but the website loads. Why?</b></summary>

Ping uses ICMP, which the security group doesn't allow by default. HTTP on port 80/443 is allowed, so the site works.
</details>

<details>
<summary><b>Q14. <code>telnet host 5432</code> says "Connection refused". What does that tell you vs. a timeout?</b></summary>

Refused: the host is reachable but nothing is listening on that port (app down or wrong port). Timeout: packets are being dropped, most likely by a firewall or security group.
</details>

<details>
<summary><b>Q15. What does <code>traceroute</code> show and how does it work?</b></summary>

It shows each router hop on the way to a destination, with its latency. It sends packets with increasing TTL values. Each router where the TTL hits 0 replies with ICMP Time Exceeded, which reveals that router.
</details>

<details>
<summary><b>Q16. Why should you never use Telnet for remote login?</b></summary>

It sends everything, including credentials, in plaintext. Use SSH (port 22), which is encrypted.
</details>

<details>
<summary><b>Q17. What happens when you type a URL into the browser?</b></summary>

DNS resolution → TCP handshake → TLS handshake → HTTP request → routing through NAT and routers → server (firewall → load balancer → app) → HTTP response → the browser renders the page. See section 13.
</details>
