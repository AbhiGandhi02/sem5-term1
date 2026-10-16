# 📓 One Process, Many Clients — Scaling Servers
### *CN at Scaler — Lesson 3 (Week 3: Server-side considerations for scale)*

> `select · epoll · CGI · FastCGI` · "and what your favourite server actually does"
> **fork → thread → loop** · **CGI → mod_php → FastCGI** · **1024 · 8 MB · L = λW**

**Legend for these notes:** 🧠 = my own addition, *not on the slides*: background you need to follow the slides, or a correction or caveat. Everything without 🧠 comes from the slides.

---

## ⭐ What this lecture is about, and why it matters

In Lesson 1 we wrote an echo server that could talk to **one client at a time**. Real servers (a web server, a database, an MQTT broker) have to talk to **thousands of clients at once** on a single machine. This lecture answers one question: **how does one box hold ten thousand clients at the same time?**

The answer comes in five blocks:

```
  01  MULTIPLEXING        fork vs thread vs ONE LOOP (select / epoll) · timeouts & retries
  02  CGI / FASTCGI       how a web server runs YOUR code (and why forking per request hurts)
  03  WEB SERVERS?        not everything speaks HTTP: databases, MQTT, Kafka
  04  LIMITS              the four walls: file descriptors · memory · context switches · the queue
  05  IN THE WILD         Apache · HAProxy · Varnish · nginx: all ended up as "a loop + a small pool"
```

Why it matters: every framework you will use (Express, Flask, Spring, Go's `net/http`) sits on top of one of these designs. Once you know the designs you can predict a server's limits, read its config, and debug it under load. The slides' closing line is **"The loop is the architecture."**

---

## 📚 Course map (from the slides)

| Week | Broad topic |
|---|---|
| 1 | Network programming 101 |
| 2 | Computer Networking: Eagle Eye View |
| **3** | **Server-side considerations for scale** ← *this lecture* |
| 4 | nginx deep dive |
| 5 | Evolution of HTTP |
| 6 | (Ab)using CDNs |
| 7 | Economics of cloud tech |
| 8 | Building for failures |

---

## 0 · PREREQUISITES FROM SCRATCH 🧠

The slides assume all of the following. Read this part if any term in the lecture felt fuzzy.

### 0.1 Process vs thread

```
 PROCESS = a running program with its OWN everything
 ┌─────────────────────────────────────────────┐
 │  address space (code, heap, globals)         │
 │  page tables (virtual → physical mapping)    │
 │  file-descriptor table (fd 0,1,2,3...)       │
 │  ┌───────────┐  ┌───────────┐                │
 │  │ thread 1  │  │ thread 2  │  ← THREADS share everything above,
 │  │  stack    │  │  stack    │    but each has its own STACK +
 │  │  registers│  │  registers│    registers + scheduler entry
 │  └───────────┘  └───────────┘                │
 └─────────────────────────────────────────────┘
```

- A **process** is isolated. If it crashes, only it dies.
- A **thread** is a unit of execution *inside* a process. Threads share memory, so they are cheaper to create than processes. The downside: one bad thread (a segfault, or memory corruption) can take down the whole process.
- Analogy: a process is a separate restaurant with its own kitchen. A thread is one more cook in the same kitchen.

### 0.2 `fork()` and `exec()`

- `fork()` creates a **copy** of the current process. It returns **twice**: `0` in the child, and the child's PID (process ID) in the parent.
- `exec()` (for example `execve`) **replaces** the current process's program with a different program. The PID stays the same, and open file descriptors stay open (unless they are marked close-on-exec).
- Your shell runs every command as **fork, then exec**. That is the cost CGI pays on every request (block 02).
- Modern kernels use **copy-on-write**: after `fork()` the pages are shared until someone writes to them. Fork is still expensive, because page tables, the fd table and scheduler state all get duplicated.

### 0.3 File descriptors (fds)

A **file descriptor** is a small integer, an index into your process's open-file table. A file, a pipe and a **socket** all get one. `0` = stdin, `1` = stdout, `2` = stderr, so your first socket is usually `3`.

**Every TCP connection your server holds costs one fd.** The OS rations fds per process. That is wall #1 in block 04.

### 0.4 Blocking I/O

By default `accept()`, `read()` and `write()` **block**: they put your thread to sleep until something happens (a client connects, data arrives, buffer space frees up).

```
 thread: read(fd_A) ──── zzz (A says nothing for 30 s) ────▶ returns
         meanwhile B, C, D are all waiting for you and nobody serves them
```

That one problem is what this whole lecture is about. There are three ways around it:
1. Give each client its own process (fork).
2. Give each client its own thread.
3. Ask the kernel *"which of my sockets are ready?"* and only touch those (select/epoll).

### 0.5 User space vs kernel space, and system calls

```
  ┌────────────── USER SPACE ──────────────┐
  │   your program: buffers, logic          │
  └─────────────┬────────────▲─────────────┘
     syscall    │            │  return
  (read/write/  ▼            │
   accept...)  ┌────────────────────────────┐
               │   KERNEL: TCP stack, disk,  │
               │   scheduler, socket buffers │
               └────────────────────────────┘
```

A **system call** (syscall) is a request from your program to the kernel. Each one crosses the user/kernel boundary, and that crossing has a cost (block 04, wall #3).

A **context switch** is when the CPU stops running one thread and starts another. The kernel has to save registers, maybe switch page tables, and the CPU caches go cold. The lecture uses "context switches" loosely for both this and the user-to-kernel crossing.

### 0.6 Pipes and environment variables

- A **pipe** is a one-way byte channel inside the kernel, with a write end and a read end (two fds). The shell's `ls | grep x` uses one.
- **Environment variables** (`envp`) are `KEY=VALUE` strings handed to a program when it starts (`echo $HOME`). The child gets them at `exec` time. CGI uses both pipes and environment variables.

### 0.7 Stateless vs stateful protocols

- **Stateless**: every request carries everything needed to answer it. HTTP is stateless (cookies/tokens put the state *inside* the request).
- **Stateful**: the connection itself remembers things (you logged in, you selected database 3, you subscribed to a topic). Databases and MQTT are stateful.

### 0.8 Caches and proxies

- A **cache** keeps a copy of something expensive so the next request is cheap.
- A **forward proxy** sits next to the *users* and makes requests on their behalf (a corporate proxy, Squid).
- A **reverse proxy** sits in front of the *servers* and receives requests on their behalf (nginx, Varnish, HAProxy).

```
  users ──▶ [forward proxy] ──▶ internet ──▶ [reverse proxy] ──▶ app servers
            (Squid)                          (nginx / Varnish / HAProxy)
```

### 0.9 Framing (callback to Lesson 2)

TCP is a **byte stream** with no message boundaries. To know where one message ends, a protocol uses either:
- a **length** ("the next 17 bytes are the message"), or
- a **delimiter** ("read until you see a blank line / a dot").

The slides say there is no third option. FastCGI uses both (block 02).

---

## 1 · MULTIPLEXING: three ways to serve many clients

### The headline: "A thread per client is a bet you lose at ten thousand"

*Plain English:* there are three ways to handle many clients, and only one of them survives at scale.

| Model | What it does | Cost / verdict (slides) |
|---|---|---|
| **Fork per request** | A whole process is copied for every client | Simple and isolated, and the most expensive thing you can do per connection |
| **Thread per request** | A new thread per client | Cheaper than fork, so latency drops. The cost does not go away: every thread still costs a stack before it does any work |
| **One loop, many sockets** | `select` or `epoll`: one thread watches every socket and only wakes up for the ready ones | **This is the one that scales** |

```
 FORK PER REQUEST          THREAD PER REQUEST         ONE LOOP (select/epoll)
 ┌──────┐                  ┌───────────────┐          ┌──────────────────────┐
 │parent│──fork──▶[P1]─A   │ process       │          │ one thread           │
 │      │──fork──▶[P2]─B   │  T1 ── A      │          │   ┌────────────┐     │
 │      │──fork──▶[P3]─C   │  T2 ── B      │          │   │ who's      │◀── A│
 └──────┘                  │  T3 ── C      │          │   │ ready?     │◀── B│
  N clients = N processes  └───────────────┘          │   └────────────┘◀── C│
                            N clients = N stacks      └──────────────────────┘
                                                       N clients = N fds, 1 thread
```

**What actually ships:** most of the time we use neither model in pure form. We **pre-fork or pre-thread a pool**, and each worker runs its own `select` or `epoll` loop. *"Isolation from the processes, concurrency from the loop."*

**Pre-fork / pre-thread** means creating the processes or threads **at startup, before any client arrives**, instead of creating one on each `accept()`. The creation cost leaves the request path.

---

### 1.1 `fork()`: a whole process per client

*Plain English:* every time a client connects, clone the whole server and let the clone handle that client.

The slide's code (an echo server on **port 2026**, the course's port):

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

**Read the code (from the slide):**
- `listen(server_fd, 1)` gives a **backlog of one**: at most about one finished connection waits in the accept queue.
- The **child** echoes, then exits.
- The **parent** closes its copy of `client_fd` and loops back to `accept()`.

```
 parent                         child (copy)
 ──────                         ────────────
 accept() → fd 4
 fork() ─────────────────────▶  fork() returned 0
 close(4)  (parent's copy)       read(4) / write(4) loop
 accept() again ...              close(4); exit
```

**What it costs (slide):**
- A full process image per connection.
- Page tables, fds and a scheduler entry, all duplicated.
- **You pay it while the user waits.** The fork happens *after* `accept`, on the request path.

**Why does it exist?** It is the simplest way to get concurrency, and the isolation is perfect: a crash in one child cannot touch another client.

**Pitfalls / gotchas 🧠**
- **Why must the parent `close(client_fd)`?** After `fork`, both processes hold the fd. The TCP connection only really closes when *every* copy is closed. If the parent forgets, it leaks one fd per client and hits the fd limit (block 04).
- **Zombies.** The parent never calls `wait()`, so every exited child stays as a zombie process table entry. Real code handles `SIGCHLD` or sets `signal(SIGCHLD, SIG_IGN)`.
- The child also inherits `server_fd`. Tidy code closes it in the child.
- The backlog of 1 is not "one client at a time". Accepted clients are handed to children right away. It only limits how many *not-yet-accepted* connections can queue (and Linux rounds it a little).

---

### 1.2 `select()`: one thread, every client

*Plain English:* instead of waiting on one socket, hand the kernel a **list of sockets** and say *"wake me when any of these has something for me."* Then serve only those sockets and ask again.

Analogy 🧠: a waiter with 20 tables does not stand at table 1 until they order. He scans the room, sees which hands are up, and serves those.

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
        FD_ZERO(&readfds);                       /* 1. empty the set           */
        FD_SET(server_fd, &readfds);             /* 2. watch the listener      */
        int maxfd = server_fd;

        for (int i = 0; i < nclients; i++) {     /* 3. watch every client      */
            FD_SET(clients[i], &readfds);
            if (clients[i] > maxfd) maxfd = clients[i];
        }

        select(maxfd + 1, &readfds, NULL, NULL, NULL);   /* 4. SLEEP until something is ready */

        if (FD_ISSET(server_fd, &readfds)) {     /* 5a. listener ready = new client */
            clients[nclients++] = accept(server_fd, NULL, NULL);
        }

        for (int i = 0; i < nclients; i++) {     /* 5b. which clients are ready? */
            int fd = clients[i];
            if (!FD_ISSET(fd, &readfds)) continue;

            char buf[4096];
            int n = read(fd, buf, sizeof(buf));
            if (n <= 0) {                        /* 0 = client closed, <0 = error */
                close(fd);
                clients[i] = clients[--nclients];   /* O(1) delete */
                i--;
            } else {
                write(fd, buf, n);
            }
        }
    }
}
```

**The loop, as a picture:**

```
     ┌────────────────────────────────────────────────────────┐
     │ build fd_set = {server_fd, client1, client2, ...}       │
     │                       │                                 │
     │                       ▼                                 │
     │       select()  ── thread sleeps in the kernel ──       │
     │                       │  returns: set now holds ONLY    │
     │                       ▼  the ready fds                  │
     │  server_fd ready? ─▶ accept() a new client              │
     │  client ready?    ─▶ read(); n<=0 → close+remove        │
     │                          else → write() echo            │
     └───────────────────────── loop ─────────────────────────┘
```

**The macros 🧠**

| Macro | Meaning |
|---|---|
| `FD_ZERO(&set)` | clear all bits |
| `FD_SET(fd, &set)` | turn on the bit for `fd` ("watch this") |
| `FD_ISSET(fd, &set)` | after `select`: is the bit still on? ("is it ready?") |
| `FD_CLR(fd, &set)` | turn off one bit |

The first argument to `select` is `maxfd + 1`: "check bits 0 to maxfd". The kernel needs to know how far into the bitmap to look.

**"Two lines worth stopping on" (slide)**

1. `clients[i] = clients[--nclients]; i--;` is an **O(1) delete**. Instead of shifting the whole array left, copy the **last** element into the hole and shrink the count. `i--` makes the loop revisit index `i`, because a new, not-yet-checked fd now sits there.
   ```
   before:  [ 4 | 5 | 6 | 7 ]  n=4   remove index 1 (fd 5)
   after:   [ 4 | 7 | 6 ]      n=3   i-- → next iteration checks fd 7
   ```
   🧠 The order of clients is not preserved. That is fine here, because order does not matter.
2. `select(..., NULL, NULL, NULL)`: **that last NULL is the timeout.** NULL means "wait forever" (see 1.5).

**Why does it exist?** One thread serves many clients, with no per-client process or stack. Idle clients cost only a bit in a bitmap and an fd.

**Pitfalls / gotchas 🧠**
- **`select` overwrites your `fd_set`.** On return the set holds only the ready fds. That is *why* the code rebuilds it with `FD_ZERO`/`FD_SET` on every iteration. Forget that and you lose track of clients.
- **One slow operation stalls everyone.** `write(fd, ...)` is still blocking. If a client stops reading, its socket send buffer fills, `write` blocks, and the whole loop (every other client) freezes. Real event loops set sockets to **non-blocking** (`O_NONBLOCK`) and also watch `writefds`.
- The code never checks whether `accept` returned `-1`, or whether `nclients` reached `FD_SETSIZE`. This is exactly what homework #1 makes you break.

---

### 1.3 select's ceiling is compiled in: `FD_SETSIZE = 1024`

*Plain English:* `select` uses a fixed-size bitmap, so there is a hard maximum fd number it can watch, and it is small.

```c
int clients[FD_SETSIZE];   /* not a style choice: this IS the hard limit */
```

- `fd_set` is a **fixed-width bitmap**. On glibc (GNU C Library, the standard C library on most Linux systems) that width is **1024 bits**.
- So **descriptor 1024 cannot be represented at all**. Valid fds are 0 to 1023.

```
 fd_set (1024 bits = 128 bytes)
 bit:  0 1 2 3 4 5 6 ...                                   1023 │ 1024 → ??? (off the end)
      [0 0 0 1 1 0 1 ......................................  0 ]│
             ↑ server_fd=3, clients 4 and 6
```

🧠 The limit applies to the fd's **numeric value**, not to how many fds you watch. Even if you only watch 5 sockets, if one of them is fd 1500 you cannot use `select` on it. `FD_SET(1500, ...)` writes past the end of the struct, which is undefined behaviour and corrupts memory silently.

#### select vs epoll

| | **select: O(n) every call** | **epoll: O(ready)** |
|---|---|---|
| Registration | You **rebuild the set on every iteration** | **Register each fd once**, with `epoll_ctl` |
| Kernel work | The kernel **scans every fd, ready or not** | `epoll_wait` **returns only what actually fired** |
| Idle sockets | **10,000 idle sockets cost the same as 10,000 busy ones** | **Idle connections cost nothing to wait on** |
| Max fds | 1024 (compiled in) | limited only by `ulimit -n` and memory |

**Same idea, different name (slide):**
- **kqueue**: BSD and macOS
- **IOCP** (I/O Completion Ports): Windows
- **io_uring**: modern Linux

🧠 There are two families here. `select`/`poll`/`epoll`/`kqueue` are **readiness** APIs ("fd 7 is readable now, *you* call `read`"). IOCP and io_uring are **completion** APIs ("here is a buffer, read into it and tell me when it's *done*"). 🧠 `poll()` is the in-between step: it takes an array instead of a bitmap, so there is no 1024 cap, but it is still O(n) per call.

#### What epoll code looks like 🧠

The slides describe epoll but show no code. This is the skeleton:

```c
#include <sys/epoll.h>

int ep = epoll_create1(0);                       /* 1. make an epoll instance (it's an fd too) */

struct epoll_event ev = { .events = EPOLLIN, .data.fd = server_fd };
epoll_ctl(ep, EPOLL_CTL_ADD, server_fd, &ev);    /* 2. register ONCE */

struct epoll_event ready[64];
while (1) {
    int n = epoll_wait(ep, ready, 64, -1);       /* 3. -1 = no timeout (like NULL) */
    for (int i = 0; i < n; i++) {                /*    loop over READY fds only */
        int fd = ready[i].data.fd;
        if (fd == server_fd) {
            int c = accept(server_fd, NULL, NULL);
            struct epoll_event cev = { .events = EPOLLIN, .data.fd = c };
            epoll_ctl(ep, EPOLL_CTL_ADD, c, &cev);   /* register the new client once */
        } else {
            /* read(fd...); on n<=0: close(fd) (close removes it from epoll) */
        }
    }
}
```

Compare with select: there is no rebuilding, no `maxfd`, and no scanning of idle fds. The kernel keeps the interest list between calls and hands you a ready list.

🧠 **Level-triggered vs edge-triggered.** The default epoll mode (**level-triggered**) keeps reporting an fd for as long as it has unread data, the same as select. `EPOLLET` (**edge-triggered**) reports only when *new* data arrives. With ET you must read until `EAGAIN` on a non-blocking socket, or you can miss data forever. nginx uses edge-triggered mode.

#### "The number to remember": 1024

> **1024.** It is `FD_SETSIZE`, it is your default `ulimit`, and it is the K in C10K. *One wall, three names.*

(🧠 "K in C10K" is a play on 1024 ≈ 1 Ki. C10K literally means 10,000 concurrent connections. See block 04.)

---

### 1.4 In production you don't pick one, you stack them

*Plain English:* real servers combine all three models. A few processes give crash safety, and each one runs an event loop for scale.

```
  AT STARTUP            PER WORKER               ON ACCEPT                    RESULT
 ┌───────────────┐    ┌──────────────────┐    ┌──────────────────────┐    ┌───────────────────┐
 │ fork/spawn a  │ ─▶ │ each runs its own│ ─▶ │ SO_REUSEPORT lets the│ ─▶ │ crash isolation   │
 │ pool of       │    │ select/epoll loop│    │ kernel spread new    │    │ from processes,   │
 │ workers, often│    │ over many sockets│    │ connections across   │    │ concurrency from  │
 │ 1 per CPU core│    │                  │    │ them                 │    │ the loop          │
 └───────────────┘    └──────────────────┘    └──────────────────────┘    └───────────────────┘
```

```
                 port 2026 (SO_REUSEPORT)
                 kernel load-balances new connections
           ┌──────────────┼──────────────┬──────────────┐
           ▼              ▼              ▼              ▼
       worker 0       worker 1       worker 2       worker 3     ← one per core
       epoll loop     epoll loop     epoll loop     epoll loop
       ~thousands     ~thousands     ~thousands     ~thousands of conns each
```

**`SO_REUSEPORT` 🧠:** a socket option (Linux 3.9+) that lets **several sockets bind the same IP:port**. The kernel then hashes each new connection to one of them. Without it, workers either share one listening socket (and can stampede to `accept` the same connection, the "thundering herd" problem) or need a lock. Don't confuse it with `SO_REUSEADDR` from Lesson 1, which is about re-binding while old connections sit in `TIME_WAIT`.

**This is not theoretical (slide):**
- It is **Apache's event MPM** (block 05).
- It is **nginx's master-and-workers** design.
- It is what the **Go runtime** does for you invisibly: **goroutines multiplexed onto OS threads by a scheduler sitting on top of epoll.**

---

### 1.5 Timeouts are the client's half of the contract

*Plain English:* never wait forever for anything over a network. The other side might be dead, slow or stuck.

The server may be busy reading, writing or accepting. **Never wait indefinitely.**

In a select loop you are waiting on three different things:

| Set | What you're waiting for (slide wording) |
|---|---|
| `readfds` | some new content to arrive |
| `writefds` | some socket allowing us to write |
| `exceptfds` | sockets being connected and closed |

> 🧠 Precision note: by the man page, `exceptfds` reports "exceptional conditions", mainly TCP out-of-band (urgent) data. Connection *completion* on a non-blocking `connect()` actually shows up in `writefds`, and a peer closing shows up in `readfds` (`read` returns 0). The slide's wording is a simplification.

```c
select(maxfd + 1, &readfds, NULL, NULL, NULL);   /* readfds, writefds, exceptfds, timeout */
```

The slide puts it as "each of those comes with a default timeout", and that is the last argument to `select()`. In our code we passed `NULL`. (🧠 Strictly, `select` takes one timeout that covers the whole call.)

**NULL timeout = block forever.** One wedged client and the whole loop stops, along with every other connection on it.

How to pass a real timeout 🧠:

```c
struct timeval tv = { .tv_sec = 5, .tv_usec = 0 };
int r = select(maxfd + 1, &readfds, NULL, NULL, &tv);
if (r == 0) {
    /* timed out: nothing happened in 5 s → do housekeeping,
       e.g. close clients idle longer than N seconds */
}
/* NOTE: on Linux select() modifies tv, so reset it every iteration */
```

#### The client side: deadline, then retry with backoff

The slide says the same rule applies on the client side: **set a deadline, then retry with backoff.**

🧠 What that means:
- **Deadline / timeout**: e.g. `curl --connect-timeout 2 --max-time 5`, `requests.get(url, timeout=5)`, or `setsockopt(SO_RCVTIMEO)`.
- **Retry with exponential backoff**: wait 100 ms, then 200 ms, 400 ms, 800 ms... between attempts, usually with random **jitter**, so a thousand clients don't all retry at the same instant and knock the server over again (the "thundering herd" or retry storm).
- **Only retry idempotent operations safely.** Retrying a `GET` is fine. Retrying "charge my card" can charge twice.

```
 attempt 1 ──X timeout
       wait 0.1s ± jitter
 attempt 2 ──X timeout
       wait 0.2s ± jitter
 attempt 3 ──✓
```

**Gotcha:** a timeout protects *you*, not the server. Without retry limits and backoff, timeouts turn into a flood of retries.

---

## 2 · CGI / FASTCGI: how the server runs *your* code

Block 01 was about connections. Block 02 asks: once the web server has an HTTP request, **how does it run your application code** (a PHP or Perl script, say) to produce the response?

### 2.1 CGI: every request forks a new copy of your program

*Plain English:* **CGI** (Common Gateway Interface) is a convention. The web server starts your program fresh for every request, passes the request details in environment variables and stdin, and sends whatever your program prints back to the browser.

> "Apache was the most popular HTTP server on earth, and this is how it ran your code."

The slide's diagram:

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

- **envp** carries the request metadata (method, headers, query string) **into** the process.
- **stdin pipe** carries the request **body** in.
- **stdout pipe** carries the **response** out, back to Apache and then the client.
- **stderr** goes to the **Apache error log**.

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

🧠 What that looks like in real C (simplified):

```c
int in[2], out[2], err[2];
pipe(in); pipe(out); pipe(err);

if (fork() == 0) {                 /* child */
    dup2(in[0],  0);               /* child's stdin  = read end of 'in'   */
    dup2(out[1], 1);               /* child's stdout = write end of 'out' */
    dup2(err[1], 2);               /* child's stderr = write end of 'err' */
    /* close the other ends ... */
    execve("/var/www/cgi-bin/hello.cgi", argv, envp);   /* envp = CGI vars */
}
/* parent: write body → in[1]; read out[0] → client; read err[0] → error log */
```

`dup2(a, b)` makes fd `b` refer to the same thing as `a`. The CGI program then just uses normal stdin and stdout, and it never knows a network exists.

**The real thing (slide):**
- **`apache/httpd` · `modules/generators/mod_cgi.c` · L576 · `run_cgi_child()`**
- It creates the child, attaches the three pipes, and **sets a timeout on each**, so a hung script cannot hold the worker forever. That is the timeout lesson from 1.5 again.

### 2.3 HTTP headers become environment variables

*Plain English:* the web server translates the HTTP request into `KEY=VALUE` environment variables, because that is something every program in every language can read.

The slide's pseudo-code:

```
Convert HTTP headers and verb to variables
E.g. REQUEST_METHOD = GET / POST, QUERY_STRING = <things after ?>
Pass these as environment variables while creating the CGI process
```

**The convention (slide):** header `User-Agent: curl` becomes `HTTP_USER_AGENT=curl`. **Upper-case it, swap dashes for underscores, prefix with `HTTP_`.** Every framework you have ever used still carries this shape.

```
 HTTP request                                    environment of CGI process
 ─────────────────────────────                   ──────────────────────────────
 POST /cgi-bin/f.cgi?x=10 HTTP/1.1      ──▶      REQUEST_METHOD=POST
 Host: localhost                        ──▶      QUERY_STRING=x=10
 User-Agent: curl/8.4                   ──▶      HTTP_HOST=localhost
 Content-Type: application/x-www-...    ──▶      HTTP_USER_AGENT=curl/8.4
 Content-Length: 17                     ──▶      CONTENT_TYPE=application/x-www-...
                                                  CONTENT_LENGTH=17
 name=alice&age=20   (body)             ──▶      (body arrives on STDIN)
                                                  GATEWAY_INTERFACE=CGI/1.1
                                                  SERVER_PROTOCOL=HTTP/1.1
```

🧠 `Content-Type` and `Content-Length` are special: by the CGI spec (RFC 3875) they become `CONTENT_TYPE`/`CONTENT_LENGTH` **without** the `HTTP_` prefix. Other standard ones: `SCRIPT_NAME`, `PATH_INFO`, `REMOTE_ADDR`, `SERVER_NAME`, `SERVER_PORT`.

🧠 You still see this shape today: Python WSGI's `environ['HTTP_USER_AGENT']`, PHP's `$_SERVER['HTTP_USER_AGENT']`, Django's `request.META['HTTP_USER_AGENT']`.

**The real thing (slide):**
- **`server/util_script.c` · L409 · `ap_add_cgi_vars()`** sets `GATEWAY_INTERFACE`, `SERVER_PROTOCOL`, `REQUEST_METHOD`, `QUERY_STRING`.
- **`ap_add_common_vars()`** (just above it in the same file) does the rest, including the `HTTP_*` headers.

### 2.4 The CGI program is just a program that prints

*Plain English:* no server library, no framework. **Read the environment, write to stdout.**

The slide's pseudo-code:

```
CGI program
  Get environment variable (REQUEST_METHOD)   // GET or POST
  If POST - read from stdin
  If GET  - parse another environment variable (QUERY_STRING)
  Output "Content-Type: text/html" - two lines and text
```

🧠 A real one in shell and in Python:

```sh
#!/bin/sh
# hello.cgi
echo "Content-Type: text/html"
echo ""                                   # ← THE BLANK LINE (the delimiter)
echo "<h1>Hello from PID $$</h1>"
echo "<p>Method: $REQUEST_METHOD, query: $QUERY_STRING</p>"
```

```python
#!/usr/bin/env python3
import os, sys
method = os.environ.get("REQUEST_METHOD", "GET")
if method == "POST":
    n = int(os.environ.get("CONTENT_LENGTH", 0))
    data = sys.stdin.read(n)              # body is on stdin
else:
    data = os.environ.get("QUERY_STRING", "")
print("Content-Type: text/html")
print()                                   # blank line ends headers
print(f"<p>You sent: {data}</p>")
```

**"Two lines" (slide):** the header block, then **a blank line**, then the body. **That blank line is the delimiter**, the same framing question Lesson 2 ended on. Apache reads your headers up to it and streams everything after it to the client.

```
 Content-Type: text/html      ← header(s)
                              ← blank line = "headers finished"   (DELIMITER)
 <h1>Hello</h1>               ← body: streamed as-is to the client
```

**That is the whole contract.** Any language that can read an environment variable and print to standard output can serve HTTP. That is exactly why the early web was written in **Perl and shell**.

**Pitfall 🧠:** forget the blank line and Apache returns **500 Internal Server Error** with "malformed header from script" or "End of script output before headers" in the error log. The script also has to be executable (`chmod +x`).

### 2.5 Copy the whole program, then throw it away

*Plain English:* CGI does the most expensive thing from block 01 (fork + exec) on **every single hit**.

| The copy | The fix people tried | The fix that won |
|---|---|---|
| The whole program image is created, run once, and discarded. Your shell does the same for every command you type, except your shell is not doing it a thousand times a second. | **Compile the module INTO the server**: `mod_php`, `mod_perl`. No new process per request, but now your application **shares an address space with your web server**, and **one segfault takes both down**. | **Keep a long-lived process of your own** and let the web server talk to it over a **Unix or TCP socket**, with a reduced vocabulary. **That is FastCGI.** |

```
 CGI:        request → fork+exec perl → load interpreter → parse script → run → exit   (EVERY time)
 mod_php:    request → (PHP already inside Apache) → run                               (fast, but shared fate)
 FastCGI:    request → socket → (php-fpm worker already running) → run                 (fast AND isolated)
```

**The pattern to notice (slide):** take the setup cost **off the request path** and pay it **once at startup**. It is the same move pre-forking made for connections in block 01.

🧠 **Unix domain socket vs TCP socket:** a Unix socket is a file path (e.g. `/run/php/php-fpm.sock`) that only works on the same machine and skips the TCP/IP stack, so it is slightly faster. A TCP socket (`127.0.0.1:9000`) also works across machines.

### 2.6 FastCGI: keep the process, talk over a socket

*Plain English:* the web server and your app become **two separate long-running programs** that talk over a socket using a compact binary protocol.

> You define an **upstream**. The user-facing server handles slow clients; an internal server does the work.

```
   clients                          nginx / httpd                         php-fpm
 ┌──────────────────┐   HTTP     ┌──────────────────────┐   FastCGI   ┌────────────────────┐
 │ thousands of     │──────────▶ │ user-facing.         │───────────▶ │ internal. a fixed  │
 │ them, many of    │            │ absorbs slow         │             │ pool of long-lived │
 │ them slow        │◀────────── │ connections, TLS,    │◀─────────── │ workers            │
 └──────────────────┘            │ buffering            │             └────────────────────┘
                                 └──────────────────────┘
```

**Why split it this way? 🧠** A slow mobile client might take 10 seconds to upload a request or download a response. If a PHP worker had to sit through that, you would need thousands of heavy PHP workers. Instead, nginx (cheap event loop, block 01) **buffers** the slow client, then hands the complete request to PHP, which answers in 20 ms and is free again. A small fixed pool of workers can then serve a huge number of clients.

**php-fpm 🧠** = PHP FastCGI Process Manager: a master that pre-forks a pool of PHP worker processes. That is pre-forking again.

**A modified HTTP, with less vocabulary (slide):** the same information (method, headers, query string, body), but **binary, typed, and multiplexed**. No parsing a text protocol twice.

**What you actually write (slide):** you *may* still get environment variables, but more often you get a **request object** with the same headers and query string, already decoded for you.

🧠 "Upstream" is nginx vocabulary: the server(s) nginx forwards to. A minimal config:

```nginx
location ~ \.php$ {
    include        fastcgi_params;              # maps request → CGI-style params
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass   127.0.0.1:9000;              # or unix:/run/php/php-fpm.sock
}
```

### 2.7 A FastCGI request is a stream of typed records

*Plain English:* instead of one big text blob, the request is sent as a sequence of small labelled packets ("records"): a BEGIN record, the parameters, the body, and empty records marking where each part ends.

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

**Same nouns as CGI (slide):** `REQUEST_METHOD`, `QUERY_STRING`, `CONTENT_LENGTH`. *The CGI environment survived.* Only the **transport** changed: **records on a socket instead of envp on a fork.**

**STDIN is the body (slide):** `name=alice&age=20` is **seventeen bytes**, exactly the `CONTENT_LENGTH` declared above it. (Count: `name=alice` = 10, `&` = 1, `age=20` = 6, total 17 ✓.)

#### 🧠 What a record looks like in bytes

Every record starts with a fixed **8-byte header**:

```
  byte:   0          1        2   3          4   5            6          7
       ┌─────────┬────────┬───────────────┬─────────────────┬──────────┬─────────┐
       │ version │  type  │  requestId    │  contentLength  │ padding  │reserved │
       │  (=1)   │        │  (2 bytes)    │   (2 bytes)     │ Length   │         │
       └─────────┴────────┴───────────────┴─────────────────┴──────────┴─────────┘
       followed by  contentLength bytes of content  +  paddingLength bytes of padding
```

| type value | name |
|---|---|
| 1 | `FCGI_BEGIN_REQUEST` |
| 2 | `FCGI_ABORT_REQUEST` |
| 3 | `FCGI_END_REQUEST` |
| 4 | `FCGI_PARAMS` |
| 5 | `FCGI_STDIN` |
| 6 | `FCGI_STDOUT` |
| 7 | `FCGI_STDERR` |
| 8 | `FCGI_DATA` |

- **`requestId`** is what makes it "multiplexed": several requests could share one connection, told apart by ID. (In practice php-fpm handles one request per connection at a time.)
- Inside `FCGI_PARAMS`, each name/value pair is **length-prefixed** (name length, value length, name, value). So FastCGI uses length framing even inside the records.
- An **empty `FCGI_PARAMS`** header looks like `01 04 00 01 00 00 00 00`: version 1, type 4 (PARAMS), request 1, content length **0**. You'll hunt for exactly this in homework #3.

### 2.8 ...and the response comes back the same way

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
- **STDERR goes to the log:** a **separate record type**, so your debug output can **never corrupt the page**. It is the same split as the pipes in the CGI diagram.

```
   CGI (pipes)                     FastCGI (record types)
   stdin  pipe   ◀── body          FCGI_STDIN   ◀── body
   stdout pipe   ──▶ client        FCGI_STDOUT  ──▶ client
   stderr pipe   ──▶ error log     FCGI_STDERR  ──▶ error log
   envp          ◀── headers       FCGI_PARAMS  ◀── headers
   process exit  = done            FCGI_END_REQUEST = done
```

### 2.9 The empty record is the delimiter

> Last session ended on framing: **length or delimiter, no third option.** Here they are, in one protocol.

| LENGTH | DELIMITER |
|---|---|
| Every record header carries its own content length. The reader knows exactly how many bytes to take *before* it takes them, so a body can contain **any byte at all, including newlines and nulls**. | A record of **length zero** means "that stream is finished". Not a byte pattern in the data: a **structural marker that cannot collide with your content**. |
| `CONTENT_LENGTH = 17` does the same job one level up. | The **blank line after `Content-Type`** is the same trick in text. |

> **SMS chose length. SMTP chose a dot. IMAP chose both. FastCGI chose both too, and for the same reason.**

🧠 Why both? A length lets you carry arbitrary binary data safely in chunks. A delimiter (the empty record) lets you **stream** something whose total size you don't know in advance: you send chunks of known length until you're done, then send the "end" marker. HTTP/1.1 chunked transfer encoding works the same way, with a zero-length chunk at the end. (SMTP ends a message with a line containing only `.`. IMAP uses literal `{length}` prefixes as well as line delimiters.)

**Gotcha:** a delimiter that is a byte pattern (like SMTP's `.`) needs **escaping** if the content contains it (SMTP "dot-stuffing"). FastCGI's empty-record delimiter needs no escaping, because it is structural.

### 2.10 Servlets: the same idea wearing a JVM

*Plain English:* Java's answer. Instead of a separate app process behind a web server, the **Java server *is* the web server**, and it runs your code on a pool of threads.

| The server is the runtime | Pre-threading | Isolation by convention |
|---|---|---|
| **Jetty** and **Tomcat** act as the web server themselves. There's no separate httpd in front doing the forking; the JVM (Java Virtual Machine) is already running your code. | A **pool of threads created at startup**. A request is handed to an idle thread, which runs your servlet and returns to the pool. Exactly the pre-threading from block 01. | Servlets are isolated by **class loader and thread, not by process**. Cheaper than fork, but **one runaway servlet can still hurt its neighbours**. |

```
 Tomcat (one JVM process)
 ┌───────────────────────────────────────────────┐
 │ acceptor ──▶ request queue ──▶ thread pool     │
 │                                  T1 runs AppA  │
 │                                  T2 runs AppB  │  ← an infinite loop / OOM in AppA
 │                                  T3 idle       │    still hurts AppB (same heap)
 └───────────────────────────────────────────────┘
```

> **CGI, FastCGI, servlets, and every modern framework are answering one question: how do we run user code per request without paying to create the world each time?**

---

## 3 · ISN'T EVERYTHING A WEB SERVER?

### 3.1 Almost everything runs a web server, but not everything *is* one

*Plain English:* lots of things speak HTTP, but many important servers use their own protocols. **The protocol decides which scaling tricks you get.**

| ✅ YES: it speaks HTTP | ❌ NO, and it doesn't need to |
|---|---|
| Your **Wi-Fi router admin page** | **Redis**: its own line protocol, **RESP** |
| Modern **agent runtimes** and cloud tooling | **MySQL**: its own binary wire protocol |
| Most **Node** applications | **MQTT brokers** across the whole IoT estate |
| **Firestore** (the popular one) | **Messenger backends**: WhatsApp-shaped things |
| **CouchDB**, which has had REST for years | **Kafka natively**: a binary protocol, not HTTP |
| **Kafka**, if you put the **REST proxy** in front | |

🧠 Quick decoder:
- **RESP** = REdis Serialization Protocol. Text-ish, e.g. `*2\r\n$3\r\nGET\r\n$3\r\nkey\r\n` (more length-prefixed framing).
- **MQTT** = Message Queuing Telemetry Transport: a tiny publish/subscribe protocol for IoT (Internet of Things) devices. A sensor keeps **one long-lived TCP connection** to a broker and publishes or subscribes to topics.
- **Kafka**: a distributed log / message queue. Clients speak a binary protocol over persistent TCP. The **Kafka REST Proxy** is a separate HTTP service that translates.
- **REST** = REpresentational State Transfer: an API style over HTTP (URLs + verbs).
- Messenger backends (WhatsApp historically used a modified XMPP on Erlang) hold **a persistent connection per online phone**.

**Why this earns a slide:** everything you are about to learn about scaling web servers **assumes HTTP's properties**. Change the protocol and the advice changes with it.

### 3.2 HTTP is stateless. What you scale next may not be.

| A web server can hang up | A database wants to stay |
|---|---|
| **Every request carries everything it needs.** So any box can answer any request, you can close the connection whenever you like, and a load balancer can send the next one anywhere. | **Redis or MySQL would rather hold a persistent connection** than pay the cost of a **TCP connect, plus a handshake, plus auth, plus session setup** on every query. |
| **Horizontal scaling is free-ish.** Add a server, add it to the pool, done. *The statelessness is what makes that legal.* | **So the problem changes shape.** The question is no longer requests per second. It is: **how many persistent connections can one box hold at once?** (That's block 04.) |

```
 STATELESS (HTTP)                          STATEFUL (DB / MQTT / chat)
 client ─req─▶ LB ─▶ any of [S1 S2 S3]     client ════ one long connection ════ DB
 client ─req─▶ LB ─▶ any of [S1 S2 S3]       (auth'd, session vars, txn state)
  connection can close after each one       connection lives for minutes/hours
  scale = add boxes                         scale = how many conns can 1 box HOLD?
```

🧠 The cost of a fresh DB query connection: TCP 3-way handshake (1 RTT, round-trip time) + maybe TLS (1–2 RTT) + MySQL greeting and auth exchange + setting session variables. That is often more work than the query itself, which is why apps use **connection pools**.

**Gotcha:** "HTTP is stateless" doesn't mean the *app* has no state. Sessions live in cookies, tokens or a shared store (Redis), which is precisely so that any HTTP server can answer.

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

### 4.1 Wall 1 · File descriptors: C10K was 1999's problem, your default ulimit is still 1024

| Year | Name | Number | What happened (slide) |
|---|---|---|---|
| **1999** | **C10K** | **10,000** | **Dan Kegel** names the problem: can one box hold ten thousand simultaneous persistent connections? At the time, **no**. |
| **2013** | **C10M** | **10,000,000** | **Robert Graham** argues *the kernel is the problem, not the solution*. Bypass it and the number moves by three orders of magnitude. |
| **Today** | | **2–3 M** | People have demonstrated two to three million connections on a single server. *The problem moved; the default did not.* |

```sh
$ ulimit -n
1024        # the soft limit on most distributions, unchanged since the problem was named
```

🧠 **What is `ulimit`?** A shell built-in that shows or sets per-process resource limits. `-n` is "max open file descriptors".
- **Soft limit**: what's enforced now. You can raise it yourself up to the hard limit: `ulimit -n 65535`.
- **Hard limit**: the ceiling (`ulimit -Hn`). Only root can raise it.
- Permanent settings live in `/etc/security/limits.conf`, systemd's `LimitNOFILE=`, or Docker's `--ulimit nofile=...`.
- System-wide cap: `/proc/sys/fs/file-max`.
- When you hit it, `accept()` fails with **`EMFILE` ("Too many open files")**. Your server is up, but it refuses new clients.
- 🧠 On **macOS** (your laptop), `ulimit -n` often shows **256**, even lower.

🧠 **Why the default is so low:** historical caution, plus old programs that use `select()` break on fds ≥ 1024. Keeping the default soft limit at 1024 protects them. Servers like nginx raise it themselves (`worker_rlimit_nofile`).

**"One wall, three names":** 1024 = `FD_SETSIZE` = default `ulimit -n` = (playfully) the K in C10K.

**Gotcha:** each connection may cost **more than one fd**. A proxy holds the client fd *plus* an upstream fd, and you also have log files, the listening socket and the epoll fd. A reverse proxy with a 1024 limit tops out at roughly 500 client connections.

### 4.2 Wall 2 · Memory: every thread costs memory before it does any work

| OS thread | Goroutine |
|---|---|
| **8 MB**: default stack, **reserved up front** | **2 KB**: initial stack, **grown on demand** |
| **10,000 threads reserves roughly 80 GB of address space** before a single byte of your data exists. | **Four thousand times smaller.** This single number is most of why **Go took over this problem space**. |

Arithmetic: 10,000 × 8 MB = 80,000 MB ≈ 80 GB. And 8 MB / 2 KB = 8192 KB / 2 KB = 4096 ≈ "four thousand".

**And underneath, it is the same loop (slide):** goroutines are lightweight because the Go runtime **multiplexes them onto a few OS threads using a netpoller built on epoll**, and runs **its own scheduler** on top. **You write blocking reads; the runtime parks you on a poll.**

```
  your Go code (looks blocking):          what really happens:
  ───────────────────────────            ──────────────────────────────────────
  n, err := conn.Read(buf)   ──▶  socket not ready? → goroutine PARKED,
                                  fd registered with epoll (netpoller),
                                  OS thread runs OTHER goroutines.
                                  epoll says ready → goroutine runnable again.
```

So Go gives you the *programming model* of thread-per-connection with the *cost model* of an event loop.

> **The more requests you handle at the same time, the more memory you need. The only question is how much per request.**

🧠 Nuances:
- "Reserved" means **virtual address space**. Linux only commits physical RAM for stack pages actually touched, so 10k idle threads don't use 80 GB of real RAM. You still pay for page tables, kernel thread structures (each thread also has a kernel stack, typically 16 KB) and scheduler overhead, and on 32-bit systems you'd simply run out of address space. Check your default with `ulimit -s` (8192 = 8 MB on most Linux).
- Besides stacks, every connection has **socket buffers** in the kernel (send and receive, often tens to hundreds of KB under load) and **app buffers** (like `char buf[4096]`). "A buffer per connection."

### 4.3 Wall 3 · Context switches: every read and write is a trip across the kernel boundary

> In fact, every system call is. At a million operations a second, **the tax is the workload.**

| The tax | Bypass the kernel | Or make fewer calls |
|---|---|---|
| User space → kernel space and back: **save state, switch page tables, validate, copy, return**. **Hundreds of nanoseconds** each, and **Spectre and Meltdown mitigations made it worse**, not better. | Some people have implemented **TCP in user space** and bypassed the kernel entirely for well-known internal networks: **DPDK, netmap, mTCP, Seastar**. You **give up every tool you own** and you get the packets. | **`sendfile()`** moves a file to a socket **without a user-space round trip**. **`io_uring`** submits a **batch** of operations through a **shared ring** instead of one syscall each. |

🧠 Decoder:
- **Spectre/Meltdown** (2018): CPU security flaws. The fix for Meltdown, **KPTI** (Kernel Page-Table Isolation), switches page tables on every syscall entry and exit. That is the "switch page tables" cost on the slide.
- **DPDK** (Data Plane Development Kit, from Intel): the app talks directly to the NIC (Network Interface Card) and polls it from user space.
- **netmap**: fast packet I/O framework (FreeBSD/Linux).
- **mTCP**: a user-space TCP stack built for multicore.
- **Seastar**: a C++ framework (behind ScyllaDB) with one thread per core, no sharing, and its own TCP stack.
- "You give up every tool you own": the kernel no longer sees the packets, so `ss`, `tcpdump`, `iptables` and the normal socket API stop working for that traffic.

```
 normal:      app ⇄ syscall ⇄ kernel TCP/IP ⇄ driver ⇄ NIC
 bypass:      app + user-space TCP ⇄ NIC (mapped directly)      ← no syscalls per packet
 io_uring:    app ─▶ [submission ring] ─▶ kernel ─▶ [completion ring] ─▶ app
              (many ops per syscall, or zero syscalls with polling mode)
```

> **Remember `sendfile`**: it is the reason nginx serves static files the way it does (block 05).

**Gotcha:** "context switch" is often used for two different things: (a) the user↔kernel mode switch on a syscall, and (b) the scheduler swapping one thread for another. Thread-per-connection designs pay a lot of (b). Event loops reduce (b) but still pay (a) on every `read`/`write`.

### 4.4 Wall 4 · The queue: concurrency isn't a setting, it's throughput × latency

*Plain English:* how many connections you hold at once is not something you choose. It falls out of how many requests arrive per second times how long each one takes.

```
   connections in flight   =   requests per second   ×   seconds per request
          L                =           λ             ×           W
```

🧠 This is **Little's Law** (named in homework #4): L = λ × W, the average number of items in a system equals arrival rate × average time each spends inside. It holds for any stable queue: supermarket checkouts, servers, anything.

The slide's chart, **at a steady 10,000 requests/second**:

```
 connections in flight
 10000 ┤                                        ████
       │                                        ████
  8000 ┤                                        ████
       │                                        ████
  6000 ┤                                        ████
  5000 ┤                               ████     ████
  4000 ┤                               ████     ████
  2000 ┤                               ████     ████
  1000 ┤                      ████     ████     ████
   500 ┤             ████     ████     ████     ████
   100 ┤    ████     ████     ████     ████     ████
       └────────────────────────────────────────────
         10 ms    50 ms   100 ms   500 ms    1 s       latency per request
```

> 🧠 The slide's x-axis is just labelled 1–5. The latencies above are worked out from L = 10,000 × W (100 → 10 ms, 500 → 50 ms, 1000 → 100 ms, 5000 → 500 ms, 10000 → 1 s).

**Ten times the cost (slide):** at **100 ms** you hold **a thousand** connections. At **one second**, **ten thousand**, *for identical traffic*.

**Slowness is a memory bug (slide):** every connection in flight holds an fd, a buffer, maybe a thread stack (walls 1 and 2). So a slower backend means more concurrent connections, which means more memory. That is why **high-latency work goes asynchronous**: take it off the request path and the queue drains.

🧠 Example: a signup endpoint that sends a welcome email synchronously takes 800 ms. Put "send email" on a message queue (Kafka, RabbitMQ) and return in 30 ms. At 1,000 rps, connections in flight drop from 800 to 30.

**Gotcha 🧠:** a slow **dependency** (a DB that went from 5 ms to 500 ms) silently multiplies your connection count by 100, even though traffic didn't change. Servers then hit the fd or memory wall and fall over. This is how latency incidents turn into outages, and it's why timeouts (1.5) matter.

---

## 5 · IN THE WILD: what does our favourite software do?

### 5.1 Apache: prefork → worker → event

> It took twenty years, and it ended up at a loop too.

**MPM** = Multi-Processing Module (the slide says "Multi process module"). It is Apache's pluggable concurrency engine.

```
  PREFORK MPM                 WORKER MPM                     EVENT MPM
 ┌───────────────────┐      ┌──────────────────────┐      ┌───────────────────────────┐
 │ pool of PROCESSES,│      │ processes each       │      │ a dedicated LISTENER      │
 │ one connection    │  ─▶  │ running MANY THREADS │  ─▶  │ thread per process parks  │
 │ each              │      │ thread pinned to a   │      │ idle keep-alive sockets   │
 │                   │      │ connection, even     │      │ on EPOLL; hands a worker  │
 │                   │      │ while idle           │      │ thread only sockets with  │
 │                   │      │                      │      │ actual work               │
 └───────────────────┘      └──────────────────────┘      └───────────────────────────┘
  bulletproof; works with      cheaper per conn             DEFAULT on Unix today
  every non-thread-safe
  module ever (mod_php!);
  most expensive per conn
```

**The keep-alive problem, in one line (slide):** under **worker**, a client that has **finished its request but is holding the connection open still owns a thread**. **Event takes the socket back and gives the thread away.** That is our select loop, arriving inside Apache in **2012**. (🧠 Event MPM became stable/production-ready with Apache 2.4 in 2012.)

🧠 **Keep-alive** = HTTP/1.1 persistent connections: after a response the TCP connection stays open (often 5–15 s) for the next request, to skip a new handshake. Most of that time the connection is **idle**, and pinning a thread to idle time is pure waste.

```
 WORKER:  thread T1 ══[req]══[ idle keep-alive 15 s .......... ]══[req]══
 EVENT:   thread T1 ══[req]══ (free → serves others)            T4 ══[req]══
          listener + epoll holds the idle socket in the meantime ──────▲
```

### 5.2 HAProxy: a select loop from before epoll existed

| The timeline explains the design | It introduced the load balancer |
|---|---|
| HAProxy arrives in **2001**. **epoll only lands in Linux 2.5.44 in late 2002.** So it was largely **select**, because select was what there was. It moved to epoll later, and only became **multi-threaded in 1.8, in 2017**. | For a lot of teams this was **the first box that sat in front of the application servers and chose between them**, and it could switch between multiple upstream services. |

| One process, enormous throughput | And it absorbs connection churn |
|---|---|
| For most of its life **a single event-driven process saturated a NIC**. *That is the loop argument made as loudly as it can be made.* | It can **reduce the number of connects and disconnects on the upstream service by holding a pool open**: the persistent-connection argument from block 03, applied by a proxy on your behalf. |

```
 thousands of short client conns            few long-lived upstream conns
 c1 ─┐                                      ┌──── app1
 c2 ─┼──▶  [ HAProxy: 1 event loop ]  ══════╪──── app2
 c3 ─┘     picks a backend (round-robin,    └──── app3
 ...        least-conn, ...), reuses conns
```

🧠 "Saturated a NIC" means it pushed as much traffic as the network card could carry (e.g. 10 Gbit/s) from one single-threaded process. The CPU wasn't the bottleneck; the wire was.

### 5.3 Varnish: the first popular caching reverse proxy

| Squid cached for the client | How it actually runs |
|---|---|
| **Squid** was a caching proxy: it sat **next to the users, on the way out**. **Varnish turned it around** and put the cache **in front of the origin, on the way in**. Same mechanism, opposite direction, entirely different business. | An **acceptor thread per listening point per pool**. A **worker thread per active connection**, drawn from pools that default to a **minimum of 100 and a maximum of 5000**. **Idle descriptors** (clients between requests, backends between fetches) are handed to a **waiter**, which is **epoll on Linux and kqueue on BSD**. |

**Still everywhere:** **Fastly's edge is built on Varnish, VCL and all.** When you write a **VCL** snippet for a CDN in 2026, you are configuring a program written in **2006**.

> **So: threads for the work, a loop for the waiting.** The same split Apache arrived at with the event MPM.

🧠 Decoder:
- **Origin** = your real backend server, where the content originally comes from.
- **VCL** = Varnish Configuration Language: a small language that decides per request whether to cache, what to hash on, and which backend to use. It is compiled to C.
- **CDN** = Content Delivery Network (Week 6).

```
  Squid (forward):   [users + Squid] ───────────▶ internet sites       (cache for many sites, few users)
  Varnish (reverse): users ───────────▶ [Varnish + YOUR origin]        (cache for one site, many users)
```

### 5.4 nginx: epoll plus sendfile, never copy the bytes at all

The slide's code contrast:

```c
read(file_fd, buf, n);                 /* disk -> kernel -> your buffer   */
write(sock_fd, buf, n);                /* your buffer -> kernel -> socket */

sendfile(sock_fd, file_fd, NULL, n);   /* kernel -> socket. you never touch it. */
```

```
 read + write:                              sendfile:
 disk → [kernel page cache]                 disk → [kernel page cache]
             │ copy #1 (syscall 1)                      │
             ▼                                          │ (in-kernel, no user copy;
        [user buffer]                                   │  often zero-copy to NIC)
             │ copy #2 (syscall 2)                      ▼
             ▼                                   [socket] → NIC
        [socket buffer] → NIC
 2 syscalls, 2 copies through user space    1 syscall, 0 copies through user space
```

| Master and workers | The first reverse proxy | And a thread pool, later |
|---|---|---|
| **One master**, then **one worker process per CPU core**. Each worker is a **single thread running one epoll loop over thousands of connections**. | Then promptly **abused for caching, load balancing, TLS termination, rate limiting** and everything else. **It became the default shape of an edge.** | **Disk reads can block even in a loop.** Since **1.7.11** nginx offloads those to a **thread pool**, because a **page fault on a cache miss stalls every connection on that worker**. |

🧠 Why disk is special: epoll tells you when a **socket** is ready, but regular files always report "ready", and a `read()` of a file that isn't in the page cache **blocks** while the disk seeks. In a single-threaded loop, that block freezes thousands of other connections. So the blocking work goes to helper threads (`aio threads;` in the nginx config).

🧠 Master's job: read config, bind ports, start or restart workers (graceful reload with `nginx -s reload`). Workers do all the actual traffic. **TLS termination** = nginx decrypts HTTPS so backends can speak plain HTTP.

### 5.5 All the options, side by side

> Four teams, four starting points, thirty years apart.

| | Concurrency model | Born as | The idea it popularised | In the wild |
|---|---|---|---|---|
| **Apache** | prefork → worker → event (listener thread + epoll) | **1995**, the web server | Modules, and CGI as a way to run anyone's code | Everywhere. Akamai started here |
| **HAProxy** | select, then epoll; multi-threaded only since 1.8 | **2001**, TCP/HTTP proxy | The load balancer as a tier of its own | The default LB in front of everything |
| **Varnish** | thread per active connection + epoll waiter for idle | **2006**, HTTP accelerator | Caching reverse proxy: Squid, turned around | Fastly's edge, VCL and all |
| **nginx** | master + worker per core, one epoll loop each | **2004**, answer to C10K | sendfile: never copy the bytes into user space | The default edge. Broadpeak, and most CDNs |

> **Read the concurrency column top to bottom: four independent teams, four starting points, and every one of them ended up with an event loop and a small pool of workers.**

### 5.6 Go and read the actual code

> Every claim in the last block is a file you can open this evening.

| Repo | File | What's there |
|---|---|---|
| `apache/httpd` | `modules/generators/mod_cgi.c` L576 | `run_cgi_child()`: fork, three pipes, a timeout on each |
| `apache/httpd` | `server/util_script.c` L409 | `ap_add_cgi_vars()`: HTTP headers become the environment |
| `nginx/nginx` | `src/event/modules/ngx_epoll_module.c` | The epoll loop itself |
| `nginx/nginx` | `src/os/unix/ngx_linux_sendfile_chain.c` | `sendfile()`, where the copy does not happen |
| `haproxy/haproxy` | `src/ev_epoll.c` | The poller, one file per platform (`ev_select.c` still sits beside it) |
| `varnishcache/varnish-cache` | `bin/varnishd/waiter/cache_waiter_epoll.c` | The waiter that holds idle connections |
| `golang/go` | `src/runtime/netpoll_epoll.go` | Where your blocking read actually parks |

> *Pick one.* Reading a poller implementation end to end is a two-hour job, and it changes how you read every server you meet afterwards. *They are all the same shape once you have seen one.*

🧠 Tip: line numbers drift between versions. If L576 doesn't land on `run_cgi_child`, search the file for the function name (on GitHub press `t` for file search, or `y` for a permalink).

---

## 6 · HANDS-ON 🧪

> ⚠️ 🧠 You're on **macOS**: `epoll`, `ss`, `strace` and `/proc` are **Linux-only**. Use a Linux VM, a cloud box, or Docker (`docker run -it --rm --ulimit nofile=200000 ubuntu bash`, then `apt update && apt install -y build-essential netcat-openbsd iproute2 strace tcpdump curl`). The macOS equivalents are given where they exist.

### 6.1 Run the fork server and watch processes appear

```sh
gcc fork_server.c -o fork_server && ./fork_server &
nc localhost 2026        # in 2–3 terminals; type lines, they echo back
ps -ef | grep fork_server   # one parent + one child per open nc
```
**Expect:** N `nc` sessions give N+1 `fork_server` processes. Close an `nc`, and its child becomes `<defunct>` (a zombie, since the code never calls `wait()`).

### 6.2 Run the select server and confirm it's ONE process

```sh
gcc select_server.c -o select_server && ./select_server &
nc localhost 2026        # several terminals, all work at once
ps -L -p $(pgrep select_server)    # Linux: one process, ONE thread (LWP)
lsof -p $(pgrep select_server) | grep TCP   # one fd per client + the listener
```
**Expect:** a single process and thread, but an extra fd for every client.

### 6.3 Check your limits

```sh
ulimit -n        # soft fd limit  → 1024 on Linux, often 256 on macOS
ulimit -Hn       # hard fd limit
ulimit -s        # thread stack size in KB → 8192 (= 8 MB) on most Linux
ulimit -n 65535  # raise the soft limit for this shell (up to the hard limit)
cat /proc/sys/fs/file-max   # Linux system-wide cap
```

### 6.4 Watch the cost of syscalls

```sh
strace -c -f ./select_server     # Linux: run some nc traffic, then Ctrl-C
```
**Expect:** a table counting `select`, `read`, `write`, `accept` calls and the time spent in each. One `select` + one `read` + one `write` per echoed message is the "kernel tax" from wall 3.

### 6.5 Count connections in flight

```sh
ss -s                       # Linux summary: "TCP: N (estab X, ...)"
ss -tn state established '( sport = :2026 )' | wc -l
netstat -an | grep 2026 | grep ESTABLISHED | wc -l    # macOS
```

### 6.6 Timeouts from the client side

```sh
curl --connect-timeout 2 --max-time 5 http://10.255.255.1/   # unroutable → fails after ~2 s instead of hanging
curl --retry 3 --retry-delay 1 --max-time 5 http://localhost:8080/
```

### 6.7 Try CGI in 30 seconds (no Apache) 🧠

```sh
mkdir -p cgi-bin && cat > cgi-bin/hello.cgi <<'EOF'
#!/bin/sh
echo "Content-Type: text/plain"
echo ""
echo "pid=$$ method=$REQUEST_METHOD query=$QUERY_STRING ua=$HTTP_USER_AGENT"
EOF
chmod +x cgi-bin/hello.cgi
python3 -m http.server --cgi 8000        # (works up to Python 3.14; --cgi was removed in 3.15)
curl 'http://localhost:8000/cgi-bin/hello.cgi?x=10'
curl 'http://localhost:8000/cgi-bin/hello.cgi?x=10'
```
**Expect:** the `pid=` value is **different on every request**, which is proof of fork + exec per hit. You can also see `HTTP_USER_AGENT=curl/...`, the header-to-env conversion.

### 6.8 sendfile in the wild

```sh
strace -f -e trace=sendfile,read,write -p $(pgrep -f 'nginx: worker')   # Linux, while curling a static file
```
**Expect:** `sendfile(...)` calls instead of `read` + `write` pairs (needs `sendfile on;` in nginx.conf, which is the default in most distro configs).

---

## 7 · HOMEWORK (from the slides)

> Four things. Do **at least the first two**, because they are the ones that leave a mark.

### HW 1 · Break the select server
> Take the code from slide 05 and open **1,100 connections** to it. Find where it dies and prove to yourself why. Then rewrite the same loop with **epoll** and do it again with **100,000**.

(🧠 The select code is printed on slide 04; slide 05 is the FD_SETSIZE slide that explains it. Same server either way.)

**Approach / hints**
1. **Generate the connections** with a tiny client that just opens sockets and holds them:
   ```python
   import socket, time
   socks = []
   for i in range(1100):
       s = socket.create_connection(("127.0.0.1", 2026))
       socks.append(s)
       if i % 100 == 0: print(i)
   time.sleep(3600)
   ```
   The *client* also needs `ulimit -n` > 1100 in its own shell.
2. **Predict three possible failure points**, then work out which you hit first:
   - **Server `ulimit -n` = 1024**: fds 0–2 + the listener use 4, so around **1020 clients** in, `accept()` returns **-1 (`EMFILE`)**. The code doesn't check, so it stores `-1` in `clients[]`, and `FD_SET(-1, ...)` is undefined behaviour. Also the listener stays "readable" forever, so the loop spins at 100% CPU.
   - **Raise the server's ulimit** (`ulimit -n 4096`) and try again. Now fds reach **1024**, and `FD_SET(1024+, ...)` writes past the 128-byte `fd_set`: stack corruption, odd behaviour or a crash.
   - **`clients[FD_SETSIZE]` overflows** at the 1025th client: `nclients++` with no bounds check writes past the array.
3. **Prove it:** use `strace -e accept,select ./select_server` to see `EMFILE`, `ls /proc/<pid>/fd | wc -l` to count fds, and print `client_fd` values as you accept them. Try building with `-fsanitize=address` or `-D_FORTIFY_SOURCE=2 -O2` (glibc can then abort on `FD_SET` with fd ≥ 1024, "buffer overflow detected").
4. **epoll rewrite:** `epoll_create1`, `epoll_ctl(ADD)` on accept, `epoll_wait` loop, and remove/close on `read <= 0` (see the skeleton in 1.3). Replace the fixed array with nothing at all, because the kernel keeps the set.
5. **For 100,000:** raise `ulimit -n` (and the hard limit / `fs.file-max`) on both sides. A single client IP connecting to one server port runs out of **ephemeral ports** (about 28k by default, `net.ipv4.ip_local_port_range`) — the Lesson 1 problem again. Use several client source IPs (`127.0.0.2`, `127.0.0.3`... all work on Linux loopback) or several server ports. Watch memory with `free -m` and `ss -s`.

### HW 2 · Watch CGI cost you
> Write a CGI script in any language, run it under Apache with **mod_cgi**, and put `ps` or `top` on a second screen while you `curl` it in a loop. Count the processes appearing and disappearing.

**Approach / hints**
- Install Apache (`apt install apache2`, then `a2enmod cgi` on Debian/Ubuntu; or Homebrew `httpd` on macOS and uncomment `LoadModule cgi_module` / `cgid_module` in `httpd.conf`). Put the script in the `cgi-bin` directory (`/usr/lib/cgi-bin/` on Ubuntu), `chmod +x`.
- Make the script a little slow (`sleep 0.2`) so processes stay visible long enough to see.
- Loop: `while true; do curl -s localhost/cgi-bin/hello.cgi > /dev/null; done` (run 2–3 loops in parallel).
- Watch: `watch -n 0.2 'ps -ef | grep hello.cgi | grep -v grep'`, or `top` sorted by PID. Print `$$` in the script and note that the PID increases on every hit.
- Measure: compare `ab -n 1000 -c 10` (ApacheBench) or `time` for the CGI script against a static file of the same size. The gap is the fork + exec cost.
- 🧠 On threaded MPMs Apache uses **mod_cgid** (a separate daemon does the forking) instead of mod_cgi, but you'll still see one new process per request.

### HW 3 · Find the FastCGI records
> Run **php-fpm** behind **nginx**. `tcpdump` the loopback socket between them and find **`FCGI_BEGIN_REQUEST`** and the **empty PARAMS record** in the bytes.

**Approach / hints**
- Make php-fpm listen on **TCP** (`listen = 127.0.0.1:9000` in the pool config, e.g. `www.conf`). tcpdump can't see a Unix socket.
- nginx: `fastcgi_pass 127.0.0.1:9000;` (see the config in 2.6). A `test.php` with `<?php echo "hi"; ?>`.
- Capture: `sudo tcpdump -i lo -X -s0 'tcp port 9000'`, then `curl -d 'name=alice&age=20' 'localhost/test.php?x=10'`. Or capture with `-w fcgi.pcap` and open it in **Wireshark**, which has a FastCGI dissector (Decode As → FCGI).
- What to look for (the 8-byte header from 2.7):
  - `01 01 00 01 00 08 00 00` → version 1, **type 1 = BEGIN_REQUEST**, requestId 1, length 8.
  - `01 04 00 01 xx xx ..` → PARAMS with content (you'll see `REQUEST_METHOD`, `POST`... in the ASCII column).
  - `01 04 00 01 00 00 00 00` → **the empty PARAMS record** (length 0 = "params finished").
  - `01 05 00 01 00 11 ..` → STDIN with length `0x11` = **17** = `name=alice&age=20` ✓, then `01 05 00 01 00 00` (empty STDIN).
  - Response direction: `01 06` (STDOUT), `01 03` (END_REQUEST).

### HW 4 · Measure Little's Law on something you own
> Pick a service. Note its **requests per second** and its **p50 latency**, predict the **connections in flight**, then check with **`ss -s`**. Then **make it slower on purpose** and watch the prediction hold.

**Approach / hints**
- 🧠 **p50** = median latency: half the requests are faster than this.
- A simple service: a tiny Flask/Go/Node app with `sleep(X)` in the handler, so you control W.
- Load generator at a *fixed rate* (not "as fast as possible"): `wrk2 -R 500`, `hey -q`, `vegeta attack -rate=500/s`, or a Python script. Fixed rate means λ stays constant.
- Predict: L = λ × W. E.g. 500 rps × 0.050 s = **25** connections. Check: `ss -tn state established '( sport = :8080 )' | wc -l` (or `ss -s`) while load runs.
- Now set the sleep to 200 ms, giving a prediction of 500 × 0.2 = **100**. Re-measure.
- Gotchas: keep-alive connections that are idle still count in `ss`. Use the right latency (time spent *inside* the server). And if the server hits a thread or worker limit, requests start queuing, W grows, and you can watch that cascade happen.

---

## 8 · THE FOUR IDEAS (closing slide): "The loop is the architecture"

| # | Idea | In one line |
|---|---|---|
| 01 | **Amortise the setup** | Pre-fork, pre-thread, FastCGI, connection pools, servlets: all the same move, *take the expensive setup off the request path and pay it once at startup.* |
| 02 | **Nothing is free per connection** | A descriptor, a stack, a buffer, a scheduler entry. *Multiply by ten thousand before you choose a design.* That multiplication is the whole discipline. |
| 03 | **Framing is still length or delimiter** | FastCGI uses a length inside a record and an empty record between them. Same two options as SMS and SMTP. *There is never a third.* |
| 04 | **Concurrency = throughput × latency** | You can't configure your way past this one. *Getting faster is the only thing that reduces what you are holding.* |

> **Next session:** we pick up at the **load balancer** and go looking for **state**.

---

## 9 · CHEAT-SHEET 📋

| Term | One-line meaning |
|---|---|
| **fork per request** | Clone the whole process per client: isolated, simple, most expensive |
| **thread per request** | New thread per client: cheaper, but still a stack (8 MB reserved) each |
| **pre-fork / pre-thread** | Create the worker pool at startup so creation cost is off the request path |
| **event loop** | One thread asks the kernel "who's ready?" and serves only those |
| **`select()`** | Readiness API with fixed 1024-bit `fd_set`; rebuild every call; O(n) |
| **`FD_SETSIZE`** | 1024: max fd *value* select can represent (compiled in) |
| **`FD_ZERO / FD_SET / FD_ISSET`** | Clear the set / add an fd / test an fd after select |
| **`epoll`** | Linux readiness API: register once (`epoll_ctl`), get only ready fds (`epoll_wait`); O(ready) |
| **kqueue / IOCP / io_uring** | Same idea on BSD+macOS / Windows / modern Linux (batched rings) |
| **`SO_REUSEPORT`** | Many sockets bind one port; the kernel spreads new connections across workers |
| **timeout (select's 5th arg)** | `NULL` = block forever, so one wedged client stalls the loop |
| **retry with backoff** | Client sets a deadline, retries with growing (jittered) waits |
| **CGI** | Fork + exec your program per request; headers → env vars, body → stdin, stdout → client, stderr → log |
| **`HTTP_USER_AGENT`** | Header naming rule: uppercase, `-`→`_`, prefix `HTTP_` |
| **`REQUEST_METHOD` / `QUERY_STRING` / `CONTENT_LENGTH`** | Core CGI variables: verb / text after `?` / body size |
| **blank line** | Delimiter between CGI/HTTP headers and body |
| **mod_php / mod_perl** | Interpreter compiled into Apache: fast, but one segfault kills both |
| **FastCGI** | Long-lived app process; web server talks to it over a Unix/TCP socket with typed binary records |
| **php-fpm** | PHP's FastCGI process manager: a pre-forked pool of PHP workers |
| **upstream** | The internal server(s) a front-end proxy forwards to |
| **FCGI_BEGIN_REQUEST / PARAMS / STDIN / STDOUT / STDERR / END_REQUEST** | FastCGI record types; an empty record ends a stream |
| **servlet (Tomcat/Jetty)** | JVM is the web server; pre-threaded pool; isolation by class loader, not process |
| **stateless (HTTP)** | Each request is self-contained, so any box can answer and horizontal scaling is easy |
| **persistent connection** | DBs/MQTT/Kafka keep connections open to avoid connect + handshake + auth per query |
| **RESP / MQTT / Kafka protocol** | Non-HTTP wire protocols for Redis / IoT pub-sub / Kafka |
| **`ulimit -n`** | Per-process fd limit; default soft limit still 1024 (`EMFILE` when hit) |
| **C10K / C10M** | 10k concurrent conns (Kegel, 1999) / 10M by bypassing the kernel (Graham, 2013) |
| **8 MB vs 2 KB** | OS thread default stack vs goroutine initial stack (~4000×) |
| **netpoller** | Go runtime's epoll-based poller that parks goroutines on blocking reads |
| **kernel boundary tax** | Every syscall costs hundreds of ns; worse after Spectre/Meltdown |
| **DPDK / netmap / mTCP / Seastar** | Kernel-bypass / user-space networking stacks |
| **`sendfile()`** | Kernel copies file → socket directly; no user-space round trip |
| **Little's Law** | connections in flight = requests/s × seconds/request (L = λW) |
| **"slowness is a memory bug"** | Higher latency means more concurrent connections, so more memory |
| **Apache MPMs** | prefork (process/conn) → worker (thread/conn) → event (listener + epoll for idle) |
| **keep-alive problem** | Idle persistent connection pinning a thread; event MPM fixes it |
| **HAProxy** | 2001; select then epoll; made the load balancer its own tier; MT since 1.8 (2017) |
| **Varnish** | 2006 caching reverse proxy; worker threads (100–5000) + epoll/kqueue waiter for idle |
| **Squid** | Forward caching proxy, near the users |
| **VCL** | Varnish Configuration Language (still used at Fastly) |
| **nginx** | 2004; master + 1 single-threaded epoll worker per core; sendfile; thread pool for disk since 1.7.11 |

**Key numbers:** `1024` (FD_SETSIZE = ulimit = "K") · `2026` (our port) · `8 MB` / `2 KB` · `80 GB` (10k threads) · `Linux 2.5.44` (epoll, 2002) · `1999 / 2013` (C10K / C10M) · `2–3 M` (today) · `100 / 5000` (Varnish pools) · `1.8` (HAProxy MT) · `1.7.11` (nginx thread pool) · `17` (bytes in `name=alice&age=20`).

---

## 10 · SELF-CHECK QUESTIONS ❓

**1. Name the three ways to serve many clients from the lecture, and which one scales.**
<details><summary>Answer</summary>

Fork per request (a process per client), thread per request (a thread per client), and one loop with many sockets (select/epoll). The loop is the one that scales. In production you combine them: a pre-forked or pre-threaded pool where each worker runs its own epoll loop.
</details>

**2. In the fork server, why does the parent call `close(client_fd)` right after `fork()`? What happens if it doesn't?**
<details><summary>Answer</summary>

After `fork()` both parent and child hold a copy of the fd. The TCP connection only closes when all copies are closed. If the parent keeps its copy, it leaks one fd per client, so connections never fully close when the child exits, and the parent soon hits the `ulimit -n` (1024) wall and `accept()` fails with `EMFILE`.
</details>

**3. Why does the select loop rebuild the `fd_set` (`FD_ZERO` + `FD_SET`) on every iteration?**
<details><summary>Answer</summary>

`select()` modifies the set in place. On return, only the ready fds are still set. To watch all clients again next time, you have to rebuild it. This is also part of why select is O(n) per call: you pass all n fds, and the kernel scans them all, every time.
</details>

**4. Explain `clients[i] = clients[--nclients]; i--;`. Why is the `i--` needed?**
<details><summary>Answer</summary>

It's an O(1) delete: move the last element into the removed slot and shrink the count, instead of shifting the array. `i--` is needed because a new, not-yet-checked fd now occupies index `i`. Without it, the `for` loop's `i++` would skip that fd for this round.
</details>

**5. What is FD_SETSIZE, why is it a "compiled-in" ceiling, and does it limit the *number* of fds or their *values*?**
<details><summary>Answer</summary>

`fd_set` is a fixed-width bitmap. On glibc it is 1024 bits, so `FD_SETSIZE = 1024`. The size is fixed when the program is compiled. It limits the fd **value**: any fd numbered ≥ 1024 cannot be represented, even if you only watch a few sockets. Using one is undefined behaviour (it writes past the bitmap).
</details>

**6. Why is epoll "O(ready)" while select is "O(n)"? When does that difference matter most?**
<details><summary>Answer</summary>

With select you hand the kernel all n fds on every call, and it scans them all. With epoll you register each fd once (`epoll_ctl`), the kernel keeps the interest list, and `epoll_wait` returns only the fds that fired. The cost is proportional to the ready count, not the total. It matters most with many mostly-idle connections (keep-alive, chat, IoT): 10,000 idle sockets cost select as much as 10,000 busy ones, and cost epoll nothing.
</details>

**7. What does passing `NULL` as select's last argument mean, and why is it dangerous? What's the client-side equivalent rule?**
<details><summary>Answer</summary>

The last argument is the timeout. `NULL` means block forever. If something wedges (in a blocking design, one stuck client), the whole loop and every connection on it stops, and you never get a chance to do housekeeping such as closing idle clients. Client-side rule: always set a deadline, then retry with backoff (ideally exponential with jitter, and only for idempotent operations).
</details>

**8. Walk through how Apache runs a CGI script for `POST /f.cgi?x=10` with body `name=alice&age=20` and `User-Agent: curl`. Where does each piece of the request go, and where does the output go?**
<details><summary>Answer</summary>

Apache creates three pipes and forks + execs the script (`run_cgi_child()` in `mod_cgi.c`). The method, query and headers become environment variables (`ap_add_cgi_vars()`): `REQUEST_METHOD=POST`, `QUERY_STRING=x=10`, `HTTP_USER_AGENT=curl`, `CONTENT_LENGTH=17`. The body `name=alice&age=20` is written to the script's **stdin**. The script prints headers, a blank line and the body to **stdout**. Apache parses the headers up to the blank line and streams the rest to the client. Anything on **stderr** goes to the Apache error log. Each pipe has a timeout so a hung script can't hold the worker forever.
</details>

**9. CGI is slow. Compare the two fixes (mod_php vs FastCGI): what does each trade off?**
<details><summary>Answer</summary>

mod_php/mod_perl compile the interpreter into the web server, so there is no process per request. But the app shares an address space with the web server, so one segfault takes both down, and it ties you to non-thread-safe models (prefork). FastCGI keeps a separate long-lived pool of app processes (php-fpm) that the web server talks to over a Unix or TCP socket with a compact binary protocol. You get both speed (no per-request fork) and isolation, and the front server can absorb slow clients and TLS while a small fixed pool of workers does the real work.
</details>

**10. How does FastCGI use *both* framing methods? Why is an empty record a better delimiter than a byte pattern?**
<details><summary>Answer</summary>

**Length:** every record's 8-byte header carries a content length, so a body can contain any byte (newlines, nulls). `CONTENT_LENGTH=17` does the same one level up. **Delimiter:** a record of length zero ends a stream (empty PARAMS = params done, empty STDIN = body done, empty STDOUT = output done), which lets you stream data of unknown total size. An empty record is a **structural** marker, so it can never collide with content and needs no escaping. A byte pattern like SMTP's lone `.` must be escaped when it shows up in the data (dot-stuffing).
</details>

**11. Why can you horizontally scale an HTTP server "for free-ish", but a database scaling problem "changes shape"?**
<details><summary>Answer</summary>

HTTP is stateless: every request carries everything it needs, so any box can answer, connections can be closed at will, and a load balancer can send the next request anywhere. Adding a server to the pool is enough. Databases (Redis, MySQL) want persistent connections, because each new one costs a TCP connect + handshake + auth + session setup. So the question stops being "requests per second" and becomes "how many persistent connections can one box hold?", which runs straight into the fd and memory walls.
</details>

**12. A service handles 2,000 req/s at 50 ms. Its database slows down so requests take 500 ms. Traffic is unchanged. What happens to connections in flight, and why does the lecture call slowness "a memory bug"? What's the fix?**
<details><summary>Answer</summary>

Little's Law: L = λ × W. Before: 2,000 × 0.05 = **100** connections. After: 2,000 × 0.5 = **1,000** connections, ten times more for the same traffic. Each in-flight connection holds an fd, buffers and maybe a thread stack (8 MB reserved), so memory and fd use grow 10×, and you may hit `ulimit -n` (1024) or run out of memory. The fix is to get faster (the only real lever) or to move high-latency work asynchronous (off the request path, e.g. to a queue) so the queue drains. Timeouts also cap how long a request can hold resources.
</details>

**13. Apache's worker MPM vs event MPM: what is "the keep-alive problem", and how does event solve it?**
<details><summary>Answer</summary>

Under worker, each connection is pinned to a thread, including while a client that finished its request just holds the keep-alive connection open, idle. Threads (and their stacks) are wasted on idle time. Event MPM adds a dedicated listener thread per process that parks idle keep-alive sockets on epoll and hands a worker thread only the sockets that have actual work. The thread is given back to the pool. That is the select loop arriving inside Apache (2012).
</details>

**14. Why does nginx use `sendfile()`, and why did it still need a thread pool (since 1.7.11) despite being event-driven?**
<details><summary>Answer</summary>

`read()` + `write()` copies file data disk → kernel → user buffer → kernel → socket: two syscalls and two copies through user space. `sendfile(sock, file, NULL, n)` moves data kernel → socket directly, with no user-space round trip, so fewer syscalls and copies (wall #3). But disk reads can block even in an event loop. If the file isn't in the page cache, the read page-faults and the single-threaded worker stalls, along with every connection on it. So nginx offloads those blocking disk reads to a thread pool.
</details>

**15. Design question: every server in block 05 "ended up with an event loop and a small pool of workers." Why does that combination keep winning?**
<details><summary>Answer</summary>

The loop makes **waiting** cheap: idle or slow connections cost only an fd and a little buffer in epoll/kqueue, not a thread or process. The small pool supplies **work capacity and isolation**: about one worker per core uses all the CPUs without excess context switching, separate processes give crash isolation, and threads can absorb blocking work (disk, CPU-heavy tasks) without stalling the loop. Creating the pool at startup amortises the setup cost. It combines all four closing ideas: amortise setup, nothing is free per connection, and concurrency = throughput × latency (holding many connections must be cheap).
</details>

---

*Slides could all be read clearly. The chart on slide 24 had no x-axis labels; the latencies in 4.4 are worked out from L = λW. The CGI diagram on slide 08 was checked against the PDF image.*
