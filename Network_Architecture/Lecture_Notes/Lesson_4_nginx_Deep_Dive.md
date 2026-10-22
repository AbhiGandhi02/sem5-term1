# 📓 nginx, and the code that explains it
### *CN at Scaler — Lesson 4* · "it did not make the worker cheaper. It deleted the worker."

> `nginx 1.31.5` · source: `github.com/nginx/nginx` · companion repo: `nginx-from-scratch`
> **history → what nginx changed → the features you configure → a walk through the source**

> 💡 **Simple words mein:** Purane servers har customer ke liye naya waiter rakhte the (fork/thread). nginx har CPU core pe ek smart waiter rakhta hai jo sirf haath uthane walon ko serve karta hai (event loop).

---

## 1 · HISTORY, COMPRESSED 📜

### 1.1 "Remember the evolution. You can forget the dates." (slide 3)

Names and dates matter less than the **chronology**. Every design in this lecture was someone's answer to a constraint that no longer exists. Knowing **which** constraint is how you tell a **law** from a **habit**. *"Understanding history is how you get to create a different timeline"* — sometimes you look at the same limitation and fix it a completely different way.

### 1.2 A correction worth making (slide 4)

| the myth you often hear | what actually happened |
|---|---|
| "Berners-Lee wrote the first server. It was called Mosaic. Mosaic became Apache." | Berners-Lee's server was **CERN httpd**, on a **NeXT**, **1990**. |
| | **Mosaic** (NCSA, 1993) was a **browser** (Marc Andreessen's), not a server. |
| | **Apache** descends from **NCSA HTTPd**, a *different* NCSA program. |

Three separate programs, two of them at NCSA, one of them a browser. Getting it right is "the difference between a story and a lineage".

### 1.3 The actual lineage (slide 5)

```
 1990     CERN httpd ........ Berners-Lee, on a NeXT. The first web server.
 1993     NCSA Mosaic ....... a BROWSER. Inline images. The web goes mainstream.
 1993     NCSA HTTPd ........ Rob McCool. A SERVER — and it invents CGI.
 1994     McCool leaves ..... development stalls. Webmasters trade patches by mail.
 1995     Apache ............ that patch set, continued as a project. "A patchy server."
 1996     Apache #1 ......... passes NCSA HTTPd. Stays #1 for twenty years.
 1996–99  IIS, then the boom  Microsoft fights for single-digit share. lighttpd & friends arrive.
```

### 1.4 Apache spent a decade trying to outrun `fork()` (slide 6)

| version | MPM | idea | remaining cost |
|---|---|---|---|
| **1.3** | **prefork** | Fork a pool of processes *at startup* instead of per request. The setup cost moves **off the request path**. | One process still **parks on one connection**. |
| **2.0** | **worker** | Processes each running many **threads**. "8 MB of stack becomes 8 MB shared plus a stack each." | 1999–2002: **pthreads on Linux are new and buggy**, and everyone is learning what *reentrant* means. |
| **2.4** | **event** | A **listener thread** parks idle **keep-alives** on **epoll** and hands a worker thread a connection **only when it has actual work**. Default on Unix since 2.4. | — |

> **Punchline:** *"Three architectures, one question: how do I make the per-connection worker cheaper? Every answer was a good answer. None of them was a different question."*

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

- **J2EE (1999)** made the **multi-layer application server** the default shape: **web tier → EJB tier → database**.
- A **servlet** is instantiated **once** and serves **every** request on a thread from a **pool**.
- **Connection pools** do the same thing for the **database handshake**.
- Same move as prefork. Same move as FastCGI: **take the expensive setup off the request path and pay it once.**

> *"Our primary goal was to avoid creating a new process. Everything else was consequence."*

### 1.6 So here is 2002 — the C10K problem (slide 8)

| number | meaning |
|---|---|
| **10,000** | concurrent connections on one box. **Dan Kegel** named this the **C10K problem** in **1999**. |
| **8 MB** | stack per thread, the Linux default. **10,000 threads = 80 GB of address space.** |
| **1024** | descriptors, **hard-coded as `FD_SETSIZE`** in a header. **`ulimit` does not move it.** |

Hardware was fast enough to serve 10,000 people from one machine; the software wasn't. The limits: a **per-connection stack**, a **per-connection scheduler entry**, and **a 1983 API with a bitmap in it**.

> *"Everyone is optimising the worker. Somebody is about to delete it."*

---

## 2 · WHAT NGINX CHANGED 🚀

### 2.1 Igor Sysoev, Rambler, 2002 (slide 9)

- **Rambler** was one of Russia's largest portals. Its Apache servers were **falling over under connection volume, not request volume**.
- Sysoev started writing nginx in **2002** to solve **C10K specifically**.
- **First public release: 4 October 2004**, chosen for the anniversary of **Sputnik**.
- **1.0.0 didn't arrive until April 2011.** Seven years of shipping before the version number said "ready".

> He did not set out to write a faster Apache. He set out to **serve ten thousand mostly-idle connections from one process, and let something else generate the dynamic content.** **That second half is why nginx is a reverse proxy and not an application server.**

### 2.2 The different question (slide 10)

> **Apache asked:** how do I make the worker cheaper? **nginx asked:** why is there a worker per connection at all?

A connection is not a thing that needs a thread. It needs **a few hundred bytes of state and a place in a list** until it has something to say. When it does, some CPU handles it, finishes, and moves on.

> **One process per core. One event loop each. Ten thousand connections, and none of them owns anything.**

> 💡 **Simple words mein:** Apache soch raha tha "waiter sasta kaise karein?". nginx ne poocha "har table pe waiter chahiye hi kyun?" Connection ko bas ek bank token chahiye; jab number aaye, tab serve karo.

### 2.3 One process per core, and a master that never touches a connection (slide 11)

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

```
$ ps -o pid,ppid,args -C nginx
2663     1  nginx: master process
2665  2663  nginx: worker process
2666  2663  nginx: worker process
2667  2663  nginx: cache manager process
```

> **The count does not move under load.** Run the same command against `01-fork/forkd` and watch it move. *Same `fork()` syscall — the only difference is how often you call it.*

**Source:** `ngx_process_cycle.c` → `ngx_master_process_cycle()` ~L74 · `ngx_start_worker_processes()` ~L358 · `ngx_process.c` → `ngx_spawn_process()` ~L87.

### 2.4 All of nginx, in six lines (slide 12)

```c
timer = ngx_event_find_timer();
ngx_process_events(cycle, timer, flags);                 /* epoll_wait() */
ngx_event_process_posted(cycle, &ngx_posted_accept_events);
ngx_shmtx_unlock(&ngx_accept_mutex);
ngx_event_expire_timers();
ngx_event_process_posted(cycle, &ngx_posted_events);
```

- **Line 2 is where a worker spends 99.9% of its life: asleep** (in `epoll_wait`).
- **The timer is an ARGUMENT to the sleep. nginx never polls for timeouts**.
- **Handlers do not run inside the poll; they are posted (queued) and drained after**.
- **Everything else in `src/http/` is a callback reached from line 2 or line 6.** "Read these six lines until they are boring."

**Source:** `src/event/ngx_event.c` → `ngx_process_events_and_timers()` ~L195 · compare: `04-epoll/epolld.c`.

### 2.5 Load balancing between workers, in one line of arithmetic (slide 13)

```c
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
```

**Read it as English:** *once I am using more than seven-eighths of my connection slots, stop competing for new connections and let my siblings take them.*

| | |
|---|---|
| **No coordinator** | No shared counter, no leader, no heartbeat. Each worker decides **alone**, from a number it already has. |
| **`accept_mutex` is now off** | Default **off since 1.11.3**. `EPOLLEXCLUSIVE` (Linux 4.5+) and `SO_REUSEPORT` let the **kernel** do the fan-out instead. |
| **A userspace fix for a kernel problem** | The **thundering herd** was real in 2004. The kernel fixed it. The workaround is now dead weight — and nginx removed it. |

**Source:** `src/event/ngx_event_accept.c` ~L139 · `src/event/ngx_event.c` `ngx_process_events_and_timers()` ~L224 (`ngx_trylock_accept_mutex`).

### 2.6 `select` is O(watched). `epoll` is O(active). (slide 14)

| | `select()` | `epoll` |
|---|---|---|
| **Where the watch list lives** | In your process. **Rebuilt every call.** | In the **kernel**. Edited with `epoll_ctl()`. |
| **Cost per iteration** | O(n) to build + O(n) in kernel + O(n) to scan | **O(number of ready events)** |
| **10,000 idle, 10 active** | **30,000** units of work to find 10 events | **10** |
| **Descriptor ceiling** | `FD_SETSIZE = 1024`, compile-time, in a header | `ulimit -n` |
| **What `ulimit -n` does** | **Nothing.** The limit is a bitmap width. | **Everything.** |
| **Portable?** | Everywhere since 1983 | Linux only (**kqueue** on BSD) |

> **"Everything else in this comparison is downstream of one decision: who owns the watch list."**

**Source:** open both in a split window — `ngx_select_module.c` → `ngx_select_process_events()` ~L222 vs `ngx_epoll_module.c` → `ngx_epoll_process_events()` ~L784. Same signature, same job, ~200 lines each.

### 2.7 The bug every event-loop author writes exactly once (slide 15)

1. `epoll_wait` returns a **BATCH**.
2. Handling **event 1** can close the connection **event 7** points at.
3. A **new client on the recycled fd** gets event 7 delivered to it.

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

- **Structs are aligned, so the low bit of the pointer is always zero.** nginx borrows it as a **generation flag** (`instance`) and **flips it on reuse**. Mismatch → stale event → skip. *"Free bit, no extra memory, one branch."*
- The repo's `04-epoll/epolld.c` does the same with an **explicit generation counter**.

**Source:** `ngx_epoll_module.c` → `ngx_epoll_process_events()` ~L839.

> 💡 **Simple words mein:** Bank mein token 30 wala customer chala gaya, aur naye customer ko wahi token 30 mil gaya. Purani call naye bande ko na jaaye, isliye token pe ek chhota "generation" mark lagao.

### 2.8 sendfile: four copies become two (slide 16)

```
 read() + write()  —  4 copies, 2 mode switches per chunk
 ┌──────┐   ┌────────┐   ╔════════╗   ╔════════╗   ┌─────┐
 │ disk │ → │ page   │ → ║  your  ║ → ║ socket ║ → │ NIC │
 └──────┘   │ cache  │   ║ buffer ║   ║ buffer ║   └─────┘
            └────────┘   ╚════════╝   ╚════════╝

 sendfile()  —  2 copies, both DMA, 0 mode switches per chunk
 ┌──────┐   ┌────────┐   ╔════════╗   ┌─────┐
 │ disk │ → │ page   │ → ║ socket ║ → │ NIC │
 └──────┘   │ cache  │   ║ buffer ║   └─────┘
            └────────┘   ╚════════╝
```

- *"The two boxes outlined are the copies you pay CPU for. sendfile deletes the one through your address space — and with it, the two mode switches per 64 KiB chunk."* **"The data never enters your process."**
- **Why you can't use it for everything:** sendfile copies **kernel-side**. If you need to **touch the bytes** — **gzip**, **TLS**, **templating** — they must come into user space. That's why **`gzip on` and `sendfile on` fight**, and why **kTLS** exists.

> 💡 **Simple words mein:** Courier parcel godown se seedha truck mein daalo (sendfile). Pehle ghar laana, phir truck tak le jaana (read+write) bekaar chakkar hai, jab tak parcel kholna (gzip/TLS) na ho.

**Source:** `ngx_linux_sendfile_chain.c` → `ngx_linux_sendfile_chain()` ~L50 · `ngx_linux_sendfile()` ~L232 · repo: `05-sendfile/`.

### 2.9 Now the honest part: the benchmark (slide 17)

```
$ ./sendfiled 8005 www/big.bin              # 64 MB, over loopback

mode=read       wall  37.4 ms | sys 37.3 |  2048 syscalls
mode=mmap       wall  23.3 ms | sys 23.1 |     4 syscalls
mode=sendfile   wall  33.2 ms | sys 32.3 |     1 syscall
```

**On loopback, sendfile does not win on wall-clock time.**

- **There is no NIC and no DMA.** The kernel `memcpy`s into the receiver's buffer either way, and `curl` competes for the same CPU. The transfer is **memory-bandwidth bound**, so all three land in the same place.
- **What is real on any machine:** **2048 syscalls become 1**, and **64 MB never enters the process's address space**.
- **The win shows up as headroom under concurrency** — CPU you did not spend copying is CPU you spend on somebody else's request — **not as a faster single transfer**.

> **"A benchmark that does not model the bottleneck measures the wrong thing. That lesson is worth as much as sendfile is."**

### 2.10 Why "reverse" proxy? Because "proxy" was taken. (slide 18)

| | Forward proxy (Squid, 1996) | Reverse proxy (nginx) |
|---|---|---|
| sits next to | the **clients** | the **servers** |
| what it does | one person on campus downloads an image, it's cached, nobody else downloads it again; knows your users, protects your uplink | same caching, **plus**: route by **path or host**, distribute across **upstreams** by **weight** or **least-connections**, **fall back** when one dies, **terminate TLS**, and **hold ten thousand slow clients** so your app never meets one |
| flow | client → [proxy] → the whole internet | the whole internet → [nginx] → your app servers |

> *Same machinery, opposite side of the internet. The client does not configure a reverse proxy and usually cannot tell it is there — which is exactly why it can do things the client would never agree to.*

**The killer feature nobody lists: it absorbs slow clients. Your app server never talks to a phone on 2G.**

---

## 3 · THE FEATURES YOU CONFIGURE ⚙️

### 3.1 The cache: two structures, one of them on disk (slide 19)

```nginx
proxy_cache_path temp/cache levels=1:2 keys_zone=demo:10m
                 max_size=100m inactive=60s use_temp_path=off;
```

| parameter | meaning |
|---|---|
| `keys_zone=demo:10m` | **10 MB of SHARED MEMORY** holding the **rbtree** and the **LRU queue**. **The index, not the data.** Roughly **8,000 keys per megabyte**. |
| `levels=1:2` | **Two directory levels on disk**, so no single directory holds a million files and destroys your filesystem's lookup time. |
| `max_size=100m` | **Disk ceiling.** The **cache manager** process evicts from the **LRU tail** to stay under it. |
| `inactive=60s` | Evicted if **not READ for 60s — regardless of TTL.** A *different knob* from `proxy_cache_valid`, and the one people confuse. |

**Ask the class:** `proxy_cache_valid` is 10s and `inactive` is 60s. Which one wins, and what does each actually mean?

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

- **Take the tail. Is it old? Is nobody using it? Delete it.** **That is the entire eviction policy.**
- **`count == 0`** is why an object **being streamed to a client right now cannot be evicted out from under it.**
- **`src/core/ngx_queue.h`** is fifty lines of macros for an intrusive doubly-linked list. The cache LRU, the posted-events list and half of `src/core/` ride on it. **Read that file first** (§4.3).

**Source:** `ngx_http_file_cache.c` → `ngx_http_file_cache_expire()` ~L1860 · `..._lookup()` ~L1029 · `..._open()` ~L265.

### 3.3 Two lines that save you at 3am (slide 21)

```nginx
proxy_cache_lock on;

proxy_cache_use_stale error timeout
                      updating http_500 http_502 http_503 http_504;
```

- **`proxy_cache_lock on;` — the anti-stampede switch.** A popular object expires. Without it, every request that arrives in the next ~200 ms is a MISS and every one goes to the origin. With it, **one goes and the rest wait for the answer.** **Off by default.**
- **`proxy_cache_use_stale ...` — serve stale rather than fail.** Your origin is down; you have a copy from four minutes ago. Which would your users rather have — a slightly old page, or a **502**?
- **`updating`** also means **one request refreshes while everyone else keeps getting the stale copy instantly.**

> 💡 **Simple words mein:** Chai stall pe chai khatam; 500 log ek saath kitchen mein ghuse toh kitchen jal jaayegi. Lock = ek banda nayi chai banwata hai, baaki wait karte hain. use_stale = thodi purani chai de do, khaali haath mat bhejo.

```
$ curl -i localhost:8082/ | grep X-Cache   →   X-Cache-Status: MISS
$ curl -i localhost:8082/ | grep X-Cache   →   X-Cache-Status: HIT
```

> **`add_header X-Cache-Status $upstream_cache_status;` — put this in every cache config you ever write.**

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

Roughly **half the room gets `/images/a.php` wrong** — they expect the regex to win. This exact config is on **:8081** in the repo.

**nginx has no `.htaccess`.** That's not a missing feature. It's the reason nginx does not `stat()` its way up the directory tree on every single request.

> 💡 **Simple words mein:** Prefix = sarkari office ka folder tree; sabse andar wala (longest) folder jeetta hai. Regex = list upar se padho; pehla match jeetta hai. `^~` = "mera match mila, ab list mat dekho".

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
- **Regexes:** a **plain array**, walked **in order, every request**.

> **"The data structure is the documentation."** Once you see that prefixes are a **tree** and regexes are an **array**, the precedence rules stop being trivia and become **the only rules those two structures could have**.

**Source:** `src/http/ngx_http_core_module.c` → `ngx_http_core_find_location()` ~L1446 · `ngx_http_core_find_static_location()` ~L1542.

### 3.6 Smooth weighted round robin, measured (slide 24)

```c
peer->current_weight += peer->effective_weight;
total                += peer->effective_weight;

if (peer->effective_weight < peer->weight)
    peer->effective_weight++;              /* passive health check */

if (best == NULL || peer->current_weight > best->current_weight)
    best = peer;     ...

best->current_weight -= total;
```

```
 weights 3 : 1
 naive WRR      a a a b      ← bursts at one server
 nginx (smooth) a b a a      ← same ratio, interleaved
```
**Same ratio. No bursts at the origin.**

```
$ for i in $(seq 24); do curl -s "localhost:8082/x-$i-$RANDOM" -o /dev/null; done
$ grep -o 'upstream=[0-9.:]*' logs/access.log | sort | uniq -c
      6 upstream=127.0.0.1:8001
     18 upstream=127.0.0.1:8004         <- exactly 3:1
```

**Source:** `ngx_http_upstream_round_robin.c` → `ngx_http_upstream_get_peer()` ~L811 · run it: `07-nginx`, port 8082.

### 3.7 Fallbacks: one line, and every framework's pretty URLs work (slide 25)

```nginx
location / {
    try_files $uri $uri/ /index.html?$args;
}
```

> *"Is this a real file? A real directory? No? Then hand the whole URI to the app and let it route."*

**The last entry is the only one that is never tested. It is the fallback, not a candidate.**

**Fallback across a network boundary:**
```nginx
location /maybe/ {
    try_files $uri @app;
}
location @app {
    proxy_pass http://app;
}
```
- `@app` is a **NAMED location**: reachable **only from `try_files` or `error_page`**, **never from a URL**. "Try disk first; if it is not there, ask the upstream."

> **WordPress, Rails, Django, every SPA router: all of them are this directive plus an application that reads `REQUEST_URI`. Once you see it, the magic evaporates.**

**Source:** `ngx_http_try_files_module.c` → `ngx_http_try_files_handler()` ~L79 · demo: port **8084**.

---

## 4 · THE WALKTHROUGH: READING THE NGINX SOURCE 🔍

### 4.1 Line numbers drift. Function names do not. (slide 26)

```
$ git clone --depth 1 https://github.com/nginx/nginx.git
$ cd nginx && git log -1 --oneline
   231a60ee nginx-1.31.5-RELEASE

$ grep -rn "^ngx_epoll_process_events" src/
```

**nginx puts the return type on its own line**, so `grep "^name("` finds **the definition and nothing else** — not the declarations, not the call sites. One habit, and the whole tree becomes navigable.

Line numbers in these notes are pinned to **1.31.5** and **will be wrong** by the time you read this. The function names will not be.

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

> **Three protocol trees, one core.** When you have read `src/http/` you can read `src/stream/` in an afternoon — and that repetition is the argument for the architecture.

### 4.3 Two files to read before any of the code (slide 28)

| file | what it is |
|---|---|
| **`src/core/ngx_queue.h`** | ~50 lines of macros. An **intrusive doubly-linked list**: the link pointers live **inside the node**, so putting something in a list **costs no allocation**, and getting from a link back to the data is **a pointer subtraction**. The cache LRU, the posted-events queues and half of `src/core/` ride on it. |
| **`src/core/ngx_palloc.h`** | The **memory pool**. **nginx does not free per-request memory.** It allocates from a **pool attached to the request** and **destroys the whole pool at the end**. **No refcounting. No leak class. No `free()` on the hot path** — and almost no cleanup branches anywhere in the codebase. |

> 💡 **Simple words mein:** Shaadi ke liye tent hire karo; function khatam, toh poora tent ek baar mein hatao. Har kursi alag se wapas karne ki zaroorat nahi.

> Read these two and a large fraction of nginx stops looking like C and starts looking like a language with a runtime.

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

Stops 1–4 and 7–9 are covered in §2 and §3. Stops 5, 6 and 10 follow.

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

- **27 states for the request line alone.** **No regex. No `strtok`. No substring. No allocation.** **One byte at a time, and it can stop between any two.**
- **Why?** The request line may arrive in **three TCP segments**. `r->state` **survives between calls**, so the parser picks up **mid-method** if it has to.

> **This is what non-blocking costs you at the source level: every parser has to be resumable, because you can be interrupted anywhere.**

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

- **`NGX_OK` here does *not* mean "finished".** It means **"suspended"**.
- **A coroutine, hand-rolled, in C, in 1999.** The request resumes in a **later loop iteration** when its event fires, **re-entering this same `while`** at the phase it left off.
- **`limit_req` in PREACCESS · `auth_basic` in ACCESS · `try_files` in PRECONTENT · `proxy_pass`, `fastcgi_pass` and static in CONTENT.**

```
$ grep -rn "NGX_HTTP_.*_PHASE\].handlers" src/http/modules/
    # the map of which module runs when.
    # one-line answer to "why did my directive not fire?"
```

**Source:** `ngx_http_core_module.c` → `ngx_http_core_run_phases()` ~L894 · `ngx_http_core_module.h` → `ngx_http_phases` enum ~L111.

### 4.7 Stop 10 — FastCGI: framing is length or delimiter. This one uses both. (slide 32)

```
 ┌─────────┬──────┬───────────┬───────────────┬────────┬──────────┐
 │ version │ type │ requestId │ contentLength │ padLen │ reserved │   8-byte record header
 └─────────┴──────┴───────────┴───────────────┴────────┴──────────┘
                     → contentData[contentLength] → padding
```

```
>> BEGIN_REQUEST
   01 01 00 01 00 08 00 00     <- the first 8 bytes of every FastCGI connection on earth

>> PARAMS     <- EMPTY = end of stream
   01 04 00 01 00 00 00 00
```

**The empty record IS the delimiter.** An **empty PARAMS** record means "no more environment variables"; an **empty STDIN** means "no more body". **There is no other end-of-stream signal. Forget it and the responder waits forever.**

> **A length inside every header, and an empty record between streams.** Session 2's closing idea, at two levels, in one twenty-page spec.

**Source:** `ngx_http_fastcgi_module.c` → `ngx_http_fastcgi_process_record()` ~L2742 · `..._create_request()` ~L858 · repo: `06-fastcgi/`.

### 4.8 The ladder: seven servers, in code you can hold in your head (slide 33)

| dir | design | historical match | what you'll see |
|---|---|---|---|
| `01-fork` | `fork()` per client | Apache 1.3, 1995 | `ps` shows **one process per in-flight request** |
| `02-thread` | pthread per client | Apache 2.0 worker, 1999 | prints its own **8 MB stack size** at startup |
| `03-select` | one thread, `select()` | …and the **1024 wall** | **dies at fd 1024**, and `ulimit` does not help |
| `04-epoll` | one thread, `epoll()` | nginx, 2004 | **5,000 connections, nothing happens** |
| `05-sendfile` | read vs mmap vs sendfile | the copies, counted | **2048 syscalls become 1** |
| `06-fastcgi` | the wire protocol, both ends | CGI without the fork | nginx → **your own 200-line responder** |
| `07-nginx` | one `nginx.conf`, five demos | all of it, for real | routing, cache, weights, fallbacks |

**About 1,200 lines total. nginx is 200,000, and the difference is every error path we ignored.** That's the honest reason nginx is big — not complexity for its own sake.

---

## 5 · HOMEWORK 📝 (slide 34)

> Everything below runs on **Linux** (`epoll`, `strace`, `ss` are Linux tools). On a Mac use a container: `docker run -it --rm --privileged -p 8080-8090:8080-8090 ubuntu`.

### HW1 — Break `select`, then fix it
**Task:** Flood `03-select` at 1,100 connections, find where it gives up, delete the `FD_SETSIZE` guard, capture the real crash, explain in two sentences why `ulimit -n 65536` doesn't help, then flood `04-epoll` at 5,000.

**✅ Solution**

```python
# flood.py PORT N  — open N idle connections and hold them
import socket, sys, time
port, n = int(sys.argv[1]), int(sys.argv[2])
socks = []
for i in range(n):
    socks.append(socket.create_connection(("127.0.0.1", port)))
print("open:", len(socks)); time.sleep(600)
```

```bash
ulimit -n 65536; ./03-select/selectd 8003 &        # server (port as in the repo)
ulimit -n 65536; python3 flood.py 8003 1100        # client needs the raised limit too
grep -n FD_SETSIZE 03-select/*.c                   # → the guard: if (fd >= FD_SETSIZE) { close(fd); ... }
```
1. **Where it gives up:** the guard line next to `accept()` — new connections get closed once fd ≥ 1024 (≈ 1020 clients, since fds 0–2 + listen socket are taken).
2. **Delete the guard, rebuild with checks, re-run:** `cc -g -fsanitize=address -o selectd 03-select/*.c && ./selectd 8003`. At fd 1024 `FD_SET` writes past the end of the 1024-bit `fd_set` → ASan reports **stack-buffer-overflow** (without ASan: glibc `*** buffer overflow detected ***`, a segfault, or silent memory corruption).
3. **Two sentences:** `ulimit -n` only raises how many fds the *kernel* will hand the process. `select()` is limited by `FD_SETSIZE`, the compile-time width of the `fd_set` bitmap, so fd 1024+ has no bit no matter what `ulimit` says.
4. **epoll:** `./04-epoll/epolld 8004 &` then `python3 flood.py 8004 5000` → nothing happens; `ss -tn state established | wc -l` shows ≈ 5,000 (+1 header line).

### HW2 — Find the sendfile decision
**Task:** Trace `sendfile on;` down to `sendfile(2)` in `ngx_linux_sendfile_chain.c`, list every function, then add `gzip on;` and show with strace that sendfile is gone. Which filter took it away?

**✅ Solution**

Function path (find each with `grep -rn "^name(" src/`):
```
"sendfile" directive → clcf->sendfile             (ngx_http_core_module.c)
ngx_http_update_location_config()   → sets r->connection->sendfile = 1
ngx_http_static_handler()           → builds a file buffer (b->in_file = 1)
ngx_http_output_filter()            → body filter chain
  ngx_http_copy_filter() → ngx_output_chain()   → keeps the buffer "in file" (as-is)
  ngx_http_write_filter()           → c->send_chain(c, in, limit)
ngx_linux_sendfile_chain()          → ngx_linux_sendfile() → sendfile(2)
```

Test:
```nginx
# inside http { server { listen 8090; root /tmp/www; sendfile on; gzip on; } }
```
```bash
mkdir -p /tmp/www && seq 1 300000 > /tmp/www/big.html        # ~2 MB text (text/html is gzipped by default)
sudo strace -f -e trace=sendfile,pread64,writev -p $(pgrep -d, -f "nginx: worker") &
curl -s -o /dev/null localhost:8090/big.html                          # gzip off → sendfile(...) lines
curl -s -o /dev/null -H 'Accept-Encoding: gzip' localhost:8090/big.html   # gzip on → pread64 + writev, no sendfile
```
**Answer:** the **gzip filter** (`ngx_http_gzip_filter_module`) took it away — its header filter sets `r->main_filter_need_in_memory = 1`, so the **copy filter** (`ngx_output_chain`) reads the file into memory with `pread`, and the write path sends memory buffers with `writev` instead of `sendfile`.

### HW3 — Speak FastCGI to something real
**Task:** Get a PHP page from a real php-fpm with a FastCGI client, tcpdump nginx doing the same request, find FCGI_BEGIN_REQUEST by hand, and diff nginx's PARAMS against ours.

**✅ Solution**

Setup:
```bash
sudo apt install -y php-fpm nginx tcpdump
sudo sed -i 's|^listen = .*|listen = 127.0.0.1:9000|' /etc/php/*/fpm/pool.d/www.conf
sudo service php*-fpm restart
mkdir -p /tmp/www && echo '<?php echo "hi from PHP ", PHP_VERSION;' > /tmp/www/x.php
```
Minimal client (same job as `06-fastcgi/fcgi_client`):
```python
# fcgi.py — BEGIN_REQUEST, PARAMS, empty PARAMS, empty STDIN, then print STDOUT
import socket, struct
def rec(t, body): return struct.pack("!BBHHBB", 1, t, 1, len(body), 0, 0) + body
def nv(k, v):                                    # lengths < 128 → 1 byte each
    return bytes([len(k), len(v)]) + k.encode() + v.encode()
params = nv("SCRIPT_FILENAME", "/tmp/www/x.php") + nv("REQUEST_METHOD", "GET")
s = socket.create_connection(("127.0.0.1", 9000))
s.sendall(rec(1, struct.pack("!HB5x", 1, 0))     # BEGIN_REQUEST, role=1 RESPONDER
          + rec(4, params) + rec(4, b"")         # PARAMS, then EMPTY PARAMS
          + rec(5, b""))                         # EMPTY STDIN
out = b""
while True:
    h = s.recv(8, socket.MSG_WAITALL)
    if len(h) < 8: break
    _, t, _, clen, pad, _ = struct.unpack("!BBHHBB", h)
    body = s.recv(clen + pad, socket.MSG_WAITALL)[:clen] if clen + pad else b""
    if t == 6: out += body                       # STDOUT
    if t == 3: break                             # END_REQUEST
print(out.decode())
```
`python3 fcgi.py` → `Content-type: text/html; charset=UTF-8` … `hi from PHP 8.x`.

nginx doing the same:
```nginx
server { listen 8085; root /tmp/www;
  location ~ \.php$ { include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass 127.0.0.1:9000; } }
```
```bash
sudo tcpdump -i lo -X -s0 'tcp port 9000' &   curl localhost:8085/x.php
```
In the first payload packet find **`0101 0001 0008 0000`** (BEGIN_REQUEST) followed by `0001 0000 0000 0000` (role=1, flags=0), then `0104 0001 LLLL PP00` (PARAMS with length LLLL), and later **`0104 0001 0000 0000`** (empty PARAMS).

**Diff — what nginx sends that we don't:** `QUERY_STRING, REQUEST_METHOD, CONTENT_TYPE, CONTENT_LENGTH, SCRIPT_NAME, REQUEST_URI, DOCUMENT_URI, DOCUMENT_ROOT, SERVER_PROTOCOL, REQUEST_SCHEME, HTTPS, GATEWAY_INTERFACE, SERVER_SOFTWARE, REMOTE_ADDR, REMOTE_PORT, SERVER_ADDR, SERVER_PORT, SERVER_NAME, REDIRECT_STATUS`, plus every request header as `HTTP_*` (`HTTP_HOST`, `HTTP_USER_AGENT`…). **Why:** PHP builds `$_SERVER`, `$_GET` (from `QUERY_STRING`), `$_POST` (needs `CONTENT_TYPE`/`CONTENT_LENGTH` + STDIN), routing (`REQUEST_URI`), client IP (`REMOTE_ADDR`) and cookies (`HTTP_COOKIE`) from them; `REDIRECT_STATUS` satisfies PHP's `cgi.force_redirect` security check. Our client sends only the two php-fpm strictly needs.

### HW4 — Make round robin lie to you
**Task:** One upstream sleeps 10 s, weights 3:1, send 100 requests; compare p99 under round robin vs `least_conn`; check Little's Law against `ss -tn | wc -l`.

**✅ Solution**

```python
# slow.py PORT DELAY — tiny threaded upstream
import http.server, sys, time
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        time.sleep(float(sys.argv[2])); self.send_response(200); self.end_headers(); self.wfile.write(b"ok\n")
http.server.ThreadingHTTPServer(("", int(sys.argv[1])), H).serve_forever()
```
```bash
python3 slow.py 8004 0 &   python3 slow.py 8001 10 &       # weight-1 server is the slow one
```
```nginx
http { upstream app { server 127.0.0.1:8004 weight=3; server 127.0.0.1:8001 weight=1; }
       server { listen 8086; location / { proxy_pass http://app; } } }   # no proxy_cache
```
```bash
# 100 requests at 10 req/s, record each latency
for i in $(seq 100); do curl -s -o /dev/null -w '%{time_total}\n' localhost:8086/u$i >> rr.txt & sleep 0.1; done
ss -tn state established '( dport = :8001 )' | tail -n +2 | wc -l     # run at ~t=10 s
wait; sort -n rr.txt | awk '{a[NR]=$1} END{print "p50",a[50],"p99",a[99]}'
```
Then add `least_conn;` inside `upstream app`, reload (`nginx -s reload`), repeat into `lc.txt`.

| | round robin | `least_conn` |
|---|---|---|
| requests sent to slow server | 25 (fixed 1-in-4 share) | ~1 (it already has an open conn; fast server has ~0) |
| p50 | ms | ms |
| p99 | **≈ 10 s** | **ms** (only the max is ≈ 10 s) |

**Little's Law:** `L = λ × W`. λ to the slow server = 10 req/s × ¼ = 2.5 req/s, W = 10 s → **L = 25** connections open to `:8001`; the `ss` count shows ≈ 25 under round robin and ≈ 1 under `least_conn`. **Lesson:** round robin balances *requests*, not *load* — it keeps feeding a stuck server its share.

### HW5 — Read one module end to end
**Task:** One page on `try_files $uri $uri/ /index.html?$args` for a URI that does not exist (every stat, allocation, rewrite) in `ngx_http_try_files_module.c`, and why the last argument is never tested.

**✅ Solution** (answer, for URI `/nope`, `root /var/www`)

1. `ngx_http_try_files_handler()` runs in the **PRECONTENT** phase and walks the `tf[]` array built at config time (`$uri/` is stored as `$uri` with a **test_dir** flag).
2. **Allocation:** one path buffer from **`r->pool`** (`ngx_pnalloc`), sized `root + longest candidate`, re-allocated only if a later candidate is longer; `$uri` is expanded into it by the script engine. Nothing is freed — the pool dies with the request.
3. **Entry 1 `$uri`:** path `/var/www/nope` → `ngx_open_cached_file()` → **stat** → ENOENT → next.
4. **Entry 2 `$uri/`:** path `/var/www/nope` again, test_dir set → **stat** → ENOENT → next.
5. **Entry 3:** the loop checks *"is this the last entry?"* **before** any stat → expands `/index.html?$args`, `ngx_http_split_args()` splits URI and query, and **`ngx_http_internal_redirect()`** rewrites `r->uri`/`r->args` and restarts location matching. (If it starts with `@` → `ngx_http_named_location()`; if it is `=code` → returns that status.)
6. The redirected `/index.html` matches `location /` again → `try_files` runs once more → one stat succeeds → static handler serves it.

Verify: `sudo strace -e trace=newfstatat,stat,openat -p <worker pid>` while `curl localhost/nope` → two failed stats for `/nope`, then one success for `/index.html`.

**Why the last argument is never tested:** it is the **fallback, not a candidate**. It doesn't have to be a file at all (`@app`, `=404`, or a URI another location proxies), and the internal redirect re-runs location matching, so whatever handles it decides — testing it on disk would be pointless and often wrong.

---

## 6 · FOUR IDEAS TO LEAVE WITH 💡 (slide 35)

1. **Ask the other question.** Apache spent a decade making the per-connection worker cheaper. nginx asked why there was one. When a whole industry is optimising the same thing, the opportunity is usually **one level up**.
2. **Check whether the constraint still exists.** Thread-per-connection died because a thread cost **8 MB and a context switch**. **Goroutines and virtual threads brought it back** when those numbers changed. Know which of your rules are **laws** and which are **2004**.
3. **The data structure is the documentation.** Prefixes are a **tree**, so **longest** wins. Regexes are an **array**, so **first** wins.
4. **Amortise the setup.** **Prefork, pre-thread, servlets, connection pools, FastCGI, upstream keepalive.** Six names for one move: **take the expensive setup off the request path and pay it once.**

---

## 7 · CHEAT-SHEET 📋

| term | one-line meaning |
|---|---|
| **CERN httpd** | first web server, Berners-Lee, NeXT, 1990 |
| **NCSA Mosaic** | 1993 **browser** (Andreessen), not a server |
| **NCSA HTTPd** | Rob McCool's 1993 server; invented CGI; Apache's ancestor |
| **Apache** | 1995, "a patchy server"; #1 from 1996 for ~20 years |
| **MPM** | Apache's concurrency strategy: prefork (1.3) → worker (2.0) → event (2.4) |
| **CGI** | fork+exec a program per request |
| **Servlet** | Java object created once, `service()` called per request on a pooled thread |
| **J2EE** | 1999; multi-tier app server: web → EJB → DB |
| **Connection pool** | open DB connections once, lend them out per request |
| **C10K** | 10,000 concurrent connections on one box; named by Dan Kegel, 1999 |
| **8 MB** | default Linux thread stack; ×10,000 = 80 GB address space |
| **FD_SETSIZE = 1024** | compile-time width of select's bitmap; `ulimit` can't change it |
| **nginx** | Igor Sysoev, Rambler; started 2002; public 4 Oct 2004; 1.0.0 April 2011 |
| **master process** | binds ports, reads config, forks, handles signals; never touches a connection |
| **worker process** | one per core, one event loop each, does all the work |
| **cache manager / loader** | evicts LRU (to stay under `max_size`) / warms the index |
| **timer as argument** | nginx sleeps until the nearest timer; never polls for timeouts |
| **posted events** | handlers queued after `epoll_wait`, drained afterwards |
| **`ngx_accept_disabled`** | `connection_n/8 − free`; using > 7/8 of slots ⇒ stop competing for new connections |
| **thundering herd** | real in 2004; the kernel fixed it (`EPOLLEXCLUSIVE`, `SO_REUSEPORT`) |
| **accept_mutex** | nginx's userspace herd fix; default off since 1.11.3 |
| **select** | O(watched) per call; watch list rebuilt in userspace each call; 1024 cap |
| **epoll** | O(active); watch list in kernel via `epoll_ctl`; Linux only (kqueue on BSD) |
| **stale event / instance bit** | low pointer bit used as generation flag to drop events for recycled fds |
| **sendfile** | file → socket inside the kernel; 2 DMA copies, 0 mode switches per chunk |
| **gzip vs sendfile** | touching bytes (gzip/TLS/templating) needs user space → no sendfile; kTLS helps TLS |
| **loopback benchmark lesson** | no NIC, no DMA, memory-bound → same wall time; the win is CPU headroom |
| **forward proxy** | next to clients (Squid, 1996) |
| **reverse proxy** | next to servers; route, balance, fallback, TLS termination, absorb slow clients |
| **`keys_zone`** | shared-memory index (rbtree + LRU), ~8,000 keys/MB; not the data |
| **`levels=1:2`** | two directory levels on disk |
| **`max_size`** | disk ceiling, enforced by cache manager from LRU tail |
| **`inactive`** | delete if not read for this long, regardless of TTL |
| **`proxy_cache_valid`** | freshness TTL; after it, content is stale |
| **`proxy_cache_lock`** | anti-stampede: one request refills, others wait; off by default |
| **`proxy_cache_use_stale`** | serve stale copy on error/timeout/5xx; `updating` = serve stale while one refreshes |
| **`X-Cache-Status`** | `add_header X-Cache-Status $upstream_cache_status;` — always add it |
| **`location =`** | exact; wins immediately |
| **`location ^~`** | longest prefix + skip regexes |
| **`location ~`** | regex; **first** match in file order |
| **`location /prefix`** | longest prefix; used only if no regex matched |
| **prefix tree vs regex array** | why longest-prefix wins vs first-regex wins; prefixes cheap, regexes linear |
| **no `.htaccess`** | nginx never `stat()`s up the tree per request |
| **smooth WRR** | `cur += w; pick max; winner −= total` → interleaved `a b a a`, not `a a a b` |
| **`effective_weight++`** | the "passive health check" line in the WRR code |
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

## 8 · MCQs (mid → advanced)

**Q1.** A blog post says: "Berners-Lee wrote Mosaic, the first web server, and Mosaic later became Apache." Which correction is fully accurate?
- A) The first server was NCSA HTTPd; Mosaic was its browser companion; Apache forked Mosaic's code.
- B) The first server was CERN httpd (1990, NeXT); Mosaic (NCSA, 1993) was a browser; Apache descends from NCSA HTTPd.
- C) The first server was CERN httpd; Mosaic was a server at NCSA; Apache descends from Mosaic.
- D) The first server was Apache; Mosaic and NCSA HTTPd were both browsers.

> ✅ **Answer: B.** Three separate programs: CERN httpd (server), Mosaic (browser), NCSA HTTPd (server, Apache's ancestor). C is tempting but Mosaic was never a server.

**Q2.** 5,000 clients hold idle HTTP keep-alive connections. Which Apache MPM avoids tying up a worker thread for each idle client?
- A) prefork, because its processes are forked at startup
- B) worker, because threads share one 8 MB region
- C) None — every Apache MPM needs one thread per open connection
- D) event, because a listener thread parks idle keep-alives on epoll and hands off only when there is work

> ✅ **Answer: D.** The event MPM (default since 2.4) parks idle keep-alives on epoll. Prefork forks at startup, but one process still parks on one connection.

**Q3.** What is the key reason a Java servlet beats CGI per request?
- A) The servlet object is created once and outlives the request; each request is just a `service()` method call
- B) Servlets compile to native code, while CGI scripts are interpreted
- C) Servlets use `sendfile` to return responses
- D) Servlets fork a process per request, but the fork is cheaper in the JVM

> ✅ **Answer: A.** "The object outlives the request. That is the trick." CGI pays fork + exec + interpreter load + parse on every request.

**Q4.** Using the Linux default thread stack size, how much address space do 5,000 thread-per-connection threads reserve for stacks?
- A) 5 GB
- B) 8 GB
- C) 40 GB
- D) 80 GB

> ✅ **Answer: C.** 5,000 × 8 MB = 40 GB. 80 GB is the slide's figure for 10,000 threads.

**Q5.** A `select()`-based server runs after `ulimit -n 65536`, and 1,100 clients connect. What happens?
- A) All 1,100 are served, because `ulimit` raised the limit
- B) It fails around fd 1024, because `FD_SETSIZE` is a compile-time bitmap width that `ulimit` does not change
- C) It fails at 65,536 connections only
- D) The kernel refuses the 1,025th `accept()` with EMFILE

> ✅ **Answer: B.** `ulimit -n` lets the kernel hand out more fds, but `fd_set` has only 1024 bits. The kernel does give out fd 1024+ (so D is wrong); select just can't represent them.

**Q6.** A server watches 20,000 connections and 50 are active in an iteration. Roughly how much work does each API do, using the slide's model?
- A) select ≈ 20,000; epoll ≈ 20,000
- B) select ≈ 50; epoll ≈ 50
- C) select ≈ 20,000; epoll ≈ 150
- D) select ≈ 60,000; epoll ≈ 50

> ✅ **Answer: D.** select is O(n) to build + O(n) in the kernel + O(n) to scan = 3 × 20,000. epoll is O(ready) = 50, because the kernel owns the watch list.

**Q7.** A worker has `worker_connections 1024` and 100 free slots. What is `ngx_accept_disabled`, and what does the worker do?
- A) 28 — positive, so it stops competing for new connections for a while
- B) −28 — negative, so it keeps accepting
- C) 924 — it accepts 924 more connections
- D) 0 — borderline, so it accepts exactly one more

> ✅ **Answer: A.** 1024/8 − 100 = 128 − 100 = 28 > 0. It is using more than 7/8 of its slots, so it lets its siblings take new connections.

**Q8.** Why is `accept_mutex` off by default since nginx 1.11.3?
- A) Workers now use a shared counter coordinated by the master
- B) The mutex caused stale events on recycled fds
- C) The kernel fixed the thundering herd (`EPOLLEXCLUSIVE`, `SO_REUSEPORT`), so the userspace workaround became dead weight
- D) Only one worker per machine is now recommended

> ✅ **Answer: C.** It was a userspace fix for a kernel problem; once the kernel did the fan-out itself, nginx turned the fix off.

**Q9.** A worker has no I/O for 30 s, and its nearest keep-alive timer expires in 5 s. What does the loop do?
- A) Wakes every 10 ms to check whether any timer expired
- B) Calls `epoll_wait` with a ~5 s timeout, so it sleeps until the timer or I/O, whichever comes first
- C) Blocks in `epoll_wait` forever until I/O arrives, then checks timers
- D) Spins in a busy loop, checking `ngx_event_find_timer()`

> ✅ **Answer: B.** "The timer is an ARGUMENT to the sleep. nginx never polls for timeouts." C would miss the timeout entirely.

**Q10.** In one `epoll_wait` batch, handling event 1 closes connection X, and a new client is then accepted on the same fd number. What does nginx do when it reaches event 7, which was for X?
- A) Delivers it to the new client, because the fd is open and valid
- B) Calls `fcntl()` to check whether the fd is still open
- C) Re-runs `epoll_wait` to get a fresh batch
- D) Compares the low pointer bit (`instance`) with the connection's current instance; they differ, so it skips the stale event

> ✅ **Answer: D.** The fd *is* open (it now belongs to someone else), so checking openness (B) doesn't help. Identity needs a generation flag, which nginx stores free in the always-zero low bit of the aligned pointer.

**Q11.** Compared with `read()` + `write()`, which copies does `sendfile()` remove?
- A) The detour through your process's buffer: the two CPU copies and the two mode switches per chunk
- B) The disk → page cache DMA copy
- C) The socket buffer → NIC DMA copy
- D) All four copies; the data goes from disk straight to the NIC

> ✅ **Answer: A.** 4 copies become 2, both DMA (disk → page cache, socket buffer → NIC). "The data never enters your process." D overstates it: two copies remain.

**Q12.** A site enables `sendfile on;` and `gzip on;`, and the client sends `Accept-Encoding: gzip`. What happens to large HTML responses?
- A) Both work; the kernel gzips during `sendfile`
- B) nginx refuses to start with both directives
- C) Those responses can't use sendfile, because gzip must touch the bytes in user space
- D) gzip is silently disabled because sendfile has priority

> ✅ **Answer: C.** sendfile copies kernel-side. Anything that touches the bytes (gzip, TLS, templating) needs them in user space, which is why the two "fight". kTLS exists to rescue the TLS case.

**Q13.** On loopback, a 64 MB transfer shows read 37.4 ms, mmap 23.3 ms and sendfile 33.2 ms. What is the right conclusion?
- A) sendfile is useless; use mmap everywhere
- B) Loopback has no NIC or DMA and is memory-bandwidth bound; the real win is 2048 → 1 syscalls and CPU headroom under concurrency
- C) The benchmark is wrong, because sendfile always wins on wall time
- D) sendfile only helps for files under 64 KiB

> ✅ **Answer: B.** "A benchmark that does not model the bottleneck measures the wrong thing." The saved CPU shows up as headroom for other requests, not as a faster single transfer.

**Q14.** A capture between nginx and php-fpm shows the record header `01 04 00 01 00 00 00 00`. What does it mean?
- A) BEGIN_REQUEST for request 1 with an 8-byte body
- B) A STDOUT record of length 1
- C) An END_REQUEST record with padding 1
- D) An empty PARAMS record for request 1, meaning "no more environment variables"

> ✅ **Answer: D.** version=1, type=4 (PARAMS), requestId=1, contentLength=0. The empty record is the end-of-stream delimiter. A is `01 01 00 01 00 08 00 00`.

**Q15.** A Python app server handles one request at a time per worker, and many users are on 2G phones. Which reverse-proxy benefit matters most here?
- A) It absorbs slow clients: the app hands the response to nginx quickly, and nginx drips it to the phone
- B) It caches the user's credentials in the browser
- C) The client configures it, so the phones can opt in to compression
- D) It turns HTTP into FastCGI, which is faster over 2G

> ✅ **Answer: A.** "The killer feature nobody lists: it absorbs slow clients." C describes a forward proxy; a reverse proxy is invisible to the client.

**Q16.** With `proxy_cache_valid 200 10s;` and `inactive=60s`, a page is cached at t = 0 and never requested again. What is its state at t = 30 s and at t = 70 s?
- A) 30 s: fresh HIT; 70 s: stale but on disk
- B) 30 s: deleted; 70 s: deleted
- C) 30 s: stale but still on disk; 70 s: deleted by idle eviction
- D) 30 s: stale; 70 s: still on disk, because it was never refreshed

> ✅ **Answer: C.** The TTL (10 s) controls freshness; `inactive` (60 s with no reads) deletes the item regardless of TTL.

**Q17.** With `keys_zone=demo:10m`, roughly how many cache keys fit, and where does the response data live?
- A) ~8,000 keys; the data is in the same shared memory
- B) ~80,000 keys; the data lives on disk, and the zone holds only the index (rbtree + LRU)
- C) ~800,000 keys; the data lives in each worker's private memory
- D) Unlimited keys; the zone holds the response bodies up to 10 MB

> ✅ **Answer: B.** About 8,000 keys per MB × 10 MB ≈ 80,000. The zone is "the index, not the data"; `max_size` bounds the disk.

**Q18.** During eviction, the LRU tail is expired, but a client is still streaming it (`count = 1`). What does `ngx_http_file_cache_expire` do?
- A) Deletes it immediately because it is expired
- B) Moves it to the head of the queue
- C) Waits for the stream to finish
- D) Does not delete it, because deletion requires `count == 0`

> ✅ **Answer: D.** `count == 0` is the guard that stops an object being streamed right now from being evicted out from under the client.

**Q19.** A hot object expires and 500 requests arrive within 200 ms. `proxy_cache_lock on;` is set. How many requests reach the origin?
- A) About 1; the rest wait for its answer
- B) 500, because the lock only applies to stale items
- C) 0, because the lock serves the expired copy to everyone
- D) One per worker process

> ✅ **Answer: A.** The lock is the anti-stampede switch: one goes, the rest wait. Without it (the default), all 500 are MISSes. Serving stale is `proxy_cache_use_stale`, not the lock.

**Q20.** Given `location ^~ /images/ {A}`, `location ~ \.php$ {B}` and `location / {C}`, which location handles `/images/x.php`?
- A) B, because regexes are checked before prefixes
- B) C, because it is the last one in the file
- C) A, because `/images/` is the longest prefix and `^~` suppresses the regex search
- D) B, because `.php$` is a more specific match

> ✅ **Answer: C.** The longest matching prefix carries `^~`, so the regexes are never tried. About half the class picks the regex.

**Q21.** Given `location /static/ {P}` (a plain prefix) followed by `location ~ \.php$ {R}`, which location handles `/static/app.php`?
- A) P, because it is the longest prefix
- B) R, because a plain prefix is used only if no regex matches
- C) P, because it is written first in the file
- D) Neither; nginx returns 404 on the ambiguity

> ✅ **Answer: B.** Without `^~`, nginx remembers the longest prefix but tries the regexes first; one matches, so R wins. Writing order never decides between a prefix and a regex.

**Q22.** A request line arrives in three TCP segments: `GE`, `T /index.ht` and `ml HTTP/1.1\r\n`. How does nginx's parser cope?
- A) It buffers until `\r\n` and then runs a regex over the whole line
- B) It blocks the worker in `read()` until the line is complete
- C) It uses `strtok` on each segment and joins the tokens
- D) It runs a byte-at-a-time state machine and saves `r->state`, so the next call resumes mid-method

> ✅ **Answer: D.** 27 states, no regex, no strtok, no allocation. Every non-blocking parser must be resumable because it can be interrupted anywhere. B would freeze every other connection on the loop.

**Q23.** An upstream has weights 3:1 and receives 40 uncached requests. What does nginx's smooth WRR produce?
- A) 30 : 10, interleaved like `a b a a` with no bursts
- B) 30 : 10, in bursts like `a a a b`
- C) 20 : 20, because smooth WRR equalises load
- D) 40 : 0, because the highest weight always wins

> ✅ **Answer: A.** Same ratio as naive WRR, but `cur += w; pick max; winner −= total` interleaves the picks. The measured 24 requests gave 18 : 6.

**Q24.** With `try_files $uri $uri/ /index.html?$args;`, a request for `/dashboard/settings?tab=2` arrives and no such file or directory exists. What happens?
- A) 404, because `/index.html` was tested and failed
- B) nginx proxies the request to the upstream
- C) An internal redirect to `/index.html?tab=2`; the last entry is used as the fallback without being tested
- D) A 301 browser redirect to `/index.html`

> ✅ **Answer: C.** "The last entry is the only one that is never tested. It is the fallback, not a candidate." It is an internal redirect, not a client-visible 301, and the SPA router reads the URL.

**Q25.** A location has `limit_req`, `auth_basic` and `proxy_pass`, and during the run, `ngx_http_core_run_phases` gets `NGX_OK` back from a checker. Which statement is correct?
- A) `auth_basic` runs first because ACCESS is the first phase; `NGX_OK` means the request finished
- B) `limit_req` (PREACCESS) runs before `auth_basic` (ACCESS), which runs before `proxy_pass` (CONTENT); `NGX_OK` means suspended, not done
- C) They run in the order written in the config; `NGX_OK` means suspended
- D) `proxy_pass` runs first so the upstream can authenticate; `NGX_OK` means an error

> ✅ **Answer: B.** The phase, not the config order, decides execution: PREACCESS → ACCESS → … → CONTENT. `NGX_OK` returns to the event loop, and the request later re-enters the same `while` (a hand-rolled coroutine).
