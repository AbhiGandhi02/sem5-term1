# Lesson 1: Network Programming 101
### CN at Scaler, Week 1: "Network programming, from the socket up"

> `2.5 hours` · `7 system calls` · `port 2026 = our port`
>
> **socket · bind · listen · accept · read · write · close**

**Legend:** 🧠 = my addition, *not on the slides* · 💡 **Simple words mein** = an easy (sometimes Hinglish) summary. Read it first if a section feels heavy.

---

## What this lecture is about and why it matters

Every web framework you have used (Express, Flask, Go's `net/http`, Django, Spring) is a wrapper around **seven operating-system calls**: `socket`, `bind`, `listen`, `accept`, `read`, `write`, `close`. This lecture builds bottom-up. It starts with the smallest C program that can accept a TCP connection and ends with designing your own binary protocol. Along the way it covers what goes wrong around that program (ports, the accept queue, the signal that kills your process), how a client works (DNS, byte order, the OSI layers), how to use telnet, nc, curl and openssl as clients, how to serve many clients at once (fork, select, epoll), how to look at real packets (tcpdump), what SSL/TLS actually secures, and finally how to frame and encode messages (fixed length, delimiters, length prefixes, BCD, ASN.1, TLV, base64, RPC, gRPC).

Why it matters: once you see the seven calls, nothing in networking is magic any more. When a production server "just hangs", "dies with no error", or "says address already in use", the cause is almost always one of the things in this lecture.

### The lecture's plan (4 parts, bottom-up)

```
 PART 1                 PART 2                  PART 3                   PART 4
 A server, and          A client, plus OSI,     curl, many clients at    Designing a protocol
 everything that        DNS and byte order      once, packets on the     of your own
 breaks around it                               wire, SSL
```

### The course map (8 weeks)

| Week | Topic |
|---|---|
| 1 | **Network programming 101** (this lecture) |
| 2 | Computer Networking Eagle Eye View |
| 3 | Server Side Considerations for scale |
| 4 | nginx deep dive |
| 5 | Evolution of HTTP |
| 6 | (Ab)using CDNs |
| 7 | Economics of cloud tech |
| 8 | Building for failures |

---

## Table of contents

0. [Prerequisites from scratch](#0-prerequisites-from-scratch)
1. [The Server](#1-the-server)
2. [You Already Have a Client](#2-you-already-have-a-client)
3. [Writing the Client](#3-writing-the-client)
4. [HTTP and curl](#4-http-and-curl)
5. [Many Clients at Once (plus packets and SSL)](#5-many-clients-at-once-plus-packets-and-ssl)
6. [Protocol Design](#6-protocol-design)
7. [Hands-on lab](#7-hands-on-lab)
8. [Homework / assignments](#8-homework--assignments)
9. [Summary cheat-sheet](#9-summary-cheat-sheet)
10. [Self-check questions](#10-self-check-questions)

---

## 0. Prerequisites from scratch

The slides assume you already know the ideas below. If any of them is new, read this section first. All of section 0 is 🧠 background.

### 0.1 🧠 Kernel, user space and system calls

Your computer runs two kinds of code:

- **User space**: your programs (your C code, Chrome, Python). They are not allowed to touch hardware directly.
- **The kernel**: the core of the operating system (Linux, macOS's XNU). It owns the network card, the disk, memory and the list of running processes.

When your program needs something only the kernel can do (send bytes over the network, open a file), it makes a **system call** ("syscall"). A syscall looks like a normal C function, but calling it hands control to the kernel, which does the work and hands back a result.

```
   your program (user space)
        │   socket(...)      "kernel, please make me a network endpoint"
        ▼
   ═════════════════════════  syscall boundary
        ▼
   kernel  ── talks to ──►  network card ──► the wire
```

> 💡 **Simple words mein:** Bank mein aap seedha vault (hardware) mein nahi ja sakte. Form bharo (syscall), teller (kernel) kaam karke result deta hai. Is lecture ke "7 system calls" yahi forms hain.

### 0.2 🧠 File descriptors (fds)

When you open a file in Unix, the kernel gives you back a small integer called a **file descriptor**. It is just an index into a per-process table of "things this process has open".

```
 Process's open-file table
 ┌────┬──────────────────────────────┐
 │ 0  │ stdin  (keyboard)            │
 │ 1  │ stdout (terminal)            │
 │ 2  │ stderr (terminal)            │
 │ 3  │ a socket  ← your first one   │
 │ 4  │ another socket / a file      │
 └────┴──────────────────────────────┘
```

Unix's big idea is "everything is a file". A **socket** is a file descriptor that has a network connection behind it instead of a disk. That is why the same `read()`, `write()` and `close()` work on files, terminals and sockets.

Each process can only have a limited number of fds open at once (see `ulimit -n` in section 5). Every socket uses one.

### 0.3 🧠 IP addresses and ports

- An **IP address** identifies a machine (more precisely, a network interface) on a network. IPv4 addresses look like `142.251.222.142`. `127.0.0.1` (also called `localhost`) always means "this same machine".
- A **port** is a 16-bit number (0 to 65535) that identifies **which program** on that machine should get the data.

**Analogy:** the IP address is the apartment building's street address; the port is the flat number. The postman (IP) gets the letter to the building; the port tells the building which flat (program) gets it.

A TCP connection is uniquely identified by **four numbers**, the **4-tuple**:

```
 (client IP, client port, server IP, server port)
 e.g. (127.0.0.1, 54321, 127.0.0.1, 2026)
```

That is why one server port (say 2026) can hold thousands of connections at once: each connection has a different client IP/port pair.

### 0.4 🧠 Client and server

- A **server** waits at a known address and port for others to contact it.
- A **client** starts the conversation by connecting to the server.

The server must be at a fixed, known place (like a shop with a fixed address). The client can be anywhere (like a customer who walks in from wherever).

### 0.5 🧠 TCP vs UDP, and what a packet is

Data travels across networks in small chunks called **packets** (often around 1500 bytes or less). Two main transport protocols use them:

| | **TCP** (Transmission Control Protocol) | **UDP** (User Datagram Protocol) |
|---|---|---|
| Connection? | Yes, set up with a handshake first | No, just send |
| Reliable? | Yes: lost packets are resent | No: lost is lost |
| Ordered? | Yes | No |
| Shape of data | A continuous **byte stream** | Separate **datagrams** (messages) |
| Used by | HTTP/1.1, HTTP/2, SSH, SMTP, databases | DNS, video calls, games |

**Key fact for the whole lecture:** TCP gives you a *stream of bytes*, like water through a pipe. It does **not** keep your "messages" separate. If you write `"hello"` then `"world"`, the other side might read `"hellowor"` then `"ld"`. Section 6 (framing) is all about fixing this.

### 0.6 🧠 The TCP handshake, FIN, RST and TIME_WAIT

TCP connections start with a **3-way handshake** and end with either a polite close (**FIN**) or an abort (**RST**).

```
  CLIENT                              SERVER
    │──────── SYN ────────────────────▶│  "I want to talk"       (SYN = synchronize)
    │◀─────── SYN + ACK ───────────────│  "OK, I want to too"     (ACK = acknowledge)
    │──────── ACK ────────────────────▶│  "Great"   → ESTABLISHED
    │                                  │
    │═════════ data both ways ════════▶│
    │                                  │
    │──────── FIN ────────────────────▶│  "I'm done sending" (polite close)
    │◀─────── ACK / FIN ───────────────│
    │──────── ACK ────────────────────▶│
                 ─── OR ───
    │──────── RST ────────────────────▶│  "ABORT" (reset): drop everything now
```

- **FIN** (finish) = hanging up the phone politely. Both sides finish cleanly.
- **RST** (reset) = slamming the door. Any unsent data is thrown away. The other side's next `read()` typically fails with `ECONNRESET` ("connection reset by peer").
- **TIME_WAIT**: after a polite close, the side that closed **first** keeps a record of the connection for a while (about 60 seconds on Linux). This stops delayed old packets from being confused with a new connection that reuses the same 4-tuple. TIME_WAIT causes two problems you will meet in this lecture: "address already in use" on server restart (fixed by `SO_REUSEADDR`) and clients running out of ephemeral ports.

### 0.7 🧠 Bytes, bits and hexadecimal

- A **bit** is 0 or 1. A **byte** is 8 bits, so it can hold 0 to 255.
- **Hexadecimal (hex)** writes numbers in base 16 using digits `0-9` and `A-F`. One hex digit = 4 bits (a **nibble**). So one byte is always exactly **two hex digits**: `0x00` to `0xFF`.

```
 decimal 234  =  binary 1110 1010  =  hex  E A  = 0xEA
                        ^^^^ ^^^^           ^ ^
                        E    A    (each nibble is one hex digit)
```

You will see hex constantly in this lecture: in byte order, in BCD, in tcpdump output.

### 0.8 🧠 Signals

A **signal** is a tiny notification the kernel sends to a process: "you pressed Ctrl-C", "your child process died", "you wrote to a broken pipe". Each signal has a **default action**, often "terminate the process". A program can choose to **catch** or **ignore** most signals; a few (like `SIGKILL`) cannot be caught.

### 0.9 🧠 Processes and fork

A **process** is a running program with its own memory and its own fd table. `fork()` is a syscall that **clones** the current process: after it, two nearly identical processes (parent and child) run the same code from the same point. Section 5 uses this to serve many clients.

### 0.10 🧠 Caches and DNS in one paragraph

A **cache** is a short-term memory of answers you already looked up, so you do not have to ask again. **DNS** (Domain Name System) is the internet's phone book: it turns names like `google.com` into IP addresses. Answers are cached for a period called the **TTL** (time to live). Section 3 covers this.

---

## 1. The Server

> Part 1 covers: a working echo server in 20 lines · ports, and why 1024 is a line in the sand · the accept queue · the signal that will kill your process.

### 1.1 A TCP server is seven system calls

**Plain English:** to run a server you ask the kernel for a socket, give it an address, tell it to start waiting, take each caller, read and write bytes, and hang up. That is it.

> "Everything else you have ever used (Express, Flask, net/http) is a wrapper around this sequence."

```
   1         2        3         4         5        6        7
 socket → bind → listen → accept → read → write → close
          └─────────────────────┘
          the four that only a server makes
```

The slide highlights **bind, listen, accept** as server-only calls (plus `socket`, which starts every server). A client uses `socket`, then `connect`, then `read`/`write`/`close`. It never calls bind/listen/accept.

| # | Call | Who uses it | What it does (plain English) |
|---|---|---|---|
| 1 | `socket()` | both | "Kernel, give me a new network endpoint." Returns an fd. |
| 2 | `bind()` | server | "Attach this socket to address X, port Y." |
| 3 | `listen()` | server | "Start accepting incoming connections, and queue them up." |
| 4 | `accept()` | server | "Give me the next connected client." Returns a **new** fd for that client. |
| 5 | `read()` | both | Receive bytes. |
| 6 | `write()` | both | Send bytes. |
| 7 | `close()` | both | Hang up and free the fd. |

**Analogy (a shop's phone line):**
`socket` = buy a phone · `bind` = get a phone number assigned · `listen` = plug it in and switch the ringer on · `accept` = pick up a ringing call · `read`/`write` = talk · `close` = hang up.

### 1.2 `echo_server.c`: send it anything, it sends it back, once

```c
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

#### Line-by-line, for a beginner

**The headers**

| Header | Why it is here |
|---|---|
| `<arpa/inet.h>` | Defines `struct sockaddr_in`, `htons`, `inet_addr`, `INADDR_ANY` (and pulls in the socket declarations). |
| `<unistd.h>` | The raw Unix calls: `read`, `write`, `close`, `sleep`, `fork`. |
| `<string.h>` | `memcpy`/`memset`, used for filling address structs. |

**Step 1: create the socket**

```c
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
```

- `AF_INET` = "Address Family: Internet", meaning IPv4. (🧠 `AF_INET6` would be IPv6.)
- `SOCK_STREAM` = a reliable byte stream, which means **TCP**. (🧠 `SOCK_DGRAM` would be UDP.)
- `0` = "pick the default protocol for this combination", which is TCP.
- Returns a file descriptor (likely `3`). At this point the socket has no address and is not connected to anything.

**Step 2: describe the address we want**

```c
struct sockaddr_in addr = {0};      // an IPv4 address+port struct, zeroed out
addr.sin_family = AF_INET;          // "this is an IPv4 address"
addr.sin_addr.s_addr = INADDR_ANY;  // IP = 0.0.0.0 = every interface on this machine
addr.sin_port = htons(2026);        // port 2026, converted to network byte order
```

- `= {0}` sets every byte to zero. 🧠 This matters because the struct has a padding field (`sin_zero`) that should be zero.
- `INADDR_ANY` (0.0.0.0) means "accept connections arriving on **any** of my IP addresses": localhost, Wi-Fi IP, VPN IP, all of them. (More in 5.6.)
- `htons(2026)` converts the port into the byte order the network expects. Section 3.3 explains why this is needed.

**Step 3: bind**

```c
bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
```

"Kernel, this socket now owns port 2026 on all interfaces." 🧠 The `(struct sockaddr*)` cast is C's old way of making one function accept many address types (IPv4, IPv6, Unix-domain sockets).

**Step 4: listen**

```c
listen(server_fd, 1);
```

Turns the socket into a **listening** (passive) socket. It will never send data itself; its job is only to produce new connections. The `1` is the **backlog**, the size of the queue of waiting connections (section 1.6).

**Step 5: the accept loop**

```c
while (1) {
    int client_fd = accept(server_fd, NULL, NULL);
```

- `accept()` **blocks** (waits, doing nothing) until a client has connected.
- It returns a **brand new fd** (`client_fd`) that represents the connection with that one client. The listening `server_fd` stays open for the next client.
- The two `NULL`s mean "I do not care who the client is". 🧠 You could pass a `struct sockaddr_in*` and a `socklen_t*` to learn the client's IP and port.

```
 server_fd (listening, port 2026) ──accept()──► client_fd #4  (client A)
                                  ──accept()──► client_fd #5  (client B) ...
```

**Step 6: read, write, close**

```c
    char buf[4096];                              // a 4 KB buffer
    int n = read(client_fd, buf, sizeof(buf));   // wait for some bytes
    write(client_fd, buf, n);                    // send the same n bytes back
    close(client_fd);                            // hang up on this client
}
```

- `read()` waits until **at least one byte** arrives, then returns how many bytes it put into `buf` (up to 4096). It does **not** wait for a "whole message"; TCP has no messages.
- 🧠 `read()` returns: **> 0** = number of bytes received; **0** = the other side closed the connection (end of file, EOF); **-1** = error.
- `write()` sends back exactly those `n` bytes. That is the entire "echo protocol".
- `close(client_fd)` closes the **client's** connection, not the server's. The loop then goes back to `accept()` for the next caller.

#### Why does this exist?
It is the smallest possible network service. Everything later in the lecture is built by fixing one of its limitations.

#### Pitfalls and gotchas
- 🧠 **No error checking.** Every one of these calls can fail (`bind` fails with `EADDRINUSE` if the port is taken; `accept` can fail with `EMFILE` if you are out of fds). The slides leave checks out for readability. Real code checks every return value.
- **It reads only once.** If the client sends two lines, only the first chunk is echoed and then the server hangs up. Fixed in 1.5.
- **One client at a time.** While it serves one client, others wait in the queue. Fixed in section 5.

### 1.3 Demo: compile it, talk to it

> "Two commands and you have a network service running on your laptop."

```bash
$ gcc -o echo_server echo_server.c      # compile
$ ./echo_server                          # 🧠 run it (in terminal 1)
$ nc localhost 2026                      # talk to it (in terminal 2)
hello                                    ← you type this
hello                                    ← server echoes it, then hangs up
```

- `gcc -o echo_server echo_server.c` compiles the C file into a program named `echo_server`. 🧠 No extra libraries are needed: socket functions live in the standard C library on Linux and macOS.
- `nc` (netcat) is a tiny tool that connects to a host and port and pipes your keyboard to the socket and the socket to your screen.
- 🧠 With this first version, after the first echo the server closes the connection, so `nc` exits (or stops echoing). That is not a crash; it is exactly what the code says.

### 1.4 Ports: everything below 1024 belongs to root

**Plain English:** low-numbered ports are "reserved seats". Only the administrator (root) may sit there.

> "Not a rule you cannot break, a rule about **who** is allowed to break it."

```
 < 1024   ROOT ONLY ("well-known / privileged ports")
          80 HTTP · 443 HTTPS · 25 SMTP · 21 FTP · 110 POP

 ≥ 1024   ANYONE
          3000 node · 8080 alt-http · 6379 redis · 2026 ours
```

| Port | Service | Meaning |
|---|---|---|
| 80 | HTTP | Web (plain text) |
| 443 | HTTPS | Web over TLS (encrypted) |
| 25 | SMTP | Simple Mail Transfer Protocol: sending email between servers |
| 21 | FTP | File Transfer Protocol |
| 110 | POP | Post Office Protocol (POP3): downloading email |
| 3000 | node | Common default for Node.js dev servers |
| 8080 | alt-http | The usual "alternative HTTP" port |
| 6379 | redis | Redis database default port |
| 2026 | ours | The lecture's echo server |

#### Why does it matter?
- To bind port 80 you must run as **root** (the all-powerful admin user).
- If you run as root, **a bug in your request parser is a root compromise**: an attacker who exploits your code gets full control of the machine.
- So real deployments **bind 8080 as an unprivileged user** and let a **front proxy** (which does almost nothing, so it has very little code that can have bugs) own ports 80 and 443 and forward traffic to 8080.

```
 Internet ──► :443 / :80  [ front proxy, e.g. nginx ]  ──► :8080 [ your app, normal user ]
               tiny, does almost nothing                    big, complex, buggy-ish
```

🧠 This is where Week 4 (nginx deep dive) picks up.

#### Pitfalls
- 🧠 On Linux you can also grant just the "bind low ports" power without full root (`CAP_NET_BIND_SERVICE`), or change the threshold with the sysctl `net.ipv4.ip_unprivileged_port_start`.
- 🧠 macOS (since 10.14 Mojave) lets ordinary users bind low ports on all interfaces, so you may not see this error on your Mac. It still applies on Linux servers.

### 1.5 Thought experiment: what breaks if you skip a call?

The lecturer asked two questions to argue about.

| You skipped | What the client sees | Why |
|---|---|---|
| **No `bind()`** (no one owns the port) | **Connection refused**, instantly | Nothing owns the port, so the kernel answers the client's SYN with an **RST** straight away. Fast, loud, easy to debug: the client knows instantly nobody is home. |
| **`bind()` (and listen) but no `accept()`** | **It just hangs** | The kernel completes the handshake **on your behalf**. The connection is established and sits in the queue. Your process never picks it up, so the client waits until TCP times out. Silent, slow, and **it looks exactly like an overloaded server**. |

```
 NO BIND                                BIND + LISTEN, NO ACCEPT
 client: SYN ─────────► kernel           client: SYN ──────────► kernel
 client: ◄──────── RST                   client: ◄──── SYN+ACK   (kernel does it for you)
 "Connection refused" (ms)               client: ACK ──────────► queued, ESTABLISHED
                                         ... waits ... nobody reads ... timeout
```

🧠 **Lesson:** a loud failure is better than a silent one. The hang looks like an overloaded server because an overloaded server *is* a process too busy to call `accept()` fast enough. (Small detail: if you call `listen()` without `bind()`, the kernel quietly picks a random port, so clients aiming at 2026 still get refused.)

### 1.6 `echo_server_persistent.c`: one `read()` is not a conversation

```c
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
        int n;
        while ((n = read(client_fd, buf, sizeof(buf))) > 0) {
            write(client_fd, buf, n);
        }
        close(client_fd);
    }
}
```

**What changed:** one inner loop. Instead of reading once, the server keeps reading and echoing until `read()` returns something that is not positive.

```c
while ((n = read(client_fd, buf, sizeof(buf))) > 0) { write(...); }
```

This is the classic C "read loop". Read it as: "read some bytes into n; if n > 0, echo and repeat".

| `read()` returned | Meaning | What the loop does |
|---|---|---|
| `n > 0` | got n bytes | echo them, read again |
| `n == 0` | **EOF**: the client closed (sent FIN) | stop, `close()` |
| `n < 0` | error (e.g. connection reset) | stop, `close()` |

Now `nc localhost 2026` echoes every line you type until you press Ctrl-C or Ctrl-D.

#### Pitfalls
- **Still one client at a time.** The server stays in the inner loop until this client leaves. A second `nc` will connect (the kernel does the handshake) but get no echo until the first one disconnects. That is the "debt" section 5 pays off.
- 🧠 Treating every negative return as fatal is sloppy. Some errors, like `EINTR` (interrupted by a signal), just mean "try again".

### 1.7 `listen(server_fd, 1)`: that second argument is a queue

> 💡 **Simple words mein:** Kernel ek receptionist hai. Caller ka greeting (handshake) khud kar deta hai aur waiting room mein bitha deta hai. `accept()` = agla token number bulao. `listen()` ka number = waiting room mein kitni kursiyan hain.

```
  client SYNs ──► [ kernel completes the 3-way handshake by itself ]
                                    │
                                    ▼
                     accept queue (backlog)
                   ┌─────┬─────┬─────┬─────┐
                   │ A ✓ │ B ✓ │ C ✓ │     │ ──► accept() takes one at a time
                   └─────┴─────┴─────┴─────┘
                     all ESTABLISHED already
```

The slide's three points:

1. **The kernel finishes the handshake without you.** By the time `accept()` returns, the TCP connection has already been established for a while. The client has been waiting in line.
2. **Full-queue behaviour is not portable** (it differs between OSes). When the queue is full, some systems answer "Connection refused"; others leave the client stuck at an indefinite `Trying...`. **Do not build logic on whichever one you saw locally.**
3. **We passed 1. Real servers pass hundreds.** And **Linux caps whatever you ask for at `net.core.somaxconn`**, so the number in your code may not be the number in effect.

🧠 Extra detail:
- Check the cap: Linux `sysctl net.core.somaxconn` (4096 by default on modern kernels, 128 on older ones); macOS `sysctl kern.ipc.somaxconn`.
- There are really two queues on Linux: the **SYN queue** (handshakes still in progress) and the **accept queue** (finished connections waiting for `accept()`). The backlog number controls the accept queue. Flooding the SYN queue with fake SYNs is a **SYN flood** attack.
- On Linux, `ss -tln` shows the accept queue for each listening socket (columns `Recv-Q` = currently queued, `Send-Q` = the backlog limit).

### 1.8 Signals: SIGPIPE is the one that will kill you

**Plain English:** if you write to a connection the other side has already thrown away, the kernel, by default, kills your process on the spot. No error message, no exception. Gone.

> 💡 **Simple words mein:** Aap phone pe bol rahe ho, par doosri taraf wala phone kaat chuka hai. Kernel ka default rule: "khaali line pe bola? Process khatam." Na error, na message.

| Number | Signal | When you see it | Can be caught? |
|---|---|---|---|
| 2 | `SIGINT` | You press **Ctrl-C** | yes |
| 9 | `SIGKILL` | `kill -9`; the "force kill" | **no, cannot be caught** |
| **13** | **`SIGPIPE`** | **write to a peer that already closed** | yes (and you should ignore it) |
| 15 | `SIGTERM` | `kill <pid>`; the **polite** kill ("please shut down") | yes |
| - | `SIGHUP` | your **SSH session disconnects** (the terminal "hung up") | yes |
| - | `SIGSTOP` / `SIGTSTP` | **Ctrl-Z** pauses a program (`SIGTSTP` from the keyboard; `SIGSTOP` cannot be caught) | TSTP yes, STOP no |

🧠 The name comes from Unix pipes (`ls | head`): when `head` exits, `ls` gets `SIGPIPE` the next time it writes, and quietly dies. That is great for shell pipelines and terrible for servers.

### 1.9 The SIGPIPE demo: `sigpipe_server.c` and `sigpipe_client.c`

#### `sigpipe_server.c`: "Two writes. The second one never runs."

```c
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(8080);
    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 1);

    int client_fd = accept(server_fd, NULL, NULL);
    char buf[4096];
    int n = read(client_fd, buf, sizeof(buf));

    sleep(1); // give the client time to force-close the connection (RST)

    write(client_fd, buf, n); // peer already reset -> SIGPIPE here
    write(client_fd, buf, n); // not reached: SIGPIPE's default action kills the process

    return 0;
}
```

Notes:
- Port **8080** this time, and **no `while(1)`**: it is a one-shot experiment. Accept one client, read once, wait, write.
- `sleep(1)` sets up the timing on purpose: it gives the client time to slam the connection shut (send RST) before the server writes.
- The writes then hit a dead connection. The slide's comments say the first write raises **SIGPIPE**, whose default action terminates the process, so the second write and `return 0` never run. No crash message, no stack trace.

🧠 **Precise behaviour varies by OS.** On Linux, the very first `write()` after an RST has arrived usually returns `-1` with `errno = ECONNRESET` (the kernel reports the stored reset error once), and it is the **next** write that raises `SIGPIPE`. That is one reason the demo has **two** writes: whichever write triggers it, the process is dead before `return 0`. In production the same thing shows up as "it worked once, then the process vanished". In a container it looks like a restart loop with an empty log.

#### `sigpipe_client.c`: "SO_LINGER {1, 0} sends RST instead of FIN."

```c
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

Line by line:
- `inet_addr("127.0.0.1")` turns the text "127.0.0.1" into a 32-bit address, already in network byte order. No DNS involved. 🧠 Modern code uses `inet_pton()` instead.
- `connect(...)` is the **client's** call: it performs the 3-way handshake with the server. 🧠 It also quietly assigns the client a local port (section 3.2).
- `write(fd, "hello", 5)` sends 5 bytes.
- `struct linger l = {1, 0}`: the two fields are `l_onoff = 1` ("linger is on") and `l_linger = 0` ("linger for zero seconds").
- `setsockopt(fd, SOL_SOCKET, SO_LINGER, ...)` applies that option to the socket.
- `close(fd)` now performs an **abortive close** (an abrupt close): instead of the polite FIN, the kernel sends an **RST** immediately.

🧠 The three linger settings:

```
 SO_LINGER off (default) → close() returns at once; kernel sends remaining data, then FIN
 SO_LINGER {1, 30}       → close() waits up to 30 s for data to be sent, then FIN
 SO_LINGER {1, 0}        → ABORT: throw away unsent data, send RST right now
```

#### Putting the demo together

```
 sigpipe_client                         sigpipe_server (port 8080)
   connect ─── SYN/SYN-ACK/ACK ──────►  accept()
   write "hello" ────────────────────►  read() → n = 5
   close() with linger{1,0} ── RST ──►  (sleeping 1 s)
   exits                                write()  → peer is gone
                                        write()  → SIGPIPE → process killed
                                        "return 0" never runs
```

🧠 Run it: start `./sigpipe_server`, then `./sigpipe_client`, then check the server's exit status with `echo $?`. `141` = 128 + 13, meaning "killed by signal 13 (SIGPIPE)".

#### 🧠 The fix every real server uses

```c
#include <signal.h>
#include <errno.h>

signal(SIGPIPE, SIG_IGN);           // once, at startup: ignore SIGPIPE

if (write(fd, buf, n) < 0) {
    if (errno == EPIPE || errno == ECONNRESET) {
        /* peer is gone: clean up this client, keep serving others */
    }
}
```

With SIGPIPE ignored, the bad `write()` simply returns `-1` with `errno == EPIPE`, which you can handle. (On Linux you can also use `send(fd, buf, n, MSG_NOSIGNAL)`; on macOS, `setsockopt(... SO_NOSIGPIPE ...)`.) Languages like Python and Go ignore SIGPIPE for you and raise an exception/error instead.

---

## 2. You Already Have a Client

> Part 2 covers: telnet, and not just to port 80 · `openssl s_client` for anything encrypted · netcat for everything else.

### 2.1 telnet is a client. Type the protocol yourself.

**Plain English:** if a protocol is plain text, you do not need special software. You can type the messages by hand.

> "If a protocol is text, your keyboard is a perfectly good implementation of it."

```bash
$ telnet google.com 80              # talk HTTP to Google
$ telnet localhost 2026             # talk to our echo server
$ telnet mail.example.com 25        # talk SMTP (email) to a mail server
```

🧠 What telnet does: `socket()` → `connect()` → copy what you type into the socket, copy what comes back onto your screen. It is just a generic client with a keyboard attached. To quit, press **Ctrl-]** (that is what `Escape character is '^]'` means), then type `quit`.

🧠 telnet is not installed by default on recent macOS. Use `brew install telnet`, or use `nc` which does the same job.

### 2.2 Demo: four lines of typing, and Google answers

```
jeet@jitendras-air cn-at-scaler % telnet google.com 80
Trying 142.251.222.142...
Connected to google.com.
Escape character is '^]'.
GET / HTTP/1.1
Host: google.com

HTTP/1.1 301 Moved Permanently
Location: http://www.google.com/
Content-Type: text/html; charset=UTF-8
Content-Security-Policy-Report-Only: object-src 'none';base-uri 'self';script-src 'nonce-...' 'strict-dynamic' ...
Date: Wed, 12 Aug 2026 16:06:35 GMT
Expires: Fri, 11 Sep 2026 16:06:35 GMT
Cache-Control: public, max-age=2592000
Server: gws
Content-Length: 219
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

#### Reading the output (🧠 annotations)

| Line | What it means |
|---|---|
| `Trying 142.251.222.142...` | telnet already did a **DNS lookup** (google.com → IP) and is now connecting. |
| `Connected to google.com.` | The TCP 3-way handshake finished. |
| `GET / HTTP/1.1` | **You typed this.** The *request line*: method `GET`, path `/`, version `HTTP/1.1`. |
| `Host: google.com` | **You typed this.** Required in HTTP/1.1: one IP can host many websites, so you must say which one. |
| *(blank line)* | **You typed Enter twice.** The blank line means "my headers are finished". Without it the server keeps waiting. |
| `HTTP/1.1 301 Moved Permanently` | The *status line*: status code 301 = "this moved". |
| `Location: http://www.google.com/` | Where it moved to. A browser would silently follow this; telnet shows you the raw truth. |
| `Cache-Control: public, max-age=2592000` | You may cache this answer for 2,592,000 seconds = 30 days. (That matches `Expires`, 30 days after `Date`.) |
| `Server: gws` | "Google Web Server". Headers reveal what the server runs. |
| `Content-Length: 219` | The body is exactly 219 bytes. This is how the client knows when the body ends (section 6.2). |
| `<HTML>...` | The body: a tiny HTML page saying "moved here". |

🧠 Notice HTTP already uses **two** framing methods in one response: a **delimiter** (the blank line) to end the headers, and a **length prefix** (`Content-Length`) to end the body. Keep this in mind for section 6.

### 2.3 Beyond port 80: most services do not speak HTTP

> "And once TLS is in the way, telnet is useless: you need something that can do the handshake for you."

```bash
$ openssl s_client -connect www.hotstar.com:443   # like telnet, but does TLS first
$ nc localhost 2026                               # netcat as a client
$ nc -l 9000                                      # netcat as a SERVER (listen on 9000)
```

| Tool | Use it for | 🧠 Notes |
|---|---|---|
| `telnet host port` | plain-text protocols (HTTP, SMTP, Redis) | no encryption |
| `openssl s_client -connect host:443` | anything encrypted with TLS (HTTPS, SMTPS...) | does the TLS handshake, prints the certificate chain, TLS version and cipher, then lets you type (e.g. `GET / HTTP/1.1` + `Host:` line) |
| `nc host port` | "everything else": a raw TCP client | works with binary data and pipes too: `echo hi \| nc localhost 2026` |
| `nc -l 9000` | being a server | lets you test a client before you have written a server. 🧠 Some older netcat versions need `nc -l -p 9000`. |

🧠 Why telnet fails on 443: TLS starts with a binary "ClientHello" message containing crypto parameters. You cannot type that by hand. `openssl s_client` does the TLS part and then gives you a plain-text session on top.

---

## 3. Writing the Client

> Part 3 covers: a client in C, and what is missing from it · ephemeral ports · byte order: htons and friends · DNS hiding inside a function call · where we are on the OSI stack.

### 3.1 `client.c`: the client is shorter. Notice what is missing.

```c
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>

int main() {
    struct hostent *he = gethostbyname("localhost");

    int fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    memcpy(&addr.sin_addr, he->h_addr_list[0], he->h_length);
    addr.sin_port = htons(8080);
    connect(fd, (struct sockaddr*)&addr, sizeof(addr));

    write(fd, "hello", 5);

    struct linger l = {1, 0}; // abortive close: sends RST instead of FIN
    setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
    close(fd);

    return 0;
}
```

#### Line by line

| Code | What it does |
|---|---|
| `#include <netdb.h>` | New header: **name resolution** (`gethostbyname`, `struct hostent`, `getaddrinfo`). |
| `gethostbyname("localhost")` | Turns a **name** into an **IP address** (this is DNS, section 3.4). Returns a pointer to a `struct hostent`. |
| `he->h_addr_list[0]` | A name can map to several IPs; this takes the **first** one. 🧠 Better clients try the others if the first fails. |
| `he->h_length` | How many bytes one address is: 4 for IPv4. |
| `memcpy(&addr.sin_addr, ...)` | Copies those raw address bytes into our struct. 🧠 They are already in network byte order, so no `htonl` is needed. The **port** still needs `htons`. |
| `connect(fd, ...)` | Does the 3-way handshake with the server. Blocks until connected or refused. |
| `write(fd, "hello", 5)` | Sends 5 bytes. |
| `SO_LINGER {1,0}` + `close` | Abortive close (RST), same as the SIGPIPE client. So this client also works as the SIGPIPE trigger. |

**What is missing?** There is **no `bind()`, no `listen()`, no `accept()`**, and no local port number anywhere in the code. Yet replies from the server still find their way back. How? Next section.

### 3.2 Ephemeral ports: clients never bind, the kernel picks a port for you

**Plain English:** a shop needs a fixed address so customers can find it. A customer does not; the shop just replies to wherever the customer came from. The kernel gives each outgoing connection a temporary ("ephemeral") port.

| SERVER | CLIENT |
|---|---|
| **Binds a known port.** It has to: clients need a fixed address to aim at. That is the entire reason well-known ports exist. | **Gets a random high port**, assigned by the kernel at `connect()` time from the **ephemeral range**, typically **32768–60999 on Linux**. |
| `bind()` → `listen()` → `accept()` | **Finite**, and **TIME_WAIT holds each one for ~60 s after close. Busy clients do run out.** |

```
 client 127.0.0.1:54321  ──────────────►  server 127.0.0.1:2026
        ^^^^^ picked by the kernel at connect()
 server replies to 127.0.0.1:54321  ◄───  (it knows from the 4-tuple)
```

#### Why can busy clients run out? (🧠 worked numbers)
Linux range 32768–60999 = about **28,000 ports**. If each closed connection keeps its port in TIME_WAIT for 60 s, one client talking to **one** server IP:port can open only about 28,000 / 60 ≈ **470 new connections per second** before it runs out. A load-testing tool or a service calling another service without connection reuse hits this quickly.

🧠 Symptoms: `connect()` fails with `EADDRNOTAVAIL` ("Cannot assign requested address"). Fixes: **reuse connections (pooling, section 5.5)**, spread across more destination IPs, or widen the range. Check the range with `cat /proc/sys/net/ipv4/ip_local_port_range` on Linux, or `sysctl net.inet.ip.portrange.first net.inet.ip.portrange.last` on macOS (49152–65535 by default).

### 3.3 `htons()`: your CPU and the network disagree on byte order

**Plain English:** a number bigger than one byte has to be stored as several bytes, and machines disagree on which byte goes first. The internet picked one order; your laptop uses the other. `htons()` swaps them.

> "htons = **h**ost **to** **n**etwork, **s**hort. It exists because x86 stores 2026 backwards from how the wire wants it."

```
 port 2026 = 0x07EA        (0x07 = 7, 0xEA = 234  →  7 × 256 + 234 = 2026)

 little endian (your x86 / ARM laptop)  :   EA 07    (least significant byte first)
 big endian    (network byte order)     :   07 EA    (most significant byte first)

 htons() swaps them.   htonl() for 4 bytes.   ntohs() / ntohl() on the way back.
```

| Function | Name | Size | Direction |
|---|---|---|---|
| `htons` | host to network short | 2 bytes (ports) | before sending / filling a struct |
| `htonl` | host to network long | 4 bytes (IPv4 addresses, lengths) | before sending |
| `ntohs` | network to host short | 2 bytes | after receiving |
| `ntohl` | network to host long | 4 bytes | after receiving |

> 💡 **Simple words mein:** Date jaisa hai: India 12/08/2026 likhta hai (din pehle), US 08/12/2026 (mahina pehle). Order pe agree nahi kiya toh galat date padhoge. Internet ne decide kiya: "bada byte pehle" (big endian).

🧠 **What happens if you forget `htons`?** On a little-endian laptop, `addr.sin_port = 2026` stores bytes `EA 07`. The network reads them big-endian as `0xEA07` = **59911**. Your server ends up listening on port 59911, and `nc localhost 2026` gets "Connection refused". Run `lsof -iTCP -sTCP:LISTEN` (macOS) or `ss -tlnp` (Linux) and you will see the wrong port.

🧠 On a big-endian machine `htons` does nothing, but always write it so your code works on every machine. Byte order also matters in section 6: a binary protocol that sends a 4-byte length must say in its spec which byte order it uses (almost always big-endian, "network order").

### 3.4 `gethostbyname()`: one function call is a whole protocol

**Plain English:** that one innocent-looking line asks the internet's phone book (DNS) for an address. It can take seconds, and your program freezes while it waits.

> "gethostbyname() is DNS: a UDP request, a recursive resolver, and a cache, hidden behind a synchronous call that can block for seconds."

The three steps from the slide:

1. **`/etc/hosts` first.** Checked before any network traffic. This is why editing hosts overrides everything, and why a stale entry there is so hard to debug.
2. **UDP query to your resolver.** One small datagram to whatever is in `/etc/resolv.conf`. It **recurses** (asks around) for you: **root → TLD → authoritative**.
3. **Cached for the TTL, and it blocks.** Fast when warm (cached), seconds when cold (not cached), with **no timeout you control**. Use `getaddrinfo()` in real code; **`gethostbyname` is IPv4-only and not thread-safe.**

```
 your program: gethostbyname("google.com")   ← program is frozen until this returns
      │
      ├─1─► /etc/hosts        "google.com listed here?"  (no network)
      │
      └─2─► resolver (from /etc/resolv.conf, e.g. your router or 8.8.8.8), via UDP port 53
                 │  (if not cached)
                 ├──► root server       "who handles .com?"
                 ├──► .com TLD server   "who handles google.com?"
                 └──► google's authoritative server  "142.251.222.142, TTL 300 s"
      ◄───────── answer, cached for the TTL ─3─
```

🧠 Terms:
- **Resolver** (recursive resolver): the server that does all the asking for you and caches results.
- **Root servers**: the top of the DNS tree; they know who runs each top-level domain.
- **TLD** (top-level domain): `.com`, `.org`, `.in`...
- **Authoritative server**: the server that holds the real records for a domain.
- **TTL** (time to live): how long an answer may be cached.

🧠 Why `getaddrinfo()` is better: it supports IPv4 **and** IPv6, is thread-safe, and returns a ready-to-use list of addresses you can loop through and try one by one. `gethostbyname` returns a pointer to one **shared static** buffer, so two threads calling it at once overwrite each other's results.

🧠 Why DNS uses UDP: the question and answer are tiny and retrying is cheap, so the connection setup of TCP would be wasted. (DNS falls back to TCP for large answers.) On macOS, `scutil --dns` shows the real resolver config (not `/etc/resolv.conf`); `/etc/hosts` is still checked first.

### 3.5 OSI: we have been writing layer 7 this whole time

**Plain English:** networking is split into layers, each doing one job and relying on the one below. Our tiny programs already touched three of them without naming them.

> "Ports are layer 4. IP addresses are layer 3. We touched all three in twenty lines and never named them."

| Layer | Name | In this lecture |
|---|---|---|
| 7 | Application | our echo protocol, HTTP |
| 4 | Transport | TCP · ports · `htons(2026)` |
| 3 | Network | IP · `INADDR_ANY` · `127.0.0.1` |
| 2 | Data link | Ethernet, Wi-Fi · MAC addresses |
| 1 | Physical | copper, fibre, radio |

(OSI = Open Systems Interconnection, a reference model. The slide skips layers 5 and 6.)

🧠 Layers 5 (Session) and 6 (Presentation) exist in the OSI model but have no clear separate home in the real internet. TLS and data encodings are sometimes placed there. The practical internet model (TCP/IP) has 4 layers: Link, Internet, Transport, Application. 🧠 **MAC address** (Media Access Control): the hardware address of your network card, used only on the local network (layer 2).

🧠 **Encapsulation**: each layer wraps the one above in its own header, like envelopes inside envelopes.

```
 [ Ethernet hdr [ IP hdr [ TCP hdr [ "hello" ] ] ] ]
    layer 2       layer 3   layer 4   layer 7 (your bytes)
                  has IPs   has ports
```

You will literally see these nested headers in tcpdump (section 5.7).

---

## 4. HTTP and curl

> Part 4 covers: an HTTP client in C is just the client you already have · a quick tour of the RFC · testing headers on purpose · curl as a teaching instrument.

### 4.1 An HTTP client is `client.c` with a string in it

> "Point it at port 80, write the request text, read until close. That is the whole protocol."

```
GET / HTTP/1.1
Host: google.com

   ↑ CRLF line endings, and that blank line is mandatory
```

Written as the actual bytes:

```
"GET / HTTP/1.1\r\nHost: google.com\r\n\r\n"
```

- **CRLF** = Carriage Return + Line Feed = the two bytes `\r\n` (hex `0D 0A`). HTTP's standard (its **RFC**, "Request for Comments", the documents that define internet protocols) says every line ends with CRLF, not just `\n`.
- **The blank line** (`\r\n\r\n` at the end) tells the server "headers are done". Without it, the server waits forever for more headers. You have built the "it just hangs" bug yourself.

🧠 A minimal HTTP client in C, changing only a few lines of `client.c`:

```c
// resolve "google.com" with gethostbyname, then:
addr.sin_port = htons(80);
connect(fd, (struct sockaddr*)&addr, sizeof(addr));

const char *req = "GET / HTTP/1.1\r\nHost: google.com\r\nConnection: close\r\n\r\n";
write(fd, req, strlen(req));

char buf[4096]; int n;
while ((n = read(fd, buf, sizeof(buf))) > 0)
    write(1, buf, n);           // fd 1 = stdout: print the response
close(fd);
```

🧠 "Read until close" only works if the server closes the connection. HTTP/1.1 keeps connections open by default (**keep-alive**), so either send `Connection: close` (as above) or parse `Content-Length` / chunked encoding to know where the response ends. That is homework 6.

#### Pitfalls
- Using `\n` instead of `\r\n`: forgiving servers accept it, strict ones do not.
- Forgetting the `Host:` header: HTTP/1.1 servers reply `400 Bad Request`.
- Forgetting the final blank line: the server waits; your client waits; nothing happens.

### 4.2 curl: the best HTTP teaching tool you have

> "You can learn a lot about headers and SSL debugging from one command with enough v's on it."

```bash
$ curl google.com                          # print the response body
$ curl -i google.com                       # include the response headers
$ curl -vv https://google.com 2>&1 | less  # verbose: show everything
```

| Command | What you see | 🧠 Why it is useful |
|---|---|---|
| `curl google.com` | Just the body: the small "301 Moved" HTML | shows that curl does **not** follow redirects by default |
| `curl -i google.com` | Status line + all headers + body | same as the telnet demo, without typing |
| `curl -vv https://google.com 2>&1 \| less` | DNS result, TCP connect, the TLS handshake, the certificate, request headers (`>`), response headers (`<`) | debugging headers and SSL. `2>&1` is needed because curl prints the verbose info to stderr; `less` lets you scroll |

🧠 What the `-v` output roughly looks like, mapped to this lecture:

```
* Trying 142.251.222.142:443...        ← DNS done, TCP connecting (sections 3.4, 1)
* Connected to google.com              ← handshake complete
* TLSv1.3 (OUT), TLS handshake, Client hello   ← TLS starting (section 5.8)
* Server certificate:
*  subject: CN=*.google.com            ← the identity in the certificate
*  issuer: ...                         ← the CA that vouches for it
> GET / HTTP/2                         ← '>' lines = what curl SENT
> Host: google.com
< HTTP/2 301                           ← '<' lines = what came BACK
< location: https://www.google.com/
```

🧠 "Testing headers on purpose" (a slide sub-topic): curl lets you send exactly the headers you want and watch how the server reacts.

```bash
curl -i -H 'Accept-Encoding: gzip' https://example.com --output -   # ask for compressed body
curl -i -H 'Host: other-site.com' http://1.2.3.4/                     # pick a virtual host by hand
curl -L google.com                                                    # follow redirects
curl --http1.1 -v https://google.com                                  # force text HTTP/1.1
curl -X POST -d 'a=1' https://httpbin.org/post                        # send a POST body
curl -o /dev/null -s -w '%{time_total}\n' https://google.com          # just measure time
```

---

## 5. Many Clients at Once (plus packets and SSL)

> Part 5 covers: `fork()` per connection · select and epoll · connection pools and ulimit · seeing the actual bytes: tcpdump, Wireshark, strace · SSL.

### 5.1 The debt comes due: our server serves exactly one client

> "When I wrote it I said it would support one at a time. Here are the two ways out, and both are still shipping in production today."

| ONE PROCESS PER CONNECTION: `fork()` | ONE PROCESS, MANY SOCKETS: `select` / `epoll` |
|---|---|
| **Dead simple**, and the OS does the scheduling for you. | Ask the kernel **which sockets are ready**, then handle only those. |
| Each child has its own memory, so **a crash is contained**. | **No context switching, no per-client memory.** |
| Costs a process per client. **Fine for hundreds, hopeless at ten thousand.** | **Harder to write**, and **one slow handler blocks everything**. |
| | **This is what nginx and Node do.** |

> 💡 **Simple words mein:** fork = har table ka alag waiter: easy, ek waiter behosh ho toh baaki tables safe, par 10,000 tables = 10,000 waiters. select/epoll = ek tez waiter + buzzer: sirf buzz karne wali table pe jaata hai. Par wo ek table pe 10 minute ruka toh sab wait karenge.

🧠 "Context switching" = the CPU stopping one process and starting another, which costs time. "Ten thousand" refers to the famous **C10K problem** (handling 10,000 concurrent connections), which is why epoll was invented. Threads (one thread per connection) are a middle ground not covered on the slides.

### 5.2 `fork()` returns twice. Read the return value.

> "It is a bit tricky: the same line of code runs in two processes, and the only way to tell which one you are is what came back."

```
                 fork()
                   │
      ┌────────────┴────────────┐
      ▼                         ▼
   PARENT                     CHILD
   fork() returned            fork() returned 0
   the child's PID (> 0)
```

| Return value | Who you are | What to do |
|---|---|---|
| `0` | **the child** | Handle this one client, then exit. |
| `> 0` | **the parent** (the number is the child's PID, process ID) | Close your copy of the fd and go back to `accept()`. |
| `-1` | fork **failed** (out of processes or memory) | Handle the error. "Almost nobody checks for this." |

### 5.3 `fork_server.c`: accept, fork, and let the child do the talking

```c
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(2026);
    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 1);

    while (1) {
        int client_fd = accept(server_fd, NULL, NULL);

        if (fork() == 0) {
            char buf[4096];
            int n;
            while ((n = read(client_fd, buf, sizeof(buf))) > 0) {
                write(client_fd, buf, n);
            }
            close(client_fd);
            return 0;
        }

        close(client_fd);
    }
}
```

#### How it works

```
 parent: accept() ──► client_fd for client A
         fork() ──────────────┐
         close(client_fd)     │  child A: echo loop with client A, then exit
         accept() ──► client B│
         fork() ─────────────────┐
         close(client_fd)        │  child B: echo loop with client B, then exit
         accept() ...            │
```

- Everything up to `accept()` is the same as the persistent server.
- After `accept()`, the parent **forks**. 🧠 The child gets a copy of the parent's memory **and of its fd table**, so both processes now hold `client_fd`, pointing to the **same** connection.
- **Inside `if (fork() == 0)`** is the child. It runs the familiar echo loop, closes the connection and **`return 0`** (exits).
- **The parent** skips the `if` block, closes **its** copy of `client_fd`, and loops straight back to `accept()` for the next client.

#### Pitfalls (the two lines that really matter)
- **The child's `return 0`.** Without it, the child would fall out of the `if` and continue the `while (1)` loop, calling `accept()` itself. Now you have two processes accepting, then four, and so on. 🧠 `exit(0)` is the more usual spelling.
- **The parent's `close(client_fd)`.** 🧠 The kernel only really closes a connection when **every** copy of the fd is closed. If the parent forgets, (1) the connection never fully closes, so the client never sees the hang-up, and (2) the parent leaks one fd per client until it hits the fd limit (`EMFILE`, "too many open files").
- **`fork()` can return -1** and the slide code does not check it.
- 🧠 **Zombies.** When a child exits, it stays as a "zombie" (shown as `<defunct>` in `ps`) until the parent collects its exit status with `wait()`/`waitpid()`. This server never does, so zombies pile up. A one-line fix: `signal(SIGCHLD, SIG_IGN);` at startup (on Linux and macOS this tells the kernel to reap children automatically).
- 🧠 `listen(server_fd, 1)` is still a tiny backlog. Since the parent calls `accept()` again right away, that is usually fine here, but real servers use a larger number.

### 5.4 `select` / `epoll`: one process, ask the kernel who is ready

> "Instead of blocking on one socket, hand the kernel your whole set and let it tell you which ones have data."

| SELECT | EPOLL |
|---|---|
| **Portable, and O(n)** | **Linux, and O(ready)** |
| Rebuild the fd set on every call; the kernel walks all of them every time. | Register interest once. The kernel returns only the descriptors that actually changed. |
| `FD_SETSIZE` caps you around **1024** descriptors. Works everywhere, which is why it is still around. | **kqueue** on BSD and macOS, **IOCP** on Windows. This is what nginx and Node are built on. |

🧠 Decoding:
- **O(n)** means the work grows with the **total** number of connections. With 10,000 connections and 3 active, select still checks all 10,000.
- **O(ready)** means the work grows only with the number of connections that **actually have something to do**: 3.
- **IOCP** = I/O Completion Ports, Windows' equivalent. **kqueue** is the BSD/macOS one (so on your Mac, Node uses kqueue).

🧠 The shape of an event loop (simplified, error handling omitted):

```c
// --- select version ---
while (1) {
    fd_set set;
    FD_ZERO(&set);
    FD_SET(server_fd, &set);                       // watch the listening socket
    for (each client fd c) FD_SET(c, &set);        // rebuild EVERY time  → O(n)
    select(max_fd + 1, &set, NULL, NULL, NULL);    // sleep until something is ready
    if (FD_ISSET(server_fd, &set)) add_client(accept(server_fd, NULL, NULL));
    for (each client fd c)                         // check everyone      → O(n)
        if (FD_ISSET(c, &set)) echo_once(c);       // read once, write once
}

// --- epoll version (Linux) ---
int ep = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = server_fd };
epoll_ctl(ep, EPOLL_CTL_ADD, server_fd, &ev);      // register ONCE
while (1) {
    struct epoll_event ready[64];
    int k = epoll_wait(ep, ready, 64, -1);         // returns only ready fds
    for (int i = 0; i < k; i++) handle(ready[i].data.fd);   // O(ready)
}
```

🧠 This loop is the **event loop**. When you write `async`/`await` in JavaScript or Python, this is what runs underneath. It also explains "one slow handler blocks everything": if `handle()` does a 2-second computation, no other client is served for 2 seconds.

### 5.5 The client side: Chrome would only open six connections to you

> "Connection limits are not a browser quirk; they are the client-side version of the same resource problem."

| | Meaning |
|---|---|
| **6** | Chrome's old **per-host connection cap**. If a page needed 10 files from the same server, requests **7–10 simply queued** until one of the first 6 finished. |
| **`ulimit -n`** | Your **hard ceiling on open file descriptors**. **Sockets count.** Often just **1024**. |
| **pool** | **Connection pool**: open once, reuse many times. You skip the **TCP and TLS handshake** every time after the first. |

🧠 More detail:
- `ulimit -n` shows your current limit; `ulimit -n 65535` raises it for this shell (up to the hard limit). A server with 1024 as its limit cannot hold more than about 1020 clients (stdin/stdout/stderr and the listening socket use some).
- Why pools help: a new TCP connection costs one round trip (the handshake) and TLS costs one or two more before any data flows. On a 100 ms link, that is 200–300 ms saved per request. Every database driver and HTTP library has a pool for this reason. HTTP's `keep-alive` is the same idea.
- Pools also save ephemeral ports (section 3.2).
- HTTP/2 (Week 5) sends many requests over **one** connection, which makes the 6-connection limit far less painful.

### 5.6 Socket options: a socket can accept connections on any IP

> "Read the man page for socket options once. Some of what is in there is a networking history lesson."

**◆ `INADDR_ANY`: listen on every interface.**
Handy in development; in production, choose it on purpose. Bind to a specific address when you mean one.

🧠 Example: a machine has `127.0.0.1` (loopback), `192.168.1.20` (Wi-Fi) and `10.8.0.2` (VPN). `INADDR_ANY` (0.0.0.0) accepts connections on all three. Binding to `127.0.0.1` makes the service reachable **only from the same machine**, which is free security for admin tools and databases. Accidentally binding a database to 0.0.0.0 on a public server is a classic way data gets leaked.

**◆ Promiscuous mode: see traffic that is not yours.**
On a **hub**, every frame reaches every port. This is how packet sniffers work, and why plaintext protocols died.

🧠 Normally a network card ignores frames (layer-2 packets) addressed to other MAC addresses. In promiscuous mode it keeps all of them. An old **hub** repeated every frame to every connected computer, so anyone could read everyone's passwords sent over telnet, FTP or plain HTTP. Modern **switches** send frames only to the right port, which makes sniffing harder but not impossible. This history is why we now encrypt everything (TLS, SSH).

**◆ `SO_REUSEADDR`: the one you will actually use.**
Without it, restarting your server fails with **"address already in use"** while the old socket sits in **TIME_WAIT**.

```c
int yes = 1;
setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));   // 🧠 BEFORE bind()
bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
```

🧠 You will hit this with the echo server: stop it with Ctrl-C while a client was connected, restart it at once, and `bind()` fails (silently, because the slide code does not check). With `SO_REUSEADDR` set before `bind()`, the restart works. Not the same as `SO_REUSEPORT`, which lets several live processes share one port.

### 5.7 tcpdump / Wireshark: stop guessing, look at the bytes

> "Capture it once, then read it as ASCII and as hex. Everything we have talked about is visible in there."

```bash
$ sudo tcpdump -ni any port 2026 -w 1.pcap     # capture traffic on port 2026 to a file
$ tcpdump -r local_capture.pcap -A             # read a capture, show payload as ASCII
$ tcpdump -r local_capture.pcap -X             # read a capture, show hex + ASCII
```

| Flag | Meaning |
|---|---|
| `sudo` | capturing packets needs admin rights |
| `-n` | do not turn IPs/ports into names (faster, and no extra DNS traffic in your capture) |
| `-i any` | capture on every interface (🧠 on macOS use `-i lo0` for localhost traffic) |
| `port 2026` | a **filter**: only packets to/from port 2026 |
| `-w 1.pcap` | write raw packets to a **pcap** file (packet capture format) |
| `-r file.pcap` | read a saved capture |
| `-A` | print each packet's payload as ASCII text (good for HTTP, echo) |
| `-X` | print hex and ASCII side by side (needed for binary protocols) |

**Wireshark** is a graphical tool that opens the same `.pcap` files and decodes every layer for you, click by click.

🧠 What you should see for one `nc` echo session:

```
IP 127.0.0.1.54321 > 127.0.0.1.2026: Flags [S]     ← SYN
IP 127.0.0.1.2026 > 127.0.0.1.54321: Flags [S.]    ← SYN-ACK  ('.' = ACK)
IP 127.0.0.1.54321 > 127.0.0.1.2026: Flags [.]     ← ACK: connected
IP 127.0.0.1.54321 > 127.0.0.1.2026: Flags [P.], length 6   ← "hello\n" (P = push data)
IP 127.0.0.1.2026 > 127.0.0.1.54321: Flags [P.], length 6   ← echoed back
IP ... Flags [F.]                                  ← FIN (polite close)
   or  Flags [R]                                   ← RST (abortive close, SO_LINGER {1,0})
```

With `-X` the payload looks like `6865 6c6c 6f0a  hello.`: `68 65 6c 6c 6f` is "hello" and `0a` is the newline.

🧠 **strace** (named on the section's opening slide): instead of the packets, it shows the **system calls** your program makes. `strace -f ./echo_server` prints lines like `accept(3, NULL, NULL) = 4`, `read(4, "hello\n", 4096) = 6`, `write(4, "hello\n", 6) = 6`. tcpdump shows the wire; strace shows your process. (Linux only; on macOS the rough equivalent is `dtruss`, which needs extra permissions.)

### 5.8 SSL / TLS: SSL secures TCP, not HTTP

**Plain English:** TLS is an encrypted tunnel built on top of a TCP connection. Whatever you send through the tunnel (HTTP, email, database queries) is protected. HTTP itself does not change.

> "It sits under the application protocol, which is why the same machinery protects SSH, SMTP, database connections and anything else you put on a socket."

🧠 **SSL** (Secure Sockets Layer) is the old name; the modern protocol is **TLS** (Transport Layer Security). People and tools (like `openssl`) still say SSL. (SSH actually uses its own protocol with the same ideas rather than TLS itself, but the principle on the slide holds: encryption sits between TCP and the application.)

```
 Without TLS:        HTTP  ──────────────► TCP ──► IP
 With TLS (HTTPS):   HTTP  ──► TLS ──────► TCP ──► IP
                            └ encrypts everything above it
```

The slide's three points:

1. **Exchange keys and certificates.** Use **asymmetric** crypto to agree on a shared secret, then **symmetric** crypto for the data, because symmetric is much faster.
2. **The certificate binds a key to an identity.** And the **CA chain** is the reason you believe it. "This is the part that expires at 3am and takes your site down."
3. **Pinning removes the CA from the trust equation.** Common in mobile apps: trust *this exact certificate*, not whatever any CA in the world is willing to sign.

🧠 Explaining the terms:
- **Symmetric crypto**: one secret key both encrypts and decrypts (e.g. AES). Very fast, but both sides need the same key, and you cannot just send it in the clear.
- **Asymmetric crypto**: a key pair, public and private. What one key locks, only the other unlocks (e.g. RSA, elliptic curves). Slow, but solves "how do two strangers agree on a secret over a public wire".
- So TLS uses the slow asymmetric part only at the start, to agree on a symmetric key, then switches to the fast symmetric part for all the data.
- **Certificate**: a signed document that says "this public key belongs to `google.com`", signed by a **CA** (Certificate Authority).
- **CA chain**: your OS/browser ships with a list of trusted root CAs. A root CA signs an intermediate CA, which signs the site's certificate. Your client checks each signature up the chain. Certificates have **expiry dates**; forget to renew and every client rejects your site (the "3am" outage).
- **Pinning**: the app ships with the expected certificate (or its key) built in and refuses anything else, even a certificate signed by a valid CA. This protects against a misbehaving CA but means the app must be updated when the certificate changes.

🧠 Simplified handshake:

```
 CLIENT                                          SERVER
   │── ClientHello (TLS versions, ciphers, site name) ──►│
   │◄─ ServerHello + certificate chain + key share ──────│
   │   verify certificate chain against trusted CAs       │
   │   both compute the same shared secret                │
   │══════ encrypted application data (symmetric) ═══════│
```

See it yourself: `openssl s_client -connect www.hotstar.com:443` prints the certificate chain (`Certificate chain`, `s:` = subject, `i:` = issuer), the protocol version and the cipher.

---

## 6. Protocol Design

> Part 6 covers: message framing (fixed, delimited, length-prefixed) · binary encodings: BCD, ASN.1, TLV · base64 and hex for debugging · RPC, gRPC and protobuf.

### 6.1 Text won. Now protocols are going binary again.

> "HTTP proves how far human-readable can take you, and HTTP/2 is binary."

| TEXT | BINARY |
|---|---|
| **You can read it.** telnet into it, tcpdump it, eyeball the logs. That is a real and large advantage; it is most of why HTTP won. | **The machine can read it faster.** Compact, clear, cheap to parse. No whitespace rules, no case-sensitivity arguments. |
| Verbose (wordy), ambiguous (unclear) at the edges, and slower to parse than it looks. | You need tooling to see anything at all, which is why **hex and base64 fluency stops being optional**. |

🧠 Example: sending the number 2026.
- As text: the 4 characters `"2026"` (4 bytes), and the parser must convert characters to a number, handle spaces, signs, leading zeros...
- As binary: 2 bytes `07 EA`. Read two bytes, done. But in a terminal it shows as garbage.

🧠 "Ambiguous at the edges" for text: Is `Content-length` the same as `Content-Length`? (Yes in HTTP, header names are case-insensitive.) Is a trailing space allowed? `\n` or `\r\n`? Each such question is a source of bugs and even security holes.

### 6.2 Message framing: how does the receiver know the message ended?

> "TCP gives you a byte stream, not messages. This is the single most important decision in your protocol, and there are only three answers."

🧠 Why: the sender does `write("hello")` and then `write("world")`. TCP only promises the bytes arrive **in order**. The receiver's `read()` calls might return:

```
 "helloworld"           (both in one read)
 "hel" + "loworld"      (split anywhere)
 "hello" + "world"      (looks right, by luck)
```

> 💡 **Simple words mein:** TCP paani ki pipe hai, parcels ka dabba nahi. Receiver ko batana padega message kahan khatam hua: fixed size dabba, end pe "khatam" ka sticker (delimiter), ya pehle hi bol do "17 bytes aa rahe hain" (length prefix).

So the protocol itself must mark where each message ends. The three options:

#### Option 1: Fixed length: every message is exactly N bytes

```
 N = 8
 [h e l l o _ _ _][w o r l d _ _ _]     (_ = padding)
```

- **Pros:** trivial to parse, zero ambiguity. Read exactly N bytes, that is one message.
- **Cons:** you pay for it in **padding** (wasted bytes), and **you can never change your mind about N**: every client and server must agree forever.

#### Option 2: Delimited: read until the sentinel

```
 hello\n world\n          (sentinel = '\n')
 HTTP headers end at \r\n\r\n
```

- **Example from the slide:** HTTP's blank line between headers and body.
- **Pros:** simple. 🧠 Human-readable, and the sender does not need to know the size in advance.
- **Cons:** simple **until the delimiter appears inside your data** and you need **escaping** (marking it as plain data). 🧠 (Like a CSV file where a value contains a comma; now you need quotes, and quotes inside quotes...) The receiver also has to scan every byte looking for the delimiter.

#### Option 3: Length prefix: send the size, then the bytes

```
 [00 05][h e l l o][00 05][w o r l d]
  ^^^^^ 2-byte length (big-endian!)
```

- **Example from the slide:** `Content-Length: 219`. Read exactly that many bytes and stop.
- **Pros:** reliable, **binary-safe** (the data can contain any byte, including what would have been a delimiter), and **what most modern protocols do**.
- 🧠 **Cons:** the sender must know the length before sending. (HTTP's **chunked encoding** solves this by sending the body as several length-prefixed chunks.) Also, never trust a received length blindly: a malicious "length = 4 GB" can make a naive server try to allocate 4 GB. Cap it.

🧠 Reading a length-prefixed message correctly (note: one `read()` may return fewer bytes than asked for, so you loop):

```c
// read exactly 'len' bytes into buf
int read_exact(int fd, char *buf, int len) {
    int got = 0;
    while (got < len) {
        int n = read(fd, buf + got, len - got);
        if (n <= 0) return -1;        // closed or error before message complete
        got += n;
    }
    return got;
}

unsigned char hdr[2];
read_exact(fd, (char*)hdr, 2);
int len = (hdr[0] << 8) | hdr[1];     // big-endian 2-byte length
read_exact(fd, buf, len);             // the message body
```

| | Fixed | Delimited | Length prefix |
|---|---|---|---|
| Parsing | trivial | scan for sentinel | read length, then read exactly that |
| Binary-safe? | yes | no (needs escaping) | yes |
| Flexible size? | no | yes | yes |
| Real examples | 🧠 old mainframe/bank records | HTTP headers, 🧠 SMTP, Redis command lines | `Content-Length`, 🧠 HTTP/2 frames, TLS records, gRPC |

### 6.3 BCD: binary coded decimal, four bits per digit

**Plain English:** store each decimal digit (0–9) in its own 4 bits. Because 4 bits is exactly one hex digit, the hex dump of a BCD number shows the decimal digits directly.

> "The curious part is that the digits just show up. Convert to hex and the representation does not change."

```
 decimal     9      8      7      6      5
 nibbles   1001   1000   0111   0110   0101
 hex          98          76          5F
```

🧠 Step by step: two digits fit in one byte. `9,8` → byte `1001 1000` = `0x98`. `7,6` → `0x76`. `5` is left alone, so it is paired with the filler nibble `1111` (`F`), giving `0x5F`. Five digits, three bytes.

The slide lists its properties:
- **Half the size of ASCII.** 🧠 ASCII stores each digit as a full byte (`'9'` = `0x39`); BCD uses half a byte.
- **No conversion step.** 🧠 The digits are stored as digits; displaying them needs no binary-to-decimal math.
- **Exact decimal arithmetic.** 🧠 No floating-point rounding errors (in float, 0.1 + 0.2 is not exactly 0.3). That matters for money.
- **Still used in SIM cards, SMS and card transactions.** 🧠 For example, phone numbers on SIM cards and amounts in card payment messages.

🧠 Pitfall: BCD wastes some space compared to pure binary (a byte can hold 0–255 in binary but only 0–99 in BCD). It is chosen for exactness and readability, not maximum compactness.

### 6.4 ASN.1: write the grammar, generate the codec

**Plain English:** instead of writing the code that converts your data to bytes and back by hand, you write a description (a **schema**) of what your data looks like, and a tool generates the encoder and decoder for you.

> "The most powerful system I have seen: you describe the shape of your data once and the encoder and decoder are produced for you."

(ASN.1 = Abstract Syntax Notation One. A **codec** = coder + decoder.)

| schema | codegen | 1984 |
|---|---|---|
| Describe your data once, in a **language-neutral grammar**. | Encoder and decoder are **generated**. You never hand-write parsing. | And **still under X.509, LDAP, SNMP and the whole telecom stack today**. |

🧠 Those names: **X.509** = the format of TLS certificates (so every HTTPS certificate is ASN.1 data). **LDAP** = Lightweight Directory Access Protocol (company user directories). **SNMP** = Simple Network Management Protocol (monitoring routers and switches).

#### The ASN.1 definition from the slide: "This is the whole schema. Everything else is generated."

```asn1
MyModule DEFINITIONS ::= BEGIN
    Item ::= SEQUENCE {
        itemCode    INTEGER (1..99999),
        color       VisibleString ("Black" | "Blue" | "Brown"),
        isTaxable   BOOLEAN
    }
END
```

| Part | Meaning |
|---|---|
| `MyModule DEFINITIONS ::= BEGIN ... END` | A module: a named group of type definitions. |
| `Item ::= SEQUENCE { ... }` | Define a type `Item` that is a **SEQUENCE**, an ordered record (like a C struct). |
| `itemCode INTEGER (1..99999)` | An integer that **must** be between 1 and 99999. 🧠 The generated decoder rejects 0 or 100000 for you. |
| `color VisibleString ("Black" \| "Blue" \| "Brown")` | A printable string restricted to one of three values. |
| `isTaxable BOOLEAN` | true or false. |

🧠 The schema says **what** the data is, not **how** it is laid out in bytes. Separate **encoding rules** decide the bytes: BER/DER (which are TLV-based, see next), PER (very compact), and others. Example in DER: the BOOLEAN `true` is encoded as `01 01 FF` = Type `01` (BOOLEAN), Length `01`, Value `FF`. That is TLV.

🧠 Why it matters: two teams writing in different languages (C on a phone switch, Java on a server) compile the same schema and are guaranteed to agree on the bytes.

### 6.5 TLV: ASN.1 you can write on a napkin

**Plain English:** every field is sent as three parts: what kind of field it is (**Type**), how many bytes its value takes (**Length**), and the bytes themselves (**Value**). Then the next field, and so on.

> "Type, length, value, repeated. That is the entire specification, and it is enough to build a real protocol."

```
 ┌──────┬────────┬─────────────┐┌──────┬────────┬─────────────┐
 │ Type │ Length │ Value       ││ Type │ Length │ Value       │ ...
 └──────┴────────┴─────────────┘└──────┴────────┴─────────────┘
 <Type><Length><Body/Value><Type><Length><Body/Value>...
```

🧠 A concrete example. Say type `0x01` = name, `0x02` = age:

```
 01 05 61 6c 69 63 65     02 01 15
 │  │  └─ "alice" ─┘      │  │  └ 21
 │  └ length 5            │  └ length 1
 └ type 1 (name)          └ type 2 (age)
```

**The key property: self-describing.**

> "A parser that has never seen this field type can still skip it, because the length tells it how far to jump. That one property is what makes TLV **forward compatible**, and it is why **TLS extensions, ISO 8583 and protobuf** all work this way."

```
 An OLD parser that only knows types 1 and 2 receives a NEW message:

 [01][05][alice]   [07][03][x y z]   [02][01][21]
   known: parse     unknown type 7:   known: parse
                    skip 3 bytes
                    (no crash, no upgrade needed)
```

🧠 **Forward compatible** = old software keeps working with data from newer software, so you can upgrade servers and clients at different times.

> 💡 **Simple words mein:** Courier parcel pe label hai "type: books, size: 3 kg". Naya type aaya jo purana staff nahi jaanta? Koi baat nahi, label dekh ke parcel side mein rakho aur agla uthao. Kuch toota nahi.

🧠 Names: **TLS extensions** = optional features in the TLS handshake (e.g. the site name, SNI), each sent as TLV. **ISO 8583** = the international standard for card payment messages between banks and card networks (it also uses BCD). **protobuf** = Google's Protocol Buffers (section 6.8).

🧠 TLV is option 3 (length prefix) applied to each field. And because a Value can itself contain more TLVs, you can build nested structures (trees). Pitfalls (🧠): agree on how big Type and Length are (1 byte? 2? 4?) and their byte order; validate that a Length does not run past the end of the message.

### 6.6 base64 / hex: how you see a binary protocol

> "Not fully related to framing, but base64 and hex are what you will end up using to debug any of this."

| HEX: for **reading** | BASE64: for **transport** |
|---|---|
| **Two characters per byte**, and `tcpdump -X` puts it beside the ASCII. | **Survives text-only channels**: Basic auth, JWTs, email attachments, data URIs. |
| Fluency here means **spotting a length prefix or a repeated tag by eye**. | **Costs 33% more bytes: 3 bytes become 4 characters.** A wrapper, not a design. |

🧠 Terms: **Basic auth** = HTTP `Authorization: Basic <base64 of user:password>`. **JWT** (JSON Web Token) = a login token made of base64url-encoded parts. **Data URI** = embedding a file inside HTML/CSS as `data:image/png;base64,...`.

#### 🧠 How hex works
Each byte becomes two hex characters. `"Hi"` = bytes `0x48 0x69` → `"4869"`. Size doubles.

#### 🧠 How base64 works (you need this for homework 4)
Take 3 bytes = 24 bits, cut them into 4 groups of 6 bits, and map each 6-bit value (0–63) to a character from `A–Z a–z 0–9 + /`.

```
 text        M          a          n
 bytes     0x4D       0x61       0x6E
 bits    01001101   01100001   01101110
 regroup 010011 010110 000101 101110      (4 groups of 6 bits)
 values    19     22      5     46
 chars     T      W       F     u         → "TWFu"
```

- Index table: 0–25 = `A–Z`, 26–51 = `a–z`, 52–61 = `0–9`, 62 = `+`, 63 = `/`.
- If the input is not a multiple of 3 bytes, pad with `=`: `"hi"` → `aGk=`, `"h"` → `aA==`.
- 3 bytes → 4 characters is where the **33%** overhead comes from.

Pitfalls (🧠):
- **base64 is not encryption.** Anyone can decode it (`echo aGk= | base64 -d`). Basic auth is only safe because TLS is underneath.
- **base64url** replaces `+` and `/` with `-` and `_` so it is safe in URLs (used by JWTs).

🧠 Eye-training examples: `48 54 54 50` = `HTTP`. A message starting `00 00 00 0C` likely begins with a 4-byte big-endian length of 12. A file starting `30 82` is usually an ASN.1 DER SEQUENCE, i.e. you are probably looking at a certificate.

### 6.7 RPC: framing plus a function name

**Plain English:** RPC (Remote Procedure Call) makes calling a function on another machine look like calling a local function. Under the hood it is just: send the function name and arguments as bytes, get the result back as bytes.

> "Send the name, send the arguments, get a response. They call it **marshalling** and **demarshalling**."

1. **Marshal: objects become bytes.** Function name, then the arguments, encoded however your protocol says. This part is the easy part.
2. **Frame it and send it.** Everything from the last four slides (the framing decision, the encoding decision) is now doing real work for you.
3. **The network is not a function call.** Calls fail halfway, time out with the work possibly done, and get retried. Every RPC framework rediscovers this the hard way.

```
 client: result = add(2, 3)
   marshal  →  [TLV: fn="add"][TLV: arg=2][TLV: arg=3]
   frame    →  [len][ ...those bytes... ]
   send     →  ─────────── network ───────────►  server: demarshal, run add(2,3)
   receive  ◄───────────  [len][TLV: result=5]   server: marshal result
   demarshal → 5
```

🧠 Why point 3 is the big one: a local function either runs or does not. A remote call has a **third** outcome: **you do not know**. If the request times out, maybe the server never got it, or maybe it did the work and the reply was lost. If you retry "charge the card ₹500", you might charge twice. Hence real systems use **idempotent** operations (safe to repeat) or idempotency keys, timeouts, and careful retries. (Week 8, "Building for failures", returns to this.)

> 💡 **Simple words mein:** UPI se ₹500 bheje, screen pe "timeout". Paise gaye ya nahi? Pata nahi. Dobara bheja toh shayad ₹1000 kat gaye. Remote call ka teesra answer hota hai: "pata nahi".

### 6.8 gRPC: RPC with protobuf underneath

> "Much faster to encode and decode than JSON, and much easier to understand than ASN.1. That combination is why it won."

| `.proto` | TLV | HTTP/2 | no curl |
|---|---|---|---|
| **Schema first.** Generate client and server code in any language. | **Tag-length-value on the wire**, so new fields do not break old clients. | **Binary and multiplexed.** Many streams over one connection. | **The cost: you cannot read it without tooling.** Speed bought with debuggability. |

🧠 What each part means:
- **gRPC**: Google's open-source RPC framework. **protobuf** (Protocol Buffers): its schema language and binary encoding.
- A `.proto` schema looks like this (compare with the ASN.1 one):

```proto
syntax = "proto3";
message Item {
  int32  item_code  = 1;   // the "= 1" is the field's tag number on the wire
  string color      = 2;
  bool   is_taxable = 3;
}
service Shop {
  rpc GetItem (ItemRequest) returns (Item);
}
```

- On the wire each field is sent as (tag, then value; strings and nested messages also carry a length). That is TLV-style, which is why an old client can skip a field number it does not know.
- **JSON** is text (`{"item_code": 150}`), so it is bigger and slower to parse. Protobuf sends field 1 = 150 as just 3 bytes: `08 96 01`.
- **Multiplexed** = many independent requests share one TCP connection at the same time. (No more 6-connection limit.)
- **No curl**: you cannot `telnet` into gRPC and type. You need tools like `grpcurl`. You have given up the "your keyboard is a client" superpower from section 2 in exchange for speed.

🧠 How the whole lecture stacks up inside one gRPC call:

```
 your function call
   → protobuf encoding (schema + TLV)                 section 6.4–6.5, 6.8
   → length-prefixed message (framing option 3)       section 6.2
   → HTTP/2 frames (binary, multiplexed)              section 6.1, Week 5
   → TLS (encryption)                                 section 5.8
   → TCP (socket, connect, read, write, close)        sections 1–3
   → IP → Ethernet/Wi-Fi                              section 3.5
```

---

## 7. Hands-on lab

Everything below runs on your laptop. On macOS use the notes marked "mac"; on Linux, "linux". Use two or three terminal windows.

### Lab 1: Echo server, version 1 and version 2

```bash
# Terminal 1
gcc -o echo_server echo_server.c && ./echo_server
# Terminal 2
nc localhost 2026
hello            ← type
hello            ← echoed once; then the connection closes (version 1)
```

Repeat with `echo_server_persistent.c`: now every line is echoed until you press Ctrl-C in terminal 2.

**What it shows:** one `read()` is not a conversation; the inner loop fixes it.

### Lab 2: One client at a time

With the persistent server running, open **two** `nc localhost 2026` windows. The second connects (the kernel did the handshake) but gets no echo until you close the first. Then all its waiting lines are echoed at once.

**What it shows:** the accept queue (1.7) and why we need fork or epoll (5.1).

### Lab 3: See who is listening and on which port

```bash
# mac
lsof -nP -iTCP -sTCP:LISTEN | grep echo
# linux
ss -tlnp | grep 2026
```

Expected: a line showing `*:2026` (the `*` / `0.0.0.0` is `INADDR_ANY`). Now remove `htons()` from the code, recompile and run again: the server will be on port **59911** instead.

### Lab 4: "Connection refused" vs "it just hangs"

```bash
nc -v localhost 2027        # nothing on 2027 → "Connection refused" immediately
```

For the hang: write a tiny server that does socket/bind/listen and then `sleep(1000)` without calling `accept()`. `nc localhost 2026` connects successfully but nothing ever comes back.

### Lab 5: SIGPIPE kills silently

```bash
# Terminal 1
gcc -o sigpipe_server sigpipe_server.c && ./sigpipe_server ; echo "exit status: $?"
# Terminal 2
gcc -o sigpipe_client sigpipe_client.c && ./sigpipe_client
```

Expected in terminal 1, after about a second: `exit status: 141` (128 + 13 = killed by SIGPIPE), with no error message printed. Then add `signal(SIGPIPE, SIG_IGN);` (and `#include <signal.h>`) at the top of `main` and print the return value of each `write()`: you get `-1` instead of a dead process.

### Lab 6: Be a client by hand

```bash
telnet google.com 80        # (or: nc google.com 80)
GET / HTTP/1.1
Host: google.com
                            ← press Enter on an empty line
```

Expected: `HTTP/1.1 301 Moved Permanently` with a `Location:` header. Try leaving out the `Host:` line (expect `400 Bad Request`), or leaving out the blank line (nothing happens, the server waits).

```bash
openssl s_client -connect www.hotstar.com:443
```

Expected: `CONNECTED(...)`, a `Certificate chain` section with `s:` (subject) and `i:` (issuer) lines, then details like `Protocol: TLSv1.3` and the cipher. You can then type an HTTP request inside the encrypted tunnel.

```bash
nc -l 9000                  # terminal 1: be a server
nc localhost 9000           # terminal 2: be a client; type in either window
curl http://localhost:9000/ # or point curl at it and see the exact request curl sends
```

The last one is a great trick: `nc -l` prints the raw HTTP request curl sent (`GET / HTTP/1.1`, `Host: localhost:9000`, `User-Agent: curl/...`, `Accept: */*`).

### Lab 7: curl with more v's

```bash
curl google.com                               # small "301 Moved" HTML body
curl -i google.com                            # same plus headers
curl -vv https://google.com 2>&1 | less       # DNS, TCP, TLS, certificate, '>' sent, '<' received
curl -L -s -o /dev/null -w '%{http_code} %{url_effective}\n' google.com   # follow redirects
```

### Lab 8: Many clients with fork

```bash
gcc -o fork_server fork_server.c && ./fork_server
# open 3 × nc localhost 2026: all three echo at the same time now
ps aux | grep fork_server     # one parent + one child per client
```

After you close some clients, `ps` may show `<defunct>` (zombie) children, because the code never reaps them (5.3).

### Lab 9: Look at the bytes

```bash
# mac (loopback interface is lo0)
sudo tcpdump -ni lo0 port 2026 -w 1.pcap
# linux
sudo tcpdump -ni any port 2026 -w 1.pcap

# in another terminal: nc localhost 2026, type "hello", then quit
# stop tcpdump with Ctrl-C, then:
tcpdump -r 1.pcap -A
tcpdump -r 1.pcap -X
```

Expected: `Flags [S]`, `[S.]`, `[.]` (the handshake), `[P.]` packets carrying `hello`, then `[F.]` (FIN). With the SO_LINGER client on port 8080 (capture `port 8080`), you see `[R]` (RST) instead. Open `1.pcap` in Wireshark to see the Ethernet/IP/TCP layers decoded.

On Linux, also try `strace -f ./echo_server` to watch the syscalls.

### Lab 10: Limits and DNS

```bash
ulimit -n                                   # your fd limit (often 256 on mac, 1024 on linux)
sysctl kern.ipc.somaxconn                   # mac: backlog cap
sysctl net.core.somaxconn                   # linux: backlog cap
cat /etc/hosts                              # checked before DNS
dig google.com                              # see the DNS answer and its TTL (or: nslookup)
echo -n 'Man' | base64                      # TWFu
echo -n 'hello' | xxd                       # hex dump: 6865 6c6c 6f
```

---

## 8. Homework / assignments

The final slides list six "things to try", plus the closing challenge: **"Go break something on port 2026. Every abstraction you use sits on the seven calls we started with. Go find the `accept()` inside your framework."**

### HW 1: Try Wireshark on your laptop
> If you use Redis, look for that traffic in Wireshark or tcpdump. RESP is text and length-prefixed: you will recognise it.

**Approach / hints (🧠):**
- Run `redis-server`, capture port 6379 (`sudo tcpdump -ni lo0 port 6379 -w redis.pcap`), then run `redis-cli SET name alice` and `redis-cli GET name`.
- RESP (REdis Serialization Protocol) for `GET name` looks like:
  ```
  *2\r\n        ← array of 2 elements
  $3\r\nGET\r\n ← bulk string of length 3
  $4\r\nname\r\n← bulk string of length 4
  ```
  Notice it is **both** delimited (`\r\n`) and length-prefixed (`$3`), just like HTTP.

### HW 2: Write a server and client in your language
> The calls have the same names. That is the point of having started in C.

**Hint (🧠)**, the same echo server in Python:

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)        # socket
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(("0.0.0.0", 2026))                                    # bind
s.listen(1)                                                  # listen
while True:
    c, addr = s.accept()                                     # accept
    while (data := c.recv(4096)):                            # read
        c.sendall(data)                                      # write
    c.close()                                                # close
```

Note that Python handles `htons` for you, and `recv` returning `b""` means EOF (like `read()` returning 0). Try Go (`net.Listen`, `Accept`) or Java (`ServerSocket.accept()`) too and find the same names.

### HW 3: Make them do more than echo
> The moment there is structure, you have to pick a framing strategy. Notice which one you reach for.

**Approach (🧠):** e.g. a tiny key-value store with commands `SET key value` and `GET key`. Most people first reach for **newline-delimited** text. Then try storing a value that contains a newline and see it break. Then switch to a length prefix. Test by sending two commands in one `write()` and by sending one command split across two `write()`s with a `sleep` in between; your server must handle both.

### HW 4: Write base64 encoding from scratch
> It forces you to think in bit groups instead of bytes, the exact shift binary protocol work needs.

**Hints (🧠):**
- Process input 3 bytes at a time: `n = (b0 << 16) | (b1 << 8) | b2`.
- Output 4 characters: `ALPHABET[(n >> 18) & 63]`, `[(n >> 12) & 63]`, `[(n >> 6) & 63]`, `[n & 63]`.
- Handle the last 1 or 2 bytes with `==` or `=` padding.
- Test against `base64` on the command line: `Man` → `TWFu`, `hi` → `aGk=`, `h` → `aA==`, empty → empty. Bonus: write the decoder.

### HW 5: Pass complicated messages with TLV
> Both directions, nested structures, optional fields. Then add a field and check your old parser still works.

**Approach (🧠):**
- Decide: Type = 1 byte, Length = 2 bytes big-endian. Define types (e.g. 1 = name, 2 = age, 3 = address, which is itself a nested TLV list).
- Optional fields: simply leave the TLV out.
- Write the parser so that unknown types are **skipped** using the length.
- Test: keep a copy of version-1 parser, add a new type 9 in the sender, and confirm the old parser still reads names and ages correctly. That proves forward compatibility.
- Always check that `Length` does not go past the end of the buffer.

### HW 6: Write a client that imitates curl
> Follow redirects, print headers, handle chunked encoding. You will end up actually reading the RFC.

**Approach (🧠):**
1. Parse a URL into host, port (default 80), path.
2. Resolve with `getaddrinfo`, connect, send `GET <path> HTTP/1.1\r\nHost: <host>\r\n\r\n`.
3. Read headers until `\r\n\r\n` (delimited framing). Print them.
4. Body: if `Content-Length: N`, read exactly N bytes (length-prefix framing). If `Transfer-Encoding: chunked`, loop: read a hex length line, then that many bytes, then `\r\n`, until a chunk of length 0.
5. If the status is 301/302 with a `Location:` header, repeat with the new URL (limit the number of redirects to avoid loops).
6. HTTPS needs TLS (e.g. OpenSSL's `SSL_connect`), which is a good stretch goal. The RFC to read is RFC 9112 (HTTP/1.1 message syntax). 🧠

---

## 9. Summary cheat-sheet

### The call sequence

```
 SERVER:  socket → bind → listen → accept → read/write ... → close
 CLIENT:  socket → connect (kernel picks ephemeral port) → write/read ... → close
```

### Key terms

| Term | One-line meaning |
|---|---|
| System call | A request from your program to the kernel (e.g. `socket`, `read`). |
| File descriptor (fd) | Small integer the kernel gives you for an open file or socket. |
| Socket | An fd with a network connection (or listener) behind it. |
| `socket()` | Create a socket. `AF_INET` = IPv4, `SOCK_STREAM` = TCP. |
| `bind()` | Attach a socket to an IP and port. Server only. |
| `listen(fd, backlog)` | Make it a listening socket; backlog = accept-queue size (capped by `somaxconn` on Linux). |
| `accept()` | Block until a client is queued; return a new fd for that client. |
| `connect()` | Client: do the 3-way handshake with a server. |
| `read()` return values | `>0` bytes, `0` = EOF (peer closed), `-1` = error. |
| `INADDR_ANY` | 0.0.0.0: listen on every interface. |
| Port < 1024 | Privileged: root only (on Linux). Use 8080 + a front proxy. |
| Ephemeral port | Temporary client port chosen by the kernel (Linux 32768–60999). |
| 4-tuple | (src IP, src port, dst IP, dst port): identifies a TCP connection. |
| SYN / ACK / FIN / RST | Start / acknowledge / polite close / abort. |
| Connection refused | Nothing on that port: kernel replied RST. |
| Hangs | Bound but nobody calling `accept()` (or the request is not finished). |
| Accept queue | Kernel-completed connections waiting for `accept()`. |
| TIME_WAIT | ~60 s hold after close; causes "address already in use" and port exhaustion. |
| `SO_REUSEADDR` | Lets a restarted server bind despite TIME_WAIT. Set before `bind()`. |
| `SO_LINGER {1,0}` | Abortive close: send RST instead of FIN. |
| SIGPIPE (13) | Write to a closed peer → default kills the process. Ignore it with `SIG_IGN`. |
| SIGINT (2) / SIGTERM (15) / SIGKILL (9) | Ctrl-C / polite kill / uncatchable kill. |
| SIGHUP / SIGTSTP, SIGSTOP | SSH disconnect / Ctrl-Z pause. |
| Big endian / network byte order | Most significant byte first. The wire format. |
| `htons` / `htonl` / `ntohs` / `ntohl` | Convert 2/4-byte numbers between host and network order. |
| `gethostbyname()` | Old DNS lookup: IPv4-only, not thread-safe, blocks. Use `getaddrinfo()`. |
| DNS lookup order | `/etc/hosts` → resolver (UDP) → root → TLD → authoritative; cached for TTL. |
| OSI layers used | L7 app (HTTP/echo), L4 transport (TCP, ports), L3 network (IP), L2 link (MAC), L1 physical. |
| telnet / nc | Raw text TCP clients; `nc -l` is a raw server. |
| `openssl s_client` | Raw client that does TLS first. |
| curl `-i` / `-vv` | Show headers / show everything (DNS, TLS, request, response). |
| HTTP request | `GET / HTTP/1.1\r\nHost: x\r\n\r\n`: CRLF and a mandatory blank line. |
| `fork()` | Clone the process. Returns 0 in child, child PID in parent, -1 on failure. |
| select | Portable, O(n), ~1024 fd cap (`FD_SETSIZE`). |
| epoll / kqueue / IOCP | Linux / BSD-macOS / Windows; O(ready). nginx and Node use these. |
| `ulimit -n` | Max open fds per process (often 1024). Sockets count. |
| Connection pool | Reuse open connections; skip TCP + TLS handshakes. |
| Chrome's 6 | Old per-host connection cap; extra requests queue. |
| Promiscuous mode | NIC keeps all frames, not just its own; how sniffers work. |
| tcpdump `-w` / `-r` / `-A` / `-X` | Write pcap / read pcap / ASCII / hex+ASCII. |
| TLS | Encrypts a TCP connection; asymmetric to agree a key, symmetric for data. |
| Certificate / CA chain | Binds a public key to a name / why you trust that binding. |
| Pinning | Trust one exact certificate, not any CA. |
| Framing | How the receiver knows where a message ends: fixed, delimited, length-prefix. |
| BCD | 4 bits per decimal digit; hex dump shows the digits; SIM, SMS, cards. |
| ASN.1 | Schema language (1984) → generated codecs; X.509, LDAP, SNMP, telecom. |
| TLV | Type-Length-Value; unknown fields skippable → forward compatible. |
| Hex | 2 chars per byte; for reading. |
| base64 | 3 bytes → 4 chars (+33%); for text-only transport. Not encryption. |
| RPC | Marshal function name + args, frame, send; the network is not a function call. |
| gRPC | RPC with protobuf (.proto schema, TLV-style) over HTTP/2; fast, not human-readable. |

### Symptom → likely cause (🧠)

| You see | Probably |
|---|---|
| `Connection refused` | Nothing listening on that port (or wrong port: forgot `htons`). |
| Client hangs forever | Server not calling `accept()`, busy with another client, or request missing its blank line. |
| `Address already in use` on restart | TIME_WAIT; set `SO_REUSEADDR` before `bind()`. |
| Server process vanishes, no error, exit code 141 | SIGPIPE. |
| `Too many open files` (EMFILE) | fd leak (missing `close`) or `ulimit -n` too low. |
| `Cannot assign requested address` under load | Ephemeral ports exhausted; use a pool. |
| Half a message, or two messages glued together | Framing bug: one `read()` is not one message. |
| `<defunct>` processes | fork server not reaping children. |

---

## 10. Self-check questions

**1. List the seven system calls of a TCP server in order. Which ones does a client not use, and what does it use instead?**
<details><summary>Answer</summary>

`socket, bind, listen, accept, read, write, close`. A client does not call `bind`, `listen` or `accept`. It calls `socket`, then `connect` (which does the handshake and gets an ephemeral port from the kernel automatically), then `read`/`write`/`close`.
</details>

**2. What do the three possible return values of `read()` on a socket mean?**
<details><summary>Answer</summary>

`> 0`: that many bytes were received. `0`: end of file, the peer closed the connection cleanly (sent FIN). `-1`: an error (check `errno`, e.g. `ECONNRESET`).
</details>

**3. Why does the first `echo_server.c` echo only your first line, and what one change fixes it?**
<details><summary>Answer</summary>

It calls `read()` once, `write()` once, and then `close()`s the client. The fix (in `echo_server_persistent.c`) is an inner loop: `while ((n = read(...)) > 0) write(...)`, so it keeps echoing until the client closes.
</details>

**4. A client gets "Connection refused" in one case and just hangs in another. Explain both in terms of what the kernel does.**
<details><summary>Answer</summary>

Refused: no socket is bound/listening on that port, so the kernel answers the SYN with an RST immediately. Hang: a socket is bound and listening, so the kernel completes the handshake by itself and puts the connection in the accept queue, but the process never calls `accept()` (or is busy), so no data ever comes back and the client waits until a timeout. The hang looks just like an overloaded server.
</details>

**5. Why do production deployments bind their application to 8080 instead of 80?**
<details><summary>Answer</summary>

Ports below 1024 require root. If the app runs as root, any bug in its request parsing becomes a root compromise. So the app runs as an unprivileged user on 8080, and a small front proxy that does almost nothing owns 80 and 443 and forwards traffic.
</details>

**6. What is the second argument of `listen()`, and why might the value in your code not be the value in effect?**
<details><summary>Answer</summary>

It is the backlog: the size of the queue of connections the kernel has already completed (handshake done) but your program has not yet `accept()`ed. On Linux, the kernel caps it at `net.core.somaxconn`, so asking for more than that silently gives you the cap. Also, what happens when the queue is full (refused vs hanging) differs between systems.
</details>

**7. Walk through what happens in the SIGPIPE demo. How would you make a real server survive it?**
<details><summary>Answer</summary>

The client connects, writes "hello", then closes with `SO_LINGER {1,0}`, which sends an RST instead of a FIN. The server read "hello", slept 1 s, and then writes to the dead connection. Writing to a peer that has gone away raises SIGPIPE, whose default action terminates the process with no error message, so the remaining code never runs. (On Linux the first write after the RST may return ECONNRESET, and the next one raises SIGPIPE; either way the process dies.) Fix: `signal(SIGPIPE, SIG_IGN)` at startup (or `MSG_NOSIGNAL` / `SO_NOSIGPIPE`) and then check `write()` for `-1` with `errno == EPIPE`/`ECONNRESET`.
</details>

**8. Port 2026 is `0x07EA`. Show how it is stored on a little-endian laptop vs on the wire. What port would the server end up on if you forgot `htons()`?**
<details><summary>Answer</summary>

Little-endian memory: `EA 07`. Network (big-endian): `07 EA`. Without `htons`, the struct contains bytes `EA 07`, which the network reads as `0xEA07` = 59911. The server listens on 59911 and clients aiming at 2026 get "Connection refused".
</details>

**9. Why is `gethostbyname()` described as "a whole protocol in one function call"? Name two reasons to prefer `getaddrinfo()`.**
<details><summary>Answer</summary>

It checks `/etc/hosts`, then sends a UDP DNS query to the resolver in `/etc/resolv.conf`, which recurses root → TLD → authoritative; the answer is cached for its TTL. It blocks the caller, possibly for seconds, with no timeout you control. Prefer `getaddrinfo()` because `gethostbyname` is IPv4-only and not thread-safe (it returns a shared static buffer).
</details>

**10. In `fork_server.c`, what goes wrong if (a) the child does not `return 0`, (b) the parent does not `close(client_fd)`?**
<details><summary>Answer</summary>

(a) The child falls through into the `while (1)` loop and starts calling `accept()` itself, so processes multiply with every connection. (b) The connection is not fully closed while the parent still holds a copy, so the client never sees the server hang up, and the parent leaks one fd per client until it hits the fd limit (`EMFILE`). (Also: `fork()` returning -1 is not checked, and finished children become zombies because nobody reaps them.)
</details>

**11. Compare fork-per-connection with select/epoll. Why is select O(n) and epoll O(ready)?**
<details><summary>Answer</summary>

fork: simple, OS schedules for you, crashes are contained per client; but one process per client is fine for hundreds, hopeless at ten thousand. select/epoll: one process handles many sockets by asking the kernel which are ready, no context switching or per-client memory; but harder to write and one slow handler blocks everyone (used by nginx and Node). select makes you rebuild the fd set on every call and the kernel scans all of them (and it is capped near 1024 by `FD_SETSIZE`). epoll registers interest once and returns only the descriptors that changed, so the work grows with the number of ready sockets.
</details>

**12. "TCP gives you a byte stream, not messages." What are the three framing options, and which does HTTP use?**
<details><summary>Answer</summary>

Fixed length (simple, but padding and N can never change), delimited (read until a sentinel; breaks when the delimiter appears in data, needs escaping), and length prefix (send the size, then the bytes; reliable and binary-safe, what most modern protocols do). HTTP/1.1 uses a delimiter (the blank line `\r\n\r\n`) for headers and a length prefix (`Content-Length`) for the body.
</details>

**13. Why is TLV forward compatible, and what does that let you do in practice?**
<details><summary>Answer</summary>

Each field carries its own length, so a parser that meets an unknown Type can skip exactly Length bytes and continue. Old clients therefore keep working when newer senders add fields, so you can roll out new versions of servers and clients at different times. TLS extensions, ISO 8583 and protobuf rely on this.
</details>

**14. "SSL secures TCP, not HTTP." What does that mean, and why does TLS use both asymmetric and symmetric crypto?**
<details><summary>Answer</summary>

TLS sits between TCP and the application protocol, so it can protect any protocol on a socket (HTTP, SMTP, database connections), not only HTTP. Asymmetric crypto lets two parties who share no secret agree on one over a public network and verify identity via certificates and the CA chain; but it is slow, so once the shared secret exists, all data is encrypted with fast symmetric crypto.
</details>

**15. What is the 33% in base64, and why is base64 "a wrapper, not a design"?**
<details><summary>Answer</summary>

base64 turns every 3 bytes (24 bits) into 4 characters of 6 bits each, so the output is 4/3 the size: 33% bigger. It exists only to carry binary data safely through text-only channels (Basic auth, JWTs, email, data URIs). It adds no structure, framing or security, so it is a wrapper around a protocol, not a way to design one.
</details>

**16. Why does the lecture say "the network is not a function call" about RPC?**
<details><summary>Answer</summary>

A local call either runs or does not; a remote call can fail halfway, time out after the work was actually done, and then be retried, possibly doing the work twice. So RPC systems need timeouts, careful retries and idempotent (safe-to-repeat) operations. "Every RPC framework rediscovers this the hard way."
</details>

---

> ### `$ Go break something on port 2026.`
> Every abstraction you use sits on the seven calls we started with. Go find the `accept()` inside your framework.
>
> **socket · bind · listen · accept · read · write · close**
