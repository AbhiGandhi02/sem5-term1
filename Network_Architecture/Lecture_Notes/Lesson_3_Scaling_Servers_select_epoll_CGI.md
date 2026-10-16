# 📓 One Process, Many Clients — Scaling Servers
### *CN at Scaler — Lesson 3 (Week 3: Server-side considerations for scale)*

> `select · epoll · CGI · FastCGI` · "and what your favourite server actually does"
> **fork → thread → loop** · **CGI → mod_php → FastCGI** · **1024 · 8 MB · L = λW**

**Legend:** 🧠 = my addition, *not on the slides* · 💡 **Simple words mein** = an easy (sometimes Hinglish) summary. Read it first if a section feels heavy.

---

## ⭐ What this lecture is about (in simple words)

Lesson 1's echo server could talk to **only one client at a time**. Real servers must talk to **thousands at once**, on one machine. So the question is: **how can one machine handle 10,000 clients at once?**

> 💡 **Simple words mein:** Ek dukaan hai, ek dukaandaar hai, aur 10,000 customers aa gaye. Har customer ke liye naya dukaandaar rakhoge (fork/thread)? Bahut mehenga padega. Ya ek smart dukaandaar rakhoge jo dekhta rahe ki *kaun ready hai* aur sirf usi ko serve kare (select/epoll)? Ye lecture yahi samjhata hai.

The lecture has five blocks:

```
  01  MULTIPLEXING        fork vs thread vs ONE LOOP (select / epoll) · timeouts & retries
  02  CGI / FASTCGI       how a web server runs YOUR code (and why forking per request hurts)
  03  WEB SERVERS?        not everything speaks HTTP: databases, MQTT, Kafka
  04  LIMITS              the four walls: file descriptors · memory · context switches · the queue
  05  IN THE WILD         Apache · HAProxy · Varnish · nginx: all ended up as "a loop + a small pool"
```

**Why care?** Every framework (Express, Flask, Spring, Go's `net/http`) is built on one of these designs. Know the design, and you can predict its limits, read its config, and debug it under load. The slides end with: **"The loop is the architecture."**

---

## 📚 Course map (from the slides)

| Week | Topic |
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

## 0 · BASICS YOU NEED FIRST 🧠

The slides assume you already know these words. If anything in the lecture felt confusing, read this part first.

### 0.1 Process vs. thread

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

- A **process** is a running program with its own separate memory. If it crashes, only that process dies.
- A **thread** is a worker *inside* a process. All threads of a process share the same memory. So threads are cheaper to create than processes. The downside: if one thread does something bad (like a crash), the whole process can die.
- 💡 Process = alag restaurant, apni kitchen. Thread = usi kitchen mein ek aur cook: sasta, par ek cook ne aag lagayi toh poora restaurant band. 🔥

### 0.2 `fork()` and `exec()`

- `fork()` makes a **copy** of the current process. It "returns twice": it returns `0` inside the child (the copy) and returns the child's PID (process ID) inside the parent.
- `exec()` (for example `execve`) **replaces** the program running in the current process with a different program. The PID stays the same, and open files stay open.
- Your shell runs every command like this: **fork, then exec**. CGI pays this same cost on every web request (block 02).
- Modern kernels use **copy-on-write**: after `fork()`, parent and child share memory pages until one of them writes to a page. Even so, fork is still expensive, because the page tables, the fd table and scheduler data all get copied.

### 0.3 File descriptors (fds)

A **file descriptor** is a small number your program uses to refer to an open file, pipe or **socket**. `0` = stdin, `1` = stdout, `2` = stderr, so your first socket is usually `3`. **Every TCP connection your server keeps open uses one fd.** The OS limits how many fds one process can have. That limit is "wall #1" in block 04.

### 0.4 Blocking I/O

By default, `accept()`, `read()` and `write()` are **blocking**. That means your thread goes to sleep and waits until something happens (a client connects, data arrives, there's space to write).

```
 thread: read(fd_A) ──── zzz (A says nothing for 30 s) ────▶ returns
         meanwhile B, C, D are all waiting for you and nobody serves them
```

This one problem is the whole lecture. Three fixes: (1) a process per client (fork), (2) a thread per client, (3) ask the kernel *"which sockets are ready?"* and only touch those (select/epoll).

### 0.5 User space vs. kernel space, and system calls

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

- Your program runs in **user space**; the OS core (the **kernel**) runs in **kernel space** and alone can touch the network card, disk, etc. A **system call** (syscall) is your program asking the kernel to do something (`read`, `write`, `accept`…). Each syscall crosses the user/kernel boundary, and **every crossing costs some time** (block 04, wall #3).
- A **context switch** is when the CPU stops running one thread and starts running another. The kernel saves the registers, maybe switches page tables, and the CPU caches go "cold" (they have the wrong data loaded). The lecture uses "context switch" loosely for both this and the user-to-kernel crossing.

### 0.6 Pipes and environment variables

- A **pipe** is a one-way channel for bytes between two programs. It has a write end and a read end (two fds). The shell's `ls | grep x` uses a pipe.
- **Environment variables** are `KEY=VALUE` strings a program gets when it starts (try `echo $HOME`). The child gets them at `exec` time. CGI uses both pipes and env vars (block 02).

### 0.7 Stateless vs. stateful protocols

- **Stateless**: every request carries everything needed to answer it. The server doesn't need to remember you. HTTP is stateless (cookies/tokens put your "state" *inside* each request).
- **Stateful**: the connection itself remembers things (you logged in, you selected database 3, you subscribed to a topic). Databases and MQTT are stateful.

### 0.8 Caches and proxies

- A **cache** keeps a copy of something that was expensive to get, so next time it's cheap.
- A **forward proxy** sits near the **users** and makes requests for them (like a college/office proxy, e.g., Squid).
- A **reverse proxy** sits in front of the **servers** and receives requests for them (nginx, Varnish, HAProxy).

```
  users ──▶ [forward proxy] ──▶ internet ──▶ [reverse proxy] ──▶ app servers
            (Squid)                          (nginx / Varnish / HAProxy)
```

### 0.9 Framing (from Lesson 2)

TCP is just a **stream of bytes**. It doesn't mark where one message ends and the next starts. So every protocol has to decide how to mark the end of a message. There are only two ways:
- **Length**: "the next 17 bytes are the message", or
- **Delimiter**: "keep reading until you see a blank line / a dot".

The slides say there's no third option. FastCGI uses both (block 02).

---

## 1 · MULTIPLEXING: three ways to serve many clients

### The headline: "A thread per client is a bet you lose at ten thousand"

There are three ways to handle many clients, and only one of them works at large scale.

| Model | What it does | Cost / verdict (from the slides) |
|---|---|---|
| **Fork per request** | Copy the whole process for every client | Simple and isolated, but the most expensive thing you can do per connection |
| **Thread per request** | Make a new thread for every client | Cheaper than fork, so faster. But each thread still needs its own stack (memory) before doing any work |
| **One loop, many sockets** | `select` or `epoll`: one thread watches all sockets and only acts on the ready ones | **This is the one that scales** |

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

**What's used in real life:** usually not one model alone. We **create a pool of processes or threads in advance (pre-fork / pre-thread)**, and each worker runs its own `select`/`epoll` loop. *"Isolation from the processes, concurrency from the loop."*

**Pre-fork / pre-thread** = create the processes/threads **at startup, before any client arrives**, instead of creating one each time a client connects. This removes the creation cost from the time the user is waiting.

> 💡 **Simple words mein:** Fork = har customer ke liye naya restaurant. Thread = naya waiter. Loop = ek smart waiter jo sirf utha hua haath dekh ke serve karta hai. ✅ Real life: kuch waiters pehle se hire karo (pre-fork), aur har waiter smart loop chalaye.

---

### 1.1 `fork()`: a whole process per client

**Idea:** every time a client connects, make a copy of the whole server, and let the copy handle that client.

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

**Reading the code (from the slide):**
- `listen(server_fd, 1)`: a **backlog of one**. At most about one connection can wait in the queue to be accepted.
- The **child** echoes data back, then exits.
- The **parent** closes its copy of `client_fd` and goes back to `accept()` to wait for the next client.

```
 parent                         child (copy)
 ──────                         ────────────
 accept() → fd 4
 fork() ─────────────────────▶  fork() returned 0
 close(4)  (parent's copy)       read(4) / write(4) loop
 accept() again ...              close(4); exit
```

**What it costs (slide):**
- A full copy of the process for every connection.
- Page tables, fds and a scheduler entry all get copied.
- **You pay this cost while the user is waiting**, because the fork happens *after* `accept`.

**Why use it at all?** It's the simplest way to handle many clients, and the isolation is perfect: if one child crashes, other clients are not affected.

**Common mistakes / gotchas 🧠**
- **Why must the parent `close(client_fd)`?** After `fork`, both parent and child have the fd. The TCP connection only really closes when *both* copies are closed. If the parent forgets, it leaks one fd per client and soon hits the fd limit (block 04).
- **Zombies.** The parent never calls `wait()`, so every child that exits stays as a "zombie" entry in the process table. Real code handles `SIGCHLD` or uses `signal(SIGCHLD, SIG_IGN)`.
- The child also gets a copy of `server_fd`. Clean code closes it in the child.
- Backlog 1 does **not** mean "one client at a time". Accepted clients go to children immediately. It only limits how many *not-yet-accepted* connections can wait (and Linux rounds it a bit).

---

### 1.2 `select()`: one thread, every client

**Idea:** instead of waiting on one socket, give the kernel a **list of sockets** and say *"wake me up when any of these has something for me."* Then handle only those sockets, and ask again.

Analogy 🧠: a waiter with 20 tables doesn't stand at table 1 until they order. He looks around the room, sees which hands are raised, and serves only those tables.

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

**The loop as a picture:**

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
| `FD_ZERO(&set)` | clear the whole set |
| `FD_SET(fd, &set)` | add `fd` to the set ("watch this one") |
| `FD_ISSET(fd, &set)` | after `select`: is `fd` still in the set? ("is it ready?") |
| `FD_CLR(fd, &set)` | remove one fd from the set |

The first argument to `select` is `maxfd + 1`, meaning "check fd numbers 0 up to maxfd". The kernel needs to know how far to look.

**"Two lines worth stopping on" (slide)**

1. `clients[i] = clients[--nclients]; i--;` is an **O(1) delete** (constant time, very fast). Instead of shifting the whole array left, it copies the **last** element into the empty spot and reduces the count by one. `i--` makes the loop check index `i` again, because a different fd (not yet checked) is now sitting there.
   ```
   before:  [ 4 | 5 | 6 | 7 ]  n=4   remove index 1 (fd 5)
   after:   [ 4 | 7 | 6 ]      n=3   i-- → next iteration checks fd 7
   ```
   🧠 The order of clients changes, but that's fine here because order doesn't matter.
2. `select(..., NULL, NULL, NULL)`: **that last NULL is the timeout.** NULL means "wait forever" (see 1.5).

**Why use it?** One thread serves many clients. No process or stack per client. An idle client costs only one bit in a bitmap and one fd.

**Common mistakes / gotchas 🧠**
- **`select` changes your `fd_set`.** When it returns, the set contains only the ready fds. That's *why* the code rebuilds the set every time with `FD_ZERO`/`FD_SET`. If you forget, you lose track of clients.
- **One slow operation can freeze everyone.** `write(fd, ...)` can still block. If a client stops reading, its send buffer fills up, `write` blocks, and the whole loop (and every other client) freezes. Real event loops make sockets **non-blocking** (`O_NONBLOCK`) and also watch `writefds`.
- The code never checks if `accept` returned `-1`, or if `nclients` reached `FD_SETSIZE`. Homework #1 makes you break exactly this.

---

### 1.3 select's limit is built in: `FD_SETSIZE = 1024`

**Idea:** `select` uses a fixed-size bitmap (a row of bits), so there's a hard maximum fd number it can watch. And it's small.

```c
int clients[FD_SETSIZE];   /* not a style choice: this IS the hard limit */
```

- `fd_set` is a **fixed-size bitmap**: **1024 bits** on glibc (Linux's standard C library). So **fd number 1024 can't fit**; only 0–1023 are allowed.

```
 fd_set (1024 bits = 128 bytes)
 bit:  0 1 2 3 4 5 6 ...                                   1023 │ 1024 → ??? (off the end)
      [0 0 0 1 1 0 1 ......................................  0 ]│
             ↑ server_fd=3, clients 4 and 6
```

🧠 **Important:** the limit is on the fd's **number**, not on how many fds you watch. Even if you watch only 5 sockets, if one of them is fd 1500, `select` can't handle it. `FD_SET(1500, ...)` writes past the end of the bitmap, which silently corrupts memory (undefined behaviour).

#### select vs. epoll

| | **select: O(n) every call** | **epoll: O(ready)** |
|---|---|---|
| Registration | You **rebuild the set every loop** | **Register each fd once**, using `epoll_ctl` |
| Kernel work | The kernel **checks every fd, ready or not** | `epoll_wait` **returns only the fds that are actually ready** |
| Idle sockets | **10,000 idle sockets cost as much as 10,000 busy ones** | **Idle connections cost nothing** while waiting |
| Max fds | 1024 (built in) | limited only by `ulimit -n` and memory |

**Same idea, other systems (slide):** **kqueue** (BSD/macOS) · **IOCP** (I/O Completion Ports, Windows) · **io_uring** (modern Linux).

🧠 Two families: **readiness** APIs (`select`/`poll`/`epoll`/`kqueue`: "fd 7 is ready, *you* call `read`") and **completion** APIs (IOCP, io_uring: "read into this buffer and tell me when it's *done*"). `poll()` takes an array instead of a bitmap (no 1024 limit) but is still O(n).

> 💡 **Simple words mein:** select = teacher jo har din poori class ka roll call karti hai, chahe 9,990 bachhe so rahe ho. epoll = teacher jo kehti hai "jisko kuch bolna hai, haath uthao", aur sirf unhi ko sunti hai. 10,000 students ho toh epoll bahut fast hai.

#### What epoll code looks like 🧠

The slides describe epoll but don't show code. Here's the basic skeleton:

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

Compared with select: no rebuilding the set, no `maxfd`, no checking idle fds. The kernel remembers the list of fds between calls and just gives you the ready ones.

🧠 **Level-triggered** (the default): epoll keeps telling you about an fd as long as it still has unread data. Same as select. **Edge-triggered** (`EPOLLET`): epoll tells you only once, when *new* data arrives. So you must keep reading until you get `EAGAIN` (on a non-blocking socket), or you might never hear about the leftover data again. nginx uses edge-triggered mode.

#### "The number to remember": 1024

> **1024.** It's `FD_SETSIZE`, it's your default `ulimit`, and it's the K in C10K. *One wall, three names.*

(🧠 "K in C10K" is a small joke: 1024 ≈ 1 "K". C10K actually means 10,000 connections at the same time. See block 04.)

---

### 1.4 In real servers, you combine all three

**Idea:** real servers use all three models together. A few processes give crash safety, and each process runs an event loop to handle lots of connections.

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

**`SO_REUSEPORT` 🧠:** a socket option (Linux 3.9+) that lets **many sockets listen on the same IP and port**. The kernel then sends each new connection to one of them. Without it, workers either share one listening socket (and all rush to `accept` the same connection, called the "thundering herd" problem) or need a lock. Don't confuse it with `SO_REUSEADDR` from Lesson 1, which is about re-using a port while old connections are still in `TIME_WAIT`.

**This is real, not theory (slide):**
- It's **Apache's event MPM** (block 05).
- It's **nginx's master + workers** design.
- It's what the **Go runtime** does for you automatically: **goroutines run on a few OS threads, managed by a scheduler that sits on top of epoll.**

---

### 1.5 Timeouts: never wait forever

**Idea:** on a network, never wait forever for anything. The other side might be dead, slow, or stuck.

The server may be busy reading, writing or accepting. **Never wait without a limit.**

In a select loop you're waiting on three kinds of things:

| Set | What you're waiting for (slide wording) |
|---|---|
| `readfds` | new data to arrive |
| `writefds` | a socket that's ready for us to write |
| `exceptfds` | sockets being connected and closed |

> 🧠 More precisely: according to the man page, `exceptfds` reports "exceptional conditions", mostly TCP urgent (out-of-band) data. A non-blocking `connect()` finishing actually shows up in `writefds`, and the other side closing shows up in `readfds` (`read` returns 0). The slide simplifies this.

```c
select(maxfd + 1, &readfds, NULL, NULL, NULL);   /* readfds, writefds, exceptfds, timeout */
```

The slide says "each of those comes with a default timeout". That's the last argument to `select()`. Our code passed `NULL`. (🧠 Strictly, `select` has one timeout for the whole call.)

**NULL timeout = wait forever.** If one client gets stuck, the whole loop stops, and so does every other connection on it.

How to give a real timeout 🧠:

```c
struct timeval tv = { .tv_sec = 5, .tv_usec = 0 };
int r = select(maxfd + 1, &readfds, NULL, NULL, &tv);
if (r == 0) {
    /* timed out: nothing happened in 5 s → do housekeeping,
       e.g. close clients idle longer than N seconds */
}
/* NOTE: on Linux select() modifies tv, so reset it every iteration */
```

#### The client side: set a deadline, then retry with backoff

The slide says the same rule applies to clients: **set a deadline, then retry with backoff.**

🧠 What that means:
- **Deadline / timeout**: for example `curl --connect-timeout 2 --max-time 5`, `requests.get(url, timeout=5)`, or `setsockopt(SO_RCVTIMEO)`.
- **Retry with exponential backoff**: wait 100 ms, then 200 ms, then 400 ms, then 800 ms… between tries. Add a bit of random **jitter** so a thousand clients don't all retry at the exact same moment and crash the server again (a "retry storm").
- **Only retry safe (idempotent) operations.** Retrying a `GET` is fine. Retrying "charge my card" could charge you twice.

```
 attempt 1 ──X timeout
       wait 0.1s ± jitter
 attempt 2 ──X timeout
       wait 0.2s ± jitter
 attempt 3 ──✓
```

**Gotcha:** a timeout protects *you*, not the server. Without retry limits and backoff, timeouts turn into a flood of retries.

---

## 2 · CGI / FASTCGI: how the web server runs *your* code

Block 01 was about handling connections. Block 02 asks: once the web server has received an HTTP request, **how does it run your application code** (a PHP or Perl script, for example) to create the response?

### 2.1 CGI: start a new copy of your program for every request

**CGI** (Common Gateway Interface) is a simple agreement: the web server **starts your program fresh for every request**, passes the details in **environment variables** and the body on **stdin**, and sends whatever your program **prints** back to the browser.

> "Apache was the most popular HTTP server on earth, and this is how it ran your code." (diagram from the slide:)

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

- **envp** (environment variables) carries the request details (method, headers, query string) **in**; the **stdin pipe** carries the **body** in.
- The **stdout pipe** carries the **response** out to Apache and the client; **stderr** goes to the **Apache error log**.

### 2.2 Server side: connect three pipes, then exec

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

`dup2(a, b)` makes fd `b` point to the same thing as `a`. So the CGI program just uses normal stdin and stdout. It never even knows there's a network involved.

**The real code (slide):**
- **`apache/httpd` · `modules/generators/mod_cgi.c` · L576 · `run_cgi_child()`**
- It creates the child, connects the three pipes, and **sets a timeout on each one**, so a stuck script can't hold the worker forever. That's the timeout lesson from 1.5 again.

### 2.3 HTTP headers become environment variables

**Idea:** the web server turns the HTTP request into `KEY=VALUE` environment variables, because every program in every language can read those.

The slide's pseudo-code:

```
Convert HTTP headers and verb to variables
E.g. REQUEST_METHOD = GET / POST, QUERY_STRING = <things after ?>
Pass these as environment variables while creating the CGI process
```

**The naming rule (slide):** the header `User-Agent: curl` becomes `HTTP_USER_AGENT=curl`. **Make it UPPERCASE, change `-` to `_`, add `HTTP_` in front.** Every framework you've used still follows this pattern.

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

🧠 `Content-Type` and `Content-Length` are special. The CGI spec (RFC 3875) turns them into `CONTENT_TYPE` / `CONTENT_LENGTH` **without** the `HTTP_` prefix. Other standard variables: `SCRIPT_NAME`, `PATH_INFO`, `REMOTE_ADDR`, `SERVER_NAME`, `SERVER_PORT`.

🧠 You still see this pattern today: Python WSGI's `environ['HTTP_USER_AGENT']`, PHP's `$_SERVER['HTTP_USER_AGENT']`, Django's `request.META['HTTP_USER_AGENT']`.

**The real code (slide):**
- **`server/util_script.c` · L409 · `ap_add_cgi_vars()`** sets `GATEWAY_INTERFACE`, `SERVER_PROTOCOL`, `REQUEST_METHOD`, `QUERY_STRING`.
- **`ap_add_common_vars()`** (just above it in the same file) sets the rest, including the `HTTP_*` headers.

### 2.4 A CGI program is just a program that prints

**Idea:** no server library, no framework. **Read the environment variables, print to stdout.** That's it.

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

**"Two lines" (slide):** first the headers, then **a blank line**, then the body. **That blank line is the delimiter**, the same framing idea from Lesson 2. Apache reads your headers up to the blank line and sends everything after it to the client.

```
 Content-Type: text/html      ← header(s)
                              ← blank line = "headers finished"   (DELIMITER)
 <h1>Hello</h1>               ← body: streamed as-is to the client
```

**That's the whole agreement.** Any language that can read an environment variable and print can serve web pages. That's why the early web was written in **Perl and shell scripts**.

**Common mistake 🧠:** if you forget the blank line, Apache returns **500 Internal Server Error**, and the error log says "malformed header from script" or "End of script output before headers". The script also needs execute permission (`chmod +x`).

### 2.5 Copy the whole program, then throw it away

**Idea:** CGI does the most expensive thing from block 01 (fork + exec) on **every single request**.

| The problem | The fix people tried | The fix that won |
|---|---|---|
| The whole program is created, runs once, and is thrown away. Your shell does the same for every command, but your shell isn't doing it a thousand times a second. | **Put the language INTO the server**: `mod_php`, `mod_perl`. No new process per request. But now your app **shares memory with the web server**, so **one crash takes both down**. | **Keep your own long-running process**, and let the web server talk to it over a **Unix or TCP socket**, using a smaller set of messages. **That's FastCGI.** |

```
 CGI:        request → fork+exec perl → load interpreter → parse script → run → exit   (EVERY time)
 mod_php:    request → (PHP already inside Apache) → run                               (fast, but shared fate)
 FastCGI:    request → socket → (php-fpm worker already running) → run                 (fast AND isolated)
```

**The pattern to notice (slide):** move the setup cost **off the request path** and pay it **once at startup**. Same trick as pre-forking in block 01.

🧠 **Unix socket vs. TCP socket:** a Unix socket is a file path (e.g., `/run/php/php-fpm.sock`). It only works on the same machine and skips the TCP/IP stack, so it's a little faster. A TCP socket (`127.0.0.1:9000`) also works between different machines.

> 💡 **Simple words mein:** CGI = har order pe naya chef bulao aur ghar bhej do (slow). mod_php = chef restaurant ke andar (fast, par uski galti = poora restaurant band). FastCGI = alag kitchen mein chefs ki team hamesha ready (fast + safe). ✅

### 2.6 FastCGI: keep the process running, talk over a socket

**Idea:** the web server and your app become **two separate programs that keep running**. They talk to each other over a socket using a compact binary protocol.

> You define an **upstream**. The user-facing server deals with slow clients; an internal server does the actual work.

```
   clients                          nginx / httpd                         php-fpm
 ┌──────────────────┐   HTTP     ┌──────────────────────┐   FastCGI   ┌────────────────────┐
 │ thousands of     │──────────▶ │ user-facing.         │───────────▶ │ internal. a fixed  │
 │ them, many of    │            │ absorbs slow         │             │ pool of long-lived │
 │ them slow        │◀────────── │ connections, TLS,    │◀─────────── │ workers            │
 └──────────────────┘            │ buffering            │             └────────────────────┘
                                 └──────────────────────┘
```

**Why split it like this? 🧠** A slow mobile user might take 10 seconds to upload a request or download a response. If a PHP worker had to wait that whole time, you'd need thousands of heavy PHP workers. Instead, nginx (a cheap event loop, block 01) **collects (buffers)** the full request from the slow client, then gives it to PHP all at once. PHP answers in 20 ms and is free again. So a small fixed group of workers can serve a huge number of users.

**php-fpm 🧠** = PHP FastCGI Process Manager. A master process that creates a pool of PHP worker processes in advance (pre-forking again).

**A smaller version of HTTP (slide):** the same information (method, headers, query string, body), but **binary, labelled, and able to carry several requests at once (multiplexed)**. No need to parse text twice.

**What you actually write (slide):** you *might* still get environment variables, but usually you get a **request object** with the headers and query string already decoded for you.

🧠 "Upstream" is nginx's word for the server(s) it forwards requests to. A minimal config:

```nginx
location ~ \.php$ {
    include        fastcgi_params;              # maps request → CGI-style params
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass   127.0.0.1:9000;              # or unix:/run/php/php-fpm.sock
}
```

### 2.7 A FastCGI request is a series of labelled records

**Idea:** instead of one big block of text, the request is sent as a series of small labelled packets called "records": a BEGIN record, the parameters, the body, and **empty records to mark where each part ends**.

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

**Same names as CGI (slide):** `REQUEST_METHOD`, `QUERY_STRING`, `CONTENT_LENGTH`. *The CGI environment survived.* Only the **delivery method** changed: **records on a socket instead of environment variables on a fork.**

**STDIN is the body (slide):** `name=alice&age=20` is **17 bytes**, exactly the `CONTENT_LENGTH` given above it. (Count: `name=alice` = 10, `&` = 1, `age=20` = 6, total 17 ✓.)

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

- **`requestId`** is what makes it "multiplexed": several requests could share one connection, and the ID tells them apart. (In practice, php-fpm handles one request per connection at a time.)
- Inside `FCGI_PARAMS`, each name/value pair is **length-prefixed** (name length, value length, name, value). So FastCGI uses length framing even inside records.
- An **empty `FCGI_PARAMS`** header looks like `01 04 00 01 00 00 00 00`: version 1, type 4 (PARAMS), request 1, content length **0**. You'll look for exactly this in homework #3.

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

- **STDOUT goes to the client:** headers, blank line, body. **Exactly what a CGI program would print**, byte for byte. The web server reads the headers and streams the rest.
- **STDERR goes to the log:** it's a **separate record type**, so your debug messages can **never accidentally appear on the web page**. Same idea as the separate pipes in CGI.

```
   CGI (pipes)                     FastCGI (record types)
   stdin  pipe   ◀── body          FCGI_STDIN   ◀── body
   stdout pipe   ──▶ client        FCGI_STDOUT  ──▶ client
   stderr pipe   ──▶ error log     FCGI_STDERR  ──▶ error log
   envp          ◀── headers       FCGI_PARAMS  ◀── headers
   process exit  = done            FCGI_END_REQUEST = done
```

### 2.9 The empty record is the delimiter

> Last session ended on framing: **length or delimiter, no third option.** FastCGI uses both, in one protocol.

| LENGTH | DELIMITER |
|---|---|
| Every record header says how long its content is. The reader knows exactly how many bytes to read *before* reading them, so the body can contain **any byte at all, even newlines and null bytes**. | A record with **length zero** means "this stream is finished". It's not a special byte pattern in the data. It's a **structural marker, so it can never clash with your content**. |
| `CONTENT_LENGTH = 17` does the same job one level higher. | The **blank line after `Content-Type`** is the same trick in text form. |

> **SMS chose length. SMTP chose a dot. IMAP chose both. FastCGI chose both too, and for the same reason.**

🧠 **Why both?** A **length** carries any binary data safely. A **delimiter** (the empty record) lets you **stream** data of unknown total size: send known-length chunks, then the "end" marker. HTTP/1.1 chunked encoding does the same (a zero-length chunk at the end). (SMTP ends with a lone `.`; IMAP uses `{length}` prefixes and line endings.)

**Gotcha:** if your delimiter is a byte pattern (like SMTP's `.`), you must **escape** it when it appears inside the content (SMTP calls this "dot-stuffing"). FastCGI's empty record needs no escaping, because it's part of the structure, not the data.

> 💡 **Simple words mein:** Length = "is dabbe mein 17 cheezein hain". Delimiter = "khaali dabba aaya, matlab khatam". FastCGI dono use karta hai.

### 2.10 Servlets: the same idea, in Java

**Idea:** this is Java's answer. Instead of a separate app process behind a web server, the **Java server *is* the web server**, and it runs your code on a pool of threads.

| The server is the runtime | Pre-threading | Isolation by convention |
|---|---|---|
| **Jetty** and **Tomcat** are the web server themselves. No separate httpd in front doing the forking. The JVM (Java Virtual Machine) is already running your code. | A **pool of threads created at startup**. A request is given to a free thread, which runs your servlet and then goes back to the pool. Same pre-threading as block 01. | Servlets are separated by **class loader and thread, not by process**. Cheaper than fork, but **one bad servlet can still hurt the others**. |

```
 Tomcat (one JVM process)
 ┌───────────────────────────────────────────────┐
 │ acceptor ──▶ request queue ──▶ thread pool     │
 │                                  T1 runs AppA  │
 │                                  T2 runs AppB  │  ← an infinite loop / OOM in AppA
 │                                  T3 idle       │    still hurts AppB (same heap)
 └───────────────────────────────────────────────┘
```

> **CGI, FastCGI, servlets, and every modern framework are answering one question: how do we run user code for each request without rebuilding everything from scratch each time?**

---

## 3 · ISN'T EVERYTHING A WEB SERVER?

### 3.1 Almost everything runs a web server, but not everything *is* one

**Idea:** lots of things speak HTTP, but many important servers use their own protocols. **The protocol decides which scaling tricks you can use.**

| ✅ YES: it speaks HTTP | ❌ NO, and it doesn't need to |
|---|---|
| Your **Wi-Fi router admin page** | **Redis**: its own text protocol, **RESP** |
| Modern **agent runtimes** and cloud tools | **MySQL**: its own binary protocol |
| Most **Node** apps | **MQTT brokers** across all IoT devices |
| **Firestore** (the popular one) | **Messenger backends**: WhatsApp-type systems |
| **CouchDB**, which has had REST for years | **Kafka itself**: a binary protocol, not HTTP |
| **Kafka**, if you put the **REST proxy** in front | |

🧠 What these words mean:
- **RESP** = REdis Serialization Protocol. Mostly text, e.g. `*2\r\n$3\r\nGET\r\n$3\r\nkey\r\n` (more length-prefixed framing).
- **MQTT** = a tiny publish/subscribe protocol for IoT (Internet of Things) devices. A sensor keeps **one long-lived TCP connection** to a broker and sends or receives messages on "topics".
- **Kafka**: a distributed log / message queue. Clients use a binary protocol over long-lived TCP connections. The **Kafka REST Proxy** is a separate HTTP service that translates.
- **REST**: a style of building APIs on HTTP (URLs + verbs like GET/POST).
- Messenger backends (WhatsApp used a modified XMPP on Erlang) keep **one open connection per online phone**.

**Why does this matter?** Everything you'll learn about scaling web servers **depends on HTTP's properties**. Change the protocol, and the advice changes too.

### 3.2 HTTP is stateless. The next thing you scale may not be.

| A web server can hang up | A database wants to stay connected |
|---|---|
| **Every request carries everything it needs.** So any server can answer any request, you can close the connection whenever you want, and a load balancer can send the next request anywhere. | **Redis or MySQL prefer to keep one connection open** instead of paying for a **TCP connect + handshake + login + session setup** on every query. |
| **Adding more servers is easy.** Add a server to the pool, done. *Statelessness is what makes this possible.* | **So the problem changes.** The question is no longer "how many requests per second?" It's **"how many open connections can one machine hold at once?"** (That's block 04.) |

```
 STATELESS (HTTP)                          STATEFUL (DB / MQTT / chat)
 client ─req─▶ LB ─▶ any of [S1 S2 S3]     client ════ one long connection ════ DB
 client ─req─▶ LB ─▶ any of [S1 S2 S3]       (auth'd, session vars, txn state)
  connection can close after each one       connection lives for minutes/hours
  scale = add boxes                         scale = how many conns can 1 box HOLD?
```

🧠 Cost of a brand-new DB connection: TCP 3-way handshake (1 RTT, round-trip time) + maybe TLS (1–2 RTT) + MySQL greeting and login + setting session variables. That's often more work than the query itself. That's why apps use **connection pools** (keep a few connections open and reuse them).

**Gotcha:** "HTTP is stateless" doesn't mean the *app* has no state. Sessions are stored in cookies, tokens, or a shared store (Redis), exactly so that any HTTP server can answer.

---

## 4 · LIMITS: four walls you will hit

> Every scaling technique in this course is a way of getting around one of these walls.

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

### 4.1 Wall 1 · File descriptors: C10K was a 1999 problem, but your default limit is still 1024

| Year | Name | Number | What happened (slide) |
|---|---|---|---|
| **1999** | **C10K** | **10,000** | **Dan Kegel** asks: can one machine hold 10,000 open connections at the same time? Back then, **no**. |
| **2013** | **C10M** | **10,000,000** | **Robert Graham** argues *the kernel is the problem, not the solution*. Skip the kernel and you can go 1,000× higher. |
| **Today** | | **2–3 M** | People have shown 2–3 million connections on a single server. *The problem moved; the default limit didn't.* |

```sh
$ ulimit -n
1024        # the soft limit on most distributions, unchanged since the problem was named
```

🧠 **What is `ulimit`?** A shell command that shows or sets limits for each process. `-n` = maximum number of open file descriptors.
- **Soft limit**: the limit that's active now. You can raise it yourself, up to the hard limit: `ulimit -n 65535`.
- **Hard limit**: the ceiling (`ulimit -Hn`). Only root can raise it.
- Permanent settings: `/etc/security/limits.conf`, systemd's `LimitNOFILE=`, or Docker's `--ulimit nofile=...`.
- System-wide limit: `/proc/sys/fs/file-max`.
- When you hit the limit, `accept()` fails with **`EMFILE` ("Too many open files")**. Your server is running, but it refuses new clients.
- 🧠 On **macOS** (your laptop), `ulimit -n` often shows **256**, even lower.

🧠 **Why is the default so low?** Partly caution, and partly because old programs that use `select()` break with fds ≥ 1024. Keeping the default at 1024 protects them. Servers like nginx raise the limit themselves (`worker_rlimit_nofile`).

**"One wall, three names":** 1024 = `FD_SETSIZE` = default `ulimit -n` = (jokingly) the K in C10K.

**Gotcha:** one connection may use **more than one fd**. A proxy holds the client's fd *plus* an fd to the backend server. You also have log files, the listening socket and the epoll fd. So a reverse proxy with a 1024 limit can only handle about 500 clients.

### 4.2 Wall 2 · Memory: every thread costs memory before doing any work

| OS thread | Goroutine (Go) |
|---|---|
| **8 MB**: default stack, **reserved up front** | **2 KB**: starting stack, **grows only when needed** |
| **10,000 threads reserve about 80 GB of address space** before storing a single byte of your data. | **About 4,000 times smaller.** This one number is a big reason **Go became popular for this kind of server**. |

Maths: 10,000 × 8 MB = 80,000 MB ≈ 80 GB. And 8 MB / 2 KB = 8192 KB / 2 KB = 4096 ≈ "four thousand".

**And underneath, it's the same loop (slide):** goroutines are light because the Go runtime **runs them on a few OS threads, using a "netpoller" built on epoll**, with **its own scheduler** on top. **You write normal blocking code; the runtime quietly puts you to sleep on epoll.**

```
  your Go code (looks blocking):          what really happens:
  ───────────────────────────            ──────────────────────────────────────
  n, err := conn.Read(buf)   ──▶  socket not ready? → goroutine PARKED,
                                  fd registered with epoll (netpoller),
                                  OS thread runs OTHER goroutines.
                                  epoll says ready → goroutine runnable again.
```

So Go gives you the **easy coding style** of thread-per-connection with the **low cost** of an event loop.

> **The more requests you handle at the same time, the more memory you need. The only question is how much per request.**

🧠 Some fine print:
- "Reserved" means **virtual address space**. Linux only uses real RAM for stack pages that are actually touched, so 10,000 idle threads don't really use 80 GB of RAM. But you still pay for page tables, kernel thread data (each thread also has a kernel stack, usually 16 KB) and scheduling work. On 32-bit systems you'd simply run out of address space. Check your default with `ulimit -s` (8192 = 8 MB on most Linux).
- Besides stacks, every connection has **socket buffers** in the kernel (send and receive, often tens to hundreds of KB under load) and **app buffers** (like `char buf[4096]`). That's "a buffer per connection".

### 4.3 Wall 3 · Context switches: every read and write crosses into the kernel

> Actually, every system call does. At a million operations a second, **this cost becomes the main work.**

| The cost | Skip the kernel | Or make fewer calls |
|---|---|---|
| User space → kernel → back again: **save state, switch page tables, check, copy, return**. **Hundreds of nanoseconds** each time, and the **Spectre and Meltdown fixes made it slower**. | Some people wrote **TCP in user space** and skipped the kernel completely, for trusted internal networks: **DPDK, netmap, mTCP, Seastar**. You **lose all your normal tools**, but you get the packets directly. | **`sendfile()`** sends a file to a socket **without passing it through your program**. **`io_uring`** sends a **batch** of operations through a **shared ring** instead of one syscall each. |

🧠 What these words mean:
- **Spectre/Meltdown** (2018): CPU security bugs. The fix for Meltdown, **KPTI** (Kernel Page-Table Isolation), switches page tables every time you enter or leave the kernel. That's the "switch page tables" cost on the slide.
- **DPDK** (Data Plane Development Kit, from Intel): your app talks directly to the network card (NIC) and checks it for packets from user space.
- **netmap**: a fast packet I/O framework (FreeBSD/Linux).
- **mTCP**: a user-space TCP stack designed for multi-core CPUs.
- **Seastar**: a C++ framework (used by ScyllaDB) with one thread per core, nothing shared, and its own TCP stack.
- "You lose all your normal tools": the kernel doesn't see the packets anymore, so `ss`, `tcpdump`, `iptables` and the normal socket API stop working for that traffic.

```
 normal:      app ⇄ syscall ⇄ kernel TCP/IP ⇄ driver ⇄ NIC
 bypass:      app + user-space TCP ⇄ NIC (mapped directly)      ← no syscalls per packet
 io_uring:    app ─▶ [submission ring] ─▶ kernel ─▶ [completion ring] ─▶ app
              (many ops per syscall, or zero syscalls with polling mode)
```

> **Remember `sendfile`**: it's why nginx serves static files so efficiently (block 05).

**Gotcha:** "context switch" is used for two different things: (a) switching between user mode and kernel mode on a syscall, and (b) the scheduler switching from one thread to another. Thread-per-connection designs pay a lot of (b). Event loops reduce (b) but still pay (a) on every `read`/`write`.

### 4.4 Wall 4 · The queue: concurrency isn't a setting, it's throughput × latency

**Idea:** you don't *choose* how many connections you hold at once. It's decided by how many requests arrive per second, multiplied by how long each one takes.

```
   connections in flight   =   requests per second   ×   seconds per request
          L                =           λ             ×           W
```

🧠 This is **Little's Law** (named in homework #4): **L = λ × W**. The average number of things inside a system = how fast they arrive × how long each one stays. It works for any steady queue: supermarket checkouts, servers, anything.

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

> 🧠 The slide's x-axis is only labelled 1–5. The latencies above are calculated from L = 10,000 × W (100 → 10 ms, 500 → 50 ms, 1000 → 100 ms, 5000 → 500 ms, 10000 → 1 s).

**Ten times the cost (slide):** at **100 ms** per request you hold **1,000** connections. At **1 second**, **10,000**, *for the exact same traffic*.

**Slowness is a memory bug (slide):** every open connection holds an fd, a buffer, and maybe a thread stack (walls 1 and 2). So a slower backend → more open connections → more memory. That's why **slow work should be done asynchronously**: take it off the request path and the queue gets shorter.

🧠 Example: a signup endpoint sends a welcome email before replying, so it takes 800 ms. Put "send email" on a message queue (Kafka, RabbitMQ) instead, and the endpoint replies in 30 ms. At 1,000 requests/second, open connections drop from 800 to 30.

**Gotcha 🧠:** a slow **dependency** (say a database that went from 5 ms to 500 ms) silently multiplies your connection count by 100, even though traffic didn't change. The server then hits the fd or memory wall and crashes. This is how "it got slow" turns into "it went down", and it's why timeouts (1.5) matter.

---

## 5 · IN THE WILD: what do popular servers actually do?

### 5.1 Apache: prefork → worker → event

> It took twenty years, and Apache ended up with a loop too.

**MPM** = Multi-Processing Module (the slide says "Multi process module"). It's the part of Apache that decides how it handles many connections, and you can swap it.

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

**The keep-alive problem in one line (slide):** with **worker**, a client that has **finished its request but keeps the connection open still owns a whole thread**. **Event takes the idle socket back and frees the thread for someone else.** That's our select loop, arriving inside Apache in **2012**. (🧠 Event MPM became stable in Apache 2.4, in 2012.)

🧠 **Keep-alive** = HTTP/1.1 persistent connections. After a response, the TCP connection stays open (often 5–15 s) for the next request, to avoid a new handshake. Most of that time the connection is **idle**, and tying a thread to idle time is pure waste.

```
 WORKER:  thread T1 ══[req]══[ idle keep-alive 15 s .......... ]══[req]══
 EVENT:   thread T1 ══[req]══ (free → serves others)            T4 ══[req]══
          listener + epoll holds the idle socket in the meantime ──────▲
```

> 💡 **Simple words mein:** Worker MPM = customer ne khana kha liya, par table pe baitha hai, aur waiter uske paas khada hai, kuch nahi kar raha. Event MPM = waiter chala jaata hai doosre customers ko serve karne, aur ek "doorman" (listener + epoll) dekhta rehta hai ki ye customer dobara kab order karega.

### 5.2 HAProxy: a select loop from before epoll existed

| The timeline explains the design | It introduced the load balancer |
|---|---|
| HAProxy came out in **2001**. **epoll only arrived in Linux 2.5.44 in late 2002.** So it mostly used **select**, because that's all there was. It moved to epoll later, and only became **multi-threaded in version 1.8, in 2017**. | For many teams this was **the first machine that sat in front of the app servers and chose which one to send each request to**. It could switch between several backend services. |

| One process, huge throughput | It also absorbs connection churn |
|---|---|
| For most of its life, **a single event-driven process could fill the whole network card**. *That's the strongest possible argument for the loop.* | It can **reduce how often backend servers have to connect and disconnect by keeping a pool of connections open**. That's the persistent-connection idea from block 03, done by a proxy for you. |

```
 thousands of short client conns            few long-lived upstream conns
 c1 ─┐                                      ┌──── app1
 c2 ─┼──▶  [ HAProxy: 1 event loop ]  ══════╪──── app2
 c3 ─┘     picks a backend (round-robin,    └──── app3
 ...        least-conn, ...), reuses conns
```

🧠 "Filled the network card" (saturated the NIC) means it sent as much traffic as the network card could carry (e.g., 10 Gbit/s) from a single-threaded process. The CPU wasn't the bottleneck; the cable was.

### 5.3 Varnish: the first popular caching reverse proxy

| Squid cached for the users | How Varnish runs |
|---|---|
| **Squid** was a caching proxy that sat **near the users, on the way out**. **Varnish flipped it** and put the cache **in front of the website's server, on the way in**. Same mechanism, opposite direction, completely different business. | An **acceptor thread for each listening point in each pool**. A **worker thread for each active connection**, taken from pools with a **minimum of 100 and maximum of 5000** by default. **Idle connections** (clients between requests, backends between fetches) are handed to a **waiter**, which is **epoll on Linux and kqueue on BSD**. |

**Still everywhere:** **Fastly's edge is built on Varnish, including VCL.** When you write a **VCL** snippet for a CDN in 2026, you're configuring a program written in **2006**.

> **So: threads do the work, a loop does the waiting.** The same split Apache reached with the event MPM.

🧠 What these words mean:
- **Origin** = your real backend server, where the content actually comes from.
- **VCL** = Varnish Configuration Language: a small language that decides, for each request, whether to cache it, what to use as the cache key, and which backend to use. It's compiled to C.
- **CDN** = Content Delivery Network (Week 6).

```
  Squid (forward):   [users + Squid] ───────────▶ internet sites       (cache for many sites, few users)
  Varnish (reverse): users ───────────▶ [Varnish + YOUR origin]        (cache for one site, many users)
```

### 5.4 nginx: epoll plus sendfile, never copy the bytes at all

The slide's code comparison:

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

| Master and workers | The first reverse proxy | A thread pool, added later |
|---|---|---|
| **One master** process, then **one worker process per CPU core**. Each worker is a **single thread running one epoll loop over thousands of connections**. | It was quickly **used for caching, load balancing, TLS termination, rate limiting**, and much more. **It became the default shape of an edge server.** | **Disk reads can block even inside a loop.** Since version **1.7.11**, nginx sends those to a **thread pool**, because **waiting on a slow disk read freezes every connection on that worker**. |

🧠 **Why is disk special?** epoll tells you when a **socket** is ready. But regular files always say "ready", and a `read()` of a file that isn't already in memory (the page cache) **blocks** while the disk works. In a single-threaded loop, that freezes thousands of other connections. So the blocking work goes to helper threads (`aio threads;` in the nginx config).

🧠 **The master's job:** read the config, open the ports, and start or restart workers (graceful reload with `nginx -s reload`). The workers handle all the actual traffic. **TLS termination** = nginx decrypts HTTPS so the backend servers can use plain HTTP.

### 5.5 All the options side by side

> Four teams, four starting points, thirty years apart.

| | How it handles many connections | Born as | The idea it made popular | Where you see it |
|---|---|---|---|---|
| **Apache** | prefork → worker → event (listener thread + epoll) | **1995**, the web server | Modules, and CGI as a way to run anyone's code | Everywhere. Akamai started here |
| **HAProxy** | select, then epoll; multi-threaded only since 1.8 | **2001**, TCP/HTTP proxy | The load balancer as its own layer | The default load balancer in front of everything |
| **Varnish** | thread per active connection + epoll waiter for idle ones | **2006**, HTTP accelerator | Caching reverse proxy: Squid, flipped around | Fastly's edge, VCL and all |
| **nginx** | master + one worker per core, one epoll loop each | **2004**, answer to C10K | sendfile: never copy the bytes into user space | The default edge. Broadpeak, and most CDNs |

> **Read the second column top to bottom: four separate teams, four starting points, and every one ended up with an event loop and a small pool of workers.**

### 5.6 Go read the actual code

> Every claim in this block is a file you can open tonight.

| Repo | File | What's there |
|---|---|---|
| `apache/httpd` | `modules/generators/mod_cgi.c` L576 | `run_cgi_child()`: fork, three pipes, a timeout on each |
| `apache/httpd` | `server/util_script.c` L409 | `ap_add_cgi_vars()`: HTTP headers become environment variables |
| `nginx/nginx` | `src/event/modules/ngx_epoll_module.c` | The epoll loop itself |
| `nginx/nginx` | `src/os/unix/ngx_linux_sendfile_chain.c` | `sendfile()`, where the copy doesn't happen |
| `haproxy/haproxy` | `src/ev_epoll.c` | The poller, one file per platform (`ev_select.c` is still next to it) |
| `varnishcache/varnish-cache` | `bin/varnishd/waiter/cache_waiter_epoll.c` | The waiter that holds idle connections |
| `golang/go` | `src/runtime/netpoll_epoll.go` | Where your blocking read actually goes to sleep |

> *Pick one.* Reading one poller from start to finish takes about two hours, and it changes how you understand every server after that. *They all look the same once you've seen one.*

🧠 Tip: line numbers change between versions. If L576 doesn't land on `run_cgi_child`, search the file for the function name (on GitHub press `t` to search files, or `y` for a permanent link).

---

## 6 · HANDS-ON 🧪

> ⚠️ 🧠 You're on **macOS**: `epoll`, `ss`, `strace` and `/proc` only exist on **Linux**. Use a Linux VM, a cloud machine, or Docker (`docker run -it --rm --ulimit nofile=200000 ubuntu bash`, then `apt update && apt install -y build-essential netcat-openbsd iproute2 strace tcpdump curl`). macOS alternatives are given where they exist.

### 6.1 Run the fork server and watch processes appear

```sh
gcc fork_server.c -o fork_server && ./fork_server &
nc localhost 2026        # in 2–3 terminals; type lines, they echo back
ps -ef | grep fork_server   # one parent + one child per open nc
```
**What you should see:** N `nc` sessions → N+1 `fork_server` processes. Close an `nc` and its child becomes `<defunct>` (a zombie, because the code never calls `wait()`).

### 6.2 Run the select server and confirm it's ONE process

```sh
gcc select_server.c -o select_server && ./select_server &
nc localhost 2026        # several terminals, all work at once
ps -L -p $(pgrep select_server)    # Linux: one process, ONE thread (LWP)
lsof -p $(pgrep select_server) | grep TCP   # one fd per client + the listener
```
**What you should see:** one process and one thread, but one extra fd for each client.

### 6.3 Check your limits

```sh
ulimit -n        # soft fd limit  → 1024 on Linux, often 256 on macOS
ulimit -Hn       # hard fd limit
ulimit -s        # thread stack size in KB → 8192 (= 8 MB) on most Linux
ulimit -n 65535  # raise the soft limit for this shell (up to the hard limit)
cat /proc/sys/fs/file-max   # Linux system-wide cap
```

### 6.4 See the cost of syscalls

```sh
strace -c -f ./select_server     # Linux: run some nc traffic, then Ctrl-C
```
**What you should see:** a table counting `select`, `read`, `write`, `accept` calls and the time spent in each. One `select` + one `read` + one `write` per echoed message is the "kernel tax" from wall 3.

### 6.5 Count open connections

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

### 6.7 Try CGI in 30 seconds (no Apache needed) 🧠

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
**What you should see:** the `pid=` number is **different every time**. That proves a new process is started for every request. You can also see `HTTP_USER_AGENT=curl/...`, which is the header-to-variable conversion.

### 6.8 See sendfile in action

```sh
strace -f -e trace=sendfile,read,write -p $(pgrep -f 'nginx: worker')   # Linux, while curling a static file
```
**What you should see:** `sendfile(...)` calls instead of `read` + `write` pairs (needs `sendfile on;` in nginx.conf, which is the default in most distro configs).

---

## 7 · HOMEWORK (from the slides)

> Four tasks. Do **at least the first two**. They're the ones you'll learn the most from.

### HW 1 · Break the select server
> Take the code from slide 05 and open **1,100 connections** to it. Find where it breaks and prove to yourself why. Then rewrite the same loop with **epoll** and do it again with **100,000**.

(🧠 The select code is on slide 04; slide 05 is the FD_SETSIZE slide that explains it. It's the same server.)

**How to approach it**
1. **Create the connections** with a small client that just opens sockets and keeps them open:
   ```python
   import socket, time
   socks = []
   for i in range(1100):
       s = socket.create_connection(("127.0.0.1", 2026))
       socks.append(s)
       if i % 100 == 0: print(i)
   time.sleep(3600)
   ```
   The *client* also needs `ulimit -n` > 1100 in its own terminal.
2. **Predict three possible breaking points**, then find which one you hit first:
   - **Server `ulimit -n` = 1024**: fds 0–2 + the listener use 4, so after about **1020 clients**, `accept()` returns **-1 (`EMFILE`)**. The code doesn't check this, so it stores `-1` in `clients[]`, and `FD_SET(-1, ...)` is undefined behaviour. Also the listener keeps saying "readable" forever, so the loop spins at 100% CPU.
   - **Raise the server's ulimit** (`ulimit -n 4096`) and try again. Now fds reach **1024**, and `FD_SET(1024+, ...)` writes past the end of the 128-byte `fd_set`: memory corruption, strange behaviour, or a crash.
   - **`clients[FD_SETSIZE]` overflows** at the 1025th client: `nclients++` has no bounds check, so it writes past the end of the array.
3. **Prove it:** use `strace -e accept,select ./select_server` to see `EMFILE`, `ls /proc/<pid>/fd | wc -l` to count fds, and print each `client_fd` value as you accept it. Try building with `-fsanitize=address` or `-D_FORTIFY_SOURCE=2 -O2` (then glibc can stop the program on `FD_SET` with fd ≥ 1024, showing "buffer overflow detected").
4. **epoll rewrite:** `epoll_create1`, `epoll_ctl(ADD)` when you accept, an `epoll_wait` loop, and close on `read <= 0` (see the skeleton in 1.3). You don't need the fixed array anymore, because the kernel keeps the list.
5. **For 100,000:** raise `ulimit -n` (and the hard limit / `fs.file-max`) on both sides. One client IP connecting to one server port runs out of **ephemeral ports** (about 28k by default, `net.ipv4.ip_local_port_range`), the same problem from Lesson 1. Use several client source IPs (`127.0.0.2`, `127.0.0.3`… all work on Linux loopback) or several server ports. Watch memory with `free -m` and `ss -s`.

### HW 2 · See how much CGI costs
> Write a CGI script in any language, run it under Apache with **mod_cgi**, and keep `ps` or `top` open on a second screen while you `curl` it in a loop. Count the processes appearing and disappearing.

**How to approach it**
- Install Apache (`apt install apache2`, then `a2enmod cgi` on Debian/Ubuntu; or Homebrew `httpd` on macOS and uncomment `LoadModule cgi_module` / `cgid_module` in `httpd.conf`). Put the script in the `cgi-bin` folder (`/usr/lib/cgi-bin/` on Ubuntu) and `chmod +x` it.
- Make the script a bit slow (`sleep 0.2`) so the processes stay visible long enough to notice.
- Loop: `while true; do curl -s localhost/cgi-bin/hello.cgi > /dev/null; done` (run 2–3 loops in parallel).
- Watch: `watch -n 0.2 'ps -ef | grep hello.cgi | grep -v grep'`, or `top` sorted by PID. Print `$$` in the script and notice the PID goes up on every request.
- Measure: compare `ab -n 1000 -c 10` (ApacheBench) or `time` for the CGI script vs. a static file of the same size. The difference is the fork + exec cost.
- 🧠 With threaded MPMs, Apache uses **mod_cgid** (a separate helper process does the forking) instead of mod_cgi, but you'll still see one new process per request.

### HW 3 · Find the FastCGI records
> Run **php-fpm** behind **nginx**. Use `tcpdump` on the loopback connection between them and find **`FCGI_BEGIN_REQUEST`** and the **empty PARAMS record** in the bytes.

**How to approach it**
- Make php-fpm listen on **TCP** (`listen = 127.0.0.1:9000` in the pool config, e.g., `www.conf`). tcpdump can't see a Unix socket.
- nginx: `fastcgi_pass 127.0.0.1:9000;` (see the config in 2.6). A `test.php` with `<?php echo "hi"; ?>`.
- Capture: `sudo tcpdump -i lo -X -s0 'tcp port 9000'`, then `curl -d 'name=alice&age=20' 'localhost/test.php?x=10'`. Or save with `-w fcgi.pcap` and open it in **Wireshark**, which understands FastCGI (Decode As → FCGI).
- What to look for (the 8-byte header from 2.7):
  - `01 01 00 01 00 08 00 00` → version 1, **type 1 = BEGIN_REQUEST**, requestId 1, length 8.
  - `01 04 00 01 xx xx ..` → PARAMS with content (you'll see `REQUEST_METHOD`, `POST`… in the text column).
  - `01 04 00 01 00 00 00 00` → **the empty PARAMS record** (length 0 = "params finished").
  - `01 05 00 01 00 11 ..` → STDIN with length `0x11` = **17** = `name=alice&age=20` ✓, then `01 05 00 01 00 00` (empty STDIN).
  - Response direction: `01 06` (STDOUT), `01 03` (END_REQUEST).

### HW 4 · Test Little's Law on your own service
> Pick a service. Note its **requests per second** and its **p50 latency**, predict the **number of open connections**, then check with **`ss -s`**. Then **slow it down on purpose** and see if the prediction still holds.

**How to approach it**
- 🧠 **p50** = median latency: half the requests are faster than this.
- A simple service: a tiny Flask/Go/Node app with `sleep(X)` in the handler, so you control W.
- Use a load generator at a *fixed rate* (not "as fast as possible"): `wrk2 -R 500`, `hey -q`, `vegeta attack -rate=500/s`, or a Python script. Fixed rate keeps λ constant.
- Predict: L = λ × W. E.g., 500 req/s × 0.050 s = **25** connections. Check with `ss -tn state established '( sport = :8080 )' | wc -l` (or `ss -s`) while the load is running.
- Now change the sleep to 200 ms. New prediction: 500 × 0.2 = **100**. Measure again.
- Gotchas: idle keep-alive connections still show up in `ss`. Use the right latency (time spent *inside* the server). And if the server hits a thread or worker limit, requests start waiting, W grows, and you can watch the problem snowball.

---

## 8 · THE FOUR IDEAS (closing slide): "The loop is the architecture"

| # | Idea | In one line |
|---|---|---|
| 01 | **Do the setup once** | Pre-fork, pre-thread, FastCGI, connection pools, servlets: all the same trick. *Take the expensive setup off the request path and do it once at startup.* |
| 02 | **Nothing is free per connection** | An fd, a stack, a buffer, a scheduler entry. *Multiply by ten thousand before you pick a design.* That multiplication is the whole skill. |
| 03 | **Framing is still length or delimiter** | FastCGI uses a length inside each record and an empty record to end a stream. Same two options as SMS and SMTP. *There's never a third.* |
| 04 | **Concurrency = throughput × latency** | You can't configure your way around this. *Getting faster is the only way to reduce how much you're holding.* |

> **Next session:** we start at the **load balancer** and go looking for **state**.

---

## 9 · CHEAT-SHEET 📋

| Term | One-line meaning |
|---|---|
| **fork per request** | Copy the whole process for each client: isolated, simple, most expensive |
| **thread per request** | New thread for each client: cheaper, but each still needs a stack (8 MB reserved) |
| **pre-fork / pre-thread** | Create the worker pool at startup so creation cost isn't paid while users wait |
| **event loop** | One thread asks the kernel "who's ready?" and only serves those |
| **`select()`** | Readiness API with a fixed 1024-bit `fd_set`; rebuild every call; O(n) |
| **`FD_SETSIZE`** | 1024: the largest fd *number* select can handle (built in) |
| **`FD_ZERO / FD_SET / FD_ISSET`** | Clear the set / add an fd / check an fd after select |
| **`epoll`** | Linux readiness API: register once (`epoll_ctl`), get only ready fds (`epoll_wait`); O(ready) |
| **kqueue / IOCP / io_uring** | Same idea on BSD+macOS / Windows / modern Linux (batched rings) |
| **`SO_REUSEPORT`** | Many sockets listen on one port; the kernel spreads new connections across workers |
| **timeout (select's 5th argument)** | `NULL` = wait forever, so one stuck client freezes the loop |
| **retry with backoff** | Client sets a deadline and retries with growing (random-jittered) waits |
| **CGI** | Fork + exec your program per request; headers → env vars, body → stdin, stdout → client, stderr → log |
| **`HTTP_USER_AGENT`** | Header naming rule: UPPERCASE, `-`→`_`, add `HTTP_` in front |
| **`REQUEST_METHOD` / `QUERY_STRING` / `CONTENT_LENGTH`** | Main CGI variables: verb / text after `?` / body size |
| **blank line** | The delimiter between CGI/HTTP headers and the body |
| **mod_php / mod_perl** | Language built into Apache: fast, but one crash kills both |
| **FastCGI** | Long-running app process; web server talks to it over a Unix/TCP socket with labelled binary records |
| **php-fpm** | PHP's FastCGI process manager: a pre-created pool of PHP workers |
| **upstream** | The internal server(s) a front proxy forwards requests to |
| **FCGI_BEGIN_REQUEST / PARAMS / STDIN / STDOUT / STDERR / END_REQUEST** | FastCGI record types; an empty record ends a stream |
| **servlet (Tomcat/Jetty)** | The JVM is the web server; pre-threaded pool; separated by class loader, not process |
| **stateless (HTTP)** | Each request carries everything it needs, so any server can answer and adding servers is easy |
| **persistent connection** | DBs/MQTT/Kafka keep connections open to avoid connect + handshake + login on every query |
| **RESP / MQTT / Kafka protocol** | Non-HTTP protocols for Redis / IoT pub-sub / Kafka |
| **`ulimit -n`** | Max fds per process; default soft limit still 1024 (`EMFILE` when you hit it) |
| **C10K / C10M** | 10k open connections (Kegel, 1999) / 10M by skipping the kernel (Graham, 2013) |
| **8 MB vs 2 KB** | OS thread default stack vs. goroutine starting stack (~4000×) |
| **netpoller** | Go runtime's epoll-based poller that puts goroutines to sleep on blocking reads |
| **kernel boundary tax** | Every syscall costs hundreds of ns; worse after Spectre/Meltdown |
| **DPDK / netmap / mTCP / Seastar** | Ways to skip the kernel and do networking in user space |
| **`sendfile()`** | Kernel copies file → socket directly; your program never touches the data |
| **Little's Law** | open connections = requests/s × seconds/request (L = λW) |
| **"slowness is a memory bug"** | Higher latency → more open connections → more memory |
| **Apache MPMs** | prefork (process per conn) → worker (thread per conn) → event (listener + epoll for idle) |
| **keep-alive problem** | An idle open connection holding a thread; event MPM fixes it |
| **HAProxy** | 2001; select then epoll; made the load balancer its own layer; multi-threaded since 1.8 (2017) |
| **Varnish** | 2006 caching reverse proxy; worker threads (100–5000) + epoll/kqueue waiter for idle ones |
| **Squid** | Forward caching proxy, near the users |
| **VCL** | Varnish Configuration Language (still used at Fastly) |
| **nginx** | 2004; master + one single-threaded epoll worker per core; sendfile; thread pool for disk since 1.7.11 |

**Key numbers:** `1024` (FD_SETSIZE = ulimit = "K") · `2026` (our port) · `8 MB` / `2 KB` · `80 GB` (10k threads) · `Linux 2.5.44` (epoll, 2002) · `1999 / 2013` (C10K / C10M) · `2–3 M` (today) · `100 / 5000` (Varnish pools) · `1.8` (HAProxy multi-threaded) · `1.7.11` (nginx thread pool) · `17` (bytes in `name=alice&age=20`).

---

## 10 · SELF-CHECK QUESTIONS ❓

**1. Name the three ways to serve many clients from the lecture. Which one scales?**
<details><summary>Answer</summary>

Fork per request (a process per client), thread per request (a thread per client), and one loop with many sockets (select/epoll). The loop scales. In real servers you combine them: a pool of processes/threads created at startup, and each one runs its own epoll loop.
</details>

**2. In the fork server, why does the parent call `close(client_fd)` right after `fork()`? What happens if it doesn't?**
<details><summary>Answer</summary>

After `fork()`, both parent and child have a copy of the fd. The TCP connection only closes when every copy is closed. If the parent keeps its copy, it leaks one fd per client. Connections never fully close when the child exits, and soon the parent hits the `ulimit -n` (1024) limit and `accept()` fails with `EMFILE`.
</details>

**3. Why does the select loop rebuild the `fd_set` (`FD_ZERO` + `FD_SET`) every time?**
<details><summary>Answer</summary>

`select()` changes the set. When it returns, only the ready fds are left in it. To watch all clients again next time, you have to rebuild it. This is also part of why select is O(n): you give it all n fds every time, and the kernel checks all of them every time.
</details>

**4. Explain `clients[i] = clients[--nclients]; i--;`. Why is `i--` needed?**
<details><summary>Answer</summary>

It's a fast (O(1)) delete: move the last element into the empty spot and reduce the count, instead of shifting the whole array. `i--` is needed because a different fd, not yet checked, is now at index `i`. Without it, the loop's `i++` would skip that fd this round.
</details>

**5. What is FD_SETSIZE, why is it "built in", and does it limit the *number* of fds or their *values*?**
<details><summary>Answer</summary>

`fd_set` is a fixed-size bitmap, 1024 bits on glibc, so `FD_SETSIZE = 1024`. The size is fixed when the program is compiled. It limits the fd's **value (number)**: any fd numbered 1024 or higher can't be stored, even if you only watch a few sockets. Using one is undefined behaviour (it writes past the end of the bitmap).
</details>

**6. Why is epoll "O(ready)" while select is "O(n)"? When does that difference matter most?**
<details><summary>Answer</summary>

With select, you give the kernel all n fds every time, and it checks all of them. With epoll, you register each fd once (`epoll_ctl`), the kernel remembers the list, and `epoll_wait` gives back only the fds that are ready. So the work depends on how many are ready, not how many there are in total. It matters most with many mostly-idle connections (keep-alive, chat, IoT): 10,000 idle sockets cost select as much as 10,000 busy ones, but cost epoll almost nothing.
</details>

**7. What does `NULL` as select's last argument mean, and why is it dangerous? What's the matching rule for clients?**
<details><summary>Answer</summary>

The last argument is the timeout. `NULL` means wait forever. If something gets stuck, the whole loop and all its connections stop, and you never get a chance to do cleanup like closing idle clients. Client rule: always set a deadline, then retry with backoff (exponential with random jitter, and only for safe/idempotent operations).
</details>

**8. Walk through how Apache runs a CGI script for `POST /f.cgi?x=10` with body `name=alice&age=20` and `User-Agent: curl`. Where does each part of the request go, and where does the output go?**
<details><summary>Answer</summary>

Apache creates three pipes and does fork + exec on the script (`run_cgi_child()` in `mod_cgi.c`). The method, query and headers become environment variables (`ap_add_cgi_vars()`): `REQUEST_METHOD=POST`, `QUERY_STRING=x=10`, `HTTP_USER_AGENT=curl`, `CONTENT_LENGTH=17`. The body `name=alice&age=20` is written to the script's **stdin**. The script prints headers, a blank line and the body to **stdout**. Apache reads the headers up to the blank line and sends the rest to the client. Anything on **stderr** goes to Apache's error log. Each pipe has a timeout, so a stuck script can't hold the worker forever.
</details>

**9. CGI is slow. Compare the two fixes (mod_php vs. FastCGI). What does each give up?**
<details><summary>Answer</summary>

mod_php/mod_perl build the language into the web server, so there's no new process per request. But the app shares memory with the web server, so one crash kills both, and it forces you to use non-thread-safe setups (prefork). FastCGI keeps a separate pool of long-running app processes (php-fpm), and the web server talks to them over a Unix or TCP socket with a compact binary protocol. You get speed (no fork per request) **and** isolation. The front server also handles slow clients and TLS, while a small fixed pool of workers does the real work.
</details>

**10. How does FastCGI use *both* framing methods? Why is an empty record a better end marker than a byte pattern?**
<details><summary>Answer</summary>

**Length:** each record's 8-byte header says how long the content is, so the body can contain any byte (newlines, null bytes). `CONTENT_LENGTH=17` does the same one level higher. **Delimiter:** a record of length zero ends a stream (empty PARAMS = params done, empty STDIN = body done, empty STDOUT = output done), which lets you send data when you don't know the total size in advance. An empty record is part of the **structure**, so it can never be confused with content and doesn't need escaping. A byte pattern like SMTP's lone `.` has to be escaped when it appears in the data (dot-stuffing).
</details>

**11. Why is it easy to add more HTTP servers, but scaling a database "changes shape"?**
<details><summary>Answer</summary>

HTTP is stateless: each request carries everything it needs, so any server can answer it, connections can be closed anytime, and a load balancer can send the next request anywhere. Adding a server to the pool is enough. Databases (Redis, MySQL) want long-lived connections, because each new one costs a TCP connect + handshake + login + session setup. So the question changes from "requests per second" to "how many open connections can one machine hold?", which runs into the fd and memory walls.
</details>

**12. A service handles 2,000 req/s at 50 ms. Its database slows down, so requests now take 500 ms. Traffic is the same. What happens to open connections, and why does the lecture call slowness "a memory bug"? What's the fix?**
<details><summary>Answer</summary>

Little's Law: L = λ × W. Before: 2,000 × 0.05 = **100** connections. After: 2,000 × 0.5 = **1,000** connections, ten times more for the same traffic. Each open connection holds an fd, buffers and maybe a thread stack (8 MB reserved), so memory and fd usage grow 10×, and you might hit `ulimit -n` (1024) or run out of memory. Fixes: get faster (the only real fix), or move slow work off the request path (async, e.g., a message queue) so the line gets shorter. Timeouts also limit how long a request can hold resources.
</details>

**13. Apache's worker MPM vs. event MPM: what is "the keep-alive problem", and how does event fix it?**
<details><summary>Answer</summary>

With worker, each connection is tied to a thread, even when the client has finished its request and is just keeping the connection open, doing nothing. Threads (and their stacks) are wasted on idle time. Event MPM adds a listener thread per process that holds idle keep-alive connections on epoll, and only gives a worker thread to connections that actually have work. The thread goes back to the pool in between. That's the select loop arriving inside Apache (2012).
</details>

**14. Why does nginx use `sendfile()`, and why did it still need a thread pool (since 1.7.11) even though it's event-driven?**
<details><summary>Answer</summary>

`read()` + `write()` copies file data disk → kernel → your buffer → kernel → socket: two syscalls and two copies through user space. `sendfile(sock, file, NULL, n)` sends data from kernel to socket directly, with no trip through your program, so fewer syscalls and copies (wall #3). But disk reads can still block inside an event loop. If the file isn't already in memory, the read waits for the disk, and the single-threaded worker freezes along with every connection on it. So nginx sends those blocking disk reads to a thread pool.
</details>

**15. Design question: every server in block 05 "ended up with an event loop and a small pool of workers." Why does that combination keep winning?**
<details><summary>Answer</summary>

The loop makes **waiting** cheap: idle or slow connections cost only an fd and a small buffer in epoll/kqueue, not a whole thread or process. The small pool gives **work capacity and isolation**: about one worker per CPU core uses all the CPUs without too much switching, separate processes protect against crashes, and threads can take blocking work (disk, heavy CPU work) without freezing the loop. Creating the pool at startup means you pay setup costs only once. It brings together all four closing ideas: do setup once, nothing is free per connection, and concurrency = throughput × latency (so holding many connections must be cheap).
</details>

---

*All slides could be read clearly. The chart on slide 24 had no x-axis labels; the latencies in 4.4 are calculated from L = λW. The CGI diagram on slide 08 was checked against the PDF image.*
