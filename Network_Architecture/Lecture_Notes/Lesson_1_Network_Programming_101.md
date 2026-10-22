# Lesson 1: Network Programming 101
### CN at Scaler, Week 1: "Network programming, from the socket up"

> `2.5 hours` · `7 system calls` · `port 2026 = our port`
>
> **socket · bind · listen · accept · read · write · close**

---

## The plan

Built bottom-up: start with the smallest program that can accept a TCP connection, end with designing your own binary protocol.

```
 PART 1                 PART 2                  PART 3                   PART 4
 A server, and          A client, plus OSI,     curl, many clients at    Designing a protocol
 everything that        DNS and byte order      once, packets on the     of your own
 breaks around it                               wire, SSL
```

---

## Table of contents

1. [The Server](#1-the-server)
2. [You Already Have a Client](#2-you-already-have-a-client)
3. [Writing the Client](#3-writing-the-client)
4. [HTTP and curl](#4-http-and-curl)
5. [Many Clients at Once](#5-many-clients-at-once)
6. [Protocol Design](#6-protocol-design)
7. [Homework: things to try (with solutions)](#7-homework-things-to-try-with-solutions)
8. [Summary cheat-sheet](#8-summary-cheat-sheet)
9. [MCQs (mid → advanced)](#9--mcqs-mid--advanced)

---

## 1. The Server

### 1.1 A TCP server is seven system calls

> "Everything else you have ever used (Express, Flask, net/http) is a wrapper around this sequence."

```
   1         2        3         4         5        6        7
 socket → bind → listen → accept → read → write → close
          └─────────────────────┘
          the four that only a server makes
```

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

It reads **once**, writes the same bytes back **once**, then closes that client and goes back to `accept()`.

### 1.3 Demo: compile it, talk to it

> "Two commands and you have a network service running on your laptop."

```bash
$ gcc -o echo_server echo_server.c
$ nc localhost 2026
```

### 1.4 Ports: everything below 1024 belongs to root

> "Not a rule you cannot break, a rule about **who** is allowed to break it."

```
 < 1024   ROOT ONLY
          80 HTTP · 443 HTTPS · 25 SMTP · 21 FTP · 110 POP

 ≥ 1024   ANYONE
          3000 node · 8080 alt-http · 6379 redis · 2026 ours
```

**Why does it matter?**
- Binding port 80 means running as **root**.
- Running as root means **a bug in your request parser is a root compromise**.
- So real deployments **bind 8080 as an unprivileged user** and let a **front proxy** (which does almost nothing) own 80 and 443.

### 1.5 Thought experiment: what breaks if you skip a call?

| You skipped | What the client sees | Why |
|---|---|---|
| **No `bind()`** | **Connection refused** | Nothing owns the port, so the kernel answers the SYN with an **RST** straight away. Fast, loud, easy to debug: the client knows instantly nobody is home. |
| **`bind()` but no `accept()`** | **It just hangs** | The kernel completes the handshake **on your behalf**. The connection is established and sits in the queue. Your process never picks it up, so the client waits until TCP times out. Silent, slow, and **it looks exactly like an overloaded server**. |

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

**What changed:** one inner loop. The server keeps reading and echoing while `read()` returns more than 0, and only then closes the client.

### 1.7 `listen(server_fd, 1)`: that second argument is a queue

The kernel accepts connections on your behalf and stacks them up. The stack has a size.

> 💡 **Simple words mein:** Kernel ek receptionist hai. Caller ka greeting (handshake) khud kar deta hai aur waiting room mein bitha deta hai. `accept()` = agla token number bulao. `listen()` ka number = waiting room mein kitni kursiyan hain.

1. **The kernel finishes the handshake without you.** By the time `accept()` returns, the TCP connection has been established for a while. The client has been waiting in line.
2. **Full-queue behaviour is not portable.** "Connection refused" on some systems; an indefinite `Trying...` on others. **Do not build logic on whichever one you saw locally.**
3. **We passed 1. Real servers pass hundreds.** And **Linux caps whatever you ask for at `net.core.somaxconn`**, so the number in your code may not be the number in effect.

### 1.8 Signals: SIGPIPE is the one that will kill you

Write to a socket the peer already closed and, by default, your process dies. No error, no exception: gone.

> 💡 **Simple words mein:** Aap phone pe bol rahe ho, par doosri taraf wala phone kaat chuka hai. Kernel ka default rule: "khaali line pe bola? Process khatam." Na error, na message.

| Number | Signal | Meaning |
|---|---|---|
| 2 | `SIGINT` | Ctrl-C |
| 9 | `SIGKILL` | cannot be caught |
| **13** | **`SIGPIPE`** | **write to a closed peer** |
| 15 | `SIGTERM` | the polite kill |
| - | `SIGHUP` | on SSH disconnect |
| - | `SIGSTOP` / `SIGTSTP` | on Ctrl-Z |

### 1.9 The SIGPIPE demo

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

**Putting it together:** the client connects, sends "hello", then closes with `SO_LINGER {1, 0}`, which sends an **RST** instead of a FIN. The server reads "hello", sleeps 1 s, then writes to the reset connection: **SIGPIPE** kills it, so the second write and `return 0` never run.

---

## 2. You Already Have a Client

### 2.1 telnet is a client. Type the protocol yourself.

> "If a protocol is text, your keyboard is a perfectly good implementation of it."

```bash
$ telnet google.com 80
$ telnet localhost 2026
$ telnet mail.example.com 25
```

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

You type `GET / HTTP/1.1`, `Host: google.com` and a blank line; Google answers with a `301 Moved Permanently`.

### 2.3 Beyond port 80: most services do not speak HTTP

> "And once TLS is in the way, telnet is useless: you need something that can do the handshake for you."

```bash
$ openssl s_client -connect www.hotstar.com:443   # for anything encrypted (does TLS for you)
$ nc localhost 2026                               # netcat for everything else
$ nc -l 9000                                      # netcat listening on port 9000
```

---

## 3. Writing the Client

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

**What is missing?** No `bind()`, no `listen()`, no `accept()`, and no local port number anywhere. Yet the return traffic finds its way back.

### 3.2 Ephemeral ports: clients never bind, the kernel picks a port for you

| SERVER | CLIENT |
|---|---|
| **Binds a known port.** It has to: clients need a fixed address to aim at. That is the entire reason well-known ports exist. | **Gets a random high port**, assigned by the kernel at `connect()` time from the **ephemeral range**, typically **32768–60999 on Linux**. |
| `bind()` → `listen()` → `accept()` | **Finite**, and **TIME_WAIT holds each one for ~60 s after close. Busy clients do run out.** |

### 3.3 `htons()`: your CPU and the network disagree on byte order

> "htons = **h**ost **to** **n**etwork, **s**hort. It exists because x86 stores 2026 backwards from how the wire wants it."

```
 port 2026 = 0x07EA

 little endian (your x86 / ARM laptop)  :   EA 07
 big endian    (network byte order)     :   07 EA

 htons() swaps them.   htonl() for 4 bytes.   ntohs() / ntohl() on the way back.
```

> 💡 **Simple words mein:** Date jaisa hai: India 12/08/2026 likhta hai (din pehle), US 08/12/2026 (mahina pehle). Order pe agree nahi kiya toh galat padhoge. Network ka order: bada byte pehle (big endian).

### 3.4 `gethostbyname()`: one function call is a whole protocol

> "gethostbyname() is DNS: a UDP request, a recursive resolver, and a cache, hidden behind a synchronous call that can block for seconds."

1. **`/etc/hosts` first.** Before any network traffic. This is why editing hosts overrides everything, and why a stale entry there is so hard to debug.
2. **UDP query to your resolver.** One small datagram to whatever is in `/etc/resolv.conf`. It recurses on your behalf: **root → TLD → authoritative**.
3. **Cached for the TTL, and it blocks.** Fast when warm, seconds when cold, with **no timeout you control**. Use `getaddrinfo()` in real code; **`gethostbyname` is IPv4-only and not thread-safe.**

### 3.5 OSI: we have been writing layer 7 this whole time

> "Ports are layer 4. IP addresses are layer 3. We touched all three in twenty lines and never named them."

| Layer | Name | In this lecture |
|---|---|---|
| 7 | Application | our echo protocol, HTTP |
| 4 | Transport | TCP · ports · `htons(2026)` |
| 3 | Network | IP · `INADDR_ANY` · `127.0.0.1` |
| 2 | Data link | Ethernet, Wi-Fi · MAC addresses |
| 1 | Physical | copper, fibre, radio |

---

## 4. HTTP and curl

### 4.1 An HTTP client is `client.c` with a string in it

> "Point it at port 80, write the request text, read until close. That is the whole protocol."

```
GET / HTTP/1.1
Host: google.com

   ↑ CRLF line endings, and that blank line is mandatory
```

### 4.2 curl: the best HTTP teaching tool you have

> "You can learn a lot about headers and SSL debugging from one command with enough v's on it."

```bash
$ curl google.com
$ curl -i google.com
$ curl -vv https://google.com 2>&1 | less
```

---

## 5. Many Clients at Once

### 5.1 The debt comes due: our server serves exactly one client

> "When I wrote it I said it would support one at a time. Here are the two ways out, and both are still shipping in production today."

| ONE PROCESS PER CONNECTION: `fork()` | ONE PROCESS, MANY SOCKETS: `select` / `epoll` |
|---|---|
| **Dead simple**, and the OS does the scheduling for you. | Ask the kernel **which sockets are ready**, then handle only those. |
| Each child has its own memory, so **a crash is contained**. | **No context switching, no per-client memory.** |
| Costs a process per client. **Fine for hundreds, hopeless at ten thousand.** | **Harder to write**, and **one slow handler blocks everything**. **This is what nginx and Node do.** |

> 💡 **Simple words mein:** fork = har table ka alag waiter: easy, ek waiter behosh ho toh baaki tables safe, par 10,000 tables = 10,000 waiters. select/epoll = ek tez waiter jo sirf ready table pe jaata hai. Par wo ek table pe atak gaya toh sab wait karenge.

### 5.2 `fork()` returns twice. Read the return value.

> "The same line of code runs in two processes, and the only way to tell which one you are is what came back."

| Return value | Who you are | What to do |
|---|---|---|
| `0` | **the child** | Handle this one client, then exit. |
| `> 0` | **the parent** (the value is the child's PID) | Close your copy of the fd and go back to `accept()`. |
| `-1` | fork **failed** (out of processes or memory) | "Almost nobody checks for this." |

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

- **Child** (`fork() == 0`): runs the echo loop for this one client, closes it, and `return 0` (exits).
- **Parent**: closes **its copy** of `client_fd` and goes back to `accept()`.

### 5.4 `select` / `epoll`: one process, ask the kernel who is ready

> "Instead of blocking on one socket, hand the kernel your whole set and let it tell you which ones have data."

| SELECT | EPOLL |
|---|---|
| **Portable, and O(n)** | **Linux, and O(ready)** |
| Rebuild the fd set on every call; the kernel walks all of them every time. | Register interest once. The kernel returns only the descriptors that actually changed. |
| `FD_SETSIZE` caps you around **1024** descriptors. Works everywhere, which is why it is still around. | **kqueue** on BSD and macOS, **IOCP** on Windows. This is what nginx and Node are built on. |

### 5.5 The client side: Chrome would only open six connections to you

> "Connection limits are not a browser quirk; they are the client-side version of the same resource problem."

| | Meaning |
|---|---|
| **6** | Chrome's old **per-host connection cap**. Requests **7–10 simply queued**. |
| **`ulimit -n`** | Your **hard ceiling on open file descriptors**. **Sockets count.** Often just **1024**. |
| **pool** | Open once, reuse many times: you skip the **TCP and TLS handshake** every time after the first. |

### 5.6 Socket options: a socket can accept connections on any IP

> "Read the man page for socket options once. Some of what is in there is a networking history lesson."

- **◆ `INADDR_ANY`: listen on every interface.** Convenient in development; a decision to make deliberately in production. Bind to a specific address when you mean one.
- **◆ Promiscuous mode: see traffic that is not yours.** On a **hub**, every frame reaches every port. This is how packet sniffers work, and why plaintext protocols died.
- **◆ `SO_REUSEADDR`: the one you will actually use.** Without it, restarting your server fails with **"address already in use"** while the old socket sits in **TIME_WAIT**.

### 5.7 tcpdump / Wireshark: stop guessing, look at the bytes

> "Capture it once, then read it as ASCII and as hex. Everything we have talked about is visible in there."

```bash
$ sudo tcpdump -ni any port 2026 -w 1.pcap     # capture port 2026 into a file
$ tcpdump -r local_capture.pcap -A             # read it back as ASCII
$ tcpdump -r local_capture.pcap -X             # read it back as hex + ASCII
```

### 5.8 SSL / TLS: SSL secures TCP, not HTTP

> "It sits under the application protocol, which is why the same machinery protects SSH, SMTP, database connections and anything else you put on a socket."

1. **Exchange keys and certificates.** **Asymmetric** crypto to agree on a shared secret, then **symmetric** crypto for the data, because symmetric is dramatically faster.
2. **The certificate binds a key to an identity.** And the **CA chain** is the reason you believe it. "This is the part that expires at 3am and takes your site down."
3. **Pinning removes the CA from the trust equation.** Common in mobile apps: trust *this exact certificate*, not whatever any CA in the world is willing to sign.

---

## 6. Protocol Design

### 6.1 Text won. Now protocols are going binary again.

> "HTTP proves how far human-readable can take you, and HTTP/2 is binary."

| TEXT | BINARY |
|---|---|
| **You can read it.** telnet into it, tcpdump it, eyeball the logs. A real and large advantage; most of why HTTP won. | **The machine can read it faster.** Compact, unambiguous, cheap to parse. No whitespace rules, no case-sensitivity arguments. |
| Verbose, ambiguous at the edges, and slower to parse than it looks. | You need tooling to see anything at all, which is why **hex and base64 fluency stops being optional**. |

### 6.2 Message framing: how does the receiver know the message ended?

> "TCP gives you a byte stream, not messages. This is the single most important decision in your protocol, and there are only three answers."

> 💡 **Simple words mein:** TCP paani ki pipe hai, parcels ka dabba nahi. Receiver ko batana padega message kahan khatam hua: fixed size dabba, end pe "khatam" ka sticker (delimiter), ya pehle hi bol do "219 bytes aa rahe hain" (length prefix).

| Option | How | Trade-off |
|---|---|---|
| **1. Fixed length** | every message is exactly N bytes | Trivial to parse, zero ambiguity. You pay in **padding**, and **you can never change your mind about N**. |
| **2. Delimited** | read until the sentinel, e.g. HTTP's blank line between headers and body | Simple, **until the delimiter appears inside your data** and you need **escaping**. |
| **3. Length prefix** | send the size, then the bytes, e.g. `Content-Length: 219`: read exactly that many and stop | Robust, **binary-safe**, and **what most modern protocols do**. |

### 6.3 BCD: binary coded decimal, four bits per digit

> "The curious part is that the digits just show up. Convert to hex and the representation does not change."

```
 decimal     9      8      7      6      5
 nibbles   1001   1000   0111   0110   0101
 hex          98          76          5F
```

**Half the size of ASCII · no conversion step · exact decimal arithmetic · still used in SIM cards, SMS and card transactions.**

### 6.4 ASN.1: write the grammar, generate the codec

> "The most powerful system I have seen: you describe the shape of your data once and the encoder and decoder are produced for you."

| schema | codegen | 1984 |
|---|---|---|
| Describe your data once, in a **language-neutral grammar**. | Encoder and decoder are **generated**. You never hand-write parsing. | And **still under X.509, LDAP, SNMP and the whole telecom stack today**. |

**The ASN.1 definition: "This is the whole schema. Everything else is generated."**

```asn1
MyModule DEFINITIONS ::= BEGIN
    Item ::= SEQUENCE {
        itemCode    INTEGER (1..99999),
        color       VisibleString ("Black" | "Blue" | "Brown"),
        isTaxable   BOOLEAN
    }
END
```

`Item` is a record of three fields: an integer limited to 1–99999, a string limited to three colours, and a true/false flag.

### 6.5 TLV: ASN.1 you can write on a napkin

> "Type, length, value, repeated. That is the entire specification, and it is enough to build a real protocol."

```
 ┌──────┬────────┬─────────────┐┌──────┬────────┬─────────────┐
 │ Type │ Length │ Value       ││ Type │ Length │ Value       │ ...
 └──────┴────────┴─────────────┘└──────┴────────┴─────────────┘
 <Type><Length><Body/Value><Type><Length><Body/Value>...
```

**Self-describing:** a parser that has never seen a field type can still skip it, because the length tells it how far to jump. That one property makes TLV **forward compatible**, and it is why **TLS extensions, ISO 8583 and protobuf** all work this way.

> 💡 **Simple words mein:** Courier parcel pe label hai "type: books, size: 3 kg". Naya type aaya jo purana staff nahi jaanta? Label dekh ke parcel side mein rakho aur agla uthao. Kuch toota nahi.

### 6.6 base64 / hex: how you see a binary protocol

> "Not fully related to framing, but base64 and hex are what you will end up using to debug any of this."

| HEX: for **reading** | BASE64: for **transport** |
|---|---|
| **Two characters per byte**, and `tcpdump -X` puts it beside the ASCII. | **Survives text-only channels**: Basic auth, JWTs, email attachments, data URIs. |
| Fluency here means **spotting a length prefix or a repeated tag by eye**. | **Costs 33% more bytes: 3 bytes become 4 characters.** A wrapper, not a design. |

### 6.7 RPC: framing plus a function name

> "Send the name, send the arguments, get a response. They call it **marshalling** and **demarshalling**."

1. **Marshal: objects become bytes.** Function name, then the arguments, encoded however your protocol says. This is the easy part.
2. **Frame it and send it.** The framing decision and the encoding decision from the last slides now do real work for you.
3. **The network is not a function call.** Calls fail halfway, time out with the work possibly done, and get retried. Every RPC framework rediscovers this the hard way.

> 💡 **Simple words mein:** UPI se ₹500 bheje, screen pe "timeout". Paise gaye ya nahi? Pata nahi. Dobara bheja toh shayad do baar kat gaye. Remote call fail, timeout aur retry ho sakta hai.

### 6.8 gRPC: RPC with protobuf underneath

> "Much faster to encode and decode than JSON, and much easier to understand than ASN.1. That combination is why it won."

| `.proto` | TLV | HTTP/2 | no curl |
|---|---|---|---|
| **Schema first.** Generate client and server code in any language. | **Tag-length-value on the wire**, so new fields do not break old clients. | **Binary and multiplexed.** Many streams over one connection. | **The cost: you cannot read it without tooling.** Speed bought with debuggability. |

---

## 7. Homework: things to try (with solutions)

> Closing challenge: **"Go break something on port 2026. Every abstraction you use sits on the seven calls we started with. Go find the `accept()` inside your framework."**

### HW 1: Try Wireshark on your laptop
> If you use Redis, look for that traffic in Wireshark or tcpdump. RESP is text and length-prefixed: you will recognise it.

**✅ Solution**

```bash
redis-server &                                     # start Redis (port 6379)
sudo tcpdump -ni lo0 port 6379 -w redis.pcap       # mac; on Linux use -i any
redis-cli SET name alice                           # in another terminal
redis-cli GET name
# Ctrl-C tcpdump, then:
tcpdump -r redis.pcap -A
```

Expected payloads (or open `redis.pcap` in Wireshark, filter `tcp.port == 6379`, right-click → Follow → TCP Stream):

```
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$5\r\nalice\r\n     → +OK\r\n
*2\r\n$3\r\nGET\r\n$4\r\nname\r\n                     → $5\r\nalice\r\n
```

**What to write down:** `*3` = array of 3 items, `$5` = "next string is 5 bytes" (length prefix), and every part ends in `\r\n` (delimiter). RESP uses both framing styles. (No Redis? Capture `curl http://example.com` on port 80 instead and spot `Content-Length`.)

### HW 2: Write a server and client in your language
> The calls have the same names. That is the point of having started in C.

**✅ Solution (Python)**

```python
# server.py
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)    # socket
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(("0.0.0.0", 2026))                                # bind
s.listen(1)                                              # listen
while True:
    c, addr = s.accept()                                 # accept
    while data := c.recv(4096):                          # read (b"" = client closed)
        c.sendall(data)                                  # write
    c.close()                                            # close
```

```python
# client.py
import socket
c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)    # socket (no bind: ephemeral port)
c.connect(("localhost", 2026))                           # connect
c.sendall(b"hello\n")                                    # write
print(c.recv(4096))                                      # read -> b'hello\n'
c.close()                                                # close
```

Run `python3 server.py`, then `python3 client.py` (or `nc localhost 2026`). Python calls `htons` for you.

### HW 3: Make them do more than echo
> The moment there is structure, you have to pick a framing strategy. Notice which one you reach for.

**✅ Solution:** a tiny key-value store with newline-delimited commands `SET key value` / `GET key`.

```python
import socket
store = {}
s = socket.create_server(("0.0.0.0", 2026))       # socket + bind + listen
while True:
    c, _ = s.accept()
    f = c.makefile("rwb")                        # buffered: handles split / glued reads
    for line in f:                               # framing = delimiter '\n'
        p = line.decode().split(maxsplit=2)
        if len(p) == 3 and p[0] == "SET": store[p[1]] = p[2]; r = "OK"
        elif len(p) == 2 and p[0] == "GET": r = store.get(p[1], "(nil)")
        else: r = "ERR"
        f.write((r + "\n").encode()); f.flush()
    c.close()
```

Test: `printf 'SET a 42\nGET a\n' | nc localhost 2026` → `OK` then `42` (two commands in one write still work).
**Observation:** you reach for the **delimiter** first; it breaks as soon as a value contains `\n`. The fix is a **length prefix** (send the value's byte count first), which is binary-safe.

### HW 4: Write base64 encoding from scratch
> It forces you to think in bit groups instead of bytes, the exact shift binary protocol work needs.

**✅ Solution:** take 3 bytes (24 bits), cut into four 6-bit numbers, map each to the alphabet; pad with `=`.

```python
A = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"

def b64(data: bytes) -> str:
    out = ""
    for i in range(0, len(data), 3):
        chunk = data[i:i+3]
        n = int.from_bytes(chunk.ljust(3, b"\0"), "big")      # 24 bits
        chars = "".join(A[(n >> s) & 63] for s in (18, 12, 6, 0))
        pad = 3 - len(chunk)
        out += chars[:4 - pad] + "=" * pad
    return out

for t in [b"Man", b"hi", b"h", b""]:
    print(t, b64(t))          # TWFu, aGk=, aA==, (empty)
```

Check against the system tool: `echo -n Man | base64` → `TWFu`.

### HW 5: Pass complicated messages with TLV
> Both directions, nested structures, optional fields. Then add a field and check your old parser still works.

**✅ Solution:** Type = 1 byte, Length = 2 bytes big-endian. Types: 1 = name, 2 = age, 3 = address (itself a TLV list: 1 = city, 2 = pin).

```python
import struct

def tlv(t, v: bytes): return struct.pack(">BH", t, len(v)) + v

def items(buf):                                   # split a buffer into (type, value)
    i = 0
    while i + 3 <= len(buf):
        t, n = struct.unpack_from(">BH", buf, i); i += 3
        if i + n > len(buf): raise ValueError("length runs past end")
        yield t, buf[i:i+n]; i += n

def parse_v1(buf):                                # the OLD parser: knows types 1, 2, 3
    m = {}
    for t, v in items(buf):
        if t == 1: m["name"] = v.decode()
        elif t == 2: m["age"] = v[0]
        elif t == 3: m["address"] = {k: x.decode() for k, x in items(v)}   # nested
        # unknown type -> skipped, the length already moved us past it
    return m

addr = tlv(1, b"Pune") + tlv(2, b"411001")
msg  = tlv(1, b"alice") + tlv(9, b"new field!") + tlv(2, bytes([21])) + tlv(3, addr)
print(parse_v1(msg))   # {'name': 'alice', 'age': 21, 'address': {1: 'Pune', 2: '411001'}}
print(parse_v1(tlv(1, b"bob")))   # optional fields missing -> {'name': 'bob'}
```

**Both directions over a socket:** send `struct.pack(">I", len(msg)) + msg` with `sendall`; the receiver reads 4 bytes, then exactly that many bytes, then calls `parse_v1`. The new type 9 field was skipped and the old parser still works: that is forward compatibility.

### HW 6: Write a client that imitates curl
> Follow redirects, print headers, handle chunked encoding. You will end up actually reading the RFC.

**✅ Solution (minimal skeleton, Python; usage `python3 mycurl.py http://google.com`)**

```python
import socket, ssl, sys
from urllib.parse import urlsplit, urljoin

def get(url, hops=5):
    u = urlsplit(url)
    https = u.scheme == "https"
    host, port = u.hostname, u.port or (443 if https else 80)
    path = (u.path or "/") + ("?" + u.query if u.query else "")
    s = socket.create_connection((host, port))                    # DNS + socket + connect
    if https: s = ssl.create_default_context().wrap_socket(s, server_hostname=host)
    s.sendall(f"GET {path} HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n".encode())
    f = s.makefile("rb")

    status = f.readline().decode().strip(); print(status)
    h = {}
    while (line := f.readline()) not in (b"\r\n", b""):          # headers end at blank line
        print(line.decode().rstrip())
        k, v = line.decode().split(":", 1); h[k.lower()] = v.strip()

    if status.split()[1] in ("301", "302", "303", "307", "308") and hops:
        return get(urljoin(url, h["location"]), hops - 1)       # follow redirect

    if h.get("transfer-encoding") == "chunked":                 # hex size, data, CRLF ... 0
        body = b""
        while size := int(f.readline().split(b";")[0], 16):
            body += f.read(size); f.readline()
    elif "content-length" in h:
        body = f.read(int(h["content-length"]))                 # length prefix
    else:
        body = f.read()                                         # read until close
    print(body.decode(errors="replace"))

get(sys.argv[1])
```

**Expected:** `python3 mycurl.py http://google.com` prints `HTTP/1.1 301 Moved Permanently` and its headers, follows the `Location`, then prints the final status, headers and HTML. The RFC to read for headers and chunked encoding is RFC 9112 (HTTP/1.1).

---

## 8. Summary cheat-sheet

```
 SERVER:  socket → bind → listen → accept → read/write ... → close
 CLIENT:  socket → connect (kernel picks an ephemeral port) → write/read ... → close
```

| Term | One-line meaning |
|---|---|
| Seven calls | socket · bind · listen · accept · read · write · close; bind/listen/accept are server-only. |
| `echo_server.c` vs persistent | Reads once then closes vs loops `read()`/`write()` until the client leaves. |
| Port < 1024 | Root only. Bind 8080 as an unprivileged user; a front proxy owns 80/443. |
| No `bind()` | Connection refused: kernel answers SYN with RST. |
| `bind()` but no `accept()` | Hangs: kernel did the handshake, connection waits in the queue until TCP times out. |
| `listen(fd, n)` | n = queue size; Linux caps it at `net.core.somaxconn`; full-queue behaviour not portable. |
| SIGINT 2 / SIGKILL 9 / SIGPIPE 13 / SIGTERM 15 | Ctrl-C / cannot be caught / write to closed peer / polite kill. |
| SIGHUP / SIGSTOP, SIGTSTP | SSH disconnect / Ctrl-Z. |
| `SO_LINGER {1, 0}` | close() sends RST instead of FIN. |
| telnet / `openssl s_client` / nc | Text protocols / anything encrypted / everything else (`nc -l` listens). |
| Ephemeral port | Kernel-assigned at `connect()`, 32768–60999 on Linux; TIME_WAIT holds each ~60 s. |
| `htons` / `htonl` / `ntohs` / `ntohl` | 2026 = 0x07EA: `EA 07` on x86/ARM, `07 EA` on the wire. |
| `gethostbyname()` | `/etc/hosts` → UDP to resolver → root → TLD → authoritative; cached for TTL; blocks; IPv4-only, not thread-safe → use `getaddrinfo()`. |
| OSI | L7 echo/HTTP · L4 TCP, ports · L3 IP, INADDR_ANY · L2 Ethernet, MAC · L1 copper, fibre, radio. |
| HTTP request | `GET / HTTP/1.1`, `Host:`, CRLF line endings, mandatory blank line. |
| curl | `curl` / `curl -i` / `curl -vv ... 2>&1 \| less`. |
| fork vs select/epoll | Process per client (simple, contained, not for 10k) vs one process asks kernel who is ready (nginx, Node). |
| `fork()` returns | 0 child · >0 parent (child's PID) · -1 failed. |
| select vs epoll | Portable, O(n), ~1024 (`FD_SETSIZE`) vs Linux, O(ready); kqueue (BSD/mac), IOCP (Windows). |
| 6 / `ulimit -n` / pool | Chrome per-host cap / fd ceiling, sockets count, often 1024 / reuse to skip TCP+TLS handshakes. |
| Socket options | `INADDR_ANY` all interfaces · promiscuous mode (hub, sniffers) · `SO_REUSEADDR` (restart despite TIME_WAIT). |
| tcpdump | `-w` capture to file · `-r` read · `-A` ASCII · `-X` hex. |
| TLS | Secures TCP, not HTTP; asymmetric to agree a secret, symmetric for data; certificate + CA chain; pinning. |
| Framing | Fixed (padding, N forever) · delimited (escaping) · length prefix (binary-safe, most modern). |
| BCD | 4 bits per digit; hex shows the digits; SIM, SMS, card transactions. |
| ASN.1 | Schema (1984) → generated codec; X.509, LDAP, SNMP, telecom. |
| TLV | Self-describing; unknown types skipped → forward compatible; TLS extensions, ISO 8583, protobuf. |
| Hex / base64 | 2 chars per byte, for reading / 3 bytes → 4 chars (+33%), for transport. |
| RPC / gRPC | Marshal + frame + send; network is not a function call / protobuf + TLV + HTTP/2, no curl. |

---

## 9 · MCQs (mid → advanced)

**Q1.** On a little-endian laptop, a server sets `addr.sin_port = 2026;` (no `htons`). Port 2026 is `0x07EA`. Which port does it actually end up on?
- A) 2026
- B) 59911
- C) 60935
- D) `bind()` fails because the port is invalid

> ✅ **Answer: B.** Memory holds `EA 07`; the network reads it big-endian as `0xEA07` = 234×256 + 7 = 59911. Nothing fails loudly, so clients on 2026 are simply refused.

**Q2.** A server calls `socket()`, `bind()` and `listen()`, but never `accept()`. A client runs `nc localhost 2026`. What does the client experience?
- A) "Connection refused" instantly
- B) An RST after the backlog fills
- C) An immediate SIGPIPE
- D) The connection succeeds, but nothing ever comes back until TCP times out

> ✅ **Answer: D.** The kernel completes the handshake on the server's behalf and queues the connection; since nobody accepts it, the client just hangs. "Refused" happens only when nothing owns the port.

**Q3.** With the first `echo_server.c` running, you type `hello` then `world` in one `nc` session. What happens?
- A) `hello` is echoed, then the server closes the connection; `world` is never echoed
- B) Both lines are echoed
- C) Nothing is echoed until you press Ctrl-D
- D) The server crashes with SIGPIPE

> ✅ **Answer: A.** Version 1 does one `read()`, one `write()` and then `close()`. That is why `echo_server_persistent.c` adds the inner read loop.

**Q4.** In `fork_server.c`, the parent forgets `close(client_fd)` after forking. The child finishes and closes its copy. What is the most likely result?
- A) The client sees the close immediately; nothing else changes
- B) The child is killed with SIGPIPE
- C) The connection stays open because the parent still holds a copy, and the parent leaks one fd per client
- D) `accept()` starts returning the same fd for every client

> ✅ **Answer: C.** The slide tells the parent to "close your copy of the fd". The connection only really closes when every copy is closed, and each forgotten copy uses one of the parent's limited fds.

**Q5.** A server reads a request, sleeps, then writes the reply. The process vanishes during the write with no error and no log. The client had closed with `SO_LINGER {1, 0}`. Which signal is responsible?
- A) SIGKILL (9)
- B) SIGTERM (15)
- C) SIGPIPE (13)
- D) SIGHUP

> ✅ **Answer: C.** Writing to a peer that already closed raises SIGPIPE, whose default action kills the process silently. SIGKILL and SIGTERM are sent by `kill`, not by a write.

**Q6.** What does `struct linger l = {1, 0}` followed by `setsockopt(... SO_LINGER ...)` and `close(fd)` do?
- A) Abortive close: the kernel sends an RST instead of a FIN
- B) Waits 1 second, then sends a FIN
- C) Sends a FIN, then waits 0 seconds in TIME_WAIT
- D) Turns lingering off, so the close is a normal polite close

> ✅ **Answer: A.** Linger on with a 0-second timeout means "abort now": the slide's line is "SO_LINGER {1, 0} sends RST instead of FIN".

**Q7.** `client.c` never calls `bind()` and contains no local port number. How do the server's replies find the client?
- A) The server replies to the client's well-known port 80
- B) The kernel assigns an ephemeral port at `connect()` time, and replies go to it
- C) `gethostbyname()` reserves a port as part of the DNS lookup
- D) Replies go back on the server's own port, 8080

> ✅ **Answer: B.** Clients get a random high port from the ephemeral range (typically 32768–60999 on Linux) when they call `connect()`.

**Q8.** On Linux with `net.core.somaxconn = 4096`, a server calls `listen(fd, 100000)`. What is the effective queue size?
- A) 100000
- B) 1
- C) 1024
- D) 4096

> ✅ **Answer: D.** "Linux caps whatever you ask for at `net.core.somaxconn`", so the number in your code is not the number in effect.

**Q9.** You base64-encode a 100-byte binary file. How many characters do you get (including padding)?
- A) 133
- B) 134
- C) 150
- D) 136

> ✅ **Answer: D.** 3 bytes become 4 characters: 100 bytes = 34 groups (the last one padded), and 34 × 4 = 136. 133 forgets that the output is padded to a whole group of 4.

**Q10.** Using the BCD scheme from the slide (a filler `F` nibble for an odd digit), how is the number 12345 encoded?
- A) `30 39`
- B) `31 32 33 34 35`
- C) `12 34 5F`
- D) `21 43 F5`

> ✅ **Answer: C.** Each digit takes 4 bits, so the hex shows the digits: `12`, `34`, `5F`. `30 39` is plain binary 12345 and `31 32...` is ASCII (twice the size).

**Q11.** A TLV parser (1-byte Type, 1-byte Length) only knows type 1 = name and type 2 = age. It receives `01 03 61 62 63 07 02 FF FF 02 01 15`. What does it extract?
- A) name = "abc", age = 21; type 7 is skipped
- B) name = "abc" only; it stops at the unknown type 7
- C) It fails: unknown types corrupt the stream
- D) name = "abc", age = 255

> ✅ **Answer: A.** The length `02` tells the parser to jump over type 7's two bytes, then `02 01 15` is age = 0x15 = 21. Skipping unknown fields is exactly why TLV is forward compatible.

**Q12.** The sender calls `write("hello")` and then `write("world")` on a TCP socket. Which statement is guaranteed?
- A) Each `read()` on the receiver returns exactly one of the writes
- B) The bytes arrive in order, but where the receiver's reads split them is not guaranteed
- C) The receiver always gets `"helloworld"` in one read
- D) TCP adds a separator between the two writes

> ✅ **Answer: B.** "TCP gives you a byte stream, not messages." The protocol itself has to mark where a message ends (fixed length, delimiter or length prefix).

**Q13.** A protocol uses a 2-byte big-endian length prefix. The header bytes on the wire are `01 2C`. How many body bytes should the receiver read?
- A) 300
- B) 11265
- C) 44
- D) 301

> ✅ **Answer: A.** Big-endian means the most significant byte comes first: 0x012C = 256 + 44 = 300. 11265 (0x2C01) is what you get if you wrongly read it little-endian.

**Q14.** A server holds 10,000 mostly idle connections, and only 3 have data at each wake-up. Why does epoll beat select here?
- A) epoll uses one thread per socket
- B) select only works for UDP sockets
- C) epoll sends data in bigger packets
- D) select rebuilds the fd set every call and the kernel walks all of them (O(n)); epoll registers once and returns only the ready ones (O(ready))

> ✅ **Answer: D.** That is the slide's comparison. select is also capped around 1024 descriptors by `FD_SETSIZE`, so 10,000 is beyond it anyway.

**Q15.** In a single-process epoll server, one request handler runs a 2-second computation. What happens to the other clients?
- A) Nothing; epoll runs each handler in its own process
- B) Only clients on the same port are delayed
- C) All of them wait about 2 seconds, because one slow handler blocks everything
- D) The kernel kills the slow handler with SIGTERM

> ✅ **Answer: C.** "Harder to write, and one slow handler blocks everything" is the cost of one process, many sockets. In a fork server only that one client would wait.

**Q16.** You stop your server with Ctrl-C while clients are connected and restart it at once. `bind()` fails with "address already in use". What is the cause and the fix?
- A) The backlog is full; pass a larger number to `listen()`
- B) The old socket is in TIME_WAIT; set `SO_REUSEADDR`
- C) The port is below 1024; run as root
- D) SIGPIPE killed the old process; ignore SIGPIPE

> ✅ **Answer: B.** The slide calls `SO_REUSEADDR` "the one you will actually use": without it, the restart fails while the old socket sits in TIME_WAIT.

**Q17.** In telnet you type `GET / HTTP/1.1`, then `Host: google.com`, and then you wait without pressing Enter on an empty line. What happens?
- A) Google returns `301 Moved Permanently`
- B) Google returns `400 Bad Request`
- C) telnet sends the blank line for you automatically
- D) Nothing: the server keeps waiting because the request has not ended

> ✅ **Answer: D.** The blank line is mandatory: it is the delimiter that tells the server the headers are finished. Without it, the server waits for more headers.

**Q18.** Which command shows the `HTTP/1.1 301 Moved Permanently` status line and the `Location:` header from google.com together with the body?
- A) `curl -i google.com`
- B) `curl google.com`
- C) `nc -l 9000`
- D) `tcpdump -r 1.pcap -X`

> ✅ **Answer: A.** `-i` includes the response headers. Plain `curl google.com` prints only the small "301 Moved" HTML body.

**Q19.** In `echo_server.c`, which OSI layers do `htons(2026)` and `INADDR_ANY` belong to?
- A) Both are layer 7
- B) Layer 2 and layer 1
- C) Layer 4 (ports) and layer 3 (IP)
- D) Layer 3 and layer 4, respectively

> ✅ **Answer: C.** The slide's table puts TCP, ports and `htons(2026)` at layer 4 and IP, `INADDR_ANY` and `127.0.0.1` at layer 3. D has them the wrong way round.

**Q20.** In `fork_server.c`, the child's `return 0;` is deleted. What happens after the child finishes echoing?
- A) The child exits anyway when `close()` is called
- B) The child falls out of the `if`, loops back and calls `accept()` too, so the number of accepting processes keeps growing
- C) The parent gets SIGPIPE
- D) The child turns into the parent

> ✅ **Answer: B.** The child is a copy running the same `while (1)` loop. Without exiting, it carries on into the loop and starts accepting clients itself.

**Q21.** A server is written as `if (fork()) { close(client_fd); } else { serve(client_fd); }`. The machine runs out of processes. What happens to the next client?
- A) `serve()` runs in the parent
- B) The server crashes with SIGKILL
- C) The client waits in the queue until a process is free
- D) fork returns -1, which counts as true, so the parent closes `client_fd` and the client is dropped without being served

> ✅ **Answer: D.** `-1` is non-zero, so the code treats a failed fork like "I am the parent". This is the case the slide warns "almost nobody checks for".

**Q22.** `/etc/hosts` contains `10.0.0.5 api.example.com`, but public DNS says `93.184.216.34`. What does `gethostbyname("api.example.com")` return?
- A) `10.0.0.5`, because `/etc/hosts` is checked before any network traffic
- B) `93.184.216.34`, because DNS is authoritative
- C) Both addresses, in random order
- D) An error, because the two answers conflict

> ✅ **Answer: A.** The hosts file comes first, "which is why editing hosts overrides everything, and why a stale entry there is so hard to debug".

**Q23.** A mobile app pins its server's certificate. The company replaces that certificate with a new one, correctly signed by a trusted CA. What happens to old app versions?
- A) They accept it, because a trusted CA signed it
- B) They downgrade to plain HTTP
- C) They reject the new certificate until the app is updated
- D) They ask the CA chain to re-pin automatically

> ✅ **Answer: C.** Pinning "removes the CA from the trust equation": the app trusts only that exact certificate, not whatever a CA is willing to sign.

**Q24.** A machine has the IPs 127.0.0.1 and 192.168.1.20. A service binds to `127.0.0.1:5432`. A colleague on the LAN connects to `192.168.1.20:5432`. What happens?
- A) It works, because a socket accepts connections on any IP by default
- B) Connection refused: the socket only listens on the address it was bound to
- C) The connection hangs in the queue
- D) It works, but only if `SO_REUSEADDR` is set

> ✅ **Answer: B.** Only `INADDR_ANY` listens on every interface. "Bind to a specific address when you mean one": 127.0.0.1 means this machine only, so nothing owns port 5432 on 192.168.1.20.

**Q25.** A server's `accept()` starts failing at about 1020 simultaneous clients. `ulimit -n` shows 1024. What is the best explanation?
- A) Every socket is a file descriptor, and the process has hit its fd ceiling
- B) The listen backlog of 1024 is full
- C) The server ran out of ephemeral ports
- D) select's `FD_SETSIZE` limit applies to every server

> ✅ **Answer: A.** `ulimit -n` is "your hard ceiling on open file descriptors. Sockets count." stdin, stdout, stderr and the listening socket use the last few. Ephemeral ports run out on the client side, not the server.
