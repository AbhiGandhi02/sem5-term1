# Networking & Web Servers — Practice Set 2

> Attempt all 25 first. The answer key with explanations is at the bottom.

---

### 1. Client-only System Call
Which system call does an ordinary TCP client make that a TCP server normally never makes?
- A. bind()
- B. listen()
- C. connect()
- D. read()

### 2. htonl() on x86
On an x86 machine, what does `htonl(1)` return (as a decimal)?
- A. 1
- B. 256
- C. 65536
- D. 16777216

### 3. Little-endian Memory Layout
The 32-bit value `0x12345678` is stored in memory on x86. Which bytes are at increasing addresses?
- A. 12 34 56 78
- B. 78 56 34 12
- C. 34 12 78 56
- D. 56 78 12 34

### 4. listen() Backlog
What does the `backlog` argument of `listen()` control?
- A. The maximum number of bytes buffered per socket
- B. The total number of clients the server can ever serve
- C. The queue of connections waiting to be accept()ed by the application
- D. The number of threads the kernel spawns for the socket

### 5. TIME_WAIT
Which side of a TCP connection enters TIME_WAIT, and why?
- A. The passive closer, so it can resend its FIN
- B. The active closer, for 2×MSL, so the final ACK can be resent and old duplicate segments die out
- C. Both sides, until the application calls close() again
- D. The server only, to stop port reuse by attackers

### 6. Base64 Output Length
How many Base64 characters (with padding) does a 10-byte input produce?
- A. 12
- B. 14
- C. 16
- D. 20

### 7. MSS over IPv6
An Ethernet link with a 1500-byte MTU carrying IPv6 and TCP, with no extension headers or options. What is the MSS?
- A. 1460 bytes
- B. 1440 bytes
- C. 1480 bytes
- D. 1420 bytes

### 8. Random Initial Sequence Numbers
Why should TCP choose its initial sequence number unpredictably?
- A. To spread connections evenly across CPU cores
- B. So an off-path attacker can't guess it and inject or spoof segments
- C. Because sequence numbers must be unique across the whole Internet
- D. To make the 3-way handshake shorter

### 9. Concatenated SMS
A long GSM-7 SMS is split into parts. Each part carries a 6-byte User Data Header (UDH). How many characters fit per part?
- A. 160
- B. 153
- C. 140
- D. 134

### 10. Another PASV Reply
`227 Entering Passive Mode (10,0,0,5,4,1)`. Which port does the client connect to?
- A. 41
- B. 401
- C. 1025
- D. 5041

### 11. FTP Active Mode and NAT
Why does FTP active mode often break when the client is behind NAT or a firewall?
- A. Active mode encrypts the control channel, which NAT can't read
- B. The server opens the data connection back to the client, and NAT/firewalls block inbound connections
- C. Active mode uses UDP for data
- D. NAT rewrites the PORT command to port 21

### 12. Ending SMTP DATA
How does an SMTP client signal the end of the message body after `DATA`?
- A. Sending `QUIT`
- B. Closing the TCP connection
- C. A line containing only a single `.` (CRLF.CRLF)
- D. Sending a zero-length chunk

### 13. How Bcc Works
How does SMTP deliver Bcc recipients without showing them to others?
- A. Bcc recipients get a separately encrypted copy
- B. They are listed in `RCPT TO` (envelope) but left out of the `To:`/`Cc:` headers
- C. The server removes the Bcc header after the first delivery
- D. Bcc uses a separate SMTP port

### 14. Edge-triggered epoll
With `EPOLLET` (edge-triggered), what must the application do when a socket becomes readable?
- A. Read exactly one byte and return to epoll_wait
- B. Read until the call returns EAGAIN, or remaining data may never trigger another event
- C. Re-register the descriptor with epoll_ctl after every read
- D. Nothing different — it behaves exactly like level-triggered

### 15. poll() vs select()
What does `poll()` fix compared with `select()`, and what does it not fix?
- A. Fixes O(n) scanning; keeps the 1024 limit
- B. Removes the FD_SETSIZE limit; still scans all n descriptors every call
- C. Fixes both the limit and the O(n) scan
- D. Fixes neither; it is just a renamed select

### 16. Thread Pool Throughput
A server has a fixed pool of 50 threads, each serving one request at a time, and each request takes 100 ms. What is the maximum throughput?
- A. 50 req/s
- B. 100 req/s
- C. 500 req/s
- D. 5,000 req/s

### 17. CGI POST Body
How does a CGI program receive the body of a POST request?
- A. In the QUERY_STRING environment variable
- B. On standard input, with its length in CONTENT_LENGTH
- C. As command-line arguments
- D. Through a temporary file named in HTTP_BODY

### 18. Why FastCGI
What is the main advantage of FastCGI over classic CGI?
- A. It runs the application inside the web server's address space
- B. Application processes are long-lived and reused across requests, removing fork+exec per request
- C. It replaces HTTP with a binary protocol for browsers
- D. It removes the need for environment parameters

### 19. nginx proxy_pass Trailing Slash
Given `location /api/ { proxy_pass http://backend/; }`, a request for `/api/users` is forwarded as:
- A. `/api/users`
- B. `/api//users`
- C. `/users`
- D. `/backend/api/users`

### 20. Counting fork()
How many times is "hi" printed?
```c
fork();
fork();
printf("hi\n");
```
- A. 2
- B. 3
- C. 4
- D. 8

### 21. Zombie Process
What is a zombie process?
- A. A process stuck in an infinite loop
- B. A child that has exited, but whose parent hasn't called wait() to collect its status
- C. A process whose parent died, now adopted by init
- D. A process blocked on network I/O

### 22. AF_INET + SOCK_DGRAM
`socket(AF_INET, SOCK_DGRAM, 0)` creates which kind of socket?
- A. TCP over IPv4
- B. UDP over IPv4
- C. Raw IP socket
- D. Unix domain datagram socket

### 23. TLV Length Field Limit
A TLV format uses a 1-byte type and a 2-byte length. What is the largest value a single field can carry?
- A. 255 bytes
- B. 4,096 bytes
- C. 65,535 bytes
- D. 4 GB

### 24. Go Scheduling Model
How does the Go runtime run thousands of goroutines?
- A. One OS process per goroutine
- B. One OS thread per goroutine (1:1)
- C. Multiplexes many goroutines onto a small number of OS threads (M:N)
- D. Runs all goroutines on a single thread with no parallelism

### 25. nginx Location Priority
Which `location` modifier has the highest matching priority in nginx?
- A. `~` (case-sensitive regex)
- B. `=` (exact match)
- C. `^~` (prefix, stop regex search)
- D. No modifier (plain prefix)

---
---

## Answer Key

| Q | Ans | Q | Ans | Q | Ans | Q | Ans | Q | Ans |
|---|---|---|---|---|---|---|---|---|---|
| 1 | C | 6 | C | 11 | B | 16 | C | 21 | B |
| 2 | D | 7 | B | 12 | C | 17 | B | 22 | B |
| 3 | B | 8 | B | 13 | B | 18 | B | 23 | C |
| 4 | C | 9 | B | 14 | B | 19 | C | 24 | C |
| 5 | B | 10 | C | 15 | B | 20 | C | 25 | B |

## Explanations

1. **C. connect()**: The client starts the handshake with connect(). bind(), listen() and accept() belong to the server side.
2. **D. 16777216**: htonl swaps 0x00000001 to 0x01000000, which is 2²⁴ = 16777216.
3. **B. 78 56 34 12**: Little-endian puts the least significant byte at the lowest address.
4. **C**: Backlog is the queue of connections that finished the handshake but haven't been accept()ed yet. It doesn't cap total clients.
5. **B**: The active closer waits 2×MSL, so a lost final ACK can be resent and stray old segments don't land in a new connection.
6. **C. 16**: ⌈10/3⌉ = 4 groups, and 4 × 4 = 16 characters (the last group is padded with `==`).
7. **B. 1440**: 1500 − 40 (IPv6 header) − 20 (TCP header) = 1440.
8. **B**: A predictable ISN lets an attacker forge segments without seeing the traffic, as in the Mitnick attack.
9. **B. 153**: (140 − 6) bytes × 8 / 7 bits = 153 characters per part.
10. **C. 1025**: 4 × 256 + 1 = 1025.
11. **B**: In active mode the server connects back to the client's PORT, and NAT/firewalls drop that inbound connection. Passive mode avoids it.
12. **C**: `<CRLF>.<CRLF>` ends DATA. A line in the body that starts with "." gets an extra dot (dot-stuffing).
13. **B**: The envelope (`RCPT TO`) decides delivery and the headers are just text, so Bcc addresses go only in the envelope.
14. **B**: Edge-triggered fires only when new data arrives. Data left unread won't wake you again, so drain the socket until EAGAIN.
15. **B**: poll takes an array of any size, so there's no 1024 cap, but the kernel still scans all n on every call.
16. **C. 500**: Little's Law gives λ = L / W = 50 / 0.1 s = 500 req/s.
17. **B**: The body arrives on stdin and the program reads CONTENT_LENGTH bytes. QUERY_STRING is only for the URL query.
18. **B**: Persistent workers talk over a socket, so there's no process-per-request cost and they stay isolated from the server.
19. **C. /users**: When proxy_pass has a URI (here `/`), the matched `/api/` part is replaced by it.
20. **C. 4**: Each fork doubles the number of processes: 1 → 2 → 4, and all of them print.
21. **B**: The child's exit status stays in the process table until the parent calls wait(). Option C describes an orphan.
22. **B. UDP over IPv4**: AF_INET selects IPv4 and SOCK_DGRAM selects UDP. SOCK_STREAM would give TCP.
23. **C. 65,535**: A 2-byte length can go up to 2¹⁶ − 1 = 65,535 bytes.
24. **C**: The Go scheduler multiplexes many goroutines onto GOMAXPROCS OS threads. That's what makes 2 KB goroutines cheap.
25. **B. `=`**: An exact match wins immediately. The order after that is `^~`, then regex (`~`/`~*`), then plain prefix.
