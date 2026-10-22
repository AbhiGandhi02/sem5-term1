# Networking & Web Servers — Practice Set 1

### 1. Client vs Server System Calls
A TCP server is built from seven system calls. Which one is never made by an ordinary TCP client?
- A. socket()
- B. accept()
- C. write()
- D. close()

> ✅ **Answer: B.** Only a server accepts incoming connections from a listening socket. A client calls `connect()` instead.

---

### 2. Byte Order and htons()
`addr.sin_port = htons(2026);` and on the wire the port bytes read `07 EA`. Why is htons() needed?
- A. To reserve the port with the kernel before bind()
- B. To widen the port from 8 to 16 bits
- C. Because x86 stores 0x07EA little-endian (EA 07) and network byte order is big-endian
- D. Because ports above 1024 must be encoded as unsigned

> ✅ **Answer: C.** x86 keeps the low byte first, but network order is big-endian. htons swaps the bytes so `07 EA` goes on the wire.

---

### 3. Privileged Ports and Front Proxies
Why do real deployments bind 8080 as an unprivileged user and let a front proxy own 80 and 443?
- A. Because 8080 is handled by a faster kernel path
- B. Ports below 1024 require root, and a bug in an app's request parser running as root is a root compromise
- C. Because HTTP/1.1 keep-alive only works above 1024
- D. Because the kernel reserves 80 and 443 for TLS termination

> ✅ **Answer: B.** Binding a port below 1024 needs root. Running the app unprivileged limits the damage from any exploit.

---

### 4. TLV Forward Compatibility
TLV (type-length-value) is called forward compatible. What property gives it that?
- A. Types are sorted, so a parser can binary-search for the field it wants
- B. Every value is a fixed size, so field offsets never move
- C. Values are base64-encoded, so unknown bytes always survive
- D. The length tells a parser how far to jump, so it can skip a field type it has never seen

> ✅ **Answer: D.** An old parser reads the length of an unknown type and jumps over that many bytes, so new fields don't break it.

---

### 5. Cost of Base64
Base64 exists so binary can travel through text-only channels. What does it cost?
- A. About 33% more bytes: 3 input bytes become 4 output characters
- B. Exactly 2x the bytes: each byte becomes two characters
- C. Nothing — it is the same size, just re-encoded
- D. About 12% more bytes, plus a 4-byte header per message

> ✅ **Answer: A.** 3 bytes (24 bits) split into four 6-bit chunks, giving 4 characters. 4/3 is about 1.33x.

---

### 6. MSS on a 1500-byte MTU
An Ethernet link with a 1500-byte MTU carrying IPv4 and TCP, neither with options. What is the MSS?
- A. 1500 bytes
- B. 1480 bytes
- C. 1518 bytes
- D. 1460 bytes

> ✅ **Answer: D.** MSS = 1500 − 20 (IPv4 header) − 20 (TCP header) = 1460.

---

### 7. Why Closing Takes Four Segments
Opening a TCP connection takes three segments, but closing it takes four. Why the extra one?
- A. TIME_WAIT needs its own round trip before the socket is released
- B. SYN and ACK can share one segment, but each side's FIN needs its own ACK — the peer may still have data to send
- C. The active closer must send FIN twice in case the first is lost
- D. Closing carries no sequence numbers, so an extra confirmation is required

> ✅ **Answer: B.** Close is half-duplex: the peer ACKs your FIN right away but may keep sending data, and sends its own FIN later.

---

### 8. Cost of SYN Cookies
SYN cookies encode the connection parameters into the server's own ISN. What is the cost?
- A. The handshake needs a fourth segment to confirm the cookie
- B. Initial sequence numbers become predictable, reopening the Mitnick attack
- C. There are only 32 bits of ISN, so most TCP options — window scale, SACK, timestamps — are lost for that connection
- D. The client must support TCP Fast Open

> ✅ **Answer: C.** The server stores no state, and 32 bits only has room for a few things (such as an MSS index), so the other options are dropped.

---

### 9. SMS Limit Under UCS-2
GSM-7 packing fits 160 characters into 140 octets. One emoji switches TP-DCS to UCS-2. What is the new limit?
- A. 160 characters
- B. 140 characters
- C. 80 characters
- D. 70 characters

> ✅ **Answer: D.** UCS-2 uses 2 bytes per character, and 140 / 2 = 70.

---

### 10. Decoding a PASV Reply
`227 Entering Passive Mode (127,0,0,1,117,48)`. Which port does the client connect to?
- A. 117
- B. 30000
- C. 11748
- D. 48

> ✅ **Answer: B.** Port = p1 × 256 + p2 = 117 × 256 + 48 = 29952 + 48 = 30000.

---

### 11. SMTP Envelope vs Header
`MAIL FROM:<jeet@scaler.com>` but `From: someone-else@bank.com` inside DATA. What does the base protocol do?
- A. It rejects the message with a 550
- B. It rewrites the From: header to match MAIL FROM
- C. Nothing — MAIL FROM is the envelope and From: is text on the letter; the base protocol checks neither
- D. It accepts the mail but strips the DATA headers

> ✅ **Answer: C.** Base SMTP never checks the envelope against the headers, which is why spoofing works. SPF, DKIM and DMARC were added later to catch it.

---

### 12. select() and FD_SETSIZE
A select()-based server dies at descriptor 1024 and `ulimit -n 65536` changes nothing. Why?
- A. The kernel refuses more than 1024 sockets per process
- B. fd_set is a fixed-width bitmap — FD_SETSIZE is compiled into a header, so fd 1024 cannot be represented at all
- C. select() takes maxfd as a 10-bit argument
- D. The listen() backlog silently caps at 1024

> ✅ **Answer: B.** The fd_set bitmap has a fixed size of 1024 bits set at compile time, so raising ulimit can't make it bigger.

---

### 13. select vs epoll at Scale
10,000 connections are open and 10 have data waiting. How do select and epoll compare?
- A. Both do work proportional to the 10 ready descriptors
- B. select is O(ready) and epoll is O(watched)
- C. epoll must rebuild its interest set on every call, so it is slower at this scale
- D. select rebuilds and scans all 10,000 every call; epoll returns only the 10 that fired

> ✅ **Answer: D.** select is O(watched) on every call. epoll keeps its interest set in the kernel and returns only the ready list, so it is O(ready).

---

### 14. Little's Law — In-flight Requests
500 requests/sec at 200 ms average latency. How many requests are in flight, and what happens if latency rises to 1 s?
- A. 100 now, rising to 500
- B. 500 now, rising to 2,500
- C. 100 now, unchanged — latency does not affect concurrency
- D. 2,500 now, falling to 500

> ✅ **Answer: A.** Little's Law says L = λ × W. 500 × 0.2 = 100, and 500 × 1 = 500.

---

### 15. Cost of In-Process Modules
mod_php and mod_perl removed CGI's fork+exec by compiling the app into the web server. What was the trade-off?
- A. Requests became stateful, so load balancing broke
- B. The application could no longer read CGI environment variables
- C. The application shares an address space with the web server, so one segfault takes both down
- D. Each request needed a second TCP connection to the module

> ✅ **Answer: C.** The app and the server run in the same process, so there's no isolation. An app crash or memory bug brings down the whole server.

---

### 16. End of FastCGI Parameters
How does the responder know the environment parameters are finished?
- A. A CRLF CRLF sequence, as in HTTP headers
- B. A PARAMS record with content length zero — the empty record is the delimiter
- C. CONTENT_LENGTH counting down to zero
- D. The arrival of the FCGI_END_REQUEST record

> ✅ **Answer: B.** FastCGI ends a stream by sending an empty record of that type. A zero-length PARAMS record means the params are done.

---

### 17. CGI Header Variable Names
Header `User-Agent: curl` arrives. What does CGI hand the program?
- A. USER_AGENT=curl
- B. HTTP_USER-AGENT=curl
- C. CGI_HTTP_USERAGENT=curl
- D. HTTP_USER_AGENT=curl

> ✅ **Answer: D.** CGI uppercases the header name, turns `-` into `_`, and adds the `HTTP_` prefix.

---

### 18. Goroutine vs Thread Stack
An OS thread reserves ~8 MB of stack up front. What does a goroutine start with?
- A. 2 KB, grown on demand
- B. 64 KB, grown on demand
- C. 2 MB, fixed
- D. 8 MB, but shared between goroutines

> ✅ **Answer: A.** A goroutine starts with a small 2 KB stack that the Go runtime grows as needed, so 10k goroutines are cheap.

---

### 19. sendfile vs gzip
`sendfile on;` and `gzip on;` fight each other. Why?
- A. sendfile copies entirely kernel-side, so to compress (or encrypt) the bytes they must be brought into user space
- B. sendfile only works on files smaller than 64 KiB, and gzip buffers more than that
- C. gzip needs the file mmap'd, which sendfile forbids
- D. sendfile cannot set Content-Length, which gzip requires

> ✅ **Answer: A.** sendfile copies straight from file to socket with zero copies into user space. gzip has to read and change the bytes in user space, so the zero-copy path is lost.

---

### 20. try_files Fallback
For `try_files $uri $uri/ /index.html?$args;`, what is the role of the final argument?
- A. It is tested like the others, and returns 404 if missing
- B. It is the fallback — the only entry that is never tested as a file
- C. It is tested first, so a real /index.html short-circuits the rest
- D. It is ignored unless error_page is also configured

> ✅ **Answer: B.** Every argument except the last is checked as a file. The last one is an internal redirect used when none of the others exist, which is common for SPAs.

---

### 21. Efficient Encoding for Game Commands
Sending up/down/left/right and space on a gaming protocol. Which encoding is most efficient?
- A. 7-bit
- B. BCD
- C. Base64
- D. Base32

> ✅ **Answer: B.** There are only 5 commands, so a small code is enough. BCD uses 4 bits per symbol, the fewest of these options (Base32 = 5, Base64 = 6, 7-bit = 7).

---

### 22. Defending Against SYN Floods
How does the TCP protocol defend against SYN flood attacks?
- A. TLS
- B. HTTPS
- C. SYN Cookies
- D. Sword and Shield

> ✅ **Answer: C.** The server keeps no half-open state. It encodes the connection info in the ISN and rebuilds it from the client's final ACK.

---

### 23. Meaning of AF_INET
What does AF_INET represent in a socket call?
- A. IP socket
- B. TCP socket
- C. UDP socket
- D. Unix socket

> ✅ **Answer: A.** AF_INET is the IPv4 address family. TCP or UDP is chosen by the socket type (`SOCK_STREAM` / `SOCK_DGRAM`).

---

### 24. Output of htons(2026)
What would be the output of htons(2026)?
- A. 2026
- B. 59911
- C. 10000
- D. 2620

> ✅ **Answer: B.** 2026 = 0x07EA. Swapping the bytes on a little-endian host gives 0xEA07 = 234 × 256 + 7 = 59911.

---

### 25. Return Value of fork()
If fork() returns 0, which process are you in?
- A. Parent process
- B. Child process
- C. Zombie process
- D. Root process

> ✅ **Answer: B.** fork() returns 0 in the child and the child's PID (> 0) in the parent. It returns −1 if it fails.
