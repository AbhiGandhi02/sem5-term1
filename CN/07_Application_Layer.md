# 7. Application Layer

## Role of the Application Layer

- **Layer 7** of the OSI model (Layer 5 in TCP/IP)
- Provides **network services directly to end-user applications**
- Interface between the user and the network
- PDU: **Data** (Messages)

---

## DNS (Domain Name System)

### What is DNS?
- Translates **domain names** (www.google.com) into **IP addresses** (142.250.190.4)
- Acts as the **"phonebook" of the Internet**
- Uses **UDP port 53** (TCP for zone transfers and large responses)

### DNS Hierarchy
```
                    . (Root)
                   / | \
               .com .org .edu .in ...     ← Top-Level Domains (TLD)
              / | \
         google  amazon  facebook ...     ← Second-Level Domains
          / \
       www  mail  maps ...                ← Subdomains
```

### Types of DNS Servers
1. **Root DNS Server** — 13 root server clusters (A to M)
   - Knows TLD servers
2. **TLD (Top-Level Domain) Server** — manages .com, .org, .edu, .in, etc.
   - Knows authoritative servers
3. **Authoritative DNS Server** — holds actual DNS records for a domain
4. **Local DNS Server (Resolver)** — first point of contact, caches results

### DNS Resolution Process
```
User types: www.example.com

1. Browser → Local DNS Resolver (cache check)
2. Local Resolver → Root Server ("Who handles .com?")
3. Root → returns TLD server for .com
4. Local Resolver → TLD Server ("Who handles example.com?")
5. TLD → returns Authoritative Server for example.com
6. Local Resolver → Authoritative Server ("What is www.example.com?")
7. Authoritative → returns IP address
8. Local Resolver → caches result → returns IP to browser
```

### Types of DNS Resolution
- **Recursive Query**: Client asks resolver to get the final answer (resolver does all the work)
- **Iterative Query**: Server returns the best answer it has (referral), client follows up

### DNS Record Types

| Record | Full Form | Purpose | Example |
|--------|-----------|---------|---------|
| **A** | Address | Maps domain → IPv4 | example.com → 93.184.216.34 |
| **AAAA** | — | Maps domain → IPv6 | example.com → 2606:2800:... |
| **CNAME** | Canonical Name | Alias for another domain | www.example.com → example.com |
| **MX** | Mail Exchange | Mail server for domain | example.com → mail.example.com |
| **NS** | Name Server | Authoritative DNS for domain | example.com → ns1.example.com |
| **PTR** | Pointer | Reverse DNS (IP → domain) | 34.216.184.93 → example.com |
| **SOA** | Start of Authority | Primary info about DNS zone | TTL, admin email, serial |
| **TXT** | Text | Arbitrary text (SPF, DKIM) | Verification records |

### DNS Caching
- Results cached at each level (browser, OS, resolver)
- **TTL (Time To Live)** — how long record is cached before refresh
- Reduces load on DNS servers, speeds up resolution

---

## HTTP / HTTPS

### HTTP (HyperText Transfer Protocol)
- **Application layer protocol** for the World Wide Web
- **Port 80** (HTTP) / **Port 443** (HTTPS)
- **Stateless** — each request is independent (no memory of previous requests)
- **Client-Server model**: Browser (client) sends request, web server responds

### HTTP Request Format
```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: text/html
Connection: keep-alive
```

### HTTP Response Format
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234
Server: Apache

<html>...</html>
```

### HTTP Methods

| Method | Purpose | Idempotent? | Body? |
|--------|---------|-------------|-------|
| **GET** | Retrieve resource | Yes | No |
| **POST** | Submit data / Create resource | No | Yes |
| **PUT** | Update/Replace resource | Yes | Yes |
| **PATCH** | Partial update | No | Yes |
| **DELETE** | Delete resource | Yes | No |
| **HEAD** | Like GET but headers only | Yes | No |
| **OPTIONS** | Get supported methods | Yes | No |

### HTTP Status Codes

| Code Range | Category | Examples |
|------------|----------|---------|
| **1xx** | Informational | 100 Continue |
| **2xx** | Success | 200 OK, 201 Created, 204 No Content |
| **3xx** | Redirection | 301 Moved Permanently, 302 Found, 304 Not Modified |
| **4xx** | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| **5xx** | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

### HTTP Versions

| Version | Key Feature |
|---------|-------------|
| **HTTP/1.0** | New TCP connection per request |
| **HTTP/1.1** | Persistent connections (keep-alive), pipelining |
| **HTTP/2** | Binary framing, multiplexing, header compression, server push |
| **HTTP/3** | Uses QUIC (over UDP), faster connection setup |

### HTTPS
- HTTP + **TLS/SSL encryption**
- Encrypts data in transit → prevents eavesdropping
- Uses **certificates** (issued by Certificate Authorities)
- Port **443**

### Persistent vs Non-Persistent HTTP

| Feature | Non-Persistent (HTTP/1.0) | Persistent (HTTP/1.1+) |
|---------|--------------------------|----------------------|
| **Connection** | New TCP per request | Reuse TCP connection |
| **Overhead** | High (3-way handshake each time) | Low |
| **RTTs per object** | 2 RTT (1 for TCP + 1 for request) | 1 RTT (after first) |
| **Performance** | Slow | Fast |

---

## FTP (File Transfer Protocol)

- Used to **transfer files** between client and server
- Uses **two connections**:
  - **Control connection**: Port **21** (commands, authentication)
  - **Data connection**: Port **20** (actual file transfer)
- **Stateful** — maintains session state
- **Modes**:
  - **Active mode**: Server initiates data connection to client
  - **Passive mode**: Client initiates data connection to server (firewall-friendly)

### FTP Commands
| Command | Purpose |
|---------|---------|
| `USER` | Send username |
| `PASS` | Send password |
| `LIST` | List files in directory |
| `RETR` | Retrieve (download) file |
| `STOR` | Store (upload) file |
| `QUIT` | Close connection |

---

## Email Protocols

### SMTP (Simple Mail Transfer Protocol)
- **Sends** email from client to server, and server to server
- **Port 25** (or 587 with TLS)
- **Push protocol** — client pushes mail to server
- Uses **TCP**
- Text-based commands: HELO, MAIL FROM, RCPT TO, DATA, QUIT

### POP3 (Post Office Protocol v3)
- **Downloads** email from server to client
- **Port 110** (or 995 with SSL)
- **Pull protocol**
- **Downloads and deletes** from server (by default)
- Simple, limited features

### IMAP (Internet Message Access Protocol)
- **Accesses** email on the server
- **Port 143** (or 993 with SSL)
- **Pull protocol**
- Emails **stay on server** — synced across devices
- Supports folders, search, partial fetch
- More complex but more powerful than POP3

### Email Flow
```
[Sender] → SMTP → [Sender's Mail Server] → SMTP → [Receiver's Mail Server]
                                                          ↓
                                              POP3/IMAP ← [Receiver]
```

### POP3 vs IMAP

| Feature | POP3 | IMAP |
|---------|------|------|
| **Port** | 110 | 143 |
| **Storage** | Downloads & deletes | Keeps on server |
| **Multi-device** | No | Yes |
| **Offline** | Yes (downloaded) | Partial |
| **Complexity** | Simple | Complex |
| **Bandwidth** | Less (one-time download) | More (sync) |

---

## DHCP (Dynamic Host Configuration Protocol)

- Automatically assigns **IP addresses** and network config to devices
- **Port**: Server = 67, Client = 68
- Uses **UDP**
- Reduces manual configuration

### DHCP Process (DORA)
```
Client                          Server
  |                                |
  |---- DISCOVER (broadcast) ---→ |    "Any DHCP server out there?"
  |                                |
  |←---- OFFER (unicast/bc) ------| 	 "Here's an available IP"
  |                                |
  |---- REQUEST (broadcast) ----→ |    "I'll take that IP"
  |                                |
  |←---- ACK (unicast/bc) --------| 	 "It's yours! Here's the config"
  |                                |
```

1. **Discover**: Client broadcasts to find DHCP servers
2. **Offer**: Server offers an IP address
3. **Request**: Client requests the offered IP
4. **Acknowledge**: Server confirms the assignment

### DHCP Lease
- IP address assigned for a **limited time** (lease period)
- Client must **renew** before lease expires
- If not renewed, IP returns to the pool

---

## Other Application Layer Protocols

### Telnet
- **Remote login** to another computer via command line
- Port **23**
- **Unencrypted** — all data (including passwords) sent in plaintext
- **Replaced by SSH**

### SSH (Secure Shell)
- **Secure remote login** and command execution
- Port **22**
- **Encrypted** — all data encrypted using public-key cryptography
- Also supports **SCP** (secure copy) and **SFTP** (secure FTP)

### SNMP (Simple Network Management Protocol)
- Used to **manage and monitor** network devices (routers, switches, servers)
- Port **161** (agent), **162** (trap)
- Uses **UDP**
- Components: **Manager** (monitoring station), **Agent** (on managed device), **MIB** (Management Information Base)

---

## Client-Server vs Peer-to-Peer (P2P)

| Feature | Client-Server | Peer-to-Peer |
|---------|--------------|-------------|
| **Architecture** | Centralized server | All nodes equal |
| **Scalability** | Server bottleneck | Scales with peers |
| **Reliability** | Server failure = system down | Resilient |
| **Security** | Easier to manage | Harder to secure |
| **Example** | Web, Email, DNS | BitTorrent, Blockchain |
| **Cost** | High (server maintenance) | Low |

---

## Quick Revision Checklist

- [ ] DNS: hierarchy (Root → TLD → Authoritative), resolution process, record types (A, AAAA, CNAME, MX, NS)
- [ ] DNS: Recursive vs Iterative query, caching, TTL
- [ ] HTTP: Methods (GET, POST, PUT, DELETE), status codes (200, 301, 404, 500)
- [ ] HTTP versions: 1.0 vs 1.1 vs 2 vs 3
- [ ] HTTPS = HTTP + TLS, port 443
- [ ] Persistent vs Non-persistent connections
- [ ] FTP: dual connection (port 21 control, 20 data), active vs passive mode
- [ ] SMTP (port 25, push) vs POP3 (port 110, pull, download) vs IMAP (port 143, pull, sync)
- [ ] Email flow: Sender → SMTP → Server → SMTP → Server → POP3/IMAP → Receiver
- [ ] DHCP: DORA process, UDP ports 67/68, lease concept
- [ ] Telnet (23, insecure) vs SSH (22, encrypted)
- [ ] Client-Server vs P2P architecture
