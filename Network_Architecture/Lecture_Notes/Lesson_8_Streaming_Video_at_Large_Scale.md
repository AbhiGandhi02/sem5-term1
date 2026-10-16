# 📓 Streaming Video at Large Scale
### *CN at Scaler — Lesson 8 (Session 8 of 8)* · "it starts with a live cricket scorecard"

> `polling` · `long poll` · `SSE` · `WebSockets` · `MQTT` · `Little's Law` · `304` · `TTL stacking` · `request collapsing` · `negative caching` · `HLS` · `segments` · `ABR` · `LL-HLS` · `multi-CDN` · `ffmpeg`
> **Every live stream is a scorecard with bigger bytes.**

**Legend:** 🧠 = my addition, *not on the slides* · 💡 **Simple words mein** = an easy (sometimes Hinglish) summary. Read it first if a section feels heavy. "(slide 21)" = PDF page number; "↩ Week N" = a callback to an earlier session.

---

## ⭐ WHAT THIS LECTURE IS ABOUT (AND WHY IT MATTERS)

The last session uses **one** system to go over everything from the last seven weeks: **a live cricket match watched on 50 million phones.** It starts with the smallest possible problem, a **~1 KB scorecard JSON** that changes every 25–45 seconds, and asks *how do you deliver this one small file to 50M phones?*

There are two families of answers:
- **Pull** (polling, with caches and a CDN in front). Every request is an ordinary, cacheable GET, so **pull scales through caches**.
- **Push** (long poll, SSE, WebSockets, MQTT). Every viewer holds a socket for four hours, so **push scales through machines**.

> 💡 **Simple words mein:** Pull = dukaan ke bahar score ka board laga do, jo chahe aake padh le (cache). Push = har customer ke ghar ek aadmi bhejo jo score batata rahe: 50M aadmi chahiye!

The lecture prices both, then makes polling survive at scale with **304s, TTL budgets, request collapsing, negative caching and diffs**. Then comes the reveal: **a versioned scorecard that everyone polls, pointing at numbered pieces that never change, is exactly how HTTP Live Streaming (HLS) works.** A video playlist (`index.m3u8`, 228 bytes, polled every 4 s) is the scorecard, and video segments (`seg_000008.ts`, 823 KB, fetched once) are the bytes it points at.

The second half covers the video parts: what's inside a segment (188-byte MPEG-TS packets), how long segments should be, the player's **buffer**, the **latency budget** (why you're ~20 s behind the stadium), the **spoiler problem** (the score arriving before the picture), **ABR** (the player choosing quality), **multi-CDN steering**, **stale-playlist bugs**, and finally **building the stream yourself with two ffmpeg commands**.

```
  the scorecard loop                         the video loop  (same design, bigger bytes)
  ───────────────────                        ──────────────────────────────────────────
  GET score.json   1 KB, every 15 s   ≡      GET index.m3u8   228 B, every 4 s    ← small, changes, POLLED
  "version": 1842 + recent changes    ≡      MEDIA-SEQUENCE + last 10 segments
                                             GET seg_000008.ts 823 KB, once       ← big, NEVER changes, cached forever
```

---

## 📋 Table of contents

0. [Opening slides: the hook, the course map, the agenda](#0--opening-slides)
1. [Prerequisites from scratch](#1--prerequisites-from-scratch-)
2. [The brief: one small file, fifty million times](#2--the-brief-one-small-file-fifty-million-times)
3. [Every way to deliver it: polling, long poll, SSE, WebSockets, MQTT](#3--every-way-to-deliver-it)
4. [What it costs: held sockets, Little's Law, four bills](#4--what-it-costs)
5. [Making polling survive: 304, TTLs, herds, collapsing, negative caching, diffs](#5--making-polling-survive)
6. [Same loop, bigger bytes: HLS](#6--same-loop-bigger-bytes-hls)
7. [Buffers and latency: buffer, latency budget, LL-HLS, spoilers, ABR](#7--buffers-and-latency)
8. [Many origins, many CDNs: steering, stale playlists, cache rules](#8--many-origins-many-cdns)
9. [Making the stream: two ffmpeg commands](#9--making-the-stream-two-ffmpeg-commands)
10. [Closing: the course in one match, four ideas to keep](#10--closing)
11. [Hands-on](#11--hands-on-)
12. [Assignment and homework](#12--assignment--homework)
13. [Cheat-sheet](#13--cheat-sheet)
14. [Self-check questions](#14--self-check-questions)

---

## 0 · OPENING SLIDES

### 0.1 The hook (slide 1)

The title slide shows three requests. Together they're the whole lecture in miniature:

```
GET /match/412/score.json      200 ·   1,088 bytes · every 15 s     ← the scorecard (Part 1)
GET /stream_1/index.m3u8       200 ·     228 bytes · every 4 s      ← the video playlist (Part 2)
GET /stream_1/seg_000008.ts    200 · 823,440 bytes · once           ← one 4-second video segment
```

Two of these are **small files polled on a timer**. One is a **big file fetched exactly once**. By the end you should see that the first two are the *same* problem, and the third is a *different billing problem* (§4.6).

### 0.2 Course map: "Eight weeks, one system" (slide 2)

```
 W1 Network programming 101   W2 Eagle-eye view   W3 Server-side considerations for scale   W4 nginx deep dive
 W5 Evolution of HTTP         W6 (Ab)using CDNs   W7 Building for failures                  W8 ▶ STREAMING VIDEO AT LARGE SCALE ◀
```
> "Today every one of them shows up in a single system: one live match on fifty million phones."

### 0.3 Agenda (slide 3)

| # | Block | What's in it |
|---|---|---|
| 01 | The brief | A scorecard for 50M phones, and every way to deliver it |
| 02 | What it costs | Held sockets, Little's Law, and four bills for one match |
| 03 | Making polling survive | 304s, stacked TTLs, herds, collapsing, negative caching, diffs |
| 04 | Same loop, bigger bytes | HLS playlists, segments and 188-byte packets |
| 05 | Buffers and latency | The buffer, the latency budget, spoilers, and ABR |
| 06 | Many origins, many CDNs | Steering, stale playlists, and cache rules |
| 07 | Making the stream | Two ffmpeg commands, run for real |
| 08 | Closing | The course in one match, four ideas, assignment, homework |

---

## 1 · PREREQUISITES FROM SCRATCH 🧠

Everything in this section is background the slides assume. Skip what you know.

### 1.1 Video in five words: frames, codec, container, keyframes, bitrate

- **Frame**: one still picture. Video is frames shown quickly: **24 fps** (film), **25/30 fps** (TV), **50/60 fps** (sport).
- **Codec**: the compression algorithm. **H.264/AVC** (`avc1…`) is supported everywhere. **H.265/HEVC** (`hvc1…`) is ~40% smaller but not everywhere. **AAC** (`mp4a.40.2`) is the usual audio codec. Raw 720p30 video is ~660 Mbps; H.264 gets it to ~3 Mbps.
- **Container**: the box that holds the encoded audio and video together with timing information: `.mp4`, `.ts` (MPEG-TS), `.m4s` (fragmented MP4).
- **Keyframe (I-frame)** vs. **P/B-frames**: a keyframe is a complete picture. The frames after it store only the *differences* from earlier frames. **A video can only be cut, or started, at a keyframe**: a frame that says "same as before, but the ball moved" is useless without the "before". The distance between keyframes is the **GOP** (group of pictures). **This is why the ffmpeg segment-length surprise in §9 happens.**
- **Bitrate**: bits per second of encoded video. Higher bitrate = better quality, bigger files. A 4 s segment at 1.5 Mbps ≈ 4 × 1.5 / 8 = 0.75 MB, close to the slide's 823 KB for 480p.

### 1.2 Renditions and the ABR ladder
The same video encoded at several quality levels (e.g., 720p @ 3 Mbps, 480p @ 1.5 Mbps, 360p @ 800 kbps). Each one is a **rendition** (also called a *variant*), and the list of them is the **ABR ladder**. **Adaptive Bitrate (ABR)** = the player switching between renditions **at segment boundaries** as network conditions change. All renditions are keyframe-aligned so a switch is seamless.

### 1.3 HLS vocabulary
| Term | Meaning |
|---|---|
| **HLS** | HTTP Live Streaming (Apple, 2009; RFC 8216). Video delivered as ordinary HTTP files: playlists + segments |
| **Master (multivariant) playlist** | `master.m3u8`. The *menu* of renditions: bandwidth, resolution, codecs, and the URL of each rendition's playlist |
| **Media playlist** | `stream_1/index.m3u8`. The list of the latest segments for **one** rendition. For live it changes every few seconds |
| **Segment** | `seg_000011.ts`. A few seconds of audio + video. **Never changes once written** |
| **Target duration** | `#EXT-X-TARGETDURATION:4`. The maximum segment length. It also sets how often players reload the playlist |
| **Media sequence** | `#EXT-X-MEDIA-SEQUENCE:2`. The number of the first segment in the current list, i.e., a version number for a sliding window |
| **Live edge** | The newest segment that exists right now |
| **VOD vs. live** | VOD = a fixed list ending with `#EXT-X-ENDLIST`. Live = a sliding window that keeps growing at the end |
| **DASH** | The non-Apple equivalent (MPEG-DASH, `.mpd` manifest). Same idea, XML manifest |
| **CMAF / fMP4** | A shared segment format (fragmented MP4) that both HLS and DASH can use |

### 1.4 Little's Law (Week 3 refresher)
> **L = λ × W**: items *in the system* = arrival rate × time each item spends there.

If 100 requests arrive per second and each takes 0.03 s, then on average 3 are in flight. The key point for this lecture: **if each item stays for hours instead of milliseconds, the number in flight explodes even at the same arrival rate.**

### 1.5 Caching vocabulary (Weeks 4 & 6 refresher)
| Term | Meaning |
|---|---|
| **Origin** | Your own server, the source of truth |
| **Edge** | A CDN server close to users |
| **Mid-tier / shield** | A CDN layer between edges and the origin, so 300 edges become ~1 request to the origin |
| **TTL** | How long a cached copy counts as fresh (`Cache-Control: max-age=10`) |
| **Cache key** | What makes two requests "the same object". Usually host + path + **query string** |
| **ETag** | A fingerprint of a response version. Clients send it back as `If-None-Match` |
| **304 Not Modified** | "Your copy is still current", sent with no body |
| **Request collapsing** | Many simultaneous misses for one object → **one** upstream fetch (nginx: `proxy_cache_lock`) |
| **Serve stale** | Answer from an expired copy while refreshing it, or while the origin is down (RFC 5861) |
| **Negative caching** | Caching *error* responses (404, 5xx) |
| **Egress** | Bytes leaving a cloud provider's network. You pay per GB |

### 1.6 Push vs. pull in one line
- **Pull**: the client asks repeatedly ("anything new?"). Servers can be stateless and caches can answer.
- **Push**: the server tells the client when something happens. This needs an **open connection per client**, so servers are stateful and no shared cache can help.

---

## 2 · THE BRIEF: ONE SMALL FILE, FIFTY MILLION TIMES

### 2.1 The numbers (slide 4)

| Number | Meaning |
|---|---|
| **25–45 s** | Between balls. An over every 4–5 minutes. So the data changes rarely in computer terms |
| **50 M** | **Peak** concurrent viewers. **20 M average** across a 4-hour match |
| **~1 KB** | Per scorecard: 4–5 KB of JSON, **about 1 KB gzipped** *(↩ Week 5 · gzip: 202,000 → 5,763 bytes)* |
| **100** | Events a year, each one the same file downloaded again and again |

🧠 **The key property:** **everyone gets the same file.** That single fact decides most of the architecture, because an identical answer for everyone is the perfect thing to cache.

### 2.2 Two families of answers (slide 5)
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

## 3 · EVERY WAY TO DELIVER IT

### 3.1 Polling: ask again every few seconds (slide 6)
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

Three ideas packed into 12 lines:

| Idea | What the slide says | 🧠 Why |
|---|---|---|
| **ETag + If-None-Match** | Nothing changed? **304 and an empty body.** *The request still happens.* | It saves bytes and parsing, **not** the request itself. This matters for billing (§4.6) |
| **Jitter** | **±20%**, so fifty million phones don't poll on the same second | Without it, everyone who opened the app at the start of the match polls in lock-step: a self-inflicted thundering herd every 15 s (Week 7) |
| **Freshness** | At worst **one poll interval plus every cache TTL on the path** | The basis of the TTL-stacking slide (§5.2) |

🧠 Note `setTimeout` instead of `setInterval`: the next poll is scheduled only *after* this one finishes, so a slow response can't stack polls on top of each other.

### 3.2 Long polling: ask once, wait for news (slide 7)
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
| **A held request is a held socket** | Every viewer has one open almost all the time: **W is 25 s, not 30 ms** (Little's Law → §4.2) |
| **Beat every idle timeout** | Answer *before* nginx's `proxy_read_timeout` and the AWS ALB idle timeout. **Both default to 60 s** (Week 4). Hence the 25 s cap: send an empty "no news" response before a middlebox kills the connection |
| **It synchronises the herd** | A wicket completes **20M responses in one second**, and **20M new requests arrive in the next** (Week 7). Every client is waiting for the same event, so they all wake together |
| **It came back, standardised** | Low-Latency HLS's **blocking playlist reload** *is* long polling (§7.2) |

🧠 `?after=1841` makes the request **resumable**: the client says what it already has. If it missed balls 1842–1844 while reconnecting, the server sends all of them at once.

### 3.3 Server-Sent Events: one response that never ends (slide 8)
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
| **Framing by delimiter** | An event ends at a **blank line**: "SMTP's lone dot in a different costume" (Week 2: you frame a message either by *length* or by *delimiter*) |
| **Resume from a version** | On reconnect the browser automatically sends **`Last-Event-ID`**, and the server replays only what was missed. `retry: 5000` tells the browser to wait 5 s before reconnecting |
| **About six per browser** | Over **HTTP/1.1**, browsers allow **~6 connections per origin**, and each tab's SSE stream holds one. **HTTP/2 puts them all on one connection** (Week 5). → Homework 3 |
| **One way only** | Server → client. Anything the phone says goes in ordinary requests |

🧠 In the browser it's just `new EventSource("/match/412/events")` plus `addEventListener("ball", …)`. Reconnecting and resuming are built in, which is SSE's main advantage over a hand-rolled long poll.

### 3.4 WebSockets: leave HTTP after one exchange (slide 9)
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

- **The Accept value is `base64(SHA-1(key + a fixed GUID))`**: base64 again (Week 1). 🧠 The GUID is `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`. It proves the server really speaks WebSocket and isn't a confused HTTP server echoing headers back.
- **Client frames are masked** so stray bytes can never look like a request to a caching proxy (Week 6). 🧠 Without masking, an attacker's script could send bytes that look like `GET /evil` and poison a transparent proxy's cache.

**After the 101, every frame starts like this:**

| Field | Bytes | What it says |
|---|---|---|
| FIN · RSV · opcode | 1 | Last fragment? Text, binary, ping, pong, close |
| MASK · length | 1 | 0–125 is the length; **126 or 127 means "read more"** |
| Extended length | 0, 2 or 8 | A 16- or 64-bit length |
| Masking key | 0 or 4 | Present on every **client-to-server** frame |
| Payload | length | Your 183-byte scorecard diff |

> **A length, not a delimiter**: the other answer from Session 2. (SSE uses a delimiter; WebSockets use a length.)

### 3.5 MQTT: publish once, fan out to everyone (slide 10)
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

**One PUBLISH from the scorer; the broker copies it to every subscriber of the topic.** Hotstar chose MQTT with the **EMQX** broker for exactly this fan-out.

| Feature | Meaning |
|---|---|
| **Retained message** | The broker keeps the last score and hands it to each new subscriber immediately: *a cache, inside the broker* |
| **QoS 1 = at-least-once** | Duplicates happen. **Apply updates by version number and they become harmless** (Week 7, idempotency: doing it twice = doing it once) |
| **Remaining Length is a varint** | 7 bits per byte; the top bit says "more". *A type, a length, a value* (TLV) |
| **Keepalives cost** | Only two bytes, but they **wake the phone's radio** (battery) and **keep a NAT mapping alive** |

🧠 *Why NAT?* Mobile carriers put phones behind carrier-grade NAT, which forgets idle mappings after a few minutes. A connection that's silent too long is silently dead, so the client has to ping regularly. Those pings are "keepalives with nothing to say", and §4.1 prices them.

🧠 MQTT QoS levels: 0 = at-most-once (fire and forget), 1 = at-least-once (acknowledged, may duplicate), 2 = exactly-once (four-step handshake, expensive).

### 3.6 Case study: how Hotstar held 50 million sockets (slide 11)
*(↩ Week 3 · from C10K to C10M)*

| Step | What they did | Result |
|---|---|---|
| **1 · One node** | EMQX on one `c5.4xlarge` | **250k** connections, latency within bounds |
| **2 · A cluster** | Publisher split from subscribers | **2M** subscribers. Mnesia (EMQX's Erlang database) wants **fewer than 10 nodes** |
| **3 · The ELB wall** | Load balancer limits | Drops at **~500k without pre-warming**; **4M per pre-warmed Classic ELB** |
| **4 · Five clusters** | 5 clusters × 8 subscribe nodes, bridged | **Past 10M** |
| **5 · Reverse bridge** | A Go service on each subscriber node **pulls** from the publisher | **200 nodes: 50M** |

> *"Caching at any level is also not possible."*
> Their **social feed** was tied to live moments, so it had to be pushed. **A scorecard is the opposite case: one file, identical for everyone.**

*Source: M. Chaudhari and P. Gupta, "Building Pubsub for 50M concurrent socket connections", Hotstar Engineering blog, 6 June 2019.*

🧠 **Takeaway:** push at this scale is possible, but every step was a fight against a *per-connection* limit (node memory, cluster membership, load balancer, fan-out). Pull has none of these, because a CDN absorbs it. The rest of the lecture explains why the instructor picks polling for the scorecard.

### 3.7 🧠 All six options side by side

| | Direction | Connection | Framing | Cacheable by CDN? | Resume after drop | Server state per client |
|---|---|---|---|---|---|---|
| Polling | client asks | short | HTTP | ✅ yes | trivial (ask again) | none |
| Long poll | client asks, server delays | held ~25 s | HTTP | ❌ | `?after=version` | held request |
| SSE | server → client | held for hours | delimiter (blank line) | ❌ | `Last-Event-ID` | open stream |
| WebSocket | both ways | held for hours | length-prefixed frames | ❌ | DIY | open socket |
| MQTT | pub/sub via broker | held for hours | TLV, varint length | ❌ | retained msg + QoS | session in broker |
| Raw binary socket | your choice | your choice | your protocol | ❌ | DIY | yours |

---

## 4 · WHAT IT COSTS

### 4.1 What a held socket costs (slide 12)
*(↩ Week 3 · nothing is free per connection)*

| Cost | Detail |
|---|---|
| **Machines** | **Stateful**: each phone lives on one box for hours. **100k–200k connections per box** on the class board; Hotstar measured **250k**. **New boxes sit empty until someone reconnects**, so rebalancing means disconnecting people |
| **Memory** | Kernel socket buffers, TLS state, the broker's per-client process. **250k phones on a 32 GiB box ≈ 137 KB each, all in.** And the soft **`ulimit -n` still starts at 1,024** (one fd per socket, Week 3) |
| **Network** | Keepalives with nothing to say: **at a 60 s keepalive, 20M idle phones send 333k pings a second** *before the first ball is bowled* (20M ÷ 60 = 333k) |
| **Load balancers** | One address instead of 200 for DNS to hand out, and an even spread across boxes. **But the LB holds every socket too**: Hotstar hit 4M per Classic ELB |

> **Testing it:** one client IP has **28,232 ephemeral ports (32768–60999) per destination**, so **250k test sockets need at least 9 source IPs** (250,000 ÷ 28,232 = 8.9). *Week 1: a port is 16 bits.*

🧠 Why per destination? A TCP connection is identified by the 4-tuple (src IP, src port, dst IP, dst port). Against one server IP:port, only the source port varies, so one client IP can open at most ~28k connections to it. Linux's default range is `net.ipv4.ip_local_port_range = 32768 60999`.

### 4.2 Little's Law, twice (slide 13)
*(↩ Week 3 · L = λ × W)*

| POLLING THROUGH A CDN | PUSH OVER HELD SOCKETS |
|---|---|
| 1.67 M requests/s × about 0.03 s at the edge | W is the whole match: **four hours** |
| **= 50,000 requests in flight** | **= 50,000,000 sockets held**, ball or no ball |

> **Same audience, same scorecard: a thousand times the concurrency, because W went from milliseconds to hours.**

> 💡 **Simple words mein:** Bank mein har customer 30 second ruke toh hall khaali rehta hai; har koi 4 ghante baith jaaye toh hall bhar jaata hai, chahe aane wale utne hi hon.

🧠 Concurrency (how many at once), not request rate, is what costs memory, file descriptors and LB capacity. Polling has a *higher* request rate but a tiny W. Push has almost no requests but W = 4 h.

### 4.3 Where the scorecard file lives (slide 14)
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

### 4.4 The arithmetic before the architecture (slide 15)
*(↩ Week 3 · multiply before you choose)*

| Quantity | Formula | Result |
|---|---|---|
| Requests per match | 20M viewers × 14,400 s ÷ 30 s | **9.6 B** |
| Peak rate | 50M viewers ÷ 30 s | **1.67 M req/s** |
| Bytes per match | 9.6 B × 1 KB | **9.6 TB** |
| *Poll every 15 s instead* | double everything | 19.2 B requests, 3.33 M req/s, 19.2 TB |
| *Push each ball instead* | one message every ~30 s per phone | about 9.6 B messages |
| × 100 events a year | | **960 B requests and 960 TB a year** |

🧠 14,400 s = 4 hours. Average viewers (20M) give the *total*, and peak viewers (50M) give the *rate you must survive*. Always calculate both.

### 4.5 Four ways to pay for one match (slide 16)
*(↩ Week 6 · a CDN is a rented reverse proxy)*

| Design | Per match | × 100 events | Mostly paying for | Breaks first |
|---|---|---|---|---|
| **S3 straight to phones** | $4.7K | $470K | GET fees ($3,840) | **5,500 GET/s per prefix** |
| **nginx fleet + ALB** | $1.2–1.8K | $120–180K | egress ($864) + load balancer units | **20–50M idle keep-alive sockets at the LB** |
| **CDN (CloudFront, India)** | **$12.5K** | **$1.25M** | **request fees: 92% of it** | nothing; it's just the bill |
| **Push: 200 × c5.4xlarge** | **$0.9–1.6K** | $90–160K | a peak-sized fleet ($680) + egress | reconnect storms, rebalancing |

> **At list price, push is the cheapest per match and the CDN the most expensive. Hold that thought.**

*AWS list prices, Sep 2026 · S3, EC2, ELB: US East · CloudFront: India · body bytes only · poll every 30 s · push sends 183-byte diffs up to full cards.*

🧠 Checking the numbers: S3 GETs at $0.0004 per 1,000 → 9.6 B × 0.0004 / 1000 = **$3,840**. Egress at ~$0.09/GB → 9.6 TB = 9,600 GB × 0.09 = **$864**. CDN request fees at $0.012 per 10k → 9.6 B × 0.012 / 10,000 = **$11,520**, plus ~$1K of bytes ≈ **$12.5K**.

### 4.6 S3 runs out of requests before money (slide 17)
*(↩ Week 7 · 503 and Retry-After)*

| 5,500 | 1.67 M | 303× |
|---|---|---|
| GETs a second that one S3 prefix promises, *at least* | GETs a second that 50M phones polling every 30 s ask for | over, and **one key lives under one prefix** |

> *"While Amazon S3 is scaling to your new higher request rate, you may see some 503 (Slow Down) errors."*
> **Fifty million clients retrying a 503 without backoff is Session 7's retry storm, pointed at your own bucket.**

*Source: Amazon S3 User Guide, "Best practices design patterns: optimizing Amazon S3 performance".*

🧠 The usual S3 trick of spreading load across prefixes doesn't help here, because it's **one file**. A single hot key can't be sharded.

### 4.7 On a CDN, a 1 KB file is billed per request (slide 18)
*(↩ Week 5 · headers: 97.5% repeats)*

Per **million** responses (CloudFront India list prices: HTTPS **$0.0120 per 10,000 requests**; **$0.109/GB** after 1 free TB, **$0.072/GB** past 5 PB):

| Object | Request fee | Byte fee | Which dominates |
|---|---|---|---|
| Scorecard, 1 KB | **$1.20** | **$0.11** | **request fee: 92%** |
| 480p segment, 823 KB | $1.20 | **$59–90** | bytes |

```
   you pay per request  ◄──────────── break-even ≈ 11 KB ────────────►  you pay per byte
   playlist 228 B · scorecard 1 KB                         360p seg 459 KB · 480p 823 KB · 720p 1.6 MB
```

🧠 **Break-even:** request fee = byte fee when size × $0.109/GB = $0.0000012 per request → size ≈ 0.000011 GB ≈ **11 KB**. Below 11 KB you're billed for *how often*. Above it, for *how much*. This is "idea #4" of the course (§10.2).

### 4.8 "I prefer polling because I want to tell you how it's used for video too." (slide 19)
*(↩ Week 3 · HTTP is stateless, the rest aren't)*

The instructor's verdict, despite the CDN being the most expensive option on paper:

| Cacheable | Stateless | Already paid for |
|---|---|---|
| One file, identical for fifty million people | Any server, any edge, any CDN can answer any phone | At list price the scorecard is **0.28% of the same match's video bill** |

🧠 The last point settles it: you already run a CDN for the video, and its bill is far bigger than the scorecard's. Adding one more small file to the CDN is a rounding error, and you get statelessness, failover and caching for free. Push saves money only if the scorecard is the *only* thing you serve.

---

## 5 · MAKING POLLING SURVIVE

### 5.1 Conditional GET: 304 Not Modified (slide 20)
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
- `If-Modified-Since` works too, **but only to the second**. 🧠 Two balls in the same second, or two servers with slightly different clocks, would make a timestamp-based check wrong. A version number can't be.

### 5.2 TTLs add up (slide 21)
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

🧠 *Why they add:* the mid-tier fetched the file just before it changed, so it serves the old version for 30 s. The edge fetched from the mid-tier at second 29 and serves that old copy for another 30 s. The phone polled the edge just before the edge refreshed and won't ask again for 30 s. Worst case: 30 + 30 + 30.

### 5.3 The thundering herd at expiry (slide 22)
*(↩ Week 4 · proxy_cache_lock + use_stale)*

> At **10.000 s** the edge's copy expires. **Every request in that instant is a miss, and all of them head for the origin.**

| Defence (four of them) | How |
|---|---|
| **Request collapsing** | One fetch per edge per object; everyone else waits in a queue |
| **Serve stale while updating** | Answer from the old copy while one request refreshes it (RFC 5861) |
| **Prefetch before expiry** | Refresh at ~25 s of a 30 s TTL, so no viewer meets an empty cache |
| **Jitter from the client** | Spread polls across the interval, not on the same second (Week 7) |

> 💡 **Simple words mein:** Chai khatam hui aur 100 log ek saath aaye. Smart chaiwala nayi chai sirf ek baar banata hai, baaki wait karte hain (collapsing), ya tab tak purani chai de deta hai (serve stale).

The nginx version:
```nginx
proxy_cache_lock              on;      # collapsing: one request goes upstream per key
proxy_cache_use_stale         updating error timeout http_500 http_502 http_503 http_504;
proxy_cache_background_update on;      # refresh in the background, answer stale meanwhile
proxy_cache_valid 200         10s;
```
🧠 Line by line:
- `proxy_cache_lock on`: only the first miss for a key goes to the origin; the others wait (up to `proxy_cache_lock_timeout`, default 5 s).
- `use_stale updating`: while a refresh is in flight, serve the expired copy instead of waiting. `error timeout http_5xx`: if the origin is broken, serve the last good copy (stale-if-error from Week 7).
- `background_update on`: the refresh happens in a background subrequest, so **no** client waits for it. This needs `use_stale updating`.

### 5.4 Request collapsing needs a shareable answer (slide 23)
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

🧠 *Why does a private response make the queue go one by one?* The CDN can only hand one response to 19,999 waiters if that response is allowed to be shared. If it turns out to be private (per-user), the CDN must send each waiter to the origin separately, one after another. Fastly then remembers "this object isn't cacheable" (a **hit-for-pass** marker) so it doesn't make people queue for it again. One accidental `Set-Cookie` from your framework can quietly turn your CDN into a pass-through.

### 5.5 Negative caching: 404 and 5xx (slide 24)
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

🧠 Picture it: a player requests `seg_000012.ts` 200 ms before the packager writes it. The origin says 404. If the CDN caches that 404 for 3 minutes, *every* viewer gets 404 for that segment for 3 minutes, even though the file has existed since 200 ms later. This is the "live-edge 404" from Week 7, turned into an outage by a cache.

### 5.6 Send diffs, not the whole card (slide 25)
*(↩ Week 6 · Railgun · RFC 9842 deltas)*

| 4,117 B | 1,088 B | 183 B |
|---|---|---|
| the full scorecard JSON | the same card, gzipped | **one ball as a gzipped JSON Patch** |

*Measured on a synthetic T20 card: two innings, batting and bowling figures, fall of wickets.*

| Push: send the patch | Pull: version the file |
|---|---|
| SSE or MQTT carry the 183-byte patch. **A phone that missed one sees a version gap and fetches the full card once** | `?since=1841` gives every client its own URL and **the hit rate collapses**. **Publish one file with the version and the last few changes instead** |

🧠 **JSON Patch** (RFC 6902) is a list of operations: `[{"op":"replace","path":"/total","value":"146/4"}]`. **Railgun** was Cloudflare's old origin-to-edge delta compression. **RFC 9842** (Compression Dictionary Transport) lets a previous response act as the compression dictionary for the next one, so an almost-identical file compresses down to roughly its diff.

🧠 *Why `?since=` kills caching:* the query string is part of the **cache key**. Ten thousand clients at ten thousand different versions means ten thousand different URLs, and each one is a miss. The fix keeps **one URL** for everyone and puts the recent history *inside* the file.

### 5.7 A versioned scorecard is a playlist (slide 26)
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

## 6 · SAME LOOP, BIGGER BYTES: HLS

### 6.1 Same loop, bigger bytes (slide 27)
*(↩ Part 1 · the scorecard loop)*

| 1 KB | 228 B | 823 KB |
|---|---|---|
| scorecard, polled every 15 s | video playlist, polled every 4 s | one 4 s segment at 480p, **fetched once** |

> **DEMO:** Open *Big Buck Bunny* or *Tears of Steel* on **videojs.org** with **DevTools → Network** open. Watch the master playlist, then a media playlist, then segment after segment. **Throttle the network and watch it change rendition.** → Homework 1, §11.1

### 6.2 HLS in one picture (slide 28)
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

🧠 **Control vs. data** (Week 2, e.g., SS7 separating signalling from the voice channel; FTP's control and data connections): the *small, changing* files describe the *big, unchanging* ones. You cache them completely differently (§8.3).

> 💡 **Simple words mein:** Playlist = railway station ka display board (chhota, har 4 s update, sab dekhte hain). Segments = trains (badi, aur ek baar aa gayi toh badalti nahi).

### 6.3 The master playlist (slide 29)
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

*From the class ffmpeg run, attribute lines unwrapped.* **`BANDWIDTH = (video + audio) × 1.1`**: (3000k + 128k) × 1.1 = **3,440,800**. 🧠 Check the others: (1500k + 96k) × 1.1 = 1,755,600 ✓; (800k + 64k) × 1.1 = 950,400 ✓. The 10% covers container overhead (§6.6) and bitrate peaks.

| Slide note | Meaning |
|---|---|
| **"Do not poll this again"** | It doesn't change during the match: **fetch once, cache for minutes** |
| **"Some people poll: switch CDNs"** *(board)* | Re-reading the menu is how a player learns a new path. **Standardised later as Content Steering** (§8.1) |
| **"Falling back to another codec"** | `CODECS` lets the player **skip what it can't decode**, e.g., HEVC down to H.264. **A fallback, not a retry** (Week 7) |

🧠 Decoding `avc1.4d401f`: `avc1` = H.264; `4d` = **Main** profile (77); `40` = constraint flags; `1f` = **level 3.1** (31). `4d401e` = Main @ level 3.0, which is enough for 360p. `mp4a.40.2` = AAC-LC audio.

### 6.4 The media playlist, polled (slide 30)
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

Between reloads, the window **slid by one**: `seg_000001` dropped off the top, `seg_000011` appeared at the bottom, and `MEDIA-SEQUENCE` went 1 → 2.

**The client's reload rules (RFC 8216):**

| Situation | Rule |
|---|---|
| **Changed?** | Wait **at least one target duration** before reloading again |
| **Unchanged?** | Wait **half a target duration**, then try again |
| **Same number, different URI?** | *"It SHOULD halt playback."* The playlist is lying: segment N was something else last time |

🧠 `#EXTINF:4.000000` = this segment's exact duration. `#EXT-X-PROGRAM-DATE-TIME` = the **wall-clock time** of the segment's first frame. That one tag makes the spoiler fix possible (§7.3).

### 6.5 VOD is a playlist that stopped changing (slide 31)
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

🧠 Total length: 63 × 10 + 5 = **635 s ≈ 10.6 minutes**. A query string is fine in a cache key **as long as every viewer's URL is identical**. Put a per-user auth token in it and every viewer gets a private URL, so the CDN hit rate drops to ~0.

### 6.6 A segment is 188-byte packets (slide 32)
*(↩ Week 2 · why 140 bytes? why 188?)*

```
┌──────────────── 4-byte header ────────────────┬──────────── 184-byte payload ────────────┐
│ 0x47 (sync) · PID (13 bits) · counter          │ a slice of video, of audio, or of the     │
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
| **A 13-bit PID says whose packet it is** | Audio, video and tables **share one byte stream** (Week 5, like HTTP/2 streams on one connection) |
| **fMP4 and CMAF segments are boxes instead** | size, type, payload. **TLV again** (Week 1) |

🧠 **Background:**
- **Why 188?** MPEG-TS was designed for broadcast and ATM networks. ATM cells carry 47 usable bytes in AAL1, and 4 × 47 = 188. (Week 2's "why 140?" was SMS: 140 bytes = 160 7-bit characters.)
- **Sync byte `0x47`**: on a lossy broadcast link a receiver can lose its place. Because every packet is exactly 188 bytes and starts with `0x47`, the receiver finds `0x47` repeating every 188 bytes and is back in sync. Fixed length + a known marker = self-synchronising framing.
- **PAT / PMT**: the Program Association Table (PID 0) points at the Program Map Table, which lists the PIDs carrying video and audio. A player reads these first, then separates the streams.
- **Continuity counter** (4 bits): increments per PID, so a gap shows that a packet was lost.
- **Why overhead is higher for smaller renditions:** the per-packet headers and tables are roughly fixed per segment, and a smaller payload makes them a bigger share.

### 6.7 How long should a segment be? (slide 33)
*(↩ Week 2 · the round-trip tax)*

| Segment | Peak segment requests (50M viewers) | Latency floor (3 ×) | On the class board |
|---|---|---|---|
| 2 s | 25 M/s | 6 s | |
| 3 s | 16.7 M/s | 9 s | "RPS?" |
| **4 s** | **12.5 M/s** | **12 s** | **the class ffmpeg run** |
| 6 s | 8.3 M/s | 18 s | **Apple's advice** |
| 30 s | 1.7 M/s | 90 s | "Too late!" |

| ADAPTIVE / HYBRID | THE TRADE |
|---|---|
| **1–2 s segments at start-up** for a fast first frame, longer ones later. ffmpeg has a knob for it: **`-hls_init_time 1 -hls_time 4`** | **Shorter**: more requests, more keyframes, more round trips. **Longer**: more latency, slower reaction to bandwidth changes |

🧠 *Why "3×"?* Players start playback at least **three target durations** behind the live edge (§7.2), so latency ≥ 3 × segment length. *Why do keyframes cost?* Every segment must start with a keyframe, and keyframes are several times bigger than other frames, so shorter segments mean worse compression at the same bitrate.

### 6.8 Playlist traffic is harder than segment traffic (slide 34)
*(↩ Part 1 · the scorecard problem, again)*

| | Media playlist | Segment |
|---|---|---|
| **Size** | ~230 B gzipped | 450–1,600 KB |
| **Changes** | every 4 s | never |
| **Cache TTL** | ½ target duration or less | minutes to forever |
| **A stale copy** | stalls, or points at deleted files | harmless |
| **Billed by** | requests, like the scorecard | bytes |

> **Board: "Rendition: poll every second, RPS?"** At 1 s that's **50 M requests a second** at the peak. At the RFC's one target duration, **12.5 M**.

🧠 The playlist **is** the scorecard problem: tiny, changing, polled by everyone, billed per request, dangerous when stale. Segments are the easy part: immutable, so cache them forever.

---

## 7 · BUFFERS AND LATENCY

### 7.1 The buffer (slide 35)
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

🧠 *"The buffer is the stale cache"*: the buffer holds video the player already has, and keeps showing it while the network is unavailable. That's `stale-if-error` (Week 7) on the client.

### 7.2 Latency is a budget (slide 36)
*(↩ Week 5 · server push died, Chrome 106)*

```
 encode   │ segment fills │ to CDN │       player hold-back: 3 × target duration
  ~2 s    │      4 s      │  ~2 s  │                    12 s
 ─────────┴───────────────┴────────┴────────────────────────────────────────────►  ≈ 20 s behind the stadium
```
*Encode and CDN times are typical guesses; the hold-back is the spec.*

**Low-Latency HLS (LL-HLS) shrinks the biggest bar with four ideas from the first half of the lecture:**

| Idea | What it is | Which earlier idea it reuses |
|---|---|---|
| **Parts** | **~1 s pieces of a segment**, published as soon as they're encoded. Apple: at least **3× the P95 round trip** (Week 6) | smaller pieces |
| **Blocking reload** | `?_HLS_msn=…` is **held until that segment exists** | **long polling** (§3.2) |
| **Preload hints** | **Name the next part early**. Replaced HTTP/2 push in **Jan 2020** | **request collapsing / 103 Early Hints** (§5.4) |
| **Delta updates** | `EXT-X-SKIP` drops old entries from the reply | **diffs, for playlists** (§5.6) |

> **With 1-second parts the 12 s hold-back becomes about 3 s:** Apple requires `PART-HOLD-BACK` of at least three parts.

🧠 LL-HLS tag names to recognise: `#EXT-X-PART-INF:PART-TARGET=1.0`, `#EXT-X-PART:DURATION=1.0,URI="seg12.part3.ts"`, `#EXT-X-PRELOAD-HINT:TYPE=PART,URI="seg12.part4.ts"`, `#EXT-X-SERVER-CONTROL:CAN-BLOCK-RELOAD=YES,PART-HOLD-BACK=3.0,CAN-SKIP-UNTIL=24`. A blocking request looks like `index.m3u8?_HLS_msn=12&_HLS_part=4`, which means "don't answer until segment 12 part 4 exists".

🧠 The "server push died" callback: the first LL-HLS draft (2019) **required HTTP/2 Server Push** to deliver parts. Push was hard to deploy through CDNs, so Apple replaced it with preload hints in January 2020. Chrome removed HTTP/2 push entirely in **Chrome 106** (2022).

### 7.3 The spoiler problem (slide 37)
*(↩ Part 1 · two polling loops, one clock)*

```
 0 s ─────────────────── ~10 s ───────────────────── ~20 s ──►
 wicket at the stadium   the scorecard says W          the video shows it
                         ◄────── spoiler window 10 s ──────►
```

The scorecard path (poll + TTLs) is faster than the video path (encode + segment + CDN + 12 s hold-back), so the phone's score overlay says **W** ten seconds before you see the wicket.

**Fix: hold the score until the picture catches up.** Stamp each update with the ball's wall-clock time. The player knows the wall-clock time of every segment, so the app reveals an update only when playback passes it.

```text
# the score update
{"v": 1842, "ball": "15.4", "at": "2026-09-27T03:16:44.1Z"}

# the segment on screen
#EXT-X-PROGRAM-DATE-TIME:2026-09-27T03:16:44.365+0000
```

🧠 In code: `currentWallClock = segmentProgramDateTime + (playhead − segmentStartTime)`. Show the update only when `update.at <= currentWallClock`. This is the same trick broadcasters use to keep commentary in sync, and it needs the encoder's clock (PROGRAM-DATE-TIME) and the scorer's clock (`at`) to agree, so both should be NTP-synced.

### 7.4 ABR: the player picks the next rendition (slide 38)
*(↩ Week 7 · who decides "worse"?)*

> *Last week: every fallback gives you something worse than ideal. When does worse stop being acceptable, and who decides?*
> **The player decides, at every segment, under a policy the product sets.**

| Throughput-based | Buffer-based |
|---|---|
| Measure how fast the last segments arrived; **pick the highest rendition that fits**. *Congestion control, one layer up* | **Watch the buffer**: nearly empty → step down; full → step up. Netflix's A/B tests: **10–20% fewer rebuffers** at a similar video rate |

> 💡 **Simple words mein:** Net slow hua toh player HD se SD pe aa jaata hai, jaise baarish mein cricket match chhota (DLS) kar dete hain: kam overs chalega, match rukna nahi chahiye.

> **The policy: a stall is worse than a lower rendition.** Even the default is policy: Apple suggests starting **cellular** viewers on the **730 kbps** variant and **Wi-Fi** viewers on **2 Mbps**.

*Huang, Johari, McKeown, Trunnell, Watson, "A Buffer-Based Approach to Rate Adaptation", SIGCOMM 2014 · Apple HLS authoring spec 1.32.*

🧠 **Throughput-based** example: the last segment (823 KB = 6.6 Mbit) took 2 s → ~3.3 Mbps measured. With a safety factor (say 0.8) that's 2.6 Mbps, so pick 480p (1.76 Mbps), not 720p (3.44 Mbps). Its weakness: throughput estimates are noisy (Wi-Fi, cellular), so it can flip quality back and forth.
🧠 **Buffer-based (BBA)** example: buffer < 5 s → lowest rendition; buffer > 15 s → highest; in between, map linearly. Its insight: the buffer *already* combines all the noise into one smooth signal. Real players (hls.js, ExoPlayer, Shaka) use hybrids.
🧠 *"Congestion control, one layer up"*: TCP probes the network and backs off on loss; ABR probes with higher renditions and backs off when the buffer drains.

---

## 8 · MANY ORIGINS, MANY CDNS

### 8.1 Steering: which CDN, which edge? (slide 39)
*(↩ Week 6 · Fastly's 49 minutes)*

```
                       ┌─► CDN A edge (primary)  ─► CDN A mid-tier (shields origin) ─► Origin 1 (packager A) ┐
 Android app ─ which? ─┤                                                                                    ├ same names
                       └─► CDN B edge (fallback) ─► CDN B mid-tier (shields origin) ─► Origin 2 (packager B) ┘
```

| Method (four ways to choose) | How | Trade-off |
|---|---|---|
| **DNS** | Hand out a CDN per region or ISP | **Slow to change: resolvers cache** (TTL) |
| **Listed twice** | The master lists each rendition **once per CDN**, for stream failover | Player falls back on errors; crude |
| **Content Steering** | **A JSON file, polled every TTL, names the preferred CDN** | Server-controlled, mid-stream switching |
| **Client failover** | Errors on one CDN **trip a circuit breaker**; the next segment tries the other (Week 7) | Fast, per-client, needs no server |

🧠 "Same names" matters: both origins/packagers must produce **identical segment names and numbering**, or a player can't switch CDN mid-stream and continue at the next segment.
🧠 **Content Steering** (Apple HLS 2021, also in DASH): the master playlist has `#EXT-X-CONTENT-STEERING:SERVER-URI="/steer.json"`, and renditions are tagged with `PATHWAY-ID="cdnA"` / `"cdnB"`. The steering JSON returns `{"VERSION":1,"TTL":300,"PATHWAY-PRIORITY":["cdnB","cdnA"]}`. Change that file and every player moves to CDN B within one TTL. It's another **small polled file controlling big immutable ones**.
🧠 *Fastly's 49 minutes* (Week 7): on 8 June 2021 Fastly's network mostly failed for ~49 min. Customers with multi-CDN steering moved traffic away; single-CDN customers went dark.

### 8.2 Stale playlists and deleted segments (slide 40)
*(↩ Week 6 · stale-if-error, with a catch)*

1. ffmpeg's `delete_segments` keeps only **one** unreferenced segment. The class watched `seg_000000` **vanish 4 s after it left the playlist**.
2. **RFC 8216 says a removed segment MUST stay available for its duration plus the playlist's: 4 + 40 = 44 s here.**
3. During an origin blip **the CDN serves a stale playlist**. It still lists a segment the origin has already deleted.
4. **404, negatively cached, and the player stalls.** *"Serve stale may cause player issues."*

```
# the class run, 360p rendition
t = 47.0 s   playlist: seg_000001 … seg_000010     on disk: seg_000000 … seg_000010
t = 51.5 s   playlist: seg_000002 … seg_000011     on disk: seg_000001 … seg_000011
```

**FIXES**
- **`-hls_delete_threshold 12`**: keep more deleted-but-maybe-referenced segments on disk. 🧠 12 × 4 s = 48 s ≥ 44 s.
- **Playlist TTL at ½ target duration or less; serve it stale for one target duration at most.**
- **Segments never change: serve them stale freely.**
- **Apple: treat a playlist as invalid when `Last-Modified` is more than 3 target durations behind `Date`.**

🧠 The general lesson: `stale-if-error` (Week 7) is safe for **immutable** objects and dangerous for **indexes**, because a stale index points at things that no longer exist. This is the "catch" in the callback.

### 8.3 Caching rules for a live stream (slide 41)
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

- 🧠 2 s for the playlist = ½ × 4 s target duration ✓. `proxy_cache_lock` on the playlist, because that's where the herd is (§6.8).
- `\.ts$` gets 10 minutes: segments never change, so any TTL is safe, and longer = fewer origin hits.
- nginx's default `gzip_types` is only `text/html`. Playlists (`application/vnd.apple.mpegurl`) and JSON must be added explicitly. Don't gzip `.ts`: it's already compressed video.
- *Prefix vs. regex* (Week 4): `location ~ \.m3u8$` is a regex match on the file extension, which is the natural way to give "one location per file type".

---

## 9 · MAKING THE STREAM: TWO FFMPEG COMMANDS

### 9.1 From an mp4: the one-liner (slide 42)
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

**WHAT RUNNING IT SHOWED** *(the "honest benchmark": run it, don't trust the flags)*
- ❌ **8.33 s segments, not 4.** x264's default keyframe interval is **250 frames: 8.33 s at 30 fps**. With `-codec: copy` ffmpeg can only cut at existing keyframes, so it cuts at the first keyframe after 4 s, which is 8.33 s.
- ⚠️ **The window was ignored.** An **EVENT** playlist only grows: 11 segments listed at the end. 🧠 `EVENT` means "a live event; segments are only ever appended, never removed", so `hls_list_size` doesn't apply. For a sliding window, omit `-hls_playlist_type`.
- **For real:** camera → **RTMP or SRT** → **transcoder** → **packager** → **origin** → **CDN**.

🧠 RTMP (Flash-era, TCP) and SRT (UDP-based, with retransmission and encryption) are **contribution** protocols: how video gets from the stadium to your encoder. HLS/DASH are **distribution** protocols: how video gets from you to viewers.

### 9.2 The ABR ladder, annotated (slide 43)
*(↩ Week 3 · amortise the setup: share the costly part)*

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
- **`split=3`: decode once, encode three times.** (*Amortise the setup*, Week 3: the costly decode is done once and shared.)
- **`-g 120`: 4 s at 30 fps only.** A **24 fps** source gave **5 s** segments. → Homework 5
- **`-hls_flags`: a 40 s window that never ends, stamped with wall-clock time.**

🧠 **Every flag, explained:**

| Flag | Meaning |
|---|---|
| `-stream_loop -1` | Loop the input forever, so the file becomes an endless "live" source |
| `split=3` / `asplit=3` | Duplicate decoded video/audio into three branches |
| `scale + pad` | Resize each branch to its resolution, letter-boxing to keep the aspect ratio |
| `-map …` | Wire each scaled video + audio pair into an output stream (0, 1, 2) |
| `-c:v libx264 -preset veryfast` | H.264 encoder; `veryfast` trades compression for speed (needed for live) |
| `-profile:v main` | H.264 Main profile, widely compatible (→ `avc1.4d…` in CODECS) |
| **`-sc_threshold 0`** | **Disable scene-cut keyframes.** Otherwise x264 inserts extra keyframes at scene changes and the renditions' keyframes stop lining up, which breaks ABR switching |
| **`-g 120 -keyint_min 120`** | **A keyframe exactly every 120 frames** = 4 s at 30 fps, aligned with `-hls_time 4` |
| `-c:a aac -ac 2 -ar 48000` | AAC audio, stereo, 48 kHz |
| `-b:v:N / -maxrate / -bufsize` | Target bitrate per rendition; `maxrate` = 1.07× target caps peaks; `bufsize` = 1.5× sets how long the encoder may exceed it (VBV) |
| `-hls_list_size 10` | 10 segments × 4 s = **40 s window** |
| `independent_segments` | Every segment starts with a keyframe and can be decoded alone (adds `#EXT-X-INDEPENDENT-SEGMENTS`) |
| `delete_segments` | Delete segments after they leave the window (→ §8.2's bug; pair it with `-hls_delete_threshold`) |
| `omit_endlist` | Never write `#EXT-X-ENDLIST`: it's live, it "never ends" |
| `program_date_time` | Write `#EXT-X-PROGRAM-DATE-TIME` → the spoiler fix (§7.3) |
| `%v` / `-var_stream_map` | One playlist and folder per variant: `stream_0`, `stream_1`, `stream_2` |
| `-master_pl_name master.m3u8` | Also write the master playlist (§6.3) |

🧠 **The keyframe lesson in one line:** *segment length = the first keyframe at or after `hls_time`*. Keyframes are set in **frames** (`-g`), segments in **seconds**, so the frame rate converts between them: `segment = g / fps` → 120/30 = 4 s, 120/24 = **5 s**. Setting keyframes by **time** (`-force_key_frames "expr:gte(t,n_forced*4)"`) removes the dependence on frame rate.

---

## 10 · CLOSING

### 10.1 The whole course in one cricket match (slide 44)

| Week | Where it showed up today |
|---|---|
| **1 · Sockets** | Ports are 16 bits: **28k test sockets per client IP**. **Base64** in the WebSocket handshake |
| **2 · Wire** | Framing: **SSE's blank line, WebSocket lengths, 188-byte packets**. Playlists as control, segments as data |
| **3 · Scale** | **Fifty million held sockets, Little's Law**, and nothing free per connection |
| **4 · nginx** | The origin: **`proxy_cache_lock`, `use_stale`, one location per file type** |
| **5 · HTTP** | **ETag and 304, keep-alive, 101 Upgrade**, and push's death shaping LL-HLS |
| **6 · CDNs** | **Tiers, collapsing, negative caching**, and paying per request or per byte |
| **7 · Failures** | **Jitter, the live-edge 404, stale-if-error, failover**, and who decides "worse" |
| **8 · Video** | **HLS: a playlist, polled like a scorecard, pointing at bytes that never change** |

> **Every week showed up in one match. That was the point of the course.**

### 10.2 Four ideas to keep (slide 45)

1. **Pull scales through caches; push scales through machines.** The same answer for everyone belongs in a CDN. A different answer per person means you're buying sockets.
2. **Number the pieces, then cache them forever.** A small changing index plus numbered pieces that never change: the versioned scorecard and the HLS playlist are one design.
3. **Latency is a budget you spend on safety.** Poll intervals, TTLs, segment length and hold-back all add up. Decide where each second goes.
4. **Know which bill you're on.** Below about 11 KB you pay per request, above it per byte. Scorecards and playlists are the first kind; segments the second.

### 10.3 Final slide (slide 48)
> **Every live stream is a scorecard with bigger bytes.** Thank you for eight weeks.

---

## 11 · HANDS-ON 🛠️

### 11.1 Watch HLS in the browser (the class demo, Homework 1)
1. Open the video.js demo page (videojs.org → "Try it" / the HLS sample, e.g., *Big Buck Bunny*).
2. DevTools → **Network** → filter `m3u8|ts|m4s`. Tick **Preserve log**.
3. Press play. You should see: `master.m3u8` (once) → one `index.m3u8` → a steady stream of segments.
4. Throttle to **Fast 4G**, then **Slow 4G**. Watch the segment URLs change from the high-rendition folder to a lower one: **that's ABR switching** (§7.4).
5. Right-click → **Save all as HAR** to keep two minutes of requests for the homework.

### 11.2 Conditional GET with curl
```bash
# 1) First fetch: note the ETag
curl -sI https://example.com/ | grep -i etag

# 2) Send it back: expect 304 and no body
curl -sI -H 'If-None-Match: "PASTE-ETAG-HERE"' https://example.com/ | head -1
```

### 11.3 Build the stream locally (class commands)
```bash
# get any short mp4 (or record one), then:
mkdir -p out/stream_0 out/stream_1 out/stream_2
# run the ladder command from §9.2 (fill in the scale+pad part, e.g.):
#   [v720]scale=1280:720:force_original_aspect_ratio=decrease,pad=1280:720:(ow-iw)/2:(oh-ih)/2[v720o]
# in another terminal, serve it:
cd out && python3 -m http.server 8000
# play http://localhost:8000/master.m3u8 in VLC, Safari, or an hls.js demo page
```

### 11.4 Verify what the slides claim
```bash
# segment durations: are they really 4 s?
grep EXTINF out/stream_1/index.m3u8

# keyframe times in the source (how far apart are they?)
ffprobe -v error -select_streams v:0 -skip_frame nokey \
        -show_entries frame=pts_time -of csv=p=0 input.mp4 | head

# source frame rate
ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate -of csv=p=0 input.mp4

# is every segment a multiple of 188 bytes? (macOS stat; Linux: stat -c%s)
for f in out/stream_1/*.ts; do s=$(stat -f%z "$f"); echo "$f $s $((s % 188))"; done

# does every packet start with the 0x47 sync byte?
xxd -c 188 -l 940 out/stream_1/seg_000001.ts | cut -c1-20   # each row starts with 47

# watch the sliding window and deletion live (the §8.2 bug)
watch -n 1 'grep -c ts out/stream_1/index.m3u8; ls out/stream_1 | head -3'
```

### 11.5 Push options in 30 seconds
```bash
# SSE: open a stream and watch events arrive (-N = no buffering)
curl -N -H 'Accept: text/event-stream' http://localhost:8080/match/412/events

# MQTT with mosquitto: one terminal subscribes, another publishes a retained score
mosquitto_sub -h test.mosquitto.org -t 'scaler/match/412/score' -q 1 -v
mosquitto_pub -h test.mosquitto.org -t 'scaler/match/412/score' -q 1 -r -m '{"v":1842,"total":"146/4"}'
```

---

## 12 · ASSIGNMENT & HOMEWORK

### 12.1 Assignment: "a match on your laptop" (slide 46)
*(↩ Weeks 5–6 · build it, then measure it honestly)*

```
 1,000 simulated viewers  ──►  mid-tier: an nginx cache in front  ──►  origin: nginx, score.json, and the ffmpeg ladder
 (pollers and players)
```

| # | Task | What to measure / show |
|---|---|---|
| **1 · Collapsing** | Origin requests per second for `score.json` with `proxy_cache_lock` **off**, then **on** | A drop from "hundreds per expiry" to "one per expiry" |
| **2 · Staleness** | For each ball, **seen minus written**, at a **30 s TTL** and at **5 s** | Compare with the TTL-stack slide (§5.2) |
| **3 · Break it** | Serve a stale playlist with **`hls_delete_threshold 1`** and catch the 404s. Fix it; show the clean log | §8.2 |
| **4 · Spoiler** | Show the score ahead of the picture, then **hold updates until `PROGRAM-DATE-TIME` passes them** | §7.3 |

**Deliverable:** code and a **one-page write-up with the numbers**. Due: [date not given on the slide].

🧠 **Hints and starter material.** *Mid-tier nginx (port 8081 → origin on 8080):*
```nginx
proxy_cache_path /tmp/midcache keys_zone=mid:10m max_size=1g inactive=10m;
server {
  listen 8081;
  location = /match/412/score.json {
    proxy_pass http://127.0.0.1:8080;
    proxy_cache mid;
    proxy_cache_valid 200 5s;                 # vary: 30s vs 5s for task 2
    proxy_cache_lock on;                      # toggle for task 1
    proxy_cache_use_stale updating;
    add_header X-Cache $upstream_cache_status;
  }
  location ~ \.m3u8$ { proxy_pass http://127.0.0.1:8080; proxy_cache mid;
                       proxy_cache_valid 200 2s; proxy_cache_valid 404 1s;
                       proxy_cache_use_stale error timeout updating; }
  location ~ \.ts$   { proxy_pass http://127.0.0.1:8080; proxy_cache mid;
                       proxy_cache_valid 200 10m; proxy_cache_valid 404 1s; }
}
```
- **Task 1:** count origin hits with the origin's `access.log`: `awk '{print $4}' access.log | cut -d: -f2-4 | uniq -c` gives requests per second. To make the herd visible, have all 1,000 pollers fire **without jitter**, and make the origin slow (e.g., a tiny Python origin that sleeps 200 ms) so concurrent misses overlap.
- **Task 2:** have a "scorer" script rewrite `score.json` every N seconds with `{"v": n, "written_at": <epoch>}`. Each poller logs `now − written_at` the **first** time it sees version n. Plot or tabulate the max and median per ball. Expect max ≈ poll + TTL(s).
- **Task 3:** run the ladder with `-hls_delete_threshold 1`. Make the mid-tier serve stale playlists for a long time (e.g., `proxy_cache_valid 200 30s` on `.m3u8` + `use_stale error`), then briefly kill the origin. Players hitting segments that the stale playlist lists but the disk no longer has → 404s in the log. The fix: threshold 12 + playlist TTL ≤ 2 s + stale for ≤ 4 s.
- **Task 4:** stamp each score update with `at` (wall-clock). In a small hls.js page, read the current segment's `programDateTime` (hls.js exposes it on `frag.programDateTime` in the `FRAG_CHANGED` event) plus the offset into it, and only render updates with `at` ≤ that time.
- **Simulated viewers:** Python `asyncio` + `aiohttp`. 1,000 coroutines, each `while True: GET; sleep(interval × uniform(0.8, 1.2))`. A "player" coroutine fetches the playlist every 4 s and GETs any new segment.

### 12.2 Homework (slide 47)

| # | Task | 🧠 Hints |
|---|---|---|
| **1** | Play the **video.js demo** with DevTools open: **log two minutes of requests**, then **throttle and catch a rendition switch** | §11.1. Show the segment URL changing folder/bitrate in the HAR, and the time it happened |
| **2** | **Re-run the four bills** for a **15 s poll**, and for a **negotiated $0.02/GB with no request fee**. Which design wins now? | **15 s poll** doubles requests to 19.2 B: S3 ≈ 2 × $4.7K ≈ **$9.4K**; CDN ≈ 19.2 B × $1.20/M ≈ $23K + ~$2K bytes ≈ **$25K**; nginx egress doubles (~$1.7K + LB); **push is unchanged** (it sends per ball, not per poll) → push wins by more. **Negotiated $0.02/GB, no request fee:** CDN at 30 s = 9,600 GB × $0.02 = **~$192** per match (~$384 at 15 s) → **the CDN becomes the cheapest by far**. The lesson: CDN pricing for small objects is a *request-fee* problem, and the fee is negotiable |
| **3** | **Serve the scorecard over SSE on HTTP/1.1, open seven tabs, and explain the one that waits. Then fix it with HTTP/2** | Browsers allow **6 connections per host** over HTTP/1.1. Each SSE tab holds one forever, so tab 7 is queued until another closes. HTTP/2 carries all the streams over **one** connection (default limit is usually 100+ concurrent streams). With nginx: `listen 443 ssl; http2 on;` (browsers need TLS for HTTP/2), and set `proxy_buffering off` for the SSE location |
| **4** | **Read Hotstar's pubsub post** and map each of its five steps to a Week 3 or Week 4 idea | 1 one node: C10K/C10M, fd limits, epoll (W3). 2 split publisher/subscriber: separate the fan-out path, cluster membership costs (W3). 3 ELB wall: the LB holds sockets too; pre-warming (W4 reverse proxy / LB). 4 five clusters bridged: horizontal sharding, bounded cluster size (W3). 5 reverse bridge: subscribers **pull** from the publisher, so the publisher's load scales with nodes, not clients (W3/W4 fan-out, upstreams) |
| **5 ⭐** | **THE ONE TO DO IF YOU DO ONLY ONE:** Run the ladder on a **24 fps** file and **prove from the playlist that the segments are 5 s**. Switch to **time-based keyframes (`force_key_frames`)** and prove they're **4 s** | Make a 24 fps test file: `ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=24 -f lavfi -i sine -t 120 -pix_fmt yuv420p in24.mp4`. Run §9.2 → `grep EXTINF out/stream_0/index.m3u8` shows `5.000000`. Then replace `-g 120 -keyint_min 120` with **`-force_key_frames "expr:gte(t,n_forced*4)"`** (keep `-sc_threshold 0`) → `4.000000`. Confirm with the `ffprobe -skip_frame nokey` command from §11.4 |

---

## 13 · CHEAT-SHEET

| Term | One-line meaning |
|---|---|
| **The brief** | ~1 KB gzipped scorecard, 50M peak / 20M avg viewers, a ball every 25–45 s, 100 events/yr |
| **Pull vs. push** | pull = stateless, cacheable, scales through caches; push = one held socket per viewer, scales through machines |
| **Polling** | GET every N s, `If-None-Match` → 304, **±20% jitter**; staleness ≤ poll + all TTLs |
| **Long poll** | request held until news or ~25 s; answer before the 60 s nginx/ALB idle timeouts; synchronises the herd |
| **SSE** | one never-ending `text/event-stream` response; events delimited by a blank line; `Last-Event-ID` resume; ~6 per browser on HTTP/1.1 |
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
| **ETag** | use the version number: a fingerprint, not a clock; `If-Modified-Since` only has 1 s resolution |
| **TTL stacking** | staleness = poll + every TTL; each CDN TTL ≤ ½ the promise; prefetch/prefresh at ~25 of 30 s |
| **Herd at expiry** | collapsing (`proxy_cache_lock`) · stale-while-updating · prefetch · client jitter |
| **Collapsing needs public** | private / `Set-Cookie` → waiters released one by one; Fastly skips collapsing for 2 min |
| **Cloudflare** | doesn't cache `.json` by default → `cf-cache-status: DYNAMIC` |
| **Negative caching** | 5xx: cache 1–2 s (shield the origin); 404: ~never (Cloudflare default 3 min!); negative TTL ≪ time-to-appear |
| **Diffs** | 4,117 B → 1,088 B gzipped → 183 B JSON Patch; pull: one file with version + recent changes, never `?since=` |
| **HLS** | master (menu, once) → media playlist (polled every target duration) → segments (immutable) |
| **BANDWIDTH** | (video + audio) × 1.1 |
| **Reload rules** | changed: wait ≥ 1 target duration; unchanged: wait ½; same seq, different URI: halt |
| **VOD** | `#EXT-X-ENDLIST` = stop polling; Apple now recommends 6 s segments |
| **MPEG-TS** | 188-byte packets (for ATM), sync `0x47`, 13-bit PID, PAT/PMT; 3–6% overhead; fMP4/CMAF = boxes (TLV) |
| **Segment length** | shorter = more RPS + keyframes; longer = latency; floor ≈ 3 × segment; `-hls_init_time 1` for fast start |
| **Playlist vs. segment** | playlist: tiny, changes, short TTL, stale = dangerous, billed per request; segment: big, immutable, cache forever, per byte |
| **Buffer** | the client's stale cache; bigger = robust but laggy |
| **Latency budget** | encode ~2 + segment 4 + CDN ~2 + hold-back 12 ≈ 20 s |
| **LL-HLS** | parts (~1 s) · blocking reload (`_HLS_msn`, long poll) · preload hints (replaced H2 push, Jan 2020) · delta updates (`EXT-X-SKIP`) → ~3 s hold-back |
| **Spoiler fix** | stamp updates with `at`; reveal when playback's `PROGRAM-DATE-TIME` passes it |
| **ABR** | throughput-based (fastest that fits) vs. buffer-based (BBA, 10–20% fewer rebuffers); policy: a stall is worse than lower quality |
| **Multi-CDN** | DNS (slow), listed twice, Content Steering (polled JSON), client failover (circuit breaker) |
| **Stale playlist bug** | stale index → deleted segment → cached 404 → stall; fix: `-hls_delete_threshold 12`, playlist TTL ≤ ½ TD, stale ≤ 1 TD |
| **Removed segment** | MUST stay for its duration + playlist duration (4 + 40 = 44 s) |
| **Cache rules** | master: minutes · playlist: ≤ ½ TD · `.ts`: long · `score.json`: ≤ ½ promise; gzip playlists; no per-viewer tokens in keys |
| **ffmpeg one-liner** | `-codec: copy` cuts only at existing keyframes → 8.33 s (x264 default GOP 250 @ 30 fps); EVENT playlists ignore the window |
| **Ladder** | `split=3`; `-sc_threshold 0`; `-g 120` = 4 s only at 30 fps (24 fps → 5 s); use `force_key_frames` for time-based |
| **Four ideas** | pull via caches / push via machines · number pieces, cache forever · latency is a budget · know your bill |

---

## 14 · SELF-CHECK QUESTIONS

**1. Using Little's Law, explain why push needs ~1,000× more concurrency than polling for the same audience.**
<details><summary>Answer</summary>

L = λ × W. Polling through a CDN: 1.67M requests/s, each held ~0.03 s at the edge → ~50k in flight. Push: each viewer's socket is held for the whole 4-hour match, so W is hours and L equals the audience, 50M sockets, whether or not anything happens. The rate barely matters; W went from milliseconds to hours.
</details>

**2. A 304 Not Modified "saves" something. What doesn't it save, and why does that matter for the scorecard's CDN bill?**
<details><summary>Answer</summary>

It saves the body bytes, the parse and the re-render. It doesn't save the request, the round trip or the headers. For a 1 KB file, 92% of the CDN bill is per-request fees, so 304s barely reduce the bill. They pay off mostly between CDN tiers (edge ↔ mid-tier ↔ origin), where they replace full copies.
</details>

**3. Every cache layer and the client poll are set to 30 s. What's the worst-case staleness, and how do you get it inside one ball?**
<details><summary>Answer</summary>

Worst case = poll interval + every TTL on the path = 30 + 30 + 30 = 90 s, two or three balls behind. Set each CDN TTL to ½ the promise or less (e.g., 5 s + 5 s) and poll every 15 s → at most 25 s. Also prefetch ("prefresh") the edge before expiry.
</details>

**4. Why does a stray `Set-Cookie` header on `score.json` hurt so much on a CDN?**
<details><summary>Answer</summary>

Request collapsing only works if the first response can be shared. A private response (or one with `Set-Cookie`) can't be handed to the 19,999 waiters, so they're released and sent to the origin one by one. Fastly then marks the object hit-for-pass and skips collapsing for it for 2 minutes. Your origin ends up seeing the full herd.
</details>

**5. You should cache 503s for 1–2 s but 404s almost never. Why the difference?**
<details><summary>Answer</summary>

A briefly cached 503 shields a struggling origin from millions of retries; the origin sees one probe per TTL per edge. A 404 is often for something that's about to exist (the newest live segment, the next match's scorecard). Caching it means serving "not found" long after the file appears (Cloudflare's default is 3 minutes). The rule: a negative TTL must be far shorter than the time it takes the missing thing to appear.
</details>

**6. Why not give each client `score.json?since=1841` so it gets only what it missed? What's the better design?**
<details><summary>Answer</summary>

The query string is part of the cache key, so each version produces a different URL and the hit rate collapses: almost every poll becomes an origin miss. Instead, publish **one** file containing the current version and the last few changes. Every client fetches the same URL, and a client that's behind still finds what it missed. That's exactly a sliding-window HLS playlist.
</details>

**7. Map the versioned scorecard onto an HLS media playlist field by field.**
<details><summary>Answer</summary>

`version` ↔ `#EXT-X-MEDIA-SEQUENCE`; the `recent` changes ↔ the last N segments listed; the poll interval ↔ `#EXT-X-TARGETDURATION`. Both are a small, frequently-polled index pointing at numbered pieces that never change.
</details>

**8. Why are playlists "harder" than segments, even though segments are 3,000× bigger?**
<details><summary>Answer</summary>

Playlists change every target duration, need TTLs ≤ ½ TD, are polled by every viewer (12.5M req/s at 4 s), are billed per request, and are dangerous when stale (they stall the player or point at deleted segments). Segments never change, so they can be cached for minutes or forever and served stale safely; their cost is just bytes.
</details>

**9. Break down the ~20 s live latency with 4 s segments. Which bar does LL-HLS shrink, and with which four ideas?**
<details><summary>Answer</summary>

Encode ~2 s + segment fill 4 s + CDN ~2 s + player hold-back 3 × 4 = 12 s ≈ 20 s. LL-HLS attacks the 12 s hold-back with: parts (~1 s pieces), blocking playlist reload (long polling via `_HLS_msn`), preload hints (name the next part early; replaced HTTP/2 push), and delta updates (`EXT-X-SKIP`). With 1 s parts, hold-back ≈ 3 s (PART-HOLD-BACK ≥ 3 parts).
</details>

**10. What is the spoiler problem and how do you fix it?**
<details><summary>Answer</summary>

The scorecard path (~10 s) is faster than the video path (~20 s), so the overlay shows a wicket ~10 s before the picture does. Stamp every score update with the ball's wall-clock time, and use the segments' `#EXT-X-PROGRAM-DATE-TIME` to know the wall-clock time currently on screen. Reveal an update only once playback has passed its timestamp.
</details>

**11. ffmpeg with `-codec: copy -hls_time 4` produced 8.33 s segments. Why? And why did the ladder give 5 s segments on a 24 fps file?**
<details><summary>Answer</summary>

Segments can only start at keyframes. With `-codec: copy` there's no re-encode, so ffmpeg must use the source's existing keyframes, and x264's default keyframe interval is 250 frames = 8.33 s at 30 fps. The ladder sets `-g 120` (a keyframe every 120 frames), which is 4 s only at 30 fps. At 24 fps, 120 frames = 5 s. Fix: time-based keyframes with `-force_key_frames "expr:gte(t,n_forced*4)"`.
</details>

**12. Walk through the stale-playlist failure chain and its fixes.**
<details><summary>Answer</summary>

`delete_segments` removes segments almost as soon as they leave the window (ffmpeg keeps just one extra) → during an origin blip, the CDN serves a stale playlist that lists an already-deleted segment → the player gets a 404 → the CDN negatively caches it → stall. Fixes: keep removed segments for their duration + the playlist duration (RFC 8216: 4 + 40 = 44 s → `-hls_delete_threshold 12`); playlist TTL ≤ ½ target duration and serve it stale for at most one target duration; serve segments stale freely; treat a playlist as invalid if `Last-Modified` is more than 3 target durations behind `Date`.
</details>

**13. Throughput-based vs. buffer-based ABR: what does each measure, and what's the product policy behind both?**
<details><summary>Answer</summary>

Throughput-based measures how fast recent segments downloaded and picks the highest rendition that fits ("congestion control one layer up"), but it's noisy. Buffer-based watches buffer occupancy: step down when it's nearly empty, step up when it's full. In Netflix's tests it gave 10–20% fewer rebuffers at a similar video rate. The policy behind both: a stall is worse than a lower rendition. Even the starting rendition is a policy choice (Apple: 730 kbps on cellular, 2 Mbps on Wi-Fi).
</details>

**14. Name the four ways to steer players across CDNs and one weakness of each.**
<details><summary>Answer</summary>

DNS per region/ISP: slow to change because resolvers cache. Listing each rendition once per CDN in the master: crude, error-driven fallback only. Content Steering, a JSON file polled every TTL naming the preferred pathway: it reacts only within one TTL and needs the server to make the decision. Client failover with a circuit breaker: a per-client decision, so it can't balance load globally.
</details>

**15. State the four ideas to keep, and give one example of each from today's lecture.**
<details><summary>Answer</summary>

1. Pull scales through caches, push through machines: a CDN serving one scorecard vs. Hotstar's 200 nodes for 50M sockets.
2. Number the pieces, cache them forever: versioned `score.json` and HLS segments `seg_000011.ts`.
3. Latency is a budget spent on safety: poll + TTLs for the scorecard; encode + segment + CDN + 3× hold-back for video.
4. Know which bill you're on: below ~11 KB per request (scorecard, playlist), above per byte (segments).
</details>
