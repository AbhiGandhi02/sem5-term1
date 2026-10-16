# 📓 Evolution of HTTP, 1996 → now
### *CN at Scaler — Lesson 5 (Week 5)* · "Same words. Three completely different ways of writing them down."

> `HTTP/1.0 → early HTTP/1.1 → HTTP/1.1 today → HTTP/2 → QUIC + HTTP/3`
> **RFC 1945 · RFC 2068 · RFC 2616 · RFC 7230 · RFC 9110 · RFC 9113 · RFC 9000 · RFC 9114**

```
  $ printf 'GET / HTTP/1.0\r\n\r\n' | nc localhost 8010
  # ... and in 2 hours, the same request over QUIC
```

**Legend for these notes:** 🧠 = my own addition, *not on the slides*. It's background I needed to follow the lecture. Everything else comes from the slides.

---

## ⭐ WHAT THIS LECTURE IS ABOUT (AND WHY IT MATTERS)

The *meaning* of HTTP has barely changed since 1996. `GET /index.html` still means "give me that resource", `200` still means OK and `404` still means not found. What changed is the **wire format**, meaning how those words are turned into bytes on a socket. That has been redesigned three times: text in HTTP/1.x, binary frames in HTTP/2, and QUIC streams over UDP in HTTP/3.

The lecture's thesis is that **each fix created the next problem.**

```
 HTTP/1.0  one request per connection        → slow (a handshake for every object)
    │ fix: keep the connection open (persistence)
    ▼
 HTTP/1.1  persistent + pipelining           → head-of-line blocking (responses queue up in order)
    │ fix: give every message an ID (streams)
    ▼
 HTTP/2    multiplexed streams over ONE TCP  → TCP head-of-line blocking (one lost packet stalls all)
    │ fix: move the streams INTO the transport
    ▼
 HTTP/3    streams over QUIC (UDP)           → HPACK breaks (needs order) → QPACK
                                             → 3–5% of networks block UDP → fallback to TCP
```

> *"The protocol did not drift — it was pushed."*

**Why it matters to you:** about 15% of web requests are still HTTP/1.1 in 2026. You'll write HTTP/1.1 parsers, debug proxies that disagree about framing, and tune HTTP/2 and HTTP/3 deployments. Most real engineering happens at the **boundary between versions**, not inside one of them.

### The agenda (from the slides)

| Hour | Version | The RFC | Ends on |
|---|---|---|---|
| 1 | HTTP/1.0 | RFC 1945 · May 1996 | One response per connection |
| 2 | Early HTTP/1.1 | RFC 2068 · Jan 1997 | "At most 2 connections" |
| 3 | HTTP/1.1 now | RFC 2616 → 7230 → 9110 | Head-of-line blocking |
| 4 | HTTP/2 | RFC 7540 → 9113 | TCP head-of-line blocking |
| 5 | QUIC + HTTP/3 | RFC 9000 · RFC 9114 | The 3–5% who block UDP |

### Where this sits in the course

| Week | Topic |
|---|---|
| 1 | Network programming 101 |
| 2 | Computer Networking Eagle Eye View |
| 3 | Server Side Considerations for scale |
| 4 | nginx deep dive |
| **5** | **Evolution of HTTP** ← you are here |
| 6 | (Ab)using CDNs |
| 7 | Building for failures |
| 8 | Streaming video at large scale |

---

## 0 · PREREQUISITES FROM SCRATCH 🧠

The slides assume all of the following. Skip any part you already know well.

### 0.1 What is an RFC, and who is the IETF?

An **RFC (Request for Comments)** is a numbered document published by the **IETF (Internet Engineering Task Force)**, the body that standardises internet protocols. Once published, an RFC never changes. To fix or replace one, a new RFC is published that **obsoletes** it. That's why HTTP/1.1 has been "RFC 2068", then "RFC 2616", then "RFC 7230–7235", then "RFC 9110–9112", all describing the same protocol more and more precisely.

Spec language uses capitalised keywords from RFC 2119:
- **MUST / MUST NOT**: required. Breaking this rule makes you non-compliant.
- **SHOULD / SHOULD NOT**: strongly recommended. You may deviate if you understand the consequences.
- **MAY**: optional.

The lecture quotes "SHOULD NOT maintain more than 2 connections". Browsers violated that SHOULD for years.

### 0.2 Bytes, sockets, and "a byte stream has no boundaries"

From Lesson 1: a **TCP socket** is a file descriptor that gives you an **ordered, reliable stream of bytes**. TCP has **no notion of messages**. If the sender calls `write("hello")` and then `write("world")`, the receiver might get `"helloworld"` in one `read()`, or `"hel"` followed by `"loworld"`.

So **every protocol on top of TCP must answer one question: where does one message end and the next begin?** That is called **framing**. Session 2's closing idea, repeated in this lecture:

```
  Framing is LENGTH or DELIMITER. There is no third option.

  LENGTH:     [ 5 ][ h e l l o ]            "the next 5 bytes are the message"
  DELIMITER:  h e l l o \n                  "the message ends at the newline"
```

HTTP uses **both**. Headers are delimiter-framed: each line ends in `\r\n`, and the header block ends at an empty line `\r\n\r\n`. The body is length-framed by `Content-Length`, or chunked (a length per chunk, explained later), or in HTTP/1.0 simply "until the server closes".

### 0.3 RTT (round-trip time) and why handshakes are expensive

**RTT** is the time for a packet to go to the other side and for the reply to come back. If you're in Bangalore and the server is in Virginia, RTT might be ~200 ms. **Speed-of-light latency can't be fixed with more bandwidth.** The only fix is to need **fewer round trips**.

A new TCP connection costs **1 RTT** for the 3-way handshake (SYN → SYN/ACK → ACK) before any data flows. TLS (Transport Layer Security, the "S" in HTTPS) adds **1 more RTT** with TLS 1.3, or **2 more** with TLS 1.2.

```
 CLIENT                                SERVER
   │── SYN ─────────────────────────────▶│ ┐
   │◀──────────────────────── SYN/ACK ───│ │ 1 RTT  (TCP)
   │── ACK + ClientHello ───────────────▶│ ┘ ┐
   │◀──────────────── ServerHello ... ───│   │ 1 RTT (TLS 1.3)
   │── Finished + GET / ────────────────▶│   ┘ ┐
   │◀──────────────────────────── 200 ───│     │ 1 RTT (the actual request)
```

### 0.4 TCP congestion window (cwnd) and slow start

TCP doesn't blast data at full speed on a new connection. It starts with a small **congestion window (cwnd)**, typically ~10 segments (about 14 KB), and roughly doubles it every RTT while nothing is lost. This is called **slow start**. So **a brand-new connection is a slow connection**. Opening 24 fresh connections means 24 slow starts, each "starting from nothing". This is why connection reuse matters so much. You can see cwnd on Linux with `ss -ti` (Homework 1 asks you to).

### 0.5 What is a cache, and what is a validator?

A **cache** keeps a copy of a response so the next request can be answered without fetching the full thing again. Your browser has one, CDNs (Content Delivery Networks) are giant shared ones, and proxies have them.

The hard part of caching is knowing whether the copy is still fresh. Two tools exist:
- **Freshness lifetime**: "this is valid until time T" (`Expires`, `Cache-Control: max-age`). No network needed until T.
- **Validation**: "I have version X, is it still current?" The server either says **304 Not Modified** (keep yours, no body) or sends the new version. The "version X" token is called a **validator**. HTTP/1.0's validator was a timestamp (`Last-Modified`). HTTP/1.1 added a fingerprint (`ETag`).

### 0.6 Proxies, middleboxes, and "a chain of five"

Between your browser and the app there are often several boxes that parse HTTP:

```
 browser → corporate proxy → CDN edge → load balancer → nginx → app server
```

Each one is a separate HTTP implementation, written by different people. If two of them disagree about where a message ends, you get security bugs (request smuggling, §3.3). **Middleboxes** is the general term for NATs, firewalls, and proxies that sit in the path and inspect or modify traffic. They're why QUIC runs on UDP (§6.1).

### 0.7 Head-of-line (HOL) blocking, in one picture

**Head-of-line blocking** happens when an item at the front of a queue is slow, so everything behind it waits even though those items are ready.

```
  Supermarket, ONE checkout lane:
  [ customer with 200 items ] [ 1 item ] [ 1 item ] [ 1 item ]
        ▲ everyone waits for this one
```

It needs a **queue**. Keep that in mind, because the lecture uses it to correct a common myth about HTTP/1.0 (§1.6).

### 0.8 Binary vs text protocols

A **text protocol** (HTTP/1.x, SMTP) is readable by humans. You can type it into `telnet` or `nc`. A **binary protocol** (HTTP/2, HTTP/3, TCP itself) uses fixed-size numeric fields, so you need a hexdump or a decoder to read it. Text is easy to implement and debug. Binary is compact and unambiguous. The lecture's key insight is that **HTTP/2's real improvement was adding lengths and IDs, not being binary.** Binary was a consequence of that (§4.6).

### 0.9 Hex in 30 seconds

`0x37` means 3×16 + 7 = **55**. `0x2a` = 42. `0x4d` = 77. `0x107` = 1×256 + 0×16 + 7 = **263**. One byte = two hex digits = 8 bits. You'll need this for chunked encoding and for HTTP/2 frame headers.

---

## 1 · HTTP/1.0 · RFC 1945 · MAY 1996

### 1.1 The whole protocol fits on one slide

**Plain English:** HTTP/1.0 is tiny. A client sends a text request, the server sends a text response, and the server hangs up.

| Methods | Headers (16) | Status codes |
|---|---|---|
| `GET`, `HEAD`, `POST` (in the spec) | `Allow`, `Authorization`, `Content-Encoding`, `Content-Length`, `Content-Type`, `Date`, `Expires`, `From`, `If-Modified-Since`, `Last-Modified`, `Location`, `Pragma`, `Referer`, `Server`, `User-Agent`, `WWW-Authenticate` | `200 OK`, `201 Created`, `202 Accepted`, `204 No Content` · `300 Multiple Choices`, `301 Moved Permanently`, `302 Moved Temporarily`, `304 Not Modified` · `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found` · `500 Internal Server Error`, `501 Not Implemented`, `502 Bad Gateway`, `503 Service Unavailable` |
| `PUT`, `DELETE`, `LINK`, `UNLINK`: Appendix D, "additional methods" that exist but are not in the spec | | **Today: 62 assigned** (IANA HTTP Status Code Registry, updated 2025-09-15) |

🧠 The methods:
- **GET**: fetch a resource.
- **HEAD**: same as GET, but the server sends **only the headers** and no body. Useful to check size, type, or `Last-Modified` cheaply.
- **POST**: send data to the server (forms, uploads).
- **LINK/UNLINK**: never caught on and are dead today.

🧠 Status-code families: **1xx** informational, **2xx** success, **3xx** redirect or "use your cache", **4xx** client's fault, **5xx** server's fault. HTTP/1.0 had no 1xx codes at all.

**The anatomy of an HTTP/1.0 exchange 🧠:**

```
REQUEST                                   RESPONSE
─────────────────────────────             ─────────────────────────────
GET / HTTP/1.0\r\n      ← request line    HTTP/1.0 200 OK\r\n        ← status line
User-Agent: x\r\n       ← headers         Content-Type: text/html\r\n
\r\n                    ← blank line =    Content-Length: 263\r\n
                          end of headers  \r\n                       ← blank line
                                          <!doctype html>...         ← body
                                          [server closes = end of body]
```

The minimal request `GET / HTTP/1.0\r\n\r\n` is valid in 1.0. **No headers are required.** That's what the title slide's `printf | nc` sends.

### 1.2 Hands-on: speak it by hand, once

The lecture uses its own tool, `tools/talk.py`, instead of curl:

```
$ python3 tools/talk.py 8010 --req "GET / HTTP/1.0" ""
--- sent 18 bytes

HTTP/1.0 200 OK
Date: Fri, 04 Sep 2026 07:11:27 GMT
Server: server10/1.0 (RFC 1945)
Content-Type: text/html
Content-Length: 263
Last-Modified: Fri, 04 Sep 2026 07:11:14 GMT
Expires: Fri, 04 Sep 2026 07:12:27 GMT
                                     <- the blank line is the delimiter
<!doctype html>
<title>site-a</title>
...

--- server closed the connection (FIN) at 2.9 ms
```

Points to notice:
- **18 bytes sent**: `GET / HTTP/1.0` is 14 characters, plus `\r\n` (2), plus the final `\r\n` (2), which makes 18.
- The **blank line** separates headers from body. It's the delimiter.
- `Expires` is exactly 1 minute after `Date`, so this server lets caches keep the page for 60 s.
- The connection **closed (FIN)** right after the response, at 2.9 ms.

> *"Not curl. curl adds four headers you did not write and hides two the server sent."*

🧠 For example, curl adds `Host`, `User-Agent` and `Accept` on its own. When you study a protocol you want to see **exactly** the bytes you sent and the bytes you got back. `nc` (netcat) or a raw-socket script gives you that.

Files from the lecture repo: `01-http10/server10.py`, `tools/talk.py`.

### 1.3 The limitation: one response, then the socket dies

**Plain English:** HTTP/1.0 is like a phone call where you can ask exactly one question, and then the other person hangs up. Every image on a web page needs a new call.

RFC 1945 §1.3 says:
> *"current practice requires that the connection be established by the client prior to each request and closed by the server after sending the response."*

**Try to reuse the socket:**

```python
s.sendall(b"GET /style.css HTTP/1.0\r\n\r\n")
r1 = recv_until_eof(s)
# request 1: 274 bytes, server sent FIN

# now try a second request on the SAME socket
s.sendall(b"GET /app.js HTTP/1.0\r\n\r\n")
r2 = s.recv(4096)
# request 2: b''
```

The second read returns `b''`. That's **not an error, it's an empty read**. In Lesson 1 we learned that a `read()`/`recv()` returning 0 bytes means EOF: the peer has closed. The connection was closed the moment the first response finished.

**The cost, computed on the slide (a 24-object page, 200 ms RTT):**

```
  24 TCP handshakes            24 × 200 ms = 4.8 s
  24 request/response          24 × 200 ms = 4.8 s
                                             ──────
                                              9.6 s
  ...and 24 congestion windows that each start from nothing.
```

**Why did it work this way?** It's simple. With "the body ends when the server closes", the server doesn't even need to know the body length in advance. EOF is the frame delimiter, for free. That simplicity is exactly what HTTP/1.1 has to give up (see the Assignment).

### 1.4 The limitation: no `Host` header. One IP, one website.

**Plain English:** an HTTP/1.0 request says *which file* (`/index.html`) but not *which website*. If two websites share one server, it can't tell which one you meant.

```
$ talk.py 8010 --req "GET / HTTP/1.0" "Host: site-b.example" ""

HTTP/1.0 200 OK
<h1>site-a</h1>                 <- not site-b. it never had a choice.

# server log:
   (ignoring Host: site-b.example - HTTP/1.0 has no vhosts)
```

The only "address" the server saw was the TCP 4-tuple (src IP, src port, **dst IP, dst port**). So **one IP address = one website** ("1:1 sites per IP address, until January 1997").

The slide's numbers:
- **1:1** sites per IP address, until January 1997.
- **~1.1 B** websites today. Without Host, that would need about **a third of all of IPv4**. 🧠 IPv4 has ~4.3 billion addresses in total.
- **SNI**: "the same problem one layer down, once TLS hid the Host" (§2.2).

> *"The fix — a single header field — arrives nine months later and is arguably the highest-leverage line in the whole standard."*

🧠 **vhosts (virtual hosting)** means one server, one IP, many websites, chosen by the `Host` header. Every nginx `server_name` block from Week 4 depends on it.

### 1.5 What it did have: caching, with a clock as the fingerprint

**Plain English:** the client says "I have the copy from 07:11:14, has it changed since then?" If it hasn't, the server says "no, keep yours" and sends no body.

```
$ talk.py 8010 --req "HEAD /style.css HTTP/1.0" ""
Last-Modified: Fri, 04 Sep 2026 07:11:14 GMT

$ talk.py 8010 --req "GET /style.css HTTP/1.0" \
        "If-Modified-Since: Fri, 04 Sep 2026 07:11:14 GMT" ""

HTTP/1.0 304 Not Modified
                                <- no body. 202,000 bytes -> 0.
```

```
 CLIENT (has cached copy, dated 07:11:14)        SERVER
   │── GET /style.css                              │
   │   If-Modified-Since: 07:11:14 ───────────────▶│  file mtime == 07:11:14 ?
   │◀────────────────────── 304 Not Modified ──────│  yes → no body
   │   (use cached 202,000 bytes)                  │
```

Spec references: RFC 1945 §10.7 `Expires`, §10.9 `If-Modified-Since`, §10.10 `Last-Modified`.

**Three things wrong with a timestamp (pitfalls):**

| Problem | Why |
|---|---|
| **1 s resolution** | No sub-second field. Two writes in one second look like *one* write to every cache alive, so the second change is invisible. |
| **Clock drift** | Your clock, my clock, the proxy's clock all differ. The spec has to legislate for disagreement. |
| **mtime ≠ identity** | A rebuild (e.g. `git checkout`, a deploy) changes every mtime and none of the bytes. Every cache throws away perfectly good copies. |

The fix in HTTP/1.1 is `ETag`, a fingerprint of the content (§2.3).

### 1.6 Closing: what HTTP/1.0 could not do

| The limitation | Concretely | Fixed in |
|---|---|---|
| Connection is not persistent | One request per TCP handshake, forever | RFC 2068 §8.1 |
| Sixteen headers, three methods | No `OPTIONS`: you cannot ask what a server supports | RFC 2068 §9.2 |
| Caching exists but is thin | `Expires` + `If-Modified-Since`. A clock, not a fingerprint | RFC 2068 §14.19 |
| No `Host` | One IP address per website | RFC 2068 §14.23 |

> **⚠️ Interview gotcha (the lecture's correction):** HTTP/1.0 is often listed as having "head-of-line blocking". **It doesn't, quite.** HOL blocking needs a *queue*, and HTTP/1.0 has no pipelining to queue anything on. What it has is a **serialisation cost**: one round trip per object, no exceptions.
>
> **Real** HOL blocking arrives in HTTP/1.1, as a side effect of the fix for this. *"Watch for that pattern all day — every solution today is the cause of the next slide."*

---

## 2 · EARLY HTTP/1.1 · RFC 2068 · JANUARY 1997

### 2.1 Persistence became the default. Now you opt out.

**Plain English:** HTTP/1.1 flips the default. The phone line stays open after each answer unless someone says "goodbye" (`Connection: close`).

| HTTP/1.0 | HTTP/1.1 (RFC 2616 §8.1.2) |
|---|---|
| **Ask, and maybe receive:** `Connection: keep-alive` | **Opt out:** `Connection: close` |
| A non-standard extension. Some servers honoured it, some ignored it, some crashed. Documented in RFC 2068 §19.7.1 as a *compatibility note*, not a feature. | *"persistent connections are the default behavior of any HTTP connection."* |
| **The default was: hang up.** | **The default was: stay.** Every implementation that did nothing at all got the fast path. |

```
$ 02-http11/demos.sh             # demo 1
  /                   HTTP/1.1 200 OK       socket still open: True
  /style.css          HTTP/1.1 200 OK       socket still open: True
  /app.js             HTTP/1.1 200 OK       socket still open: True
  three responses, one TCP handshake.
```

```
 HTTP/1.0 (3 objects)                   HTTP/1.1 keep-alive (3 objects)
 ─────────────────────                  ─────────────────────────────
 SYN/SYNACK/ACK  GET /     200  FIN     SYN/SYNACK/ACK
 SYN/SYNACK/ACK  GET /css  200  FIN       GET /      → 200
 SYN/SYNACK/ACK  GET /js   200  FIN       GET /css   → 200
                                          GET /js    → 200
 = 3 handshakes, 3 slow starts          = 1 handshake, cwnd keeps growing
```

**Why it exists:** it removes the per-object handshake and lets the TCP congestion window warm up and *stay* warm.

**The hidden cost (this is the assignment):** once the server stops closing, **EOF no longer marks the end of the body.** Every response now *must* carry its length (`Content-Length`) or use chunked encoding. A server that just writes the body and waits will hang the client forever.

🧠 **Design point:** "secure/fast by default" beats "opt in". Making the good behaviour the default means lazy implementations get it for free.

### 2.2 `Host`: one header, one IP address, a thousand sites (RFC 2068 §14.23)

```
$ talk.py 8011 --req "GET / HTTP/1.1" ""                # no Host
HTTP/1.1 400 Bad Request
HTTP/1.1 requires a Host header (RFC 2616 14.23).

$ ... "Host: site-a.local"      ->   <h1>site-a</h1>
$ ... "Host: site-b.local"      ->   <h1>site-b</h1>
                                     same IP. same port. same socket.
```

In HTTP/1.1, a request without `Host` **must** get a `400`. (RFC 2616 made it a MUST, with a MUST-400 for violations. See §3.1.)

**The same move, four layers, thirty years:**

| Layer | The problem | The fix |
|---|---|---|
| HTTP/1.0 | The four-tuple was the address; one IP = one site | `Host:`, which puts the name in the message |
| TLS | Host is encrypted; the cert must be chosen first | **SNI** in the ClientHello, in the clear |
| HTTP/2 | Headers are compressed and unordered | `:authority` as a pseudo-header |
| Everywhere else | Many logical destinations, one connection | Topic. Routing key. Ingress host rule. |

🧠 **SNI (Server Name Indication)**: with HTTPS, the `Host` header is inside the encrypted part, but the server has to choose *which certificate* to present *before* encryption starts. So the TLS ClientHello carries the hostname in plaintext. It's the same problem one layer down, solved the same way. 🧠 The "everywhere else" row refers to Kafka **topics**, RabbitMQ **routing keys**, and Kubernetes **Ingress host rules**, which all put the destination name inside the message so that one connection or endpoint can serve many logical destinations.

### 2.3 `ETag`: a fingerprint, not a clock (RFC 2068 §14.19)

**Plain English:** instead of "the file was changed at 07:11:14", the server says "the file's content hash is `55274ac22c651b90`". If the content is the same, the hash is the same, whatever the clock says.

```
$ talk.py 8011 --req "HEAD /big.txt HTTP/1.1" "Host: site-a.local" ""
ETag: "55274ac22c651b90"                  <- sha256 of the bytes, first 16 hex

$ talk.py 8011 --req "GET /big.txt HTTP/1.1" \
         "Host: site-a.local" "If-None-Match: \"55274ac22c651b90\"" ""

HTTP/1.1 304 Not Modified
ETag: "55274ac22c651b90"
--- received 256 bytes
```

(256 bytes of headers only. The 200 would have carried the whole 202,000-byte file.)

| Concept | Meaning |
|---|---|
| **202,000 → 0** | bytes of body. The 200 carries the whole file; the 304 carries nothing. |
| **`If-None-Match`** | "Send it only if your ETag is *not* this one." Used for cache revalidation (RFC 9110 §13.1.2). |
| **`W/"…"`** | A **weak validator**: *equivalent*, not identical (e.g. the same page with a different timestamp in the footer). **Range requests need a strong one**, because you can't stitch byte ranges together from two "equivalent but different" files. (RFC 9110 §8.8.1, strong vs weak) |
| **`If-Match`** | The same idea pointed at **writes**: "update this only if it is still version X." That's **optimistic concurrency**, and the fix for **lost updates** (RFC 9110 §13.1.1). |

🧠 **Lost-update example:** Alice and Bob both GET a document (ETag "v1"). Alice PUTs her edit with `If-Match: "v1"`, which succeeds, and the ETag becomes "v2". Bob PUTs with `If-Match: "v1"` and gets **412 Precondition Failed** instead of silently overwriting Alice. He re-fetches and merges.

**Pitfall:** an ETag built from mtime (Homework 2) is just a timestamp in disguise, and it inherits every problem in §1.5.

### 2.4 `Accept-*` and `Vary`: the header that keeps caches honest (§14.3, §14.44)

**Plain English:** the client says "I can understand gzip". The server compresses the body and tells caches "this URL has different answers depending on `Accept-Encoding`, so don't give a gzipped copy to a client that can't read it."

```
$ 02-http11/demos.sh                # demo 7
   Accept-Encoding: identity  -> Content-Length: 202000   Vary: Accept-Encoding
   Accept-Encoding: gzip      -> Content-Length: 5763     Content-Encoding: gzip
   Accept-Encoding: deflate   -> Content-Length: 5751     Content-Encoding: deflate
```

5763 / 202000 ≈ **2.8% of its original size**.

> *"Which quietly undercuts 'HTTP/1.1 is slow because it is text'. Text compresses. The body was never the problem — the headers were, because Content-Encoding compresses the body only."*

**`Vary` is a cache-key instruction.** It says: *this URL has more than one answer, and here is what the answer depends on.*

| `Vary` value | Effect |
|---|---|
| `Vary: Accept-Encoding` | correct: a handful of variants (gzip / br / identity) |
| `Vary: User-Agent` | millions of keys (every browser version string is unique), hit rate → 0 |
| `Vary: *` | caching off entirely |

```
 cache key WITHOUT Vary:   GET  example.com/style.css
 cache key WITH Vary:      GET  example.com/style.css  + [Accept-Encoding: gzip]
```

**Pitfall:** forgetting `Vary: Accept-Encoding` means a CDN may serve gzipped bytes to a client that asked for identity, and the user sees garbage.

### 2.5 `Range`: the header that runs your data lake (§14.35)

**Plain English:** "give me bytes 100 to 199 of this file, not the whole thing."

```
Range: bytes=100-199     ->   HTTP/1.1 206 Partial Content
                              Content-Range: bytes 100-199/202000
                              Content-Length: 100

Range: bytes=-50         ->   Content-Range: bytes 201950-201999/202000
                              the last 50 bytes, size not known in advance

Range: bytes=999999-     ->   HTTP/1.1 416 Range Not Satisfiable
                              Content-Range: bytes */202000
```

Reading the syntax:
- `100-199` is **inclusive** at both ends, so it's 100 bytes.
- `-50` is a **suffix range**: the last 50 bytes. You don't need to know the file size first.
- `999999-` means from byte 999999 to the end. The file is only 202000 bytes, so the server returns 416, and `*/202000` tells you the real size.

**What one header bought:**

| Use | How |
|---|---|
| **Video scrubbing** | The player asks for the bytes around the timestamp you clicked, not the whole film. |
| **Resumable downloads** | Died at 60%? `Range: bytes=600000000-` and carry on where you stopped. |
| **Parquet / ZIP footers** | The directory is at the **end**. Read the last few KB, then fetch only the columns you need. 🧠 This is how tools like DuckDB and Spark read Parquet straight from S3 over HTTP. |

> *416 was not in RFC 2068. RFC 2616 added it to separate "your syntax is wrong" from "that file is smaller than you think".*

### 2.6 Chunked transfer encoding: send a body without knowing how long it is (§3.6.1)

**Plain English:** with persistence, every body needs a length. But what if you're generating the response on the fly (a database query streaming rows, a live log) and don't know the length yet? You send it in **chunks**, each with its own length, and finish with an empty chunk.

```
$ talk.py 8011 --req "GET /style.css?chunked=1 HTTP/1.1" ...

HTTP/1.1 200 OK
Content-Type: text/css
Transfer-Encoding: chunked
                                    <- and NO Content-Length

37
body{font-family:monospace;background:#111;color:#eee}
0
                                    <- 0x37 = 55 bytes, then the empty chunk
```

**The exact byte layout 🧠:**

```
 Transfer-Encoding: chunked\r\n
 \r\n
 37\r\n                           ← chunk size in HEX (0x37 = 55)
 <55 bytes of data>\r\n           ← data, then CRLF
 0\r\n                            ← size 0 = "last chunk"
 \r\n                             ← end of message (optional trailers would go before this)
```

> **Session 2, closing idea 2:** framing is length or delimiter — there is no third option.
> **Chunked encoding is both, nested:** each chunk is **length-prefixed**, and the sequence is **delimited** by an empty one. That's FastCGI's record structure from Session 3, exactly. Same shape, different decade. It's also what **streaming JSON, server-sent events and gRPC trailers** all stand on.

🧠 **Trailers** are headers sent *after* the last chunk, e.g. a checksum computed while streaming. gRPC sends its status code as a trailer.

**Pitfalls:**
- The size is **hex**, not decimal. `37` means 55, not 37.
- `Transfer-Encoding: chunked` and `Content-Length` **must not both** be trusted. That's the request-smuggling attack in §3.3.
- 🧠 `Transfer-Encoding` is hop-by-hop (a property of this connection). `Content-Encoding` (gzip) is a property of the content. They're different layers.

### 2.7 And the rest of it, in one breath

| Arrival | What it is for | Where you meet it now |
|---|---|---|
| `OPTIONS` | Ask what a server supports | Every **CORS preflight** your browser sends |
| `PUT` · `DELETE` | Write and remove, not just read | The R and D of every REST API since (🧠 in the Read/Replace/Delete sense) |
| `TRACE` | Echo my request back, proxies and all | Disabled everywhere: **Cross-Site Tracing**, 2003 |
| `Via` | Each proxy appends itself | Working out which box in the chain is lying |
| `Expect: 100-continue` | Ask before uploading four gigabytes | curl's mysterious 1-second pause on big PUTs |
| `Cache-Control` | A vocabulary instead of one timestamp | `max-age`, `s-maxage`, `no-cache` ≠ `no-store` |
| ~40 status codes | 303, 405, 409, 410, 412, 413, 415, 504… | 504 is here already, not a 1999 addition |

🧠 Short explanations:
- **CORS preflight**: before a cross-origin request with custom headers, the browser sends `OPTIONS` to ask "may I?"
- **Cross-Site Tracing (XST)**: JavaScript could send `TRACE` and read the echoed request, including `HttpOnly` cookies the script wasn't supposed to see. So TRACE got switched off.
- **`Expect: 100-continue`**: the client sends headers only and waits for `100 Continue` before sending a huge body. If the server would reject it anyway (413, 401), the client never wastes the upload. curl waits up to ~1 s for that 100 before sending anyway, which is the "mysterious pause".
- **`max-age`** is freshness for every cache. **`s-maxage`** applies to shared caches (CDNs) only.
- New codes: **303** See Other, **405** Method Not Allowed (you need it in the assignment), **409** Conflict, **410** Gone, **412** Precondition Failed, **413** Payload Too Large, **415** Unsupported Media Type, **504** Gateway Timeout.

> **⚠️ Classic gotcha:** `no-cache` means **revalidate before use** (you *can* store it, you just have to check with the server each time). `no-store` means **do not write it down** at all. *"Everybody gets this wrong exactly once."*

### 2.8 "SHOULD NOT maintain more than 2 connections" (§8.1.4)

**Plain English:** RFC 2068 told clients to open at most 2 connections per server, to be polite. Browsers wanted speed, so they ignored it.

```
$ python3 03-limits/fetch_page.py --port 9111 --rtt 150

   6 assets, RTT 150 ms
   mode                     bytes   round trips   TCP conns    wall ms
   close      (HTTP/1.0)     4867        12            6       1369.2
   keepalive  (1 conn)       5095         7            1        985.4
   parallel2  (RFC says)     5095         4            2        607.0
   parallel6  (browsers)     5095         2            6        608.2
   pipelined  (1 batch)      5095         2            1        303.7
```

How to read the round-trip counts (6 assets):
- **close**: each asset needs a handshake plus a request, so 6 × 2 = **12 RTT**.
- **keepalive**: 1 handshake + 6 sequential requests = **7 RTT**.
- **parallel2**: 2 connections opened in parallel (1 RTT), then 3 requests each = **4 RTT**.
- **parallel6**: 6 parallel handshakes (1 RTT) + 1 request each = **2 RTT**.
- **pipelined**: 1 handshake + all 6 requests sent at once and answered back to back = **2 RTT, on ONE connection**.

(🧠 `close` probably has fewer bytes because its HTTP/1.0 requests carry fewer headers. Wall time roughly tracks round trips × 150 ms, but not exactly: parallel6 measured about the same as parallel2 (608 vs 607 ms) despite fewer round trips, and the slide doesn't explain why. Treat the "round trips" column as the conceptual comparison. Pipelining is the clear winner on both columns.)

| Number | Meaning |
|---|---|
| **6** | connections per origin, what every browser actually shipped by about 2008 |
| **18** | via **domain sharding** (img1, img2, img3.example.com → 3 hostnames × 6). *"A hack on top of a violated rule."* |
| **dropped** | RFC 7230 removed the sentence in 2014. *"The spec conceded."* |

> *"The best row in that table is the one nobody shipped."*

That's **pipelining**: sending many requests without waiting for each response. It's the fastest row, but browsers never turned it on by default. The next sections explain why.

🧠 **Pipelining, drawn:**

```
 keep-alive (no pipelining)           pipelining
 ──────────────────────────           ─────────────────────────────
 C: GET /a  ──▶                        C: GET /a GET /b GET /c ──▶
 S:        ◀── 200 a                   S:           ◀── 200 a 200 b 200 c
 C: GET /b  ──▶                           (responses MUST come back in
 S:        ◀── 200 b                       the same order as requests)
 C: GET /c  ──▶
 S:        ◀── 200 c
```

---

## 3 · HTTP/1.1 TODAY · RFC 2616 (1999) → 7230 (2014) → 9110 (2022)

### 3.1 1999: mostly tightening, not adding

RFC 2616 (June 1999) replaced RFC 2068. The **genuinely new** items (§19.6.3, "Changes from RFC 2068"):

- **307 Temporary Redirect**
- **416 Range Not Satisfiable**
- **417 Expectation Failed** (+ `Expect:`)
- `identity` content-coding
- `TE:` and a transfer-coding registry
- **`Host` is now a MUST (and a MUST-400)**
- `ETag` required on resource creation
- charset wildcards in `Accept-*`
- byte ranges without a known length

**NOT new: 504 Gateway Timeout.** It's in RFC 2068 §10.5.5 already. The whole 5xx family arrived in 1997.

**Why 307 exists: a standard losing to deployed reality.**

```
 RFC 2068 said:   302 keeps the method.   (POST /form → 302 → POST /new)
 Reality:         every browser turned the POST into a GET anyway.
 RFC 2616 could have said "no, really".
 Instead it:      renamed 302 to "Found",
                  wrote down what implementations do,
                  added 307 for people who meant what 302 originally said
                  (307 = MUST keep the method and body).
```

> *"It is easier to add a number than to fix a billion deployed clients."*

🧠 For completeness: **303 See Other** explicitly means "now GET this other URL" (the Post/Redirect/Get pattern). **308 Permanent Redirect** (later, RFC 7538) is the method-preserving version of 301.

### 3.2 Framing: watch Postel's law lose an argument

🧠 **Postel's law (the robustness principle):** "be conservative in what you send, be liberal in what you accept." It comes from Jon Postel's early TCP/IP RFCs.

| RFC 2616 §4.4 · June 1999 | RFC 9112 §6.3 · June 2022 |
|---|---|
| *be liberal in what you accept* | *the same situation, 23 years later* |
| "If a message is received with both a Transfer-Encoding header field and a Content-Length header field, the latter MUST be ignored." | "A server MAY reject a request that contains both … or process such a request in accordance with the Transfer-Encoding alone. Regardless, the server MUST close the connection after responding." |
| **Recover. Do not reject.** In 1999 this was simply what good engineers did. | **Rejection is now allowed. Recovery is now optional. And either way, destroy the connection.** |

> *"Be liberal in what you accept is a fine rule for two programs. It is a catastrophic rule for a chain of five."*

Why it's catastrophic: if box 1 is "liberal" one way and box 4 is "liberal" another way, they read **the same bytes as different messages**. That's the next slide.

### 3.3 Request smuggling: one byte stream, two readings, one attack

**Plain English:** an attacker sends one ambiguous message. The front proxy (e.g. a CDN) thinks it's one request. The back server thinks it's two. The second one, hidden inside the first, slips past the CDN's security checks.

**One stream, sent once:**

```
POST /echo HTTP/1.1
Host: site-a.local
Content-Length: 6
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: site-a.local
```

**Two readings:**

```
 Reading with Content-Length: 6
 ──────────────────────────────
   ONE request. Its 6-byte body is   0 \r \n \r \n G
                                     1  2  3  4  5  6

 Reading with Transfer-Encoding: chunked
 ───────────────────────────────────────
   "0\r\n\r\n" = last chunk → body is EMPTY, request 1 ends here.
   Then "GET /admin HTTP/1.1 ..." = TWO requests. The second is GET /admin.
```

```
 attacker ──▶ [ CDN: uses Content-Length ] ──▶ [ origin: uses Transfer-Encoding ]
                 sees: POST /echo                 sees: POST /echo
                 (checks pass, /admin                    GET  /admin   ← smuggled!
                  never seen)
```

> *"Put a CDN that does the first in front of an origin that does the second, and you have posted a request the CDN never saw — and left a spare response in the connection for whoever is routed onto it next."*

🧠 The "spare response" part is the nastiest. The front proxy reuses its backend connection for **other users** (keep-alive again). The origin produced two responses but the proxy expected one, so the extra response is queued and **delivered to the next, innocent user**. Remember that HTTP/1.1 responses have no request ID (§4.2). Whoever is next in line gets it.

**Demo:**

```
$ python3 03-limits/framing.py --port 8011
    this server answered with 2 response(s):
     | HTTP/1.1 200 OK
     | HTTP/1.1 404 Not Found           <- the request nobody sent
```

(404 because the demo server has no `/admin`. The point is that a second request was parsed and answered.)

> **HTTP/2 makes this unrepresentable:** the length is in the frame header, and there's no second opinion to disagree with.

**Defences 🧠:** reject messages with both headers (RFC 9112 now allows this), close the connection after such messages, use HTTP/2 end to end, and make sure every box in the chain uses the same parser rules.

### 3.4 RFC 7230–7235 (June 2014): one document became six

| RFC | Title | Who has to read it carefully |
|---|---|---|
| 7230 | Message Syntax and Routing | Proxies, load balancers, CDNs, and the `https` scheme lives here now |
| 7231 | Semantics and Content | Application frameworks: methods, status codes, representations |
| 7232 | Conditional Requests | Anything with an ETag: browsers, caches, object stores |
| 7233 | Range Requests | Video, downloads, Parquet readers |
| 7234 | Caching | Every CDN on earth |
| 7235 | Authentication | The `WWW-Authenticate` / `Authorization` framework |

> The lecturer jokingly calls it **"Microservices for RFC"**. Same year, same argument: one large artefact with many stakeholders becomes several with clear ownership, versioned independently, so a change to caching doesn't require re-ratifying the definition of GET. Same trade-off too: 2616 was one place to look, and **everyone still says "2616" when they mean HTTP/1.1.**

### 3.5 Why your JWT is base64 (RFC 7230 §3.2.6)

The grammar for a **token** (used in header names, method names, and some header values):

```
tchar = "!" / "#" / "$" / "%" / "&" / "'" / "*" / "+" / "-" / "." /
        "^" / "_" / "`" / "|" / "~" / DIGIT / ALPHA
token = 1*tchar
excluded:   " ( ) , / : ; < = > ? @ [ \ ] { }
```

🧠 How to read it (ABNF, Augmented Backus-Naur Form): `/` means "or", and `1*tchar` means "one or more tchars". So a token can't contain spaces, quotes, braces, commas, colons, and so on.

**Correction on the slide: this isn't a 2014 rule.** RFC 7230 only wrote it as a list. The rule is from **RFC 2616 §2.2 (1999)**, where token = any CHAR except CTLs (control characters) or separators, and the separators list already contains `{` and `}`. Same rule, better prose, fifteen years earlier.

**And the JWT reason is broader.** A **JWT (JSON Web Token)** must survive an `Authorization` header, a URL query string, a cookie, a form field and a JSON body. JSON is full of `{ } " :`, which are all forbidden in tokens. **base64url** also swaps `+` and `/` for `-` and `_` so it's URL-safe. And the signature is **raw bytes**, which need encoding whatever the grammar says.

> *"Every layer you pass through has an alphabet. The intersection of all of them is about sixty-four characters wide — which is exactly why base64 exists, and why Session 2 met the same problem in MIME (mail)."*

🧠 base64 uses 64 symbols (A–Z, a–z, 0–9, plus two more), so each character carries 6 bits. 3 bytes become 4 characters, about 33% overhead.

### 3.6 RFC 9110–9114 (June 2022): the split that matters, and a teapot

| RFC | What it is | Version-specific? |
|---|---|---|
| **9110** | HTTP Semantics: methods, status, headers, ranges, auth | **No.** Shared by 1.1, 2 and 3. |
| **9111** | HTTP Caching | **No.** Shared by all three. |
| 9112 | HTTP/1.1 | Wire format only |
| 9113 | HTTP/2 | Wire format only |
| 9114 | HTTP/3 | Wire format only |

**The table of contents is the argument.** Everything HTTP *means* is in 9110 and 9111, shared by all three versions. 9112, 9113 and 9114 only describe how to write it down on a socket. **Three encodings, one language.**

```
             ┌──────────────────────────────────────────┐
             │  SEMANTICS (RFC 9110) + CACHING (9111)   │  GET, 200, ETag, Range, Vary...
             └──────────────────────────────────────────┘
                 │                 │                │
          ┌──────▼─────┐   ┌───────▼──────┐  ┌──────▼──────┐
          │ RFC 9112   │   │ RFC 9113     │  │ RFC 9114    │
          │ HTTP/1.1   │   │ HTTP/2       │  │ HTTP/3      │
          │ text lines │   │ binary frames│  │ QUIC streams│
          └──────┬─────┘   └───────┬──────┘  └──────┬──────┘
                TCP          TCP (+TLS)           QUIC/UDP
```

**418 (Unused):** an April Fool's joke (RFC 2324, 1998, "I'm a teapot") that *"has been widely implemented as an easter egg, and therefore is effectively consumed by this use."* It's reserved forever, not because it's useful, but because too many people shipped the joke. (RFC 9110 §15.5.19; the effort to keep it was `draft-nottingham-thanks-larry`.) The IANA registry, as of 2025-09-15, has **62 assigned codes**.

---

## 4 · WHAT IT COST: THE BILL FOR HTTP/1.1

### 4.1 Head-of-line blocking: five fast responses, waiting 2 seconds for a sixth

```
 Pipelined, one connection              One connection each
 all six requests sent at t=0           same six requests

 /big.txt?delay=2.0   2003.4 ms         /big.txt?delay=2.0   2006.0 ms
 /style.css           2003.6 ms         /style.css              3.2 ms
 /app.js              2003.8 ms         /app.js                 5.7 ms
 /logo.png            2004.1 ms         /logo.png               6.1 ms
 /icon.png            2004.2 ms         /icon.png               6.9 ms
 /hero.png            2004.4 ms         /hero.png               7.2 ms
```

Every one of those five was ready in microseconds. They waited two seconds for a file none of them needed. Same server, same files, same work. The only difference is whether they had to queue behind each other.

This is **real head-of-line blocking**: a queue (pipelining) plus a slow item at the head. It was created by the fix for HTTP/1.0's serialisation cost.

🧠 That's why browsers never enabled pipelining by default. One slow response (or a buggy proxy that mangled pipelined requests) made everything worse. They opened 6 parallel connections instead.

### 4.2 The cause: there is no request ID on the wire

> **The order IS the identifier.**
> Nothing in an HTTP/1.1 response says which request it answers. The client knows only because it counted. That makes order mandatory, and it makes the slowest response set the pace for everything behind it.

**Multiplexing is only possible if messages carry identity:**

| Protocol | The identifier | Where you met it |
|---|---|---|
| TCP | source port / destination port | Session 2 |
| SS7 / TCAP | transaction id | Session 2: SMS and call setup |
| FastCGI | `requestId`, in every record header | Session 3: php-fpm behind nginx |
| **HTTP/1.1** | **— nothing —** | and so: strict response ordering |
| HTTP/2 · HTTP/3 | stream identifier, 31 bits, every frame | |

🧠 **Multiplexing** means carrying many independent conversations over one channel by tagging each piece with which conversation it belongs to. 🧠 SS7 (Signalling System No. 7) is the telephone network's signalling system, and TCAP (Transaction Capabilities Application Part) is its transaction layer.

### 4.3 Headers: 97.5% of what you upload is a repeat

```
$ python3 03-limits/headers.py --requests 80

   one request, headers only               805 bytes
   x 80 requests for one page            64320 bytes
   new information (the paths)            1600 bytes     2.5%
   byte-for-byte repeats                 62720 bytes    97.5%

   On a 1 Mbit uplink that is 0.51 s of upload before
   a single byte of your application data has moved.
```

(Check: 64320 bytes × 8 = 514,560 bits ÷ 1,000,000 bit/s ≈ 0.51 s.)

| | |
|---|---|
| **805 B** | of headers on one realistic request: cookies, `User-Agent`, `Accept-*`, `sec-fetch-*` |
| **upload** | the direction you have least of (🧠 home links are asymmetric), spent at the start of the page, while the user watches |
| **why not gzip?** | the obvious answer, and the wrong one (see CRIME, §5.6) |

### 4.4 A bug the lecturer hit: when keep-alive made it slower (Nagle + delayed ACK)

**Plain English:** two TCP optimisations, each sensible alone, deadlock for 40 ms when combined. It only happens on a persistent connection.

**Two algorithms, both correct:**
- **Nagle (RFC 896, 1984):** don't send a small segment while an earlier one is unacknowledged. (It batches tiny writes, which originally targeted telnet keystrokes.)
- **Delayed ACK (RFC 1122):** don't acknowledge immediately. Wait up to 40 ms to piggyback the ACK on some data.

**The deadlock:**

```
 SERVER                                         CLIENT
 write(head)  ──── goes out (nothing unacked) ──▶
 write(body)  → Nagle HOLDS it (head not ACKed yet)
                                                 has nothing to send back yet
                                                 (waiting for body) → ACK DELAYED
              ............ 40 ms ............
              ◀──────────────────────── ACK ────
 body goes out ─────────────────────────────────▶
                                    Every request but the 1st.
```

**Measured, on loopback (`03-limits/nagle.py`):**

```
 write(head); write(body)
   /style.css      0.9 ms
   /app.js        41.5 ms    <- delayed ACK
   /logo.png      44.0 ms    <- delayed ACK
   /icon.png      43.9 ms    <- delayed ACK
   /hero.png      44.0 ms    <- delayed ACK
   total         174.3 ms

 one write, + TCP_NODELAY
   total           3.2 ms
```

**Why only on a persistent connection?** With HTTP/1.0, the server calls `close()` right after `write(body)`, and closing pushes the pending data out immediately. *"The fix for one problem exposed it."* 🧠 The first request is fast because nothing is unacknowledged yet when the body is written, or because the client side starts in quick-ACK mode.

**The fix (two parts):**
1. **Coalesce:** build head + body into one buffer and do a single `write()` (or use `writev()`/`sendmsg()`).
2. **`TCP_NODELAY`:** turn Nagle off on that socket.

```python
import socket
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)   # disable Nagle
s.sendall(head + body)                                     # one write
```

> *"Your protocol sits on a protocol with its own opinions. nginx and Apache both coalesce and both set TCP_NODELAY — now you know why."*

### 4.5 Closing: the bill for HTTP/1.1

| The wall | Measured today | Fixed by |
|---|---|---|
| Head-of-line blocking | 2003 ms vs 3 ms for the same file, pipelined | Stream ids: HTTP/2 |
| Header bloat | 97.5% of uploaded header bytes are repeats | HPACK: HTTP/2 |
| Too many connections | 6 per origin, 18 with domain sharding | One connection: HTTP/2 |
| Ambiguous framing | One byte stream, two readings, one attack | Frame headers: HTTP/2 |
| TCP itself | 3 round trips before the first byte | QUIC: HTTP/3 |

### 4.6 The big insight: it was never "text is slow"

> *"Text based — you die early, or you live long enough…"* Text framing is **why HTTP won**: anyone could implement it, anyone could debug it with telnet. That beat every better-engineered rival of the 1990s.
>
> But "text is slow" isn't the finding: we measured gzip at 2.8%. **The problem was text without lengths and without identifiers.** HTTP/2 isn't "HTTP but binary". It's **"HTTP but with lengths and ids"**, and binary is the consequence.

---

## 5 · HTTP/2 · RFC 7540 (2015) → RFC 9113 (2022)

### 5.1 Where it came from: ship it to a billion users, then standardise it

| When | What | The number that made the case |
|---|---|---|
| Nov 2009 | **SPDY** announced (Belshe & Peon, Chromium blog) | "up to 55% faster", top 25 sites, in a lab |
| 2010–2014 | Chrome, Firefox, nginx, Twitter, Facebook ship it | Google runs both ends and measures production |
| Feb 2015 | Google announces SPDY's retirement | >25% of Chrome resources on HTTP/2 vs <5% SPDY |
| May 2015 | **RFC 7540 (HTTP/2) + RFC 7541 (HPACK)** | SPDY with the edges filed off |
| May 2016 | SPDY removed in Chrome 51 | six and a half years, experiment to obsolete |
| Jun 2022 | **RFC 9113 obsoletes 7540** | deprecates priorities and the h2c Upgrade dance |

🧠 Google could do this because it controlled **both ends**, the most popular browser (Chrome) and the most popular websites. It could deploy, measure real traffic, and iterate. That's the same playbook later used for QUIC.

> **Naming gotcha:** it's called **HTTP/2, not HTTP/2.0**, on purpose. The working group dropped the minor version because "1.1" had produced a decade of confusion about what a 1.x server must support. There's no HTTP/2.1 and there won't be one. The next version is a new number and a new wire format.

### 5.2 The connection preface: twenty-four bytes that are also a joke (RFC 9113 §3.4)

Before anything else, an HTTP/2 client sends this exact 24-byte "client connection preface":

```
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
50 52 49 20 2a 20 48 54 54 50 2f 32 2e 30 0d 0a
0d 0a 53 4d 0d 0a 0d 0a                      exactly 24 octets, always
```

**Why this string?** An HTTP/1.1 server that receives it sees method `PRI`, path `*`, version `HTTP/2.0`, then an empty line, which ends the headers. It fails cleanly with **501** instead of trying to parse the binary frames behind it as text. It's a deliberate **tripwire for old servers**. (`PRI…SM` is what's left of "PRISM". It was 2013. The RFC doesn't say so.)

🧠 Hex check: `50 52 49` = "PRI", `20` = space, `2a` = "*", `0d 0a` = `\r\n`, `53 4d` = "SM".

**How the client knows the server speaks h2:**

| Over TLS: how it actually happens (RFC 9113 §3.2 · ALPN is RFC 7301) | Cleartext: the dance that died (RFC 9113 §3.3 and §3.1) |
|---|---|
| `$ h2_client.py --port 8443 --tls` → `TLS TLSv1.3  ALPN negotiated: 'h2'` | `$ h2_client.py --port 8020` → `h2c by prior knowledge` |
| The client lists `h2` and `http/1.1` inside the **ClientHello**. The server picks one in the **ServerHello**. **Zero extra round trips**, because it rides in a handshake you already paid for. | RFC 7540 had an `Upgrade: h2c` dance. RFC 9113 §3.1: it *"was never widely deployed and is deprecated by this document."* **No browser ever shipped cleartext HTTP/2 at all.** |

🧠 **ALPN (Application-Layer Protocol Negotiation)** is a TLS extension. Think of it as the client saying "I speak h2, http/1.1" inside the TLS hello, and the server replying "h2". **h2c** means HTTP/2 over cleartext TCP. "Prior knowledge" means the client simply knows the server speaks h2c and starts with the preface straight away (common between internal services, e.g. gRPC without TLS).

### 5.3 The frame header: nine bytes, and 31 bits of them are the answer (§4.1)

**Plain English:** everything in HTTP/2 is sent as a **frame**. Every frame starts with the same 9-byte header that says how long it is, what type it is, and **which stream (request) it belongs to**. Those 31 bits of stream ID are what HTTP/1.1 never had.

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-----------------------------------------------+
 |                 Length (24)                   |
 +---------------+---------------+---------------+
 |   Type (8)    |   Flags (8)   |
 +-+-------------+---------------+-------------------------------+
 |R|                 Stream Identifier (31)                      |
 +=+=============================================================+
 |                   Frame Payload (0 … Length)                  |
 +---------------------------------------------------------------+

   3 bytes length + 1 type + 1 flags + 4 (1 reserved bit + 31-bit id) = 9 bytes
```

**Every frame header, raw, from a client written by hand:**

```
$ python3 04-http2/h2_client.py --port 8020 --path /
   -> preface    b'PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n'
   -> HEADERS stream=1 GET /         67B of text -> 15B hpack
   <- 00 00 2a 04 00 00 00 00 00     SETTINGS   len=42   stream=0
   <- 00 00 4d 01 04 00 00 00 01     HEADERS    len=77   stream=1   END_HEADERS
   <- 00 01 07 00 00 00 00 00 01     DATA       len=263  stream=1
   <- 00 00 00 00 01 00 00 00 01     DATA       len=0    stream=1   END_STREAM
```

**Decode one by hand 🧠** (`00 00 4d 01 04 00 00 00 01`):

```
 00 00 4d     Length   = 0x00004d = 77 bytes of payload follow
 01           Type     = 0x01 = HEADERS
 04           Flags    = 0x04 = END_HEADERS (no CONTINUATION follows)
 00 00 00 01  R + Stream ID = 1  (first client stream)
```

And `00 01 07 00 00 00 00 00 01`: length 0x000107 = **263** (the same 263-byte HTML page as HTTP/1.0's `Content-Length: 263`), type DATA, stream 1. The last frame is length 0 with flag `0x01` = **END_STREAM**: "this request/response is done".

**Stream ID rules (§5.1.1):**
- **Client streams are odd** (1, 3, 5, …), **server streams are even**, so nobody has to coordinate.
- **Stream 0 is the connection itself**: SETTINGS, PING, GOAWAY.
- 🧠 IDs only go up and are never reused. When you run out (2³¹), you open a new connection.

> **Request smuggling isn't "fixed" here. It's unrepresentable:** the length is in a fixed place, at a fixed size, with no second opinion.

**🧠 Why 24 / 8 / 8 / 31?** (The course project asks you to defend your own choices, so it's worth thinking about.)
- **24-bit length** allows frames up to 16 MB, but the default `MAX_FRAME_SIZE` is only 16384, so big frames have to be negotiated. Small frames let streams interleave finely, so one huge DATA frame can't hog the connection.
- **8-bit type** gives 256 possible types. Only 10 are defined, which leaves room for extensions. Unknown types **must be ignored**, and that's what makes extensions possible.
- **8 flags** give per-frame booleans (END_STREAM, END_HEADERS, PADDED, PRIORITY, ACK).
- **31-bit stream ID + 1 reserved bit**: 2 billion streams per connection is plenty. The reserved bit is commonly explained as avoiding sign problems with signed 32-bit ints (e.g. Java has no unsigned int).

### 5.4 Ten frame types, and what each replaced (§6)

| Type | Name | What it does | What it replaced in HTTP/1.1 |
|---|---|---|---|
| 0x00 | DATA | The body, in chunks ≤ MAX_FRAME_SIZE | `Content-Length` / chunked encoding |
| 0x01 | HEADERS | Opens a stream; carries an HPACK block | The request line and header block |
| 0x02 | PRIORITY | Deprecated by RFC 9113 | → RFC 9218 `PRIORITY_UPDATE` |
| 0x03 | RST_STREAM | Cancel **ONE** stream | Closing the TCP connection, killing everything |
| 0x04 | SETTINGS | Announce your limits, both directions | Nothing: there was no way to say |
| 0x05 | PUSH_PROMISE | Server push | Nothing, and it didn't last (§5.9) |
| 0x06 | PING | Round-trip time; keep intermediaries awake | Nothing |
| 0x07 | GOAWAY | "Last stream id I will process": drain | `Connection: close`, mid-flight |
| 0x08 | WINDOW_UPDATE | Flow control, per stream **AND** per connection | TCP's window, and only TCP's |
| 0x09 | CONTINUATION | A header block bigger than one frame | Nothing, and a **2024 DoS class** |

**§6.5.2 SETTINGS defaults:**
- `HEADER_TABLE_SIZE` **4096** (HPACK dynamic table size in bytes)
- `INITIAL_WINDOW_SIZE` **65535** (flow-control window per stream)
- `MAX_FRAME_SIZE` **16384**
- `MAX_CONCURRENT_STREAMS` has **no default** (unlimited), but the RFC recommends it be *"no smaller than 100"*.

🧠 Notes:
- **RST_STREAM** lets a browser cancel one image download (the user scrolled away) without killing the other 50 in-flight requests. 🧠 It was also abused in the 2023 "Rapid Reset" DDoS (open a stream, cancel it at once, repeat).
- **GOAWAY** enables graceful shutdown. The server says "I'll finish streams up to N, then go away", so the client knows exactly which requests are safe to retry elsewhere.
- **Flow control** exists because many streams share one TCP connection. One fast stream shouldn't be able to fill the receiver's memory and starve the others. So each stream has its own window, *plus* there's one for the whole connection.
- **CONTINUATION flood (2024)**: an attacker sends an endless stream of CONTINUATION frames with no END_HEADERS. Some servers buffered them all in memory.

### 5.5 HPACK (RFC 7541): 174 bytes → 79 → 6

**Plain English:** most headers repeat on every request. HPACK gives both sides a shared numbered list of headers. After the first request, you send "header #62" instead of the whole 300-byte cookie.

**Three mechanisms, none of them clever:**

**1. Static table: 61 entries.** Fixed, identical everywhere (Appendix A). Index 2 is `:method: GET`, name **and** value. The whole of `GET` costs one byte: **0x82**. *Not compressed. Indexed.*

🧠 Why `0x82`? An "indexed header field" starts with bit pattern `1`, followed by a 7-bit index. `1` + `0000010` (2) = `10000010` = 0x82. A few static entries for reference: 1 `:authority`, 2 `:method GET`, 3 `:method POST`, 4 `:path /`, 6 `:scheme http`, 7 `:scheme https`, 8 `:status 200`.

🧠 **Pseudo-headers** (`:method`, `:path`, `:scheme`, `:authority`, `:status`) replace the HTTP/1.1 request line and status line. `:authority` is the new `Host`.

**2. Dynamic table: from index 62.** Anything sent before **on this connection** gets an index. Your 300-byte cookie costs 300 bytes once, then 1–2 bytes forever.

```
 Request 1:  cookie: session=abc...(300 B)  → send literally, "add to table" → becomes #62
 Request 2:  cookie: session=abc...         → send 0xBE  (index 62)  ← 1 byte!

 index:  1 ........ 61 | 62  63  64 ...
         STATIC (fixed)| DYNAMIC (newest entry gets 62, older ones shift up;
                       |          oldest evicted when > HEADER_TABLE_SIZE 4096 B)
```

**3. Huffman: Appendix B.** A canonical code built from real header text: "e" is 5 bits, "X" is 8. It's applied per string, and only when it actually shrinks it.

🧠 **Integer encoding with an N-bit prefix** (tested by RFC vectors C.1.1 and C.1.2). Integers fill the leftover bits of a byte. If the value doesn't fit, the prefix is all 1s and continuation bytes follow, 7 bits at a time:

```
 C.1.1:  10 in a 5-bit prefix     → 10 < 31, fits:   xxx01010            (1 byte)
 C.1.2:  1337 in a 5-bit prefix   → 1337 ≥ 31:
           prefix = 11111 (31)        remaining 1337 − 31 = 1306
           1306 % 128 = 26 → 26+128 = 154 = 10011010   (high bit 1 = "more")
           1306 / 128 = 10            = 00001010       (high bit 0 = "last")
         → xxx11111 10011010 00001010                                     (3 bytes)
```

**Ours, against the RFC's own vectors (`04-http2/hpack_mini.py`):**

```
[ok] C.1.1   10 in a 5-bit prefix
[ok] C.1.2   1337 in a 5-bit prefix
[ok] C.4.1   huffman 'www.example.com'
[ok] C.6.1   decode response 1
[ok] round trip request 1
[ok] round trip request 2
[ok] pypi 'hpack' agrees with our encoder

the same header set, three ways:
  HTTP/1.1 text          174 bytes
  HPACK, first time       79 bytes    55%
  HPACK, second time       6 bytes    97%
```

**Six bytes**: six one-byte indices for six header fields.

> **The dynamic table is per connection**, which is why HTTP/2 wants **ONE** connection. **Domain sharding was clever under 1.1 and is actively harmful here**: every shard starts with an empty table (and a cold cwnd, and its own TLS handshake).

### 5.6 CRIME (2012): why not just gzip the headers?

**Plain English:** compression makes repeated text smaller. If an attacker can put their own guess next to your secret and watch the compressed size, a correct guess shrinks the output by a byte. They learn the secret one character at a time, **without decrypting anything**.

**Because compressed size leaks the secret:**

```
// the attacker cannot decrypt anything. it only watches SIZE.
GET /aaaa?guess=a      Cookie: session=SECRET…
    -> compressed length 412        no repeat found
GET /aaaa?guess=b      Cookie: session=SECRET…
    -> compressed length 412        no repeat found
GET /aaaa?guess=S      Cookie: session=SECRET…
    -> compressed length 411        <- the compressor found "S" twice
// one byte smaller means "correct". repeat per character.
```

🧠 How the attacker gets that position: they make your browser send requests (e.g. via JavaScript on an evil page) to the target site, so your cookie is attached automatically. They then watch encrypted packet sizes on the network (e.g. on the same Wi-Fi). gzip/DEFLATE uses LZ77, which replaces repeats with back-references, so attacker-controlled text that matches the secret compresses better.

**HPACK's defence is structural:**
- The **Huffman table is fixed**: built once from a corpus, published in Appendix B, never adapted to the data flowing through it. There's no shared adaptive dictionary for attacker text and secret text to meet in.
- The **dynamic table is an exact-match index, not a substring matcher**. Guessing a *prefix* of your cookie wins nothing. You must guess the whole value.
- **RFC 7541 §7.1** goes further: mark sensitive fields **"never indexed"**.

> **Yours too:** gzip a JSON response containing a user-supplied search term and an API key together, and you've rebuilt CRIME. (🧠 The response-body version of this is called BREACH, 2013.)

### 5.7 Multiplexing: same six requests, one connection, 400× faster

```
 HTTP/1.1 pipelined (one connection)      HTTP/2 (one connection, the SAME one)

 /big.txt?delay=2.0   2003.4 ms           stream  3  /style.css      4.9 ms
 /style.css           2003.6 ms           stream  7  /logo.png       5.7 ms
 /app.js              2003.8 ms           stream  9  /icon.png       6.9 ms
 /logo.png            2004.1 ms           stream  5  /app.js         6.9 ms
 /icon.png            2004.2 ms           stream 11  /hero.png       7.8 ms
 /hero.png            2004.4 ms           stream  1  /big.txt     2005.9 ms
```

| HTTP/1.1 pipelined | HTTP/2 |
|---|---|
| Responses **must** arrive in request order, because order is the only identifier there is. | Odd ids, client-initiated. Arriving **out of order, on purpose**, because each frame says who it's for. |

(2003 ms → ~5 ms for the small files is the "400×".)

```
 HTTP/2 on the wire (interleaved frames, one TCP connection):

 → HEADERS s1 | HEADERS s3 | HEADERS s5 | HEADERS s7 | ...
 ← HEADERS s3 | DATA s3 | HEADERS s7 | DATA s7 | DATA s9 | ... ... | DATA s1 (2 s later)
```

**And the dynamic table warming up in real time:**

```
-> HEADERS stream=1   GET /big.txt?delay=2.0   84B of text -> 30B hpack
-> HEADERS stream=3   GET /style.css           76B of text -> 12B hpack
-> HEADERS stream=5   GET /app.js              73B of text -> 11B hpack
-> HEADERS stream=7   GET /logo.png            75B of text -> 12B hpack
```

The first request pays to fill the table (30 B). Later ones are mostly indices, and only the changing `:path` costs anything.

> *Same GET. Same 200. Same ETag. Our h2 handler is line-for-line the shape of the HTTP/1.1 one — that's RFC 9110 doing its job.* (Semantics stayed the same while the encoding changed.)

### 5.8 The bug the lecturer shipped first: a protocol can only give you permission

**Handling requests inline in the read loop (what was written first):**

```python
for event in conn.receive_data(data):
    if isinstance(event, RequestReceived):
        self.serve(event.stream_id, ...)        # blocks the loop for 2 s on /big.txt
```

```
stream 1    /big.txt?delay=2    2005.6 ms
stream 3    /style.css          2005.7 ms
stream 5    /app.js             2005.7 ms
stream 7    /logo.png           2005.8 ms
stream 9    /icon.png           2005.8 ms
stream 11   /hero.png           2005.9 ms
             ^ Exactly the HTTP/1.1 numbers.
```

**One thread per stream, and a lock around the connection:**

```python
threading.Thread(target=self.safe_serve,
                 args=(event.stream_id, ...),
                 daemon=True).start()
```

```
stream 3    /style.css            4.9 ms
stream 7    /logo.png             5.7 ms
stream 9    /icon.png             6.9 ms
stream 5    /app.js               6.9 ms
stream 11   /hero.png             7.8 ms
stream 1    /big.txt?delay=2   2005.9 ms
             ^ Same frames. Same client. Same wire.
```

🧠 The lock is needed because all threads write frames to the **same** socket and the same HPACK encoder state. Two threads writing at once would interleave bytes mid-frame and corrupt the connection.

> **Multiplexing in the protocol only *permits* concurrency. You still have to implement it.**
> You'll meet this exact shape again: a worker pool sharing one database connection · a blocking call inside a coroutine · a gRPC handler holding a global mutex · six goroutines waiting on one channel. *The transport is fine. You serialised the work yourself.*

### 5.9 Server push: nobody deprecated it, everybody removed it

🧠 **The idea:** the client asks for `index.html`, and the server knows it'll need `style.css`, so it sends a `PUSH_PROMISE` and pushes the CSS without being asked. That saves one round trip.

| What the RFC says (still standard, today) | What implementers did (Chrome 106, 27 September 2022) |
|---|---|
| Retained in RFC 9113 §8.4. Retained in RFC 9114 §4.6 (HTTP/3). | **1.25%** of HTTP/2 sites used it. Later **0.7%**. |
| Search RFC 9113 for "deprecat" and you get exactly two hits, neither of them push: RFC 7540 **PRIORITY** (§2, §5.3) and the **h2c Upgrade token** (§3.1). | *"no clear net performance gain and in many cases performance regressions"* |
| If you've heard "the RFC deprecated server push": **it didn't.** | **The fundamental problem: the server doesn't know what's in your cache.** So it pushes a file you already have, spending bandwidth AND competing with what you actually needed, on the same connection and congestion window. |

**The replacement: 103 Early Hints (RFC 8297).** Don't push the file. Say "you're going to want style.css". The client checks its own cache and decides.

```
 C: GET /index.html
 S: 103 Early Hints
    Link: </style.css>; rel=preload        ← client checks its cache; fetches only if missing
 S: 200 OK  (index.html, after the server finishes generating it)
```

> *"The server keeps the knowledge; the client keeps the decision. When one side has the knowledge and the other has the state, **inform — do not act.**"*

### 5.10 The one thing HTTP/2 could not fix: TCP head-of-line blocking

> *"because the parallel nature of HTTP/2's multiplexing is not visible to TCP's loss recovery mechanisms, a lost or reordered packet causes all active transactions to experience a stall regardless of whether that transaction was directly impacted by the lost packet."* (RFC 9114 §1.1)

**Plain English:** HTTP/2 has six independent streams, but TCP sees one long ordered byte stream. If one TCP packet is lost, TCP holds back **every byte after it** until the retransmission arrives, even bytes belonging to streams that lost nothing.

```
 TCP byte stream:   [s1][s3][s5][ LOST ][s7][s9][s1][s3]...
                                  ▲
         kernel cannot hand ANY later byte to the app
         until this hole is filled (in-order guarantee)
```

```
$ h2_client.py --port 9020 --multiplex          # through tools/stall_tcp.py
   stall: holding the stream at byte 3000 for 300 ms

   run 1   logo 4.5   style 4.7  | app 306.3   hero 307.0   icon 307.3   big 308.9
   run 2   style 4.0  icon 5.2   | logo 308.2  big 308.3    hero 308.4   app 308.4
   run 3   style 3.5             | big 306.4   app 306.5    icon 306.5   logo 306.7  hero 306.8
           ^ through before        ^ everything behind the hole, whether or not
             the hole                it lost a single byte
```

The later bytes had **arrived**. They were in the receive buffer. The kernel can't hand them over without breaking the in-order guarantee that is the reason you chose TCP. And **which** streams get caught changes every run: the blast radius is "whoever had bytes behind the gap", and you don't get to choose.

> Google's SPDY data: **11.81% speedup at 0% loss, 47.7% at 2% loss.** Multiplexing helps most exactly where TCP hurts most.

🧠 There's an irony here. Under HTTP/1.1 with 6 connections, one lost packet stalls only 1 of the 6. Under HTTP/2 with **one** connection, it stalls **all** of them. On lossy networks, the single connection that HTTP/2 wanted becomes a liability. That pushes the fix below TCP, into QUIC.

---

## 6 · QUIC + HTTP/3 · RFC 9000 · RFC 9114

🧠 **What QUIC is:** a new transport protocol (RFC 9000; TLS integration in RFC 9001). It provides what TCP + TLS provided (reliability, ordering, congestion control, encryption) plus **multiple independent streams**, and it runs over UDP. **HTTP/3** (RFC 9114) is HTTP's semantics mapped onto QUIC streams.

```
  HTTP/1.1          HTTP/2            HTTP/3
 ┌────────┐       ┌────────┐       ┌──────────┐
 │  HTTP  │       │ HTTP/2 │       │  HTTP/3  │
 │ (text) │       │ frames │       │ + QPACK  │
 ├────────┤       ├────────┤       ├──────────┤
 │ (TLS)  │       │  TLS   │       │   QUIC   │ ← streams, reliability,
 ├────────┤       ├────────┤       │ (TLS 1.3 │   congestion ctrl, crypto
 │  TCP   │       │  TCP   │       │  inside) │   all in one, in userspace
 ├────────┤       ├────────┤       ├──────────┤
 │   IP   │       │   IP   │       │   UDP    │
 └────────┘       └────────┘       ├──────────┤
                                   │    IP    │
                                   └──────────┘
```

### 6.1 Why UDP: because the internet stopped accepting new protocols

**The obvious answer, and why it fails:**

```
 IP protocol numbers:   TCP = 6    UDP = 17    SCTP = 132
 "Just take protocol number 253 and build a proper transport."
```

Except that every NAT, firewall, CGNAT, corporate proxy and hotel router on earth **drops what it doesn't recognise**. **SCTP** is a good protocol, standardised in 2000, and you can't use it across the public internet. That's why.

> *"UDP is not a good foundation. It is a hole that already exists."*

🧠 **CGNAT** (Carrier-Grade NAT) is NAT run by your ISP, shared by thousands of customers. 🧠 **SCTP** (Stream Control Transmission Protocol) already had multi-streaming in 2000, so it would have solved HOL blocking, but middleboxes killed it. **Protocol ossification** is the general term: the network "fossilises" around what it has seen.

**And so: encrypt almost everything.**

> *"middleboxes could observe traits of a specific version of QUIC and assume that when other versions exhibit similar traits the same underlying semantic is being expressed."* (RFC 8999 §7)

Anything a middlebox **can** read, it'll eventually **depend on**, and then you can never change it. That's how **TCP options became unusable** (🧠 middleboxes strip or choke on unknown TCP options, so new TCP features like TCP Fast Open struggled to deploy). QUIC encrypts packet numbers and most of the header, **not only for privacy, but to keep the protocol changeable.**

**And it lives in userspace.** A TCP change is a kernel upgrade on a billion machines and takes a decade. A QUIC change ships with your app. 🧠 Chrome ships its own QUIC stack and can update it every few weeks.

### 6.2 The round trips: one handshake, not two

```
 TCP + TLS 1.3 + HTTP/2  (3 round trips, cold)   QUIC + HTTP/3  (2 round trips, cold)
 ────────────────────────────────────────────    ─────────────────────────────────────
 SYN         ->                                  Initial   ->  TLS ClientHello inside
             <-  SYN/ACK                                       a QUIC CRYPTO frame
 ACK         ->                    1 RTT                   <-  Handshake            1 RTT
     ...nothing has been said yet.
 ClientHello ->   (ALPN rides here)              GET       ->
             <-   ServerHello …                            <-  200                  2 RTT
 Finished    ->                    2 RTT
 GET         ->
             <-   200              3 RTT
 With TLS 1.2 it was four.
```

- **RFC 9001 §2.1:** there's no transport handshake followed by a crypto handshake. **There is one handshake.**
- **TLS is not optional. There is no unencrypted QUIC.**

```
$ python3 05-http3/handshake_race.py --rtt 150
                                          measured     round trips
   TCP + TLS 1.3 + h2 (new connection)    456.6 ms        3.0
   QUIC + HTTP/3 (new connection, 1-RTT)  319.8 ms        2.1
```

> *"Layering is how we make systems comprehensible — and every layer boundary is a place you may pay a round trip."*

### 6.3 0-RTT (RFC 9001 §4.6): zero round trips, and the price of them

**Plain English:** if you've talked to this server before, you kept a "resumption ticket". Next time, you can send your GET **in the very first packet**, encrypted with keys derived from that ticket, without waiting for the handshake to finish.

```
$ python3 05-http3/handshake_race.py --rtt 150
                                                measured    round trips
   TCP + TLS 1.3 + h2 (new connection)          456.6 ms       3.0
   QUIC + HTTP/3 (new connection, 1-RTT)        319.8 ms       2.1
   QUIC + HTTP/3 (0-RTT, request in flight 1)   161.7 ms       1.1

   Cold connection: QUIC saves 137 ms.     Resumed: 0-RTT saves 295 ms.
```

(456.6 − 319.8 ≈ 137; 456.6 − 161.7 ≈ 295.)

**The price: replay attacks.**

> RFC 9001 §9.2: *"use of TLS early data comes with an exposure to replay attack… Endpoints MUST implement and use the replay protections described in [TLS13], however it is recognized that these protections are imperfect."*

**Why imperfect:** the defence is a single-use ticket cache, and a server farm has many servers. An attacker who captured your early data can send it to a **different** server, which has never seen the ticket.

```
 You ──[0-RTT: POST /pay ₹500]──▶ server A   ✓ charged
 Attacker copies the packet ─────▶ server B   (B's ticket cache never saw it) ✓ charged AGAIN
```

**So the rule is yours to make:**

> *"An application protocol that uses QUIC MUST describe how the protocol uses 0-RTT and the measures that are employed to protect against replay attack."*

**In practice: idempotent requests only.** GET is fine. A POST that charges a card isn't. **Cloudflare allows only GET and HEAD**, and sets an `Early-Data` header so your origin can refuse with **425 Too Early**. (🧠 425 and `Early-Data` are defined in RFC 8470.)

🧠 **Idempotent** means doing it twice has the same effect as doing it once (GET, HEAD, PUT, DELETE are idempotent by definition; POST isn't).

> *"0-RTT is a cache of a negotiation. Every cache of a negotiation trades a round trip for the possibility of a replay."*

### 6.4 Head-of-line blocking, gone: only the stream that lost data waited

```
 HTTP/2 over TCP                          HTTP/3 over QUIC
 one 300 ms stall in the byte stream      18 datagrams really deleted (5%)

 /logo.png     4.5 ms  before the hole    /style.css     4.5 ms
 /style.css    4.7 ms  before the hole    /app.js        5.4 ms
 /app.js     306.3 ms                     /logo.png      6.7 ms
 /hero.png   307.0 ms                     /icon.png      7.8 ms
 /icon.png   307.3 ms                     /hero.png     11.0 ms
 /big.txt    308.9 ms                     /big.txt      78.2 ms   <- lost packets
```

| HTTP/2 over TCP | HTTP/3 over QUIC |
|---|---|
| Streams that lost nothing stalled, and a different set of them each run. | Only the stream that lost data waited. |
| TCP can't see your streams. One ordered byte pipe, one gap, everything behind it waits. | Every **STREAM frame carries a stream id and an offset**. Ordered **within** a stream, unordered **between** them. |

```
 QUIC packets:  [s3 off 0][s1 off 0][ LOST: s1 off 1200 ][s5 off 0][s3 off 1200]
                                         ▲
                  only stream 1 waits for the retransmit; s3 and s5
                  are delivered to the app immediately
```

> **⚠️ Be precise about this experiment (the lecturer's own caveat):** the TCP side *delays* a chunk (the shape a lost segment makes). The QUIC side *really deletes* 18 datagrams. These are different perturbations, so **don't read 307 vs 78 as a benchmark.** What *is* comparable is the **blast radius**: on TCP it's "whoever had bytes behind the gap", a different set every run. On QUIC it's **exactly one stream, every time.**

### 6.5 Connection IDs (§5.1, §9): your connection survives changing networks

**TCP: the connection IS the four-tuple** (src IP, src port, dst IP, dst port). Change any one of them and it's a different connection, which is to say a **dead** connection.

```
 Walk out of the coffee shop:
    wifi -> LTE, your IP changes
    every TCP connection dies
    video buffers, download restarts,
    websocket reconnects, spinner
 Same thing when your NAT quietly rebinds your external port after idle.
```

**QUIC: identity is not addressing.**

> *"The primary function of a connection ID is to ensure that changes in addressing at lower protocol layers (UDP, IP) do not cause packets for a QUIC connection to be delivered to the wrong endpoint."* (§5.1)

New address, same connection ID, **path validated**, connection carries on.

```
 phone (Wi-Fi 10.0.0.5:5000) ──[CID=7f3a..]──▶ server
 phone (LTE 100.64.1.9:6123) ──[CID=7f3a..]──▶ server   "same connection, new path"
                                                        → validate path (challenge/response)
                                                        → continue streams, no new handshake
```

🧠 **Path validation** (PATH_CHALLENGE / PATH_RESPONSE frames) stops an attacker from pointing traffic at a victim's address by forging packets with a spoofed source.

**And the abuse case, handled in the same section:** a stable, visible ID is a **tracking identifier**. So IDs *"MUST NOT contain any information that can be used by an external observer to correlate them"*. Endpoints issue a **pool** of connection IDs and **rotate** them when the path changes.

> *"Identity belongs at the layer that owns the semantics. TCP made addressing and identity the same thing, and inherited every problem mobility causes."*

### 6.6 QPACK (RFC 9204): HPACK had its own head-of-line blocking

> *"If HPACK were used for HTTP/3, it would induce head-of-line blocking for field sections due to built-in assumptions of a total ordering across frames on all streams."* (RFC 9204)

**The problem QUIC created.** HPACK's dynamic table is **shared mutable state**. "Field number 67" only means something if you've already processed every insertion that made 67 mean it.
- On TCP that's free, because TCP gives a total ordering.
- QUIC deliberately **removed** exactly that guarantee. A header block on stream 12 could reference an entry inserted on stream 8 that hasn't arrived yet, so stream 12 has to stall. HOL blocking is back, inside the header compressor.

**QPACK's fix: take the mutable state off the request streams.**

```
  ┌─────────┐   encoder stream (unidirectional, type 0x02)   ┌─────────┐
  │ ENCODER │ ─────── "insert entry #67 = cookie: ..." ─────▶ │ DECODER │
  │ (sender)│                                                  │(receiver)
  │         │ ◀────── "I have seen these (up to #67)" ──────── │         │
  └─────────┘   decoder stream (unidirectional, type 0x03)   └─────────┘

  Request streams only REFERENCE entries.
  The encoder only references entries it knows are safe (acknowledged),
  unless the peer allows some streams to block.
```

| Setting | Default |
|---|---|
| `QPACK_BLOCKED_STREAMS` | **0** (🧠 the max number of streams allowed to wait for a not-yet-arrived table entry. 0 means "never make me block") |
| `QPACK_MAX_TABLE_CAPACITY` | **0** (🧠 0 means no dynamic table at all until the peer raises it) |

**Static table:** HPACK has **61 entries from 1**. QPACK has **99 from 0**, rebuilt on newer traffic, so the `sec-fetch` headers are in it.

> *"Shared mutable state and out-of-order delivery do not mix. Relax an ordering guarantee, then go and find what was relying on it."*

(Homework 4 asks what the encoder may do differently when `BLOCKED_STREAMS` is 0 versus 16. 🧠 Hint: with 0, it can only reference acknowledged entries or send literals. With 16, it may reference fresh entries and risk a short stall on up to 16 streams to get better compression.)

### 6.7 Discovery and fallback: how do you find HTTP/3, and what if UDP is blocked?

🧠 The problem: a browser typing `https://example.com` starts with TCP by default. How does it learn that the server also speaks QUIC on UDP?

| Mechanism | RFC · date | How it works | What it costs |
|---|---|---|---|
| **ALPN** | RFC 7301 · Jul 2014 | Protocol chosen inside the TLS handshake: "h2", "h3" | Nothing. But it needs a handshake first. |
| **Alt-Svc** | RFC 7838 · Apr 2016 | A response header: `h3=":443"`. The browser remembers. | One full TCP+TLS connection, every cold cache |
| **HTTPS DNS RR** | RFC 9460 · Nov 2023 | Record type 65 carries the ALPN list in the DNS answer | Nothing: you were doing the lookup anyway |

```
 Alt-Svc flow:
   1st visit:  TCP+TLS → GET / → 200 + "Alt-Svc: h3=\":443\"; ma=86400"
   next visit: try QUIC on UDP 443 directly

 HTTPS RR flow:
   DNS: example.com HTTPS?  → "1 . alpn=h3,h2"   → go straight to QUIC on first visit
```

**The numbers:**

| | |
|---|---|
| **3–5%** | of networks block all UDP traffic (RFC 9308 §2, quoting measurement studies) |
| **95.3%** | of Google's clients could use QUIC; **4.4%** blocked or had packet-size problems |
| **downgrade** | drop the UDP and wait for the client to fall back to TCP by itself |

> RFC 9308 §2: *"A fallback to TCP and TLS exposes control information to modification and manipulation in the network."* **Any protocol with a fallback path has, by construction, a downgrade attack.** The only question is what the fallback costs you.

🧠 Browsers usually race QUIC and TCP (like "Happy Eyeballs" for IPv4/IPv6) so a blocked UDP path costs little time. Homework 5 asks you to measure your own fallback timeout.

### 6.8 Adoption: where the web actually is

| Source | Measurement | Figure | Read it how? |
|---|---|---|---|
| HTTP Archive Web Almanac 2024 | By request, real crawl | HTTP/1.1 ≈15% · HTTP/2+ ≈85% | Cleanest number here |
| HTTP Archive Web Almanac 2024 | By homepage | h1 21–22% · h2 70–71% · h3 7–9% | Cleanest number here |
| HTTP Archive Web Almanac 2024 | Advertising h3 | Alt-Svc ≈26–28% · DNS HTTPS RR ≈9–10% | The DNS record is the laggard |
| W3Techs, 3 Sep 2026 | By website | HTTP/2 34.6% · HTTP/3 40.3% | **Treat with suspicion**: h3 > h2 means the categories aren't what they look like |

🧠 Why "h3 7–9%" by homepage but "40% HTTP/3" on W3Techs? W3Techs counts a site as "HTTP/3" if it *supports or advertises* it (e.g. it's behind Cloudflare). HTTP Archive counts which protocol a crawl *actually used*. Always ask what a metric counts.

> *"Fifteen per cent of requests are still HTTP/1.1, in 2026, eleven years after HTTP/2 shipped."* Internal services, embedded devices, that one load balancer nobody wants to touch, a thousand APIs behind a gateway. **You'll write HTTP/1.1 parsers in your career. There's no cutover. There's a very long tail**, and most engineering happens at the boundary between versions rather than inside one.

### 6.9 Four ideas (the lecture's closing)

| # | Idea | In one breath |
|---|---|---|
| 1 | **Multiplexing needs identity** | HTTP/1.1 had no request id, so order became the identifier, and ordering became mandatory. 31 bits fixed it. Same move as TCP ports, SS7 transaction ids, FastCGI request ids and QUIC stream ids. *When a client must wait before sending the next message, check whether you left the identifier out.* |
| 2 | **Every fix is the next problem** | Persistence made pipelining possible; pipelining created HOL blocking. HTTP/2 fixed it above TCP and exposed it inside TCP. QUIC fixed that and broke HPACK, which needed QPACK. *There's no final version, only a sequence of trades, and knowing which one you're standing on.* |
| 3 | **Semantics and encoding differ** | RFC 9110 is what HTTP *means*; 9112, 9113 and 9114 are three ways of writing it down. GET hasn't changed since 1996; the wire format has changed three times. *Every long-lived system needs this seam. Without it, a new encoding means rewriting your semantics.* |
| 4 | **Deployed reality beats specification** | 302 lost to browsers, so we got 307. Two connections lost to six, and the sentence was deleted. Postel's law lost to request smuggling. 418 was a joke that got reserved forever. Push was never deprecated and is dead. QUIC runs on UDP because middleboxes have a veto. |

### 6.10 Next session teaser (Session 6: "Where did the state go?")

Sessions and sticky routing · what "stateless" actually costs · TLS termination and where you put it · shared caches and the thundering herd · connection pools, which is amortise-the-setup for the sixth time.

**Bring this question:** *Today's connection can migrate across networks and survive a NAT rebind. Your session cookie cannot. Which of those two is the real identity of a user?*

**Lecture repo:** `http-evolution.tar.gz`, with 5 servers · 5 clients · 4 proxies · ~2,900 lines · no network required. `./setup.sh && ./run_all.sh` reproduces every number on every slide in about 40 seconds.

**Not covered, and worth your own time:** TLS beyond ALPN · HTTP/2 priorities and RFC 9218 · WebTransport and QUIC datagrams · MASQUE · RFC 9111, the whole of caching.

---

## 7 · HANDS-ON: TRY IT YOURSELF 🧠

These use standard tools, so they work even without the lecture repo. Replace `example.com` with any site, or use the repo's local servers on ports 8010/8011/8020.

### 7.1 Speak HTTP/1.0 by hand with `nc`

```bash
printf 'GET / HTTP/1.0\r\n\r\n' | nc example.com 80
```
**Expect:** a status line, headers, blank line, HTML, and then `nc` exits by itself because the server closed (HTTP/1.0 semantics). Many modern servers reply `HTTP/1.1 200 OK` even to a 1.0 request. That's allowed: the response version is the server's highest supported version, and it will still close.

### 7.2 See that HTTP/1.1 needs `Host`, and keeps the line open

```bash
printf 'GET / HTTP/1.1\r\n\r\n' | nc example.com 80          # expect 400 Bad Request
printf 'GET / HTTP/1.1\r\nHost: example.com\r\n\r\n' | nc example.com 80
```
**Expect:** the second one returns 200, and then `nc` **hangs** after the body. The server is keeping the connection open and waiting for your next request. That's persistence. Press Ctrl-C, or add `Connection: close`:

```bash
printf 'GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' | nc example.com 80
```

### 7.3 Two requests on one socket (keep-alive) and pipelining

```bash
printf 'GET / HTTP/1.1\r\nHost: example.com\r\n\r\nGET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' \
  | nc example.com 80 | grep -c '^HTTP/1.1'
```
**Expect:** `2`. Two responses from one TCP connection, sent pipelined (both requests at once).

### 7.4 Watch curl reuse a connection

```bash
curl -v http://example.com/ http://example.com/ 2>&1 | grep -Ei 're-?us|Connected to'
```
**Expect:** "Connected to …" once, then "Re-using existing connection" for the second URL.

### 7.5 Conditional requests (304)

```bash
curl -sI https://example.com/ | grep -Ei 'etag|last-modified'
curl -s -o /dev/null -w '%{http_code}\n' -H 'If-None-Match: "<paste-etag>"' https://example.com/
```
**Expect:** `304` if the ETag matches.

### 7.6 Range requests (206 / 416)

```bash
curl -s -r 0-99 -D - -o /dev/null https://example.com/     # 206, Content-Range: bytes 0-99/...
curl -s -H 'Range: bytes=99999999-' -D - -o /dev/null https://example.com/   # 416 (if server supports ranges)
```

### 7.7 Compression and `Vary`

```bash
curl -s -D - -o /dev/null https://example.com/ | grep -i -E 'content-length|vary'
curl -s -D - -o /dev/null -H 'Accept-Encoding: gzip' https://example.com/ | grep -i -E 'content-(length|encoding)|vary'
```
**Expect:** a smaller length with `Content-Encoding: gzip`, plus `Vary: Accept-Encoding`.

### 7.8 Chunked encoding, raw

```bash
curl -s --raw -H 'Accept-Encoding: identity' https://httpbin.org/stream/2 | head
```
**Expect:** hex chunk sizes on their own lines before each piece of JSON. (`--raw` stops curl from decoding the chunks.)

### 7.9 HTTP/2 and ALPN

```bash
curl -sv --http2 -o /dev/null https://example.com/ 2>&1 | grep -iE 'ALPN|HTTP/2'
openssl s_client -connect example.com:443 -alpn h2,http/1.1 </dev/null 2>/dev/null | grep -i alpn
```
**Expect:** `ALPN: server accepted h2` / `ALPN protocol: h2`. To see actual frames: `nghttp -nv https://example.com/` (from nghttp2) prints every SETTINGS/HEADERS/DATA frame with its stream ID.

### 7.10 HTTP/3 discovery

```bash
curl -sI https://cloudflare.com | grep -i alt-svc            # h3=":443"; ma=86400
dig +short cloudflare.com HTTPS                              # type 65 record with alpn="h3,h2"
curl -sv --http3 -o /dev/null https://cloudflare.com 2>&1 | grep -i 'HTTP/3'   # needs an h3-enabled curl
```

### 7.11 Watch the handshakes and cwnd

```bash
sudo tcpdump -i any -n 'tcp port 80 and (tcp[tcpflags] & (tcp-syn|tcp-fin) != 0)'
# in another terminal: run 7.1 several times vs. 7.3 once → count SYNs

ss -ti dst example.com    # (Linux) while a transfer is running: shows cwnd, rtt
```
**Expect:** one SYN per HTTP/1.0 request, and one SYN total for keep-alive.

### 7.12 Your own `talk.py` (a minimal stand-in for the lecture tool)

```python
#!/usr/bin/env python3
# usage: talk.py PORT "GET / HTTP/1.1" "Host: x" ""     (last "" = blank line)
import socket, sys, time
port, lines = int(sys.argv[1]), sys.argv[2:]
req = ("\r\n".join(lines) + "\r\n").encode()
s = socket.create_connection(("localhost", port))
t0 = time.time()
s.sendall(req); print(f"--- sent {len(req)} bytes\n")
s.settimeout(2)
try:
    while True:
        d = s.recv(65536)
        if not d:
            print(f"\n--- server closed the connection (FIN) at {(time.time()-t0)*1000:.1f} ms"); break
        sys.stdout.write(d.decode(errors="replace"))
except socket.timeout:
    print("\n--- no FIN after 2 s: connection still open (persistent)")
```

---

## 8 · ASSIGNMENT: "BUILD A CALCULATOR THAT STAYS ON THE LINE"

**Early HTTP/1.1 · due before Session 7.** Any language, no framework, **just a socket.**

### 8.1 The spec

| Request | Response |
|---|---|
| `GET /add?a=2&b=3` | `200` `5` |
| `GET /sub?a=10&b=4` | `200` `6` |
| `GET /mul?a=6&b=7` | `200` `42` |
| `GET /div?a=9&b=3` | `200` `3` |
| `GET /div?a=1&b=0` | `400` (divide by zero) |
| `GET /add?a=x&b=3` | `400` (not a number) |
| `GET /pow?a=2&b=8` | `404` (unknown operation) |
| `POST /add` | `405` (wrong method) |
| `GET /add` (no `Host`) | `400` (HTTP/1.1 requires Host) |

> *"That is the entire feature set. You could write it in an afternoon, and the arithmetic is not the point."*

### 8.2 How it'll be marked: one socket, every request

```python
s = socket.create_connection(("localhost", 8080))
#   GET /add?a=2&b=3     -> 200  5
#   GET /sub?a=10&b=4    -> 200  6
#   GET /mul?a=6&b=7     -> 200  42
#   GET /div?a=1&b=0     -> 400
#   GET /pow?a=2&b=8     -> 404
#   POST /add            -> 405
# socket still open: True
# 1 TCP handshake, 6 responses
```

> *"If the socket dies before I am done, you have written a 1996 server."*
> *"Don't read `02-http11/server11.py` until you have failed at least once."*

### 8.3 The part that's actually hard

Keeping the connection open forces a question HTTP/1.0 never asked: **where does this request end and the next one begin?** While you were hanging up, the answer was "at EOF", for free. Now you must consume **exactly `Content-Length` bytes and not one more**, because byte n+1 belongs to somebody else.

That applies **both ways**:
- **Your responses** must carry `Content-Length`, or the grader's client can't tell where your `5` ends.
- **Incoming requests** (e.g. `POST /add` with a body) must have their body consumed exactly, even though you reply 405. Otherwise the leftover body bytes get parsed as the *next* request line.

### 8.4 Stretch goals (all optional)

1. Honour `Connection: close` (respond, then close).
2. An **idle timeout** you can defend (🧠 e.g. 5–15 s; nginx's default `keepalive_timeout` is 75 s; justify it in terms of fds and memory per idle connection vs. the handshake cost).
3. **Chunked encoding** (accept chunked request bodies, or send chunked responses).
4. **Take all six at once and answer in order**, which is **pipelining**.

### 8.5 Approach and hints 🧠 (not a full solution)

**The core loop shape:**

```
 accept() → conn
 buf = b""
 loop:
     while b"\r\n\r\n" not in buf:          # need a full header block
         data = recv(); if not data: close & stop    # client hung up
         buf += data
     head, buf = split buf at first b"\r\n\r\n"      # KEEP the remainder in buf!
     parse request line + headers
     n = int(headers.get("content-length", 0))
     while len(buf) < n: buf += recv()      # read exactly n body bytes
     body, buf = buf[:n], buf[n:]           # leftover = start of NEXT request
     resp = handle(method, target, headers)
     send(resp)                             # with Content-Length!
     if headers says Connection: close: close & stop
```

**Key hints:**
- **Never assume one `recv()` = one request.** A single `recv()` may return half a request, or (with pipelining) three and a half requests. Keep a buffer that **persists across requests**. If you get this right, pipelining (stretch 4) comes free.
- **Header names are case-insensitive** (`host`, `Host`, `HOST`). Lowercase them when parsing.
- **Check order matters** for producing the expected codes. A sensible order: Host present? (400) → method is GET? (405) → path known? (404) → params valid numbers? (400) → b == 0 for div? (400) → 200. Think about which code `POST /pow` should get, and be able to defend it.
- **Parse the query string** by splitting the target on `?`, then `&`, then `=`. (Python's `urllib.parse.parse_qs` is fine. It's not a framework.)
- **Integer vs float:** `9/3` should print `3`, not `3.0`. Decide what `7/2` returns and document it.
- **Every response:** `HTTP/1.1 <code> <reason>\r\nContent-Length: <n>\r\nContent-Type: text/plain\r\n\r\n<body>`. Even a 400/404/405 needs a `Content-Length` (it can be 0). For a 405, 🧠 RFC 9110 says you should include `Allow: GET`.
- **Build the whole response in one buffer and do one `sendall()`**, and set `TCP_NODELAY`. Otherwise you'll hit the Nagle/delayed-ACK 40 ms stall from §4.4 on every request after the first.
- **Test like the grader:** open one socket in Python, send six requests one after another, read each response by parsing `Content-Length`, and finally check that the socket is still open (e.g. send one more request and confirm it gets an answer).
- **To handle multiple clients**, use a thread per connection, or `select`/`poll` from Lesson 1. A single-threaded server that blocks on one idle keep-alive client will starve everyone else. That's exactly why an idle timeout matters.

---

## 9 · COURSE PROJECT: "HTTP, IN BINARY · TWO TRACKS, ONE PROTOCOL"

**Now you write the spec.** Work in pairs: one person builds the server, the other the client, and **the only thing that crosses between you is the spec.**

| Track 1: the server `$ ./bserve ./www 9000` | Track 2: the client `$ ./bcurl -v localhost:9000/index.html` |
|---|---|
| accept a TCP connection | build the binary request frame |
| read one binary request frame | read the response, body to stdout |
| map the path to a file under a root | `-v` hexdumps every frame |
| reply: status, headers, the bytes | exit non-zero on 4xx / 5xx |
| 404 if it isn't there | **and never open a second connection** |
| 400 if the frame is malformed | |
| **and keep the connection open** | |

**The bit in the middle is the actual project:**
- **A fixed-size frame header.** You pick the fields and the widths, and you defend them. *HTTP/2 chose 24 / 8 / 8 / 31. Why?* (See §5.3.)
- **Headers:** number the ten names you actually send, length-prefix the rest. That's HPACK's first two mechanisms (static table + literals), in an evening.
- **One line you may not skip:** a receiver meeting a frame type it doesn't know **MUST skip it cleanly**. That's how you leave room for a version 2. (🧠 It works only because the length is in the fixed header: you can skip `Length` bytes without understanding them.)

**What you hand in:**
1. **The spec.** Two pages. Enough for a stranger.
2. **Your program.**
3. **An annotated hexdump** of one complete request and response. *If you can't annotate your own bytes, the spec isn't finished.*

> *"A client that only works against your own server is an implementation, not a protocol."*

**Hints 🧠:**
- Write down **byte order** (network byte order = big-endian, as HTTP/2 uses) and whether lengths are unsigned.
- Decide the **maximum frame size** and what happens when it's exceeded (400? close?).
- Include a **stream/request ID** even if you only do one request at a time. It costs a few bytes and lets you multiplex later (idea #1).
- Spec what happens on a **truncated frame** (the connection closes mid-header).
- Python's `struct.pack(">IBBI", ...)` style format strings make fixed headers easy. `xxd` or `hexdump -C` help with the annotated dump.
- Swap partners' programs with another pair as a real interop test.

---

## 10 · HOMEWORK (from the final slides)

| # | Do this | Until you can explain |
|---|---|---|
| 1 | Extend `fetch_page.py` to N connections. Plot wall time against N at 150 ms RTT for 6, 20 and 60 assets. Find the knee. | Why the knee is where it is, and it isn't "you ran out of connections". Read cwnd in `ss -ti`. |
| 2 | Make `server11.py` build its ETag from mtime, then modify a file twice within one second. | Which of RFC 9110 §8.8.1's two validators you just built, and why Range requests need the other one. |
| 3 | Decode a HEADERS frame's HPACK block **by hand** (three field lines, against RFC 7541 §6) before letting `hpack_mini.py` do it. | Why the second identical request costs 6 bytes. *You'll never forget HPACK after this.* |
| 4 | Set `SETTINGS_QPACK_BLOCKED_STREAMS` to 0, then to 16, in `h3_server.py`. | What the encoder may do differently in each case. *The hardest one here.* |
| 5 | `iptables -A OUTPUT -p udp --dport 4433 -j DROP`, then time the fallback. If you have `tc netem`, redo slide 41 (the HTTP/2-vs-HTTP/3 HOL comparison) with real loss on both sides. | What your own h3 timeout should be, and whether the caveat on slide 41 was too cautious, or not enough. |

All five are in the repo README. **Number three is the one to do if you only do one.**

🧠 Hints:
- **HW2**: the mtime ETag doesn't change on a second write within the same second, so the cache keeps stale content. A validator that doesn't change on every byte change is effectively **weak**.
- **HW3**: decode each first byte. `1xxxxxxx` = indexed field (7-bit index). `01xxxxxx` = literal with incremental indexing (6-bit name index). `0000xxxx` = literal without indexing. `0001xxxx` = never indexed. For strings, the first bit means Huffman (H) and the next 7 bits are the length.
- **HW5**: `tc qdisc add dev lo root netem loss 5%` adds real loss on loopback.

---

## 11 · SUMMARY CHEAT-SHEET

### Versions at a glance

| | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|---|
| Year / RFC | 1996 · 1945 | 1997 · 2068 → 2616 → 7230 → 9112 | 2015 · 7540 → 9113 | 2022 · 9114 (QUIC 9000, 2021) |
| Wire format | text | text | binary frames | binary frames on QUIC streams |
| Transport | TCP | TCP | TCP (+TLS in practice) | QUIC over UDP (TLS 1.3 built in) |
| Connections | 1 per request | persistent, 6/origin in browsers | 1 per origin | 1 per origin, migratable |
| Body framing | EOF / Content-Length | Content-Length / chunked | DATA frames, length in header | STREAM frames, offset + length |
| Request ID | none | none (order = id) | 31-bit stream id | QUIC stream id |
| Header compression | none | none | HPACK | QPACK |
| HOL blocking | n/a (serialisation cost) | yes (pipelining queue) | at the TCP layer | per-stream only |
| Cold setup (TLS 1.3) | 1 RTT (no TLS) | 2–3 RTT with TLS | 3 RTT incl. request | 2 RTT; 0-RTT on resume |

### Key terms

| Term | One-line meaning |
|---|---|
| **RFC** | Numbered, immutable IETF document; replaced by newer RFCs that "obsolete" it |
| **Framing** | How a receiver knows where a message ends: length or delimiter, nothing else |
| **Persistent connection / keep-alive** | Reusing one TCP connection for many requests; default in HTTP/1.1 |
| **`Connection: close`** | HTTP/1.1 opt-out: close after this response |
| **`Host`** | Header naming the website; enables virtual hosting; mandatory (400 if missing) in 1.1 |
| **SNI** | Hostname in the TLS ClientHello so the server can choose a certificate |
| **`Last-Modified` / `If-Modified-Since`** | Timestamp validator (1 s resolution, clock drift, mtime ≠ content) |
| **`ETag` / `If-None-Match`** | Content-fingerprint validator → 304 Not Modified |
| **Weak ETag `W/"…"`** | "Equivalent" not byte-identical; not usable for Range |
| **`If-Match`** | Conditional write → optimistic concurrency, prevents lost updates (412) |
| **`Vary`** | Tells caches which request headers change the response (cache key) |
| **`Range` / 206 / 416** | Partial content; 416 when the range is past the end |
| **Chunked encoding** | Hex-length-prefixed chunks ending in a 0-length chunk |
| **`Expect: 100-continue`** | Ask permission before sending a large body |
| **`no-cache` vs `no-store`** | Revalidate before use vs. never store |
| **Pipelining** | Sending multiple requests without waiting; responses must come back in order |
| **Head-of-line blocking** | Ready items stuck behind a slow item at the head of a queue |
| **Domain sharding** | Splitting assets across hostnames to get more connections (harmful in h2) |
| **302 vs 307 (vs 303)** | 302 = "Found" (browsers switch to GET); 307 = keep method; 303 = switch to GET explicitly |
| **Postel's law** | "Be liberal in what you accept", which turned out catastrophic across proxy chains |
| **Request smuggling** | CL vs TE disagreement makes two parsers see different request boundaries |
| **RFC 9110 vs 9112/9113/9114** | Semantics (shared) vs. three wire formats |
| **418** | The teapot joke, reserved forever because everyone implemented it |
| **Nagle + delayed ACK** | 40 ms stall on write-write-read patterns; fix: one write + `TCP_NODELAY` |
| **SPDY** | Google's 2009 experiment that became HTTP/2 |
| **Connection preface** | `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`, 24 bytes, trips up 1.1 servers with 501 |
| **ALPN** | TLS extension that negotiates `h2`/`h3`/`http/1.1` with no extra RTT |
| **h2c** | Cleartext HTTP/2; the Upgrade dance is deprecated, prior knowledge only |
| **Frame header** | 9 bytes: Length 24 · Type 8 · Flags 8 · R 1 · Stream ID 31 |
| **Stream IDs** | Odd = client, even = server, 0 = connection control |
| **SETTINGS defaults** | table 4096 · window 65535 · max frame 16384 · concurrent streams unlimited (≥100 advised) |
| **RST_STREAM / GOAWAY** | Cancel one stream / graceful connection drain |
| **WINDOW_UPDATE** | Per-stream and per-connection flow control |
| **HPACK** | Static table (61) + dynamic table (62+) + fixed Huffman |
| **`0x82`** | HPACK encoding of `:method: GET` (static index 2) |
| **CRIME** | Compression-size side channel that leaks secrets; why no gzip for headers |
| **Server push** | Still in the RFCs, removed by browsers; replaced by 103 Early Hints |
| **TCP HOL blocking** | One lost TCP segment stalls all HTTP/2 streams behind it |
| **QUIC** | Encrypted, multi-stream, userspace transport over UDP (RFC 9000) |
| **Ossification** | Middleboxes block anything unfamiliar; why QUIC uses UDP and encrypts headers |
| **1-RTT / 0-RTT** | QUIC's combined handshake; 0-RTT sends data on resume but can be replayed |
| **425 Too Early** | Origin refuses replayable 0-RTT data |
| **Connection ID** | QUIC connection identity independent of IP/port → migration; rotated for privacy |
| **QPACK** | HPACK for HTTP/3; table updates on separate encoder (0x02) / decoder (0x03) streams |
| **Alt-Svc / HTTPS RR** | Ways to advertise h3: a response header vs. DNS record type 65 |
| **Downgrade** | Blocking UDP forces TCP fallback; any fallback path is a downgrade attack |

### Key numbers

`9.6 s` (24 objects, HTTP/1.0, 200 ms RTT) · `2.8%` (gzip ratio) · `6` / `18` connections · `2003 ms vs 3 ms` (HOL) · `97.5%` repeated headers · `40 ms` delayed ACK · `24`-byte preface · `9`-byte frame header · `174 → 79 → 6` (HPACK) · `61` / `99` static entries · `3 RTT vs 2 RTT vs 0-RTT` · `3–5%` block UDP · `~15%` of requests still HTTP/1.1 · `62` assigned status codes.

---

## 12 · SELF-CHECK QUESTIONS

**1. What does a `recv()` return when you send a second request on an HTTP/1.0 socket, and why?**
<details><summary>Answer</summary>

`b''`, an empty read, which means EOF. HTTP/1.0 servers close the connection after every response (RFC 1945 §1.3), so the socket is already half-closed by the server's FIN. It's not an error, just the end of the stream.
</details>

**2. Is it accurate to say HTTP/1.0 suffers from head-of-line blocking? Explain.**
<details><summary>Answer</summary>

Not quite. HOL blocking needs a queue, and HTTP/1.0 has no pipelining, so nothing ever queues behind anything else. What it has is a **serialisation cost**: one round trip (plus a handshake) per object. Real HOL blocking appears in HTTP/1.1 when pipelining queues responses behind a slow one.
</details>

**3. Why did adding a single `Host` header matter so much, and where does the same idea reappear?**
<details><summary>Answer</summary>

Without it the only "address" was the TCP 4-tuple, so one IP could serve only one site. ~1.1 billion sites would need about a third of IPv4. `Host` puts the name in the message, enabling virtual hosting. The same move reappears as **SNI** in TLS (because Host is encrypted), as **`:authority`** in HTTP/2 (compressed pseudo-header), and as topics, routing keys and ingress host rules elsewhere.
</details>

**4. Give three problems with `Last-Modified` as a cache validator, and say how `ETag` fixes them.**
<details><summary>Answer</summary>

(1) 1-second resolution: two writes in one second are indistinguishable. (2) Clock drift between client, server and proxies. (3) mtime isn't identity: a rebuild changes mtimes without changing bytes. An ETag computed from the content (e.g. a SHA-256 prefix) changes if and only if the bytes change, and it needs no clock.
</details>

**5. Decode this HTTP/2 frame header: `00 01 07 00 00 00 00 00 01`.**
<details><summary>Answer</summary>

Length `0x000107` = 263 bytes; Type `0x00` = DATA; Flags `0x00` = none; R=0, Stream ID = 1 (the first client-initiated stream). It's the 263-byte HTML body for the request on stream 1.
</details>

**6. Explain the CL.TE request-smuggling example from the lecture. Why can't it happen in HTTP/2?**
<details><summary>Answer</summary>

The message has both `Content-Length: 6` and `Transfer-Encoding: chunked`. A front end reading Content-Length sees one POST with body `0\r\n\r\nG`. A back end reading chunked sees an empty body (`0\r\n\r\n`), then a second request `GET /admin`, which the front end never inspected. It also leaves an extra response on the shared connection for the next user. In HTTP/2 the length lives in a fixed 24-bit field of every frame header, so there's only one possible reading and no "second opinion" to disagree with.
</details>

**7. Why does keep-alive with `write(head); write(body)` produce ~40 ms delays on every request except the first, and why didn't HTTP/1.0 see it?**
<details><summary>Answer</summary>

Nagle holds the small second write until the first is ACKed. The client has nothing to send yet (it's waiting for the body), so delayed ACK waits up to 40 ms before sending the ACK. That's a 40 ms mutual wait. HTTP/1.0 calls `close()` immediately after writing, which flushes the pending data, so it never hit the stall. Fix: coalesce into one write and set `TCP_NODELAY`.
</details>

**8. Why does HTTP/2 want exactly one connection per origin, making domain sharding harmful?**
<details><summary>Answer</summary>

HPACK's dynamic table is per connection. Every extra connection starts with an empty table (so headers get resent in full), plus a cold congestion window and its own TLS handshake. Multiplexing already gives parallelism on one connection, so sharding only adds cost.
</details>

**9. Why not simply gzip HTTP headers? What does HPACK do differently?**
<details><summary>Answer</summary>

Because of **CRIME**: with adaptive compression, attacker-controlled text and a secret (cookie) share a dictionary. A correct guess makes the compressed output smaller, and since encryption doesn't hide length, the attacker learns the secret byte by byte. HPACK uses a **fixed** Huffman table (no adaptation), a dynamic table that only matches **whole** fields exactly (prefix guesses win nothing), and lets sensitive fields be marked "never indexed".
</details>

**10. The lecturer's first HTTP/2 server produced exactly HTTP/1.1's timings. Why, and what's the general lesson?**
<details><summary>Answer</summary>

Requests were served inline in the frame-reading loop, so the 2-second `/big.txt` handler blocked processing of every other stream. The protocol allowed concurrency but the code serialised the work. The fix was a thread per stream plus a lock around the shared connection. Lesson: a protocol can only give you permission. The same trap shows up as a blocking call in a coroutine, a global mutex in a gRPC handler, or a pool sharing one DB connection.
</details>

**11. HTTP/2 multiplexes streams. Why does a single lost TCP packet still stall unrelated streams, and how does QUIC fix it?**
<details><summary>Answer</summary>

TCP delivers one ordered byte stream. Bytes after a gap can't be handed to the application until the gap is retransmitted, even if they belong to other HTTP/2 streams and have already arrived in the receive buffer. TCP can't see streams. QUIC makes streams a transport concept: each STREAM frame carries a stream ID and offset, ordering is enforced **within** a stream only, so a loss stalls only the stream that lost data.
</details>

**12. Why is QUIC built on UDP instead of a new IP protocol number, and why does it encrypt most of its header?**
<details><summary>Answer</summary>

NATs, firewalls, CGNATs and proxies drop unknown protocol numbers (which is why SCTP never worked across the internet), but UDP already passes. UDP is "a hole that already exists". QUIC encrypts packet numbers and most header fields so middleboxes can't observe and then depend on them. That prevents ossification (what happened to TCP options) and keeps the protocol changeable. Living in userspace also lets it ship with apps instead of waiting a decade for kernel upgrades.
</details>

**13. What's the danger of 0-RTT, and what's the standard mitigation?**
<details><summary>Answer</summary>

0-RTT early data can be **replayed**: an attacker captures it and resends it, possibly to a different server in the farm whose single-use ticket cache never saw it. RFC 9001 admits the protections are "imperfect" and makes the application protocol responsible. In practice, allow only idempotent requests (e.g. Cloudflare allows only GET/HEAD), mark early requests with `Early-Data: 1`, and let the origin reject with **425 Too Early**.
</details>

**14. Why couldn't HTTP/3 just reuse HPACK, and how does QPACK solve it?**
<details><summary>Answer</summary>

HPACK's dynamic table is shared mutable state that assumes every insertion is processed in a total order. QUIC removes ordering between streams, so a header block on one stream might reference an entry inserted on another stream that hasn't arrived, which is HOL blocking again. QPACK moves table insertions onto a dedicated **encoder stream** (0x02) with acknowledgements on a **decoder stream** (0x03). The encoder only references entries it knows the decoder has, unless the peer permits some blocking (`QPACK_BLOCKED_STREAMS`, default 0).
</details>

**15. Server push is still in RFC 9113 and RFC 9114. Why is it effectively dead, and what replaced it?**
<details><summary>Answer</summary>

The server doesn't know what's already in the client's cache, so it often pushes files the client already has, wasting bandwidth and competing with needed resources for the same congestion window. Usage was 1.25% (later 0.7%) of HTTP/2 sites, and Chrome 106 (Sep 2022) removed it, citing no clear net gain. Replacement: **103 Early Hints**. The server *informs* ("you'll want style.css") and the client decides using its own cache state.
</details>
