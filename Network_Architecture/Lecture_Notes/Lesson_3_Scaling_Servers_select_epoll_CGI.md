# 📓 One Process, Many Clients — Scaling Servers
### *CN at Scaler — Lesson 3 (Week 3: Server-side considerations for scale)*

> `select · epoll · CGI · FastCGI` · "and what your favourite server actually does"

---

## ⭐ What this lecture is about

**One question: how do servers scale?** How does one box hold ten thousand clients at the same time?

> 💡 **Simple words mein:** Ek machine, 10,000 clients ek saath. Har client ke liye naya process/thread banaoge toh bahut mehenga. Ek smart loop jo sirf *ready* clients ko serve kare, wahi scale karta hai.

```
  01  MULTIPLEXING        Repeat select / epoll · handling multiple clients · timeouts, retries
  02  CGI / FASTCGI       How the server runs your code
  03  WEB SERVERS?        Router admin · databases · IoT/MQTT · Kafka
  04  LIMITS              ulimit & C10K · memory · context switches · queueing
  05  IN THE WILD         Apache · HAProxy · Varnish · nginx
```

---

## 1 · MULTIPLEXING: three ways to serve many clients

### The headline: "A thread per client is a bet you lose at ten thousand"

Three ways to serve many clients, and only one of them survives at scale.

| Model | What the slide says |
|---|---|
| **Fork per request** | A whole process copied for every client. Simple, isolated, and the most expensive thing you can do per connection. |
| **Thread per request** | Cheaper than fork, so latency drops. But it does not go away: every thread still costs a stack before it does any work. |
| **One loop, many sockets** | `select` or `epoll`. One thread watches every socket and only wakes for the ones that are ready. **This is the one that scales.** |

**What actually ships:** most of the time we do neither purely. We **pre-fork or pre-thread a pool**, and each worker runs its own `select` or `epoll` loop. *"Isolation from the processes, concurrency from the loop."*

> 💡 **Simple words mein:** Fork = har customer ke liye naya restaurant. Thread = naya waiter. Loop = ek smart waiter jo sirf utha hua haath dekh ke serve karta hai. ✅ Real life: kuch waiters pehle se rakho (pre-fork), aur har waiter smart loop chalaye.

---

### 1.1 `fork()`: a whole process per client

The slide's code (an echo server on **port 2026**):

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

        if (fork() == 0) {                       /* CHILD */
            char buf[4096];
            int n;
            while ((n = read(client_fd, buf, sizeof(buf))) > 0) {
                write(client_fd, buf, n);        /* echo back */
            }
            close(client_fd);
            return 0;                            /* child exits */
        }

        close(client_fd);                        /* PARENT closes its copy, loops */
    }
}
```

**Read the code (slide):**
- `listen(server_fd, 1)`: a **backlog of one**.
- The **child** echoes, then exits.
- The **parent** closes its copy and loops.

**What it costs (slide):**
- A full process image per connection.
- Page tables, fds, scheduler entry: all duplicated.
- **You pay it while the user waits.**

---

### 1.2 `select()`: one thread, every client

The slide's code (same echo server, now with `select`):

```c
int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(2026);
    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 16);

    int clients[FD_SETSIZE];
    int nclients = 0;

    while (1) {
        fd_set readfds;
        FD_ZERO(&readfds);
        FD_SET(server_fd, &readfds);
        int maxfd = server_fd;

        for (int i = 0; i < nclients; i++) {
            FD_SET(clients[i], &readfds);
            if (clients[i] > maxfd) maxfd = clients[i];
        }

        select(maxfd + 1, &readfds, NULL, NULL, NULL);

        if (FD_ISSET(server_fd, &readfds)) {
            clients[nclients++] = accept(server_fd, NULL, NULL);
        }

        for (int i = 0; i < nclients; i++) {
            int fd = clients[i];
            if (!FD_ISSET(fd, &readfds)) continue;

            char buf[4096];
            int n = read(fd, buf, sizeof(buf));
            if (n <= 0) {
                close(fd);
                clients[i] = clients[--nclients];
                i--;
            } else {
                write(fd, buf, n);
            }
        }
    }
}
```

**"Two lines worth stopping on" (slide)**

1. `clients[i] = clients[--nclients]; i--;` is an **O(1) delete**: the last element is moved into the empty spot and the count goes down by one. `i--` makes the loop look at index `i` again, because a different fd now sits there.
2. `select(..., NULL, NULL, NULL)`: **that last NULL is the timeout** (see 1.5).

---

### 1.3 select's ceiling is compiled in: `FD_SETSIZE = 1024`

```c
int clients[FD_SETSIZE];   /* not a style choice: this IS the hard limit */
```

- `fd_set` is a **fixed-width bitmap**. On glibc that width is **1024 bits**, so **descriptor 1024 cannot be represented at all**.

| **select: O(n) every call** | **epoll: O(ready)** |
|---|---|
| You **rebuild the set on every iteration** | **Register each fd once**, with `epoll_ctl` |
| The kernel **scans every fd, ready or not** | `epoll_wait` **returns only what actually fired** |
| **10,000 idle sockets cost the same as 10,000 busy ones** | **Idle connections cost nothing** to wait on |

**Same idea, different name (slide):** **kqueue** (BSD, macOS) · **IOCP** (Windows) · **io_uring** (modern Linux).

> 💡 **Simple words mein:** select = teacher jo har baar poori class ka roll call karti hai, chahe 9,990 bachhe so rahe ho. epoll = sirf jinhone haath uthaya, unhi ko sunti hai. 10,000 students ho toh epoll bahut fast.

> **The number to remember: 1024.** It's `FD_SETSIZE`, it's your default `ulimit`, and it's the K in C10K. *One wall, three names.*

---

### 1.4 In production you don't pick one: you stack them

Pre-fork for isolation, pre-thread for cheapness, a loop inside each worker for scale.

```
  AT STARTUP            PER WORKER               ON ACCEPT                    RESULT
 ┌───────────────┐    ┌──────────────────┐    ┌──────────────────────┐    ┌───────────────────┐
 │ fork/spawn a  │ ─▶ │ each runs its own│ ─▶ │ SO_REUSEPORT lets the│ ─▶ │ crash isolation   │
 │ pool of       │    │ select/epoll loop│    │ kernel spread new    │    │ from processes,   │
 │ workers, often│    │ over many sockets│    │ connections across   │    │ concurrency from  │
 │ 1 per CPU core│    │                  │    │ them                 │    │ the loop          │
 └───────────────┘    └──────────────────┘    └──────────────────────┘    └───────────────────┘
```

**This is not theory (slide):** it's **Apache's event MPM**, it's **nginx's master-and-workers**, and it's what the **Go runtime** does for you invisibly: **goroutines multiplexed onto OS threads by a scheduler sitting on top of epoll.**

---

### 1.5 Timeouts are the client's half of the contract

The server may be busy reading, writing, or accepting. **Never wait indefinitely.**

In a select loop you are waiting on three different things:

| Set | What you're waiting for |
|---|---|
| `readfds` | some new content to arrive |
| `writefds` | some socket allowing us to write |
| `exceptfds` | sockets being connected and closed |

Each of those comes with a default timeout. That is the **last argument to `select()`**, and in our code we passed `NULL`.

```c
select(maxfd + 1, &readfds, NULL, NULL, NULL);   /* readfds, writefds, exceptfds, timeout */
```

- **NULL timeout = block forever.** One wedged client and the whole loop (every other connection on it) stops.
- **Client side, same rule:** set a deadline, then **retry with backoff**.

---

## 2 · CGI / FASTCGI: how the web server runs *your* code

### 2.1 CGI: every request forks a new copy of your program

> "Apache was the most popular HTTP server on earth, and this is how it ran your code."

```
                      HTTP request
                           │
                           ▼
                     ┌─────────────┐
                     │ Apache httpd│
                     └──────┬──────┘
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▲
       [ envp ]       [ stdin pipe ]     [ stdout pipe ]
          │                 │                  │
          └──────────▶ ┌─────────────┐ ────────┘
                       │ CGI process │
                       └──────┬──────┘
                              │ stderr
                              ▼
                     [ Apache error log ]
```

### 2.2 Server side: wire up three pipes, then exec

The slide's pseudo-code:

```
Set stdin  for the new process to a pipe/fd we control
Set stdout for the new process to another pipe/fd we control
Set stderr for the new process to yet another pipe/fd we control

Send text received from client     -> stdin
Send text written to stdout        -> client
Send text written to stderr        -> error log
```

**The real thing (slide):** **`apache/httpd` · `modules/generators/mod_cgi.c` L576 · `run_cgi_child()`**. It creates the child, attaches the three pipes, and **sets a timeout on each**, so a hung script cannot hold the worker forever.

### 2.3 HTTP headers become environment variables

The translation layer between a protocol and a process. The slide's pseudo-code:

```
Convert HTTP headers and verb to variables
E.g. REQUEST_METHOD = GET / POST, QUERY_STRING = <things after ?>
Pass these as environment variables while creating the CGI process
```

**The convention (slide):** header `User-Agent: curl` becomes `HTTP_USER_AGENT=curl`. **Upper-case it, swap dashes for underscores, prefix with `HTTP_`.** Every framework you have ever used still carries this shape.

**The real thing (slide):** **`server/util_script.c` L409 · `ap_add_cgi_vars()`** sets `GATEWAY_INTERFACE`, `SERVER_PROTOCOL`, `REQUEST_METHOD`, `QUERY_STRING`. **`ap_add_common_vars()`** above it does the rest.

### 2.4 The CGI program is just a program that prints

No server library. No framework. **Read the environment, write to stdout.** The slide's pseudo-code:

```
CGI program
  Get environment variable (REQUEST_METHOD)   // GET or POST
  If POST - read from stdin
  If GET  - parse another environment variable (QUERY_STRING)
  Output "Content-Type: text/html" - two lines and text
```

**"Two lines" (slide):** the header block, then **a blank line**, then the body. **That blank line is the delimiter**, the same framing question from last session. Apache reads your headers up to it and streams everything after it to the client.

**That is the whole contract.** Any language that can read an environment variable and print to standard output can serve HTTP. That's why the early web was written in **Perl and shell**.

### 2.5 Copy the whole program, then throw it away

Forking and executing is costly, and **CGI does it on every single hit**.

| The copy | The fix people tried | The fix that won |
|---|---|---|
| The whole program image is created, run once, and discarded. Your shell does the same for every command you type, but your shell isn't doing it a thousand times a second. | **Compile the module INTO the server**: `mod_php`, `mod_perl`. No new process per request, but now your app **shares an address space with your web server**, and **one segfault takes both down**. | **Keep a long-lived process of your own** and let the web server talk to it over a **Unix or TCP socket**, with a reduced vocabulary. **That is FastCGI.** |

**Notice the move (slide):** take the setup cost **off the request path** and pay it **once at startup**. Exactly what pre-forking did for connections in block 01.

> 💡 **Simple words mein:** CGI = har order pe naya chef bulao aur ghar bhej do (slow). mod_php = chef restaurant ke andar (fast, par uski galti = poora restaurant band). FastCGI = alag kitchen mein chefs ki team hamesha ready (fast + safe). ✅

### 2.6 FastCGI: keep the process, talk over a socket

You define an **upstream**. The user-facing server handles slow clients; an internal server does the work.

```
   clients                          nginx / httpd                         php-fpm
 ┌──────────────────┐   HTTP     ┌──────────────────────┐   FastCGI   ┌────────────────────┐
 │ thousands of     │──────────▶ │ user-facing.         │───────────▶ │ internal. a fixed  │
 │ them, many of    │            │ absorbs slow         │             │ pool of long-lived │
 │ them slow        │◀────────── │ connections, TLS,    │◀─────────── │ workers            │
 └──────────────────┘            │ buffering            │             └────────────────────┘
                                 └──────────────────────┘
```

- **A modified HTTP, with less vocabulary (slide):** same information (method, headers, query string, body), but **binary, typed, and multiplexed**. No parsing a text protocol twice.
- **What you actually write (slide):** you *may* still get environment variables, but more often you get a **request object** with the same headers and query string, already decoded for you.

### 2.7 A FastCGI request is a stream of typed records

From the slide:

```
FCGI_BEGIN_REQUEST

FCGI_PARAMS
    REQUEST_METHOD   = POST
    SCRIPT_FILENAME  = /var/www/test.php
    QUERY_STRING     = x=10
    CONTENT_TYPE     = application/x-www-form-urlencoded
    CONTENT_LENGTH   = 17
    HTTP_HOST        = localhost
    HTTP_USER_AGENT  = curl/...

FCGI_PARAMS
    <empty record means parameters finished>

FCGI_STDIN
    name=alice&age=20

FCGI_STDIN
    <empty record means stdin finished>
```

- **Same nouns as CGI (slide):** `REQUEST_METHOD`, `QUERY_STRING`, `CONTENT_LENGTH`. *The CGI environment survived.* Only the transport changed: **records on a socket instead of envp on a fork.**
- **STDIN is the body (slide):** `name=alice&age=20` is **seventeen bytes**, exactly the `CONTENT_LENGTH` declared above it.

### 2.8 …and the response comes back the same way

```
FCGI_STDOUT
    Content-Type: text/html

    <html>...</html>
FCGI_STDOUT
    <empty>

FCGI_STDERR
    ...

FCGI_END_REQUEST
```

- **STDOUT goes to the client:** headers, blank line, body, **byte for byte what a CGI program printed**. The web server parses off the headers and streams the rest.
- **STDERR goes to the log:** a **separate record type**, so your debug output **can never corrupt the page**. The same split as the pipes in the CGI diagram.

### 2.9 The empty record is the delimiter

> Last session ended on framing: **length or delimiter, no third option.** Here they are, in one protocol.

| LENGTH | DELIMITER |
|---|---|
| Every record header carries its own content length. The reader knows exactly how many bytes to take before it takes them, so a body can contain **any byte at all, including newlines and nulls**. | A record of **length zero** means "that stream is finished". Not a byte pattern in the data: a **structural marker that cannot collide with your content**. |
| `CONTENT_LENGTH = 17` does the same job one level up. | The **blank line after `Content-Type`** is the same trick in text. |

> **SMS chose length. SMTP chose a dot. IMAP chose both. FastCGI chose both too, and for the same reason.**

> 💡 **Simple words mein:** Length = "is dabbe mein 17 cheezein hain". Delimiter = "khaali dabba aaya, matlab khatam". FastCGI dono use karta hai.

### 2.10 Servlets are the same idea wearing a JVM

Another way to avoid making a different program per request.

| The server is the runtime | Pre-threading | Isolation by convention |
|---|---|---|
| **Jetty** and **Tomcat** act as the web server themselves. No separate httpd in front doing the forking: the JVM is already running your code. | A **pool of threads created at startup**. A request is handed to an idle thread, which runs your servlet and returns to the pool. Exactly the pre-threading from block 01. | Servlets are isolated by **class loader and thread, not by process**. Cheaper than fork, and **one runaway servlet can still hurt its neighbours**. |

> **CGI, FastCGI, servlets, and every modern framework are answering one question: how do we run user code per request without paying to create the world each time?**

---

## 3 · ISN'T EVERYTHING A WEB SERVER?

### 3.1 Almost everything runs a web server

Not everything *is* one, and **the difference decides which scaling tricks you get**.

| ✅ YES: it speaks HTTP | ❌ NO, and it doesn't need to |
|---|---|
| Your **Wi-Fi router admin page** | **Redis**: its own line protocol, **RESP** |
| Modern **agent runtimes** and cloud tooling | **MySQL**: its own binary wire protocol |
| Most **Node** applications | **MQTT brokers** across the whole IoT estate |
| **Firestore** (the popular one) | **Messenger backends**: WhatsApp-shaped things |
| **CouchDB**, which has had REST for years | **Kafka natively**: a binary protocol, not HTTP |
| **Kafka**, if you put the **REST proxy** in front | |

Everything you'll learn about scaling web servers **assumes HTTP's properties**. Change the protocol and the advice changes with it.

### 3.2 HTTP is stateless. What you scale next may not be.

| A web server can hang up | A database wants to stay |
|---|---|
| **Every request carries everything it needs.** So any box can answer any request, you can close the connection when you like, and a load balancer can send the next one anywhere. | **Redis or MySQL would rather hold a persistent connection** than pay for a **TCP connect + handshake + auth + session setup** on every query. |
| **Horizontal scaling is free-ish.** Add a server, add it to the pool, done. *The statelessness is what makes that legal.* | **So the problem changes shape.** The question is no longer requests per second. It's **"how many persistent connections can one box hold at once?"** (block 04) |

---

## 4 · LIMITS: four walls, and you will hit all of them

> Every scaling technique in this course is a way of dodging one of these.

```
 ┌───────────────────────────┬───────────────────────────┐
 │ 01  FILE DESCRIPTORS      │ 02  MEMORY                 │
 │ A connection is an fd.    │ A stack per thread, a      │
 │ fds are rationed.         │ buffer per connection.     │
 │ Default is still 1024.    │ Paid before any work.      │
 ├───────────────────────────┼───────────────────────────┤
 │ 03  CONTEXT SWITCHES      │ 04  THE QUEUE              │
 │ Every read/write crosses  │ Latency and throughput are │
 │ the kernel boundary. So   │ not independent. Being     │
 │ does every syscall.       │ slow costs you memory.     │
 └───────────────────────────┴───────────────────────────┘
```

### 4.1 Wall 1 · C10K was 1999's problem. Your default ulimit is still 1024.

| Year | Name | Number | What the slide says |
|---|---|---|---|
| **1999** | **C10K** | **10,000** | **Dan Kegel** names the problem: can one box hold ten thousand simultaneous persistent connections? At the time, **no**. |
| **2013** | **C10M** | **10,000,000** | **Robert Graham** argues *the kernel is the problem, not the solution*. Bypass it and the number moves by three orders of magnitude. |
| **Today** | | **2–3 M** | People have demonstrated two to three million connections on a single server. *The problem moved; the default did not.* |

```sh
$ ulimit -n
1024        # the soft limit on most distributions, unchanged since the problem was named
```

### 4.2 Wall 2 · Every thread costs memory before it does any work

| OS thread | Goroutine (Go) |
|---|---|
| **8 MB** default stack, **reserved up front** | **2 KB** initial stack, **grown on demand** |
| **10,000 threads reserve roughly 80 GB of address space** before a single byte of your data exists. | **Four thousand times smaller.** This single number is most of why **Go took over this problem space**. |

Maths: 10,000 × 8 MB = 80 GB. 8 MB / 2 KB ≈ 4,000.

**And underneath, it's the same loop (slide):** goroutines are lightweight because the Go runtime **multiplexes them onto a few OS threads using a netpoller built on epoll**, and runs **its own scheduler** on top. **You write blocking reads; the runtime parks you on a poll.**

> **The more requests you handle at the same time, the more memory you need. The only question is how much per request.**

### 4.3 Wall 3 · Every read and write is a trip across the kernel boundary

> In fact, every system call is. At a million operations a second, **the tax is the workload.**

| The tax | Bypass the kernel | Or make fewer calls |
|---|---|---|
| User space → kernel space and back: **save state, switch page tables, validate, copy, return**. **Hundreds of nanoseconds** each, and **Spectre and Meltdown mitigations made it worse**, not better. | Some people implemented **TCP in user space** and bypassed the kernel entirely, for well-known internal networks: **DPDK, netmap, mTCP, Seastar**. You **give up every tool you own**, and you get the packets. | **`sendfile()`** moves a file to a socket **without a user-space round trip**. **`io_uring`** submits a **batch** of operations through a **shared ring** instead of one syscall each. |

> **Remember `sendfile`**: it's the reason nginx serves static files the way it does (block 05).

### 4.4 Wall 4 · Concurrency isn't a setting, it's throughput × latency

```
   connections in flight   =   requests per second   ×   seconds per request
          L                =           λ             ×           W
```

(This is the **Little's Law** named in homework 4.)

- **Ten times the cost (slide):** at **100 ms** you hold **a thousand** connections. At **one second**, **ten thousand**, *for identical traffic*.
- **Slowness is a memory bug (slide):** that's why **high-latency work goes asynchronous**: take it off the request path and the queue drains.

---

## 5 · IN THE WILD: what does our favourite software do?

### 5.1 Apache: prefork → worker → event

> It took twenty years, and it ended up at a loop too.

| PREFORK MPM (Multi-process module) | WORKER MPM | EVENT MPM |
|---|---|---|
| A pool of **processes**, one connection each. Bulletproof, compatible with every non-thread-safe module ever written, and the **most expensive per connection**. | Processes each running **many threads**. Cheaper per connection, but a **thread is still pinned to a connection, even while it sits idle on keep-alive**. | A dedicated **listener thread per process** parks idle keep-alive sockets **on epoll** and hands a worker thread **only the sockets with actual work**. **The default on Unix today.** |

**The keep-alive problem, in one line (slide):** under **worker**, a client that has finished its request but holds the connection open **still owns a thread**. **Event takes the socket back and gives the thread away.** That's our select loop, arriving inside Apache in **2012**.

> 💡 **Simple words mein:** Worker MPM = customer ne khana kha liya, par table pe baitha hai, aur waiter uske paas khada hai, kuch nahi kar raha. Event MPM = waiter doosre customers ke paas chala jaata hai, aur listener (epoll) dekhta rehta hai ki ye customer dobara kab order karega.

### 5.2 HAProxy: a select loop from before epoll existed

| The timeline explains the design | It introduced the load balancer |
|---|---|
| HAProxy arrives in **2001**. **epoll only lands in Linux 2.5.44 in late 2002.** So it was largely **select**, because select was what there was. It moved to epoll later, and only became **multi-threaded in 1.8, in 2017**. | For a lot of teams this was **the first box that sat in front of the application servers and chose between them**, and could switch between multiple upstream services. |
| **One process, enormous throughput:** for most of its life **a single event-driven process saturated a NIC**. *The loop argument made as loudly as it can be made.* | **It absorbs connection churn:** it can reduce connects and disconnects on the upstream service **by holding a pool open**. The persistent-connection argument from block 03, applied by a proxy for you. |

### 5.3 Varnish: the first popular caching reverse proxy

| Squid cached for the client | How Varnish actually runs |
|---|---|
| **Squid** was a caching proxy that sat **next to the users, on the way out**. **Varnish turned it around** and put the cache **in front of the origin, on the way in**. Same mechanism, opposite direction, entirely different business. | An **acceptor thread per listening point per pool**. A **worker thread per active connection**, drawn from pools that default to a **minimum of 100 and a maximum of 5000**. **Idle descriptors** (clients between requests, backends between fetches) are handed to a **waiter**: **epoll on Linux, kqueue on BSD**. |

**Still everywhere:** **Fastly's edge is built on Varnish, VCL and all.** When you write a **VCL** snippet for a CDN in 2026, you're configuring a program written in **2006**.

> **So: threads for the work, a loop for the waiting.** The same split Apache arrived at with the event MPM.

### 5.4 nginx: epoll plus sendfile, never copy the bytes at all

The slide's code:

```c
read(file_fd, buf, n);                 /* disk -> kernel -> your buffer   */
write(sock_fd, buf, n);                /* your buffer -> kernel -> socket */

sendfile(sock_fd, file_fd, NULL, n);   /* kernel -> socket. you never touch it. */
```

| Master and workers | The first reverse proxy | And a thread pool, later |
|---|---|---|
| **One master**, then **one worker process per CPU core**. Each worker is a **single thread running one epoll loop over thousands of connections**. | Then promptly used for **caching, load balancing, TLS termination, rate limiting** and everything else. **It became the default shape of an edge.** | **Disk reads can block even in a loop.** Since **1.7.11** nginx offloads those to a **thread pool**, because **a page-fault on a cache miss stalls every connection on that worker**. |

### 5.5 All the options side by side

| | Concurrency model | Born as | The idea it popularised | In the wild |
|---|---|---|---|---|
| **Apache** | prefork → worker → event (listener thread + epoll) | **1995**, the web server | Modules, and CGI as a way to run anyone's code | Everywhere. Akamai started here |
| **HAProxy** | select, then epoll; multi-threaded only since 1.8 | **2001**, TCP/HTTP proxy | The load balancer as a tier of its own | The default LB in front of everything |
| **Varnish** | thread per active connection + epoll waiter for idle | **2006**, HTTP accelerator | Caching reverse proxy: Squid, turned around | Fastly's edge, VCL and all |
| **nginx** | master + worker per core, one epoll loop each | **2004**, answer to C10K | sendfile: never copy the bytes into user space | The default edge. Broadpeak, and most CDNs |

> **Read the second column top to bottom: four independent teams, four starting points, and every one ended up with an event loop and a small pool of workers.**

### 5.6 Go and read the actual code

| Repo | File | What's there |
|---|---|---|
| `apache/httpd` | `modules/generators/mod_cgi.c` L576 | `run_cgi_child()`: fork, three pipes, a timeout on each |
| `apache/httpd` | `server/util_script.c` L409 | `ap_add_cgi_vars()`: HTTP headers become the environment |
| `nginx/nginx` | `src/event/modules/ngx_epoll_module.c` | The epoll loop itself |
| `nginx/nginx` | `src/os/unix/ngx_linux_sendfile_chain.c` | `sendfile()`, where the copy does not happen |
| `haproxy/haproxy` | `src/ev_epoll.c` | The poller, one file per platform (`ev_select.c` still sits beside it) |
| `varnishcache/varnish-cache` | `bin/varnishd/waiter/cache_waiter_epoll.c` | The waiter that holds idle connections |
| `golang/go` | `src/runtime/netpoll_epoll.go` | Where your blocking read actually parks |

---

## 6 · HOMEWORK (with solutions)

> Run these on **Linux** (a VM or `docker run -it --rm --ulimit nofile=200000:200000 ubuntu bash`); epoll, `ss` and `/proc` don't exist on macOS. Inside the container: `apt update && apt install -y build-essential python3 iproute2`.

### HW 1 · Break the select server
> Take the select code (1.2), open **1,100 connections** to it, find where it dies and prove why. Then rewrite the loop with **epoll** and do it again with **100,000**.

**✅ Solution**

1. Save the 1.2 code as `select_server.c` (add `#include <sys/select.h>`, `<unistd.h>`, `<arpa/inet.h>` at the top) and run it:
   ```sh
   gcc -O2 -D_FORTIFY_SOURCE=2 select_server.c -o select_server
   ulimit -n 4096 && ./select_server
   ```
2. Client `flood.py` (opens N connections and holds them):
   ```python
   import socket, sys, time
   N, socks = int(sys.argv[1]), []
   for i in range(N):
       s = socket.socket()
       s.bind((f"127.0.0.{2 + i // 20000}", 0))   # new source IP every 20k → no port shortage
       s.connect(("127.0.0.1", 2026)); socks.append(s)
       if i % 100 == 0: print(i, flush=True)
   print("all open"); time.sleep(3600)
   ```
   Run in a 2nd terminal: `ulimit -n 200000; python3 flood.py 1100`
3. **What you see:** around client #1021 the server gets fd **1024** and aborts with `*** buffer overflow detected ***`. **Why:** fds 0–2 + listener = 4 fds, so the 1021st client is fd 1024, which `fd_set` (1024 bits, `FD_SETSIZE`) cannot represent; `clients[FD_SETSIZE]` would also overflow. Prove it: `ls /proc/$(pgrep select_server)/fd | wc -l` just before the crash shows ~1024. (With the default `ulimit -n 1024` instead, `accept()` fails at the same point with "Too many open files", the same 1024 wall under its other name.)
4. epoll rewrite, `epoll_server.py`:
   ```python
   import socket, select
   srv = socket.socket(); srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
   srv.bind(("0.0.0.0", 2026)); srv.listen(4096)
   ep = select.epoll(); ep.register(srv.fileno(), select.EPOLLIN)   # register once
   conns = {}
   while True:
       for fd, _ in ep.poll():                       # only the ready fds come back
           if fd == srv.fileno():
               c, _ = srv.accept(); conns[c.fileno()] = c
               ep.register(c.fileno(), select.EPOLLIN)
           else:
               c = conns[fd]
               try: data = c.recv(4096)
               except ConnectionError: data = b""
               if data: c.send(data)
               else: ep.unregister(fd); c.close(); del conns[fd]
   ```
   Run `ulimit -n 200000; python3 epoll_server.py`, then `python3 flood.py 100000`. **What you see:** it prints `all open` and the server keeps working; `ss -s` shows ~100,000 established. The only limit now is `ulimit -n` and memory, not 1024.

### HW 2 · Watch CGI cost you
> Write a CGI script, run it under Apache with **mod_cgi**, keep `ps`/`top` open while you `curl` it in a loop, and count processes appearing and disappearing.

**✅ Solution**
```sh
apt update && apt install -y apache2 curl procps
a2enmod cgi && service apache2 start          # on a threaded MPM this enables mod_cgid (same idea)
cat > /usr/lib/cgi-bin/hello.cgi <<'EOF'
#!/bin/sh
sleep 0.2
echo "Content-Type: text/plain"
echo ""
echo "pid=$$"
EOF
chmod +x /usr/lib/cgi-bin/hello.cgi
while true; do curl -s localhost/cgi-bin/hello.cgi; done      # terminal 1
watch -n 0.2 'ps -ef | grep [h]ello.cgi'                      # terminal 2
```
**What you see / write down:** terminal 1 prints `pid=` with a **new, increasing PID on every request**; terminal 2 shows a `hello.cgi` process that appears and vanishes about 5 times a second. One request = one fork + exec + exit: exactly "copy the whole program, then throw it away" (2.5).

### HW 3 · Find the FastCGI records
> Run **php-fpm** behind **nginx**, `tcpdump` the loopback socket between them, and find **`FCGI_BEGIN_REQUEST`** and the **empty PARAMS record** in the bytes.

**✅ Solution**
```sh
DEBIAN_FRONTEND=noninteractive apt install -y nginx php-fpm tcpdump curl
sed -i 's|^listen = .*|listen = 127.0.0.1:9000|' /etc/php/*/fpm/pool.d/www.conf   # TCP, so tcpdump can see it
cat > /etc/nginx/conf.d/fcgi.conf <<'EOF'
server { listen 8080; root /var/www/html;
  location ~ \.php$ { include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass 127.0.0.1:9000; } }
EOF
echo '<?php echo "hi"; ?>' > /var/www/html/test.php
service $(ls /etc/init.d | grep fpm) start; service nginx start
tcpdump -i lo -X -s0 'tcp port 9000'                                 # terminal 2
curl -d 'name=alice&age=20' 'localhost:8080/test.php?x=10'           # terminal 1
```
**What to find** (each record starts with an 8-byte header: version, type, request id (2 bytes), content length (2 bytes), padding, reserved):
- `01 01 00 01 00 08 ...` → type **1 = FCGI_BEGIN_REQUEST**, request 1.
- `01 04 00 01 ...` → FCGI_PARAMS; the ASCII column shows `REQUEST_METHOD POST`, `QUERY_STRING x=10`, `CONTENT_LENGTH 17`.
- `01 04 00 01 00 00 00 00` → **the empty PARAMS record** (length 0 = params finished).
- `01 05 00 01 00 11 ...` → FCGI_STDIN, length `0x11` = **17** = `name=alice&age=20`; then `01 05 00 01 00 00` = empty STDIN.
- Reply direction: `01 06` (STDOUT with `Content-Type` + `hi`), then `01 03` (END_REQUEST).

### HW 4 · Measure Little's Law on something you own
> Pick a service, note **requests/second** and **p50 latency**, predict connections in flight, check with **`ss -s`**, then slow it down and see if the prediction holds.

**✅ Solution** (a toy service where you control W, and a fixed-rate load so λ is fixed)
```python
# server.py — usage: python3 server.py 0.05   (W = 50 ms)
import sys, time
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
W = float(sys.argv[1])
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        time.sleep(W); self.send_response(200); self.end_headers(); self.wfile.write(b"ok")
    def log_message(self, *a): pass
ThreadingHTTPServer(("", 8080), H).serve_forever()
```
```python
# load.py — usage: python3 load.py 100   (λ = 100 req/s)
import sys, time, threading, urllib.request
rate = int(sys.argv[1])
hit = lambda: urllib.request.urlopen("http://127.0.0.1:8080/").read()
while True:
    threading.Thread(target=hit, daemon=True).start(); time.sleep(1 / rate)
```
Measure while `load.py` runs: `watch -n1 "ss -tn state established '( sport = :8080 )' | tail -n +2 | wc -l"` (plus `ss -s` for the summary). p50: `curl -o /dev/null -s -w '%{time_total}\n' localhost:8080` ≈ W.

| Run | λ | W (p50) | Predicted L = λ × W | Measured (`ss`) |
|---|---|---|---|---|
| normal | 100/s | 0.05 s | **5** | ≈ 5 |
| slowed (`server.py 0.5`) | 100/s | 0.5 s | **50** | ≈ 50 |

**Write down:** same traffic, 10× latency → 10× connections held. The prediction holds: "slowness is a memory bug" (4.4).

---

## 7 · THE FOUR IDEAS (closing slide): "The loop is the architecture"

| # | Idea | In one line |
|---|---|---|
| 01 | **Amortise the setup** | Pre-fork, pre-thread, FastCGI, connection pools, servlets: all the same move. *Take the expensive setup off the request path and pay it once at startup.* |
| 02 | **Nothing is free per connection** | A descriptor, a stack, a buffer, a scheduler entry. *Multiply by ten thousand before you choose a design.* That multiplication is the whole discipline. |
| 03 | **Framing is still length or delimiter** | FastCGI uses a length inside a record and an empty record between them. Same two options as SMS and SMTP. *There is never a third.* |
| 04 | **Concurrency = throughput × latency** | You cannot configure your way past this one. *Getting faster is the only thing that reduces what you are holding.* |

---

## 8 · CHEAT-SHEET 📋

| Term | One-line meaning (from the slides) |
|---|---|
| **fork per request** | Whole process copied per client: simple, isolated, most expensive |
| **thread per request** | Cheaper than fork, but every thread still costs a stack (8 MB reserved) |
| **pre-fork / pre-thread** | Pool created at startup; each worker runs its own select/epoll loop |
| **`select()`** | One thread, every client; rebuild the set each iteration; kernel scans every fd: O(n) |
| **`FD_SETSIZE`** | 1024-bit `fd_set` on glibc: fd 1024 cannot be represented |
| **`epoll`** | Register once (`epoll_ctl`); `epoll_wait` returns only what fired: O(ready) |
| **kqueue / IOCP / io_uring** | Same idea on BSD+macOS / Windows / modern Linux |
| **`SO_REUSEPORT`** | Lets the kernel spread new connections across workers |
| **select timeout (last arg)** | `NULL` = block forever: one wedged client stops the whole loop |
| **client rule** | Set a deadline, then retry with backoff |
| **CGI** | Fork + exec per request; headers → envp, body → stdin, stdout → client, stderr → error log |
| **`HTTP_USER_AGENT`** | Upper-case, dashes → underscores, prefix `HTTP_` |
| **blank line** | Delimiter between CGI headers and body |
| **mod_php / mod_perl** | Module inside the server: shared address space, one segfault takes both down |
| **FastCGI** | Long-lived process behind a Unix/TCP socket; binary, typed, multiplexed records |
| **FCGI records** | BEGIN_REQUEST, PARAMS, STDIN, STDOUT, STDERR, END_REQUEST; empty record ends a stream |
| **servlets** | Jetty/Tomcat are the server; pre-threaded; isolated by class loader + thread |
| **stateless HTTP** | Any box answers any request → horizontal scaling free-ish |
| **persistent connections** | Redis/MySQL: question becomes how many connections one box can hold |
| **C10K / C10M** | Kegel 1999 / Graham 2013 (bypass the kernel); today 2–3 M |
| **8 MB vs 2 KB** | OS thread stack vs goroutine initial stack (~4000×) |
| **netpoller** | Go's epoll-based poller that parks blocking reads |
| **kernel tax** | Hundreds of ns per syscall; worse after Spectre/Meltdown |
| **DPDK / netmap / mTCP / Seastar** | User-space TCP, bypassing the kernel |
| **`sendfile()` / `io_uring`** | File → socket without user-space copy / batch ops through a shared ring |
| **L = λ × W** | Connections in flight = requests/s × seconds/request |
| **Apache MPMs** | prefork (process/conn) → worker (thread/conn) → event (listener + epoll, 2012) |
| **HAProxy** | 2001, select then epoll (Linux 2.5.44, 2002), multi-threaded since 1.8 (2017) |
| **Varnish** | 2006; worker threads 100–5000 + epoll/kqueue waiter; Fastly + VCL |
| **nginx** | 2004; master + one epoll worker per core; sendfile; thread pool since 1.7.11 |

**Key numbers:** `1024` · `2026` (port) · `8 MB` / `2 KB` · `80 GB` · `2.5.44` (epoll, 2002) · `1999 / 2013` · `2–3 M` · `100 / 5000` · `1.8` (2017) · `1.7.11` · `17` bytes.

---

## 9 · MCQs (mid → advanced)

**Q1.** A developer "optimises" the select server by building the `fd_set` once before `while (1)` instead of inside it. What is the problem?
- A) `select()` will now return immediately with an error every time
- B) The listening socket can no longer accept new clients, but old clients keep working
- C) The set must be rebuilt on every iteration; reusing one set means clients get missed and new clients are never watched
- D) Nothing; building the set once is how epoll works too

> ✅ **Answer: C.** The slide says that with select "you rebuild the set on every iteration". Rebuilding once per call is exactly what makes select O(n). Registering fds once is how epoll works (`epoll_ctl`), not select, so D is wrong.

**Q2.** `clients = [4, 5, 6, 7]`, `nclients = 4`. At `i = 1`, fd 5 returns `n <= 0`. After `clients[i] = clients[--nclients]; i--;`, what is the array and why is `i--` needed?
- A) `[4, 7, 6]`; fd 7 now sits at index 1 and has not been checked yet
- B) `[4, 6, 7]`; the array is shifted left, and `i--` prevents a crash
- C) `[4, 7, 6, 7]`; `i--` cleans up the duplicate entry
- D) `[4, 6, 7]`; `i--` restarts the loop from the beginning

> ✅ **Answer: A.** The O(1) delete moves the last element (7) into the gap and shrinks the count to 3. Without `i--`, the loop's `i++` would skip fd 7 this round. There is no shifting, so B and D are wrong.

**Q3.** A select-based server watches only 5 sockets, but one of them happens to be fd 1024. What does the lecture say about this?
- A) It works, because the limit is on how many fds you watch, not their numbers
- B) It works, but select becomes O(n²)
- C) select automatically falls back to epoll
- D) It cannot work: `fd_set` is a 1024-bit bitmap, so descriptor 1024 cannot be represented at all

> ✅ **Answer: D.** The limit is the fixed bitmap width (FD_SETSIZE = 1024 on glibc). It limits the fd's number, so even 5 sockets fail if one of them is fd 1024. A is the tempting wrong answer.

**Q4.** A chat server holds 10,000 connections, and only 10 of them send data in a given second. Compare the per-call kernel work of select and epoll.
- A) Both scan 10,000 fds; epoll is only faster because it uses threads
- B) select scans all 10,000 fds every call; epoll_wait returns only the 10 that fired
- C) select scans only the 10 ready fds; epoll scans all 10,000 once at registration
- D) Both are O(ready), but select is capped at 1024 ready fds

> ✅ **Answer: B.** select is O(n) every call: "10,000 idle sockets cost the same as 10,000 busy ones". epoll is O(ready): "idle connections cost nothing to wait on".

**Q5.** Why does the lecture say the fork server's cost is paid "while the user waits"?
- A) Because `listen(server_fd, 1)` makes clients wait for one another
- B) Because the child must finish before the parent can reply
- C) Because the fork (copying the process image, page tables, fds, scheduler entry) happens after `accept()`, on the request path
- D) Because the parent never closes its copy of `client_fd`

> ✅ **Answer: C.** The process is copied only once a client has connected, so the copying delays that client. Pre-forking moves this cost to startup. D is false: the parent does close its copy.

**Q6.** In production, which mix gives "isolation from the processes, concurrency from the loop"?
- A) One process with one thread per client
- B) A pool of pre-forked workers (often one per core), each running its own select/epoll loop, with SO_REUSEPORT spreading new connections
- C) One fork per request, with each child running epoll
- D) One giant select loop with no workers

> ✅ **Answer: B.** This is the "stack them" slide: startup pool → a loop per worker → SO_REUSEPORT on accept. C still forks per request, so it pays the cost the pool is meant to avoid.

**Q7.** A select loop passes `NULL` as the last argument, and one client gets wedged. What happens?
- A) Only that client's connection hangs; the kernel serves the others
- B) select returns an error after the default 30 s timeout
- C) The kernel closes the wedged socket automatically
- D) `NULL` means block forever, so the whole loop stops, along with every other connection on it

> ✅ **Answer: D.** The last argument is the timeout, and NULL means block forever. One thread serves everyone, so one wedged client stalls them all. The client-side rule is the mirror: set a deadline, then retry with backoff.

**Q8.** How does a Go program get "blocking-style" code while still scaling like an event loop?
- A) The runtime multiplexes goroutines onto a few OS threads using a netpoller built on epoll; a blocking read parks the goroutine on a poll
- B) Each goroutine gets its own 8 MB OS thread, but the kernel shares the stacks
- C) Go forks a process per goroutine using copy-on-write
- D) Go uses select with a larger FD_SETSIZE

> ✅ **Answer: A.** The slide says: "You write blocking reads; the runtime parks you on a poll." Goroutines start with a 2 KB stack, not an 8 MB thread, so B is wrong.

**Q9.** A service uses 25,000 OS threads, one per connection, each with the default stack. Roughly how much address space is reserved for stacks, and what would goroutines need instead?
- A) 25 GB vs 25 MB
- B) 200 GB vs 200 MB
- C) 200 GB vs about 50 MB
- D) 80 GB vs about 20 MB

> ✅ **Answer: C.** 25,000 × 8 MB = 200,000 MB ≈ 200 GB. 25,000 × 2 KB = 50,000 KB ≈ 50 MB (about 4,000× smaller). D uses the slide's 10,000-thread figure by mistake.

**Q10.** A service handles a steady 2,000 req/s at 50 ms. Its database slows down, so each request now takes 500 ms. Traffic is unchanged. How many connections are in flight now?
- A) 100 → 100 (traffic didn't change)
- B) 100 → 1,000
- C) 1,000 → 10,000
- D) 100 → 200

> ✅ **Answer: B.** L = λ × W: 2,000 × 0.05 = 100, then 2,000 × 0.5 = 1,000. A is the trap: concurrency is not a setting, so latency alone multiplies it by 10.

**Q11.** `ss -s` shows about 300 connections in flight on a server doing 1,500 req/s. What is the average time per request?
- A) 5 ms
- B) 50 ms
- C) 450 s
- D) 200 ms

> ✅ **Answer: D.** W = L / λ = 300 / 1,500 = 0.2 s = 200 ms. C comes from multiplying instead of dividing.

**Q12.** By the slides, what is the right fix for "slowness is a memory bug" when a request does slow work?
- A) Make high-latency work asynchronous: take it off the request path so the queue drains
- B) Raise `ulimit -n`, which reduces the number of connections in flight
- C) Switch from epoll to select
- D) Add more threads so latency stops affecting concurrency

> ✅ **Answer: A.** "You cannot configure your way past" L = λW; getting faster is the only thing that reduces what you hold. Raising limits (B) only lets you hold more; it doesn't hold less.

**Q13.** An HTTP request carries `X-Forwarded-For: 1.2.3.4`. Using the CGI convention, which environment variable will the script see?
- A) `X_FORWARDED_FOR=1.2.3.4`
- B) `HTTP-X-FORWARDED-FOR=1.2.3.4`
- C) `HTTP_X_FORWARDED_FOR=1.2.3.4`
- D) `HTTP_x_forwarded_for=1.2.3.4`

> ✅ **Answer: C.** Upper-case it, swap dashes for underscores, prefix with `HTTP_`. B keeps the dashes and D isn't upper-cased.

**Q14.** For `POST /f.cgi?x=10` with body `name=alice&age=20`, where does a CGI program find each piece?
- A) Both in `QUERY_STRING`
- B) Body in `REQUEST_METHOD`, query on stdin
- C) Both on stdin, separated by a blank line
- D) Query in the `QUERY_STRING` env variable, body on stdin

> ✅ **Answer: D.** The slide's pseudo-code: "If POST, read from stdin; if GET, parse QUERY_STRING". The text after `?` always goes to `QUERY_STRING`.

**Q15.** A CGI script prints a debug message to stderr. Where does it end up?
- A) In the HTTP response body, after the headers
- B) In the Apache error log, through the separate stderr pipe
- C) Nowhere: CGI ignores stderr
- D) In the `HTTP_STDERR` environment variable

> ✅ **Answer: B.** Apache wires three pipes: stdin ← client body, stdout → client, stderr → error log. FastCGI keeps the same split with the separate FCGI_STDERR record type.

**Q16.** With mod_php, a PHP extension segfaults. With php-fpm behind nginx, the same bug happens. Which comparison is correct?
- A) mod_php: the app shares the web server's address space, so the crash takes the server process down too; FastCGI: only the separate long-lived PHP worker dies
- B) Both take down the web server, because PHP runs inside it in both setups
- C) mod_php survives because it forks per request
- D) FastCGI crashes nginx because they share a socket

> ✅ **Answer: A.** The slide's "fix people tried" (mod_php/mod_perl) means "one segfault takes both down". FastCGI keeps your own process behind a Unix or TCP socket, so the crash stays isolated.

**Q17.** What does the CGI → mod_php → FastCGI story have in common with pre-forking?
- A) All of them fork once per request
- B) All of them use the blank line as a delimiter
- C) All of them take the setup cost off the request path and pay it once at startup
- D) All of them need HTTP to be stateful

> ✅ **Answer: C.** "Notice the move... exactly what pre-forking did for connections in block 01." This is closing idea 01, "Amortise the setup".

**Q18.** In a FastCGI request, how does php-fpm know that all parameters have arrived?
- A) It sees a blank line inside the FCGI_PARAMS content
- B) The CONTENT_LENGTH parameter tells it how many params there are
- C) The connection closes
- D) An empty FCGI_PARAMS record (length zero) arrives

> ✅ **Answer: D.** "Empty record means parameters finished." The blank line (A) is the CGI header/body delimiter in text. CONTENT_LENGTH (B) is the body size.

**Q19.** Why can a FastCGI body safely contain newlines and null bytes?
- A) FastCGI escapes them like SMTP
- B) Every record header carries its own content length, so the reader knows exactly how many bytes to take
- C) The body is base64-encoded
- D) The empty record is a special byte pattern that never appears in data

> ✅ **Answer: B.** Length framing means content is never scanned for a marker. The empty record is a structural marker, not a byte pattern, so D is wrong.

**Q20.** Which protocol, according to the slides, uses only a delimiter to end a message?
- A) SMTP (a dot)
- B) SMS
- C) IMAP
- D) FastCGI

> ✅ **Answer: A.** "SMS chose length. SMTP chose a dot. IMAP chose both. FastCGI chose both too."

**Q21.** A Tomcat app has a runaway servlet in an infinite loop. Why can it still hurt other servlets, even though Tomcat is pre-threaded?
- A) Tomcat forks a new process per request, which copies the bug
- B) Servlets talk to Tomcat over FastCGI
- C) Servlets are isolated by class loader and thread, not by process
- D) Each servlet has its own JVM, but they share one port

> ✅ **Answer: C.** "Isolation by convention": cheaper than fork, but "one runaway servlet can still hurt its neighbours". A and D describe process isolation, which servlets don't have.

**Q22.** Why can you add web servers behind a load balancer easily, while scaling Redis/MySQL "changes shape"?
- A) HTTP is binary and databases are text
- B) HTTP is stateless (any box can answer any request); databases prefer persistent connections, so the question becomes how many connections one box can hold
- C) Databases cannot run behind a load balancer
- D) HTTP servers use select, databases use epoll

> ✅ **Answer: B.** Statelessness is "what makes horizontal scaling legal". Databases would rather keep a connection than pay connect + handshake + auth + session setup on every query.

**Q23.** On Apache's worker MPM, thousands of clients finish their requests but keep their connections open on keep-alive. What is wasted, and which MPM fixes it?
- A) Processes; prefork fixes it
- B) File descriptors; prefork fixes it
- C) CPU; mod_php fixes it
- D) Threads stay pinned to idle connections; event MPM parks idle sockets on epoll in a listener thread and gives the thread away

> ✅ **Answer: D.** This is "the keep-alive problem in one line". Event MPM (2012, the default on Unix today) is the select loop arriving inside Apache. Prefork is the most expensive per connection, so A is wrong.

**Q24.** Why did HAProxy (2001) start mostly as a select loop?
- A) epoll only arrived in Linux 2.5.44 in late 2002, so select was what there was
- B) select scales better than epoll for proxies
- C) HAProxy was multi-threaded from day one and needed select
- D) select has no 1024 limit on BSD

> ✅ **Answer: A.** "The timeline explains the design." HAProxy moved to epoll later and became multi-threaded only in 1.8 (2017), so C is wrong.

**Q25.** An nginx worker serves a large static file that is not in cache, from a slow disk. Before 1.7.11, what happened, and how does nginx deal with it now?
- A) epoll told nginx the disk wasn't ready, so nothing stalled
- B) sendfile skipped the disk entirely
- C) The page fault on a cache miss stalled every connection on that worker; since 1.7.11 nginx offloads such disk reads to a thread pool
- D) nginx forked a new worker for every file request

> ✅ **Answer: C.** "Disk reads can block even in a loop." Each worker is a single thread with one epoll loop, so one blocked read stalls thousands of connections. sendfile (B) avoids the user-space copy, not the disk read.
