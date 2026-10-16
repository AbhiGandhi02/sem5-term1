# 📓 Building for Failures
### *CN at Scaler — Lesson 7 (Session 7 of 8)* · "fail on purpose"

> `timeouts` · `retries` · `backoff + jitter` · `circuit breakers` · `idempotency` · `fallbacks`
> **classify → stop if structural → retry if transient → trip the breaker → degrade, don't die**

**Legend:** 🧠 = my addition, *not on the slides* · 💡 **Simple words mein** = an easy (sometimes Hinglish) summary. Read it first if a section feels heavy. "(slide 16)" = page number in the deck's footer.

---

## ⭐ WHAT THIS LECTURE IS ABOUT (AND WHY IT MATTERS)

A network request never fails "in general". It fails at a **specific stage** (DNS, TCP handshake, TLS, writing, server processing, reading). The one rule of this lecture: **before you retry anything, find out what kind of failure it is.** Ask *"will anything be different next time?"* If yes, retry *carefully* (limited count, exponential backoff, full jitter, obey `Retry-After`). If the same service keeps failing, stop calling it (circuit breaker). If nothing good is left, **switch to a backup, give a smaller answer, or serve old data — anything but nothing.**

Why it matters: bad retry code doesn't just fail to help — it makes outages **worse** (infinite loops, retry storms, the "thundering herd" that crashes a server again the moment it comes back). The case study is a **video player**: it makes hundreds of requests per play, fails *silently* (a frozen frame, not an error message), and the viewer will only wait a few seconds.

> 💡 **Simple words mein:** Dukaan band hai toh baar baar darwaza khatkhatane se nahi khulegi (don't retry); bas bheed hai toh thoda ruk ke aao (retry); aur kuch na mile toh doosri dukaan se le lo (fallback).

```
  ┌──────────────────────────────────────────────────────────────┐
  │   "Will anything be different next time?"                    │
  │                                                              │
  │    NO  ──▶ don't retry. HANDLE it (refresh token, fall back) │
  │    YES ──▶ retry, but: cap · backoff · jitter · Retry-After  │
  │    KEEPS FAILING ──▶ circuit breaker: stop asking            │
  │    NOTHING WORKS ──▶ failover / degrade / serve stale        │
  └──────────────────────────────────────────────────────────────┘
```

---

## 📋 Table of contents

0. [Housekeeping slides: assignment, project, dates, course map](#0--housekeeping-slides)
1. [Prerequisites from scratch](#1--prerequisites-from-scratch-)
2. [Recap: echo_server.c and sigpipe_client.c](#2--recap-the-code-from-earlier-sessions)
3. [Failure taxonomy: one request, six places to fail](#3--failure-taxonomy-one-request-six-places-to-fail)
4. [HTTP status classes, revisited with one new question](#4--http-status-revisited-with-one-new-question)
5. [Definitions: is this actually a retry?](#5--definitions-is-this-actually-a-retry)
6. [When to retry: classify before you decide](#6--when-to-retry-classify-before-you-decide)
7. [Idempotency: when blind retry is even safe](#7--idempotency-the-question-that-decides-if-blind-retry-is-safe)
8. [How to retry: the naive loop, backoff, jitter](#8--how-to-retry-mechanics)
9. [When the server tells you how long to wait (429 / 503 / 408)](#9--when-the-server-tells-you-how-long-to-wait)
10. [The circuit breaker](#10--the-circuit-breaker)
11. [Case study: a video player's 96 reasons to retry](#11--case-study-a-video-players-96-reasons-to-retry)
12. [Systems that survive: Fastly, stale-if-error, failover, flags, degradation, thundering herd](#12--systems-that-survive)
13. [Synthesis: one flowchart for every failure](#13--synthesis-one-flowchart)
14. [Hands-on](#14--hands-on-)
15. [Assignments and homework](#15--assignments--homework)
16. [Cheat-sheet](#16--cheat-sheet)
17. [Self-check questions](#17--self-check-questions)

---

## 0 · HOUSEKEEPING SLIDES

The deck starts with three admin slides. Two of them are graded work, so read them.

### 0.1 The early HTTP/1.1 assignment — "Build a calculator that stays on the line"

*(Due before Session 7. Full approach + hints are in [§15.1](#151-the-http11-calculator-that-stays-on-the-line).)*

**The task** — any language, no framework, just a socket:

| Request | Expected response |
|---|---|
| `GET /add?a=2&b=3` | `200` body `5` |
| `GET /sub?a=10&b=4` | `200` body `6` |
| `GET /mul?a=6&b=7` | `200` body `42` |
| `GET /div?a=9&b=3` | `200` body `3` |
| `GET /div?a=1&b=0` | `400` (divide by zero) |
| `GET /add?a=x&b=3` | `400` (not a number) |
| `GET /pow?a=2&b=8` | `404` (no such operation) |
| `POST /add` | `405` (wrong method) |
| `GET /add` *(no `Host` header)* | `400` |

**How it will be marked** — *one socket, every request*:

```python
s = socket.create_connection(("localhost", 8080))
# then, on that SAME socket:
GET /add?a=2&b=3   -> 200  5
GET /sub?a=10&b=4  -> 200  6
GET /mul?a=6&b=7   -> 200  42
GET /div?a=1&b=0   -> 400
GET /pow?a=2&b=8   -> 404
POST /add          -> 405

socket still open: True
1 TCP handshake, 6 responses
```

> *"If the socket dies before I am done, you have written a 1996 server."* (HTTP/1.0 was 1996 — one request per connection.)

> *"Don't read `02-http11/server11.py` until you have failed at least once."*

**The hard part:** if the connection stays open, you must answer a question HTTP/1.0 never had: **where does this request end and the next one start?** When the server hung up after every reply, the answer was simply "at EOF". Now you must read **exactly `Content-Length` bytes and not one more — byte n+1 belongs to the next request.**

**Stretch goals (optional):** obey `Connection: close` · an idle timeout you can justify · chunked encoding · receive all six requests at once and answer in order (**pipelining**).

### 0.2 The course project — "HTTP, in binary · two tracks, one protocol"

> *"Now you write the spec."*

| Track 1 — the server | Track 2 — the client |
|---|---|
| `$ ./bserve ./www 9000` | `$ ./bcurl -v localhost:9000/index.html` |
| accept a TCP connection | build the binary request frame |
| read one binary request frame | read the response, body to stdout |
| map the path to a file under a root | `-v` hexdumps every frame |
| reply: status, headers, the bytes | exit non-zero on 4xx / 5xx |
| 404 if it's not there; 400 if the frame is malformed | and **never open a second connection** |
| and keep the connection open | |

**The bit in the middle is the actual project:**
- A **fixed-size frame header** — you choose the fields and their sizes, and you explain why. *HTTP/2 chose 24 / 8 / 8 / 31. Why?*
- **Headers:** give numbers to the ten names you actually send, and send the rest with a length in front — *"HPACK's first two mechanisms, in an evening."*
- One rule you can't skip: **if a receiver gets a frame type it doesn't know, it MUST skip it cleanly.** That keeps room for a version 2.

**What you hand in:** (1) the spec — two pages, enough for a stranger; (2) your program; (3) a hexdump of one full request and response, with notes on each part. *"If you cannot annotate your own bytes, the spec is not finished."*

**In pairs:** one writes the server, one the client; only the spec is shared between you. *"A client that only works against your own server is an implementation, not a protocol."*

🧠 **Decoding "24 / 8 / 8 / 31"** — the HTTP/2 frame header (RFC 9113) is 9 bytes:

```
 ┌──────────────────────────── 24 bits ─────────────────────────┐
 │                     Length (payload size)                    │  up to 16 MB, default max 16 KB
 ├─────────── 8 bits ──────────┬────────── 8 bits ──────────────┤
 │        Type (DATA, HEADERS…) │        Flags (END_STREAM…)     │
 ├─┬────────────────────────────────── 31 bits ─────────────────┤
 │R│                     Stream Identifier                      │  R = 1 reserved bit
 └─┴────────────────────────────────────────────────────────────┘
```
- **Length first** → the receiver knows exactly how many bytes to read next (the same problem as the calculator, solved in binary).
- **8-bit type** → 256 possible frame types; unknown ones are ignored — exactly the "MUST skip cleanly" rule.
- **31-bit stream ID** → many requests share one connection (multiplexed); the spare bit is reserved.
- 🧠 **HPACK** (HTTP/2 header compression) has a *static table* (common header names/values get small numbers — "number the names you send"), *plain strings with a length in front*, Huffman coding and a dynamic table. The slide asks you to copy only the first two ideas.

### 0.3 Dates (from the "Upcoming Quiz Schedule" slide)

| Event | Date |
|---|---|
| Quiz 1 | 21 Sep, Monday |
| Assignment submission | 25 Sep, Friday — marked "Today" on the slide |
| Project submission | 8 Oct |
| Quiz 2 | 12 Oct, Monday |

### 0.4 Course map — week 7 of 8

```
 W1 Network programming 101   W2 Eagle-eye view   W3 Server-side considerations for scale   W4 nginx deep dive
 W5 Evolution of HTTP         W6 (Ab)using CDNs   W7 ▶ BUILDING FOR FAILURES ◀              W8 Streaming video at large scale
```

### 0.5 Agenda — "One discipline: fail on purpose"

| # | Section | Content |
|---|---|---|
| 01 | Recap + failure taxonomy | basic client/server, HTTP status recap, where a request actually breaks |
| 02 | Is this even a retry? | the definition that keeps you out of trouble |
| 03 | When to retry | status codes, idempotency, classify before you act |
| 04 | How to retry | backoff, jitter, caps, circuit breakers — built live |
| 05 | Case study: a video player's 96 failure modes | DNS to DRM to iOS ResetMedia |
| 06 | Systems that survive | failover, degradation, the thundering herd on recovery |

---

## 1 · PREREQUISITES FROM SCRATCH 🧠

This section is background the slides expect you to know. Skip what you know.

### 1.1 What does one HTTP request actually involve?

When you type `curl https://example.com/`, a lot happens before even one byte of HTTP is sent:

```
  YOU                     DNS RESOLVER            SERVER
   │── "IP of example.com?" ──▶│                    │
   │◀── "93.184.x.x" ──────────│                    │        (1) DNS
   │─────────────── SYN ───────────────────────────▶│
   │◀────────────── SYN-ACK ────────────────────────│        (2) TCP handshake
   │─────────────── ACK ───────────────────────────▶│
   │─────────────── ClientHello ───────────────────▶│
   │◀────────────── ServerHello, certificate ───────│        (3) TLS handshake
   │─────────────── key exchange, Finished ────────▶│
   │─────────────── GET / HTTP/1.1 ────────────────▶│        (4) request write
   │                                    [server works]       (5) server processing
   │◀────────────── HTTP/1.1 200 OK + body ─────────│        (6) response read
```

- **DNS (Domain Name System)** — the phone book that turns a name (`example.com`) into an IP address.
- **TCP (Transmission Control Protocol)** — gives you a reliable byte stream, in order. Starts with the 3-way handshake **SYN → SYN-ACK → ACK**.
- **TLS (Transport Layer Security)** — the "S" in HTTPS. Proves who the server is with a **certificate** and agrees on encryption keys.
- **HTTP (HyperText Transfer Protocol)** — the actual request/response text (or binary, in HTTP/2) on top.

Each stage can fail on its own. This is the "six places to fail" slide in §3.

### 1.2 What is a timeout?

A **timeout** is a deadline: "if X hasn't happened in N seconds, stop waiting." Without one, a program can wait **forever** (if the other end of a TCP connection silently disappears, nobody tells you). There are different kinds, and they mean very different things:

| Timeout | Fires when… | Did we reach a server? |
|---|---|---|
| **DNS timeout** | resolver didn't answer | no |
| **Connect timeout** | TCP handshake didn't complete | no |
| **TLS handshake timeout** | TLS didn't complete | reached a host, but no request sent |
| **Read timeout** | connected, request sent, then silence | **yes — and it may have done some of the work** |
| **Total / overall deadline** | the whole thing took too long | maybe |

### 1.3 FIN vs RST — two ways to end a TCP connection

```
   graceful:  A ──FIN──▶ B     "I'm done sending; you finish up"   → read() on B returns 0 (EOF)
   abortive:  A ──RST──▶ B     "connection is dead, drop everything" → B's read()/write() fail (ECONNRESET / EPIPE)
```
- **FIN** = polite hang-up. **RST (reset)** = slammed door; any unsent/unread data is thrown away.
- **SIGPIPE** = the Unix signal a process gets when it `write()`s to a socket whose other side is gone. **By default it kills the process.** Servers usually ignore it (`signal(SIGPIPE, SIG_IGN)`) or use `send(..., MSG_NOSIGNAL)` on Linux, and then handle the `EPIPE` error instead.

### 1.4 What is a cache, a CDN, an origin?

- **Origin** — the "real" server that owns the content.
- **Cache** — a copy kept closer to the user so the origin isn't asked every time.
- **CDN (Content Delivery Network)** — caching servers ("edges") all over the world, run by companies like Fastly, Akamai, Cloudflare. Session 6 covered this.
- **Stale** — a cached copy that is past its "fresh" time. Normally it must be **re-checked (revalidated)** with the origin before use.

### 1.5 What is an idempotent operation?

**Idempotent** = doing it once or ten times gives the same final result.
- "Set my thermostat to 22°C" → idempotent.
- "Turn the thermostat up by 1°C" → **not** idempotent.
- "Charge ₹500 to this card" → **not** idempotent (the classic disaster case).

Note: idempotent is about the **effect on the server**, not the response. `DELETE /photo/7` may return `200` the first time and `404` the second — but either way the photo is gone.

### 1.6 Video streaming vocabulary (for the case study)

| Term | Meaning |
|---|---|
| **Manifest / playlist** | a small text file listing the available qualities and the URLs of every chunk of the video |
| **HLS (HTTP Live Streaming)** | Apple's streaming format; manifest is an `.m3u8` file |
| **DASH (Dynamic Adaptive Streaming over HTTP)** | the open standard; its manifest is the **MPD (Media Presentation Description)**, an XML file |
| **Master vs child manifest** | master lists all renditions (qualities/codecs); each child (media playlist) lists the segments for one rendition |
| **Segment** | a few seconds of video, fetched as a normal HTTP GET |
| **Init segment** | a small header chunk the decoder needs before it can decode segments |
| **Rendition** | one version of the video (e.g. 1080p H.264, 720p HEVC) |
| **Bitrate** | how many bits per second a rendition needs — higher = better quality, more bandwidth |
| **Buffer** | the seconds of video already downloaded but not yet played — your safety cushion |
| **Rebuffering** | the buffer ran dry → the spinner |
| **Live edge** | the newest segment of a live stream — just being produced |
| **VOD (video on demand)** | pre-recorded content; every segment already exists |
| **DRM (Digital Rights Management)** | encryption of the video; the player must fetch a **license/key** from a license server to decrypt it |
| **Widevine L1 / L3** | Google's DRM security levels: **L1** = decryption in secure hardware (allows HD/4K), **L3** = software-only (usually capped to SD) |
| **Codec** | the compression format (H.264/AVC, H.265/HEVC, AV1…). A device can only play codecs its decoder supports |
| **AVPlayer** | Apple's media playback class on iOS |

### 1.7 HTTP headers you'll meet

- **`Retry-After`** — sent by the server with 429/503 (and 3xx): either a number of **seconds** (`Retry-After: 120`) or an **HTTP date**.
- **`Authorization`** — carries your credential (e.g. `Bearer <token>`).
- **`Content-Length`** — exact size of the body in bytes.
- **`Connection: close` / keep-alive** — whether the TCP connection is reused after this response.
- **`Cache-Control: stale-if-error=N`** — "if the origin errors, you may serve a stale copy for up to N seconds."
- **`Idempotency-Key`** — not an official HTTP standard yet, but the common practice (made popular by Stripe) to stop duplicate POSTs.

---

## 2 · RECAP: THE CODE FROM EARLIER SESSIONS

The recap shows two C programs from earlier sessions, as a reminder that **every failure we discuss today finally happens on a socket like these.**

### 2.1 `echo_server.c` — "Send it anything. It sends it back, once."

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

| Line | What it does |
|---|---|
| `socket(AF_INET, SOCK_STREAM, 0)` | create an IPv4 TCP socket; returns a file descriptor |
| `addr = {0}`, `sin_family`, `INADDR_ANY`, `htons(2026)` | "listen on every interface, port 2026" (`htons` = convert to network byte order) |
| `bind` + `listen(server_fd, 1)` | claim the port; become a passive socket with a backlog of 1 |
| `accept` | block until a client connects; get a new fd for that client |
| `read` → `write` → `close` | read **once** (up to 4096 bytes), echo it back, hang up |

**Through the lens of today's lecture — how many failure modes can you spot?** 🧠
- **No return values are checked.** If `bind` fails (port in use), the program just carries on with a broken socket.
- If `read` returns `-1` (error, e.g. the client sent RST), `write(client_fd, buf, -1)` passes `-1` as a size → undefined, garbage behaviour.
- If `read` returns `0` (client closed at once), it writes 0 bytes — harmless, but nothing was "echoed".
- **"Once"**: it reads only one chunk. TCP doesn't mark where a message ends, so a 10 KB message gets cut to whatever the first `read` returned.
- **No timeout**: a client that connects and sends nothing blocks `read` **forever** — and since this server handles one client at a time, *every other client is stuck too*. One slow client = whole service down.
- **SIGPIPE**: if the client is gone when we `write`, SIGPIPE can kill the process → server dead for everyone. The next program causes exactly this.

### 2.2 `sigpipe_client.c` — "SO_LINGER {1, 0} sends RST instead of FIN."

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

**What it does:**
1. Connects to `127.0.0.1:8080`, writes `"hello"`.
2. Sets **`SO_LINGER`** with `{l_onoff = 1, l_linger = 0}`.
3. `close(fd)` — normally sends FIN; with linger `{1, 0}` the kernel instead **sends RST and throws away any unsent data.**

**`struct linger` explained 🧠:**

| `l_onoff` | `l_linger` | Behaviour of `close()` |
|---|---|---|
| 0 | (ignored) | default: returns immediately, kernel sends remaining data then FIN in the background |
| 1 | > 0 | `close()` blocks up to `l_linger` seconds while data drains, then FIN |
| 1 | **0** | **abortive close: RST immediately, buffered data thrown away, no `TIME_WAIT`** |

**Why it's called "sigpipe" client 🧠:** it's a rude client on purpose. The server reads `"hello"` and tries to write the echo back, but the client has already reset the connection. The server's `write` fails with `ECONNRESET`/`EPIPE`, and — if the server hasn't ignored SIGPIPE — the server **gets SIGPIPE and dies.** One bad client killed the server. What exactly you see depends on whether the RST arrives before or after the server's `read`/`write`.

⚠️ **Gotcha:** the client connects to port **8080**, but `echo_server.c` listens on **2026**. Change one of the ports (or run another server on 8080).

**The point of the recap:** these failures aren't abstract. Each one is a real socket event: a DNS lookup that never returns, a SYN with no SYN-ACK, a `read()` that blocks, an RST in the middle of a `write()`.

---

## 3 · FAILURE TAXONOMY: ONE REQUEST, SIX PLACES TO FAIL

**Plain English:** a request passes through six stages, one after another. Knowing *which* stage failed tells you if retrying can help.

```
   1            2               3               4               5                   6
 DNS lookup → TCP handshake → TLS handshake → Request write → Server processing → Response read
 name → IP    SYN/SYN-ACK/ACK  cert + key      bytes leave     their problem,      bytes arrive,
                               exchange        your socket     now yours too       or stop arriving
 └──────────── no server has seen your request ──────────┘└── the server MAY have acted ──┘
```

**Two key messages from the slide:**
1. *"Every failure this session talks about lives on this line. Nothing fails generically — it fails at a specific stage, and the stage decides whether a retry can possibly help."*
2. *"A DNS timeout and a read timeout are both 'timeouts' in your logs and completely different problems: one never reached a server, the other reached one that might have half-done the work."*

### What can go wrong at each stage 🧠

| # | Stage | Typical failure | Error you'd see | Side effects on server? |
|---|---|---|---|---|
| 1 | DNS | no such name (NXDOMAIN); resolver not answering | `getaddrinfo: Name or service not known` / timeout | none |
| 2 | TCP | port closed (RST back); host unreachable; SYN dropped | `ECONNREFUSED`, `EHOSTUNREACH`, connect timeout | none |
| 3 | TLS | bad/expired cert; protocol mismatch; handshake stalls | `certificate verify failed`, handshake timeout | none (no HTTP request yet) |
| 4 | Request write | connection reset while sending | `EPIPE`, `ECONNRESET` | maybe partial — server might have received the request |
| 5 | Server processing | crash, overload, bug, dependency down | `500/502/503/504`, or silence | **unknown — this is the dangerous zone** |
| 6 | Response read | connection dropped mid-body, read timeout | `ECONNRESET`, truncated body, read timeout | **the work was probably done** |

### Why the stage matters — the big asymmetry

```
  Failure BEFORE the request reached the server:
      → nothing happened on the server. Safe to retry (if anything would change).

  Failure AFTER the request reached the server:
      → the server may have done the work.
      → retrying a non-idempotent action might do it TWICE.
```

**Pitfall:** treating every "timeout" as the same thing. Always log *which* timeout (`connect` vs `read`) — they need different handling.

---

## 4 · HTTP STATUS, REVISITED WITH ONE NEW QUESTION

**Plain English:** Session 5 taught what each status class *means*. Today's new question: **"does this code tell me if trying again will give a different result?"**

| Class | Meaning | What actually happened | Retry? |
|---|---|---|---|
| **1xx** | Informational | Interim — keep listening, not a final answer | n/a |
| **2xx** | Success | The server did the thing | no need |
| **3xx** | Redirection | Ask again, somewhere else, maybe with a different method | depends |
| **4xx** | Client error | You sent something the server will refuse **every time you send it unchanged** | usually not |
| **5xx** | Server error | The server admits it failed, but says nothing about whether it changed anything | usually yes |

> *"That single question — 'will anything be different next time?' — is the entire retry decision. Everything from slide 9 onward is that question applied code by code."*

### Notes on each row 🧠
- **1xx** examples: `100 Continue` (go ahead and send the body), `101 Switching Protocols` (WebSocket upgrade), `103 Early Hints`.
- **3xx "depends"**: following a redirect isn't really a "retry" (see §5) — it's a *different* request to a *different* URL. Also, `301/302` historically let clients change POST to GET; `307/308` don't allow changing the method.
- **4xx "usually not"**: the key word is *unchanged*. A `400 Bad Request` stays `400` until you fix the request. Exceptions coming later: `408` and `429` (§9), `401` (handle with a token refresh, §5), and a live-edge `404` (§11.3).
- **5xx "usually yes" — but careful:** "the server failed" does **not** mean "the server did nothing." A `500` can come after the database row was already written. That's why §7 (idempotency) exists.

---

## 5 · DEFINITIONS: IS THIS ACTUALLY A RETRY?

**Plain English:** a retry is pressing the *exact same button again*. If you change anything — a new token, a different server, a changed body — you're not retrying, you're **handling** the failure.

> 💡 **Simple words mein:** Retry = wahi form, wahi window pe dobara jama karna. Handling = form ki galti theek karke ya doosri window pe jaana — woh retry nahi hai.

| RETRY | NOT A RETRY |
|---|---|
| Trying the **exact same step again, with nothing changed**. | A **valid, different action** that looks similar. |
| Same request. Same method, same body, same headers that matter. | Refreshing an expired token and re-sending the request is **not** a retry — it's a different request (new `Authorization` header) responding to information the server just gave you. |
| You're betting that the **SAME** action will get a **DIFFERENT** result, because you believe the failure was **transient** — the network blinked, the server was briefly overloaded, a load balancer picked an unhealthy backend. | Auth failing outright is usually not something a retry fixes at all: it needs a different credential, not another attempt with the old one. |

> **The test:** *if you changed nothing about the request and only waited, would you call the second attempt identical to the first? If the answer is no — new token, different endpoint, altered payload — you are not retrying, you are handling.*

### Why does the distinction matter? 🧠
- **Retry limits** apply to retries. If you count a token refresh as "retry 1 of 3", you waste your limit on a problem that won't go away by itself.
- **Backoff** makes sense for retries (the problem may clear with time). It makes *no* sense for handling: waiting 2 seconds won't fix an expired token — refresh it now.
- **Metrics:** a spike in "retries" should mean "the network/upstream is unstable", not "tokens expire every hour".

### Examples

```
  401 Unauthorized ──▶ refresh token ──▶ send again with NEW Authorization
                       └─ HANDLING (different request). Do it once; if it still fails, stop.

  503 Service Unavailable ──▶ wait ──▶ send the IDENTICAL request
                       └─ RETRY. Apply backoff + jitter + cap.

  CDN A times out ──▶ send the request to CDN B
                       └─ FAILOVER (different endpoint). Not a retry.
```

---

## 6 · WHEN TO RETRY: CLASSIFY BEFORE YOU DECIDE

**Plain English:** before you retry, answer four questions about the failure — what kind it is and how bad it is.

| # | Question | What it distinguishes |
|---|---|---|
| 1 | **Where in the request did it fail?** | Before a connection existed (DNS, handshake) vs after (mid-response) |
| 2 | **Is the server admitting fault?** | 5xx says "I failed". 4xx says "you failed". |
| 3 | **Is the action idempotent?** | Same effect no matter how many times it runs? |
| 4 | **Is this transient or structural?** | Will waiting 200 ms change anything, or is the resource actually gone? |

- **Transient** = temporary, likely to clear by itself (blip, short overload). **Structural** = fixed; won't change unless something else changes (wrong URL, banned country, bad credential).

### 6.1 No point retrying ("mindmap: no point in retrying")

| Failure | Why retrying is useless |
|---|---|
| **404 — the resource is not there** | If it doesn't exist now, it won't exist 200 ms from now — *unless* you have specific reason to believe otherwise (the CDN-propagation exception, slide 22 → §11.3). |
| **Before the handshake completes** | DNS never resolved, or TCP/TLS never finished — you never reached a server, so nothing about "the server" failed. Retrying the **same broken path** just burns time twice. |
| **Geo-blocks** | The request is refused by **policy**, not by accident. No amount of retrying changes geography. 🧠 (often a `403` or `451 Unavailable For Legal Reasons`) |
| **Preflight failures** | The browser/client already decided the real request isn't allowed (**CORS**, **mixed content**). The real request was never sent. |
| **Auth failing outright** | Not "don't handle it" — handle it with a **token refresh**, which is a different, valid flow, not a retry (slide 8 → §5). |

🧠 **Terms explained:**
- **CORS (Cross-Origin Resource Sharing)** — a browser rule. Before some cross-site requests, the browser sends an `OPTIONS` "preflight" asking the server "may page X call you?" If the answer is no, the browser blocks the real request. Retrying gives the same answer.
- **Mixed content** — an `https://` page trying to load an `http://` resource. The browser just blocks it.

⚠️ **Looks like a contradiction:** this slide says "before the handshake completes → no point retrying", but the next slide lists "connection timeout", "DNS server not responding" and "TLS handshake failure on an otherwise-healthy host" as *worth* retrying. Both are true 🧠 — the difference is **structural vs transient**. The slide's words are "the **same broken path**": if the path is really broken, retrying is pointless; if it just had a hiccup, it isn't:
- "DNS **never resolved**" (NXDOMAIN — the name doesn't exist), "connection refused" (nothing listening), a certificate that is simply invalid → **structural**, same answer every time → don't retry the same path.
- "DNS server **not responding** (no answer at all)", a connect **timeout** (SYN lost), a one-off TLS failure on a host that's otherwise fine → **transient**, the network blinked → retry is reasonable.

### 6.2 Worth retrying — "nothing was decided against you"

| SERVER ADMITS FAULT | ENVIRONMENT SHIFTED UNDER YOU |
|---|---|
| 5xx from the server (**500 / 502 / 503 / 504**) | **Network switch mid-flight** (4G → 5G, wifi → cellular) |
| **Connection timeout** — you never got a socket | **CDN edge / load balancer** momentarily can't take the request |
| **Read timeout** — you had one, then it went quiet | **Cache not found** (an upstream fetch will repopulate it) |
| **DNS server not responding** (not "no record" — no answer at all) | **TLS handshake failure** on an otherwise-healthy host |

🧠 The 5xx codes:

| Code | Name | Typical cause |
|---|---|---|
| 500 | Internal Server Error | bug / unhandled exception |
| 502 | Bad Gateway | a proxy/LB got an invalid response from the upstream |
| 503 | Service Unavailable | overloaded or in maintenance (often with `Retry-After`) |
| 504 | Gateway Timeout | a proxy/LB gave up waiting for the upstream |

**Key phrase:** *"nothing was decided against you."* A 4xx is a decision against your request. A timeout or a 5xx is not a decision — it's an accident, and next time may go differently.

**Pitfall — "read timeout is retryable" needs §7!** The slide lists read timeout as "worth retrying", but a read timeout happens *after* the server got your request (stage 6). Retrying **without checking** is only safe if the action is idempotent. That's the next slide.

---

## 7 · IDEMPOTENCY: THE QUESTION THAT DECIDES IF BLIND RETRY IS SAFE

*(callback: RFC 9110 §9.2 — Session 5)*

> 💡 **Simple words mein:** Lift ka button 5 baar dabao, lift ek hi baar aati hai (idempotent). ATM se ₹500 do baar nikalo toh ₹1000 gaye (not idempotent). Idempotency key = bank ka token number: same token dobara aaye toh kaam dobara nahi hota.

| Method | Idempotent? | Retry after timeout? | Why |
|---|---|---|---|
| **GET** | yes | safe, blindly | reading twice changes nothing |
| **PUT** | yes | safe, blindly | "set it to X" twice = set to X |
| **DELETE** | yes | safe, blindly | already gone → already gone |
| **POST** | **no** | **only with an idempotency key** | "charge the card" twice = two charges |

> - *You didn't get a response — you don't know if the server finished the work before the connection died.*
> - *For a non-idempotent action, blind retry can double-execute. The fix isn't "don't retry", it's "make the server able to tell attempt #2 was the same as attempt #1" — an **idempotency key** sent with the request, checked and deduplicated server-side.*

### 7.1 The ambiguity, as a sequence diagram

```
  CLIENT                                   SERVER                    DB
    │── POST /payments {₹500} ────────────▶│                         │
    │                                      │── INSERT charge ───────▶│  ✅ charged
    │                                      │◀── ok ──────────────────│
    │        ✂  connection dies here  ✂    │
    │   (read timeout — no response)       │
    │                                      │
    │   Client's view: "did it work??"     │
    │   - if I DON'T retry → maybe lost the payment
    │   - if I DO retry blindly → maybe charged twice ❌
```

### 7.2 The fix — an idempotency key

```
  CLIENT                                   SERVER
    │  key = uuid4()  (generated ONCE per logical operation)
    │── POST /payments  Idempotency-Key: 7f3a… {₹500} ─▶│ key unseen → do the work,
    │                                                    │ store (key → response)
    │        ✂  connection dies  ✂                       │
    │── POST /payments  Idempotency-Key: 7f3a… {₹500} ─▶│ key SEEN → don't charge again,
    │◀── 201 Created (the SAME stored response) ─────────│ replay the stored response
```

**Rules that make it work** 🧠:
1. The client makes the key **once per real action** (one "Pay" click), and reuses it on every retry of that action. A new key per attempt makes it useless.
2. The server stores *key → result* (status + body). On a repeat, return the stored result.
3. Store it **atomically** (all-or-nothing) with the work (same DB transaction, or a unique constraint on the key), or two retries at the same moment can both get through.
4. If the same key arrives while the first attempt is still running → reply "in progress" (e.g. `409 Conflict`) instead of running it a second time.
5. If the same key arrives with a **different body** → reject it (that's a client bug, not a retry).
6. Keys expire after some time (Stripe uses 24 hours) — long enough to cover all realistic retries.

A minimal server-side sketch (Python, in memory — a real system uses a database or Redis):

```python
seen = {}   # idempotency_key -> (status, body)

def handle_payment(key, amount):
    if key in seen:                     # attempt #2, #3, ... of the SAME operation
        return seen[key]                # replay, don't re-execute
    result = (201, charge_card(amount)) # attempt #1: do the real work
    seen[key] = result                  # (real systems: same transaction as the charge!)
    return result
```

### 7.3 Gotchas 🧠
- **Idempotent ≠ safe.** *Safe* (RFC 9110) = doesn't change server state at all (GET, HEAD, OPTIONS, TRACE). *Idempotent* = may change state, but repeating it changes nothing more (PUT, DELETE, plus all safe methods).
- **The method is a promise, not a guarantee.** A badly written `GET /delete-account?id=7` is not really idempotent, whatever the RFC says. The table assumes the server follows HTTP's rules.
- **PATCH** isn't in the table: in general it's not idempotent ("append item" vs "set field").
- **DELETE returning 404 on the second try** is fine — idempotency is about state, not response codes.

---

## 8 · HOW TO RETRY: MECHANICS

> *"Build the actual retry loop live — count, backoff, jitter, and the circuit breaker that stops it from making things worse."*

### 8.1 The retry loop everyone writes first — and why it's dangerous

```python
while True:
    try:
        return call_upstream(request)
    except TimeoutError:
        continue   # try again. immediately. forever.
```

Three bugs, straight from the slide:

| Missing piece | What goes wrong |
|---|---|
| **No max count** | A permanently-down dependency becomes an **infinite loop**, not a fast failure. |
| **No delay** | Every client hammers the upstream **at wire speed** the instant it starts failing. |
| **No coordination across clients** | When the upstream recovers, every retry loop fires **on the same tick**. This is the **thundering herd**, and it can re-crash a server that had just come back up. |

🧠 **Retry amplification (retries multiplying):** if a struggling server answers slowly, naive clients send *more* requests, which makes it slower, which causes *more* retries. Load goes **up** exactly when capacity goes **down**. A small blip becomes a full outage. Add layers (the app retries 3×, its HTTP library 3×, the load balancer 3× = up to 27 attempts per click) and it gets much worse.

### 8.2 Exponential backoff, with jitter *(built with the class)*

```python
def retry(fn, max_attempts=5,
          base=0.2, cap=8.0):
    for attempt in range(max_attempts):
        try:
            return fn()
        except Retryable as e:
            if attempt == max_attempts - 1:
                raise
            delay = min(cap, base * 2**attempt)
            delay = random.uniform(0, delay)
            #       ^^^^^^^^^^^^^^^^^^^^^^^
            #       "full jitter" — AWS 2015
            time.sleep(delay)
```

**THREE KNOBS, THREE JOBS** (from the slide):

| Knob | Job |
|---|---|
| **MAX ATTEMPTS** | caps total damage — a dead dependency fails in **seconds**, not forever |
| **EXPONENTIAL** `base * 2^attempt` | spaces retries out so load on the upstream **DROPS** over time instead of staying high |
| **FULL JITTER** `uniform(0, delay)`, *not* `delay ± a bit` | **spreads clients out** in time. Without it, every client gets the SAME delay and they all retry together — the herd, just delayed |

> 💡 **Simple words mein:** Sabko "1 ghante baad aana" bologe toh sab ek saath aa jayenge aur counter pe phir bheed. Jitter = har customer ko alag random time dena, taaki line ek saath na lage.

#### Walking through the numbers 🧠 (`base=0.2`, `cap=8.0`, `max_attempts=5`)

| attempt | `base * 2**attempt` | after `min(cap, …)` | actual sleep (full jitter) |
|---|---|---|---|
| 0 fails | 0.2 s | 0.2 s | random in [0, 0.2] |
| 1 fails | 0.4 s | 0.4 s | random in [0, 0.4] |
| 2 fails | 0.8 s | 0.8 s | random in [0, 0.8] |
| 3 fails | 1.6 s | 1.6 s | random in [0, 1.6] |
| 4 fails | — | — | **raise** (no sleep after the last attempt) |

- Worst-case total sleep = 0.2 + 0.4 + 0.8 + 1.6 = **3.0 s**; on average about half that.
- With these defaults the **8 s cap is never reached** — it only starts at attempt 6 (0.2 × 2⁶ = 12.8 → capped to 8). The cap matters when you raise `max_attempts`. Without a cap, attempt 10 would wait 0.2 × 1024 ≈ 205 s.

#### Why jitter? A picture 🧠

```
 1000 clients all fail at t=0.

 Backoff WITHOUT jitter (everyone computes the same delay):
   t=0.2  ████████████████████ 1000 requests  ← spike
   t=0.6  ████████████████████ 1000 requests  ← spike
   t=1.4  ████████████████████ 1000 requests  ← spike
          (fewer total retries than the naive loop, but still a synchronized herd)

 Backoff WITH full jitter:
   0.0-0.2 ▇▇▇▇▇ spread out
   0.2-0.6 ▇▇▇▇  spread out
   ...     smooth, low, decaying load
```

- **"Full jitter"** = sleep a random time between **0 and the full delay**. 🧠 The name comes from Marc Brooker's 2015 AWS Architecture Blog post *"Exponential Backoff and Jitter"*, which compared "no jitter", "equal jitter" (`delay/2 + random(0, delay/2)`), "full jitter" and "decorrelated jitter" in simulation; full jitter did about the least total work and still finished quickly.
- **"Not `delay ± a bit`"** — adding ±10% noise still leaves everyone bunched around the same moment. Spreading over the *whole* range is what really separates them.

#### Things the slide code assumes 🧠
- `import random, time` at the top.
- `Retryable` is *your* exception class (or tuple of classes) for failures you **classified** as transient (§6). Non-retryable errors (a 404, a 400) are not caught and go straight up — *the classification lives in the `except` clause.*
- `raise` with no argument re-raises the last error, so the caller sees the real failure after the final attempt.

**Pitfalls:**
- Retrying at several layers multiplies attempts (see above). Retry at **one** layer, usually the outermost one that knows if the operation is idempotent.
- Forgetting an **overall deadline**. 5 attempts × a 30 s read timeout = 2.5 minutes, plus sleeps. Limit the total time, not just the count.
- Retrying non-idempotent calls without a key (§7).

---

## 9 · WHEN THE SERVER TELLS YOU HOW LONG TO WAIT

| Code | Name | What it means | Retry strategy |
|---|---|---|---|
| **429** | Too Many Requests | Rate limit hit — the server is fine, **you're asking too fast** | Honor the `Retry-After` header; don't back off blindly on top of it |
| **503** | Service Unavailable | Often paired with `Retry-After` during planned maintenance / overload shedding | Same — trust the header over your own backoff curve |
| **408** | Request Timeout | Server gave up waiting for **you** to finish sending | Retry is fine — nothing about your request was rejected |

> - *Connection limits and 5/10-second interval caps (mindmap: "how do we retry") exist for exactly this reason — clients that ignore `Retry-After` and back off on their own schedule can still cumulatively overwhelm a server that told them, explicitly, to slow down.*
> - ***Rule of thumb:** a server that hands you a number is smarter about its own load than your exponential curve is. Use the number.*

### Example exchange 🧠

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
Content-Type: application/json

{"error": "rate limit exceeded"}
```
```http
HTTP/1.1 503 Service Unavailable
Retry-After: Wed, 01 Oct 2026 07:28:00 GMT
```

### Adding `Retry-After` to the retry loop 🧠

```python
def delay_for(attempt, response, base=0.2, cap=8.0):
    ra = response.headers.get("Retry-After") if response is not None else None
    if ra is not None:
        try:
            return float(ra)                  # "Retry-After: 30"
        except ValueError:
            return seconds_until(ra)          # "Retry-After: <HTTP-date>"
    return random.uniform(0, min(cap, base * 2**attempt))   # fall back to full jitter
```
- Many real clients add a **small** jitter on top of `Retry-After` (so 10,000 clients told "30 s" don't all come back at exactly 30.000 s) — but they never wait *less* than the server asked.
- If `Retry-After` is very long (an hour), it's fine to fail instead of waiting — but don't retry *sooner*.

### Why is 408 a 4xx but retryable? 🧠
A 4xx normally means "you sent something wrong". 408 means "you didn't send *anything* (in time)" — e.g. your upload got stuck or a keep-alive connection sat idle. The request itself wasn't judged. Servers usually close the connection after a 408, so retry on a **new connection**.

### Why is 429 a 4xx but retryable? 🧠
Because the problem isn't *what* you sent, it's *how fast* you sent it. The fix is "same request, later" — exactly a retry, as long as "later" means what the server said.

---

## 10 · THE CIRCUIT BREAKER

*(callback: Michael Nygard, *Release It!* (2007) — the book that popularised the pattern)*

**Plain English:** like the electrical breaker (MCB) in your house. When a wire keeps shorting, the breaker **trips** and cuts power so the house doesn't burn. A software circuit breaker stops calling a service that has clearly failed: *"stop asking a question you already know the answer to."*

> 💡 **Simple words mein:** Dost ka phone 5 baar switched off aaya? Ab har minute call mat karo (OPEN). 10 minute baad ek baar try karo (HALF-OPEN); lag gaya toh normal (CLOSED), nahi laga toh phir ruko.

### 10.1 The three states

```
                  failures ≥ threshold
     ┌──────────┐ ─────────────────────▶ ┌──────────┐
     │  CLOSED  │                        │   OPEN   │
     │ requests │                        │ fail fast│◀──────────┐
     │ flow;    │                        │ locally, │           │
     │ failures │                        │ no call  │           │ probe fails
     │ counted  │                        │ attempted│           │ → re-open
     └──────────┘                        └──────────┘           │
          ▲                                   │ cooldown elapses │
          │ probe succeeds                    ▼                  │
          │ → close                     ┌───────────┐            │
          └──────────────────────────── │ HALF-OPEN │ ───────────┘
                                        │ let ONE   │
                                        │ request   │
                                        │ through   │
                                        │ as a probe│
                                        └───────────┘
```

| State | Behaviour (from the slide) |
|---|---|
| **CLOSED** | Requests flow through normally. Failures are counted. |
| **OPEN** | Failure threshold crossed. Requests **fail instantly, locally** — no call is even attempted. |
| **HALF-OPEN** | After a cooldown, let **ONE** request through as a probe. Success → close. Failure → re-open. |

⚠️ Naming trap: "closed" = **healthy** (current flows), "open" = **broken / blocking**. It's the opposite of how a door works, and interviewers love asking it.

### 10.2 Why it exists — the slide's two bullets

1. **Backoff protects the upstream from ONE client retrying too fast. A circuit breaker protects the upstream from ALL the clients** still calling a service everyone already knows is down — **and protects the caller** from waiting the full timeout on every one of those useless calls.
2. *"This is the max-retry-count idea, one level up: instead of capping retries **per request**, cap failures **per DEPENDENCY**, and remember the verdict for a while."*

Picture why point 1's "protects the caller" matters 🧠:

```
  Upstream is dead. Timeout = 5 s. Your service gets 200 req/s.

  WITHOUT breaker: every request waits 5 s then fails
     → 1000 requests stuck in flight at once, threads/connections exhausted,
       YOUR service now falls over too  (cascading failure)

  WITH breaker (open): every request fails in ~0 ms
     → you return a fallback/error immediately; your service stays healthy
```

### 10.3 A minimal circuit breaker 🧠

```python
import time

class CircuitOpen(Exception):
    pass

class CircuitBreaker:
    def __init__(self, threshold=5, cooldown=10.0):
        self.threshold = threshold      # N consecutive failures → open
        self.cooldown = cooldown        # seconds to stay open before probing
        self.failures = 0
        self.state = "CLOSED"
        self.opened_at = 0.0

    def call(self, fn):
        if self.state == "OPEN":
            if time.monotonic() - self.opened_at < self.cooldown:
                raise CircuitOpen("failing fast")      # no network call at all
            self.state = "HALF_OPEN"                   # cooldown over: allow one probe

        try:
            result = fn()
        except Exception:
            self._on_failure()
            raise
        self._on_success()
        return result

    def _on_success(self):
        self.failures = 0
        self.state = "CLOSED"

    def _on_failure(self):
        self.failures += 1
        if self.state == "HALF_OPEN" or self.failures >= self.threshold:
            self.state = "OPEN"
            self.opened_at = time.monotonic()
```

Notes:
- One breaker **per dependency** (per host / per service), not one global breaker. Only count failures that show the dependency is unhealthy (timeouts, 5xx) — a `404` is your problem, not theirs, and shouldn't trip it.
- This toy version isn't thread-safe and lets *every* caller through once HALF_OPEN starts; a real one allows exactly one (or a few) probes at a time.
- Combining with retry: `breaker.call(lambda: retry(fetch))` or the other way round — choose on purpose. Usually each single attempt goes through the breaker, so once it opens, the remaining retries fail fast too.
- 🧠 Real libraries: Resilience4j (Java), Polly (.NET), `pybreaker` (Python); service meshes like Envoy/Istio do "outlier detection", a breaker at the proxy layer.

---

## 11 · CASE STUDY: A VIDEO PLAYER'S 96 REASONS TO RETRY

> *"Real production taxonomy, condensed — what changes when the client is a player, not a browser tab."* The slides shorten a real list of 96 failure types; they show a sample, not all 96.

### 11.1 Why a player's retry logic is harder than an API client's

| Point (from the slide) | What it means |
|---|---|
| An API client fails **visibly** — an exception, a 5xx in a log. A player that handles a failure badly fails **silently**, as a frozen frame or a spinner the user stares at. | No stack trace, just a bad experience — and users leave. |
| One "play" makes **hundreds of requests** — manifest, key, and dozens of segments — each with a **different retry policy**, on **one continuous timeline the user watches**. | The failure budget is shared by the whole session, not per request. |
| Some failures **aren't network failures at all**: a DRM key can be denied, a codec can be unsupported, a decoder can crash — and the right response is often not "retry" but "**switch to a different stream entirely**." | Retry is only one of several tools. |
| The time budget is a **human** one: a viewer accepts a few seconds of rebuffering, not minutes of perfectly-correct exponential backoff. | A "correct" 2-minute backoff is a *wrong* answer for a player. |

The request timeline of one play session 🧠:

```
 tap ▶
  │─ DNS ─ GET master manifest ─ GET child manifest ─ POST DRM license ─ GET init segment
  │─ GET seg 1 ─ GET seg 2 ─ GET seg 3 ─ … ─ GET seg 300 (+ ad requests interleaved)
  │
  └── each arrow has its own timeout, retry count, and fallback
```

### 11.2 Network issues — same taxonomy, tighter clock *(mindmap: "network issues")*

| Failure | What's actually different for a player | Retry? |
|---|---|---|
| **DNS server not responding** | Same as any client — but a player often has a **fallback CDN with a different hostname** to try instead | retry, or switch CDN |
| **Connection / read timeout** | **Differential timeouts by request type** — a manifest fetch and a segment fetch don't get the same budget | yes, tuned per type |
| **Network switch (4G→5G)** | **IP address can change mid-download** — the in-flight request is **dead, not slow** | yes, fresh request |
| **Wifi off mid-playback** | **Buffer keeps playing while offline** — retry only needs to start once buffer runs low | yes, deferred |
| **CDN edge not responding** | **Multi-CDN** players fail over to a second CDN rather than retry the same one | switch, don't retry |

Explanations 🧠:
- **Differential (different-per-type) timeouts:** the manifest is tiny and blocks everything (no manifest → nothing plays), so it gets a short timeout and quick retries. A segment is bigger; its budget depends on how many seconds of buffer are left. Some players also scale the timeout with the segment's length.
- **Network switch:** a TCP connection is tied to the 4-tuple (src IP, src port, dst IP, dst port). New IP → the old connection can never finish, so waiting is pointless — drop it and open a new one at once. (🧠 QUIC/HTTP/3 has *connection migration* to survive this.)
- **Wifi off, deferred (delayed) retry:** with 30 s of buffer, no need to start retrying hard right away — the user sees nothing wrong yet. Retry urgently only as the buffer runs out.
- **"Switch, don't retry"** = failover (§12.3), which by the §5 definition is not a retry at all.

### 11.3 When 404 doesn't mean "give up"

| THE EXCEPTION: 404 on a live segment | CUSTOM CODES: 419 / 474 / 475 — not in the RFC |
|---|---|
| General rule from slide 11: 404 means don't retry, the resource isn't there. **Exception:** a live stream's newest segment can 404 for a real reason — the origin is **still encoding** it, or the CDN **hasn't finished propagating** it yet. It **WILL** exist in a few hundred milliseconds. | Real deployments layer application-specific codes on top of the HTTP registry: **419** (token/session expiry — handle like auth, not a retry) and **474/475** (internal codes seen when a segment is retried in a way that returns a **mismatched, partial result**). |
| The fix isn't "retry blindly" — it's a **short, bounded retry (2-3 attempts, small fixed delay)** specifically for **live-edge segment requests, and nowhere else**. A 404 on an **on-demand** asset still means give up. | The lesson isn't the specific numbers — it's that **your OWN system's status codes deserve the same classify-before-retry discipline as RFC ones.** Document what they mean and whether they're retryable, the same way this deck documented 5xx. |

```
 Live stream timeline:
   origin encoding:  [seg 100 ✓][seg 101 ✓][seg 102 … still encoding …]
   player asks for 102 too eagerly → 404
   retry after ~300 ms → 404 → retry after ~300 ms → 200 ✓
   (bounded: give up after 2–3 tries, then e.g. step back from the live edge)
```

🧠 Note the **fixed** delay here, not exponential: the expected wait is short and known (about one segment's spreading time), and the viewer's patience is tiny.

🧠 About the custom codes: HTTP status codes are an official list kept by IANA; 419, 474, 475 aren't in it. (419 is used by some frameworks, e.g. Laravel's "Page Expired" for an expired CSRF/session token.) A client that doesn't know a code treats it as the plain `x00` of its class (RFC 9110), i.e. 4xx → 400 — so it may make the wrong retry decision. Hence "document them".

### 11.4 Sometimes the answer isn't retry — it's fall back *(mindmap: "codec/DRM fallback", "stream issue")*

| Failure | What to do (from the slide) |
|---|---|
| **DRM key denied / L1 vs L3** | Retrying the same license request returns the same denial. **Fall back** to a lower DRM security level or a different key system. |
| **Decoder / codec unsupported** | **Not transient** — the device genuinely cannot decode this. Fall back to an **alternate codec rendition** in the manifest. |
| **Master manifest failure → MPD to HLS** | If the DASH manifest fails, some players fall back to an **HLS rendition of the same content** rather than retrying DASH. |
| **Child manifest / segment / init failure** | Retryable in the network sense (section 4 backoff applies) — but a repeated failure **escalates to a fatal, coded error** (e.g. 8018, 8022), not an infinite retry loop. |
| **Ad failure or timeout** | Falls back to **content**, not to a retried ad — the user's time budget for an ad failure is near zero. |
| **iOS ResetMedia** | A platform-specific recovery: **tear down and rebuild the entire AVPlayer pipeline**, because the failure is in **device media state**, not in a request at all. |

Explanations 🧠:
- **L1 vs L3:** a premium title may need Widevine L1 (hardware-secured) for HD. A device with only L3 is denied the HD key. Asking again won't give it a secure chip — instead ask for the SD rendition that L3 is allowed, or use another key system (e.g. PlayReady, FairPlay on Apple).
- **Codec fallback:** a manifest may offer the same video in HEVC and H.264. If HEVC decoding fails, switch to the H.264 rendition.
- **MPD → HLS:** different manifest format, maybe different packaging/hosts → a truly different path.
- **"Fatal, coded error (e.g. 8018, 8022)":** these are the player's own error codes (the slides don't say which player). The point: the retry loop **ends** with a specific, documented error the app can show / log / alert on.
- **Ads:** the viewer came for the content. If the ad breaks, skip it — a spinner to retry an ad is the worst possible trade.
- **ResetMedia:** sometimes the network is fine but the media system itself is stuck. Then the thing you "retry" is the whole player, not a request.

By §5's definition, almost every row here is **handling, not retrying** — each changes something (security level, codec, format, content, the whole pipeline).

### 11.5 Not every retry decision is a network decision

| Idea | Detail from the slide |
|---|---|
| **Business-rule-based retries** | e.g. retry a payment-gated stream start **only if the user's entitlement check itself times out** — not if it cleanly returns "not entitled." The HTTP layer succeeded; the **business layer made a decision.** |
| **Change player behavior based on information in the response** | The response **body**, not just the status code, can carry a message — "**degraded, use lower bitrate**" or "**maintenance, try again in N minutes**" — that tells the player what to do next much more exactly than a status code can. |
| **The general lesson** | Sections 3–4 classified failures using **transport-layer signals** (status code, timeout type). Real systems also carry **APPLICATION-layer signals**, and a good retry policy reads **both**. |

```
  entitlement check ── timeout ──────────────▶ retry (nothing was decided)
                    ── 200 {"entitled": false} ▶ STOP. Show "subscribe" screen.
                       (HTTP success, business "no" — a structural answer)
```

---

## 12 · SYSTEMS THAT SURVIVE

> *"What survives isn't the request — it's the system."* Failover, graceful degradation, and the thundering herd that shows up right when everyone thinks the outage is over.

### 12.1 "49 minutes, and the thing that was still serving traffic" *(callback: Session 6 — the edge, revisited)*

| 85% | ~1 min | 49 min |
|---|---|---|
| of Fastly's network returning errors | to detect the fault | to 95% recovered — **8 Jun 2021** |

🧠 What happened (from Fastly's public incident report): a software bug shipped in May 2021 sat hidden until one customer pushed a valid config change on 8 June that triggered it, so most of Fastly's edge started returning errors. Big sites (news outlets, Reddit, Amazon, gov.uk, and others) went down worldwide for most of an hour. Detection was fast; disabling the config and rolling back took longer.

- **Session 6's closing question:** today's edge survives your origin dying, because a cache can serve a **stale hit**. *What does your system do when the edge — or a whole region — is the thing that dies?*
- **`stale-if-error` (RFC 5861)** is the same idea in the other direction: a cache set up to **keep serving its last good copy** even when it can't reach the origin to re-check it. **Not a retry at all — a choice to serve something known to be old rather than nothing.**
- **The lesson goes beyond caches:** a system that can answer with **slightly-wrong data** survives outages that a **perfectly-fresh-or-nothing** system does not.

> 💡 **Simple words mein:** Taaza samosa khatam? Thoda purana garam karke de do (stale) ya chai-biscuit de do (degrade) — customer ko khaali haath mat bhejo.

🧠 Example header:
```http
Cache-Control: max-age=60, stale-while-revalidate=30, stale-if-error=86400
```
"Fresh for 60 s; for 30 s after that, serve stale while refreshing in the background; and if the origin is erroring, keep serving the stale copy for up to a day."

### 12.2 Failover, feature flags, graceful degradation

| FAILOVER — *Primary → backup* | FEATURE FLAGS — *Turn off the expensive part* | GRACEFUL DEGRADATION — *Answer smaller, not "no"* |
|---|---|---|
| A **circuit breaker** (slide 18) trips OPEN on the primary and **routes to a backup** — different region, different provider, different CDN. The client doesn't retry a dead dependency; **it stops asking it and asks someone else.** | During an incident, a flag can **disable a non-essential feature** (recommendations, personalization) to **shed load off the critical path** (playback itself), **without a deploy**. This is graceful degradation as a **lever you can pull in seconds**. | **Lower bitrate** instead of full quality. **Cached recommendations** instead of live ones. A **slightly stale price** instead of a spinner. Every case study today has one: the player's own **buffer**, the CDN's **stale-if-error**, a **lower DRM tier**. |

🧠 Definitions:
- **Failover** — switch from a failed primary to a standby. *Active-passive*: backup sits idle until needed. *Active-active*: both serve traffic; if one dies, its load moves to the other (which must have spare capacity!).
- **Feature flag** — an on/off switch for code, read from config while running (not built into the code), so ops can flip it during an incident.
- **Graceful degradation** — keep the core working at lower quality instead of failing fully. The opposite is a "brittle" (easily broken) system: one missing piece → whole page errors.

```
  Netflix-style home page when the recommendation service is down 🧠:
    brittle:   [ 500 Internal Server Error ]
    graceful:  [ ▶ Continue watching ] [ Popular in India (cached 2 h ago) ] …
```

### 12.3 The second outage: everyone recovering at once *(callback: slide 16 — jitter, at fleet scale)*

- **The moment a dependency comes back up is the most dangerous moment of the whole incident** — not the failure itself.
- Every client that was retrying, backing off, or waiting on a breaker's cooldown now tries again, and **without jitter they tend to move together** — they all started failing at about the same time, so their backoff timings line up.
- The recovering service, at **1% of normal capacity** while it fills caches and reopens connections, gets **close to 100% of normal traffic** in its first second — and can crash again.
- **Mitigations (fixes):** **jitter** (already have it, slide 16) · **staggered circuit-breaker half-open probes** rather than all-at-once · **slow-start** on the recovering side (accept traffic at a slowly rising rate, not instantly at full capacity)

> 💡 **Simple words mein:** Bijli wapas aate hi poore mohalle ke AC aur fridge ek saath on → transformer phir ud gaya. Isliye load dheere dheere badhao.

```
 capacity vs incoming load during recovery

   load │          ┌─ every client fires at once (no jitter)
        │          █
        │          █              ← exceeds capacity → crashes AGAIN
        │          █   ▄▄▄
   cap  │- - - - - █ - - - - - - - - - - - - - - - - -   (recovering service, mostly cold)
        │__________█___________________________________ time
                   ↑ service comes back

   load │          with jitter + staggered probes + slow-start:
   cap  │- - - - - ▁▂▃▄▅▆▇█ ramps up with capacity - - -
        │__________________________________________ time
```

🧠 Why is the recovering service at "1% capacity"? Its caches are empty (every request goes to the database), its connection pools are empty (every request pays the handshake cost), JIT-compiled code is cold, new auto-scaled machines are still booting. 🧠 Load balancers have this built in, e.g. nginx's `slow_start` parameter on upstream servers (commercial version) and HAProxy's `slowstart`.

---

## 13 · SYNTHESIS: ONE FLOWCHART

> *"One flowchart for every failure this session covered."*

```
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ 1. CLASSIFY      Where did it fail? Server-admitted (5xx) or              │
  │                  client-refused (4xx)? Idempotent?                        │
  └───────────────────────────────┬───────────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ 2. STRUCTURAL?   404 / geo-block / preflight / bad auth → STOP.           │
  │                  Handle, don't retry.                                     │
  └───────────────────────────────┬───────────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ 3. TRANSIENT →   Backoff + full jitter. Cap attempts.                     │
  │    RETRY         Honor Retry-After if given.                              │
  └───────────────────────────────┬───────────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ 4. DEPENDENCY-   Repeated failures trip a circuit breaker —               │
  │    LEVEL         stop asking, fail fast.                                  │
  └───────────────────────────────┬───────────────────────────────────────────┘
                                  ▼
  ┌───────────────────────────────────────────────────────────────────────────┐
  │ 5. NO GOOD       Fail over, degrade gracefully, or serve stale —          │
  │    ANSWER LEFT   anything but nothing.                                    │
  └───────────────────────────────────────────────────────────────────────────┘
```

Mapping each lecture section onto the flowchart:

| Step | Lecture sections |
|---|---|
| 1 Classify | six stages (§3), status classes (§4), four questions (§6), idempotency (§7) |
| 2 Structural | "no point retrying" (§6.1), retry vs handling (§5), entitlement "no" (§11.5) |
| 3 Retry | naive loop → backoff + jitter (§8), Retry-After (§9), live-edge 404 (§11.3) |
| 4 Dependency | circuit breaker (§10) |
| 5 No answer | fallbacks (§11.4), stale-if-error, failover, flags, degradation (§12) |

---

## 14 · HANDS-ON 🛠️

All of these are 🧠 suggested exercises built around the slide content (the slides don't list commands). They work on macOS and Linux.

### 14.1 See the six stages of one request with curl

```bash
curl -s -o /dev/null https://example.com/ -w '
  dns       : %{time_namelookup}s
  tcp       : %{time_connect}s
  tls       : %{time_appconnect}s
  1st byte  : %{time_starttransfer}s
  total     : %{time_total}s
'
```
Expected output (numbers vary):
```
  dns       : 0.012s
  tcp       : 0.045s
  tls       : 0.120s
  1st byte  : 0.210s
  total     : 0.212s
```
Each value is **cumulative** from the start. That's stages 1→2→3, then 4+5 up to the first byte, then 6. *What it shows:* the timeline from §3, measured.

### 14.2 Make each stage fail on purpose

```bash
# Stage 1 — DNS: name doesn't exist (structural → don't retry)
curl https://this-does-not-exist.invalid/
#  curl: (6) Could not resolve host: this-does-not-exist.invalid

# Stage 2 — TCP: nothing listening (connection refused, fast RST)
curl http://127.0.0.1:9/
#  curl: (7) Failed to connect to 127.0.0.1 port 9 ... Connection refused

# Stage 2 — TCP: SYN goes into a black hole (connect TIMEOUT, slow)
curl --connect-timeout 3 http://10.255.255.1/
#  curl: (28) Connection timed out after 3001 milliseconds

# Stage 3 — TLS: bad certificate
curl https://expired.badssl.com/
#  curl: (60) SSL certificate problem: certificate has expired

# Stage 6 — read: server accepts but never answers (read timeout)
nc -l 8081          # in terminal A (macOS); on Linux: nc -l -p 8081
curl --max-time 3 http://127.0.0.1:8081/   # in terminal B
#  curl: (28) Operation timed out after 3001 milliseconds with 0 bytes received
```
*What it shows:* refused vs timed out — two very different failures that both look like "can't connect". A refused connection fails in milliseconds; a black-holed one (packets silently dropped) uses up your whole timeout.

### 14.3 curl's built-in retry policy

```bash
curl --retry 4 --retry-max-time 20 -v http://127.0.0.1:8082/
```
curl's `--retry` retries only "transient" errors: timeouts and HTTP 408, 429, 500, 502, 503, 504. It waits 1 s, then doubles each time (unless `--retry-delay` is set), and modern curl obeys `Retry-After`. `--retry-all-errors` makes it retry everything — by this lecture's logic, usually a mistake. *What it shows:* a real tool doing classify + cap + backoff.

### 14.4 Watch the RST from `sigpipe_client.c`

```bash
# terminal A: compile and run the echo server (port 2026)
gcc echo_server.c -o echo_server && ./echo_server

# terminal B: change htons(8080) → htons(2026) in sigpipe_client.c first, then
gcc sigpipe_client.c -o sigpipe_client

# terminal C: capture loopback traffic on port 2026
sudo tcpdump -i lo0 -n 'tcp port 2026'      # Linux: -i lo

# terminal B:
./sigpipe_client
```
Expected in tcpdump: the handshake (`[S]`, `[S.]`, `[.]`), the 5-byte `"hello"` (`[P.]`, `length 5`), then **`[R.]` or `[R]`** — a reset instead of `[F.]`. Run it several times; sometimes the echo server may die (SIGPIPE) or misbehave because it doesn't check errors. Then try the same with the `setsockopt` line removed: you'll see `[F.]` instead.

### 14.5 A misbehaving test server (start of the "Things to try" assignment)

```python
# flaky_server.py — python3 flaky_server.py
import random
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/sometimes-500" and random.random() < 0.5:
            return self.reply(500, b"boom\n")
        if self.path == "/always-503-with-retry-after":
            return self.reply(503, b"busy\n", {"Retry-After": "2"})
        if self.path == "/always-down":
            self.connection.close()      # drop the socket without replying
            return
        self.reply(200, b"ok\n")

    def reply(self, code, body, headers={}):
        self.send_response(code)
        for k, v in headers.items():
            self.send_header(k, v)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

ThreadingHTTPServer(("127.0.0.1", 8082), H).serve_forever()
```
Try:
```bash
for i in $(seq 10); do curl -s -o /dev/null -w '%{http_code}\n' localhost:8082/sometimes-500; done
curl -i localhost:8082/always-503-with-retry-after       # see Retry-After: 2
curl -v localhost:8082/always-down                       # curl: (52) Empty reply from server
```

### 14.6 Simulate the herd (homework 1 starter)

```python
import random
N, BASE = 1000, 0.2
naive  = [0.0] * N                                   # everyone retries at the same instant
nojit  = [BASE * 2**0] * N                           # same delay for all → one spike at 0.2 s
full   = [random.uniform(0, BASE * 2**0) for _ in range(N)]   # spread across [0, 0.2]
# bucket each into 10 ms bins and print the max bin size: that's the peak load
```
*What it shows:* jitter doesn't reduce total retries, it reduces the **peak**.

---

## 15 · ASSIGNMENTS & HOMEWORK

### 15.1 The HTTP/1.1 calculator that stays on the line

**Restated:** write a plain-socket HTTP/1.1 server (no framework) on `localhost:8080` implementing `/add`, `/sub`, `/mul`, `/div` with query params `a` and `b`, returning the status codes in the table in §0.1, and **keeping the TCP connection open** so all six graded requests travel over one connection.

**Approach / hints** (not a full solution):

1. **Accept loop**, and for each connection a **request loop**:
   ```
   accept()
   └── loop:
         read until you have a full header block (ends with "\r\n\r\n")
         parse request line + headers
         if Content-Length: read EXACTLY that many more body bytes
         handle → write response (WITH Content-Length!)
         if client said Connection: close → close; else keep looping
         if read() returns 0 (EOF) → client hung up → close
   ```
2. **Keep a buffer per connection.** One `recv()` may hold half a request, or one and a half requests. Add to a buffer, cut full requests off the front, and **keep the leftover bytes for the next request** — that's "byte n+1 belongs to somebody else". This is the core of the assignment.
3. **Every response must say where it ends**, or the client can't tell when your body stops: always send `Content-Length: <len(body)>` (or chunked encoding as a stretch).
4. **Status rules to check:**
   - HTTP/1.1 requires a `Host` header; missing → **400** (RFC 9112 says a server MUST answer 400 here).
   - Unknown path (`/pow`) → **404**.
   - Known path, wrong method (`POST /add`) → **405**; 🧠 RFC 9110 says a 405 must include an `Allow: GET` header.
   - `a`/`b` missing or non-numeric, or `b == 0` for `/div` → **400**.
   - Order of checks matters: think about which error wins for `POST /pow` (404 or 405?) and be consistent.
   - 🧠 `POST /add` might carry a body — you must still **read and discard** `Content-Length` bytes of it, or those bytes will be parsed as the start of the next request.
5. **Integer vs float:** `9/3` should print `3`, not `3.0`. Decide what `7/2` returns and document it.
6. **Test like the grader:** `socket.create_connection(("localhost", 8080))`, send each request with `s.sendall(...)`, read exactly one response each (parse its `Content-Length`!), then check the socket is still open.
7. **Stretch goals:**
   - `Connection: close` → send the response, then close.
   - **Idle timeout you can justify** — e.g. `settimeout(...)` on the client socket; links straight to today's lecture: a client that connects and goes silent must not block your server forever. Pick a number and explain it (nginx's default `keepalive_timeout` is 75 s, for comparison 🧠).
   - **Chunked encoding** — `Transfer-Encoding: chunked`: each chunk is `<hex length>\r\n<data>\r\n`, ending with `0\r\n\r\n`.
   - **Pipelining** — client sends all six requests in one write; your buffer logic from hint 2 should already handle it if you answer in order.

Quick manual test 🧠:
```bash
printf 'GET /add?a=2&b=3 HTTP/1.1\r\nHost: x\r\n\r\nGET /mul?a=6&b=7 HTTP/1.1\r\nHost: x\r\n\r\n' | nc localhost 8080
# expect two complete responses, bodies 5 and 42, on one connection
```

### 15.2 Course project — HTTP in binary (pairs)

Restated in §0.2. Hints 🧠:
- Write the spec **first**, together; then split server/client. Test against your partner's code, never only your own.
- Frame header idea to *justify*, not copy: `length | type | flags | stream/request id`. Put **length first** so a receiver can always skip a frame it doesn't understand — the required "MUST skip unknown types cleanly" rule.
- Choose a byte order (network byte order = big-endian) and say so in the spec.
- Header compression: assign numbers 1–10 to the header names you actually send (`:path`, `content-length`, `content-type`…); anything else is sent as `length-prefixed name` + `length-prefixed value`.
- Your `-v` hexdump is your debugger; annotate it byte by byte for the hand-in.
- Status: 404 if the file isn't under the root (and refuse `..` path escapes 🧠), 400 if the frame is malformed, and keep the connection open.

### 15.3 "Things to try" — wrap a flaky client in a policy *(due before Session 8)*

**Restated:**
- Take a client you already have (the calculator client from Session 5, or any HTTP client) and wrap it in a **failure-handling wrapper you write from scratch.**
- The wrapper must:
  1. **classify** the failure (retryable vs not, using this session's rules)
  2. **back off with full jitter**, capped at a max attempt count
  3. **honor a `Retry-After`** header when present
  4. **open a circuit after N consecutive failures** to the same dependency and **fail fast while open**
- Point it at a server you also write, one you can tell to misbehave: `/sometimes-500` (fails a settable fraction of the time) · `/always-503-with-retry-after` · `/always-down` (to trigger the circuit breaker)
- **Deliverable:** the wrapper, the test server, and a short **log/plot showing request timestamps** under each failure mode — the same kind of evidence Session 6's assignment asked for. **Show the backoff curve, the circuit opening, and it closing again after the server recovers.**

**Approach:**
- Starting points: §8.2 (retry), §9 (Retry-After), §10.3 (breaker), §14.5 (test server).
- Make the failure fraction configurable, e.g. `/sometimes-500?p=0.3` or an env var.
- To show the circuit **closing again**, the server needs a way to "recover" — e.g. `/always-down` that becomes healthy after T seconds, or a `/toggle` admin endpoint.
- Log every attempt as `timestamp, attempt#, outcome, sleep_chosen, breaker_state`. Plot timestamp (x) vs attempt (y): the gaps should grow roughly exponentially with some randomness; while OPEN you should see instant failures with **no** server-side log lines (prove "no call is even attempted" by also logging on the server).
- Be careful with classification: should `/always-down` (connection dropped, no response) be retryable for a `POST`? (Only with an idempotency key — §7.)

### 15.4 Homework — "Five, do at least one"

| # | Task (restated) | Hints |
|---|---|---|
| 1 | Take the **naive retry loop** (slide 15) and the **backoff+jitter** version (slide 16); simulate **1,000 clients** against a **rate-limited server** and count total requests in the **first second** under each. Explain the gap. | A simple step-by-step simulation is enough: track each client's next-attempt time, and a server that accepts K req/s and rejects the rest. The naive loop's count is limited only by how fast each failure returns — it's huge. Backoff+jitter totals at most 5 attempts per client and spreads them out. |
| 2 | Pick **three items** from the video-player case study (section 5) that **aren't retries at all** — write one sentence each on what they ARE instead, using the definition from slide 8. | Candidates: switch CDN (failover), DRM L1→L3 (fallback to a different key request), codec fallback, MPD→HLS, ad → content, iOS ResetMedia (rebuild pipeline), 419 token expiry (auth handling). Each changes *something* about the request. |
| 3 | Design an **idempotency key scheme** for a POST endpoint of your choice. **Where does the server store seen keys, and for how long?** | §7.2. Think: DB table with UNIQUE(key) in the same transaction as the work vs Redis with TTL; store the response to replay; scope keys per user; TTL ≥ your clients' maximum total retry window (and the "in-progress" state). |
| 4 | Read the **Fastly postmortem** in full (linked in notes) and map its timeline onto the **five-step framework (slide 29)** — which step, if it had existed, would have shortened the 49 minutes? | Note detection took ~1 minute, so the delay was in *mitigation*. Consider step 5 for *customers* (multi-CDN failover, `stale-if-error`) and for Fastly itself (a feature flag / fast config rollback). |
| 5 | **The one to do if you do only one:** instrument a **real client you use daily** (a CLI tool, a script) that currently has no retry logic. Add **classify + backoff + jitter + a cap**. **Break something on purpose** (kill the wifi, block the port) and watch it **recover instead of hang**. | Good targets: a script using `requests`/`urllib`. Don't forget **timeouts** — without a timeout the client never *fails*, so retry logic never runs ("hang" vs "recover"). Block a port with a firewall rule, or point it at `nc -l` that never answers. |

### 15.5 Next session — Week 8: Streaming video at large scale (the last one)

- Today's retry policy, circuit breaker, and fallback logic were for a general client. Week 8 puts them **inside a real video pipeline**: **adaptive bitrate, segment prefetch**, and the **CDN/tiered-cache** material from Session 6 all come back.
- The video-player case study was a **preview** — next session DRM tiers, multi-CDN switching and manifest fallback stop being examples and become the thing we design.
- **Closing question to think about:** *every fallback discussed today — stale cache, backup region, lower bitrate — answers with something WORSE than the ideal. At what point does "worse" stop being acceptable, and who decides?* 🧠 (Hint: it's a product/business decision written as SLOs (service-level targets) — e.g. "stale prices are OK for 5 minutes, never for checkout totals"; a stale news page is fine, a stale bank balance isn't.)

---

## 16 · CHEAT-SHEET

| Term | One-line meaning |
|---|---|
| **Six failure stages** | DNS → TCP handshake → TLS handshake → request write → server processing → response read |
| **Before vs after** | failed before the server saw the request = no side effects; after = server may have acted |
| **Connect timeout** | never got a socket — server never saw the request |
| **Read timeout** | had a socket, went quiet — server may have half-done the work |
| **Retry** | the *exact same* request again, nothing changed, betting the failure was transient |
| **Handling** | a different, valid action (new token, other endpoint, other stream) — not a retry |
| **Transient** | will likely clear by itself (blip, brief overload) → retry can help |
| **Structural** | won't change by waiting (404, geo-block, preflight, bad auth) → don't retry |
| **4xx** | "you failed" — refused every time if unchanged; usually don't retry |
| **5xx** | "I failed" — says nothing about whether it changed anything; usually retry |
| **Idempotent** | same effect whether run once or many times (GET, PUT, DELETE; not POST) |
| **Idempotency key** | client-generated ID per logical operation; server dedupes repeats and replays the stored result |
| **Naive retry loop** | no cap, no delay, no coordination → infinite loop, hammering, thundering herd |
| **Exponential backoff** | delay = `min(cap, base · 2^attempt)` → load on the upstream decays |
| **Full jitter** | sleep `uniform(0, delay)` → spreads clients out in time (AWS 2015) |
| **Max attempts** | caps total damage; dead dependency fails in seconds |
| **429 Too Many Requests** | you're too fast; honour `Retry-After` |
| **503 Service Unavailable** | overloaded/maintenance; trust `Retry-After` over your curve |
| **408 Request Timeout** | server gave up waiting for you to send; retry is fine |
| **Retry-After** | server-provided wait (seconds or HTTP date) — use the number |
| **Circuit breaker** | per-dependency failure cap that remembers its verdict (Nygard, *Release It!*, 2007) |
| **CLOSED / OPEN / HALF-OPEN** | normal & counting / fail fast locally / one probe after cooldown |
| **Thundering herd** | many clients retrying in sync — especially right when a service recovers |
| **Slow-start** | recovering server ramps its accepted traffic instead of taking full load instantly |
| **Live-edge 404** | newest live segment not ready yet → 2–3 short fixed-delay retries, only there |
| **Custom codes (419/474/475)** | non-RFC codes; classify and document them like real ones |
| **Fallback** | DRM L1→L3, alternate codec, MPD→HLS, ad→content, iOS ResetMedia |
| **Fatal coded error** | repeated segment/manifest failure ends in a specific error (e.g. 8018, 8022), not an infinite loop |
| **Application-layer signals** | response body says "degraded, lower bitrate" / "maintenance, try in N min" |
| **stale-if-error (RFC 5861)** | cache serves its last good copy when the origin errors — not a retry |
| **Failover** | stop asking the dead primary; ask a backup (region/provider/CDN) |
| **Feature flag** | runtime switch to turn off non-essential features and shed load, no deploy |
| **Graceful degradation** | answer smaller, not "no" (lower bitrate, cached recs, stale price) |
| **Fastly, 8 Jun 2021** | 85% of network erroring, ~1 min to detect, 49 min to 95% recovered |
| **SO_LINGER {1,0}** | `close()` sends RST instead of FIN (abortive close) |
| **SIGPIPE** | signal on writing to a dead socket; kills the process by default |
| **5-step synthesis** | classify → structural? stop → transient? retry well → breaker → fail over/degrade/serve stale |

---

## 17 · SELF-CHECK QUESTIONS

**1. Your logs show two "timeouts": one is a DNS timeout and one is a read timeout. Why are these completely different problems?**
<details><summary>Answer</summary>

The DNS timeout never reached a server, so no work happened anywhere and the server has nothing to undo. The read timeout reached a server that received the request and may have partly or fully done the work; retrying could repeat it. Only the read timeout raises the idempotency question.
</details>

**2. A request fails with 401 because the access token expired. The client refreshes the token and sends the request again. Is that a retry? Why does the distinction matter?**
<details><summary>Answer</summary>

No. The second request has a different `Authorization` header, so it's a different request responding to information the server gave you: it's handling, not retrying (slide 8's test: "if you changed nothing and only waited, would the attempts be identical?"). It matters because backoff and retry budgets shouldn't apply. Waiting doesn't fix an expired token, and counting the refresh as a retry wastes the budget and muddies the metrics.
</details>

**3. Why is it pointless to retry a 404, a geo-block or a CORS preflight failure? What's the one exception the lecture gives for 404?**
<details><summary>Answer</summary>

They're structural: the resource doesn't exist, policy forbids you, or the client already decided not to send the request. Nothing will be different 200 ms later. The exception is a 404 on a live stream's newest segment, which may still be encoding or propagating through the CDN. Use a short, bounded retry (2–3 attempts, small fixed delay) only for live-edge segments. A 404 on a VOD asset still means give up.
</details>

**4. A POST /payments call hits a read timeout. Why can't you just retry it, and what's the fix?**
<details><summary>Answer</summary>

POST isn't idempotent, and you don't know whether the server charged the card before the connection died, so a blind retry might charge twice. The fix isn't "never retry". Send an idempotency key that the client generates once per logical payment and reuses on every attempt. The server stores the key with the result (atomically, alongside the work) and replays the stored result for repeats instead of running the work again.
</details>

**5. List the three problems with the `while True: ... continue` retry loop.**
<details><summary>Answer</summary>

(1) No max count: a permanently-down dependency becomes an infinite loop instead of a fast failure. (2) No delay: every client hammers the upstream at wire speed. (3) No coordination across clients: when the upstream recovers, every loop fires on the same tick (the thundering herd) and can re-crash it.
</details>

**6. Why does the slide insist on "full jitter, `uniform(0, delay)`, not `delay ± a bit`"?**
<details><summary>Answer</summary>

Clients that started failing at the same moment get the same exponential delays, so they retry together — just a delayed herd. A small ± still bunches them around the same instant. Spreading each sleep across the whole [0, delay] range separates them and flattens the peak load on the upstream. Total retries stay the same; the peak drops a lot.
</details>

**7. A 429 arrives with `Retry-After: 30`. Your backoff curve says wait 0.4 s. What do you do, and why?**
<details><summary>Answer</summary>

Wait (at least) 30 s. A server that hands you a number knows its own load better than your exponential curve does. Clients that ignore `Retry-After` and use their own timing can together still overload a server that explicitly asked them to slow down. (Adding a little jitter on top is fine. Going sooner isn't.)
</details>

**8. Explain the three circuit-breaker states. What does a breaker protect that backoff doesn't?**
<details><summary>Answer</summary>

CLOSED: requests flow and failures are counted. OPEN: the threshold was crossed, so calls fail instantly and locally without being attempted. HALF-OPEN: after a cooldown, one probe goes through; success closes the breaker, failure re-opens it. Backoff protects the upstream from one client retrying too fast. A breaker protects the upstream from a whole fleet calling a service everyone knows is down, and protects the caller from paying the full timeout on every doomed call (which would otherwise exhaust its threads and connections and cascade the failure). It's the max-retry idea moved up a level: failures are capped per dependency instead of per request, and the verdict is remembered.
</details>

**9. Give three video-player failures where the right answer is "fall back", not "retry", and say why for each.**
<details><summary>Answer</summary>

Any three of these:
- DRM key denied (L1 vs L3): the same license request gets the same denial, so drop to a lower security level or a different key system.
- Codec unsupported: the device can't decode it, which isn't transient, so switch to an alternate codec rendition.
- DASH manifest failure: fall back to an HLS rendition of the same content.
- Ad failure: fall back to the content, because the user's patience for a broken ad is near zero.
- iOS ResetMedia: the fault is in the device's media state, not a request, so rebuild the AVPlayer pipeline.
- CDN edge not responding: switch CDNs.
</details>

**10. Why is the moment a dependency recovers "the most dangerous moment of the whole incident"? Name three mitigations.**
<details><summary>Answer</summary>

Every client that was retrying, backing off or waiting on a breaker cooldown tries again at about the same time, because they all started failing together and their schedules line up. The service is still cold (empty caches, no connections), running at around 1% capacity, yet gets close to 100% of normal traffic in its first second, so it can fall over again. Mitigations: jitter, staggered half-open probes instead of all at once, and slow-start on the recovering side (accept traffic at a ramped rate).
</details>

**11. What is `stale-if-error`, and why does the lecture say it's "not a retry at all"? What's the general lesson?**
<details><summary>Answer</summary>

`stale-if-error` (RFC 5861) is a `Cache-Control` extension that lets a cache keep serving its last good copy when it can't reach the origin to revalidate. Nothing is attempted again. It's a decision to serve something known to be stale instead of nothing. The general lesson: a system that can answer with slightly wrong data survives outages that a "perfectly fresh or nothing" system doesn't, which is the idea behind graceful degradation (lower bitrate, cached recommendations, a stale price).
</details>

**12. In the calculator assignment, why is keeping the connection open "the part that is actually hard"?**
<details><summary>Answer</summary>

In HTTP/1.0 the server closed the connection after each response, so "the message ends at EOF" came for free. On a persistent HTTP/1.1 connection, several requests share one byte stream. The server has to find where each request ends: the header block ends at `\r\n\r\n`, then it reads exactly `Content-Length` body bytes and no more, because byte n+1 is the start of the next request. Its own responses need `Content-Length` (or chunked encoding) too, so the client knows where they end. That also needs a per-connection buffer that keeps leftover bytes, which is what makes pipelining work.
</details>
