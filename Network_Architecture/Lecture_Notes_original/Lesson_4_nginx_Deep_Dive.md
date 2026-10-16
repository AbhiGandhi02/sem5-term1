# 📓 nginx, and the code that explains it
### *CN at Scaler — Lesson 4* · "it did not make the worker cheaper. It deleted the worker."

> `4 hrs` · `36 slides` · `nginx 1.31.5` · source: `github.com/nginx/nginx` · companion repo: `nginx-from-scratch`
> **history → what nginx changed → the features you configure → a walk through the source**

**Legend for these notes:** 🧠 = my own addition, *not on the slides* — background needed to actually follow the material.

---

## ⭐ What this lecture is about (and why it matters)

Every web request you have ever made almost certainly passed through something like nginx. This lecture tells the story of **how web servers handle many connections at once**, from forking one process per client (1990s) to nginx's design of **one process per CPU core, each running an event loop** that juggles thousands of connections. Then it looks at the features you actually type into `nginx.conf` (caching, `location` routing, load balancing, `try_files`) and shows, for each one, **the few lines of C in the nginx source that implement it**. The aim is that nginx stops being a box of magic directives: you should be able to *predict* what it does because you know the data structure underneath.

```
  The whole lecture in one picture
  ────────────────────────────────
  1995  Apache prefork   ─┐
  1999  Apache worker    ─┼── "make the per-connection worker CHEAPER"
  2012  Apache event     ─┘
  1997  Servlets / J2EE  ──── "stop CREATING things per request"
  2004  nginx            ──── "why is there a worker per connection AT ALL?"
                                    │
                    event loop + epoll + sendfile + reverse proxy
```

**Roadmap (slide 2):**

| part | title | what's inside |
|---|---|---|
| 01 | History, compressed | who wrote the first web server · Apache outrunning `fork()` · J2EE's answer · why 2002 needed something new |
| 02 | What nginx changed | it did not make the worker cheaper, it deleted it · event loop · epoll · sendfile · reverse proxy |
| 03 | The features you configure | caching · routing precedence · upstreams & weights · fallbacks — each next to its code |
| 04 | The walkthrough | ten stops in the nginx source · then seven servers, from `fork()` to FastCGI |

---

## 0 · 🧠 PREREQUISITES FROM SCRATCH

The slides assume all of the following. If any of it is shaky, read this section first.

### 0.1 Process vs thread

```
  PROCESS                                   THREADS (inside one process)
  ┌──────────────────────────┐              ┌──────────────────────────────┐
  │ own memory (address space)│              │ ONE shared memory space       │
  │ own file-descriptor table │              │ shared fd table               │
  │ one thread of execution   │              │ ┌──────┐ ┌──────┐ ┌──────┐    │
  │                          │              │ │stack1│ │stack2│ │stack3│    │
  └──────────────────────────┘              │ └──────┘ └──────┘ └──────┘    │
   created with fork()                      └──────────────────────────────┘
                                             created with pthread_create()
```

- A **process** is a running program with its own private memory. `fork()` makes a copy of the current process (the child). Expensive-ish: new page tables, new kernel bookkeeping.
- A **thread** is a separate flow of execution *inside* a process. Threads share memory, but **each thread needs its own stack** (the memory where function calls keep local variables). On Linux the default stack reservation is **8 MB** (`ulimit -s` shows `8192` KB). This number shows up again and again in the lecture.
- **Context switch** = the kernel stops one thread/process and starts another. It costs CPU time (saving registers, switching page tables for processes, cache pollution). With 10,000 threads, a lot of CPU goes into switching rather than working.
- **Reentrant / thread-safe** code = code that works correctly when two threads run it at the same time. Old C library functions (like `strtok`, `gethostbyname`) kept hidden global state and were *not* safe. That's the "everyone is learning what reentrant means" line on slide 6.

### 0.2 File descriptors (fds)

A **file descriptor** is a small integer the kernel gives you when you open a file *or a socket*. It's an index into your process's open-file table. `0, 1, 2` are stdin/stdout/stderr, so your first socket is usually `3`.

- `ulimit -n` = the maximum number of fds one process may have open (often 256 on macOS, 1024 on Linux by default, raisable to 65536+).
- Every TCP connection a server holds is **one fd**. 10,000 connections = 10,000 fds.

### 0.3 Blocking vs non-blocking I/O

```
  BLOCKING read()                       NON-BLOCKING read()
  ───────────────                       ───────────────────
  no data yet? → thread SLEEPS          no data yet? → returns -1 at once,
  until bytes arrive                    errno = EAGAIN ("try later")
  → needs 1 thread per connection       → one thread can juggle many fds
```

If `read()` blocks, a thread that's waiting for client A can't serve client B. So either you have a thread per client, **or** you make sockets non-blocking and ask the kernel *"which of my sockets are ready right now?"*. That question is what `select()`, `poll()`, `epoll` and `kqueue` answer. This is called **I/O multiplexing**.

### 0.4 What an event loop is

```
  while (true) {
      events = wait_for_any_ready_fd(timeout);   // sleep here
      for each event in events:
          call the handler for that fd             // do a little work, never block
  }
```

An **event loop** is a single thread that sleeps until *something* happens (data arrived, a socket became writable, a timer fired), handles it quickly, and goes back to sleep. Analogy: one waiter serving 50 tables. He doesn't stand at one table while they read the menu; he goes wherever someone raises a hand.

The one rule: **a handler must never block.** If it does, every other connection on that loop freezes.

### 0.5 Kernel space, user space, mode switches, DMA

- Your program runs in **user space**. The kernel (which owns disks, network cards, memory) runs in **kernel space**.
- A **system call** (syscall) like `read()` or `write()` crosses from user to kernel and back. Each crossing is a **mode switch**, which costs CPU.
- **Page cache** = the kernel keeps recently read file data in RAM so the next read doesn't hit the disk.
- **DMA (Direct Memory Access)** = hardware (disk controller, network card / NIC) copies data to/from RAM *by itself*, without the CPU doing the copying. DMA copies are "free" for the CPU. Copies done by `memcpy` inside the kernel or in your program are **CPU copies**.

### 0.6 What a proxy is, and what a cache is

- A **proxy** is a middleman: client talks to the proxy, the proxy talks to the real server on the client's behalf.
- A **cache** stores a copy of something expensive to fetch, so the next request is answered from the copy. Key words:
  - **HIT** = the answer was in the cache. **MISS** = it wasn't; we had to go to the origin.
  - **TTL / freshness** = how long a cached copy is considered good.
  - **Stale** = a copy whose freshness has expired (may still be usable in emergencies).
  - **Eviction** = deleting things from the cache to make room.
  - **LRU (Least Recently Used)** = evict the item nobody has touched for the longest time.
- **Origin / upstream / backend** = the real application server behind the proxy.

### 0.7 Data structures that matter here

| structure | one-line meaning | where it shows up |
|---|---|---|
| **Doubly-linked list** | each node points to the next and previous | the LRU queue, posted-event queues |
| **Intrusive list** | the `next/prev` pointers live *inside* your struct, no separate node allocation | `ngx_queue.h` |
| **Red-black tree (rbtree)** | a self-balancing binary search tree, O(log n) lookup | timers, cache key index |
| **Ternary / binary search tree of prefixes** | a tree you walk by comparing strings left/right/down | `location` prefix matching |
| **Array** | a plain list, walked from start to end | `location` regexes |
| **State machine** | a variable `state` + a `switch` that says what to do with the next input byte in each state | the HTTP parser |
| **Memory pool (arena)** | allocate lots of small chunks from one big block; free the whole block at once | `ngx_palloc.h` |

### 0.8 CGI, FastCGI, and "application server"

- **CGI (Common Gateway Interface, 1993)**: to make a page dynamic, the web server **forks and execs a program** (a Perl/C script) for *every request*, passes request info in environment variables, and sends whatever the program prints back to the client. Simple, but very slow under load.
- **FastCGI**: same idea, but the program stays running and the web server sends it requests **over a socket** using a small binary protocol. No fork per request. PHP-FPM is the famous FastCGI server.
- **Application server**: the thing that runs your business logic (Tomcat, Gunicorn, Node, Puma, PHP-FPM).

### 0.9 Big-O, quickly

`O(n)` = work grows with n. `O(1)` = constant. The lecture's key comparison: `select` is **O(watched fds)**, `epoll` is **O(active fds)**.

---

## 1 · HISTORY, COMPRESSED 📜

### 1.1 "Remember the evolution. You can forget the dates." (slide 3)

The instructor's point: names and dates matter less than the **chronology**. Each design in the lecture was someone's answer to a constraint. If you know *which* constraint, you can tell whether a rule is a **law** (still true) or a **habit** (true in 2004, not any more).

> *"Understanding history is how you get to create a different timeline."*
> Sometimes you look at the same limitation and fix it a completely different way. (That's nginx's story.)

### 1.2 A correction worth making (slide 4)

| the myth you often hear | what actually happened |
|---|---|
| "Berners-Lee wrote the first server. It was called Mosaic. Mosaic became Apache." | Berners-Lee's server was **CERN httpd**, on a **NeXT** computer, **1990**. |
| | **Mosaic** (NCSA, 1993) was a **browser** (Marc Andreessen's), not a server. |
| | **Apache** descends from **NCSA HTTPd**, a *different* NCSA program. |

Three separate programs, two of them at NCSA (the National Center for Supercomputing Applications, University of Illinois 🧠), one of them a browser. Getting it right is "the difference between a story and a lineage".

### 1.3 The actual lineage (slide 5)

```
 1990     CERN httpd ........ Berners-Lee, on a NeXT. The first web server.
            │
 1993     NCSA Mosaic ....... a BROWSER. Inline images. The web goes mainstream.
 1993     NCSA HTTPd ........ Rob McCool. A SERVER — and it invents CGI.
            │
 1994     McCool leaves ..... development stalls. Webmasters trade patches by mail.
            │
 1995     Apache ............ that patch set, continued as a project. "A patchy server."
            │
 1996     Apache #1 ......... passes NCSA HTTPd. Stays #1 for twenty years.
 1996–99  IIS, then the boom  Microsoft fights for single-digit share. lighttpd & friends arrive.
```

- **CGI was invented by NCSA HTTPd.** Remember this: CGI's per-request `fork()+exec()` is the "expensive setup" that everyone later tries to remove.
- "A patchy server" is the famous pun behind the name Apache (patches traded by email, assembled into one server).
- 🧠 **IIS** = Microsoft Internet Information Services. **lighttpd** ("lighty", 2003 🧠) was one of the early event-driven servers aimed at the same C10K problem nginx targeted.

### 1.4 Apache spent a decade trying to outrun `fork()` (slide 6)

Apache's architectures are called **MPMs (Multi-Processing Modules)** 🧠 — pluggable strategies for "who handles which connection".

```
 Apache 1.3: PREFORK                 Apache 2.0: WORKER MPM              Apache 2.4: EVENT MPM
 ───────────────────                 ──────────────────────              ─────────────────────
  master                              master                              master
   ├─ proc ── conn A                   ├─ proc ─┬─ thread ── conn A         ├─ proc ─┬─ listener thread
   ├─ proc ── conn B                   │        ├─ thread ── conn B         │        │   (parks idle keep-alives
   ├─ proc ── conn C                   │        └─ thread ── conn C         │        │    on epoll)
   └─ proc ── (idle, waiting)          └─ proc ─┬─ thread ── conn D         │        ├─ worker thr ← only when
                                                └─ thread ── conn E         │        └─ worker thr   there's WORK
  pool forked at STARTUP,             processes running many threads      default on Unix since 2.4
  not per request                     8 MB stack → 8 MB shared + a
  1 process parks on 1 connection     stack each
```

| version | MPM | idea | remaining cost |
|---|---|---|---|
| **1.3** | **prefork** | Fork a pool of processes *at startup* instead of per request. The `fork()` cost moves **off the request path**. | One process still **parks on one connection** for its whole life, even when the client is idle. |
| **2.0** | **worker** | Processes each running many **threads**. Instead of a full process (8 MB of stack + memory) per connection, you get "8 MB shared plus a stack each". | 1999–2002: **pthreads on Linux were new and buggy**, and everyone was learning what *reentrant* means (lots of non-thread-safe libraries, e.g. PHP extensions 🧠). Still one thread per connection. |
| **2.4** | **event** | A **listener thread** parks idle **keep-alive** connections on **epoll** and hands a worker thread a connection **only when it has actual work**. Default on Unix since 2.4. | Still a thread per *active* request. |

🧠 **Keep-alive** = HTTP/1.1 keeps the TCP connection open after a response so the browser can send the next request without a new handshake. Great for clients, terrible for prefork/worker: an idle keep-alive connection pins a whole process/thread doing nothing.

> **The slide's punchline:** *"Three architectures, one question: how do I make the per-connection worker cheaper? Every answer was a good answer. None of them was a different question."*

**Pitfall:** "Apache is slow" is a lazy take. Apache 2.4's event MPM borrowed the same epoll trick. The difference is that Apache *started* from a worker-per-connection world and moved towards events; nginx started from events.

### 1.5 Java's answer: stop creating things (slide 7)

```c
// CGI, 1993
per request:   fork() + exec() + load interpreter
               + parse script + run + exit

// Servlet, 1997
at startup:    load class, instantiate once
per request:   service(req, res)    <- a method call

// the object outlives the request. that is the trick.
```

- **J2EE (Java 2 Enterprise Edition, 1999)** made the **multi-layer application server** the default shape: **web tier → EJB tier → database**. 🧠 *EJB = Enterprise JavaBeans, server-side business components.*
- A **servlet** is instantiated **once** and serves **every** request on a thread from a **thread pool**.
- **Connection pools** do the same thing for the **database handshake**: open N DB connections at startup, lend one to each request, return it after.
- Same move as prefork. Same move as FastCGI: **take the expensive setup off the request path and pay it once.**

> *"Our primary goal was to avoid creating a new process. Everything else was consequence."*

**Why this matters:** compare costs per request. CGI = create a process, load an interpreter, parse the script (milliseconds). Servlet = a method call (microseconds). That's orders of magnitude.

**Gotcha:** because the servlet object is shared by all request threads, instance variables in a servlet are **shared state** → race conditions if you store per-request data in them. 🧠

### 1.6 So here is 2002 — the C10K problem (slide 8)

| number | meaning |
|---|---|
| **10,000** | concurrent connections on one box. **Dan Kegel** named this the **C10K problem** in **1999**. |
| **8 MB** | stack per thread, the Linux default. **10,000 threads = 80 GB of address space.** |
| **1024** | descriptors, **hard-coded as `FD_SETSIZE`** in a header for `select()`. **`ulimit` does not move it.** |

Hardware was fast enough to serve 10,000 people from one machine; the software wasn't. The limits were:
1. a **per-connection stack** (memory),
2. a **per-connection scheduler entry** (context switches),
3. **a 1983 API with a bitmap in it** (`select()`, from 4.2BSD 🧠).

🧠 *Address space vs physical RAM:* 80 GB of **virtual** stack doesn't mean 80 GB of RAM is used (pages are only backed when touched), but on 32-bit systems of 2002 the whole address space was only 4 GB, so 10,000 × 8 MB simply didn't fit. And even touched memory, scheduler cost, and cache misses add up.

> *"Everyone is optimising the worker. Somebody is about to delete it."*

---

## 2 · WHAT NGINX CHANGED 🚀

### 2.1 Igor Sysoev, Rambler, 2002 (slide 9)

- **Rambler** was one of Russia's largest web portals. Its Apache servers were **falling over under connection volume, not request volume** (lots of slow/idle connections, not lots of work).
- Sysoev started writing nginx in **2002** to solve **C10K specifically**.
- **First public release: 4 October 2004**, chosen for the anniversary of **Sputnik** (launched 4 Oct 1957 🧠).
- **1.0.0 didn't arrive until April 2011.** Seven years of shipping in production before the version number said "ready".

> He did not set out to write a faster Apache. He set out to **serve ten thousand mostly-idle connections from one process, and let something else generate the dynamic content.**
> **That second half is why nginx is a reverse proxy and not an application server.**

### 2.2 The different question (slide 10)

> **Apache asked:** how do I make the worker cheaper?
> **nginx asked:** why is there a worker per connection at all?

A connection is not a thing that needs a thread. It needs **a few hundred bytes of state and a place in a list** until it has something to say. When it does, some CPU handles it, finishes, and moves on.

```
  THREAD-PER-CONNECTION                    EVENT LOOP (nginx)
  ─────────────────────                    ──────────────────
  conn 1 ── thread (8 MB stack, asleep)    worker (1 per core)
  conn 2 ── thread (8 MB stack, asleep)      │
  conn 3 ── thread (8 MB stack, WORKING)     ├─ epoll: [c1 c2 c3 ... c10000]
  ...                                        │          few hundred bytes each
  conn 10000 ── thread (asleep)              └─ wakes up → handles c3 → sleeps
```

> **One process per core. One event loop each. Ten thousand connections, and none of them owns anything.**

### 2.3 Process model: one process per core, and a master that never touches a connection (slide 11)

```
                         ┌───────────────┐  ┌───────────────┐       ┌───────────────┐
                         │   worker 0    │  │   worker 1    │  ...  │   worker N    │
  ┌──────────────────┐   │  event loop   │  │  event loop   │       │  event loop   │
  │      master      │   └───────────────┘  └───────────────┘       └───────────────┘
  │ binds :80, reads │
  │ config, forks,   │   ┌───────────────┐  ┌───────────────┐
  │ handles signals. │   │ cache manager │  │ cache loader  │
  │ Nothing else.    │   │  evicts LRU   │  │  warms index  │
  └──────────────────┘   └───────────────┘  └───────────────┘
```

| process | job |
|---|---|
| **master** | binds `:80` (needs root for ports < 1024), reads config, `fork()`s the others, handles signals (reload, stop). **Never touches a connection.** |
| **workers** | one per CPU core (🧠 `worker_processes auto;`). Each runs its own event loop and does all the real work. |
| **cache manager** | periodically **evicts** from the LRU to keep the disk cache under `max_size`. |
| **cache loader** | at startup, **warms the index**: scans the on-disk cache and loads its keys into shared memory (then exits 🧠). |

```
$ ps -o pid,ppid,args -C nginx
2663     1  nginx: master process
2665  2663  nginx: worker process
2666  2663  nginx: worker process
2667  2663  nginx: cache manager process
```

Read it: master's parent (PPID) is `1` (init/systemd); every other process's parent is the master `2663`.

> **The count does not move under load.** Run the same `ps` against `01-fork/forkd` (the repo's fork-per-client server) and watch it move. *Same `fork()` syscall — the only difference is how often you call it.*

🧠 **Why a master/worker split?** Graceful reloads: `nginx -s reload` sends `SIGHUP` to the master, which re-reads config, starts new workers, and tells old workers to finish their current requests and exit. Zero dropped connections. Also, the master can keep root (for binding ports) while workers drop to an unprivileged user.

**Source pointers:** `ngx_process_cycle.c` → `ngx_master_process_cycle()` ~L74 · `ngx_start_worker_processes()` ~L358 · `ngx_process.c` → `ngx_spawn_process()` ~L87.

### 2.4 All of nginx, in six lines (slide 12)

```c
timer = ngx_event_find_timer();
ngx_process_events(cycle, timer, flags);                 /* epoll_wait() */
ngx_event_process_posted(cycle, &ngx_posted_accept_events);
ngx_shmtx_unlock(&ngx_accept_mutex);
ngx_event_expire_timers();
ngx_event_process_posted(cycle, &ngx_posted_events);
```

Line by line:

| # | line | what it does |
|---|---|---|
| 1 | `ngx_event_find_timer()` | "How many ms until my **nearest timer** fires?" 🧠 Timers (read timeouts, keep-alive timeouts...) live in a **red-black tree**, so finding the smallest is cheap. |
| 2 | `ngx_process_events(cycle, timer, flags)` | Calls **`epoll_wait()`** with that timer as the timeout. **This is where a worker spends 99.9% of its life: asleep.** |
| 3 | `ngx_event_process_posted(..., &ngx_posted_accept_events)` | Run handlers for **new connections** (accept events) first. |
| 4 | `ngx_shmtx_unlock(&ngx_accept_mutex)` | Release the accept mutex (if used) so sibling workers can accept. Done right after accepting so the lock is held as briefly as possible. |
| 5 | `ngx_event_expire_timers()` | Fire every timer whose time has come (e.g. close a client that's been silent too long). |
| 6 | `ngx_event_process_posted(..., &ngx_posted_events)` | Run handlers for **all other** ready events (reads, writes). |

Key ideas on the slide:
- **The timer is an ARGUMENT to the sleep. nginx never polls for timeouts.** It doesn't wake up every 10 ms to check "has anyone timed out?". It sleeps exactly until the next deadline *or* until I/O happens, whichever is first.
- **Handlers do not run inside the poll; they are posted and drained after.** `epoll_wait` returns a batch → events are put on queues (`posted` lists) → the queues are processed. This lets nginx do accepts first and release the mutex quickly.
- **Everything else in `src/http/` is a callback reached from line 2 or line 6.** "Read these six lines until they are boring."

**Source:** `src/event/ngx_event.c` → `ngx_process_events_and_timers()` ~L195 · compare with the repo's `04-epoll/epolld.c`.

### 2.5 Load balancing between workers, in one line of arithmetic (slide 13)

```c
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
```

- `connection_n` = total connection slots this worker has (`worker_connections`).
- `free_connection_n` = how many are still free.
- **Read it as English:** *once I am using more than seven-eighths of my connection slots, stop competing for new connections and let my siblings take them.*

Worked example with `worker_connections 1024`:

| free slots | `1024/8 − free` | result |
|---|---|---|
| 900 | 128 − 900 = −772 | negative → keep accepting |
| 128 | 128 − 128 = 0 | borderline |
| 50 | 128 − 50 = **78** | positive → skip the next 78 chances to grab new connections |

🧠 In the source, when this value is positive the worker just decrements it each loop iteration instead of trying to take the accept mutex, so it naturally "backs off" for a while.

Three side-notes from the slide:

| | |
|---|---|
| **No coordinator** | No shared counter, no leader, no heartbeat. Each worker decides **alone**, from a number it already has. |
| **`accept_mutex` is now off** | Default **off since 1.11.3**. `EPOLLEXCLUSIVE` (Linux 4.5+) and `SO_REUSEPORT` let the **kernel** do the fan-out instead. |
| **A userspace fix for a kernel problem** | The **thundering herd** was real in 2004. The kernel fixed it. The workaround became dead weight — and nginx removed it (by default). |

🧠 **Thundering herd** explained:

```
 one new connection arrives on :80
          │
          ▼
 kernel wakes ALL workers sleeping on the listen socket
   worker0 ✔ accept() succeeds
   worker1 ✘ EAGAIN (someone else got it)  ← woke up for nothing
   worker2 ✘ EAGAIN                         ← woke up for nothing
   worker3 ✘ EAGAIN                         ← woke up for nothing
```
nginx's fix (2004): an **accept mutex** in shared memory, so only one worker at a time listens for new connections. The kernel's fixes: **`EPOLLEXCLUSIVE`** (wake only one waiter) and **`SO_REUSEPORT`** (each worker gets its *own* listening socket on the same port; the kernel hashes connections across them — 🧠 enabled in nginx with `listen 80 reuseport;`).

**Source:** `src/event/ngx_event_accept.c` ~L139 · `src/event/ngx_event.c` `ngx_process_events_and_timers()` ~L224 (`ngx_trylock_accept_mutex`).

### 2.6 `select` is O(watched). `epoll` is O(active). (slide 14)

🧠 First, what each API looks like:

```c
/* select(): you hand the kernel a BITMAP of fds every single call */
fd_set rd;
FD_ZERO(&rd);
for (each of my n fds) FD_SET(fd, &rd);          // O(n) build
select(maxfd + 1, &rd, NULL, NULL, &timeout);    // O(n) kernel scan
for (each of my n fds) if (FD_ISSET(fd, &rd))    // O(n) your scan
    handle(fd);

/* epoll: the watch list lives IN THE KERNEL */
int ep = epoll_create1(0);
epoll_ctl(ep, EPOLL_CTL_ADD, fd, &ev);           // once per fd, when it opens
int n = epoll_wait(ep, events, MAX, timeout_ms); // returns ONLY ready ones
for (i = 0; i < n; i++) handle(events[i]);       // O(ready)
```

| | `select()` | `epoll` |
|---|---|---|
| **Where the watch list lives** | In your process. **Rebuilt every call.** | In the **kernel**. Edited with `epoll_ctl()`. |
| **Cost per iteration** | O(n) to build + O(n) in kernel + O(n) to scan | **O(number of ready events)** |
| **10,000 idle, 10 active** | **30,000** units of work to find 10 events | **10** |
| **Descriptor ceiling** | `FD_SETSIZE = 1024`, compile-time, in a header | `ulimit -n` |
| **What `ulimit -n` does** | **Nothing.** The limit is a bitmap width. | **Everything.** |
| **Portable?** | Everywhere since 1983 | Linux only (**kqueue** on BSD/macOS) |

> **"Everything else in this comparison is downstream of one decision: who owns the watch list."**

**Why `ulimit` doesn't help `select`:** `fd_set` is a fixed-size bitmap of 1024 bits, compiled into your program. An fd numbered 1024 or higher has no bit to set. Raising `ulimit -n` lets the kernel *give* you fd 1500, but `FD_SET(1500, &rd)` writes **past the end of the array** → memory corruption (🧠 with glibc's fortify checks you get `*** buffer overflow detected ***` and an abort).

🧠 `poll()` fixes the 1024 limit (you pass an array of `struct pollfd`, any size) but is still O(n) per call, because the list is still passed in every time.

**Source:** open both in a split window — `ngx_select_module.c` → `ngx_select_process_events()` ~L222 vs `ngx_epoll_module.c` → `ngx_epoll_process_events()` ~L784. Same signature, same job, ~200 lines each.

**Pitfalls:**
- "epoll is faster than select" is only true when **most fds are idle**. With 10 fds, all busy, the difference is negligible.
- epoll is **Linux-only**. On your Mac, nginx uses **kqueue** (same idea). nginx picks the method at `./configure` time (see §4.2).

### 2.7 The bug every event-loop author writes exactly once (slide 15)

**The problem:**
1. `epoll_wait` returns a **batch** of events, say 10.
2. Handling **event 1** closes the connection that **event 7** points at (e.g. a proxy closes the paired upstream socket).
3. The fd number is freed... and if a **new client** is accepted in the same iteration, the kernel reuses the lowest free fd number → the **same fd**.
4. Now **event 7** (which was about the *old* connection) is delivered to the *new* client. Wrong data, wrong state, chaos.

```
  batch from epoll_wait:  [ ev1(fd 12) ... ev7(fd 30) ... ]
  handle ev1  → closes fd 30
  accept()    → new client gets fd 30 (recycled!)
  handle ev7  → "fd 30 is readable" ... but that event was for the DEAD connection
```

**nginx's fix: steal the low bit of a pointer.**

```c
c = event_list[i].data.ptr;

instance = (uintptr_t) c & 1;
c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);

if (c->fd == -1 || rev->instance != instance) {
    /* the stale event from a file descriptor
       that was just closed in this iteration */
    continue;
}
```

- When registering an fd with epoll, nginx stores a **pointer to the connection struct** in `data.ptr`.
- **Structs are aligned** (to at least 2, usually 8 bytes), so **the low bit of the pointer is always 0**. nginx **borrows that bit** as a **generation flag** (`instance`) and **flips it every time the connection struct is reused**.
- When the event comes back: extract the bit, clear it to recover the real pointer, and compare with the connection's current `instance`. **Mismatch → stale event → skip.** Also skip if `c->fd == -1` (already closed).
- *"Free bit, no extra memory, one branch."*

The repo's `04-epoll/epolld.c` does the same thing with an **explicit generation counter** (easier to read, costs a few bytes).

**Source:** `ngx_epoll_module.c` → `ngx_epoll_process_events()` ~L839.

**Interview gotcha:** "Why can't I just check if the fd is still open?" Because the fd number *is* open — it now belongs to a different connection. fd numbers are recycled; identity needs a generation.

### 2.8 sendfile: four copies become two (slide 16)

**Plain English:** to send a file over the network the naive way, the data takes a detour through your program's memory, even though your program never looks at it. `sendfile()` tells the kernel "send this file to that socket", and the detour disappears.

```
 read() + write()  —  4 copies, 2 mode switches per chunk
 ┌──────┐   ┌────────┐   ╔════════╗   ╔════════╗   ┌─────┐
 │ disk │ → │ page   │ → ║  your  ║ → ║ socket ║ → │ NIC │
 └──────┘   │ cache  │   ║ buffer ║   ║ buffer ║   └─────┘
            └────────┘   ╚════════╝   ╚════════╝
      DMA        CPU copy      CPU copy       DMA
           (kernel→user)  (user→kernel)
                  ▲ outlined in red on the slide = the copies you pay CPU for

 sendfile()  —  2 copies, both DMA, 0 mode switches per chunk
 ┌──────┐   ┌────────┐   ╔════════╗   ┌─────┐
 │ disk │ → │ page   │ → ║ socket ║ → │ NIC │
 └──────┘   │ cache  │   ║ buffer ║   └─────┘
            └────────┘   ╚════════╝
```

- The slide: *"The two boxes outlined above are the copies you pay CPU for. sendfile deletes the one through your address space — and with it, the two mode switches per 64 KiB chunk."*
- **"The data never enters your process."**
- 🧠 Precise detail: on modern Linux with a NIC that supports **scatter-gather DMA**, the socket buffer only holds *descriptors* pointing into the page cache, and the NIC DMAs straight from the page cache — that's the "both DMA" case. Without that hardware support there's still one in-kernel CPU copy (page cache → socket buffer), but it's still fewer copies and far fewer syscalls than read/write.

**Why you can't use it for everything:** sendfile copies **kernel-side**. If you need to **touch the bytes** — **gzip**, **TLS** encryption, **templating** — they must come into user space.
- That's why **`gzip on` and `sendfile on` fight** (a gzipped response can't be sendfile'd).
- And it's why **kTLS** (kernel TLS 🧠: the kernel does the encryption, so encrypted responses can still use sendfile) exists.

🧠 In nginx config: `sendfile on;` in the `http` block. nginx only uses it for responses that are plain file buffers going out unmodified.

**Source:** `ngx_linux_sendfile_chain.c` → `ngx_linux_sendfile_chain()` ~L50 · `ngx_linux_sendfile()` ~L232 · repo: `05-sendfile/`.

### 2.9 Now the honest part: the benchmark (slide 17)

```
$ ./sendfiled 8005 www/big.bin              # 64 MB, over loopback

mode=read       wall  37.4 ms | sys 37.3 |  2048 syscalls
mode=mmap       wall  23.3 ms | sys 23.1 |     4 syscalls
mode=sendfile   wall  33.2 ms | sys 32.3 |     1 syscall
```

**On loopback, sendfile does not win on wall-clock time.** Students will run this and see it. Why?

- **There is no NIC and no DMA** on loopback. The kernel `memcpy`s into the receiver's buffer either way, and `curl` (the receiver) is competing for the same CPU. The transfer is **memory-bandwidth bound**, so all three land in the same place.
- **What is real on any machine:** **2048 syscalls become 1**, and **64 MB never enters the process's address space**.
- **The win shows up as headroom under concurrency** — CPU you did not spend copying is CPU you spend on somebody else's request — **not as a faster single transfer**.

🧠 Where 2048 comes from: 64 MB ÷ 64 KiB chunks = 1024 chunks; each chunk = one `read()` + one `write()` = 2048 syscalls. `mmap` mode maps the whole file into memory and writes it from there (a handful of calls: map, write, unmap...).

> **"A benchmark that does not model the bottleneck measures the wrong thing. That lesson is worth as much as sendfile is."**

### 2.10 Why "reverse" proxy? Because "proxy" was taken. (slide 18)

```
 FORWARD PROXY (Squid, 1996)                      REVERSE PROXY (nginx)
 sits next to the CLIENTS                         sits next to the SERVERS

  [student]─┐                                                      ┌─[app server 1]
  [student]─┼─▶[proxy]──▶ the whole internet   the whole internet ─▶[nginx]─┼─[app server 2]
  [student]─┘                                                      └─[app server 3]
   knows your users,                               client doesn't know it's there
   protects your uplink                            protects your app servers
```

| | Forward proxy (Squid, 1996) | Reverse proxy (nginx) |
|---|---|---|
| sits next to | the **clients** | the **servers** |
| classic use | one person on campus downloads an image, it's cached, nobody else downloads it again | same caching, **plus**: route by **path or host**, distribute across **upstreams** by **weight** or **least-connections**, **fall back** when one dies, **terminate TLS**, and **hold ten thousand slow clients** so your app never meets one |
| who configures it | the client (browser proxy settings) | the server operator; the client usually **cannot tell it's there** |

> *Same machinery, opposite side of the internet. The client does not configure a reverse proxy and usually cannot tell it is there — which is exactly why it can do things the client would never agree to.*

**The killer feature nobody lists: it absorbs slow clients. Your app server never talks to a phone on 2G.**

🧠 Why that matters: an app server (say a Python/Gunicorn worker) is expensive, often one request at a time. If it has to drip a 2 MB response to a phone at 20 KB/s, that worker is stuck for 100 seconds. With nginx in front: the app hands the full response to nginx over a fast local link in milliseconds and is free again; nginx (cheap, event-driven) does the slow dripping. Same for slow uploads — nginx **buffers** the request body before passing it on.

**Term check:** **TLS termination** = nginx does the HTTPS decryption; the app behind it speaks plain HTTP on a private network.

---

## 3 · THE FEATURES YOU CONFIGURE ⚙️

### 3.1 The cache: two structures, one of them on disk (slide 19)

```nginx
proxy_cache_path temp/cache levels=1:2 keys_zone=demo:10m
                 max_size=100m inactive=60s use_temp_path=off;
```

```
          SHARED MEMORY (keys_zone=demo:10m)              DISK (temp/cache/)
   ┌──────────────────────────────────────────┐      ┌──────────────────────────┐
   │  rbtree: key-hash → node   (fast lookup) │      │ temp/cache/c/29/b7f54b...│
   │  LRU queue: head ⇄ ... ⇄ tail (eviction) │ ───▶ │ temp/cache/a/1f/09c3e1...│
   │  node: expire time, use count, ...       │      │   (the actual responses) │
   └──────────────────────────────────────────┘      └──────────────────────────┘
     the INDEX — shared by all workers                  the DATA
```

| parameter | meaning |
|---|---|
| `keys_zone=demo:10m` | **10 MB of SHARED MEMORY** (visible to all workers) holding the **rbtree** and the **LRU queue**. **The index, not the data.** Roughly **8,000 keys per megabyte**. |
| `levels=1:2` | **Two directory levels on disk**, so no single directory holds a million files and destroys your filesystem's lookup time. 🧠 The file name is the MD5 of the cache key; `1:2` means the last 1 hex char is the first dir and the next 2 chars are the second dir, e.g. `.../c/29/b7f54b2df7773722d382f4809d65029c`. |
| `max_size=100m` | **Disk ceiling.** The **cache manager** process evicts from the **LRU tail** to stay under it. |
| `inactive=60s` | Evicted if **not READ for 60s — regardless of TTL.** A *different knob* from `proxy_cache_valid`, and the one people confuse. |
| `use_temp_path=off` | 🧠 Write responses directly into the cache directory instead of a separate temp dir then moving them (avoids a cross-filesystem copy). |

🧠 To actually *use* the zone you also need, inside a `location`:

```nginx
proxy_cache demo;
proxy_cache_valid 200 10s;      # how long a 200 response is FRESH
proxy_pass http://app;
```

**"Ask the class":** `proxy_cache_valid` is **10s** and `inactive` is **60s**. Which one wins, and what does each mean?

<details><summary>Answer</summary>

Neither "wins" — they measure different things:

- **`proxy_cache_valid 10s`** = **freshness (TTL)**. After 10 s the cached copy is **stale**: the next request will go to the origin to refresh it (or may be served stale under `proxy_cache_use_stale`). The file stays on disk.
- **`inactive 60s`** = **idle eviction**. If *nobody requests* the item for 60 s, the cache manager **deletes it from disk and from the index**, whether it's fresh or stale.

So a popular item: stays on disk, refreshed from origin every 10 s. An unpopular item: becomes stale at 10 s, still on disk (usable as a stale fallback) until 60 s of no reads, then deleted. If you set `inactive` *shorter* than `proxy_cache_valid`, items can be thrown away while still fresh.
</details>

### 3.2 The LRU is one line (slide 20)

```c
q   = ngx_queue_last(&cache->sh->queue);
fcn = ngx_queue_data(q, ngx_http_file_cache_node_t, queue);

wait = fcn->expire - now;

if (wait > 0) { ... break; }                    /* tail is still fresh */

if (fcn->count == 0) {                          /* nobody is streaming it */
    ngx_http_file_cache_delete(cache, q, name);
}
```

- **Take the tail** (the least recently used item). **Is it old? Is nobody using it? Delete it.**
- **That is the entire eviction policy.**
- If the tail isn't expired yet, nothing behind it can be older, so `break` — stop.
- **`count == 0`** is why an object **being streamed to a client right now cannot be evicted out from under it.** `count` is a reference count of active users.
- `ngx_queue_data(q, type, field)` = "given a pointer to the embedded link `q`, give me the struct that contains it" (see §4.3).

🧠 How it stays "LRU": every time an item is used, nginx moves its node to the **head** of the queue. So the tail is always the one touched longest ago.

**Read `src/core/ngx_queue.h` first** — fifty lines of macros for an intrusive doubly-linked list. The cache LRU, the posted-events list and half of `src/core/` ride on it.

**Source:** `ngx_http_file_cache.c` → `ngx_http_file_cache_expire()` ~L1860 · `..._lookup()` ~L1029 · `..._open()` ~L265.

### 3.3 Two lines that save you at 3am (slide 21)

```nginx
proxy_cache_lock on;

proxy_cache_use_stale error timeout
                      updating http_500 http_502 http_503 http_504;
```

**`proxy_cache_lock on;` — the anti-stampede switch.**

```
 WITHOUT lock                           WITH lock
 popular object expires                 popular object expires
 req1 ─ MISS ─▶ origin                  req1 ─ MISS ─▶ origin
 req2 ─ MISS ─▶ origin                  req2 ─ waits ┐
 req3 ─ MISS ─▶ origin                  req3 ─ waits ├─ all get req1's answer
 ... 500 requests in 200 ms ─▶ origin   ...    waits ┘
 origin melts 🔥                        origin sees ONE request
```
Without it, every request that arrives in the next ~200 ms is a MISS and every one goes to the origin. With it, one goes and the rest wait for the answer. **Off by default.** (🧠 This is called a *cache stampede* or *dog-pile*.)

**`proxy_cache_use_stale ...` — serve stale rather than fail.**
Your origin is down. You have a copy from four minutes ago. Which would your users rather have — a slightly old page, or a **502**?
- The listed conditions (`error`, `timeout`, `http_500`, `http_502`, `http_503`, `http_504`) say *when* a stale copy may be served instead of the error.
- **`updating`** also means **one request refreshes while everyone else keeps getting the stale copy instantly.** (🧠 Combined with `proxy_cache_background_update on;`, even that one request doesn't wait.)

**See it work:**
```
$ curl -i localhost:8082/ | grep X-Cache   →   X-Cache-Status: MISS
$ curl -i localhost:8082/ | grep X-Cache   →   X-Cache-Status: HIT
```

> **`add_header X-Cache-Status $upstream_cache_status;` — put this in every cache config you ever write.**

🧠 Possible values of `$upstream_cache_status`: `MISS`, `HIT`, `EXPIRED`, `STALE`, `UPDATING`, `REVALIDATED`, `BYPASS`.

### 3.4 `location` precedence has nothing to do with the order you wrote them in (slide 22)

```nginx
location = /exact     { ... }   # 1
location ^~ /images/  { ... }   # 2
location ~ \.php$     { ... }   # 3
location /            { ... }   # 4
```

| # | form | rule | request | result |
|---|---|---|---|---|
| 1 | `location = /exact` | **Exact match.** Wins immediately, search stops. | `/exact` | **1** |
| 2 | `location ^~ /images/` | **Longest prefix, and it suppresses the regex search.** | `/images/a.php` | **2** |
| 3 | `location ~ \.php$` | **Regex. FIRST match in file order, not longest.** | `/other/a.php` | **3** |
| 4 | `location /` | **Longest prefix, used only if no regex matched.** | `/anything` | **4** |

🧠 The full algorithm as a flowchart:

```
 request URI
     │
     ▼
 exact "=" match? ──yes──▶ USE IT. stop.
     │no
     ▼
 find the LONGEST matching prefix (plain or ^~), remember it
     │
     ▼
 is that longest prefix a "^~"? ──yes──▶ USE IT. stop. (regexes never checked)
     │no
     ▼
 try regexes (~ case-sensitive, ~* case-insensitive) IN FILE ORDER
     │
     ├── first one that matches ──▶ USE IT. stop.
     │
     └── none matched ──▶ use the remembered longest prefix
```

**Make them predict all four answers before you press enter.** Roughly **half the room gets `/images/a.php` wrong** — they expect the regex to win. But `^~` means "if I'm the longest prefix, don't even look at regexes".

This exact config is on **:8081** in the repo.

**nginx has no `.htaccess`.** That's not a missing feature. It's the reason nginx does not `stat()` its way up the directory tree on every single request. 🧠 Apache with `AllowOverride` must check `/var/www/.htaccess`, `/var/www/site/.htaccess`, `/var/www/site/images/.htaccess`... for every request, because those files can change at any time. nginx reads all config once at startup/reload.

**Pitfalls:**
- `^~` is **not** a regex. The `~` in it is misleading. It's a prefix match with a "no regex" flag.
- Regex order **does** matter (first match wins); prefix order **does not** (longest wins).
- `location /images` (no trailing slash) also matches `/imagesXYZ`. 🧠

### 3.5 Why 500 prefixes are free and 500 regexes are not (slide 23)

```c
n  = (len <= node->len) ? len : node->len;
rc = ngx_filename_cmp(uri, node->name, n);

if (rc != 0) {
    node = (rc < 0) ? node->left : node->right;           /* a TREE */
    continue;
}

if (len > node->len && node->inclusive) {
    node = node->tree;     uri += n;    len -= n;         /* descend */
}
```

- **Prefixes:** a **ternary search tree**, **built at config-parse time**, walked **one URI segment at a time**.
  - Compare the URI with the node's name. Less → go **left**. Greater → go **right**. Equal and the URI is longer → **descend** into the child `tree` with the rest of the URI. Each step discards a whole branch, so lookup cost grows roughly with the *depth* (log of the number of prefixes), not the count.
- **Regexes:** a **plain array**, walked **in order, every request**. 500 regexes = up to 500 regex executions per request.

```
  prefix tree for  /  /api/  /api/v1/  /images/  /static/
                 "/"
                  │ (descend)
           ┌── "images/" ──┐
       "api/"           "static/"
         │ (descend)
        "v1/"
```

> **"The data structure is the documentation."** Once you see that prefixes are a **tree** and regexes are an **array**, the precedence rules stop being trivia and become **the only rules those two structures could have**: a tree walk naturally ends at the *longest* match; an array scan naturally stops at the *first* match.

**Source:** `src/http/ngx_http_core_module.c` → `ngx_http_core_find_location()` ~L1446 · `ngx_http_core_find_static_location()` ~L1542.

**Practical takeaway:** prefer prefix locations; use regexes sparingly; put the most-hit regexes first.

### 3.6 Upstreams & weights: smooth weighted round robin, measured (slide 24)

🧠 Config shape:

```nginx
upstream app {
    server 127.0.0.1:8004 weight=3;
    server 127.0.0.1:8001 weight=1;
}
```

**The algorithm (from the source):**

```c
peer->current_weight += peer->effective_weight;
total                += peer->effective_weight;

if (peer->effective_weight < peer->weight)
    peer->effective_weight++;              /* passive health check */

if (best == NULL || peer->current_weight > best->current_weight)
    best = peer;     ...

best->current_weight -= total;
```

In words, for each request:
1. Every peer adds its weight to its `current_weight`.
2. Pick the peer with the **highest** `current_weight`.
3. Subtract the **total** of all weights from the winner.

| field | meaning |
|---|---|
| `weight` | what you configured (3 or 1) |
| `effective_weight` | weight actually used right now. **Reduced when a peer fails**, then crawls back up by 1 per selection → that's the **passive health check** comment. A flaky server gets less traffic automatically, and is gradually trusted again. |
| `current_weight` | running score that decides who's next |

🧠 **Trace with weights a=3, b=1 (total = 4):**

Strict `>` means that on a tie the peer listed **first** stays `best`. Call the weight-3 server `a` and the weight-1 server `b`, and assume `b` is listed first:

| request | after adding weights (b, a) | chosen (max) | after winner −= 4 (b, a) |
|---|---|---|---|
| 1 | b=0+1=1, a=0+3=**3** | **a** | b=1, a=−1 |
| 2 | b=1+1=**2**, a=−1+3=2 (tie → first listed) | **b** | b=−2, a=2 |
| 3 | b=−2+1=−1, a=2+3=**5** | **a** | b=−1, a=1 |
| 4 | b=−1+1=0, a=1+3=**4** | **a** | b=0, a=0 ← back to the start |

Sequence `a b a a`, then it repeats. If `a` is listed first instead, the tie at request 2 goes to `a` and you get `a a b a` — a **rotation of the same cycle**: 3 a's and 1 b per 4 requests, with the b sitting *between* a's rather than after a burst of three. The slide shows `a b a a`.

```
 weights 3 : 1
 naive WRR      a a a b  a a a b      ← bursts of 3 at server a
 nginx (smooth) a b a a  a b a a      ← same ratio, interleaved
```
**Same ratio. No bursts at the origin.**

**Measured:**
```
$ for i in $(seq 24); do curl -s "localhost:8082/x-$i-$RANDOM" -o /dev/null; done
$ grep -o 'upstream=[0-9.:]*' logs/access.log | sort | uniq -c
      6 upstream=127.0.0.1:8001
     18 upstream=127.0.0.1:8004         <- exactly 3:1
```
- 24 requests → 18 : 6 = **exactly 3 : 1**.
- 🧠 The URL contains `$i-$RANDOM` so every request is a **unique URL** → a cache MISS → it actually reaches an upstream (port 8082 also has the cache on). Same URL 24 times would mostly be cache HITs and you'd see 1 upstream hit.
- 🧠 The access log contains `upstream=...` because the repo's `log_format` includes `$upstream_addr`.

**Source:** `ngx_http_upstream_round_robin.c` → `ngx_http_upstream_get_peer()` ~L811 · run it: `07-nginx`, port 8082.

🧠 **Other balancing methods** you'll hear about: `least_conn` (send to the server with fewest active connections — homework 4), `ip_hash` / `hash $key` (same client → same server, "sticky"), and `random two least_conn`.

### 3.7 Fallbacks: `try_files` — one line, and every framework's pretty URLs work (slide 25)

```nginx
location / {
    try_files $uri $uri/ /index.html?$args;
}
```

> *"Is this a real file? A real directory? No? Then hand the whole URI to the app and let it route."*

```
 request /dashboard/settings?tab=2
     │
     ├─ is $uri  (/dashboard/settings) a FILE on disk?       → no
     ├─ is $uri/ (/dashboard/settings/) a DIRECTORY on disk? → no
     └─ fallback: internally redirect to /index.html?tab=2   ← NOT tested, just used
                  the SPA's JavaScript router reads the URL and shows the settings page
```

- `$uri` = the request path; `$args` = the query string.
- **The last entry is the only one that is never tested. It is the fallback, not a candidate.** (🧠 It can also be `=404` to return a status code.)

**Fallback across a network boundary:**
```nginx
location /maybe/ {
    try_files $uri @app;
}
location @app {
    proxy_pass http://app;
}
```
- `@app` is a **NAMED location**: reachable **only from `try_files` or `error_page`**, **never from a URL**. A client can't request `/@app`.
- "Try disk first; if it is not there, ask the upstream." 🧠 Classic use: serve static assets directly from disk, send everything else to the app server.

> **WordPress, Rails, Django, every SPA router: all of them are this directive plus an application that reads `REQUEST_URI`. Once you see it, the magic evaporates.**

🧠 WordPress's canonical nginx rule is `try_files $uri $uri/ /index.php?$args;` — exactly the same pattern, with PHP's `index.php` as the router.

**Source:** `ngx_http_try_files_module.c` → `ngx_http_try_files_handler()` ~L79 · demo: port **8084**.

**Pitfall:** `try_files` checks the **filesystem** (`stat()`), so it's relative to `root`. If `root` is wrong, every request falls through to the fallback and it *looks* like routing is broken.

---

## 4 · THE WALKTHROUGH: READING THE NGINX SOURCE 🔍

### 4.1 Line numbers drift. Function names do not. (slide 26)

```
$ git clone --depth 1 https://github.com/nginx/nginx.git
$ cd nginx && git log -1 --oneline
   231a60ee nginx-1.31.5-RELEASE

$ grep -rn "^ngx_epoll_process_events" src/
```

**nginx puts the return type on its own line:**

```c
static ngx_int_t                       ← return type, own line
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{                                      ← ^name( matches ONLY this line
```

So `grep "^name("` finds **the definition and nothing else** — not the declarations (`static ngx_int_t ngx_epoll_process_events(...);` doesn't start with the name), not the call sites (indented). One habit, and the whole tree becomes navigable.

- `--depth 1` = 🧠 shallow clone, only the latest commit (fast; no history).
- Every pointer in these notes gives a **function name first** and a line number second. The line numbers are pinned to **1.31.5** and **will be wrong** by the time you read this. The names will not be.

> *"That is not a disclaimer. It is the skill: navigate a codebase by structure, not by coordinates."*

### 4.2 The tree in two minutes (slide 27)

| dir | contents |
|---|---|
| `src/core/` | **nginx's own standard library**: string, array, list, hash, rbtree, queue, palloc, buf, conf, cycle, connection. It has its own everything because **in 1999 you could not portably rely on anyone else's.** |
| `src/event/` | **The loop.** `modules/` holds **six ways to wait** — **epoll, kqueue, select, poll, devpoll, eventport**. Same interface, **chosen at `./configure` time.** |
| `src/http/` | The **HTTP state machine**, the **phase engine**, the **upstream framework**. `modules/` is every directive you've ever typed. |
| `src/os/unix/` | The **syscall layer**: sendfile per OS, process spawning, shared memory, atomics. |
| `src/stream/` | The same architecture **again**, for raw **TCP and UDP**. nginx as an **L4 load balancer**. |
| `src/mail/` | And a **third time**, for **IMAP, POP3 and SMTP** proxying. |

🧠 The six wait methods: **epoll** (Linux), **kqueue** (FreeBSD/macOS), **select** and **poll** (portable POSIX), **/dev/poll** (Solaris, older), **event ports** (Solaris/illumos). One interface (`ngx_event_actions_t`: add, del, process_events...), six back-ends — classic strategy pattern.

🧠 **L4 vs L7:** an **L7 (layer 7, application layer)** load balancer understands HTTP (can route by URL, add headers). An **L4 (transport layer)** load balancer only sees TCP/UDP connections and forwards bytes — used for databases, MQTT, game servers, etc.

> **Three protocol trees, one core.** When you have read `src/http/` you can read `src/stream/` in an afternoon — and that repetition is the argument for the architecture.

### 4.3 Two files to read before any of the code (slide 28)

**`src/core/ngx_queue.h` — the intrusive list (~50 lines of macros)**

- An **intrusive doubly-linked list**: the link pointers live **inside the node**, so putting something in a list **costs no allocation**, and getting from a link back to the data is **a pointer subtraction**.
- The cache LRU, the posted-events queues and half of `src/core/` ride on it.

🧠 What that means in code:

```c
typedef struct ngx_queue_s ngx_queue_t;
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};

typedef struct {
    ngx_str_t     key;
    time_t        expire;
    ngx_queue_t   queue;      /* <- the links are EMBEDDED in the data */
} my_node_t;

/* from a link back to its containing struct: subtract the field offset */
#define ngx_queue_data(q, type, link) \
    (type *) ((u_char *) q - offsetof(type, link))
```

```
 NON-intrusive (e.g. std::list<T*>)        INTRUSIVE (ngx_queue)
 ┌──────┐   ┌──────┐                       ┌────────────────┐   ┌────────────────┐
 │ node │──▶│ node │──▶ ...                │ my data        │   │ my data        │
 │ *data│   │ *data│                       │ ┌────────────┐ │   │ ┌────────────┐ │
 └──┬───┘   └──┬───┘                       │ │prev | next │◀┼──▶┼▶│prev | next │ │
    ▼          ▼                           │ └────────────┘ │   │ └────────────┘ │
  [data]     [data]   2 allocations each   └────────────────┘   └────────────────┘
                                             1 allocation, links inside
```
(It's the same trick as the Linux kernel's `list_head` / `container_of`.)

**`src/core/ngx_palloc.h` — the memory pool**

- **nginx does not free per-request memory.** It allocates from a **pool attached to the request** and **destroys the whole pool at the end**.
- **No refcounting. No leak class. No `free()` on the hot path** — and almost no cleanup branches anywhere in the codebase.

```
 request starts → pool = ngx_create_pool(4096)
                  ngx_palloc(pool, 40)    → bump pointer
                  ngx_palloc(pool, 200)   → bump pointer
                  ngx_palloc(pool, 16)    → bump pointer
                  ...hundreds of small allocations, never individually freed...
 request ends   → ngx_destroy_pool(pool)  → everything gone at once
```

🧠 Why "no leak class": in normal C, every early `return` on an error path must remember to `free()` what was allocated so far. That's where most leaks come from. With a pool, error paths just return; the pool is destroyed when the request ends anyway.

> Read these two and a large fraction of nginx stops looking like C and starts looking like a language with a runtime. Skip them and every function has three constructs in it you do not recognise.

### 4.4 Ten stops (slide 29)

| # | what | file | function | in the repo |
|---|---|---|---|---|
| 1 | Process model | `os/unix/ngx_process_cycle.c` | `ngx_start_worker_processes` | `01-fork` |
| 2 | The loop | `event/ngx_event.c` | `ngx_process_events_and_timers` | `04-epoll` |
| 3 | Accept + herd | `event/ngx_event_accept.c` | `ngx_event_accept` | `04-epoll` |
| 4 | epoll + stale events | `event/modules/ngx_epoll_module.c` | `ngx_epoll_process_events` | `04-epoll` |
| 5 | The parser | `http/ngx_http_parse.c` | `ngx_http_parse_request_line` | `03-select` |
| 6 | Phases | `http/ngx_http_core_module.c` | `ngx_http_core_run_phases` | — |
| 7 | Routing | `http/ngx_http_core_module.c` | `ngx_http_core_find_static_location` | `07-nginx :8081` |
| 8 | sendfile | `os/unix/ngx_linux_sendfile_chain.c` | `ngx_linux_sendfile` | `05-sendfile` |
| 9 | Cache LRU | `http/ngx_http_file_cache.c` | `ngx_http_file_cache_expire` | `07-nginx :8082` |
| 10 | FastCGI records | `http/modules/ngx_http_fastcgi_module.c` | `ngx_http_fastcgi_process_record` | `06-fastcgi` |

Stops 1–4 and 7–9 are covered in §2 and §3 above. Stops 5, 6 and 10 follow. The full tour with line numbers, code excerpts and Socratic questions is **`NGINX-TOUR.md`** in the companion repo.

### 4.5 Stop 5 — The HTTP parser is a resumable state machine, not a string function (slide 30)

```c
enum {
    sw_start = 0, sw_method, sw_spaces_before_uri, sw_schema,
    sw_schema_slash, sw_schema_slash_slash, sw_host_start,
    sw_host, sw_port, sw_after_slash_in_uri, sw_check_uri,
    sw_uri, sw_http_09, sw_http_H, sw_http_HT, ...
} state;

state = r->state;                      /* <- resume where we stopped */
for (p = b->pos; p < b->last; p++) {
    ch = *p;
    switch (state) { ... }
}
```

- **27 states for the request line alone.**
- **No regex. No `strtok`. No substring. No allocation.**
- **One byte at a time, and it can stop between any two.**

**Why?** Because the request line may arrive in **three TCP segments**. `r->state` **survives between calls**, so the parser picks up **mid-method** if it has to.

```
 TCP segment 1: "GE"                     → parser: state=sw_method, saves state, returns AGAIN
 (event loop serves other clients...)
 TCP segment 2: "T /index.ht"            → resumes in sw_method → sw_spaces_before_uri → sw_uri...
 TCP segment 3: "ml HTTP/1.1\r\n"        → resumes in sw_uri → sw_http_H → ... → DONE
```

🧠 The state names read like a path through `GET http://host:port/uri HTTP/1.1`: `sw_method` → `sw_spaces_before_uri` → `sw_schema` (`http`) → `sw_schema_slash` (`:/`) → `sw_schema_slash_slash` (`//`) → `sw_host_start` / `sw_host` / `sw_port` → `sw_after_slash_in_uri` / `sw_uri` → `sw_http_H` → `sw_http_HT` → ... `sw_http_09` handles ancient HTTP/0.9 requests that have no version at all.

> **This is what non-blocking costs you at the source level: every parser has to be resumable, because you can be interrupted anywhere.**

Contrast: a thread-per-connection server can just call a blocking `read()` in a loop until it sees `\r\n` and then parse a complete string. A non-blocking server can't wait, so the parser must remember *where it was*.

🧠 And if the headers are bigger than the initial buffer, nginx allocates a larger one (`large_client_header_buffers`) and copies — that's `ngx_http_alloc_large_header_buffer()`.

**Source:** `ngx_http_parse.c` → `ngx_http_parse_request_line()` ~L108 · `..._header_line()` ~L871 · `ngx_http_request.c` → `..._alloc_large_header_buffer()` ~L1657. Repo: `03-select`.

### 4.6 Stop 6 — Eleven phases, four lines, and `return` means "suspended" (slide 31)

```
POST_READ → SERVER_REWRITE → FIND_CONFIG → REWRITE → POST_REWRITE → PREACCESS
          → ACCESS → POST_ACCESS → PRECONTENT → CONTENT → LOG
```

```c
while (ph[r->phase_handler].checker) {
    rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);
    if (rc == NGX_OK) { return; }              /* NOT done. SUSPENDED. */
}
```

- Every request walks an array of phase handlers `ph[]`. Each **checker** runs the module handlers for its phase and advances `r->phase_handler`.
- **`NGX_OK` here does *not* mean "finished".** It means **"suspended — I'm waiting for something (disk, upstream, client body)"**. The function returns to the event loop.
- **A coroutine, hand-rolled, in C, in 1999.** The request resumes in a **later loop iteration** when its event fires, **re-entering this same `while`** at the phase it left off.

🧠 What each phase is for:

| phase | typical job |
|---|---|
| POST_READ | right after headers are read (e.g. `realip` fixes client IP) |
| SERVER_REWRITE | `rewrite` directives at `server` level |
| FIND_CONFIG | pick the `location` (§3.4) — internal, no modules |
| REWRITE | `rewrite` inside the location |
| POST_REWRITE | internal: if URI changed, go back to FIND_CONFIG |
| PREACCESS | **`limit_req`**, `limit_conn` (rate limiting) |
| ACCESS | **`auth_basic`**, `allow`/`deny` |
| POST_ACCESS | internal: act on ACCESS results (`satisfy any/all`) |
| PRECONTENT | **`try_files`**, `mirror` |
| CONTENT | **`proxy_pass`, `fastcgi_pass`, static file** — generate the response |
| LOG | write the access log |

The slide's mapping: **`limit_req` in PREACCESS · `auth_basic` in ACCESS · `try_files` in PRECONTENT · `proxy_pass`, `fastcgi_pass` and static in CONTENT.**

```
$ grep -rn "NGX_HTTP_.*_PHASE\].handlers" src/http/modules/
    # the map of which module runs when.
    # one-line answer to "why did my directive not fire?"
```

**Gotcha:** directive order in the config file does not decide execution order — the **phase** does. E.g. a `rewrite ... last` (REWRITE phase) runs before `auth_basic` (ACCESS phase) no matter where you write them.

**Source:** `ngx_http_core_module.c` → `ngx_http_core_run_phases()` ~L894 · `ngx_http_core_module.h` → `ngx_http_phases` enum ~L111.

### 4.7 Stop 10 — FastCGI: framing is length or delimiter. This one uses both. (slide 32)

🧠 **Framing recap:** TCP is a byte stream with no message boundaries. A protocol has two ways to say where a message ends: a **length prefix** ("the next 300 bytes are the message") or a **delimiter** ("the message ends at `\r\n\r\n`"). (This is the closing idea of Session 2 that the slide refers back to.)

**Every FastCGI record:**

```
 ┌─────────┬──────┬───────────┬───────────────┬────────┬──────────┐
 │ version │ type │ requestId │ contentLength │ padLen │ reserved │   8-byte header
 │  1 B    │ 1 B  │   2 B     │     2 B       │  1 B   │   1 B    │
 └─────────┴──────┴───────────┴───────────────┴────────┴──────────┘
                     → contentData[contentLength] → padding[padLen]
```

**Decoding the slide's bytes:**

```
>> BEGIN_REQUEST
   01 01 00 01 00 08 00 00     <- the first 8 bytes of every FastCGI connection on earth
   │  │  └─┬─┘ └─┬─┘ │  └ reserved
   │  │    │     │   └─── padLen = 0
   │  │    │     └─────── contentLength = 8 (the 8-byte BEGIN_REQUEST body follows)
   │  │    └───────────── requestId = 1
   │  └────────────────── type = 1 = FCGI_BEGIN_REQUEST
   └───────────────────── version = 1

>> PARAMS     <- EMPTY = end of stream
   01 04 00 01 00 00 00 00
      │           └─┬─┘
      │             └──── contentLength = 0  ← an EMPTY record
      └────────────────── type = 4 = FCGI_PARAMS
```

🧠 Record types worth knowing: `1 BEGIN_REQUEST`, `2 ABORT_REQUEST`, `3 END_REQUEST`, `4 PARAMS`, `5 STDIN`, `6 STDOUT`, `7 STDERR`. The 8-byte BEGIN_REQUEST body holds the **role** (1 = RESPONDER) and a **flags** byte (`FCGI_KEEP_CONN` = keep the socket open for the next request).

**The empty record IS the delimiter.**
- An **empty PARAMS** record means "no more environment variables".
- An **empty STDIN** record means "no more body".
- There is **no other end-of-stream signal. Forget it and the responder waits forever.**

> **A length inside every header, and an empty record between streams. Session 2's closing idea, at two levels, in one twenty-page spec.**

🧠 A full request from nginx to php-fpm looks like:

```
 nginx ──▶ BEGIN_REQUEST (role=RESPONDER)
       ──▶ PARAMS  "SCRIPT_FILENAME=/var/www/index.php", "REQUEST_METHOD=GET", "REQUEST_URI=/..."
       ──▶ PARAMS  (empty)          ← end of params
       ──▶ STDIN   (request body, if any)
       ──▶ STDIN   (empty)          ← end of body
 php-fpm ◀── STDOUT "Content-Type: text/html\r\n\r\n<html>..."
         ◀── STDOUT (empty)
         ◀── END_REQUEST (app status, protocol status)
```

The PARAMS are the **same environment variables CGI used in 1993** (`REQUEST_METHOD`, `QUERY_STRING`, `REQUEST_URI`...), just sent over a socket instead of via `exec()`. That's "CGI without the fork".

**Source:** `ngx_http_fastcgi_module.c` → `ngx_http_fastcgi_process_record()` ~L2742 · `..._create_request()` ~L858 · repo: `06-fastcgi/`.

### 4.8 The ladder: seven servers, in code you can hold in your head (slide 33)

| dir | design | historical match | what you'll see |
|---|---|---|---|
| `01-fork` | `fork()` per client | Apache 1.3, 1995 | `ps` shows **one process per in-flight request** |
| `02-thread` | pthread per client | Apache 2.0 worker, 1999 | prints its own **8 MB stack size** at startup |
| `03-select` | one thread, `select()` | …and the **1024 wall** | **dies at fd 1024**, and `ulimit` does not help |
| `04-epoll` | one thread, `epoll()` | nginx, 2004 | **5,000 connections, nothing happens** (it just works) |
| `05-sendfile` | read vs mmap vs sendfile | the copies, counted | **2048 syscalls become 1** |
| `06-fastcgi` | the wire protocol, both ends | CGI without the fork | nginx → **your own 200-line responder** |
| `07-nginx` | one `nginx.conf`, five demos | all of it, for real | routing, cache, weights, fallbacks |

**About 1,200 lines total. nginx is 200,000, and the difference is every error path we ignored.** That's the honest reason nginx is big — not complexity for its own sake.

```
  cost per connection, going down the ladder
  01-fork    ████████████████████  a whole process
  02-thread  ██████████            a thread + 8 MB stack reservation
  03-select  ██                    a bit in a 1024-bit bitmap (hard cap!)
  04-epoll   █                     a few hundred bytes in a kernel list
```

---

## 5 · HANDS-ON 🛠️

> 🧠 **You're on macOS.** `epoll`, `sendfile` (Linux flavour), `strace`, `ss` and `ps -C` are Linux things. Either run the demos in a Linux VM/container (`docker run -it --rm -p 8080-8090:8080-8090 ubuntu`), or use the Mac equivalents noted below. The repo `nginx-from-scratch` is the instructor's; the commands from the slides are marked **[slide]**, the rest are my additions.

### 5.1 Install and look at the process model

```bash
# Linux
sudo apt install nginx            # or: brew install nginx (macOS)
nginx -V                          # build flags: which modules + event method were compiled in
nginx -t                          # test config syntax
nginx -T                          # dump the full effective config

ps -o pid,ppid,args -C nginx      # [slide] Linux
ps -axo pid,ppid,command | grep [n]ginx   # macOS equivalent
```
Expect: one `master process` with PPID 1 (or your shell), N `worker process` children, plus `cache manager`/`cache loader` if a `proxy_cache_path` is configured. Now hammer it with requests: **the count stays the same.**

Compare: run the repo's `01-fork/forkd`, open several connections (`nc localhost <port>` in a few terminals), and re-run `ps` — **one new process per connection**.

### 5.2 See the thread stack size and the fd limits

```bash
ulimit -s        # stack size in KB → 8192 on most Linux = 8 MB
ulimit -n        # max open fds per process
grep -r "define.*FD_SETSIZE" /usr/include/ 2>/dev/null | head    # → 1024
```

### 5.3 Break `select`, watch `epoll` shrug (homework 1 preview)

```bash
# terminal 1
ulimit -n 65536
./03-select/selectd <port>       # (name as in the repo)

# terminal 2: open 1,100 idle connections
python3 - <<'EOF'
import socket, time
PORT = 8003          # set to whatever port 03-select listens on
s = []
for i in range(1100):
    c = socket.create_connection(("127.0.0.1", PORT)); s.append(c)
    if i % 100 == 0: print(i)
time.sleep(600)
EOF
```
Expect `03-select` to give up / crash near fd 1024 despite `ulimit -n 65536`. Repeat with 5,000 against `04-epoll`: **nothing happens**.
Count established connections: `ss -tn state established | wc -l` (Linux) or `netstat -an | grep ESTABLISHED | wc -l` (macOS).

### 5.4 Count the sendfile syscalls

```bash
./sendfiled 8005 www/big.bin                 # [slide] 64 MB file
curl -s localhost:8005/ -o /dev/null         # receiver

# Linux: count syscalls by type
strace -c -f -p $(pgrep sendfiled)
# or for nginx workers:
sudo strace -f -e trace=sendfile,sendfile64,read,write,writev -p $(pgrep -d, -f "nginx: worker")
```
Expect: `mode=read` ≈ 2048 read/write calls; `mode=sendfile` ≈ 1 `sendfile`. Wall time roughly similar on loopback (see §2.9 — that's the lesson). macOS: `sudo dtruss -p <pid>` is the rough equivalent of strace.

### 5.5 🧠 A minimal `nginx.conf` reproducing the slide demos

Use this if you don't have the repo. Start two tiny backends first:
```bash
mkdir -p /tmp/a /tmp/b && echo "I am 8001" > /tmp/a/index.html && echo "I am 8004" > /tmp/b/index.html
(cd /tmp/a && python3 -m http.server 8001) &
(cd /tmp/b && python3 -m http.server 8004) &
```

```nginx
worker_processes auto;
events { worker_connections 1024; }

http {
    log_format up '$remote_addr "$request" $status upstream=$upstream_addr cache=$upstream_cache_status';
    access_log logs/access.log up;
    sendfile on;

    proxy_cache_path temp/cache levels=1:2 keys_zone=demo:10m
                     max_size=100m inactive=60s use_temp_path=off;

    upstream app {
        server 127.0.0.1:8004 weight=3;
        server 127.0.0.1:8001 weight=1;
    }

    # ---- :8081 location precedence ----
    server {
        listen 8081;
        location = /exact     { return 200 "1 exact\n"; }
        location ^~ /images/  { return 200 "2 ^~ prefix\n"; }
        location ~ \.php$     { return 200 "3 regex\n"; }
        location /            { return 200 "4 plain prefix\n"; }
    }

    # ---- :8082 cache + weighted upstream ----
    server {
        listen 8082;
        location / {
            proxy_pass http://app;
            proxy_cache demo;
            proxy_cache_valid 200 10s;
            proxy_cache_lock on;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            add_header X-Cache-Status $upstream_cache_status;
        }
    }

    # ---- :8084 try_files fallback ----
    server {
        listen 8084;
        root /tmp/a;
        location / {
            try_files $uri $uri/ /index.html?$args;
        }
        location /maybe/ {
            try_files $uri @app;
        }
        location @app {
            proxy_pass http://app;
        }
    }
}
```

Run it from a working directory: `mkdir -p logs temp && nginx -p "$PWD" -c "$PWD/nginx.conf"` (stop: `nginx -p "$PWD" -s stop`).

### 5.6 Location precedence — predict first!

```bash
curl localhost:8081/exact          # → 1 exact
curl localhost:8081/images/a.php   # → 2 ^~ prefix   (NOT the regex!)
curl localhost:8081/other/a.php    # → 3 regex
curl localhost:8081/anything       # → 4 plain prefix
```

### 5.7 Cache MISS → HIT → EXPIRED

```bash
curl -si localhost:8082/ | grep X-Cache    # [slide] → MISS
curl -si localhost:8082/ | grep X-Cache    # [slide] → HIT
sleep 11
curl -si localhost:8082/ | grep X-Cache    # → EXPIRED (fetched again, >10s valid)
ls -R temp/cache | head                    # see the levels=1:2 directory layout
```
Then kill both Python backends and request again: with `use_stale error` you still get a page (`STALE`) instead of a 502.

### 5.8 Weighted round robin, measured

```bash
for i in $(seq 24); do curl -s "localhost:8082/x-$i-$RANDOM" -o /dev/null; done   # [slide]
grep -o 'upstream=[0-9.:]*' logs/access.log | sort | uniq -c                       # [slide]
#   6 upstream=127.0.0.1:8001
#  18 upstream=127.0.0.1:8004      ← 3:1
```
(Python's `http.server` returns 404 for `/x-...`, which is fine — the request still reached the upstream, and 404s aren't cached by `proxy_cache_valid 200`.)

### 5.9 try_files

```bash
curl -i localhost:8084/index.html          # real file → served
curl -i localhost:8084/dashboard/settings  # not a file → falls back to /index.html (200, "I am 8001")
curl -i localhost:8084/@app                # → 404: named locations aren't reachable by URL
curl -i localhost:8084/maybe/xyz           # not on disk → proxied to the upstream
```

### 5.10 Watch FastCGI on the wire (homework 3 preview)

```bash
sudo tcpdump -i lo -X 'tcp port 9000'      # Linux loopback; macOS: -i lo0
```
In the hex dump, after the TCP handshake, the first payload bytes of the connection should be `0101 0001 0008 0000` — FCGI_BEGIN_REQUEST. Later look for `0104 0001 0000 0000` — the empty PARAMS.

### 5.11 Find code in the nginx source

```bash
git clone --depth 1 https://github.com/nginx/nginx.git && cd nginx      # [slide]
git log -1 --oneline                                                     # [slide]
grep -rn "^ngx_epoll_process_events" src/                                # [slide] definition only
grep -rn "^ngx_http_core_find_static_location" src/
grep -rn "NGX_HTTP_.*_PHASE\].handlers" src/http/modules/               # [slide] phase map
grep -n "ngx_accept_disabled" src/event/*.c
wc -l src/core/ngx_queue.h src/http/modules/ngx_http_try_files_module.c
```

---

## 6 · HOMEWORK 📝 (slide 34)

### HW1 — Break `select`, then fix it
**Task:** Flood `03-select` at **1,100** connections. Find the line where it gives up. **Delete the `FD_SETSIZE` guard**, run it again, capture the **real crash**. Then explain **in two sentences** why `ulimit -n 65536` does not help — and run the same flood at **5,000** against `04-epoll`.

**Hints:**
- The guard is probably an `if (fd >= FD_SETSIZE) { close(fd); ... }` near `accept()`. `grep -n FD_SETSIZE` in the source.
- Without the guard, `FD_SET(fd, &set)` with `fd ≥ 1024` writes beyond the `fd_set` struct. What lives next to it on the stack? Expect a stack smash / fortify abort / segfault, or worse, silent corruption. Compile with `-g` and run under `gdb` or with `-fsanitize=address` to see the exact overflow.
- Two-sentence answer shape: *what `ulimit` controls* (kernel's per-process fd limit) vs *what `select` is limited by* (the compile-time width of the `fd_set` bitmap).

### HW2 — Find the sendfile decision
**Task:** Start from `sendfile on;` in a config and get to `sendfile(2)` in `ngx_linux_sendfile_chain.c`, writing down **every function you pass through**. Then add `gzip on;`, re-run, and show with **strace** that sendfile is gone. **Which filter took it away?**

**Hints:**
- Start by grepping for the directive: `grep -rn '"sendfile"' src/http/` → find which config field it sets (in `ngx_http_core_module.c`). Then find where that field is read (`grep -rn "clcf->sendfile"`).
- Follow the connection's `send_chain` function pointer: the output goes through the **filter chain** (header filters, body filters, `ngx_http_write_filter`) and eventually `c->send_chain(...)` → on Linux `ngx_linux_sendfile_chain` → `ngx_linux_sendfile`.
- Look for the buffer flag that says "this data lives in a file, not in memory" (`in_file` / `ngx_buf_in_memory`). What happens to that flag when a filter needs to read the bytes? Which body filter must read every byte to compress it?
- For strace: `strace -f -e trace=sendfile,write,writev -p <worker pid>`, request a big text file with `curl -H 'Accept-Encoding: gzip'` (gzip only kicks in if the client asks).

### HW3 — Speak FastCGI to something real
**Task:** Point `06-fastcgi/fcgi_client` at a real **php-fpm** and get a PHP page back. Then **tcpdump loopback** while nginx does the same request and find **FCGI_BEGIN_REQUEST by hand**. **Diff nginx's PARAMS against ours** — what does it send that we don't, and why?

**Hints:**
- Install php-fpm, make it listen on `127.0.0.1:9000` (`listen = 127.0.0.1:9000` in the pool config) — TCP is easier to tcpdump than a Unix socket.
- php-fpm needs at least `SCRIPT_FILENAME` and `REQUEST_METHOD`; otherwise you get "File not found." or an empty response.
- nginx's params come from `fastcgi_params` / `fastcgi.conf` (look at those files: `QUERY_STRING`, `CONTENT_TYPE`, `CONTENT_LENGTH`, `SERVER_NAME`, `REMOTE_ADDR`, `HTTPS`, `REDIRECT_STATUS`, ... plus every request header as `HTTP_*`). For each extra one ask: "which PHP feature reads this?" (`$_SERVER`, `$_GET`, security checks).
- Name-value pairs in PARAMS are length-prefixed too: 1 byte if the length < 128, else 4 bytes with the top bit set. Decode a couple by hand.

### HW4 — Make round robin lie to you
**Task:** Put a **10-second sleep** in one upstream, leave the weights at **3:1**, send **100 requests**. Compare **p99** latency under round robin and under `least_conn`. Compute the concurrency **Little's Law** predicts, then check it against `ss -tn | wc -l`.

**Hints:**
- 🧠 **Little's Law:** `L = λ × W` — average number of requests in the system = arrival rate × average time each spends in the system. E.g. arrivals of 10 req/s, and a quarter (or three quarters, depending which server you slowed) of them taking 10 s → work out the expected number concurrently open to the slow server.
- Send requests **concurrently** (e.g. `xargs -P 20`, `ab -n 100 -c 20`, `hey`, or a bash loop with `&`), otherwise you just measure a queue of one.
- **p99** = the latency 99% of requests are faster than. Round robin keeps sending its fixed share to the slow server no matter how backed-up it is ("round robin lies": it balances *requests*, not *load*). `least_conn` notices the slow server has many open connections and routes around it.
- Make sure the cache doesn't hide the effect: unique URLs, or turn `proxy_cache` off for this test.

### HW5 — Read one module end to end
**Task:** `ngx_http_try_files_module.c`, **390 lines**. Write **one page** on what happens to `try_files $uri $uri/ /index.html?$args` for a URI that **does not exist** — every **stat**, every **allocation**, where the URI gets **rewritten**. Then explain **why the last argument is never tested**.

**Hints:**
- Find the handler (`ngx_http_try_files_handler`) and the loop over `tf` entries. Notice how it detects the last entry.
- Allocations come from `r->pool` (§4.3) — note which ones (building the path from `root` + `$uri`).
- Filesystem checks go through nginx's open-file-cache helper (`ngx_open_cached_file`) — that's where the `stat()` happens. `$uri/` (trailing slash) means "test for a directory".
- For the last entry, find the branch that sets the new URI (and `$args`) and calls `ngx_http_internal_redirect`. Why test it? An internal redirect re-runs location matching (§3.4), so the target is handled by whatever location matches it — testing first would be both pointless and wrong (it might not be a file at all; e.g. `@app` or `=404`).

---

## 7 · FOUR IDEAS TO LEAVE WITH 💡 (slide 35)

1. **Ask the other question.** Apache spent a decade making the per-connection worker cheaper. nginx asked why there was one. When a whole industry is optimising the same thing, the opportunity is usually **one level up**.
2. **Check whether the constraint still exists.** Thread-per-connection died because a thread cost **8 MB and a context switch**. **Goroutines and virtual threads brought it back** when those numbers changed. Know which of your rules are **laws** and which are **2004**.
   🧠 A Go goroutine starts with a ~2–8 KB stack that grows as needed, and Java 21 *virtual threads* are scheduled by the JVM onto a few OS threads — under the hood, both sit on an epoll/kqueue event loop. You get to *write* blocking-style code while the runtime does what nginx does.
3. **The data structure is the documentation.** Prefixes are a **tree**, so **longest** wins. Regexes are an **array**, so **first** wins. The precedence rules aren't trivia — they're the only rules those structures could have.
4. **Amortise the setup.** **Prefork, pre-thread, servlets, connection pools, FastCGI, upstream keepalive.** Six names for one move: **take the expensive setup off the request path and pay it once.**
   🧠 *Upstream keepalive* = `keepalive 32;` inside an `upstream` block: nginx reuses TCP connections to the app servers instead of opening a new one (handshake!) per request.

### Next session (slide 36)
**"Where did all the state go?"** We put a reverse proxy in front of everything and made the app servers interchangeable. Next: **sessions, sticky routing, TLS termination, shared caches, and what "stateless" actually costs.**

---

## 8 · CHEAT-SHEET 📋

| term | one-line meaning |
|---|---|
| **CERN httpd** | first web server, Berners-Lee, NeXT, 1990 |
| **NCSA Mosaic** | 1993 **browser** (Andreessen), not a server |
| **NCSA HTTPd** | Rob McCool's 1993 server; invented CGI; Apache's ancestor |
| **Apache** | 1995, "a patchy server"; #1 from 1996 for ~20 years |
| **MPM** | Apache's concurrency strategy: prefork (1.3) → worker (2.0) → event (2.4) |
| **CGI** | fork+exec a program per request; env vars in, stdout out |
| **Servlet** | Java object created once, `service()` called per request on a pooled thread |
| **J2EE** | 1999; multi-tier app server: web → EJB → DB |
| **Connection pool** | open DB connections once, lend them out per request |
| **C10K** | 10,000 concurrent connections on one box; named by Dan Kegel, 1999 |
| **8 MB** | default Linux thread stack; ×10,000 = 80 GB address space |
| **FD_SETSIZE = 1024** | compile-time width of select's bitmap; `ulimit` can't change it |
| **nginx** | Igor Sysoev, Rambler; started 2002; public 4 Oct 2004; 1.0.0 April 2011 |
| **master process** | binds ports, reads config, forks, handles signals; never touches a connection |
| **worker process** | one per core, one event loop each, does all the work |
| **cache manager / loader** | evicts LRU to stay under `max_size` / warms the index at startup |
| **event loop** | sleep in `epoll_wait`, handle ready events, repeat; never block |
| **timer as argument** | nginx sleeps until the nearest timer; never polls for timeouts |
| **posted events** | handlers queued after `epoll_wait`, drained afterwards (accepts first) |
| **`ngx_accept_disabled`** | `connection_n/8 − free`; > 0 ⇒ back off from accepting (using > 7/8 slots) |
| **thundering herd** | all sleepers woken for one event; fixed by `EPOLLEXCLUSIVE` / `SO_REUSEPORT` |
| **accept_mutex** | nginx's userspace herd fix; default off since 1.11.3 |
| **select** | O(watched) per call; watch list rebuilt in userspace each call; 1024 cap |
| **epoll** | O(active); watch list in kernel via `epoll_ctl`; Linux only (kqueue on BSD/macOS) |
| **stale event / instance bit** | low pointer bit used as generation flag to drop events for recycled fds |
| **sendfile** | file → socket inside the kernel; 2 DMA copies, no user-space copy, 1 syscall |
| **gzip vs sendfile** | touching bytes (gzip/TLS/templating) needs user space → no sendfile; kTLS helps TLS |
| **loopback benchmark lesson** | no NIC, no DMA, memory-bound → same wall time; the win is CPU headroom |
| **forward proxy** | next to clients (Squid, 1996); client configures it |
| **reverse proxy** | next to servers; route, balance, fallback, TLS termination, absorb slow clients |
| **`keys_zone`** | shared-memory index (rbtree + LRU), ~8,000 keys/MB; not the data |
| **`levels=1:2`** | two-level cache directory hashing |
| **`max_size`** | disk ceiling, enforced by cache manager from LRU tail |
| **`inactive`** | delete if not read for this long, regardless of freshness |
| **`proxy_cache_valid`** | freshness TTL; after it, content is stale |
| **`proxy_cache_lock`** | anti-stampede: one request refills, others wait; off by default |
| **`proxy_cache_use_stale`** | serve stale copy on error/timeout/5xx; `updating` = serve stale while one refreshes |
| **`X-Cache-Status`** | `add_header X-Cache-Status $upstream_cache_status;` — always add it |
| **`location =`** | exact; wins immediately |
| **`location ^~`** | longest prefix + skip regexes |
| **`location ~` / `~*`** | regex (case-sens / insens); **first** match in file order |
| **`location /prefix`** | longest prefix; used only if no regex matched |
| **prefix tree vs regex array** | why longest-prefix wins vs first-regex wins; prefixes cheap, regexes linear |
| **no `.htaccess`** | nginx never `stat()`s up the tree per request |
| **smooth WRR** | `cur += w; pick max; winner −= total` → interleaved `a b a a`, not `a a a b` |
| **`effective_weight`** | lowered on failures, +1 per pick → passive health check |
| **`least_conn`** | pick server with fewest active connections |
| **`try_files`** | test each candidate on disk; last arg is the untested fallback |
| **named location `@x`** | reachable only from `try_files` / `error_page`, never a URL |
| **`ngx_queue.h`** | intrusive doubly-linked list; link→data by pointer subtraction |
| **`ngx_palloc.h`** | per-request memory pool; destroy all at once; no `free()` on hot path |
| **`^name(` grep** | finds function definitions only (return type on its own line) |
| **resumable parser** | byte-at-a-time state machine; `r->state` survives across TCP segments |
| **11 phases** | POST_READ … CONTENT → LOG; `NGX_OK` from a checker = suspended |
| **FastCGI record** | 8-byte header (version, type, requestId, contentLength, padLen, reserved) + data + padding |
| **empty record** | end of a stream (PARAMS / STDIN); forget it → responder waits forever |
| **amortise the setup** | prefork, pre-thread, servlets, pools, FastCGI, upstream keepalive |

---

## 9 · SELF-CHECK QUESTIONS ✅

**1. Who wrote the first web server, what was it called, and what was Mosaic?**
<details><summary>Answer</summary>

Tim Berners-Lee wrote **CERN httpd** on a NeXT in 1990. **Mosaic** (NCSA, 1993) was a **browser** (Andreessen's). Apache descends from **NCSA HTTPd** (Rob McCool), a different NCSA program — the one that invented CGI.
</details>

**2. Prefork, worker and event MPMs all answered the same question. What was the question, and what different question did nginx ask?**
<details><summary>Answer</summary>

Apache asked *"how do I make the per-connection worker cheaper?"* (process → thread → thread only when there's work). nginx asked *"why is there a worker per connection at all?"* — a connection only needs a few hundred bytes of state and a place in a list until it has something to say, so one event loop per core can hold thousands.
</details>

**3. Why does `ulimit -n 65536` not let a `select()`-based server handle 2,000 connections, while it does for an epoll server?**
<details><summary>Answer</summary>

`select()` uses an `fd_set` bitmap whose width, `FD_SETSIZE = 1024`, is fixed at compile time in a header; fds ≥ 1024 have no bit, and setting one writes out of bounds. `ulimit -n` only changes how many fds the kernel will hand out. epoll keeps its watch list in the kernel with no bitmap, so its only limit is `ulimit -n`.
</details>

**4. With 10,000 idle connections and 10 active ones, roughly how much work does each of select and epoll do per loop iteration, and why?**
<details><summary>Answer</summary>

select: ~30,000 units — O(n) to build the set, O(n) for the kernel to scan it, O(n) for you to scan the result. epoll: ~10 — the kernel already holds the watch list and returns only the ready events. The difference comes from who owns the watch list.
</details>

**5. Explain the stale-event bug and how nginx fixes it without using extra memory.**
<details><summary>Answer</summary>

`epoll_wait` returns a batch. Handling an early event can close a connection that a later event in the same batch refers to; the fd can then be reused by a new client, which would wrongly receive the old event. nginx stores a pointer to the connection struct in the epoll data. Because structs are aligned, the pointer's low bit is always 0, so nginx uses it as an `instance` (generation) bit and flips it every time the struct is reused. On delivery it compares the bit with the connection's current `instance` (and checks `fd == -1`); a mismatch means the event is stale and is skipped.
</details>

**6. The sendfile benchmark over loopback shows `read` 37.4 ms, `mmap` 23.3 ms, `sendfile` 33.2 ms. Does that mean sendfile is useless? What does it actually buy you?**
<details><summary>Answer</summary>

No. On loopback there is no NIC and no DMA; the kernel memcpy's into the receiver's buffer whatever mode you use, and curl competes for the same CPU, so the run is memory-bandwidth bound and all modes end up about the same. What's real: 2048 syscalls become 1, and 64 MB never enters the process's address space. On real hardware under concurrency, that saved CPU goes to serving other requests (headroom), rather than making one transfer faster. The general lesson: a benchmark that doesn't model the bottleneck measures the wrong thing.
</details>

**7. Why do `gzip on` and `sendfile on` conflict? What exists to rescue the TLS case?**
<details><summary>Answer</summary>

sendfile copies file data to the socket entirely inside the kernel. Compressing (or encrypting, or templating) means the bytes have to be read and changed, so they must come up into user space, and sendfile can't be used for that response. **kTLS** moves TLS encryption into the kernel, so encrypted responses can still go out with sendfile.
</details>

**8. Given the four locations on slide 22, which one handles `/images/a.php`, and why does about half the class get it wrong?**
<details><summary>Answer</summary>

Location **2** (`^~ /images/`). The exact match doesn't apply. The longest matching prefix is `/images/`, and its `^~` modifier tells nginx to skip the regex search, so `\.php$` is never checked. People get it wrong because they expect a regex to beat a prefix, or they think file order decides.
</details>

**9. Explain the location precedence rules using the data structures, not memorisation.**
<details><summary>Answer</summary>

Prefix locations are put into a search tree when the config is parsed. Walking a tree segment by segment naturally ends at the deepest, i.e. **longest**, match, and it's cheap however many prefixes there are. Regex locations sit in a plain array that is scanned in order on every request, so the natural result is the **first** match, and the cost grows with each regex you add. Exact matches and `^~` are flags that let the tree walk stop early. "The data structure is the documentation."
</details>

**10. `proxy_cache_valid 200 10s;` and `inactive=60s`. A page is requested once and then never again. What is its state at t = 5 s, 30 s and 70 s?**
<details><summary>Answer</summary>

- 5 s: **fresh**. A request would be a HIT.
- 30 s: **stale** (older than 10 s) but **still on disk and in the index**. A request would go to the origin (EXPIRED), or could be served STALE if `proxy_cache_use_stale` conditions apply.
- 70 s: **gone**. It hasn't been read for more than 60 s, so the cache manager deleted it regardless of TTL.
</details>

**11. With weights 3:1, what sequence does naive weighted round robin produce, and what does nginx's smooth WRR produce? Why does it matter?**
<details><summary>Answer</summary>

Naive: `a a a b`, three requests in a row to `a`. Smooth: `a b a a` (an interleaved rotation of the same cycle). Each round, every peer adds its weight to `current_weight`, the highest one is picked, and the total is subtracted from the winner. The ratio is the same, but there are no bursts hitting one origin. Measured result: 24 requests → 18 : 6.
</details>

**12. In `try_files $uri $uri/ /index.html?$args;`, why is the last argument never tested, and how does this one line make SPAs and WordPress "pretty URLs" work?**
<details><summary>Answer</summary>

The last argument is the **fallback**, not a candidate. If nothing before it exists on disk, nginx does an **internal redirect** to that URI, and location matching runs again for it. Checking it first would be pointless, and wrong for targets like `@app` or `=404` that aren't files. Pretty URLs work because any path that isn't a real file is handed to a single entry point (`index.html` or `index.php`), and the application reads `REQUEST_URI` to do its own routing.
</details>

**13. Why must nginx's HTTP parser be a resumable state machine, and what does `return` mean inside `ngx_http_core_run_phases`?**
<details><summary>Answer</summary>

On a non-blocking server the request line can arrive split across several TCP segments, and the parser can't block waiting for the rest. So it handles one byte at a time and saves `r->state` in the request, which lets it carry on mid-token when more bytes arrive. Likewise, when a phase checker returns `NGX_OK`, the `while` loop returns, but that means **suspended**, not done. The request is waiting on an event, and a later loop iteration re-enters the same `while` at the saved phase: a coroutine written by hand in C.
</details>

**14. In FastCGI, how does the receiver know where the PARAMS stream ends? What happens if the client forgets?**
<details><summary>Answer</summary>

Each record's 8-byte header carries `contentLength`, which frames individual records by length. The **end of a stream** is marked by an **empty record** of that type: PARAMS with `contentLength = 0` means no more variables, and an empty STDIN means no more body. That's length framing and delimiter framing at two levels. If the empty record is never sent, the responder (such as php-fpm) waits forever.
</details>

**15. "Six names for one move" — list them and name the move.**
<details><summary>Answer</summary>

Prefork, pre-thread, servlets, connection pools, FastCGI, upstream keepalive. The move is **amortising the setup**: take the expensive setup (process creation, thread creation, class loading, DB handshake, interpreter startup, TCP handshake) off the request path and pay for it once.
</details>
