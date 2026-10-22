# 📓 Streaming Video at Large Scale
### *CN at Scaler — Lesson 8 (Session 8 of 8)* · "it starts with a live cricket scorecard"

> `polling` · `long poll` · `SSE` · `WebSockets` · `MQTT` · `Little's Law` · `304` · `TTL stacking` · `request collapsing` · `negative caching` · `HLS` · `segments` · `ABR` · `LL-HLS` · `multi-CDN` · `ffmpeg`
> **Every live stream is a scorecard with bigger bytes.**

---

## ⭐ THE HOOK (slide 1)

One live cricket match, watched on fifty million phones. The title slide shows three requests:

```
GET /match/412/score.json      200 ·   1,088 bytes · every 15 s
GET /stream_1/index.m3u8       200 ·     228 bytes · every 4 s
GET /stream_1/seg_000008.ts    200 · 823,440 bytes · once
```

Two are **small files polled on a timer** (the scorecard, the video playlist). One is a **big file fetched once** (a video segment).

> 💡 **Simple words mein:** Pull = dukaan ke bahar score ka board laga do, jo chahe aake padh le (cache). Push = har customer ke ghar ek aadmi bhejo jo score batata rahe: 50M aadmi chahiye!

---

## 📋 Table of contents

1. [The brief: one small file, fifty million times](#1--the-brief-one-small-file-fifty-million-times)
2. [Every way to deliver it: polling, long poll, SSE, WebSockets, MQTT](#2--every-way-to-deliver-it)
3. [What it costs: held sockets, Little's Law, four bills](#3--what-it-costs)
4. [Making polling survive: 304, TTLs, herds, collapsing, negative caching, diffs](#4--making-polling-survive)
5. [Same loop, bigger bytes: HLS](#5--same-loop-bigger-bytes-hls)
6. [Buffers and latency: buffer, latency budget, LL-HLS, spoilers, ABR](#6--buffers-and-latency)
7. [Many origins, many CDNs: steering, stale playlists, cache rules](#7--many-origins-many-cdns)
8. [Making the stream: two ffmpeg commands](#8--making-the-stream-two-ffmpeg-commands)
9. [Four ideas to keep](#9--four-ideas-to-keep)
10. [Assignment and homework (with solutions)](#10--assignment--homework-with-solutions)
11. [Cheat-sheet](#11--cheat-sheet)
12. [MCQs (mid → advanced)](#12--mcqs-mid--advanced)

---

## 1 · THE BRIEF: ONE SMALL FILE, FIFTY MILLION TIMES

### 1.1 The numbers (slide 4)
*(↩ Week 5 · gzip: 202,000 → 5,763 bytes)*

| Number | Meaning |
|---|---|
| **25–45 s** | Between balls. An over every 4–5 minutes |
| **50 M** | **Peak** concurrent viewers. **20 M on average** across a 4-hour match |
| **~1 KB** | Per scorecard: 4–5 KB of JSON, **about 1 KB gzipped** |
| **100** | Events a year, each one the same file downloaded again and again |

### 1.2 Two families of answers (slide 5)
*(↩ Week 5 · HTTP/1.0 hung up after every response)*

| NON-PERSISTENT: *the client asks* | PERSISTENT: *the server tells* |
|---|---|
| **Polling**: GET the same URL every few seconds. The connection may close in between | **Long poll**: a GET the server answers only when there's news |
| **Conditional GET**: 304 when nothing changed | **Server-Sent Events**: one response that never ends |
| **Caches**: nginx in front, then a CDN | **WebSockets**: HTTP upgrades into a two-way, framed pipe |
| **Any server**: no memory of who asked last time | **PubSub (MQTT)**: topics, a broker, fan-out to every subscriber |
| | **Plain binary socket**: your own protocol, like the Week 5 course project |
| ⬇ *Every request is an ordinary GET: stateless servers, cacheable answers, and a CDN can stand in front.* | ⬇ *Every viewer holds a connection for the whole match: stateful servers, and nothing a CDN can cache.* |

---

## 2 · EVERY WAY TO DELIVER IT

### 2.1 Polling: ask again every few seconds (slide 6)
*(↩ Week 7 · full jitter)*

```js
let etag = null;
const EVERY = 15_000;   // ms

async function poll() {
  const res = await fetch("/match/412/score.json", {
    headers: etag ? { "If-None-Match": etag } : {},
  });
  if (res.status === 200) {          // something changed
    etag = res.headers.get("ETag");
    render(await res.json());
  }                                   // 304 Not Modified: keep what we have
  const jitter = 0.8 + Math.random() * 0.4;   // ±20%
  setTimeout(poll, EVERY * jitter);
}
poll();
```

| Idea | What the slide says |
|---|---|
| **ETag + If-None-Match** | Nothing changed? **304 and an empty body.** *The request still happens.* |
| **Jitter** | **±20%**, so fifty million phones don't poll on the same second |
| **Freshness** | At worst **one poll interval plus every cache TTL on the path** |

### 2.2 Long polling: ask once, wait for news (slide 7)
*(↩ Week 3 · Little's Law: hold longer, hold more)*

```http
GET /match/412/score?after=1841 HTTP/1.1
   … the server holds the request open …
   … until ball 1842 is bowled, or 25 s pass

HTTP/1.1 200 OK
{"version":1842,"over":"15.4","total":"146/4"}

GET /match/412/score?after=1842 HTTP/1.1      ← … and ask again at once
```

| Point | Explanation |
|---|---|
| **A held request is a held socket** | Every viewer has one open almost all the time: **W is 25 s, not 30 ms** |
| **Beat every idle timeout** | Answer *before* nginx's `proxy_read_timeout` and the ALB idle timeout. **Both default to 60 s** (Week 4) |
| **It synchronises the herd** | A wicket completes **20M responses in one second**, and **20M new requests arrive in the next** (Week 7) |
| **It came back, standardised** | Low-Latency HLS's **blocking playlist reload** *is* long polling (§6.2) |

### 2.3 Server-Sent Events: one response that never ends (slide 8)
*(↩ Week 2 · framing: length or delimiter)*

```http
GET /match/412/events HTTP/1.1
Accept: text/event-stream
Last-Event-ID: 1841

HTTP/1.1 200 OK
Content-Type: text/event-stream

retry: 5000

id: 1842
event: ball
data: {"over":"15.4","runs":4,"total":"146/4"}

id: 1843
event: ball
data: {"over":"15.5","runs":0,"total":"146/4"}
```

| Point | Explanation |
|---|---|
| **Framing by delimiter** | An event ends at a **blank line**: "SMTP's lone dot in a different costume" |
| **Resume from a version** | On reconnect the browser sends **`Last-Event-ID`**, and the server replays only what was missed |
| **About six per browser** | Over **HTTP/1.1** each tab holds its own connection; **HTTP/2 multiplexes them onto one** (Week 5) |
| **One way only** | Server → client. Anything the phone says goes in ordinary requests |

### 2.4 WebSockets: leave HTTP after one exchange (slide 9)
*(↩ Week 5 · 101 Switching Protocols)*

```http
GET /live HTTP/1.1
Host: score.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- **The Accept value is `base64(SHA-1(key + a fixed GUID))`**: base64 again (Week 1).
- **Client frames are masked** so stray bytes can never look like a request to a caching proxy (Week 6).

**After the 101, every frame starts like this:**

| Field | Bytes | What it says |
|---|---|---|
| FIN · RSV · opcode | 1 | Last fragment? Text, binary, ping, pong, close |
| MASK · length | 1 | 0–125 is the length; **126 or 127 means "read more"** |
| Extended length | 0, 2 or 8 | A 16- or 64-bit length |
| Masking key | 0 or 4 | Present on every **client-to-server** frame |
| Payload | length | Your 183-byte scorecard diff |

> **A length, not a delimiter**: the other answer from Session 2. (SSE uses a delimiter; WebSockets use a length.)

### 2.5 MQTT: publish once, fan out to everyone (slide 10)
*(↩ Week 1 · type, length, value)*

```
phone                         broker
  CONNECT    client=phone-8812 keepalive=60   →
             ← CONNACK   accepted
  SUBSCRIBE  match/412/score qos=1            →
             ← SUBACK
             ← PUBLISH   match/412/score retain=1 {"v":1842,…}
  PINGREQ    C0 00   (two bytes, every keepalive) →
             ← PINGRESP  D0 00
```

**One PUBLISH from the scorer; the broker copies it to every subscriber of the topic.** Hotstar picked MQTT with the **EMQX** broker for exactly this fan-out.

| Feature | Meaning |
|---|---|
| **Retained message** | The broker keeps the last score and hands it to each new subscriber at once: *a cache, inside the broker* |
| **QoS 1 = at-least-once** | Duplicates happen. **Apply updates by version number and they become harmless** (Week 7) |
| **Remaining Length is a varint** | 7 bits per byte; the top bit says "more". *A type, a length, a value* |
| **Keepalives cost** | Only two bytes, but they **wake the phone's radio** and **keep a NAT mapping alive** |

### 2.6 Case study: how Hotstar held 50 million sockets (slide 11)
*(↩ Week 3 · from C10K to C10M)*

| Step | What they did | Result |
|---|---|---|
| **1 · One node** | EMQX on one `c5.4xlarge` | **250k** connections, latency in bounds |
| **2 · A cluster** | Publisher split from subscribers | **2M** subscribers. Mnesia wants **fewer than 10 nodes** |
| **3 · The ELB wall** | Load balancer limits | Drops at **~500k without pre-warming**; **4M per pre-warmed Classic ELB** |
| **4 · Five clusters** | 5 clusters × 8 subscribe nodes, bridged | **Past 10M** |
| **5 · Reverse bridge** | A Go service on each subscriber node **pulls** from the publisher | **200 nodes: 50M** |

> *"Caching at any level is also not possible."*
> Their **social feed** was tied to live moments, so it had to be pushed. **A scorecard is the opposite case: one file, identical for everyone.**

*Source: M. Chaudhari and P. Gupta, "Building Pubsub for 50M concurrent socket connections", Hotstar Engineering blog, 6 June 2019.*

---

## 3 · WHAT IT COSTS

### 3.1 What a held socket costs (slide 12)
*(↩ Week 3 · nothing is free per connection)*

| Cost | Detail |
|---|---|
| **Machines** | **Stateful**: each phone lives on one box for hours. **100k–200k connections per box** on the class board; Hotstar measured **250k**. **New boxes sit empty until someone reconnects**, so rebalancing means disconnecting people |
| **Memory** | Kernel socket buffers, TLS state, the broker's per-client process. **250k phones on a 32 GiB box ≈ 137 KB each, all in.** And the soft **`ulimit -n` still starts at 1,024** |
| **Network** | Keepalives with nothing to say: **at a 60 s keepalive, 20M idle phones send 333k pings a second** *before the first ball is bowled* |
| **Load balancers** | One address instead of 200 for DNS to hand out, and an even spread across boxes. **But the LB holds every socket too**: Hotstar hit 4M per Classic ELB |

> **Testing it:** one client IP has **28,232 ephemeral ports (32768–60999) per destination**, so **250k test sockets need at least 9 source IPs**. *Week 1: a port is 16 bits.*

### 3.2 Little's Law, twice (slide 13)
*(↩ Week 3 · L = λ × W)*

| POLLING THROUGH A CDN | PUSH OVER HELD SOCKETS |
|---|---|
| 1.67 M requests/s × about 0.03 s at the edge | W is the whole match: **four hours** |
| **= 50,000 requests in flight** | **= 50,000,000 sockets held**, ball or no ball |

> **Same audience, same scorecard: a thousand times the concurrency, because W went from milliseconds to hours.**

> 💡 **Simple words mein:** Bank mein har customer 30 second ruke toh hall khaali rehta hai; har koi 4 ghante baith jaaye toh hall bhar jaata hai, chahe aane wale utne hi hon.

### 3.3 Where the scorecard file lives (slide 14)
*(↩ Week 4 · nginx serves it from memory)*

```
 ① Your own servers          ② Object store, no cache          ③ Through a CDN
      Phones                       Phones                            Phones
        │                            │                                 │
       ELB                     S3 / R2 / GCS  ← Writer          CDN edge + mid-tier
        │                                                              │
  nginx × N, in memory        "Every poll is a GET                S3 / R2 / GCS  ← Writer
        │                      on the bucket."
    NFS / NAS                                                 "The edge absorbs the polls;
        │                                                      the bucket sees a few per TTL."
      Writer
```

### 3.4 The arithmetic before the architecture (slide 15)
*(↩ Week 3 · multiply before you choose)*

| Quantity | Formula | Result |
|---|---|---|
| Requests per match | 20M viewers × 14,400 s ÷ 30 s | **9.6 B** |
| Peak rate | 50M viewers ÷ 30 s | **1.67 M req/s** |
| Bytes per match | 9.6 B × 1 KB | **9.6 TB** |
| *Poll every 15 s* | | 19.2 B requests, 3.33 M req/s, 19.2 TB |
| *Push each ball* | one message every ~30 s per phone | about 9.6 B messages |
| × 100 events | | **960 B requests and 960 TB a year** |

### 3.5 Four ways to pay for one match (slide 16)
*(↩ Week 6 · a CDN is a rented reverse proxy)*

| Design | Per match | × 100 events | Mostly paying for | Breaks first |
|---|---|---|---|---|
| **S3 straight to phones** | $4.7K | $470K | GET fees ($3,840) | **5,500 GET/s per prefix** |
| **nginx fleet + ALB** | $1.2–1.8K | $120–180K | egress ($864) + load balancer units | **20–50M idle keep-alive sockets at the LB** |
| **CDN (CloudFront, India)** | **$12.5K** | **$1.25M** | **request fees: 92% of it** | nothing; it's just the bill |
| **Push: 200 × c5.4xlarge** | **$0.9–1.6K** | $90–160K | a peak-sized fleet ($680) + egress | reconnect storms, rebalancing |

> **At list price, push is the cheapest per match and the CDN the dearest. Hold that thought.**

*AWS list prices, Sep 2026 · S3, EC2, ELB: US East · CloudFront: India · body bytes only · poll every 30 s · push sends 183-byte diffs up to full cards.*

### 3.6 S3 runs out of requests before money (slide 17)
*(↩ Week 7 · 503 and Retry-After)*

| 5,500 | 1.67 M | 303× |
|---|---|---|
| GETs a second that one S3 prefix promises, *at least* | GETs a second that 50M phones polling every 30 s ask for | over, and **one key lives under one prefix** |

> *"While Amazon S3 is scaling to your new higher request rate, you may see some 503 (Slow Down) errors."*
> **Fifty million clients retrying a 503 without backoff is Session 7's retry storm, pointed at your own bucket.**

### 3.7 On a CDN, a 1 KB file is billed per request (slide 18)
*(↩ Week 5 · headers: 97.5% repeats)*

Per **million** responses (CloudFront India list prices: HTTPS **$0.0120 per 10,000 requests**; **$0.109/GB** after 1 free TB, **$0.072/GB** past 5 PB):

| Object | Request fee | Byte fee | Which dominates |
|---|---|---|---|
| Scorecard, 1 KB | **$1.20** | **$0.11** | **request fee: 92%** |
| 480p segment, 823 KB | $1.20 | **$59–90** | bytes |

```
   you pay per request  ◄──────────── break-even ≈ 11 KB ────────────►  you pay per byte
   playlist 228 B · scorecard 1 KB                         360p seg 459 KB · 720p 1.6 MB
```

### 3.8 "I prefer polling because I want to tell you how it's used for video too." (slide 19)
*(↩ Week 3 · HTTP is stateless, the rest aren't)*

| Cacheable | Stateless | Already paid for |
|---|---|---|
| One file, identical for fifty million people | Any server, any edge, any CDN can answer any phone | At list price the scorecard is **0.28% of the same match's video bill** |

---

## 4 · MAKING POLLING SURVIVE

### 4.1 Conditional GET: 304 Not Modified (slide 20)
*(↩ Week 5 · ETag: a fingerprint, not a clock)*

```http
GET /match/412/score.json HTTP/1.1
If-None-Match: "v1841"

HTTP/1.1 304 Not Modified
ETag: "v1841"
Cache-Control: public, max-age=10
```

| SAVES | DOES NOT SAVE | PAYS MOST BETWEEN CDN TIERS |
|---|---|---|
| The 1 KB body, the JSON parse, the re-render | **The request fee, the round trip, the headers** | Edge → mid-tier, mid-tier → origin: 304s instead of copies |

- **Make the ETag the scorecard's version number**: a *fingerprint, not a clock*.
- `If-Modified-Since` works too, **but only to the second**.

### 4.2 TTLs add up (slide 21)
*(↩ Week 6 · tiered cache: edge, then mid-tier)*

```
Every layer set to 30 s
┌─────────────────────┬─────────────────────┬─────────────────────┐
│  mid-tier TTL 30 s  │    edge TTL 30 s    │   client poll 30 s  │   → up to 90 s stale:
└─────────────────────┴─────────────────────┴─────────────────────┘     two or three balls behind the stadium

Each CDN TTL at half the promise, or less
┌────┬────┬─────────────┐
│ 5 s│ 5 s│  poll 15 s  │   → up to 25 s: inside one ball
└────┴────┴─────────────┘
```

> **Worst-case staleness = the poll interval + every cache TTL on the path.** Keep each CDN TTL at **½ of the freshness you promise or less**, and **refresh the edge before it expires**: a prefetch at ~25 s of a 30 s TTL (**Akamai calls it prefresh**).

> 💡 **Simple words mein:** Khabar courier ki tarah haath badalti hai: mid-tier → edge → phone. Har haath pe 30 s ruki, toh phone tak 90 s purani khabar pahunchegi.

### 4.3 The thundering herd at expiry (slide 22)
*(↩ Week 4 · proxy_cache_lock + use_stale)*

> At **10.000 s** the edge's copy expires. **Every request in that instant is a miss, and all of them head for the origin.**

| Defence | How |
|---|---|
| **Request collapsing** | One fetch per edge per object; everyone else waits in a queue |
| **Serve stale while updating** | Answer from the old copy while one request refreshes it (RFC 5861) |
| **Prefetch before expiry** | Refresh at ~25 s of a 30 s TTL, so no viewer meets an empty cache |
| **Jitter from the client** | Spread polls across the interval, not on the same second (Week 7) |

> 💡 **Simple words mein:** Chai khatam hui aur 100 log ek saath aaye. Smart chaiwala nayi chai sirf ek baar banata hai, baaki wait karte hain (collapsing), ya tab tak purani chai de deta hai (serve stale).

```nginx
proxy_cache_lock              on;
proxy_cache_use_stale         updating error timeout http_500 http_502 http_503 http_504;
proxy_cache_background_update on;
proxy_cache_valid 200         10s;
```

### 4.4 Request collapsing needs a shareable answer (slide 23)
*(↩ Week 5 · 103 Early Hints replaced push)*

```
 20,000 phones                     edge                                     origin
 ask one edge for score.json   →   sends 1 request upstream;          →     sees 1 request
 in the same 50 ms                 19,999 wait in a queue
```

| Make it public early | Early hints |
|---|---|
| **The first response decides.** `Cache-Control: public` releases every waiter at once. **A private answer, or a stray `Set-Cookie`, dequeues them one by one**, and **Fastly then skips collapsing for that object for 2 minutes** | Name the next thing before it exists: **HTTP 103** for pages. In **Low-Latency HLS**, a **preload hint** names the next piece, requests for it queue at the edge, and **one arrival releases them all** |

> **Check the default first:** **Cloudflare does not cache `.json` out of the box** (Week 6). Without a cache rule, `cf-cache-status` says **DYNAMIC** and every poll reaches you.

### 4.5 Negative caching: 404 and 5xx (slide 24)
*(↩ Week 7 · the live-edge 404)*

| 5XX · CACHE IT, BRIEFLY | 404 · ALMOST NEVER |
|---|---|
| One or two seconds of a cached 503 **shields a struggling origin from fifty million retries**. The edge answers; the origin sees **one probe per TTL per edge** | A 404 for a file that's **about to exist** keeps being served until its TTL runs out. *The next match's scorecard, the newest live segment.* **Cloudflare's default edge TTL for a 404 is 3 minutes** |

```nginx
proxy_cache_valid 200              10s;
proxy_cache_valid 404               1s;
proxy_cache_valid 500 502 503 504   2s;
```

> **Rule: a negative TTL must be far shorter than the time it takes the missing thing to appear.**

> 💡 **Simple words mein:** Sarkari office ke clerk ne "file nahi aayi" ka notice 3 minute ke liye laga diya. File 1 second baad aa gayi, par sabko 3 minute tak "nahi hai" hi sunne ko milega.

### 4.6 Send diffs, not the whole card (slide 25)
*(↩ Week 6 · Railgun · RFC 9842 deltas)*

| 4,117 B | 1,088 B | 183 B |
|---|---|---|
| the full scorecard JSON | the same card, gzipped | **one ball as a gzipped JSON Patch** |

*Measured on a synthetic T20 card: two innings, batting and bowling figures, fall of wickets.*

| Push: send the patch | Pull: version the file |
|---|---|
| SSE or MQTT carry the 183-byte patch. **A phone that missed one sees a version gap and fetches the full card once** | `?since=1841` gives every client its own URL and **the hit rate collapses**. **Publish one file with the version and the last few changes instead** |

### 4.7 A versioned scorecard is a playlist (slide 26)
*(↩ Week 2 · sequence numbers make streams resumable)*

```
score.json · polled every 15 s                   index.m3u8 · polled every 4 s
{                                                #EXTM3U
  "version": 1842,                               #EXT-X-TARGETDURATION:4
  "recent": [                                    #EXT-X-MEDIA-SEQUENCE:2
    {"v": 1839, "ball": "15.1", "runs": 1},      #EXTINF:4.000000,
    {"v": 1840, "ball": "15.2", "runs": 0},      seg_000002.ts
    …                                            …
    {"v": 1842, "ball": "15.4", "runs": 4}       #EXTINF:4.000000,
  ]                                              seg_000011.ts
}
```

| Scorecard | Playlist |
|---|---|
| `version` | `MEDIA-SEQUENCE` |
| `recent` changes | the last N segments |
| poll interval | `TARGETDURATION` |

> **A small index everyone polls, pointing at numbered pieces that never change. That design has a name: HTTP Live Streaming.**

---

## 5 · SAME LOOP, BIGGER BYTES: HLS

### 5.1 Same loop, bigger bytes (slide 27)
*(↩ Part 1 · the scorecard loop)*

| 1 KB | 228 B | 823 KB |
|---|---|---|
| scorecard, polled every 15 s | video playlist, polled every 4 s | one 4 s segment at 480p, **fetched once** |

> **DEMO:** Open *Big Buck Bunny* or *Tears of Steel* on **videojs.org** with **DevTools → Network** open. Watch the master playlist, then a media playlist, then segment after segment. **Throttle the network and watch it change rendition.** → Homework 1

### 5.2 HLS in one picture (slide 28)
*(↩ Week 2 · control and data want separating)*

```
 The player                                   ┌───────────────────────────────────────────────────────────┐
 1 Read the master once; pick a rendition ──► │ 1 · once        CONTROL   master.m3u8                     │
                                              │   The menu: 3 renditions with bandwidth, resolution, codecs│
 2 Reload its playlist every target     ──►   │ 2 · every 4 s   CONTROL   stream_1/index.m3u8             │
   duration                                   │   The last 10 segments, 4 s each; changes every 4 s       │
 3 Fetch each new segment into the buffer ──► │ 3 · each new one  DATA    stream_1/seg_000011.ts          │
                                              │   4 s of audio and video, 823 KB; never changes           │
 4 Play from the buffer; switch rendition     └───────────────────────────────────────────────────────────┘
   when throughput changes
```

> 💡 **Simple words mein:** Playlist = railway station ka display board (chhota, har 4 s update, sab dekhte hain). Segments = trains (badi, aur ek baar aa gayi toh badalti nahi).

### 5.3 The master playlist (slide 29)
*(↩ Week 7 · a codec fallback is not a retry)*

```m3u8
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-STREAM-INF:BANDWIDTH=3440800,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
stream_0/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1755600,RESOLUTION=854x480,CODECS="avc1.4d401f,mp4a.40.2"
stream_1/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=950400,RESOLUTION=640x360,CODECS="avc1.4d401e,mp4a.40.2"
stream_2/index.m3u8
```

*From the class ffmpeg run.* **`BANDWIDTH = (video + audio) × 1.1`**: (3000k + 128k) × 1.1 = **3,440,800**.

| Slide note | Meaning |
|---|---|
| **"Do not poll this again"** | It doesn't change during the match: **fetch once, cache for minutes** |
| **"Some people poll: switch CDNs"** | Re-reading the menu is how a player learns a new path. **Standardised later as Content Steering** (§7.1) |
| **"Falling back to another codec"** | `CODECS` lets the player **skip what it can't decode**, e.g., HEVC down to H.264. **A fallback, not a retry** |

### 5.4 The media playlist, polled (slide 30)
*(↩ Part 1 · the scorecard, every 4 s)*

```m3u8
# reload at t = 47.0 s                          # reload at t = 51.5 s
#EXT-X-TARGETDURATION:4                         #EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:1                         #EXT-X-MEDIA-SEQUENCE:2
#EXTINF:4.000000,                               #EXTINF:4.000000,
#EXT-X-PROGRAM-DATE-TIME:…T03:16:12.365+0000    #EXT-X-PROGRAM-DATE-TIME:…T03:16:16.365+0000
seg_000001.ts                                   seg_000002.ts
 … 8 more …                                      … 8 more …
#EXTINF:4.000000,                               #EXTINF:4.000000,
#EXT-X-PROGRAM-DATE-TIME:…T03:16:48.365+0000    #EXT-X-PROGRAM-DATE-TIME:…T03:16:52.365+0000
seg_000010.ts                                   seg_000011.ts
```
*RFC 8216 §6.3.4 · two reloads of the live 360p playlist, 4.5 s apart, trimmed.*

**The client's reload rules (RFC 8216):**

| Situation | Rule |
|---|---|
| **Changed?** | Wait **at least one target duration** before reloading again |
| **Unchanged?** | Wait **half a target duration**, then try again |
| **Same number, different URI?** | *"It SHOULD halt playback."* |

### 5.5 VOD is a playlist that stopped changing (slide 31)
*(↩ Week 6 · the query string is in the cache key)*

```m3u8
#EXTM3U
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:10,
1719543778001_4328560526001_s-1.ts?pubId=1719543778001&videoId=4328294689001
#EXTINF:10,
1719543778001_4328560526001_s-2.ts?pubId=1719543778001&videoId=4328294689001
 … 61 more, 10 s each …
#EXTINF:5,
1719543778001_4328560526001_s-64.ts?pubId=1719543778001&videoId=4328294689001
#EXT-X-ENDLIST
```

| `#EXT-X-ENDLIST` | 63 × 10 s + 5 s | `?pubId=…&videoId=…` |
|---|---|---|
| No more segments: **read it once, stop polling** | **10 s segments: old advice. Apple now says 6 s** | **Same for everyone, so it caches.** A per-viewer token wouldn't |

### 5.6 A segment is 188-byte packets (slide 32)
*(↩ Week 2 · why 140 bytes? why 188?)*

```
┌──────────────── 4-byte header ────────────────┬──────────── 184-byte payload ────────────┐
│ 0x47 · PID (13 bits) · counter                 │ a slice of video, of audio, or of the     │
│                                                │ PAT / PMT tables                          │
└────────────────────────────────────────────────┴───────────────────────────────────────────┘
```

| Rendition | One segment | 188-byte packets | Overhead |
|---|---|---|---|
| 720p | 1,613,416 B | 8,582 | 3.3% |
| 480p | 823,440 B | 4,380 | 4.1% |
| 360p | 452,328 B | 2,406 | 5.6% |

*Measured in the class run: every segment is an exact multiple of 188. Overhead = everything that isn't audio or video payload.*

| Point | Explanation |
|---|---|
| **Fixed length and a sync byte** | **Both of Session 2's framing answers at once.** 188 was chosen for **ATM** |
| **A 13-bit PID says whose packet it is** | Audio, video and tables **share one byte stream** (Week 5) |
| **fMP4 and CMAF segments are boxes instead** | size, type, payload. **TLV again** (Week 1) |

### 5.7 How long should a segment be? (slide 33)
*(↩ Week 2 · the round-trip tax)*

| Segment | Peak segment requests | Latency floor (3 ×) | On the class board |
|---|---|---|---|
| 2 s | 25 M/s | 6 s | |
| 3 s | 16.7 M/s | 9 s | "RPS?" |
| **4 s** | **12.5 M/s** | **12 s** | **the class ffmpeg run** |
| 6 s | 8.3 M/s | 18 s | **Apple's advice** |
| 30 s | 1.7 M/s | 90 s | "Too late!" |

| ADAPTIVE / HYBRID | THE TRADE |
|---|---|
| **1–2 s segments at start-up** for a fast first frame, longer ones later. ffmpeg knob: **`-hls_init_time 1 -hls_time 4`** | **Shorter**: more requests, more keyframes, more round trips. **Longer**: more latency, slower reaction to bandwidth |

### 5.8 Playlist traffic is harder than segment traffic (slide 34)
*(↩ Part 1 · the scorecard problem, again)*

| | Media playlist | Segment |
|---|---|---|
| **Size** | ~230 B gzipped | 450–1,600 KB |
| **Changes** | every 4 s | never |
| **Cache TTL** | ½ target duration or less | minutes to forever |
| **A stale copy** | stalls, or points at deleted files | harmless |
| **Billed by** | requests, like the scorecard | bytes |

> **Board: "Rendition: poll every second, RPS?"** At 1 s that's **50 M requests a second** at the peak. At the RFC's one target duration, **12.5 M**.

---

## 6 · BUFFERS AND LATENCY

### 6.1 The buffer (slide 35)
*(↩ Week 7 · the buffer is the stale cache)*

```
 ◄── played ──►│ seg 9 │ seg 10 │ seg 11 │ seg 12 … ░░░(downloading)░░░ │ live edge
               ▲
           playhead  ──── 12 s buffered ahead of it, next segment still downloading ────►
```

| Large buffer | One-segment buffer |
|---|---|
| Survives a bad minute on a train, but **costs latency**, and **memory** on low-end phones and TV sticks | **Lowest latency.** One slow segment and the picture stalls |

| Network bandwidth | Stale playlist |
|---|---|
| **More network, less buffer**: on a fast link a refill takes a fraction of a segment, so a small buffer is enough | When the playlist stops moving, **the buffer is what keeps the picture going** |

### 6.2 Latency is a budget (slide 36)
*(↩ Week 5 · server push died, Chrome 106)*

```
 encode   │ segment fills │ to CDN │       player hold-back: 3 × target duration
  ~2 s    │      4 s      │  ~2 s  │                    12 s
 ─────────┴───────────────┴────────┴────────────────────────────────────────────►  ≈ 20 s behind the stadium
```
*Encode and CDN times are typical guesses; the hold-back is the spec.*

**Low-Latency HLS (LL-HLS) shrinks the biggest bar with four ideas from the first half:**

| Idea | What it is | Earlier idea it reuses |
|---|---|---|
| **Parts** | **~1 s pieces of a segment**. Apple: at least **3× the P95 round trip** (Week 6) | — |
| **Blocking reload** | `?_HLS_msn=…` is **held until that segment exists** | **long polling** (§2.2) |
| **Preload hints** | **Name the next part early**. Replaced HTTP/2 push in **Jan 2020** | **early hints / collapsing** (§4.4) |
| **Delta updates** | `EXT-X-SKIP` drops old entries from the reply | **diffs, for playlists** (§4.6) |

> **With 1-second parts the 12 s hold-back becomes about 3 s:** Apple requires `PART-HOLD-BACK` of at least three parts.

### 6.3 The spoiler problem (slide 37)
*(↩ Part 1 · two polling loops, one clock)*

```
 0 s ─────────────────── ~10 s ───────────────────── ~20 s ──►
 wicket at the stadium   the scorecard says W          the video shows it
                         ◄────── spoiler window 10 s ──────►
```

**Fix: hold the score until the picture catches up.** Stamp each update with the ball's wall-clock time. The player knows the wall-clock time of every segment, so the app reveals an update only when playback passes it.

```text
# the score update
{"v": 1842, "ball": "15.4", "at": "2026-09-27T03:16:44.1Z"}

# the segment on screen
#EXT-X-PROGRAM-DATE-TIME:2026-09-27T03:16:44.365+0000
```

### 6.4 ABR: the player picks the next rendition (slide 38)
*(↩ Week 7 · who decides "worse"?)*

> *Last week: every fallback gives you something worse than ideal. When does worse stop being acceptable, and who decides?*
> **The player decides, at every segment, under a policy the product sets.**

| Throughput-based | Buffer-based |
|---|---|
| Measure how fast the last segments arrived; **pick the highest rendition that fits**. *Congestion control, one layer up* | **Watch the buffer**: nearly empty → step down; full → step up. Netflix's A/B tests: **10–20% fewer rebuffers** at a similar video rate |

> 💡 **Simple words mein:** Net slow hua toh player HD se SD pe aa jaata hai, jaise baarish mein cricket match chhota (DLS) kar dete hain: kam overs chalega, match rukna nahi chahiye.

> **The policy: a stall is worse than a lower rendition.** Even the default is policy: Apple suggests starting **cellular** viewers on the **730 kbps** variant and **Wi-Fi** viewers on **2 Mbps**.

*Huang, Johari, McKeown, Trunnell, Watson, "A Buffer-Based Approach to Rate Adaptation", SIGCOMM 2014 · Apple HLS authoring spec 1.32.*

---

## 7 · MANY ORIGINS, MANY CDNS

### 7.1 Steering: which CDN, which edge? (slide 39)
*(↩ Week 6 · Fastly's 49 minutes)*

```
                       ┌─► CDN A edge (primary)  ─► CDN A mid-tier (shields origin) ─► Origin 1 (packager A) ┐
 Android app ─ which? ─┤                                                                                    ├ same names
                       └─► CDN B edge (fallback) ─► CDN B mid-tier (shields origin) ─► Origin 2 (packager B) ┘
```

| Method | How |
|---|---|
| **DNS** | Hand out a CDN per region or ISP. **Slow to change: resolvers cache** |
| **Listed twice** | The master lists each rendition **once per CDN**, for stream failover |
| **Content Steering** | **A JSON file, polled every TTL, names the preferred CDN** |
| **Client failover** | Errors on one CDN **trip a circuit breaker**; the next segment tries the other (Week 7) |

### 7.2 Stale playlists and deleted segments (slide 40)
*(↩ Week 6 · stale-if-error, with a catch)*

1. ffmpeg's `delete_segments` keeps only **one** unreferenced segment. The class watched `seg_000000` **vanish 4 s after it left the playlist**.
2. **RFC 8216 says a removed segment MUST stay available for its duration plus the playlist's: 4 + 40 = 44 s here.**
3. During an origin blip **the CDN serves a stale playlist**. It still lists a segment the origin has deleted.
4. **404, negatively cached, and the player stalls.** *"Serve stale may cause player issues."*

```
# the class run, 360p rendition
t = 47.0 s   playlist: seg_000001 … seg_000010     on disk: seg_000000 … seg_000010
t = 51.5 s   playlist: seg_000002 … seg_000011     on disk: seg_000001 … seg_000011
```

**FIXES**
- **`-hls_delete_threshold 12`**
- **Playlist TTL at ½ target duration or less; serve it stale for one target duration at most.**
- **Segments never change: serve them stale freely.**
- **Apple: treat a playlist as invalid when `Last-Modified` is more than 3 target durations behind `Date`.**

### 7.3 Caching rules for a live stream (slide 41)
*(↩ Week 4 · location blocks: prefix vs. regex)*

| Object | TTL | Serve stale | 404 |
|---|---|---|---|
| `master.m3u8` | minutes | yes | 1 s |
| media playlist | **½ target duration or less** | **one target duration at most** | 1 s, or not at all |
| segment `.ts` | **long: it never changes** | yes | 1 s at most, at the live edge |
| `score.json` | ½ your promise or less | while updating | 1 s (5xx: 2 s) |

```nginx
location ~ \.m3u8$ { proxy_cache_valid 200 2s;  proxy_cache_lock on; proxy_cache_use_stale updating; }
location ~ \.ts$   { proxy_cache_valid 200 10m; proxy_cache_valid 404 1s; }
gzip_types application/vnd.apple.mpegurl application/json;
```

> **Also: gzip every playlist (Apple: MUST), and keep per-viewer tokens out of the cache key.**

---

## 8 · MAKING THE STREAM: TWO FFMPEG COMMANDS

### 8.1 From an mp4: the one-liner (slide 42)
*(↩ Week 4 · the honest benchmark)*

```bash
ffmpeg -re -i input.mp4 -codec: copy -hls_time 4 -hls_list_size 5 \
       -hls_playlist_type event -f hls index.m3u8
```

| Flag | What it does |
|---|---|
| `-re` | Read at native speed: **a file pretending to be a camera** |
| `-codec: copy` | **No re-encode**, so it can **only cut at existing keyframes** |
| `-hls_time 4` | **A target** segment length, *not a promise* |
| `-hls_list_size 5` | A 5-segment window, unless… |
| `-hls_playlist_type event` | …the playlist is **append-only** |

**WHAT RUNNING IT SHOWED**
- ❌ **8.33 s segments, not 4.** x264's default keyframe interval is **250 frames: 8.33 s at 30 fps**.
- ⚠️ **The window was ignored.** An **EVENT** playlist only grows: 11 segments listed at the end.
- **For real:** camera → **RTMP or SRT** → **transcoder** → **packager** → **origin** → **CDN**.

### 8.2 The ABR ladder, annotated (slide 43)
*(↩ Week 3 · amortise the setup)*

```bash
ffmpeg -re -stream_loop -1 -i input.mp4 \
  -filter_complex "[0:v] split=3[v720][v480][v360]; …scale + pad to 1280x720, 854x480, 640x360… ; \
                   [0:a]asplit=3[a0][a1][a2]" \
  -map "[v720o]" -map "[a0]" -map "[v480o]" -map "[a1]" -map "[v360o]" -map "[a2]" \
  -c:v libx264 -preset veryfast -profile:v main -sc_threshold 0 -g 120 -keyint_min 120 \
  -c:a aac -ac 2 -ar 48000 \
  -b:v:0 3000k -maxrate:v:0 3210k -bufsize:v:0 4500k -b:a:0 128k \
  -b:v:1 1500k -maxrate:v:1 1605k -bufsize:v:1 2250k -b:a:1 96k \
  -b:v:2  800k -maxrate:v:2  856k -bufsize:v:2 1200k -b:a:2 64k \
  -f hls -hls_time 4 -hls_list_size 10 \
  -hls_flags independent_segments+delete_segments+omit_endlist+program_date_time \
  -hls_segment_filename "out/stream_%v/seg_%06d.ts" -master_pl_name master.m3u8 \
  -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2" "out/stream_%v/index.m3u8"
```

The three annotations on the slide:
- **`split=3`: decode once, encode three times.**
- **`-g 120`: 4 s at 30 fps only.** A **24 fps** source gave **5 s** segments.
- **`-hls_flags`: a 40 s window that never ends, stamped with wall-clock time.**

---

## 9 · FOUR IDEAS TO KEEP (slide 45)

1. **Pull scales through caches; push scales through machines.** The same answer for everyone belongs in a CDN. A different answer per person means you're buying sockets.
2. **Number the pieces, then cache them forever.** A small changing index plus numbered pieces that never change: the versioned scorecard and the HLS playlist are one design.
3. **Latency is a budget you spend on safety.** Poll intervals, TTLs, segment length and hold-back all add up. Decide where each second goes.
4. **Know which bill you're on.** Below about 11 KB you pay per request, above it per byte. Scorecards and playlists are the first kind; segments the second.

---

## 10 · ASSIGNMENT & HOMEWORK (WITH SOLUTIONS)

### 10.1 Assignment: "a match on your laptop" (slide 46)
*(↩ Weeks 5–6 · build it, then measure it honestly)*

```
 1,000 simulated viewers  ──►  mid-tier: an nginx cache in front  ──►  origin: score.json and the ffmpeg ladder
```

Tasks: **(1) Collapsing** — origin requests/s for `score.json` with `proxy_cache_lock` off, then on. **(2) Staleness** — for each ball, *seen minus written*, at a 30 s TTL and at 5 s. **(3) Break it** — serve a stale playlist with `hls_delete_threshold 1`, catch the 404s, fix it. **(4) Spoiler** — show the score ahead of the picture, then hold updates until `PROGRAM-DATE-TIME` passes them. **Deliverable:** code + a one-page write-up with the numbers.

**✅ Solution (easiest setup: 4 small files in one folder + one nginx config)**

**Step 0 · test video and limits** (1,000 sockets need more than the default fd limit):
```bash
ulimit -n 8192        # in every terminal you use below
ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=30 -f lavfi -i sine -t 120 -pix_fmt yuv420p input.mp4
mkdir -p match/412 out/stream_1
```

**`scorer.py`** — writes a new ball every 25–45 s, stamped with the time it was written:
```python
import json, os, random, time
v = 0
while True:
    v += 1
    with open("match/412/score.tmp", "w") as f:
        json.dump({"v": v, "written_at": time.time()}, f)
    os.replace("match/412/score.tmp", "match/412/score.json")   # atomic swap
    time.sleep(random.uniform(25, 45))
```

**`origin.py`** — a slow (200 ms) origin, so concurrent misses overlap; it logs every request to stderr:
```python
import http.server, time
class H(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        time.sleep(0.2)
        super().do_GET()
class S(http.server.ThreadingHTTPServer):
    request_queue_size = 1024
S(("", 8080), H).serve_forever()
```

**`mid.conf`** — the mid-tier nginx cache on port 8081:
```nginx
events { worker_connections 4096; }
http {
  proxy_cache_path /tmp/midcache keys_zone=mid:10m inactive=10m;
  server {
    listen 8081;
    location = /match/412/score.json {
      proxy_pass http://127.0.0.1:8080;
      proxy_cache mid;
      proxy_cache_valid 200 5s;           # task 2: run with 30s, then 5s
      proxy_cache_lock on;                # task 1: "off" run = comment out this AND the next line
      proxy_cache_use_stale updating;
    }
    location ~ \.m3u8$ { proxy_pass http://127.0.0.1:8080; proxy_cache mid; proxy_cache_valid 200 2s; }
    location ~ \.ts$   { proxy_pass http://127.0.0.1:8080; proxy_cache mid; proxy_cache_valid 200 10m; proxy_cache_valid 404 1s; }
    location /         { proxy_pass http://127.0.0.1:8080; }
  }
}
```
Start / reload: `nginx -c "$PWD/mid.conf"` and `nginx -c "$PWD/mid.conf" -s reload`.

**`viewers.py`** — 1,000 pollers; prints *seen minus written* per ball:
```python
import json, random, sys, threading, time, urllib.request
URL = "http://127.0.0.1:8081/match/412/score.json"
EVERY, JITTER = float(sys.argv[1]), sys.argv[2] == "jitter"
seen = {}                                   # version -> list of delays (s)

def viewer():
    last = 0
    while True:
        try:
            d = json.load(urllib.request.urlopen(URL))
        except Exception:                   # a 502 or a reset must not kill the viewer
            time.sleep(1); continue
        if d["v"] > last:
            last = d["v"]
            seen.setdefault(last, []).append(time.time() - d["written_at"])
        time.sleep(EVERY * (random.uniform(0.8, 1.2) if JITTER else 1))

for _ in range(1000):
    threading.Thread(target=viewer, daemon=True).start()
while True:
    time.sleep(30)
    for v, ds in sorted(seen.items())[-3:]:
        ds = sorted(ds)
        print(f"ball v{v}: median {ds[len(ds)//2]:.1f}s  max {ds[-1]:.1f}s")
```

Run (separate terminals): `python3 scorer.py` · `python3 origin.py 2> origin.log` · `nginx -c "$PWD/mid.conf"` · `python3 viewers.py 15 nojitter`

| Task | What to do | What you should see / write down |
|---|---|---|
| **1 · Collapsing** | `python3 viewers.py 15 nojitter`. Count origin hits per second: `awk '/score.json/ {print $5}' origin.log \| uniq -c` | Lock **off** (and `use_stale` off): a spike of **hundreds (up to ~1,000)** origin requests in the second of each burst. Lock **on**: **1 request per expiry** |
| **2 · Staleness** | `python3 viewers.py 15 jitter`, once with `proxy_cache_valid 200 30s`, once with `5s` (ignore the first ball) | Max ≈ poll + TTL: **≈ 45 s** at 30 s TTL vs **≈ 20 s** at 5 s TTL. Matches "staleness = poll + every TTL" (§4.2) |
| **3 · Break it** | See below | Stale playlist → **404s**; after the fix → **all 200** |
| **4 · Spoiler** | See below | "Raw" score changes ~10–20 s before the wicket is on screen; "held" score changes with the picture |

**Task 3 · Break it.** Start a 1-rendition live stream on the origin (delete threshold 1 = ffmpeg's default), and make the playlist stale by caching it for 30 s (change `.m3u8` to `proxy_cache_valid 200 30s;`):
```bash
ffmpeg -re -stream_loop -1 -i input.mp4 -c:v libx264 -preset veryfast -g 120 -keyint_min 120 -sc_threshold 0 -c:a aac \
  -f hls -hls_time 4 -hls_list_size 10 -hls_delete_threshold 1 \
  -hls_flags delete_segments+omit_endlist+program_date_time \
  -hls_segment_filename "out/stream_1/seg_%06d.ts" out/stream_1/index.m3u8
```
`check.py` — reads the playlist through the cache, asks the origin for every listed segment (what any cache miss would do):
```python
import time, urllib.request, urllib.error
while True:
    pl = urllib.request.urlopen("http://127.0.0.1:8081/out/stream_1/index.m3u8").read().decode()
    for seg in [l for l in pl.splitlines() if l.endswith(".ts")]:
        try:
            urllib.request.urlopen("http://127.0.0.1:8080/out/stream_1/" + seg); code = 200
        except urllib.error.HTTPError as e:
            code = e.code
        print(time.strftime("%X"), seg, code)
    time.sleep(4)
```
Result: the oldest ~6 entries of the 30-s-stale playlist print **404** (already deleted). **Fix:** restart ffmpeg with `-hls_delete_threshold 12` (12 × 4 s = 48 s ≥ 4 + 40 = 44 s) and set the `.m3u8` TTL back to `2s` → every line prints **200**. Put both logs in the write-up.

**Task 4 · Spoiler.** Keep the Task 3 stream running. Save as `spoiler.html` in the same folder, open `http://127.0.0.1:8081/spoiler.html`:
```html
<!doctype html>
<video id="v" controls muted autoplay width="640"></video>
<p>Raw (spoiler): <b id="raw"></b> &nbsp; Held (synced): <b id="held"></b></p>
<script src="https://cdn.jsdelivr.net/npm/hls.js@1.5.13"></script>
<script>
const hls = new Hls(); hls.loadSource("/out/stream_1/index.m3u8"); hls.attachMedia(v);
const queue = [];
setInterval(async () => {
  const s = await (await fetch("/match/412/score.json", {cache: "no-store"})).json();
  raw.textContent = "v" + s.v;
  if (!queue.length || queue.at(-1).v < s.v) queue.push(s);
  const onScreen = hls.playingDate;          // wall-clock time (PROGRAM-DATE-TIME) of the frame on screen
  while (onScreen && queue.length && queue[0].written_at * 1000 <= onScreen.getTime())
    held.textContent = "v" + queue.shift().v;
}, 1000);
</script>
```
Write down: when `scorer.py` writes a new ball, **Raw** changes within one poll, **Held** changes only when the video reaches that moment (the gap = the spoiler window, ~10–20 s).

### 10.2 Homework (slide 47)

**HW 1 · Play the video.js demo with DevTools open: log two minutes of requests, then throttle and catch a rendition switch.**

> ✅ **Solution:** videojs.org → HLS demo (*Big Buck Bunny*) → DevTools → **Network**, filter `m3u8`, tick **Preserve log**, press play. You see `master.m3u8` **once**, then the media playlist, then segments one after another. After ~2 min, throttle **Slow 4G**: within a few segments the segment URLs move to a **lower-rendition** folder/name — that is the rendition switch. Save as HAR and write down: time of the switch, the URL before and after, and their `BANDWIDTH` from the master.

**HW 2 · Re-run the four bills for a 15 s poll, and for a negotiated $0.02/GB with no request fee. Which design wins now?**

> ✅ **Solution:** a 15 s poll doubles requests and bytes (19.2 B requests, 19.2 TB).
>
> | Design | 30 s poll (slide) | **15 s poll** | **CDN at $0.02/GB, no request fee** |
> |---|---|---|---|
> | S3 | $4.7K | 19.2 B × $0.0004/1k = $7,680 + egress 19.2 TB × $0.09 = $1,728 → **≈ $9.4K** | — |
> | nginx + ALB | $1.2–1.8K | egress doubles (+$864) → **≈ $2–2.7K** | — |
> | CDN | $12.5K | 19.2 B × $0.012/10k = $23,040 + ~$2K bytes → **≈ $25K** | 30 s: 9,600 GB × $0.02 = **$192**; 15 s: **$384** |
> | Push | $0.9–1.6K | **unchanged** (it sends per ball, not per poll) | — |
>
> **15 s poll → push wins (by even more).** **Negotiated $0.02/GB, no request fee → the CDN wins by far** ($192–384 per match). Lesson: the CDN cost of a small file is a request-fee problem, and that fee is negotiable.

**HW 3 · Serve the scorecard over SSE on HTTP/1.1, open seven tabs, and explain the one that waits. Then fix it with HTTP/2.**

> ✅ **Solution:** `sse.py` (open `http://localhost:8080/` in 7 tabs):
> ```python
> import http.server, json, time
> PAGE = b'<pre id=o></pre><script>new EventSource("/match/412/events").addEventListener("ball", e => o.textContent += e.data + "\\n")</script>'
> class H(http.server.BaseHTTPRequestHandler):
>     def do_GET(self):
>         if self.path != "/match/412/events":
>             self.send_response(200); self.send_header("Content-Type", "text/html"); self.end_headers()
>             self.wfile.write(PAGE); return
>         self.send_response(200)
>         self.send_header("Content-Type", "text/event-stream"); self.send_header("Cache-Control", "no-cache")
>         self.end_headers()
>         v = int(self.headers.get("Last-Event-ID", 1841))       # resume from a version
>         while True:
>             v += 1
>             self.wfile.write(f'id: {v}\nevent: ball\ndata: {json.dumps({"v": v})}\n\n'.encode())
>             time.sleep(3)
> http.server.ThreadingHTTPServer(("", 8080), H).serve_forever()
> ```
> **Why tab 7 waits:** over HTTP/1.x the browser opens **about six connections per host**, and each SSE tab keeps one busy forever, so tab 7 sits in the queue (it loads the moment you close another tab).
> **Fix with HTTP/2** (browsers need TLS for it) — nginx in front, all tabs share **one** connection:
> ```bash
> openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 7 -subj /CN=localhost
> ```
> ```nginx
> events {}
> http { server {
>   listen 8443 ssl;  http2 on;                       # nginx ≥ 1.25.1
>   ssl_certificate /ABS/PATH/cert.pem;  ssl_certificate_key /ABS/PATH/key.pem;
>   location / { proxy_pass http://127.0.0.1:8080; proxy_buffering off; }
> } }
> ```
> Open `https://localhost:8443/` in 7+ tabs: all of them stream; DevTools → Protocol column shows **h2**.

**HW 4 · Read Hotstar's pubsub post and map each of its five steps to a Week 3 or Week 4 idea.**

> ✅ **Solution:**
>
> | Hotstar step | Week 3 / 4 idea |
> |---|---|
> | 1 · One node, 250k | **W3** C10K → C10M: per-connection cost, fd limits (`ulimit -n`) |
> | 2 · Publisher split from subscribers, Mnesia < 10 nodes | **W3** scale out, but cluster membership has a size limit |
> | 3 · ELB wall (500k cold, 4M pre-warmed) | **W4** the load balancer / reverse proxy holds every socket too |
> | 4 · Five clusters, bridged → 10M | **W3** horizontal sharding into bounded-size clusters |
> | 5 · Reverse bridge, subscribers pull → 50M | **W3/W4** fan-out: the publisher serves 200 nodes, not 50M clients (pull from upstream) |

**HW 5 ⭐ (the one to do if you do only one) · Run the ladder on a 24 fps file and prove from the playlist that the segments are 5 s. Switch to time-based keyframes (`force_key_frames`) and prove they're 4 s.**

> ✅ **Solution:**
> ```bash
> ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=24 -f lavfi -i sine -t 120 -pix_fmt yuv420p in24.mp4
> mkdir -p out/stream_0 out/stream_1 out/stream_2
> ffmpeg -re -stream_loop -1 -i in24.mp4 \
>   -filter_complex "[0:v]split=3[a][b][c];[a]scale=1280:720[v720o];[b]scale=854:480[v480o];[c]scale=640:360[v360o];[0:a]asplit=3[a0][a1][a2]" \
>   -map "[v720o]" -map "[a0]" -map "[v480o]" -map "[a1]" -map "[v360o]" -map "[a2]" \
>   -c:v libx264 -preset veryfast -profile:v main -sc_threshold 0 -g 120 -keyint_min 120 \
>   -c:a aac -ac 2 -ar 48000 -b:v:0 3000k -b:v:1 1500k -b:v:2 800k \
>   -f hls -hls_time 4 -hls_list_size 10 -hls_flags independent_segments+delete_segments+omit_endlist+program_date_time \
>   -hls_segment_filename "out/stream_%v/seg_%06d.ts" -master_pl_name master.m3u8 \
>   -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2" "out/stream_%v/index.m3u8"
> # after ~30 s, in another terminal:
> grep EXTINF out/stream_0/index.m3u8          # → #EXTINF:5.000000,   (120 frames ÷ 24 fps = 5 s)
> ```
> Now stop it, replace `-g 120 -keyint_min 120` with `-force_key_frames "expr:gte(t,n_forced*4)"` (keep `-sc_threshold 0`), delete `out/`, run again:
> ```bash
> grep EXTINF out/stream_0/index.m3u8          # → #EXTINF:4.000000,   (a keyframe every 4 s of time, any fps)
> ```
> Write-up: `-g` counts **frames**, `-hls_time` counts **seconds**, so `-g 120` is 4 s only at 30 fps; time-based keyframes fix it.

---

## 11 · CHEAT-SHEET

| Term | One-line meaning |
|---|---|
| **The brief** | ~1 KB gzipped scorecard, 50M peak / 20M avg viewers, a ball every 25–45 s, 100 events/yr |
| **Pull vs. push** | pull = stateless, cacheable, scales through caches; push = one held socket per viewer, scales through machines |
| **Polling** | GET every N s, `If-None-Match` → 304, **±20% jitter**; staleness ≤ poll + all TTLs |
| **Long poll** | request held until news or 25 s; answer before the 60 s nginx/ALB idle timeouts; synchronises the herd |
| **SSE** | one never-ending `text/event-stream` response; events end at a blank line; `Last-Event-ID` resume; ~6 per browser on HTTP/1.1 |
| **WebSocket** | `Upgrade` → `101`; `Accept = base64(SHA-1(key + GUID))`; length-prefixed frames; client frames masked |
| **MQTT** | broker fan-out; retained message = cache in the broker; QoS 1 = at-least-once → apply by version; varint length; PINGREQ `C0 00` |
| **Hotstar** | EMQX, 250k/node → 2M → ELB wall (4M pre-warmed) → 5 clusters → reverse bridge, 200 nodes = 50M |
| **Held socket cost** | ~137 KB each; 100–250k per box; 333k keepalives/s at 20M × 60 s; `ulimit -n` 1,024; 28,232 ports per client IP per destination |
| **Little's Law** | L = λ × W: polling 1.67M/s × 0.03 s = 50k in flight; push = 50M held (W = 4 h) |
| **The arithmetic** | 9.6 B req/match, 1.67 M req/s peak, 9.6 TB (30 s poll) |
| **Four bills** | S3 $4.7K · nginx+ALB $1.2–1.8K · CDN $12.5K (92% request fees) · push $0.9–1.6K |
| **S3 limit** | 5,500 GET/s per prefix vs. 1.67M needed = 303× → 503 Slow Down |
| **Break-even 11 KB** | below: billed per request (scorecard, playlist); above: per byte (segments) |
| **Why polling anyway** | cacheable, stateless, 0.28% of the video bill you already pay |
| **ETag** | use the version number: a fingerprint, not a clock; `If-Modified-Since` only to the second |
| **TTL stacking** | staleness = poll + every TTL; each CDN TTL ≤ ½ the promise; prefetch/prefresh at ~25 of 30 s |
| **Herd at expiry** | collapsing (`proxy_cache_lock`) · stale-while-updating · prefetch · client jitter |
| **Collapsing needs public** | private / `Set-Cookie` → waiters released one by one; Fastly skips collapsing for 2 min |
| **Cloudflare** | doesn't cache `.json` by default → `cf-cache-status: DYNAMIC` |
| **Negative caching** | 5xx: cache 1–2 s (shield the origin); 404: almost never (Cloudflare default 3 min!); negative TTL ≪ time-to-appear |
| **Diffs** | 4,117 B → 1,088 B gzipped → 183 B JSON Patch; pull: one file with version + recent changes, never `?since=` |
| **HLS** | master (menu, once) → media playlist (polled every target duration) → segments (never change) |
| **BANDWIDTH** | (video + audio) × 1.1 |
| **Reload rules** | changed: wait ≥ 1 target duration; unchanged: wait ½; same number, different URI: halt |
| **VOD** | `#EXT-X-ENDLIST` = stop polling; Apple now says 6 s segments |
| **MPEG-TS** | 188-byte packets (for ATM), sync `0x47`, 13-bit PID, PAT/PMT; 3–6% overhead; fMP4/CMAF = boxes (TLV) |
| **Segment length** | shorter = more RPS + keyframes; longer = latency; floor = 3 × segment; `-hls_init_time 1` for fast start |
| **Playlist vs. segment** | playlist: tiny, changes, short TTL, stale = dangerous, billed per request; segment: big, never changes, cache forever, per byte |
| **Buffer** | the stale cache; bigger = survives bad network but adds latency |
| **Latency budget** | encode ~2 + segment 4 + CDN ~2 + hold-back 12 ≈ 20 s |
| **LL-HLS** | parts (~1 s) · blocking reload (`_HLS_msn`, long poll) · preload hints (replaced H2 push, Jan 2020) · delta updates (`EXT-X-SKIP`) → ~3 s hold-back |
| **Spoiler fix** | stamp updates with `at`; reveal when playback's `PROGRAM-DATE-TIME` passes it |
| **ABR** | throughput-based (highest that fits) vs. buffer-based (10–20% fewer rebuffers); policy: a stall is worse than lower quality |
| **Multi-CDN** | DNS (slow), listed twice, Content Steering (polled JSON), client failover (circuit breaker) |
| **Stale playlist bug** | stale index → deleted segment → cached 404 → stall; fix: `-hls_delete_threshold 12`, playlist TTL ≤ ½ TD, stale ≤ 1 TD |
| **Removed segment** | MUST stay for its duration + playlist duration (4 + 40 = 44 s) |
| **Cache rules** | master: minutes · playlist: ≤ ½ TD · `.ts`: long · `score.json`: ≤ ½ promise; gzip playlists; no per-viewer tokens in keys |
| **ffmpeg one-liner** | `-codec: copy` cuts only at existing keyframes → 8.33 s (x264 default 250 frames @ 30 fps); EVENT playlists ignore the window |
| **Ladder** | `split=3` decode once, encode three times; `-g 120` = 4 s only at 30 fps (24 fps → 5 s); `force_key_frames` for time-based |
| **Four ideas** | pull via caches / push via machines · number pieces, cache forever · latency is a budget · know your bill |

---

## 12 · MCQs (mid → advanced)

**Q1.** 50M phones poll the scorecard every 30 s through a CDN, and each request spends about 0.03 s at the edge. Using Little's Law, how many requests are in flight at the peak?
- A) 1.67 million
- B) 50,000
- C) 50 million
- D) 500

> ✅ **Answer: B.** L = λ × W = 1.67M req/s × 0.03 s ≈ 50,000. 1.67M is the arrival rate (λ), not the number in flight.

**Q2.** The team switches from polling to push for the same 50M audience. Each phone now gets only one message per ball (~every 30 s). What happens to concurrency?
- A) It falls to about 25,000, because there are fewer messages than polls
- B) It stays at about 50,000, because the audience is the same
- C) It rises to 50 million held sockets, because W becomes the whole 4-hour match
- D) It rises to 9.6 billion, the number of messages per match

> ✅ **Answer: C.** With push, every viewer holds a socket for the whole match, so W goes from milliseconds to hours and L = the whole audience, "ball or no ball". Fewer messages lowers the rate, not the time each socket is held.

**Q3.** 20M average viewers (4-hour match, 50M peak) poll a 1 KB scorecard every **15 s** instead of 30 s. Requests per match and peak rate?
- A) 19.2 B requests per match; 3.33 M req/s at the peak
- B) 9.6 B requests per match; 1.67 M req/s at the peak
- C) 19.2 B requests per match; 1.67 M req/s at the peak
- D) 48 B requests per match; 3.33 M req/s at the peak

> ✅ **Answer: A.** 20M × 14,400 s ÷ 15 s = 19.2 B, and 50M ÷ 15 s = 3.33 M/s. Option C forgets that the peak rate also doubles; D wrongly uses the 50M peak for the per-match total.

**Q4.** 20M idle phones hold MQTT connections with a **30 s** keepalive (instead of the slide's 60 s). How many pings per second reach the brokers before the first ball?
- A) 333,000
- B) 20 million
- C) 6.7 million
- D) about 667,000

> ✅ **Answer: D.** 20M ÷ 30 s ≈ 667k pings/s, twice the slide's 333k at 60 s. Each ping is only 2 bytes, but it still wakes the radio and costs the server work.

**Q5.** A long-poll endpoint holds requests for up to 25 s before sending an empty "no news" reply. Why 25 s and not, say, 90 s?
- A) To answer before nginx's `proxy_read_timeout` and the ALB idle timeout, which both default to 60 s
- B) Because browsers close any HTTP request after 30 s
- C) Because a cricket ball is bowled every 25 s, so news always arrives in time
- D) To stay below the 6-connections-per-browser limit

> ✅ **Answer: A.** The slide's rule is "beat every idle timeout": a middlebox would kill a request held past 60 s. Balls come every 25–45 s, so C is not guaranteed.

**Q6.** You serve the scorecard over SSE on HTTP/1.1 and open it in seven tabs of one browser. Six tabs update; the seventh just spins. What is happening, and what fixes it?
- A) The server only allows six `Last-Event-ID` values; raise the limit
- B) SSE has a six-event buffer; send smaller events
- C) The seventh tab is blocked by CORS; add an `Access-Control-Allow-Origin` header
- D) The browser allows about six HTTP/1.1 connections per host and each SSE stream holds one; HTTP/2 multiplexes all of them onto one connection

> ✅ **Answer: D.** Each SSE tab keeps its connection busy forever, so tab 7 waits for a free one. HTTP/2 puts all streams on one connection, which is the slide's fix.

**Q7.** A browser sends a 300-byte text message over a WebSocket. How is the length written, and how big is the frame header?
- A) The 7-bit length field holds 300 directly; 2-byte header
- B) The 7-bit field says 126, then a 2-byte extended length (300), plus a 4-byte masking key: 8-byte header
- C) The 7-bit field says 127, then an 8-byte extended length, no mask: 10-byte header
- D) No length: the frame ends at a blank line, like SSE

> ✅ **Answer: B.** 0–125 fits in the 7-bit field; 126 means "read a 16-bit length", which covers 300. Client-to-server frames always carry a 4-byte masking key: 1 + 1 + 2 + 4 = 8 bytes. WebSockets frame by length, SSE by delimiter.

**Q8.** With MQTT QoS 1, a phone sometimes gets ball 1842 twice, and a phone that subscribes mid-over sees the score at once instead of waiting for the next ball. Which pair explains both?
- A) QoS 1 is exactly-once; the broker polls the scorer on subscribe
- B) QoS 1 is at-most-once; the phone caches the last score locally
- C) QoS 1 is at-least-once, so apply updates by version number; a retained message is handed to each new subscriber
- D) Keepalive pings resend old messages; the CDN serves the first score

> ✅ **Answer: C.** At-least-once means duplicates happen, and applying by version makes them harmless. The retained message is "a cache, inside the broker".

**Q9.** After adding `ETag`/`If-None-Match`, 95% of scorecard polls get `304 Not Modified`. What happens to the CloudFront bill for the scorecard?
- A) It falls by about 95%, since almost no bodies are sent
- B) It barely changes: the request fee (92% of the bill) is still paid on every 304
- C) It doubles, because a 304 costs two requests
- D) It drops to zero, because 304s are free on a CDN

> ✅ **Answer: B.** A 304 saves the body, parse and re-render but not the request fee, round trip or headers. For a 1 KB file the request fee dominates; 304s pay most between CDN tiers.

**Q10.** Mid-tier TTL = 10 s, edge TTL = 10 s, and clients poll every 20 s. What is the worst-case staleness of the score on a phone?
- A) 10 s
- B) 20 s
- C) 30 s
- D) 40 s

> ✅ **Answer: D.** Worst-case staleness = poll interval + every cache TTL on the path = 20 + 10 + 10 = 40 s. Picking only the largest value (20 s) is the common mistake.

**Q11.** Twenty thousand phones hit one Fastly edge for `score.json` in the same 50 ms, but the origin still sees thousands of requests. The response carries a `Set-Cookie` header. Why?
- A) A private answer or stray `Set-Cookie` can't be shared, so waiters are dequeued one by one, and Fastly then skips collapsing for that object for 2 minutes
- B) `Set-Cookie` makes the response bigger than 11 KB, so it is billed per byte
- C) Request collapsing only works for `.ts` segments
- D) Fastly does not cache JSON by default

> ✅ **Answer: A.** Collapsing needs a shareable first response (`Cache-Control: public`). D describes Cloudflare's default for `.json`, not Fastly's collapsing behaviour.

**Q12.** Which origin cache setting best follows the lecture's negative-caching rules for a live stream?
- A) `404` cached 3 min, `5xx` not cached
- B) `404` cached 1 min, `5xx` cached 1 min
- C) `404` cached 1 s, `500 502 503 504` cached 2 s
- D) `404` cached 10 s, `5xx` never cached

> ✅ **Answer: C.** Cache 5xx briefly (1–2 s) to shield a struggling origin; 404 almost never, because the missing file (the newest segment) is about to exist. A is Cloudflare's dangerous 3-minute 404 default.

**Q13.** To send only missed balls, a developer changes clients to poll `score.json?since=<their version>`. The CDN hit rate collapses. What is the lecture's fix?
- A) Turn off the CDN and poll S3 directly
- B) Add `Cache-Control: private` so each client gets its own copy
- C) Increase the edge TTL to 10 minutes
- D) Publish one file for everyone with the current version and the last few changes

> ✅ **Answer: D.** `?since=` gives every client its own URL, so every poll is a miss. One shared file with version + recent changes keeps a single cached object. That is exactly a sliding-window playlist.

**Q14.** A player reloads a live media playlist with `#EXT-X-TARGETDURATION:4` and finds it **unchanged**. Per RFC 8216, when should it try again?
- A) Immediately
- B) After 2 s (half a target duration)
- C) After 4 s (one target duration)
- D) After 12 s (three target durations)

> ✅ **Answer: B.** Unchanged → wait half a target duration; changed → wait at least one target duration. 12 s is the player's hold-back, not a reload rule.

**Q15.** A rendition has 2,000 kbps video and 128 kbps audio. Using the lecture's rule, what `BANDWIDTH` should the master playlist show?
- A) 2,128,000
- B) 2,200,000
- C) 2,340,800
- D) 2,560,000

> ✅ **Answer: C.** BANDWIDTH = (video + audio) × 1.1 = 2,128,000 × 1.1 = 2,340,800. A forgets the 1.1 factor.

**Q16.** A 480p MPEG-TS segment arrives as 823,500 bytes (the healthy one is 823,440). What is the best conclusion?
- A) It is not an exact multiple of 188 bytes, so the segment is damaged or has extra bytes; a good segment is whole 188-byte packets that each start with `0x47`
- B) Normal: overhead varies between 3% and 6%
- C) Normal: TS packets can be 188 or 248 bytes
- D) The PID field grew, which is expected for 480p

> ✅ **Answer: A.** 823,440 = 4,380 × 188; 823,500 leaves 60 bytes over. Every segment in the class run was an exact multiple of 188. Overhead (B) changes payload share, not packet size.

**Q17.** 50M viewers, segment length changed from 6 s to 2 s. What happens to peak segment requests and the latency floor (3 × segment)?
- A) Requests drop from 25 M/s to 8.3 M/s; floor rises from 6 s to 18 s
- B) Requests stay the same; floor drops from 18 s to 6 s
- C) Requests double; floor halves
- D) Requests triple from 8.3 M/s to 25 M/s; floor drops from 18 s to 6 s

> ✅ **Answer: D.** 50M ÷ 2 s = 25 M/s vs 50M ÷ 6 s = 8.3 M/s, and 3 × 2 = 6 s vs 3 × 6 = 18 s. Shorter segments buy latency with more requests and keyframes.

**Q18.** Using the lecture's latency budget (encode ~2 s, CDN ~2 s, hold-back = 3 × target duration), how far behind the stadium is a viewer with **6 s** segments?
- A) about 12 s
- B) about 20 s
- C) about 28 s
- D) about 18 s

> ✅ **Answer: C.** 2 (encode) + 6 (segment fills) + 2 (to CDN) + 18 (3 × 6 hold-back) = 28 s. 18 s is only the hold-back; 20 s is the 4-second-segment case.

**Q19.** Which Low-Latency HLS feature is long polling under another name, and what does preload hint replace?
- A) Parts; it replaced WebSockets
- B) Blocking playlist reload (`?_HLS_msn=…` held until that segment exists); preload hints replaced HTTP/2 push (Jan 2020)
- C) Delta updates (`EXT-X-SKIP`); preload hints replaced SSE
- D) `PART-HOLD-BACK`; preload hints replaced 103 Early Hints

> ✅ **Answer: B.** The server holds the blocking reload until the requested segment exists — exactly long polling. Preload hints name the next part early and replaced HTTP/2 push. Delta updates are "diffs for playlists".

**Q20.** The app's score overlay shows a wicket ~10 s before the video does. What fix does the lecture give?
- A) Stamp each score update with the ball's wall-clock time and reveal it only when playback's `PROGRAM-DATE-TIME` passes it
- B) Lower every CDN TTL on the scorecard to 1 s
- C) Move the scorecard to WebSockets so it arrives together with the video
- D) Increase the poll interval to 20 s to slow the scorecard down

> ✅ **Answer: A.** Hold the score until the picture catches up, using the segment's wall-clock tag. B and C make the score arrive even earlier; D is a guess that breaks when latency changes.

**Q21.** Which statement matches the lecture's view of ABR?
- A) The CDN picks the rendition for each viewer from its logs
- B) Buffer-based ABR always picks the highest rendition that fits the measured throughput
- C) A stall is acceptable if it keeps the highest quality
- D) The player decides at every segment under a product policy (a stall is worse than a lower rendition); buffer-based ABR gave 10–20% fewer rebuffers in Netflix's tests

> ✅ **Answer: D.** The player decides per segment under a policy the product sets. B mixes up the two: "highest rendition that fits" is throughput-based; buffer-based steps down when the buffer is nearly empty.

**Q22.** During a match, CDN A starts failing. Ops want **all** players moved to CDN B mid-stream, decided centrally on the server. Which steering method fits best?
- A) DNS: hand out CDN B to every region
- B) Client failover with a circuit breaker
- C) Content Steering: a JSON file, polled every TTL, that names the preferred CDN
- D) Re-encode the stream with a new ladder

> ✅ **Answer: C.** Content Steering is server-controlled and polled, so players switch within one TTL. DNS is slow to change because resolvers cache; client failover is decided per client, not centrally.

**Q23.** A live stream uses `-hls_time 6 -hls_list_size 5`. By RFC 8216, a removed segment must stay available for its duration plus the playlist's. What is the smallest `-hls_delete_threshold` that meets this?
- A) 1
- B) 6
- C) 5
- D) 36

> ✅ **Answer: B.** It must stay 6 + (5 × 6) = 36 s; 36 s ÷ 6 s per segment = 6 segments. 36 is the time, not the segment count; 1 is ffmpeg's default that caused the stale-playlist 404s.

**Q24.** `ffmpeg -re -i input.mp4 -codec: copy -hls_time 4 …` produces 8.33 s segments from a 30 fps x264 file. Why?
- A) `-codec: copy` cannot re-encode, so it can only cut at existing keyframes, and x264's default keyframe interval is 250 frames = 8.33 s at 30 fps
- B) `-re` slows the input, which stretches segments
- C) `-hls_time` is in frames, so 4 means 4 × 2 frames
- D) The EVENT playlist type doubles segment length

> ✅ **Answer: A.** `-hls_time` is a target, not a promise; the cut happens at the first keyframe after 4 s. The EVENT type only stops the window from sliding — it does not change segment length.

**Q25.** The class ladder (`-g 120 -keyint_min 120 -hls_time 4`) is run on a **25 fps** source. How long will the segments be?
- A) 4 s
- B) 5 s
- C) 4.8 s
- D) 8.33 s

> ✅ **Answer: C.** A keyframe every 120 frames ÷ 25 fps = 4.8 s, and segments can only be cut at keyframes. 4 s holds only at 30 fps (5 s was the 24 fps case); time-based `force_key_frames` would give 4 s.
