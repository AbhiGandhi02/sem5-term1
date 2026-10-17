# Networking & Web Servers — MCQ Answers

### 1. Client vs Server System Calls
A TCP server is built from seven system calls. Which one is never made by an ordinary TCP client?
- socket()
- **accept()** ✅
- write()
- close()

**Why:** Only a server accepts incoming connections from a listening socket. A client calls `connect()` instead.

---

### 2. Byte Order and htons()
`addr.sin_port = htons(2026);` and on the wire the port bytes read `07 EA`. Why is htons() needed?
- To reserve the port with the kernel before bind()
- To widen the port from 8 to 16 bits
- **Because x86 stores 0x07EA little-endian (EA 07) and network byte order is big-endian** ✅
- Because ports above 1024 must be encoded as unsigned

**Why:** x86 keeps the low byte first, but network order is big-endian. htons swaps the bytes so `07 EA` goes on the wire.

---

### 3. Privileged Ports and Front Proxies
Why do real deployments bind 8080 as an unprivileged user and let a front proxy own 80 and 443?
- Because 8080 is handled by a faster kernel path
- **Ports below 1024 require root, and a bug in an app's request parser running as root is a root compromise** ✅
- Because HTTP/1.1 keep-alive only works above 1024
- Because the kernel reserves 80 and 443 for TLS termination

**Why:** Binding a port below 1024 needs root. Running the app unprivileged limits the damage from any exploit.

---

### 4. TLV Forward Compatibility
TLV (type-length-value) is called forward compatible. What property gives it that?
- Types are sorted, so a parser can binary-search for the field it wants
- Every value is a fixed size, so field offsets never move
- Values are base64-encoded, so unknown bytes always survive
- **The length tells a parser how far to jump, so it can skip a field type it has never seen** ✅

**Why:** An old parser reads the length of an unknown type and jumps over that many bytes, so new fields don't break it.

---

### 5. Cost of Base64
Base64 exists so binary can travel through text-only channels. What does it cost?
- **About 33% more bytes: 3 input bytes become 4 output characters** ✅
- Exactly 2x the bytes: each byte becomes two characters
- Nothing — it is the same size, just re-encoded
- About 12% more bytes, plus a 4-byte header per message

**Why:** 3 bytes (24 bits) split into four 6-bit chunks, giving 4 characters. 4/3 is about 1.33x.

---

### 6. MSS on a 1500-byte MTU
An Ethernet link with a 1500-byte MTU carrying IPv4 and TCP, neither with options. What is the MSS?
- 1500 bytes
- 1480 bytes
- 1518 bytes
- **1460 bytes** ✅

**Why:** MSS = 1500 − 20 (IPv4 header) − 20 (TCP header) = 1460.

---

### 7. Why Closing Takes Four Segments
Opening a TCP connection takes three segments, but closing it takes four. Why the extra one?
- TIME_WAIT needs its own round trip before the socket is released
- **SYN and ACK can share one segment, but each side's FIN needs its own ACK — the peer may still have data to send** ✅
- The active closer must send FIN twice in case the first is lost
- Closing carries no sequence numbers, so an extra confirmation is required

**Why:** Close is half-duplex: the peer ACKs your FIN right away but may keep sending data, and sends its own FIN later.

---

### 8. Cost of SYN Cookies
SYN cookies encode the connection parameters into the server's own ISN. What is the cost?
- The handshake needs a fourth segment to confirm the cookie
- Initial sequence numbers become predictable, reopening the Mitnick attack
- **There are only 32 bits of ISN, so most TCP options — window scale, SACK, timestamps — are lost for that connection** ✅
- The client must support TCP Fast Open

**Why:** The server stores no state, and 32 bits only has room for a few things (such as an MSS index), so the other options are dropped.

---

### 9. SMS Limit Under UCS-2
GSM-7 packing fits 160 characters into 140 octets. One emoji switches TP-DCS to UCS-2. What is the new limit?
- 160 characters
- 140 characters
- 80 characters
- **70 characters** ✅

**Why:** UCS-2 uses 2 bytes per character, and 140 / 2 = 70.

---

### 10. Decoding a PASV Reply
`227 Entering Passive Mode (127,0,0,1,117,48)`. Which port does the client connect to?
- 117
- **30000** ✅
- 11748
- 48

**Why:** Port = p1 × 256 + p2 = 117 × 256 + 48 = 29952 + 48 = 30000.

---

### 11. SMTP Envelope vs Header
`MAIL FROM:<jeet@scaler.com>` but `From: someone-else@bank.com` inside DATA. What does the base protocol do?
- It rejects the message with a 550
- It rewrites the From: header to match MAIL FROM
- **Nothing — MAIL FROM is the envelope and From: is text on the letter; the base protocol checks neither** ✅
- It accepts the mail but strips the DATA headers

**Why:** Base SMTP never checks the envelope against the headers, which is why spoofing works. SPF, DKIM and DMARC were added later to catch it.

---

### 12. select() and FD_SETSIZE
A select()-based server dies at descriptor 1024 and `ulimit -n 65536` changes nothing. Why?
- The kernel refuses more than 1024 sockets per process
- **fd_set is a fixed-width bitmap — FD_SETSIZE is compiled into a header, so fd 1024 cannot be represented at all** ✅
- select() takes maxfd as a 10-bit argument
- The listen() backlog silently caps at 1024

**Why:** The fd_set bitmap has a fixed size of 1024 bits set at compile time, so raising ulimit can't make it bigger.

---

### 13. select vs epoll at Scale
10,000 connections are open and 10 have data waiting. How do select and epoll compare?
- Both do work proportional to the 10 ready descriptors
- select is O(ready) and epoll is O(watched)
- epoll must rebuild its interest set on every call, so it is slower at this scale
- **select rebuilds and scans all 10,000 every call; epoll returns only the 10 that fired** ✅

**Why:** select is O(watched) on every call. epoll keeps its interest set in the kernel and returns only the ready list, so it is O(ready).

---

### 14. Little's Law — In-flight Requests
500 requests/sec at 200 ms average latency. How many requests are in flight, and what happens if latency rises to 1 s?
- **100 now, rising to 500** ✅
- 500 now, rising to 2,500
- 100 now, unchanged — latency does not affect concurrency
- 2,500 now, falling to 500

**Why:** Little's Law says L = λ × W. 500 × 0.2 = 100, and 500 × 1 = 500.

---

### 15. Cost of In-Process Modules
mod_php and mod_perl removed CGI's fork+exec by compiling the app into the web server. What was the trade-off?
- Requests became stateful, so load balancing broke
- The application could no longer read CGI environment variables
- **The application shares an address space with the web server, so one segfault takes both down** ✅
- Each request needed a second TCP connection to the module

**Why:** The app and the server run in the same process, so there's no isolation. An app crash or memory bug brings down the whole server.

---

### 16. End of FastCGI Parameters
How does the responder know the environment parameters are finished?
- A CRLF CRLF sequence, as in HTTP headers
- **A PARAMS record with content length zero — the empty record is the delimiter** ✅
- CONTENT_LENGTH counting down to zero
- The arrival of the FCGI_END_REQUEST record

**Why:** FastCGI ends a stream by sending an empty record of that type. A zero-length PARAMS record means the params are done.

---

### 17. CGI Header Variable Names
Header `User-Agent: curl` arrives. What does CGI hand the program?
- USER_AGENT=curl
- HTTP_USER-AGENT=curl
- CGI_HTTP_USERAGENT=curl
- **HTTP_USER_AGENT=curl** ✅

**Why:** CGI uppercases the header name, turns `-` into `_`, and adds the `HTTP_` prefix.

---

### 18. Goroutine vs Thread Stack
An OS thread reserves ~8 MB of stack up front. What does a goroutine start with?
- **2 KB, grown on demand** ✅
- 64 KB, grown on demand
- 2 MB, fixed
- 8 MB, but shared between goroutines

**Why:** A goroutine starts with a small 2 KB stack that the Go runtime grows as needed, so 10k goroutines are cheap.

---

### 19. sendfile vs gzip
`sendfile on;` and `gzip on;` fight each other. Why?
- **sendfile copies entirely kernel-side, so to compress (or encrypt) the bytes they must be brought into user space** ✅
- sendfile only works on files smaller than 64 KiB, and gzip buffers more than that
- gzip needs the file mmap'd, which sendfile forbids
- sendfile cannot set Content-Length, which gzip requires

**Why:** sendfile copies straight from file to socket with zero copies into user space. gzip has to read and change the bytes in user space, so the zero-copy path is lost.

---

### 20. try_files Fallback
For `try_files $uri $uri/ /index.html?$args;`, what is the role of the final argument?
- It is tested like the others, and returns 404 if missing
- **It is the fallback — the only entry that is never tested as a file** ✅
- It is tested first, so a real /index.html short-circuits the rest
- It is ignored unless error_page is also configured

**Why:** Every argument except the last is checked as a file. The last one is an internal redirect used when none of the others exist, which is common for SPAs.

---

### 21. Efficient Encoding for Game Commands
Sending up/down/left/right and space on a gaming protocol. Which encoding is most efficient?
- 7-bit
- **BCD** ✅
- Base64
- Base32

**Why:** There are only 5 commands, so a small code is enough. BCD uses 4 bits per symbol, the fewest of these options (Base32 = 5, Base64 = 6, 7-bit = 7).

---

### 22. Defending Against SYN Floods
How does the TCP protocol defend against SYN flood attacks?
- TLS
- HTTPS
- **SYN Cookies** ✅
- Sword and Shield

**Why:** The server keeps no half-open state. It encodes the connection info in the ISN and rebuilds it from the client's final ACK.

---

### 23. Meaning of AF_INET
What does AF_INET represent in a socket call?
- **IP socket** ✅
- TCP socket
- UDP socket
- Unix socket

**Why:** AF_INET is the IPv4 address family. TCP or UDP is chosen by the socket type (`SOCK_STREAM` / `SOCK_DGRAM`).

---

### 24. Output of htons(2026)
What would be the output of htons(2026)?
- 2026
- **59911** ✅
- 10000
- 2620

**Why:** 2026 = 0x07EA. Swapping the bytes on a little-endian host gives 0xEA07 = 234 × 256 + 7 = 59911.

---

### 25. Return Value of fork()
If fork() returns 0, which process are you in?
- Parent process
- **Child process** ✅
- Zombie process
- Root process

**Why:** fork() returns 0 in the child and the child's PID (> 0) in the parent. It returns −1 if it fails.
