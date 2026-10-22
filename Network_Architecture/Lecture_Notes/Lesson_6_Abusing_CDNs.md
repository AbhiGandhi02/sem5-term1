# 📓 (Ab)using CDNs
### *CN at Scaler — Lesson 6 (Week 6 of 8)* · "someone else's computers, closer to your users"

> **Akamai · CloudFront · Fastly · Google · Broadpeak · Cloudflare** · **SSI · ESI · Railgun · BBR · anycast**
>
> ```
> $ curl -s https://www.cloudflare.com/cdn-cgi/trace | grep colo
> colo=BOM        # you just found out which building answered you
> colo=MAA
> ```

---

## What this lecture is about

A CDN is someone else's computers, closer to your users, and everything they can do while the traffic is passing through.

> **The lecture's one-liner:** *"Weeks 1–5 made one origin fast. Today we put a network in front of it."*

The six questions about a box in the middle:

| # | section | one-line promise |
|---|---|---|
| 01 | Why the edge exists | The speed of light is a fixed budget, and your origin spends all of it. |
| 02 | What a CDN actually is | A reverse proxy (Session 4's nginx) rented in 300+ cities. Who the players are. |
| 03 | Caching, including the uncacheable | `cf-cache-status` end to end. SSI → ESI → Ajax → Railgun. Poisoning and deception. |
| 04 | Moving bytes faster | BBR, connection pooling, multiplexing to the origin. |
| 05 | Everything that isn't speed | WAF, rate limiting, DDoS, API gateway, and the trust you hand over. |
| 06 | Cloudflare, hands-on | Demos you can run now with no account; dashboard and Workers. |

---

## Table of contents

1. [Physics: why the edge exists](#1--physics-why-the-edge-exists)
2. [What a CDN actually is](#2--what-a-cdn-actually-is)
3. [Caching, including the uncacheable](#3--caching-including-the-uncacheable)
4. [Moving bytes faster](#4--moving-bytes-faster)
5. [Everything that isn't speed](#5--everything-that-isnt-speed)
6. [Cloudflare, hands-on](#6--cloudflare-hands-on)
7. [Closing: the bill, the four ideas, the repo](#7--closing-the-bill-the-four-ideas-the-repo)
8. [Assignment and homework, with solutions](#8--assignment-and-homework-with-solutions)
9. [Cheat-sheet](#9--cheat-sheet)
10. [MCQs (mid → advanced)](#10--mcqs-mid--advanced)

---

## 1 · Physics: why the edge exists

### 1.1 The speed of light is the one number you cannot optimise

From the slide (`python3 bench/01_physics.py`):

```
 light in vacuum     299,792 km/s
 light in fibre      204,218 km/s     # glass slows it to ~2/3 (index 1.468)
                     = 4.90 ms per 1000 km, one way
```

| number | what it means |
|---|---|
| **10 ms** | Mumbai → Chennai and back, in perfect straight glass. **Measured ping: 23 ms.** |
| **126 ms** | Mumbai → Ashburn (AWS `us-east-1`) and back: the floor. "You cannot go under this. Ever." |
| **47%** | how much faster it would be if the glass were *hollow*. Microsoft is deploying hollow-core fibre now (Azure, Sep 2025), "but it is not on your path today." |

(Source line on the slide: great-circle distance · fibre group index 1.468.)

**The punchline:**
> A round trip to Virginia costs **126 ms** before a single byte of yours is processed. TLS 1.3 needs **three** of those round trips to first byte. That is **~380 ms of nothing**, and no faster CPU touches it.

### 1.2 One handshake, not two (QUIC, from Session 5)

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

- **RFC 9001 §2.1**: "there is no transport handshake and then a crypto handshake. There is one handshake."
- **TLS is not optional.** There is no unencrypted QUIC.

Measured (`python3 05-http3/handshake_race.py --rtt 150`):

| stack | measured | round trips |
|---|---|---|
| TCP + TLS 1.3 + h2 (new connection) | 456.6 ms | 3.0 |
| QUIC + HTTP/3 (new connection, 1-RTT) | 319.8 ms | 2.1 |

> *"Layering is how we make systems comprehensible — and every layer boundary is a place you may pay a round trip."*

### 1.3 Distance charges you twice: once in latency, once in loss

From an Akamai edge, 2010 (**Nygren, Sitaraman & Sun, "The Akamai Network", ACM SIGOPS OSR 44(3), 2010, Table 1**):

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

A single TCP flow's ceiling falls with RTT **and** with the square root of loss, and long paths have more of both. **"The far edge is 110× slower, not 60× closer."**

**Why one flow isn't enough:** Session 5's fix (**HTTP/2 multiplexing**, many streams on one connection) and **BBR** (§4.1) both exist to claw back this exact loss.

### 1.4 What the edge buys back: handshakes are free, only the cache saves the round trip

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

**The lesson (from the slide):**
> A cache **miss** through the edge is no faster than going direct, and sometimes slower. The edge earns its place two ways: it keeps **warm connections** to your origin (handshakes), and, when it can **cache**, it deletes the origin round trip entirely (194 ms → 0).

---

## 2 · What a CDN actually is

### 2.1 A CDN is Session 4's reverse proxy, rented in 300+ cities

```
                                                     shield /
   you ───10 ms, nearby───▶  edge PoP  ───one long, controlled leg───▶ tiered cache ───▶ your origin
                          (cache · TLS · WAF)
```

**Nothing here is new:**

| CDN piece | you already met it as |
|---|---|
| the reverse proxy | nginx (Session 4) |
| the caching | `proxy_cache` (Session 4) |
| the pool | upstream keepalive (Session 3) |
| one IP in every city | anycast (Session 2's addressing) |

**What geography adds:**
- the edge is near the user, so the 3-RTT handshake is **cheap**
- the origin leg is long, but the CDN **owns both ends** of it: its own transport, its own pooling
- a shield / tiered-cache layer means many edges share **one** origin fetch, not N
- **one anycast address**, announced from every city at once

> *"Squid did reverse-proxy caching in 1996. A CDN is this, multiplied by geography and sold as a service."*

### 2.2 The founding algorithm: which cache holds which URL?

"Get this wrong and every upgrade is a cold cache." The benchmark (`python3 bench/06_consistent_hash.py`: 100,000 URLs, 10 caches → add an 11th):

| scheme | URLs that move | busiest vs mean |
|---|---|---|
| `hash(url) % N` | **90.9%** | 1.02× |
| consistent hash, 1 point/server | 3.3% | **2.93×** |
| consistent hash, 160 points/server | 8.2% | 1.14× |

**Modulo hashing is a trap:** add one server and `hash(url) % N` moves ~90% of URLs to a different cache. A capacity upgrade becomes a near-total **cache flush**, and a **stampede on your origin at the worst moment**.

**Consistent hashing (Karger, Lehman, Leighton, Levine, Lewin, Panigrahy, STOC 1997):** only the new server's fair share moves (~1/11). **Virtual points** even out the load. This paper is what **Akamai was founded on**. nginx ships it as `hash $request_uri consistent;` (160 points/server, **ketama**-compatible).

> 💡 **Simple words mein:** Modulo = class mein ek naya bachha aaya aur poori class ki seating badal gayi. Consistent hashing = naya bachha aaya, sirf uske hisse ki seats shift hui.

> *"The same idea recurs all week: choosing an owner without a central registry — DHTs, sharded caches, this. Add capacity without moving the world."*

### 2.3 The players: six CDNs, and the software you already know inside them

| CDN | born | runs on / notable for | you met it in |
|---|---|---|---|
| **Akamai** | 1998 | own "ghost" edge servers; **ESI** (2001); FastTCP → BBR | consistent hashing |
| **Amazon CloudFront** | 2008 | AWS edge; BBR to viewers since 2019 (+22%) | S2 congestion |
| **Fastly** | 2011 | a heavily-modified **Varnish** + **H2O**; instant purge | S3/S4 Varnish |
| **Google (Media CDN)** | 2022 | YouTube's edge, opened up; QUIC/BBR everywhere | S5 QUIC · BBR |
| **Broadpeak** | born of video | operator/telco CDN for streaming; on GKE too | S8 |
| **Cloudflare** | 2010 | was nginx+LuaJIT (FL1) → Rust/Oxy (FL2, 2025) | S4 nginx → Pingora |

**The software layer is Sessions 3 & 4.** Apache · HAProxy · Varnish · nginx are what most CDNs were built on. Fastly *is* Varnish. Cloudflare *was* nginx, and outgrew it:
- customer edge: nginx+LuaJIT (FL1) → Rust on Oxy (FL2) in **Oct 2025** (**−10 ms, +25%**)
- origin-facing proxy → **Pingora** in **2022**

(H2O = Kazuho Oku; Pingora & FL2 = Cloudflare engineering blog.)

### 2.4 Demo: one address, announced from 330+ cities — which one did you get?

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

- **`colo` is an airport code:** BOM Mumbai · DEL Delhi · MAA Chennai · SIN Singapore · LHR London · IAD Ashburn. It is the CDN telling you which of its buildings your packets reached, **chosen by BGP anycast, not by you**.
- **The same trick is in every response:** `cf-ray: 8c1a…-BOM`. The suffix is the colo. Every Cloudflare response carries the PoP that served it, which is how you debug *"why is this user slow / stale"*.

> Run it on the classroom wifi, then ask someone on mobile data to run it. **Same domain, same IP, different colo.** That is anycast, Session 2's addressing, put to work.

> 💡 **Simple words mein:** Anycast = ek hi toll-free number, par call aapke paas wale call centre pe lagti hai. IP same, building alag.

---

## 3 · Caching, including the uncacheable

### 3.1 The defaults: a CDN caches by file extension, and HTML is not on the list

**What Cloudflare caches with no config:**
- **68 static extensions**: `css js png jpg webp woff mp4 pdf zip …`
- **NOT cached:** HTML, JSON, or anything with `Set-Cookie`, or `Cache-Control: private / no-store / no-cache / max-age=0`
- **Default edge TTL when the origin is silent:** `200` / `301` → **2 hrs** · `404` → **3 min** · others: not cached

**The cache key is the question:**
- Default key = **scheme + host + path + query**.
- Everything else (**cookies, most headers**) is NOT in the key. Two users with different cookies **share one cached object**.
- That is correct for `app.css` and **catastrophic for `/account`**.
- `Vary:`, or a **custom key**, is how you say "these differ". Session 5's `Vary`, at the edge.

The demo (`demos/cache.sh`):
```bash
$ curl -sI edge/static/app.css | grep -i x-cache
x-cache: MISS        # first visitor pays the origin round trip
x-cache: HIT         # everyone after is free — origin never touched

$ curl -sI edge/page/home | grep -i x-cache
x-cache: DYNAMIC     # HTML: not an eligible extension, straight to origin
```

### 3.2 The state machine: `cf-cache-status` is a small language

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

> *"One header, eight states. REVALIDATED is Session 5's conditional request saving bandwidth; STALE is next week's whole topic — the cache is the last thing standing when the origin dies."*

### 3.3 Caching the uncacheable: pages got personal, four tricks to cache them anyway

| trick | from | idea |
|---|---|---|
| **SSI — Server-Side Includes** | NCSA HTTPd 1.0a1, **1994** | Holes in an HTML file, filled by the **server** before sending. Predates CGI. nginx still ships it (Session 4). |
| **ESI — Edge-Side Includes** | **Akamai + Oracle**, W3C Note **2001** | The same holes, but filled at the **EDGE**. Cache the shared shell everywhere; assemble per-user at the PoP. |
| **Ajax** | Web 2.0, **~2005** | Push the holes all the way to the **browser**. Cache the body on the CDN; fetch the personal bits async. |
| **Railgun** | Cloudflare **2013–2024** | Give up on caching the dynamic page; send only the **BYTES that changed** since last time. |

> **One arc:** move the personalisation *outward* (server → edge → browser) so the big shared part can be cached, and only the small personal part is ever recomputed. Each step shrinks what the origin has to do.

> 💡 **Simple words mein:** Thali socho: dal-chawal-roti sabke liye same, pehle se ready (cached shell). Sirf personal item alag se daalo (the holes).

### 3.4 ESI shrinks the origin's job by 98%, and the user downloads the same page

The origin sends **one shell**, cacheable by everyone, with two holes:
```html
<header><esi:include src="/fragment/user"/> <esi:include src="/fragment/cart"/></header>
```
```
Cache-Control: s-maxage=300      Surrogate-Control: content="ESI/1.0"
```

The benchmark (`python3 bench/04_esi_bytes.py`, 100 logged-in users, same page; *pages identical per user: yes, 39,676 B each*):

| | origin requests | origin → edge bytes | bytes to users | edge cache |
|---|---|---|---|---|
| **A** · origin assembles the page | 100 | 3,984,200 | 3,992,400 | DYNAMIC ×100 |
| **B** · ESI, edge assembles | **201** | **83,454** | 3,994,793 | HIT ×99, MISS ×1 |

**Read the trade carefully:**
- **Origin bytes fell 98%**: the shell is fetched once, not 100 times.
- **Bytes to the user are unchanged**: ESI does *not* shrink the download.
- **Origin requests went UP (100 → 201)**, because each personal fragment is its own tiny fetch.
- You move load **off the origin's page-rendering, onto the edge's assembly**. Whether that wins depends on which is *your bottleneck*. (The bench asserts both pages are byte-for-byte identical before counting; also as `cloudflare/worker-esi`, run in workerd.)

### 3.5 Delta encoding: Railgun's idea came back as an internet standard in 2025

Benchmark (`python3 bench/05_delta.py`, a ~24 KB news page regenerated every minute):

| compress the new page… | bytes | % of raw |
|---|---|---|
| gzip -6 (no dictionary) | 4,276 | 17.7% |
| deflate, dict = page 1 min ago | 598 | 2.5% |
| zstd, dict = page 1 min ago (dcz) | **436** | **1.8%** |
| zstd, dict = page 1 HOUR ago | 1,277 | 5.3% |

**Railgun (Cloudflare, 2013–2024):** a daemon inside your origin + a sender at the edge kept the **last version of each page on both ends**, and sent only the delta. *"Caching down to the individual bytes."* **Retired Jan 2024**: the network got fast enough, and a standard arrived.

**RFC 9842 — Compression Dictionary Transport (Sep 2025):**
- The same idea, standardised, **browser ↔ server**: the version you already have becomes the dictionary for the next one.
- `Available-Dictionary: <sha-256>` · `Content-Encoding: dcb` / `dcz` (also RFC 9841, shared brotli)
- This is Session 5's **"why not gzip the headers"**: dictionaries **leak**, so it is **same-origin, HTTPS-only, hashed**.

> A one-minute-old copy turns 24 KB into **436 bytes, ~10× past gzip**. The catch: the dictionary must be **recent** and **identical on both ends**, which is exactly what Railgun kept, and what RFC 9842 names by hash.

### 3.6 A shared cache is a shared weapon: poisoning and deception

The demo (`demos/poison.sh`):
```bash
$ curl -H 'X-Forwarded-Host: evil.example' edge/page/landing      # ONE attacker request
<script src="https://evil.example/app.js"></script>

$ curl edge/page/landing                                          # every visitor, next 60 s
<script src="https://evil.example/app.js"></script>      x-cache: HIT
```

**Cache poisoning (Kettle, "Practical Web Cache Poisoning", 2018):** the origin reflects a header (`X-Forwarded-Host`) into the page. That header is **NOT in the cache key**. One malicious request caches a hostile page; everyone else is served it. **Fix:** drop the unkeyed header at the edge, **or** add it to the key, **or** `Vary` on it.

**Cache deception (Gil, "Web Cache Deception Attack", Black Hat USA 2017):** trick the victim into loading `/account/settings/fake.css`. The edge sees ".css" and caches the response, which contained the victim's private page. The attacker then fetches the same URL from the cache. **Fix:** don't cache by extension alone; honour the origin's content-type and `no-store`.

> 💡 **Simple words mein:** Poisoning = attacker ne dukaan ke board pe galat cheez likh di, ab har customer wahi dekhta hai. Deception = victim ka private page ".css" label ki wajah se public shelf pe rakh diya gaya.

> *"Both bugs are the cache key disagreeing with reality — the same root as the personalisation problem, turned into an exploit. A cache is shared state, and shared state is an attack surface."*

---

## 4 · Moving bytes faster

### 4.1 BBR — and it is Session 2's sawtooth

**The correction:** the source doc said *"BBR algorithm by Google … relooked at TCP retries, congestion and the famous saw-tooth."* Precisely: **BBR = Bottleneck Bandwidth and Round-trip propagation** (Cardwell, Cheng, Gunn, Yeganeh & Jacobson, Google, ACM Queue 14(5), 2016). The saw-tooth is Session 2's slide.

**How it differs:**
- **Loss-based (CUBIC):** fill the pipe until a packet drops, then halve: the saw-tooth. On long, lossy CDN paths it **mistakes distance for congestion**.
- **BBR:** measure the actual **bottleneck bandwidth** and the **min RTT**, pace to that. **Loss ≠ congestion signal.**

> 💡 **Simple words mein:** CUBIC = driver jo takkar (loss) hone tak race deta hai, phir zor se brake. BBR = driver jo road ki capacity naap ke steady speed pe chalta hai.

| deployed | who / where | result |
|---|---|---|
| Google B4 + YouTube, 2016 | the paper | **2–25×** CUBIC throughput; median RTT **−53%** globally |
| Amazon CloudFront, 2019 | edge → viewer | up to **+22%** throughput; some US mobile **+100%** |
| Akamai, Nov 2019 | media delivery | BBR on **>80%** of end-user connections |
| measured across the web, 2023 | top-10k sites | BBR now the **most-deployed CCA**; CUBIC ~7% |

(Sources: AWS 2019 · Akamai 2019 · SIGCOMM 2024 CCA census.)

> **Only the sender changes:** no new protocol, no client, no router. That is why a CDN can just switch it on for you: it **owns the sender** on the long leg back to your origin, and the one out to the user.

### 4.2 Pooling: reuse the origin connection, or pay the handshake every time

Benchmark (`python3 bench/03_pool.py`: 300 visitors, edge↔origin 50 ms, TLS 1.3, origin idle-close 1 s):

| edge configuration | origin reqs | new conns | reuse | mean TTFB |
|---|---|---|---|---|
| no pool | 300 | 300 | 0.0% | 160 ms |
| 32 worker pools | 300 | 164 | 45.3% | 112 ms |
| 8 worker pools | 300 | 52 | 82.7% | 74 ms |
| 1 shared pool | 300 | **4** | **98.7%** | **57 ms** |

**Why the shared pool wins:** a pooled connection opened by one worker can only be reused **by that worker**. Split the pool across many workers and each sees too few requests to keep its connections warm **before the origin closes them**. This is **nginx's per-worker pool vs Pingora's shared pool** (Session 4's Cloudflare thread).

**Cloudflare's Pingora, in production:** moving from per-worker nginx pools to one shared Pingora pool took one big customer's reuse from **87.1% → 99.92%**: a **160× drop** in new origin connections, and **"434 years of handshake time saved every day."** (Cloudflare, "How we built Pingora", Sep 2022.) Same shape as the bench, three orders of magnitude bigger.

> 💡 **Simple words mein:** Har sawari ke liye naya auto bulana (no pool) vs auto ko waiting pe rakhna (pool). Ek common auto-stand (shared pool) har worker ke apne alag stand se zyada warm rehta hai.

> *"Amortise the setup" — Session 3's first closing idea — now spanning a continent. The setup you're amortising is a 2-RTT TLS handshake to your origin.*

### 4.3 Multiplexing and shielding: fewer connections, fewer origin fetches

**Multiplexing (Session 5, applied to the origin):**
- HTTP/2 puts many streams on one connection; the source doc's line is *"we've discussed multiplexing already."* A CDN uses it on both legs: many user streams in, a few multiplexed streams back to origin.
- **But note the default: most CDN→origin hops are still HTTP/1.1.** `cloudflared`'s `http2Origin` defaults to **false**; you turn it on. **Pooling** is what actually saves you there today.

**Tiered cache / origin shield:**
- **Without it:** 300 edges each MISS → **300 requests** hit your origin for the same object. A built-in **thundering herd** (Session 3 & 4's `cache_lock`, globally).
- **With it:** edges fetch from a **regional parent**; the parent fetches from origin **once**. Your origin sees one request, not 300.
- Cloudflare: **Tiered Cache / Argo**. nginx did the single-flight version with **`proxy_cache_lock`**.

**Request collapsing** at one edge (`bench/harness --collapse`: 50 users, cold cache, 1 object):
```
 without collapsing   50 origin fetches    # every miss stampedes
 with collapsing       1 origin fetch      # 49 wait for the first, then all HIT
```

> *Collapsing (one PoP) and tiered cache (across PoPs) are the same move at two scales: **never fetch the same object twice at once**. It is why a CDN protects your origin during a traffic spike instead of amplifying it.*

---

## 5 · Everything that isn't speed

### 5.1 All your traffic already passes through it. That changes what it's for.

> *"All requests are coming to the CDN anyway…"* — the source doc, three times in a row

| feature | what it does |
|---|---|
| **WAF** | SQLi, XSS, path traversal, inspected before your app sees them |
| **Rate limiting** | cap requests per client, per path, before they cost you anything |
| **DDoS** | absorb a 31.4 Tbps flood with capacity you could never buy |
| **API gateway** | authenticate, add/strip headers, route, fall back, at the edge |

> **The biggest shift in what a CDN is:** it started as a cache and became the **security perimeter**, because the perimeter is wherever all the traffic converges, and for you, that is the edge. *Speed got you to adopt it; control is why you can't leave.*

### 5.2 A WAF is a denylist of spellings. The bug still lives in your origin.

The demo (`bash demos/waf.sh`; origin builds SQL with string formatting; edge `--waf on`):
```
 q="' OR 1=1 --"          → blocked by edge rule sqli-tautology
 q="'/**/OR/**/1=1--"     → 5 rows leak    # comments instead of spaces; regex misses
 (edge --waf full, strips comments first)
 q="'/**/OR/**/1=1--"     → blocked        # normalisation caught it…
 q="' --"                 → 5 rows leak    # …but no OR, no UNION, no digits to match
```

**The arms race is normalisation:** a WAF matches patterns in text, so the game is how much decoding it does first (raw → URL-decode → strip comments → …) vs how many ways the attacker can spell the same attack. **You never win it outright.**

**So what is it FOR?** Buying time and blocking the mass-scanned, known payloads (**Log4Shell**, the **OWASP** top patterns): a real, useful layer. **The actual fix is in the origin: bind parameters.** `demos/waf.sh` ends by turning on `--safe`, and every bypass returns "no results."

Lab: `edge/waf.py` (**5 rules, 3 normalisation levels**). Cloudflare **Free Managed Ruleset** is on all plans; **OWASP Core** needs **Pro**.

### 5.3 Rate limiting: "100 per 10 seconds" has at least five different meanings

Benchmark (`python3 bench/07_ratelimit.py`, limit 100 / 10 s, simulated time):

| algorithm | burst at the window edge | normal page load |
|---|---|---|
| fixed window | **200 in one second !!** | all pass |
| sliding log (exact) | 100 | all pass |
| sliding window (estimate) | 104 | all pass |
| token bucket | 109 | all pass |
| leaky bucket, burst=0 | 10 | **97% refused !!** |

**The two failure modes:**
- **Fixed window:** 100 at 9.999 s + 100 at 10.001 s = **200 in a blink**. Cloudflare's **free plan only offers this 10 s window**.
- **Leaky bucket with no burst:** mathematically pure, but **refuses a real browser that opens 30 requests at once**. That's why nginx's `limit_req` has a **`burst`** parameter (Session 4's self-study).

**Cloudflare's real answer ("Counting things", Jun 2017):** the **sliding-window estimate**: weight the previous window by how much still overlaps. **Two counters per client.** Measured across **400M requests**: only **0.003%** wrongly allowed or blocked. Cheap and close enough.

**Counts are per data-center, not global:** a botnet spread across 330 cities gets **330 budgets**. The algorithm and the geography both matter. (Lab: `edge/ratelimit.py`, nginx `limit_req` excess formula.)

> 💡 **Simple words mein:** Fixed window = counter har 10 sec pe reset, toh boundary ke dono taraf 100-100 ghus jaate hain. Sliding window = pichhle window ka bacha hua hissa bhi gino, reset ka loophole khatam.

### 5.4 DDoS: the one thing you genuinely cannot do yourself

| number | meaning |
|---|---|
| **31.4 Tbps** | largest DDoS on record (Cloudflare, Dec 2025). **Aisuru** botnet, **1–4M devices**. Lasted **35 seconds**. |
| **auto** | mitigated automatically, no human in the loop. You cannot buy 31 Tbps of inbound to soak this. |
| **~500 Tbps** | Cloudflare's total network capacity, across 330+ cities: the pool the flood lands in. |

(Source: Cloudflare Q4-2025 DDoS report, "Aisuru-Kimwolf".)

**Why only a CDN can do this:** a volumetric flood is won with **more capacity than the attacker**, spread across enough locations that **anycast splits the flood across all of them** (Session 2). Your origin has one pipe; the CDN has hundreds. This is the purest case of *"rent capacity you could never own"*, and it is why sites move behind a CDN **the day after** an attack, not before.

**The catch (Session 7 preview):** one network absorbing everyone's attacks is also one network being **everyone's single point of failure**. **June 2021:** a Fastly config bug took **85%** of its network to errors: Reddit, Spotify, the UK government, gone at once. *Concentration cuts both ways.*

### 5.5 API gateway: authenticate, scrub, route — at the edge, before the origin is touched

The demo (`bash demos/gateway.sh`; edge verifies the JWT; origin only trusts the edge):
```
 no token                    → 401 edge: missing bearer token   # origin never sees it
 valid JWT                   → {"user":"alice","orders":[…]}    # edge sets X-User
 expired JWT                 → 401 edge: expired
 alg:none forgery            → 401 edge: alg 'none' refused     # never let the token pick
 client sends X-User: admin  → ignored — edge overwrites the identity it derived
 attacker hits origin IP     → {"orders":[EVERYTHING]}   !! the edge is bypassed
```

**The gateway is only real if the origin can't be reached directly.** The last line is the whole point: put auth at the edge and leave the origin open, and an attacker just skips the edge. **The fix is mutual TLS (authenticated origin pulls) or a Tunnel**: the origin only answers clients holding the edge's certificate. `demos/gateway.sh` ends by turning that on: the direct request gets **"connection reset"**, the edge still works.

**Header hygiene too:** the origin leaks `X-Powered-By: PHP/5.4.45`; the edge strips it. (Lab: `edge/jwtlite.py` (HS256, WebCrypto) · `cloudflare/worker-gateway`, same logic, run in workerd.)

> 💡 **Simple words mein:** Main gate pe guard (edge) ID check kar raha hai, par peeche ka darwaza (origin IP) khula hai, toh guard bekaar. Peeche wala darwaza band karo (mTLS / Tunnel).

---

## 6 · Cloudflare, hands-on

### 6.1 Six things every one of you can run on a laptop, no account

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
(6: also open **radar.cloudflare.com** and **speed.cloudflare.com** in a browser.)

**The quick-tunnel demo is the one to do together:** that random URL puts your laptop behind Cloudflare's edge **in seconds, no account, no DNS**. Share it; the class opens it. `tools/echo_server.py` prints each request's **`CF-Connecting-IP`**, the **`CF-Ray`** with the visitor's colo, **`CDN-Loop`**, and **how few TCP connections `cloudflared` uses for the whole room** (that's the pool). **Limits:** 200 in-flight, **no Server-Sent Events**, testing only. (`cloudflare/no-account.sh` has every command, runnable.)

### 6.2 The account demos — free plan, one zone (instructor drives)

| do this | in the dashboard | the point | free-plan limit |
|---|---|---|---|
| orange vs grey cloud | DNS → toggle proxy | `dig` shows origin IP vanish behind anycast | — |
| cache an HTML page | Cache → Cache Rules | MISS→HIT→purge→MISS; `Age` climbs | 10 rules |
| block naive SQLi | Security → WAF custom rule | same bypasses as `demos/waf.sh` | 5 rules, no regex |
| rate-limit `/api/*` | Security → Rate limiting | ~20 pass then **429**; per-colo counts | 1 rule, 10 s only |
| strip `X-Powered-By` | Rules → Transform Rules | header hygiene, no code | unlimited |
| deploy a Worker | `wrangler deploy` | your code at the edge | 100k req/day |

**The limits are themselves the lesson:** a WAF rule with no regex is why `demos/waf.sh`'s bypasses matter; a 10-second-only rate window is exactly the **fixed-window weakness**; HTML not cached until you add a Cache Rule is the **default-extension rule**. *"The constraints teach the concepts."* (Full steps, with exact expressions to paste: `cloudflare/INSTRUCTOR.md`.)

### 6.3 Workers: the edge runs your code — Session 3's question, answered again

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

**Session 3 asked: "isn't everything a web server?"** CGI forked a process per request; FastCGI kept them warm; nginx ran an event loop. A **Worker** is the next answer: a **V8 isolate, not a process**, starts in **~5 ms, ~3 MB**, not the **35 MB + cold-start of a container**.

**Amortise the setup, one more time:** no fork+exec per request (Session 3's CGI cost); no VM to boot. The runtime is **already resident in every PoP**; your code just runs in it. It is prefork / FastCGI / servlets, the same closing idea, taken to the edge. (Cloudflare, "Cloud computing without containers", 2018.)

**What actually ran here:** both Workers in `cloudflare/` were executed locally in **workerd** (Cloudflare's real runtime, via **Miniflare**) against the lab origin. The **gateway Worker** reproduced every line of `demos/gateway.sh` (401 on no token, `alg:none` refused, failover to backup). The **ESI Worker**'s assembled page was checked byte-for-byte against the origin's own, and there's a **framing bug** in it (why not **HTMLRewriter**) that is Session 2's *"where does this element end?"* all over again.

---

## 7 · Closing: the bill, the four ideas, the repo

### 7.1 The bill: what you hand over when you put a stranger in the middle

| what you hand over | detail |
|---|---|
| **Your TLS private keys** | The edge terminates HTTPS, so it holds the keys (or its own cert for your domain). **It can read every byte.** That is the deal. |
| **A single point of failure** | Fastly, **8 Jun 2021**: one customer's config change → **85%** of the network to errors → Reddit, Spotify, gov.uk down. **49 min** to recover. |
| **The power to see and shape traffic** | **Domain fronting** used CDNs to hide censored traffic behind innocent domains, until **AWS, Google (2018), Fastly (2024)** all banned it (Azure 2022). *Whoever runs the edge decides what flows.* |
| **Lock-in by architecture** | Once WAF, rate limits, auth and Workers live at the edge, it is not a cache you swap: **it is your security perimeter**. |

> *"A CDN is enormous leverage rented cheaply — and the rent is **trust, concentration, and dependence**. Use it. Know the bill."*

### 7.2 The four ideas

1. **Move the work, or move the data.** You cannot beat the speed of light, so you move content closer (the cache) or push personalisation outward (SSI → ESI → Ajax) until only the shared part travels far. Every CDN trick is one of these two moves.
2. **The edge is wherever all the traffic converges.** Caching routed everything through the edge; so the firewall, the rate limiter, the DDoS scrubber, the auth and now your code all followed the traffic there. Speed made you adopt it; control is why it's now your perimeter.
3. **It's all Sessions 2–5, rented by the city.** Reverse proxy (S4), connection pool (S3), ETag revalidation (S5), BBR vs the sawtooth (S2), multiplexing (S5), consistent hashing, framing (S2). A CDN invents almost nothing; it distributes what you already know.
4. **Concentration cuts both ways.** One network can absorb a 31 Tbps flood no one could survive alone, and one config bug in it takes 85% of the web down at once. The same pooling that is your strongest defence is your single point of failure.

### 7.3 The repo: `cdn-lab` — every number today came out of this, with no network

```bash
$ ./setup.sh       # a private CA + 3 certs. openssl only.
$ ./run_all.sh     # 7 benches + 6 demos, ~80 s, no network
```

| file | what it is |
|---|---|
| `origin/origin.py` | the naive app server (trusts `X-User`, f-string SQL, leaks headers) |
| `edge/edge.py` | a CDN PoP in one file: cache · pool · WAF · rate limit · JWT · ESI |
| `tools/laggy.py` | a TCP proxy that adds distance (the physics floor) |
| `cloudflare/` | `no-account.sh` · `INSTRUCTOR.md` · two Workers (run in workerd) |

**Honest about what it does NOT model:** `laggy.py` **delays, it never drops** (no loss, no bandwidth; BBR vs CUBIC needs `tc netem`, homework 2). Every hop is **HTTP/1.1**. **Anycast and BGP can't live on one machine** (use `cloudflare/no-account.sh`). The worker-pool bench assigns each visitor a **random simulated worker** rather than a real kernel accept queue. Where a number is a model and not a measurement, the slide and the README say so.

> Read `edge/edge.py` end to end: it is ~450 lines and it is the whole lecture as code. Then break it: set `--pool 0` and watch reuse collapse, `--waf off` and watch the injection land.

---

## 8 · Assignment and homework, with solutions

### 8.1 Lesson 6 assignment — "Put something real behind the edge, and prove what it did"

**Task:** any language. Serve **one cacheable static asset** and **one personalised HTML page**, put it behind Cloudflare (free zone, or a Quick Tunnel), and prove with numbers: **Cache** (MISS→HIT for the asset, DYNAMIC for the page, then make the page cacheable and show the origin request count drop) · **The key** (one cache-key hazard, then close it) · **Measure** (TTFB for cold origin vs warm edge vs cache HIT) · **One edge feature** (WAF rule, rate limit, or token-checking Worker: one blocked, one allowed request). **Deliverable:** code + one-page write-up. Due before Session 7.

#### ✅ Solution

**Step 0: the origin** (`origin.py`, standard library only). It counts and logs every request it receives, and it deliberately reflects an `X-Theme` header (the hazard).

```python
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
hits = 0
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        global hits; hits += 1
        print(f"origin hit #{hits}: {self.path}", flush=True)        # origin request count
        if self.path.startswith("/static/app.css"):
            body, ctype, cc = b"body{color:teal}", "text/css", "public, max-age=3600"
        else:                                                         # /page and /api/*
            name  = self.headers.get("Cookie", "user=guest").split("=")[-1]
            theme = self.headers.get("X-Theme", "light")              # reflected, NOT in cache key
            body  = f'<html><body class="{theme}">Hi, {name}</body></html>'.encode()
            ctype, cc = "text/html", "public, s-maxage=60"
        self.send_response(200)
        self.send_header("Content-Type", ctype); self.send_header("Cache-Control", cc)
        self.send_header("Content-Length", str(len(body))); self.end_headers()
        self.wfile.write(body)
ThreadingHTTPServer(("127.0.0.1", 8000), H).serve_forever()
```

**Step 1: put it behind Cloudflare.** Cache Rules, Transform Rules and rate limits need a zone, so the easiest full path is a free zone (any cheap domain) + a named tunnel. The origin needs no open port:
```bash
python3 origin.py                                   # terminal 1
cloudflared tunnel login                            # terminal 2 (pick your zone)
cloudflared tunnel create lab
cloudflared tunnel route dns lab lab.yourdomain.com
cloudflared tunnel run --url http://localhost:8000 lab
U=https://lab.yourdomain.com
st(){ curl -s -o /dev/null -D - "$@" | grep -i '^cf-cache-status'; }   # GET, print status
```
(A Quick Tunnel, `cloudflared tunnel --url http://localhost:8000`, also puts the origin behind the edge, but you can't add Cache/Transform/rate-limit rules to a `trycloudflare.com` name, so use a zone for Steps 2–5.)

**Step 2: Cache.**
```bash
st $U/static/app.css; st $U/static/app.css      # cf-cache-status: MISS, then HIT
st $U/page; st $U/page                          # cf-cache-status: DYNAMIC, DYNAMIC
for i in $(seq 50); do curl -s -o /dev/null $U/page; done   # origin log: +50 hits
```
Now add **Caching → Cache Rules → "URI Path starts with /page" → Eligible for cache** (the origin already sends `s-maxage=60`). Run the 50-request loop again: the origin log shows **+1** (the first MISS), and `st $U/page` shows `HIT`. Write down: **50 → 1 origin requests**. Also note that every visitor now sees the same "Hi, <name>", which is exactly why HTML is not cached by default.

**Step 3: The key (cache-key hazard).** Use a fresh query string so the entry is new (query is part of the key):
```bash
curl -s -H 'X-Theme: evil' "$U/page?t=1"     # MISS: origin builds class="evil", edge stores it
curl -s "$U/page?t=1"                        # HIT: a normal visitor also gets class="evil"
```
**Why it worked:** the response depends on `X-Theme`, but the key is only scheme + host + path + query, so one attacker request decided what everyone got. **Close it:** Rules → Transform Rules → Modify Request Header → **Remove `X-Theme`**. Repeat with `?t=2`: the second request now shows `class="light"`.

**Step 4: Measure** (run from a far-away box, e.g. a cloud VM in another region; 10 runs each, report the median):
```bash
t(){ for i in $(seq 10); do curl -s -o /dev/null -w '%{time_starttransfer}\n' "$1${2:+$2$RANDOM}"; done | sort -n | sed -n 5p; }
t "$U/page" "?r="                     # cold: new query each run → MISS → full origin round trip
t $U/static/app.css                   # warm edge: cache HIT, new connection each time
curl -s -o /dev/null -o /dev/null -w '%{time_starttransfer}\n' $U/static/app.css $U/static/app.css
                                      # 2nd line = HIT on a kept-alive connection
```
Put the three numbers in one table like bench/02. Expect: MISS ≫ HIT (new conn) > HIT (keep-alive).

**Step 5: One edge feature: a rate limit.** Security → Rate limiting: *URI Path starts with `/api/`*, **20 requests per 10 seconds**, per IP, action **Block**.
```bash
for i in $(seq 30); do curl -s -o /dev/null -w '%{http_code}\n' $U/api/x; done | sort | uniq -c
#   20 200      ← allowed
#   10 429      ← blocked (counts are per colo, approximate)
```

**Write-up (one page):** the MISS/HIT/DYNAMIC lines, the origin count 50 → 1, the `X-Theme` story (why it poisoned, which fix closed it), the TTFB table, and the 200/429 counts.

### 8.2 Homework (5 items)

**1. Run the whole lab and read `edge/edge.py`.** `./setup.sh && ./run_all.sh`, then change one flag (`--pool 0`, `--waf off`, `--strip-unkeyed`) and explain the number that moves.

> ✅ **Solution:** run `./setup.sh && ./run_all.sh` (~80 s). Then rerun with one flag changed and compare:
> - `--pool 0` → connection reuse drops to **0%** and TTFB rises (bench/03: 57 ms → ~160 ms at 50 ms RTT), because every request pays the 2-RTT TCP + TLS handshake to the origin again.
> - `--waf off` → `q="' OR 1=1 --"` is no longer blocked and the rows leak, because the f-string SQL bug is still in the origin.
> - `--strip-unkeyed` → `demos/poison.sh` stops working: the second (victim) request gets the normal page, because the edge drops `X-Forwarded-Host` before the origin can reflect it.

**2. Redo BBR vs CUBIC for real.** On a real Linux box, use `tc netem` to add **1% loss at 150 ms RTT**, download a large file under `cubic` and under `bbr` (`sysctl`), and report the gap.

> ✅ **Solution** (one Linux machine is enough: on loopback every packet passes netem once in each direction, so 75 ms gives a 150 ms RTT):
> ```bash
> sudo modprobe tcp_bbr; sysctl net.ipv4.tcp_available_congestion_control   # must list bbr
> head -c 5M /dev/urandom > big
> sudo ip link set lo mtu 1500                              # realistic MSS (~1460 B)
> sudo tc qdisc add dev lo root netem delay 75ms loss 1%
> ping -c 3 127.0.0.1                                       # check: ~150 ms
> for cc in cubic bbr; do
>   sudo sysctl -qw net.ipv4.tcp_congestion_control=$cc
>   python3 -m http.server 8000 >/dev/null 2>&1 & sleep 1   # restart so the server uses $cc
>   curl -s -o /dev/null -w "$cc: %{speed_download} B/s\n" http://127.0.0.1:8000/big
>   kill $!
> done
> sudo tc qdisc del dev lo root; sudo ip link set lo mtu 65536    # clean up
> ```
> **Predict CUBIC first with Mathis:** 1460 B / (0.15 s × √0.01) ≈ **97 KB/s ≈ 0.8 Mbps**. CUBIC should land near that; BBR should be many times faster, because it does not treat the 1% random loss as congestion. Report the two numbers and the ratio.

**3. Poison your own cache, then fix it three ways** (drop the header, add it to the key, Vary on it), and argue which a real CDN should default to.

> ✅ **Solution:** the easiest "site you control" is nginx (Session 4) as the cache in front of `origin.py` from §8.1. Save as `poison.conf`, then run `mkdir -p logs && nginx -p $PWD -c $PWD/poison.conf`:
> ```nginx
> events {}
> http {
>   proxy_cache_path /tmp/nc keys_zone=c:1m;
>   server {
>     listen 8080;
>     location / {
>       proxy_pass http://127.0.0.1:8000;
>       proxy_cache c;  proxy_cache_valid 200 60s;
>       add_header X-Cache $upstream_cache_status;
>       # FIX 1 (drop):  proxy_set_header X-Theme "";
>       # FIX 2 (key):   proxy_cache_key $scheme$host$request_uri$http_x_theme;
>     }
>   }
> }
> ```
> ```bash
> curl -s -H 'X-Theme: evil' localhost:8080/page?n=1      # MISS, stored
> curl -si localhost:8080/page?n=1 | grep -iE 'x-cache|body'   # X-Cache: HIT, class="evil" → poisoned
> ```
> Uncomment one fix, `nginx -p $PWD -c $PWD/poison.conf -s reload`, and repeat with a new `?n=`: the victim now gets `class="light"`. **FIX 3 (Vary):** add `self.send_header("Vary", "X-Theme")` in `origin.py`; nginx then keeps a separate copy per `X-Theme` value.
>
> **Which should be the default: drop the header.** Adding it to the key or using Vary keeps the origin's behaviour but splits the cache into one copy per header value (lower hit rate, and an attacker can still fill the cache with junk variants). Dropping removes the unkeyed input completely, so the key matches reality again. Key or Vary on a header only when the origin truly needs it.

**4. Find a real `cf-cache-status` in the wild.** `curl -I` ten sites, explain each status, its `Cache-Control`, and its cache key. Find one HIT you think is a mistake.

> ✅ **Solution:**
> ```bash
> for s in site1.com site2.com site3.com; do echo "== $s"
>   curl -sI "https://$s/" | grep -iE '^(cf-cache-status|cache-control|age|vary|set-cookie):'; done
> ```
> Read each line with the slide rules: **HIT** + `public, max-age=…` = cached; the key is scheme + host + path + query. **DYNAMIC** = not eligible (usually HTML/JSON, or `private`/`no-store`), so every request goes to origin. **MISS/EXPIRED** = eligible but not fresh in this colo; run it again and it should become HIT. **BYPASS** = origin said no-store. **REVALIDATED** = 304 from origin. A climbing `age` shows how long the copy has been at the edge.
> **A "mistaken" HIT** is usually an HTML or API response that is a HIT while it also carries `Set-Cookie` or user-specific content, or says `Cache-Control: private`. Those are personal responses sitting in a shared cache whose key ignores cookies.

**5. (The one to do if you do only one) Measure the speed of light to your users.** For 5 user cities, compute the RTT floor and measure the real ping. Where is the gap biggest, and would a CDN close it?

> ✅ **Solution** (origin assumed in Mumbai, AWS endpoints used as targets; TCP connect time ≈ 1 RTT):
> ```python
> import math, subprocess
> def km(a, b):
>     (la1, lo1), (la2, lo2) = map(lambda p: map(math.radians, p), (a, b))
>     h = math.sin((la2-la1)/2)**2 + math.cos(la1)*math.cos(la2)*math.sin((lo2-lo1)/2)**2
>     return 2 * 6371 * math.asin(math.sqrt(h))
> origin = (19.08, 72.88)                                         # Mumbai
> users = {"Singapore": ((1.35, 103.82), "ap-southeast-1"), "Tokyo": ((35.68, 139.69), "ap-northeast-1"),
>          "Frankfurt": ((50.11, 8.68), "eu-central-1"), "London": ((51.51, -0.13), "eu-west-2"),
>          "Virginia": ((39.04, -77.49), "us-east-1")}
> for city, (ll, reg) in users.items():
>     floor = 2 * km(origin, ll) / 204.218                       # 204.218 km per ms in fibre
>     w = subprocess.run(["curl", "-so", "/dev/null", "-w", "%{time_namelookup} %{time_connect}",
>                         f"https://dynamodb.{reg}.amazonaws.com/ping"], capture_output=True, text=True).stdout
>     dns, conn = map(float, w.split()); real = (conn - dns) * 1000
>     print(f"{city:10} floor {floor:4.0f} ms   real {real:4.0f} ms   gap {real - floor:4.0f} ms")
> ```
> Virginia's floor comes out at ~126 ms, as on the slide. The **biggest gap** is usually on the far or indirectly routed cities. **Would a CDN close it?** Only partly. A cache HIT removes the whole origin round trip, and a nearby edge with a warm pool removes the long handshakes. For dynamic, uncacheable content the one long origin round trip stays (the 621 → 233 ms row in §1.4), and nothing gets under the physics floor.

### 8.3 Other assignments shown on the slides

- **Early HTTP/1.1 assignment, "Build a calculator that stays on the line":** see Lesson 5 §8 for this assignment and its solution.
- **Course project, "HTTP, in binary · two tracks, one protocol":** see Lesson 5 §9.

---

## 9 · Cheat-sheet

### Numbers worth remembering

```
 204,218 km/s in fibre = 4.90 ms / 1000 km one way     47% faster if hollow-core
 10 ms Mumbai↔Chennai floor (ping 23)    126 ms Mumbai↔Virginia floor → ~380 ms TLS 1.3 to first byte
 3 RTT  TCP+TLS1.3+HTTP/2 cold           2 RTT  QUIC cold        (TLS 1.2: 4)
 621 direct new · 625 edge no pool · 233 warm pool · 34 HIT · 11 HIT keep-alive (ms)
 Mathis: throughput ≈ MSS / (RTT × √loss) — "110× slower, not 60× closer"
 90.9% moved (mod N) · 3.3% / 2.93× (1 point) · 8.2% / 1.14× (160 points)
 68 extensions · TTL 200/301 → 2 h · 404 → 3 min · key = scheme+host+path+query
 ESI: origin bytes −98%, origin requests 100 → 201, user bytes unchanged
 24 KB → 436 B (zstd, 1-min dict) vs 4,276 B gzip · 1-hour dict 1,277 B
 BBR: 2–25× CUBIC, RTT −53% · CloudFront +22% · Akamai >80% · CUBIC ~7% of top-10k
 Pool: 0% / 45.3% / 82.7% / 98.7% reuse · Pingora 87.1% → 99.92%, 160×, 434 years/day
 Collapsing: 50 → 1 origin fetches · shield: 300 → 1
 Rate limit 100/10 s: fixed 200 · log 100 · sliding 104 · token 109 · leaky(0) 10, 97% refused
 Sliding estimate: 2 counters, 0.003% error over 400M · per colo = 330 budgets
 DDoS 31.4 Tbps, 35 s, 1–4M devices, auto · ~500 Tbps capacity, 330+ cities
 Fastly 8 Jun 2021: 85% errors, 49 min · Worker ~5 ms, ~3 MB vs 35 MB container
 Free plan: 10 cache rules · 5 WAF rules, no regex · 1 rate rule, 10 s only · Worker 100k req/day
```

### cf-cache-status in one line each

```
 MISS eligible, fetched · HIT fresh · EXPIRED re-fetched · REVALIDATED 304
 UPDATING stale now, refresh in bg · STALE origin down · DYNAMIC never eligible · BYPASS no-store
```

### The four ideas
```
 1. Move the work, or move the data.
 2. The edge is wherever all the traffic converges.
 3. It's all Sessions 2–5, rented by the city.
 4. Concentration cuts both ways.
```

---

## 10 · MCQs (mid → advanced)

**Q1.** Your origin is in Ashburn (Virginia) and your users are in Mumbai. What is the minimum time before a brand-new TCP + TLS 1.3 connection can deliver the first byte, counting physics only?
- A) ~126 ms
- B) ~252 ms
- C) ~380 ms
- D) ~500 ms

> ✅ **Answer: C.** The round-trip floor is 126 ms, and TCP + TLS 1.3 + the request need three round trips: 3 × 126 ≈ 380 ms. 126 ms (A) is only one round trip.

**Q2.** Light in fibre takes 4.90 ms per 1000 km one way. What is the RTT floor between two cities 5,000 km apart?
- A) 24.5 ms
- B) 49 ms
- C) 33 ms
- D) 98 ms

> ✅ **Answer: B.** One way = 5 × 4.90 = 24.5 ms, so the round trip = 49 ms. Option A forgets the return trip.

**Q3.** How many round trips does a cold connection need before the first response byte with QUIC + HTTP/3, and with TCP + TLS 1.2 + HTTP?
- A) 2 and 3
- B) 1 and 3
- C) 2 and 4
- D) 1 and 4

> ✅ **Answer: C.** QUIC merges the transport and crypto handshakes into one ("there is one handshake"), so it needs 1 + the request = 2. TCP + TLS 1.3 needs 3, and with TLS 1.2 "it was four".

**Q4.** By the Mathis formula, a single TCP flow moves to a path with **double the RTT** and **four times the loss rate**. Its throughput ceiling:
- A) falls by 4×
- B) falls by 8×
- C) falls by 2×
- D) falls by 16×

> ✅ **Answer: A.** Throughput ≈ MSS / (RTT × √loss). 2× the RTT and √4 = 2× from loss give 2 × 2 = 4×. Option B forgets the square root on loss.

**Q5.** A CDN edge with a **warm origin pool** serves an uncacheable page to a brand-new user (you–edge 10 ms, edge–origin 194 ms). Compared with going direct on a new connection (621 ms), what happens?
- A) TTFB drops to ~34 ms, because the edge removes the origin round trip
- B) Nothing changes, because a miss through the edge is never faster
- C) TTFB rises to ~625 ms, because there are two handshakes
- D) TTFB drops to ~233 ms: the handshakes are now short, but the 194 ms origin round trip remains

> ✅ **Answer: D.** A warm pool saves the long handshakes, but only a cache HIT deletes the origin round trip (34 ms is the HIT row). 625 ms is the edge **without** a pool.

**Q6.** Users in Mumbai and Chennai both run `curl -s cloudflare.com/cdn-cgi/trace`. They get the same IP but `colo=BOM` and `colo=MAA`. What explains this?
- A) DNS returns a different IP to each user based on location
- B) The same IP is announced from many cities, and BGP anycast routing picks the building
- C) The user's browser picks the nearest PoP after measuring ping
- D) The origin redirects each user to a regional mirror

> ✅ **Answer: B.** The colo is "chosen by BGP anycast, not by you". The IP is the same, so it is not DNS-based steering (A).

**Q7.** In `bench/06`, consistent hashing with **1 point per server** moved only 3.3% of URLs when an 11th cache was added. Why is that setup still a bad choice?
- A) The busiest server carried 2.93× the mean load; virtual points (160/server) fix the uneven load
- B) It moves too many URLs compared with `hash % N`
- C) It cannot add servers without a full cache flush
- D) It needs a central registry to map URLs to servers

> ✅ **Answer: A.** With one point per server the load is very uneven (2.93× mean). With 160 points it is 1.14×, and the 8.2% that moves is close to the fair ~1/11 share. B and C describe modulo hashing.

**Q8.** Why is `hash(url) % N` called "a trap" for a CDN that adds capacity during a traffic peak?
- A) Its load balance is poor (busiest server ≫ mean)
- B) It requires 160 virtual points per server
- C) It only works with nginx
- D) Adding one server moves ~90% of URLs, which acts like a cache flush and a stampede on the origin at the worst moment

> ✅ **Answer: D.** 90.9% of URLs move. Its balance is actually fine (1.02×), so A is wrong.

**Q9.** A `.js` file (an eligible extension) is requested through Cloudflare, and the origin responds with `Cache-Control: no-store`. What `cf-cache-status` do you see?
- A) DYNAMIC
- B) MISS
- C) BYPASS
- D) EXPIRED

> ✅ **Answer: C.** BYPASS = "eligible, but origin said no-store". DYNAMIC is for responses that were never eligible at all, like HTML by default.

**Q10.** A cached object's TTL has passed. The edge asks the origin, and the origin replies `304 Not Modified`. The status is:
- A) REVALIDATED
- B) EXPIRED
- C) UPDATING
- D) STALE

> ✅ **Answer: A.** REVALIDATED = TTL passed, origin said 304, cache reused (Session 5's ETag / If-None-Match). EXPIRED means the full object was fetched again.

**Q11.** With **no Cache Rule**, an HTML page whose origin sends `Cache-Control: public, max-age=600` goes through Cloudflare. What happens?
- A) HIT after the first request, for 600 s
- B) DYNAMIC: HTML is not one of the default cached extensions, so it goes straight to the origin
- C) BYPASS, because `public` is not allowed on HTML
- D) MISS forever, because the default key ignores cookies

> ✅ **Answer: B.** By default the CDN caches by file extension, and HTML is not on the list. You need a Cache Rule ("cache an HTML page" demo) to make it eligible.

**Q12.** The origin sends **no caching headers**. With Cloudflare's defaults, an eligible `.png` that returns `200` and another `.png` that returns `404` are cached for:
- A) 2 hours and not cached
- B) 3 minutes and 2 hours
- C) not cached and 3 minutes
- D) 2 hours and 3 minutes

> ✅ **Answer: D.** Default edge TTL when the origin is silent: 200/301 → 2 hrs, 404 → 3 min, others not cached.

**Q13.** Under the default cache key, which pair of requests is served the **same** cached object?
- A) `https://a.com/app.css?v=3` and `https://a.com/app.css?v=4`
- B) `http://a.com/app.css` and `https://a.com/app.css`
- C) `https://a.com/app.css` from user X (cookie `id=1`) and from user Y (cookie `id=2`)
- D) `https://a.com/app.css` and `https://b.com/app.css`

> ✅ **Answer: C.** The key is scheme + host + path + query. Cookies are not in it, so both users share one object. A changes the query, B the scheme, and D the host.

**Q14.** An ESI shell has **three** personal holes, and 100 logged-in users load the page once each on a cold cache. How many origin requests, and how do bytes to users compare with origin assembly?
- A) 301; bytes to users stay about the same
- B) 100; bytes to users drop ~98%
- C) 201; bytes to users stay about the same
- D) 1; bytes to users drop ~98%

> ✅ **Answer: A.** 1 shell fetch + 100 × 3 fragments = 301 (the slide's two-hole case gave 201). ESI cuts origin **bytes**, not the user's download, so B and D are wrong.

**Q15.** In `bench/05`, why did zstd with a one-**hour**-old dictionary produce 1,277 B instead of 436 B?
- A) zstd cannot use dictionaries older than one minute
- B) Delta compression works best when the dictionary is recent; an older copy shares less text with the new page
- C) The hour-old dictionary was not identical on both ends
- D) RFC 9842 forbids dictionaries older than one hour

> ✅ **Answer: B.** The catch on the slide: the dictionary must be **recent** and **identical on both ends**. An hour-old copy is still identical but less similar, so it saves less, though still better than gzip's 4,276 B.

**Q16.** A logged-in victim clicks `https://shop.com/account/settings/fake.css`. The origin ignores the suffix and returns the private settings page, and the edge caches it. Then the attacker fetches the same URL. Which attack is this, and what is the slide's fix?
- A) Cache poisoning; drop the unkeyed header
- B) Cache poisoning; add the header to the key
- C) Cache deception; Vary on Cookie
- D) Cache deception; don't cache by extension alone, and honour the origin's content-type and no-store

> ✅ **Answer: D.** This is Gil's deception attack: the victim creates the cached entry and the attacker reads it. Poisoning (A, B) is when the attacker's own request plants hostile content via an unkeyed header.

**Q17.** The origin reflects `X-Forwarded-Host` into a cached landing page. Which of these is **not** one of the slide's fixes?
- A) Drop the header at the edge
- B) Add the header to the cache key
- C) Lower the edge TTL from 60 s to 10 s
- D) Vary on the header

> ✅ **Answer: C.** A shorter TTL only shortens the poisoned window; one request can still poison it again. The three fixes are drop, key, or Vary.

**Q18.** On a long CDN path with some random packet loss, why does BBR usually beat CUBIC?
- A) CUBIC treats every loss as congestion and cuts its rate (the saw-tooth); BBR paces to the measured bottleneck bandwidth and min RTT, so loss ≠ congestion signal
- B) BBR retransmits lost packets faster using a new protocol
- C) BBR needs the client and the routers to cooperate, so it gets priority
- D) CUBIC cannot use more than 7% of the link

> ✅ **Answer: A.** CUBIC "mistakes distance for congestion". BBR needs only the sender to change: no new protocol (B), no client or router (C). The "~7%" is CUBIC's share of top-10k sites, not a link limit (D).

**Q19.** Why can a CDN switch BBR on for your users without them changing anything?
- A) Browsers already ship BBR as a default
- B) Congestion control lives only in the sender, and the CDN owns the sender on both legs (to the user and back to the origin)
- C) BBR is part of QUIC, which all CDNs use
- D) BBR is negotiated in the TLS handshake via ALPN

> ✅ **Answer: B.** "Only the sender changes — no new protocol, no client, no router." The CDN is the sender out to the user and on the long leg to the origin.

**Q20.** In `bench/03` (origin idle-close 1 s), why did **32 per-worker pools** reach only 45.3% reuse while **one shared pool** reached 98.7%?
- A) Per-worker pools use HTTP/1.1 and the shared pool uses HTTP/2
- B) The shared pool keeps TLS session tickets that worker pools can't
- C) The origin limits each worker to one connection
- D) A connection can be reused only by the worker that opened it, so each of 32 workers sees too few requests to reuse its connections before the origin closes them

> ✅ **Answer: D.** Spreading 300 requests over 32 small pools leaves connections idle until the 1 s idle-close. One shared pool keeps 4 connections busy and warm. This is nginx's per-worker pool vs Pingora's shared pool.

**Q21.** 300 edge PoPs all MISS on the same new object at once. With tiered cache (regional parent) plus request collapsing, roughly how many requests reach your origin?
- A) 300
- B) 50
- C) 1
- D) 0, because the edges serve stale content

> ✅ **Answer: C.** Edges fetch from the parent, and the parent fetches from the origin once. "Never fetch the same object twice at once." 300 (A) is the case without a shield, a built-in thundering herd.

**Q22.** Which statement about multiplexing on the CDN → origin leg matches the lecture?
- A) Most CDN→origin hops are still HTTP/1.1 (`cloudflared`'s `http2Origin` defaults to false), so pooling is what actually saves you today
- B) All CDN→origin traffic is HTTP/3, so pooling is no longer needed
- C) Multiplexing is only used on the user-facing leg, never towards the origin
- D) HTTP/2 to the origin is on by default, which removes the 194 ms origin round trip

> ✅ **Answer: A.** The slide says "note the default": you have to turn `http2Origin` on. Multiplexing never removes the origin round trip (D); only a cache HIT does.

**Q23.** The WAF strips SQL comments before matching (`--waf full`). An attacker sends `q="' --"` and 5 rows still leak. What is the correct lesson?
- A) Add one more regex for `' --` and the WAF is complete
- B) A WAF matches spellings, not meanings; the real fix is bind parameters in the origin
- C) Normalisation should be turned off, because it causes false negatives
- D) The OWASP Core ruleset on the free plan would have blocked it

> ✅ **Answer: B.** "You never win it outright." The bug lives in the origin's string-formatted SQL, and `--safe` (bind parameters) makes every bypass return "no results". Also, OWASP Core needs Pro (D).

**Q24.** Limit: 100 requests per 10 s using a **fixed window**. A client sends 100 requests at t = 9.999 s and 100 more at t = 10.001 s. How many are allowed?
- A) 100
- B) 104
- C) 109
- D) 200

> ✅ **Answer: D.** Each burst falls in a different window, and each window sees only 100, so all 200 pass "in a blink". This is the weakness of Cloudflare's free plan, which only offers the fixed 10 s window.

**Q25.** Cloudflare's **sliding-window estimate** with a limit of 100 per 10 s. It is 2 s into the current window, the previous window had 80 requests and the current one has 50 so far. What is the estimate, and is the next request allowed?
- A) 66, allowed
- B) 130, refused
- C) 114, refused
- D) 90, allowed

> ✅ **Answer: C.** Weight the previous window by how much still overlaps: 8 s of 10 = 0.8. So 80 × 0.8 + 50 = 114 > 100 → refused. 66 (A) wrongly uses the elapsed fraction (0.2), and 130 (B) is the fixed-sum mistake.
