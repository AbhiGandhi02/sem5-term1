# 📓 Evolution of HTTP, 1996 → now
### *CN at Scaler — Lesson 5 (Week 5)* · "Same words. Three completely different ways of writing them down."

> **RFC 1945 · RFC 2068 · RFC 2616 · RFC 7230 · RFC 9110 · RFC 9113 · RFC 9000 · RFC 9114**

```
  $ printf 'GET / HTTP/1.0\r\n\r\n' | nc localhost 8010
```

### Agenda: four wire formats

| Hour | Version | The RFC | Ends on |
|---|---|---|---|
| 1 | HTTP/1.0 | RFC 1945 · May 1996 | One response per connection |
| 2 | Early HTTP/1.1 | RFC 2068 · Jan 1997 | "At most 2 connections" |
| 3 | HTTP/1.1 now | RFC 2616 → 7230 → 9110 | Head-of-line blocking |
| 4 | HTTP/2 | RFC 7540 → 9113 | TCP head-of-line blocking |
| 5 | QUIC + HTTP/3 | RFC 9000 · RFC 9114 | The 3–5% who block UDP |

> *"The protocol did not drift — it was pushed."*

---

## 1 · HTTP/1.0 · RFC 1945 · MAY 1996

### 1.1 The whole protocol fits on one slide

| Methods | Headers (16) | Status codes |
|---|---|---|
| `GET`, `HEAD`, `POST` | `Allow`, `Authorization`, `Content-Encoding`, `Content-Length`, `Content-Type`, `Date`, `Expires`, `From`, `If-Modified-Since`, `Last-Modified`, `Location`, `Pragma`, `Referer`, `Server`, `User-Agent`, `WWW-Authenticate` | `200 OK`, `201 Created`, `202 Accepted`, `204 No Content` · `300 Multiple Choices`, `301 Moved Permanently`, `302 Moved Temporarily`, `304 Not Modified` · `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found` · `500 Internal Server Error`, `501 Not Implemented`, `502 Bad Gateway`, `503 Service Unavailable` |
| `PUT`, `DELETE`, `LINK`, `UNLINK`: Appendix D, "additional methods" that exist but are not in the spec | | **Today: 62 assigned** (IANA HTTP Status Code Registry, updated 2025-09-15) |

### 1.2 Hands-on: speak it by hand, once

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

- The **blank line** separates headers from body. It is the delimiter.
- The server **closed the connection (FIN)** right after the response.

> *"Not curl. curl adds four headers you did not write and hides two the server sent."*

### 1.3 The limitation: one response, then the socket dies

RFC 1945 §1.3:
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

Nothing. **Not an error — an empty read.** The connection was closed the moment the first response finished.

**The cost (a 24-object page, 200 ms RTT):**

```
  24 TCP handshakes            24 × 200 ms = 4.8 s
  24 request/response          24 × 200 ms = 4.8 s
                                             ──────
                                              9.6 s
  ...and 24 congestion windows that each start from nothing.
```

### 1.4 The limitation: no `Host` header. One IP, one website.

```
$ talk.py 8010 --req "GET / HTTP/1.0" "Host: site-b.example" ""

HTTP/1.0 200 OK
<h1>site-a</h1>                 <- not site-b. it never had a choice.

# server log:
   (ignoring Host: site-b.example - HTTP/1.0 has no vhosts)
```

- **1:1** sites per IP address, until January 1997.
- **~1.1 B** websites today. Without Host, that would need **a third of all of IPv4**.
- **SNI**: the same problem one layer down, once TLS hid the Host (§2.2).

> *"The fix — a single header field — arrives nine months later and is arguably the highest-leverage line in the whole standard."*

### 1.5 What it did have: caching, with a clock as the fingerprint

```
$ talk.py 8010 --req "HEAD /style.css HTTP/1.0" ""
Last-Modified: Fri, 04 Sep 2026 07:11:14 GMT

$ talk.py 8010 --req "GET /style.css HTTP/1.0" \
        "If-Modified-Since: Fri, 04 Sep 2026 07:11:14 GMT" ""

HTTP/1.0 304 Not Modified
                                <- no body. 202,000 bytes -> 0.
```

Spec references: RFC 1945 §10.7 `Expires`, §10.9 `If-Modified-Since`, §10.10 `Last-Modified`.

**Three things wrong with a timestamp:**

| Problem | Why |
|---|---|
| **1 s resolution** | No sub-second field. Two writes in one second are *one* write to every cache alive. |
| **Clock drift** | Your clock, my clock, the proxy's clock. The spec has to make rules for when they disagree. |
| **mtime ≠ identity** | A rebuild changes every mtime and none of the bytes. |

### 1.6 Closing: what HTTP/1.0 could not do

| The limitation | Concretely | Fixed in |
|---|---|---|
| Connection is not persistent | One request per TCP handshake, forever | RFC 2068 §8.1 |
| Sixteen headers, three methods | No `OPTIONS`: you cannot ask what a server supports | RFC 2068 §9.2 |
| Caching exists but is thin | `Expires` + `If-Modified-Since`. A clock, not a fingerprint | RFC 2068 §14.19 |
| No `Host` | One IP address per website | RFC 2068 §14.23 |

> **⚠️ The lecture's correction:** HTTP/1.0 is usually listed as having "head-of-line blocking". **It doesn't, quite.** HOL blocking needs a *queue*, and HTTP/1.0 has no pipelining to queue anything on. What it has is a **serialisation cost**: one round trip per object, no exceptions.
>
> **Real** HOL blocking arrives in HTTP/1.1, as a side effect of the fix for this. *"Every solution today is the cause of the next slide."*

---

## 2 · EARLY HTTP/1.1 · RFC 2068 · JANUARY 1997

### 2.1 Persistence became the default. Now you opt out.

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

### 2.2 `Host`: one header, one IP address, a thousand sites (RFC 2068 §14.23)

```
$ talk.py 8011 --req "GET / HTTP/1.1" ""                # no Host
HTTP/1.1 400 Bad Request
HTTP/1.1 requires a Host header (RFC 2616 14.23).

$ ... "Host: site-a.local"      ->   <h1>site-a</h1>
$ ... "Host: site-b.local"      ->   <h1>site-b</h1>
                                     same IP. same port. same socket.
```

**The same move, four layers, thirty years:**

| Layer | The problem | The fix |
|---|---|---|
| HTTP/1.0 | The four-tuple was the address; one IP = one site | `Host:`, which puts the name in the message |
| TLS | Host is encrypted; the cert must be chosen first | **SNI** in the ClientHello, in the clear |
| HTTP/2 | Headers are compressed and unordered | `:authority` as a pseudo-header |
| Everywhere else | Many logical destinations, one connection | Topic. Routing key. Ingress host rule. |

### 2.3 `ETag`: a fingerprint, not a clock (RFC 2068 §14.19)

```
$ talk.py 8011 --req "HEAD /big.txt HTTP/1.1" "Host: site-a.local" ""
ETag: "55274ac22c651b90"                  <- sha256 of the bytes, first 16 hex

$ talk.py 8011 --req "GET /big.txt HTTP/1.1" \
         "Host: site-a.local" "If-None-Match: \"55274ac22c651b90\"" ""

HTTP/1.1 304 Not Modified
ETag: "55274ac22c651b90"
--- received 256 bytes
```

| Concept | Meaning |
|---|---|
| **202,000 → 0** | bytes of body. The 200 would have carried the whole file; the 304 carries nothing. |
| **`W/"…"`** | A **weak validator**: *equivalent*, not identical. **Range requests need a strong one.** (RFC 9110 §8.8.1, strong vs weak) |
| **`If-Match`** | The same idea pointed at **writes**: optimistic concurrency, and the fix for **lost updates** (RFC 9110 §13.1.1). |
| **`If-None-Match`** | RFC 9110 §13.1.2 |

### 2.4 `Accept-*` and `Vary`: the header that keeps caches honest (§14.3, §14.44)

```
$ 02-http11/demos.sh                # demo 7
   Accept-Encoding: identity  -> Content-Length: 202000   Vary: Accept-Encoding
   Accept-Encoding: gzip      -> Content-Length: 5763     Content-Encoding: gzip
   Accept-Encoding: deflate   -> Content-Length: 5751     Content-Encoding: deflate
```

gzip = **2.8% of its original size**.

> *"Which quietly undercuts 'HTTP/1.1 is slow because it is text'. Text compresses. The body was never the problem — the headers were, because Content-Encoding compresses the body only."*

**`Vary` is a cache-key instruction.** It says: *this URL has more than one answer, and here is what the answer depends on.*

| `Vary` value | Effect |
|---|---|
| `Vary: Accept-Encoding` | correct |
| `Vary: User-Agent` | millions of keys, hit rate → 0 |
| `Vary: *` | caching off entirely |

### 2.5 `Range`: the header that runs your data lake (§14.35)

```
Range: bytes=100-199     ->   HTTP/1.1 206 Partial Content
                              Content-Range: bytes 100-199/202000
                              Content-Length: 100

Range: bytes=-50         ->   Content-Range: bytes 201950-201999/202000
                              the last 50 bytes, size not known in advance

Range: bytes=999999-     ->   HTTP/1.1 416 Range Not Satisfiable
                              Content-Range: bytes */202000
```

**What one header bought:**

| Use | How |
|---|---|
| **Video scrubbing** | The player asks for the bytes around the timestamp you clicked, not the film. |
| **Resumable downloads** | Died at 60%? `Range: bytes=600000000-` and carry on where you stopped. |
| **Parquet / ZIP footers** | The directory is at the **end**. Read the last few KB, then fetch only the columns you need. |

> *416 was not in RFC 2068. RFC 2616 added it to separate "your syntax is wrong" from "that file is smaller than you think".*

### 2.6 Chunked transfer encoding: send a body without knowing how long it is (§3.6.1)

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

> **Framing is length or delimiter — there is no third option.**
> **Chunked encoding is both, nested:** each chunk is **length-prefixed**, and the sequence is **delimited** by an empty one. That's FastCGI's record structure from Session 3, exactly. Same shape, different decade. It's also what **streaming JSON, server-sent events and gRPC trailers** all stand on.

### 2.7 And the rest of it, in one breath

| Arrival | What it is for | Where you meet it now |
|---|---|---|
| `OPTIONS` | Ask what a server supports | Every **CORS preflight** your browser sends |
| `PUT` · `DELETE` | Write and remove, not just read | The R and D of every REST API since |
| `TRACE` | Echo my request back, proxies and all | Disabled everywhere: **Cross-Site Tracing**, 2003 |
| `Via` | Each proxy appends itself | Working out which box in the chain is lying |
| `Expect: 100-continue` | Ask before uploading four gigabytes | curl's mysterious 1-second pause on big PUTs |
| `Cache-Control` | A vocabulary instead of one timestamp | `max-age`, `s-maxage`, `no-cache` ≠ `no-store` |
| ~40 status codes | 303, 405, 409, 410, 412, 413, 415, 504… | 504 is here already, not a 1999 addition |

> **⚠️ Classic gotcha:** `no-cache` means **revalidate before use**. `no-store` means **do not write it down**. *"Everybody gets this wrong exactly once."*

### 2.8 "SHOULD NOT maintain more than 2 connections" (§8.1.4)

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

| Number | Meaning |
|---|---|
| **6** | connections per origin, what every browser actually shipped by about 2008 |
| **18** | via **domain sharding** (img1, img2, img3). *"A hack on top of a violated rule."* |
| **dropped** | RFC 7230 removed the sentence in 2014. *"The spec conceded."* |

> *"The best row in that table is the one nobody shipped."* That row is **pipelining**: sending all requests without waiting for each response.

---

## 3 · HTTP/1.1 TODAY · RFC 2616 (1999) → 7230 (2014) → 9110 (2022)

### 3.1 1999: mostly tightening, not adding

RFC 2616 (June 1999). **Genuinely new** (§19.6.3, "Changes from RFC 2068"):

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
 RFC 2068 said:   302 keeps the method.
 Reality:         every browser turned the POST into a GET anyway.
 RFC 2616 could have said "no, really".
 Instead it:      renamed 302 to "Found",
                  wrote down what implementations do,
                  added 307 for people who meant what 302 originally said.
```

> *"It is easier to add a number than to fix a billion deployed clients."*

### 3.2 Framing: watch Postel's law lose an argument

| RFC 2616 §4.4 · June 1999 | RFC 9112 §6.3 · June 2022 |
|---|---|
| *be liberal in what you accept* | *the same situation, 23 years later* |
| "If a message is received with both a Transfer-Encoding header field and a Content-Length header field, the latter MUST be ignored." | "A server MAY reject a request that contains both … or process such a request in accordance with the Transfer-Encoding alone. Regardless, the server MUST close the connection after responding." |
| **Recover. Do not reject.** In 1999 this was simply what good engineers did. | **Rejection is now allowed. Recovery is now optional. And either way, destroy the connection.** |

> *"Be liberal in what you accept is a fine rule for two programs. It is a catastrophic rule for a chain of five."*

### 3.3 Request smuggling: one byte stream, two readings, one attack

> 💡 **Simple words mein:** Ek parcel pe do label hain. Gate ka guard ek label padh ke "ek hi parcel hai" bolta hai, andar wala clerk doosra label padh ke usme se ek chhupa hua doosra parcel nikaal leta hai. Guard ne woh kabhi check hi nahi kiya.

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
- **With `Content-Length: 6`:** ONE request. Its 6-byte body is `0\r\n\r\nG`.
- **With `Transfer-Encoding: chunked`:** TWO requests. The second is `GET /admin`.

> *"Put a CDN that does the first in front of an origin that does the second, and you have posted a request the CDN never saw — and left a spare response in the connection for whoever is routed onto it next."*

```
$ python3 03-limits/framing.py --port 8011
    this server answered with 2 response(s):
     | HTTP/1.1 200 OK
     | HTTP/1.1 404 Not Found           <- the request nobody sent
```

> **HTTP/2 makes this unrepresentable:** the length is in the frame header, and there's no second opinion to disagree with.

### 3.4 RFC 7230–7235 (June 2014): one document became six

| RFC | Title | Who has to read it carefully |
|---|---|---|
| 7230 | Message Syntax and Routing | Proxies, load balancers, CDNs, and the `https` scheme lives here now |
| 7231 | Semantics and Content | Application frameworks: methods, status codes, representations |
| 7232 | Conditional Requests | Anything with an ETag: browsers, caches, object stores |
| 7233 | Range Requests | Video, downloads, Parquet readers |
| 7234 | Caching | Every CDN on earth |
| 7235 | Authentication | The `WWW-Authenticate` / `Authorization` framework |

> The lecturer jokingly calls it **"Microservices for RFC"**. One large artefact with many stakeholders becomes several with clear ownership, versioned independently, so a change to caching doesn't need re-ratifying the definition of GET. Same trade-off too: 2616 was one place to look, and **everyone still says "2616" when they mean HTTP/1.1.**

### 3.5 Why your JWT is base64 (RFC 7230 §3.2.6)

```
tchar = "!" / "#" / "$" / "%" / "&" / "'" / "*" / "+" / "-" / "." /
        "^" / "_" / "`" / "|" / "~" / DIGIT / ALPHA
token = 1*tchar
excluded:   " ( ) , / : ; < = > ? @ [ \ ] { }
```

**Correction: this isn't a 2014 rule.** RFC 7230 only wrote it as a list. The rule is from **RFC 2616 §2.2 (1999)**, where token = any CHAR except CTLs or separators, and the separators list already contains `{` and `}`. Same rule, better prose, fifteen years earlier.

**And the JWT reason is broader.** A JWT must survive an `Authorization` header, a URL query string, a cookie, a form field and a JSON body. **base64url** also swaps `+` and `/` for `-` and `_` so it's URL-safe. And the signature is **raw bytes**, which need encoding whatever the grammar says.

> *"Every layer you pass through has an alphabet. The intersection of all of them is about sixty-four characters wide — which is exactly why base64 exists, and why Session 2 met the same problem in MIME (mail)."*

### 3.6 RFC 9110–9114 (June 2022): the split that matters, and a teapot

| RFC | What it is | Version-specific? |
|---|---|---|
| **9110** | HTTP Semantics: methods, status, headers, ranges, auth | **No.** Shared by 1.1, 2 and 3. |
| **9111** | HTTP Caching | **No.** Shared by all three. |
| 9112 | HTTP/1.1 | Wire format only |
| 9113 | HTTP/2 | Wire format only |
| 9114 | HTTP/3 | Wire format only |

**The table of contents is the argument.** Everything HTTP *means* is in 9110 and 9111, shared by all three versions. 9112, 9113 and 9114 only describe how to write it down on a socket. **Three encodings, one language.**

**418 (Unused):** an April Fool's joke (RFC 2324, 1998) that *"has been widely implemented as an easter egg, and therefore is effectively consumed by this use."* Reserved forever, not because it's useful, but because too many people shipped the joke. (RFC 9110 §15.5.19 · `draft-nottingham-thanks-larry` · IANA registry, 2025-09-15: **62 assigned codes**.)

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

> 💡 **Simple words mein:** HTTP/1.1 = bina token wali chai stall: jo pehle order kiya, use pehle milega, chahe uski chai late bane. HTTP/2 = har order pe token number (stream ID), jiski chai ready, uska number bulao.

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

| | |
|---|---|
| **805 B** | of headers on one realistic request: cookies, `User-Agent`, `Accept-*`, `sec-fetch-*` |
| **upload** | the direction you have least of, spent at the start of the page, while the user watches |
| **why not gzip?** | the obvious answer, and the wrong one (see CRIME, §5.6) |

### 4.4 A bug the lecturer hit: when keep-alive made it slower (Nagle + delayed ACK)

**Two algorithms, both correct:**
- **Nagle (RFC 896, 1984):** do not send a small segment while an earlier one is unacknowledged.
- **Delayed ACK (RFC 1122):** do not acknowledge immediately; wait up to 40 ms to piggyback it.

```
 write(head)   -> goes out, nothing unacked
 write(body)   -> Nagle holds it
 client        -> nothing to send, ACK delayed
   ... 40 ms ...
 ACK, then body.          Every request but the 1st.
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

It only appears on a **persistent** connection: `close()` pushes it out. *"The fix for one problem exposed it."*

> *"Your protocol sits on a protocol with its own opinions. nginx and Apache both coalesce and both set TCP_NODELAY — now you know why."*

### 4.5 Closing: the bill for HTTP/1.1

| The wall | Measured today | Fixed by |
|---|---|---|
| Head-of-line blocking | 2003 ms vs 3 ms for the same file, pipelined | Stream ids: HTTP/2 |
| Header bloat | 97.5% of uploaded header bytes are repeats | HPACK: HTTP/2 |
| Too many connections | 6 per origin, 18 with domain sharding | One connection: HTTP/2 |
| Ambiguous framing | One byte stream, two readings, one attack | Frame headers: HTTP/2 |
| TCP itself | 3 round trips before the first byte | QUIC: HTTP/3 |

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

> **Naming:** it's called **HTTP/2, not HTTP/2.0**, on purpose. The working group dropped the minor version because "1.1" had produced a decade of confusion about what a 1.x server must support. There's no HTTP/2.1 and there won't be one. The next version is a new number and a new wire format.

### 5.2 The connection preface: twenty-four bytes that are also a joke (RFC 9113 §3.4)

```
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
50 52 49 20 2a 20 48 54 54 50 2f 32 2e 30 0d 0a
0d 0a 53 4d 0d 0a 0d 0a                      exactly 24 octets, always
```

An HTTP/1.1 server that receives this sees method `PRI`, version `HTTP/2.0`, and **fails cleanly with 501** instead of trying to parse the binary frames behind it as text. (`PRI…SM` is what's left of "PRISM". It was 2013. The RFC doesn't say so.)

| Over TLS: how it actually happens (RFC 9113 §3.2 · ALPN is RFC 7301) | Cleartext: the dance that died (RFC 9113 §3.3 and §3.1) |
|---|---|
| `$ h2_client.py --port 8443 --tls` → `TLS TLSv1.3  ALPN negotiated: 'h2'` | `$ h2_client.py --port 8020` → `h2c by prior knowledge` |
| The client lists `h2` and `http/1.1` inside the **ClientHello**. The server picks one in the **ServerHello**. **Zero extra round trips**: it rides in a handshake you already paid for. | RFC 7540 had an `Upgrade: h2c` dance. RFC 9113 §3.1: it *"was never widely deployed and is deprecated by this document."* **No browser ever shipped cleartext HTTP/2 at all.** |

### 5.3 The frame header: nine bytes, and 31 bits of them are the answer (§4.1)

```
 | Length (24) | Type (8) | Flags (8) | R | Stream Identifier (31) |
 |                 Frame Payload (0 … Length)                     |
                                       └─ 31 bits that HTTP/1.1 never had
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

**Stream ID rules (§5.1.1):** **client streams are odd**, **server streams are even**, so nobody has to coordinate · **stream 0 is the connection itself**: SETTINGS, PING, GOAWAY.

> **Request smuggling isn't "fixed" here. It's unrepresentable:** the length is in a fixed place, at a fixed size, with no second opinion.

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

**§6.5.2 defaults:** `HEADER_TABLE_SIZE` **4096** · `INITIAL_WINDOW_SIZE` **65535** · `MAX_FRAME_SIZE` **16384** · `MAX_CONCURRENT_STREAMS` has **none** (*"no smaller than 100"*).

### 5.5 HPACK (RFC 7541): 174 bytes → 79 → 6

> 💡 **Simple words mein:** Roz wala customer chai stall pe bas "bhaiya, wahi usual" bolta hai, poora order dobara nahi batata. HPACK mein "usual" = table ka index number.

**Three mechanisms, none of them clever:**

1. **Static table: 61 entries.** Fixed, identical everywhere (Appendix A). Index 2 is `:method: GET`, name **and** value. The whole of `GET` costs **0x82**. *Not compressed. Indexed.*
2. **Dynamic table: from index 62.** Anything sent before **on THIS connection** gets an index. Your 300-byte cookie costs 300 bytes once, then 1–2 bytes forever.
3. **Huffman: Appendix B.** A canonical code built from real header text: "e" is 5 bits, "X" is 8. Per string, and only when it actually shrinks it.

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

> **The dynamic table is per connection**, which is why HTTP/2 wants **ONE**. **Domain sharding was clever under 1.1 and is actively harmful here.**

### 5.6 CRIME (2012): why not just gzip the headers?

**Because compressed size leaks the secret.**

> 💡 **Simple words mein:** Locked dabba khol nahi sakte, par uska wazan dekh sakte ho. Har sahi guess pe dabba thoda halka ho jaata hai, toh dheere dheere poora password pata chal jaata hai.

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

**HPACK's defence is structural:**
- The **Huffman table is FIXED**: built once from a corpus, published in Appendix B, never adapted to the data flowing through it. There's no shared adaptive dictionary for attacker text and secret text to meet in.
- The **dynamic table is an exact-match index, not a substring matcher**. Guessing a *prefix* of your cookie wins nothing.
- **RFC 7541 §7.1** goes further: mark sensitive fields **"never indexed"**.

> **Yours too:** gzip a JSON response containing a user-supplied search term and an API key together, and you've rebuilt CRIME.

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

**And the dynamic table warming up in real time:**

```
-> HEADERS stream=1   GET /big.txt?delay=2.0   84B of text -> 30B hpack
-> HEADERS stream=3   GET /style.css           76B of text -> 12B hpack
-> HEADERS stream=5   GET /app.js              73B of text -> 11B hpack
-> HEADERS stream=7   GET /logo.png            75B of text -> 12B hpack
```

> *"Same GET. Same 200. Same ETag. Our h2 handler is line-for-line the shape of the HTTP/1.1 one — that is RFC 9110 doing its job."*

### 5.8 The bug the lecturer shipped first: a protocol can only give you permission

**Handling requests inline in the read loop (what was written first):**

```python
for event in conn.receive_data(data):
    if isinstance(event, RequestReceived):
        self.serve(event.stream_id, ...)
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

> **A protocol can only give you permission.** You'll meet this exact shape again: a worker pool sharing one database connection · a blocking call inside a coroutine · a gRPC handler holding a global mutex · six goroutines waiting on one channel. *The transport is fine. You serialised the work yourself.*

### 5.9 Server push: nobody deprecated it, everybody removed it

| What the RFC says (still standard, today) | What implementers did (Chrome 106, 27 September 2022) |
|---|---|
| Retained in RFC 9113 §8.4. Retained in RFC 9114 §4.6 (HTTP/3). | **1.25%** of HTTP/2 sites used it. Later **0.7%**. |
| Search RFC 9113 for "deprecat" and you get exactly two hits, neither of them push: RFC 7540 **PRIORITY** (§2, §5.3) and the **h2c Upgrade token** (§3.1). | *"no clear net performance gain and in many cases performance regressions"* |
| If you've heard "the RFC deprecated server push": **it didn't.** | **The fundamental problem: the server doesn't know what's in your cache.** So it pushes a file you already have, spending bandwidth AND competing with what you actually needed, on the same connection and congestion window. |

**The replacement: 103 Early Hints (RFC 8297).** Don't push the file. Say "you're going to want style.css". The client checks its own cache and decides.

> *"The server keeps the knowledge; the client keeps the decision. When one side has the knowledge and the other has the state, **inform — do not act.**"*

### 5.10 The one thing HTTP/2 could not fix: TCP head-of-line blocking

> *"because the parallel nature of HTTP/2's multiplexing is not visible to TCP's loss recovery mechanisms, a lost or reordered packet causes all active transactions to experience a stall regardless of whether that transaction was directly impacted by the lost packet."* (RFC 9114 §1.1)

> 💡 **Simple words mein:** Saare parcels ek hi courier truck mein. Ek parcel gum hua toh truck ruk gaya, aur baaki sab ke parcels bhi atak gaye, jabki unka kuch nahi khoya.

```
$ h2_client.py --port 9020 --multiplex          # through tools/stall_tcp.py
   stall: holding the stream at byte 3000 for 300 ms

   run 1   logo 4.5   style 4.7  | app 306.3   hero 307.0   icon 307.3   big 308.9
   run 2   style 4.0  icon 5.2   | logo 308.2  big 308.3    hero 308.4   app 308.4
   run 3   style 3.5             | big 306.4   app 306.5    icon 306.5   logo 306.7  hero 306.8
           ^ through before        ^ everything behind the hole, whether or not
             the hole                it lost a single byte
```

The later bytes had **arrived**. They were in the receive buffer. The kernel may not hand them over without breaking the in-order guarantee that is the reason you chose TCP. And **which** streams get caught changes every run: the blast radius is "whoever had bytes behind the gap", and you don't get to choose.

> Google's SPDY data: **11.81% speedup at 0% loss, 47.7% at 2% loss.** Multiplexing helps most exactly where TCP hurts most.

---

## 6 · QUIC + HTTP/3 · RFC 9000 · RFC 9114

### 6.1 Why UDP: because the internet stopped accepting new protocols

**The obvious answer, and why it fails:**

```
 TCP = 6    UDP = 17    SCTP = 132
 "Just take protocol number 253 and build a proper transport."
```

Except that every NAT, firewall, CGNAT, corporate proxy and hotel router on earth **drops what it doesn't recognise**. **SCTP** is a good protocol, standardised in 2000, and you can't use it across the public internet. That's why.

> *"UDP is not a good foundation. It is a hole that already exists."*

**And so: encrypt almost everything.**

> *"middleboxes could observe traits of a specific version of QUIC and assume that when other versions exhibit similar traits the same underlying semantic is being expressed."* (RFC 8999 §7)

Anything a middlebox **can** read, it'll eventually **depend on**, and then you can never change it. That's how **TCP options became unusable**. QUIC encrypts packet numbers and most of the header, **not only for privacy, but to keep the protocol changeable.**

**And it lives in userspace.** A TCP change is a kernel upgrade on a billion machines and a decade. A QUIC change ships with your app.

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

**RFC 9001 §2.1:** there is no transport handshake and then a crypto handshake. **There is one handshake.** And **TLS is not optional: there is no unencrypted QUIC.**

> *"Layering is how we make systems comprehensible — and every layer boundary is a place you may pay a round trip."*

### 6.3 0-RTT (RFC 9001 §4.6): zero round trips, and the price of them

```
$ python3 05-http3/handshake_race.py --rtt 150
                                                measured    round trips
   TCP + TLS 1.3 + h2 (new connection)          456.6 ms       3.0
   QUIC + HTTP/3 (new connection, 1-RTT)        319.8 ms       2.1
   QUIC + HTTP/3 (0-RTT, request in flight 1)   161.7 ms       1.1

   Cold connection: QUIC saves 137 ms.     Resumed: 0-RTT saves 295 ms.
```

**The price: replay attacks.**

> RFC 9001 §9.2: *"use of TLS early data comes with an exposure to replay attack… Endpoints MUST implement and use the replay protections described in [TLS13], however it is recognized that these protections are imperfect."*

**Imperfect because** the defence is a single-use ticket cache, and a server farm has many servers. An attacker who captured your early data sends it to a **different** one.

> 💡 **Simple words mein:** Cheque ki photocopy leke doosri bank branch mein jao; us branch ko pata hi nahi ki ye pehle hi cash ho chuka hai. Isliye 0-RTT sirf "dobara ho toh koi nuksaan nahi" wale (idempotent) requests ke liye.

**So the rule is yours to make:**

> *"An application protocol that uses QUIC MUST describe how the protocol uses 0-RTT and the measures that are employed to protect against replay attack."*

**In practice: idempotent requests only.** GET is fine; a POST that charges a card isn't. **Cloudflare allows only GET and HEAD**, and sets an `Early-Data` header so your origin can refuse with **425 Too Early**.

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

> **⚠️ Be precise about this experiment:** the TCP side *delays* a chunk (the shape a lost segment makes). The QUIC side *really deletes* 18 datagrams. Different perturbations, so **don't read 307 vs 78 as a benchmark.** What *is* comparable is the **blast radius**: on TCP it's "whoever had bytes behind the gap", a different set every run. On QUIC it's **exactly one stream, every time.**

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

> 💡 **Simple words mein:** TCP mein connection = tumhara ghar ka address; ghar badla toh sab khatam. QUIC mein connection = tumhara mobile number (connection ID); ghar badlo, number wahi rehta hai.

**And the abuse case, handled in the same section:** a stable, visible ID is a **tracking identifier**. So IDs *"MUST NOT contain any information that can be used by an external observer to correlate them"*. Endpoints issue a **pool** and **rotate**.

> *"Identity belongs at the layer that owns the semantics. TCP made addressing and identity the same thing, and inherited every problem mobility causes."*

### 6.6 QPACK (RFC 9204): HPACK had its own head-of-line blocking

> *"If HPACK were used for HTTP/3, it would induce head-of-line blocking for field sections due to built-in assumptions of a total ordering across frames on all streams."* (RFC 9204)

**The problem QUIC created.** HPACK's dynamic table is **shared mutable state**. "Field number 67" only means something if you've already processed every insertion that made 67 mean it.
- On TCP that's free: total ordering.
- QUIC deliberately **removed** exactly that guarantee. A block on stream 12 could reference an entry inserted on stream 8 that hasn't arrived. So: stall stream 12.

**QPACK's fix: take the mutable state off the request streams.**
- **Encoder stream (type 0x02):** encoder → decoder, table insertions.
- **Decoder stream (type 0x03):** decoder → encoder, "I have seen these".
- The encoder only references entries it **knows are safe**.

| Setting | Default |
|---|---|
| `QPACK_BLOCKED_STREAMS` | **0** |
| `QPACK_MAX_TABLE_CAPACITY` | **0** |

**Static table:** HPACK has **61 entries from 1**. QPACK has **99 from 0**, rebuilt on newer traffic, so the `sec-fetch` headers are in it.

> *"Shared mutable state and out-of-order delivery do not mix. Relax an ordering guarantee, then go and find what was relying on it."*

### 6.7 Discovery and fallback: how do you find it, and what if UDP is blocked?

| Mechanism | RFC · date | How it works | What it costs |
|---|---|---|---|
| **ALPN** | RFC 7301 · Jul 2014 | Protocol chosen inside the TLS handshake: "h2", "h3" | Nothing. But it needs a handshake first. |
| **Alt-Svc** | RFC 7838 · Apr 2016 | A response header: `h3=":443"`. The browser remembers. | One full TCP+TLS connection, every cold cache |
| **HTTPS DNS RR** | RFC 9460 · Nov 2023 | Record type 65 carries the ALPN list in the DNS answer | Nothing: you were doing the lookup anyway |

| | |
|---|---|
| **3–5%** | of networks block all UDP traffic (RFC 9308 §2, quoting measurement studies) |
| **95.3%** | of Google's clients could use QUIC; **4.4%** blocked or had packet-size problems |
| **downgrade** | drop the UDP and wait for the client to fall back to TCP by itself |

> RFC 9308 §2: *"A fallback to TCP and TLS exposes control information to modification and manipulation in the network."* **Any protocol with a fallback path has, by construction, a downgrade attack.** The only question is what the fallback costs you.

---

## 7 · CLOSING: ADOPTION AND FOUR IDEAS

### 7.1 Adoption: where the web actually is

| Source | Measurement | Figure | Read it how? |
|---|---|---|---|
| HTTP Archive Web Almanac 2024 | By request, real crawl | HTTP/1.1 ≈15% · HTTP/2+ ≈85% | Cleanest number here |
| HTTP Archive Web Almanac 2024 | By homepage | h1 21–22% · h2 70–71% · h3 7–9% | Cleanest number here |
| HTTP Archive Web Almanac 2024 | Advertising h3 | Alt-Svc ≈26–28% · DNS HTTPS RR ≈9–10% | The DNS record is the laggard |
| W3Techs, 3 Sep 2026 | By website | HTTP/2 34.6% · HTTP/3 40.3% | **Treat with suspicion**: h3 > h2 means the categories aren't what they look like |

> *"Fifteen per cent of requests are still HTTP/1.1, in 2026, eleven years after HTTP/2 shipped."* Internal services, embedded devices, that one load balancer nobody wants to touch, a thousand APIs behind a gateway. **You'll write HTTP/1.1 parsers in your career. There's no cutover. There's a very long tail**, and most engineering happens at the boundary between versions rather than inside one.

### 7.2 Four ideas

| # | Idea | In one breath |
|---|---|---|
| 1 | **Multiplexing needs identity** | HTTP/1.1 had no request id, so order became the identifier, and ordering became mandatory. 31 bits fixed it. Same move as TCP ports, SS7 transaction ids, FastCGI request ids and QUIC stream ids. *When a client must wait before sending the next message, check whether you left the identifier out.* |
| 2 | **Every fix is the next problem** | Persistence made pipelining possible; pipelining created HOL blocking. HTTP/2 fixed it above TCP and exposed it inside TCP. QUIC fixed that and broke HPACK, which needed QPACK. *There's no final version, only a sequence of trades, and knowing which one you're standing on.* |
| 3 | **Semantics and encoding differ** | RFC 9110 is what HTTP *means*; 9112, 9113 and 9114 are three ways of writing it down. GET hasn't changed since 1996; the wire format has changed three times. *Every long-lived system needs this seam. Without it, a new encoding means rewriting your semantics.* |
| 4 | **Deployed reality beats specification** | 302 lost to browsers, so we got 307. Two connections lost to six, and the sentence was deleted. Postel's law lost to request smuggling. 418 was a joke that got reserved forever. Push was never deprecated and is dead. QUIC runs on UDP because middleboxes have a veto. |

---

## 8 · ASSIGNMENT: "BUILD A CALCULATOR THAT STAYS ON THE LINE"

**Early HTTP/1.1 · due before Session 7.** Any language, no framework, **just a socket.**

### 8.1 The task

| Request | Response |
|---|---|
| `GET /add?a=2&b=3` | `200` `5` |
| `GET /sub?a=10&b=4` | `200` `6` |
| `GET /mul?a=6&b=7` | `200` `42` |
| `GET /div?a=9&b=3` | `200` `3` |
| `GET /div?a=1&b=0` | `400` |
| `GET /add?a=x&b=3` | `400` |
| `GET /pow?a=2&b=8` | `404` |
| `POST /add` | `405` |
| `GET /add` (no `Host`) | `400` |

**How it'll be marked: one socket, every request.**

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

**The part that's actually hard:** keeping the connection open forces a question HTTP/1.0 never asked: **where does this request end and the next one begin?** While you were hanging up, the answer was "at EOF", for free. Now you must consume **exactly `Content-Length` bytes and not one more**, because byte n+1 belongs to somebody else.

**Stretch, all optional:** honour `Connection: close` · an idle timeout you can defend · chunked encoding · take all six at once and answer in order (pipelining).

### 8.2 ✅ Solution (`calc.py`, Python stdlib only)

Run `python3 calc.py`, then run the test client below.

```python
import socket, threading
from urllib.parse import urlsplit, parse_qs

OPS = {"add": lambda a, b: a + b, "sub": lambda a, b: a - b,
       "mul": lambda a, b: a * b, "div": lambda a, b: a / b}
REASON = {200: "OK", 400: "Bad Request", 404: "Not Found", 405: "Method Not Allowed"}

def num(s):
    try:
        return int(s)
    except ValueError:
        return float(s)                      # "x" -> ValueError -> 400

def handle(method, target, headers):
    if "host" not in headers:
        return 400, "Host header required"
    if method != "GET":
        return 405, "only GET"
    url = urlsplit(target)
    op = OPS.get(url.path.lstrip("/"))
    if op is None:
        return 404, "unknown operation"
    q = parse_qs(url.query)
    try:
        r = op(num(q["a"][0]), num(q["b"][0]))
    except (KeyError, ValueError, ZeroDivisionError):
        return 400, "bad input"
    if isinstance(r, float) and r.is_integer():
        r = int(r)                           # 9/3 -> "3", 7/2 -> "3.5"
    return 200, str(r)

def serve(conn):
    conn.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    conn.settimeout(15)                      # idle timeout for keep-alive
    buf = b""                                # survives across requests
    try:
        while True:
            while b"\r\n\r\n" not in buf:    # wait for a full header block
                data = conn.recv(4096)
                if not data:
                    return                   # client hung up
                buf += data
            head, buf = buf.split(b"\r\n\r\n", 1)
            lines = head.decode("latin-1").split("\r\n")
            method, target, _ = lines[0].split(" ")
            headers = {}
            for line in lines[1:]:
                k, _, v = line.partition(":")
                headers[k.strip().lower()] = v.strip()
            n = int(headers.get("content-length", 0))
            while len(buf) < n:              # consume the body EXACTLY (even for 405)
                data = conn.recv(4096)
                if not data:
                    return
                buf += data
            buf = buf[n:]                    # leftover = start of the next request

            code, body = handle(method, target, headers)
            close = headers.get("connection", "").lower() == "close"
            resp = (f"HTTP/1.1 {code} {REASON[code]}\r\n"
                    f"Content-Type: text/plain\r\n"
                    f"Content-Length: {len(body)}\r\n"
                    + ("Allow: GET\r\n" if code == 405 else "")
                    + ("Connection: close\r\n" if close else "")
                    + "\r\n" + body)
            conn.sendall(resp.encode())      # ONE write (no Nagle stall, §4.4)
            if close:
                return
    except (OSError, ValueError):            # timeout, reset, or malformed request
        pass
    finally:
        conn.close()

srv = socket.create_server(("", 8080))
while True:
    c, _ = srv.accept()
    threading.Thread(target=serve, args=(c,), daemon=True).start()
```

**Why it passes:**
- One `buf` lives for the whole connection, so half a request or several pipelined requests in one `recv()` both work. That gives you **pipelining (stretch 4) for free**: requests are answered one by one, in order.
- Every response has `Content-Length`, so the grader knows where `5` ends.
- The `POST /add` body is read and thrown away before replying 405, so it isn't mistaken for the next request line.
- Check order: no Host → 400, not GET → 405, unknown op → 404, bad/missing number or divide by zero → 400.
- Stretch goals 1 and 2 are done: `Connection: close` is honoured, and an idle client is dropped after 15 s so it can't hold a thread and a socket forever.

**Test client (same as the grader):**

```python
import socket
s = socket.create_connection(("localhost", 8080))
f = s.makefile("rb")

def ask(method, path, body=b""):
    s.sendall(f"{method} {path} HTTP/1.1\r\nHost: x\r\n"
              f"Content-Length: {len(body)}\r\n\r\n".encode() + body)
    status = f.readline().split()[1].decode()
    n = 0
    while (line := f.readline()) != b"\r\n":          # read headers
        k, _, v = line.decode().partition(":")
        if k.lower() == "content-length":
            n = int(v)
    out = f.read(n).decode()                          # read EXACTLY n bytes
    print(f"  {method} {path:<16} -> {status}  {out if status == '200' else ''}")

for p in ["/add?a=2&b=3", "/sub?a=10&b=4", "/mul?a=6&b=7", "/div?a=1&b=0", "/pow?a=2&b=8"]:
    ask("GET", p)
ask("POST", "/add", b"a=1&b=2")
ask("GET", "/div?a=9&b=3")                            # answered => socket still open
```

Output (tested):

```
  GET /add?a=2&b=3     -> 200  5
  GET /sub?a=10&b=4    -> 200  6
  GET /mul?a=6&b=7     -> 200  42
  GET /div?a=1&b=0     -> 400
  GET /pow?a=2&b=8     -> 404
  POST /add             -> 405
  GET /div?a=9&b=3     -> 200  3
```

---

## 9 · COURSE PROJECT: "HTTP, IN BINARY · TWO TRACKS, ONE PROTOCOL"

**Now you write the spec.** In pairs: one server, one client, and **the only thing that crosses between you is the spec.**

| Track 1: the server `$ ./bserve ./www 9000` | Track 2: the client `$ ./bcurl -v localhost:9000/index.html` |
|---|---|
| accept a TCP connection | build the binary request frame |
| read one binary request frame | read the response, body to stdout |
| map the path to a file under a root | `-v` hexdumps every frame |
| reply: status, headers, the bytes | exit non-zero on 4xx / 5xx |
| 404 if it isn't there | **and never open a second connection** |
| 400 if the frame is malformed | |
| **and keep the connection open** | |

**The bit in the middle is the actual project:** a **fixed-size frame header** (you pick and defend the fields and widths; HTTP/2 chose 24 / 8 / 8 / 31) · **headers:** number the ten names you actually send, length-prefix the rest · a receiver meeting an **unknown frame type MUST skip it cleanly**.

**Hand in:** (1) the spec, two pages, enough for a stranger · (2) your program · (3) an annotated hexdump of one complete request and response.

> *"A client that only works against your own server is an implementation, not a protocol."*

### 9.1 ✅ Solution: minimal working skeleton

**Spec (v1, all integers big-endian):**

| Field | Bytes | Why this width |
|---|---|---|
| Length | 4 (u32) | payload size; anything > 16 MB is malformed → 400 |
| Type | 1 | `0x01` REQUEST (payload = path, UTF-8), `0x02` RESPONSE (payload = u16 status + body). Any other type: read `Length` bytes and skip. |
| Flags | 1 | reserved, send 0 (room for END_STREAM-style flags later) |
| Stream ID | 2 (u16) | the request ID: the response copies it, so multiplexing is possible later |

**`proto.py`** (shared by both tracks):

```python
import struct
HDR = struct.Struct(">IBBH")            # length u32 | type u8 | flags u8 | stream u16 = 8 bytes
REQUEST, RESPONSE = 0x01, 0x02
MAX_LEN = 16 * 1024 * 1024

def recv_exact(sock, n):
    buf = b""
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise EOFError                # peer closed (maybe mid-frame)
        buf += chunk
    return buf

def send_frame(sock, ftype, sid, payload, verbose=False):
    frame = HDR.pack(len(payload), ftype, 0, sid) + payload
    if verbose: print(">>", frame.hex(" "))
    sock.sendall(frame)

def recv_frame(sock, verbose=False):
    while True:
        raw = recv_exact(sock, HDR.size)
        length, ftype, flags, sid = HDR.unpack(raw)
        if length > MAX_LEN:
            raise ValueError("frame too large")
        payload = recv_exact(sock, length)
        if verbose: print("<<", (raw + payload).hex(" "))
        if ftype in (REQUEST, RESPONSE):
            return ftype, sid, payload
        # unknown type: its bytes are already consumed -> skip it cleanly
```

**`bserve.py`** (Track 1): `python3 bserve.py ./www 9000`

```python
import os, socket, struct, sys
from proto import *

root, port = os.path.realpath(sys.argv[1]), int(sys.argv[2])
srv = socket.create_server(("", port))
while True:
    conn, _ = srv.accept()
    with conn:
        try:
            while True:                                   # keep the connection open
                ftype, sid, payload = recv_frame(conn)
                if ftype != REQUEST:
                    raise ValueError("expected REQUEST")
                path = os.path.realpath(os.path.join(root, payload.decode().lstrip("/")))
                if path.startswith(root + os.sep) and os.path.isfile(path):
                    status, body = 200, open(path, "rb").read()
                else:
                    status, body = 404, b""
                send_frame(conn, RESPONSE, sid, struct.pack(">H", status) + body)
        except EOFError:
            pass                                          # client closed
        except ValueError:                                # malformed frame
            send_frame(conn, RESPONSE, 0, struct.pack(">H", 400))
```

**`bcurl.py`** (Track 2): `python3 bcurl.py -v localhost:9000/index.html`

```python
import socket, struct, sys
from proto import *

args = sys.argv[1:]
verbose = "-v" in args
host_port, _, path = [a for a in args if a != "-v"][0].partition("/")
host, port = host_port.split(":")
s = socket.create_connection((host, int(port)))           # the ONLY connection
send_frame(s, REQUEST, 1, ("/" + path).encode(), verbose)
_, sid, payload = recv_frame(s, verbose)
status = struct.unpack(">H", payload[:2])[0]
sys.stdout.buffer.write(payload[2:])
sys.exit(0 if status < 400 else 1)
```

**Annotated hexdump (real output, `www/index.html` = `<h1>hi</h1>\n`):**

```
>> 00 00 00 0b | 01 | 00 | 00 01 | 2f 69 6e 64 65 78 2e 68 74 6d 6c
   len=11       REQ  flg  sid=1    "/index.html"
<< 00 00 00 0e | 02 | 00 | 00 01 | 00 c8 | 3c 68 31 3e 68 69 3c 2f 68 31 3e 0a
   len=14       RESP flg  sid=1    200     "<h1>hi</h1>\n"
```

**Key steps to finish it:**
1. **Headers:** add a header block at the start of the payload: 1 byte count, then per header 1 byte name index (1–10 for your ten common names) or `0` + length-prefixed name, then a length-prefixed value.
2. **Truncated frame:** `recv_exact` raises `EOFError`, so the connection just closes. Write this rule in the spec.
3. Swap programs with another pair: if their client works with your server, it's a protocol.

---

## 10 · HOMEWORK (from the final slides)

### HW1 · Extend `fetch_page.py` to N connections; plot wall time vs N at 150 ms RTT for 6, 20 and 60 assets. Find the knee.

**✅ Solution:** give the script a pool of N keep-alive connections and hand the assets out round-robin. Then loop `for N in 1 2 3 4 6 8 10 20 30 60` and note the wall time.

Expected shape: round trips ≈ **1 (handshakes, in parallel) + ⌈assets / N⌉**, so wall ≈ 150 ms × (1 + ⌈A/N⌉).

| Assets | N=1 | N=2 | N=6 | N=10 | N=20 | N=60 |
|---|---|---|---|---|---|---|
| 6 | 1050 ms | 600 | 300 | 300 | 300 | 300 |
| 20 | 3150 | 1650 | 750 | 450 | 300 | 300 |
| 60 | 9150 | 4650 | 1650 | 1050 | 600 | 300 |

**The knee:** the curve is a hyperbola (A/N). Most of the gain comes from the first few connections. After that, saving one more round trip needs *doubling* N. Meanwhile, each new connection costs its own handshake and starts with its own small congestion window. In `ss -ti` a fresh connection shows a small `cwnd` (≈10) that never gets to grow, while one reused connection's `cwnd` keeps growing. So the knee comes from diminishing returns plus cold windows, not from "running out of connections".

### HW2 · Build the ETag from mtime in `server11.py`, then modify a file twice within one second.

**✅ Solution:**

```python
etag = '"%d"' % int(os.stat(path).st_mtime)        # whole seconds = 1 s resolution
```

```bash
# both writes in the same second (one command line):
echo v1 > f.txt; curl -sI localhost:8011/f.txt | grep -i etag; echo v2 > f.txt   # ETag: "1757000000"
curl -s -o /dev/null -w '%{http_code}\n' \
     -H 'If-None-Match: "1757000000"' localhost:8011/f.txt                       # 304 -> stale v1 kept!
```

**What to write:** you built a **weak** validator (RFC 9110 §8.8.1): it does not change on every change of the bytes, so the cache keeps v1. **Range requests need a strong validator** (one that changes whenever any byte changes, e.g. a hash of the bytes), because otherwise byte ranges from two different versions could be stitched into one corrupt file.

### HW3 · Decode a HEADERS frame's HPACK block by hand (RFC 7541 §6), then check with `hpack_mini.py`.

**✅ Solution** (worked on RFC 7541's own example C.3, no Huffman). Read the **first bits of each byte**: `1xxxxxxx` = indexed field (7-bit index) · `01xxxxxx` = literal, add to dynamic table (6-bit name index) · then a string = 1 bit Huffman flag + 7-bit length + bytes.

```
Request 1:  82 86 84 41 0f 77 77 77 2e 65 78 61 6d 70 6c 65 2e 63 6f 6d
  82 = 1|0000010 -> static #2  :method: GET
  86 = 1|0000110 -> static #6  :scheme: http
  84 = 1|0000100 -> static #4  :path: /
  41 = 01|000001 -> literal, name = static #1 (:authority), ADD to dynamic table
  0f = H=0, length 15 -> "www.example.com"           => becomes dynamic #62

Request 2:  82 86 84 be 58 08 6e 6f 2d 63 61 63 68 65
  82 86 84       -> same three static entries
  be = 1|0111110 -> index 62 = :authority: www.example.com    (1 byte, not 17)
  58 = 01|011000 -> literal, name = static #24 (cache-control), add to table
  08 -> length 8 -> "no-cache"
```

**Why the second identical request costs 6 bytes:** after the first request, every one of the six fields is in the static or dynamic table, so each one is sent as a **single one-byte index**: 6 fields = 6 bytes.

### HW4 · Set `SETTINGS_QPACK_BLOCKED_STREAMS` to 0, then 16, in `h3_server.py`.

**✅ Solution:**
- **0 (default):** the encoder may only reference dynamic-table entries the decoder has **already acknowledged** on the decoder stream. A brand-new header must go as a literal (or static reference) until the ack arrives. No stream can ever be blocked, but compression is worse at the start.
- **16:** the encoder may insert an entry and **reference it right away**. Up to 16 request streams may stall waiting for that insertion to arrive on the encoder stream. Better compression, with a small risk of HOL blocking inside QPACK.
- Note: with `QPACK_MAX_TABLE_CAPACITY` left at its default **0** there is no dynamic table at all, so raise it too, or you will see no difference.

### HW5 · Block UDP, time the fallback; redo slide 41 (HTTP/2 vs HTTP/3 HOL) with real loss.

**✅ Solution:**

```bash
sudo iptables -A OUTPUT -p udp --dport 4433 -j DROP     # block QUIC to the lab server
time <your h3 client> ...                               # note how long until it gives up / uses TCP
sudo iptables -D OUTPUT -p udp --dport 4433 -j DROP     # undo

sudo tc qdisc add dev lo root netem loss 5%             # real loss for BOTH TCP and QUIC
# rerun the h2 --multiplex client and the h3 client, compare per-stream times
sudo tc qdisc del dev lo root                           # undo
```

**What to write:** waiting for the full QUIC handshake timeout before trying TCP is far too slow. A good h3 timeout is short, about a few RTTs (a few hundred ms), or better, start TCP in parallel and use whichever connects first. With real loss on both sides the times become comparable, and the slide's main point still holds: on TCP a loss stalls every stream behind the gap, on QUIC only the stream that lost data.

---

## 11 · SUMMARY CHEAT-SHEET

| Version | RFC | Wire format | Key fix | What it broke / left |
|---|---|---|---|---|
| HTTP/1.0 | 1945 · May 1996 | text | — | one response per connection, no Host, clock-based caching |
| Early HTTP/1.1 | 2068 · Jan 1997 | text | persistence by default, Host, ETag, Vary, Range, chunked | pipelining → HOL blocking; "at most 2 connections" |
| HTTP/1.1 today | 2616 → 7230 → 9110/9112 | text | tightening: 307, 416, Host MUST-400 | no request id, header bloat, ambiguous framing (smuggling) |
| HTTP/2 | 7540 → 9113 (+ HPACK 7541) | binary frames, 31-bit stream id | multiplexing, HPACK | TCP head-of-line blocking |
| HTTP/3 | 9114 on QUIC 9000 | QUIC streams over UDP | one handshake, 0-RTT, per-stream loss, connection IDs | HPACK → QPACK; 3–5% block UDP → fallback/downgrade |

| Term | One-line meaning |
|---|---|
| **Framing** | Length or delimiter. Chunked = both, nested. |
| **`Connection: close`** | HTTP/1.1 opt-out from persistence |
| **`Host` / SNI / `:authority`** | The name in the message: HTTP/1.1, TLS, HTTP/2 |
| **`ETag` / `If-None-Match` → 304** | Fingerprint validator; `W/` = weak, Range needs strong |
| **`If-Match`** | Optimistic concurrency, fixes lost updates |
| **`Vary`** | Cache-key instruction (`User-Agent` → hit rate 0, `*` → no caching) |
| **`Range` → 206 / 416** | Partial content; 416 added by RFC 2616 |
| **`no-cache` vs `no-store`** | Revalidate before use vs. do not write it down |
| **302 / 307** | 302 renamed "Found" (browsers turned POST into GET); 307 keeps the method |
| **Request smuggling** | CL vs TE: one byte stream, two readings |
| **Nagle + delayed ACK** | 40 ms stall on persistent connections; fix: one write + `TCP_NODELAY` |
| **Preface** | `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`, 24 bytes, 1.1 servers fail with 501 |
| **ALPN** | Picks h2/h3 inside the TLS handshake, zero extra RTT |
| **Frame header** | 9 bytes: Length 24 · Type 8 · Flags 8 · R 1 · Stream ID 31 |
| **Stream IDs** | Odd = client, even = server, 0 = connection |
| **SETTINGS defaults** | 4096 · 65535 · 16384 · no max-streams default ("no smaller than 100") |
| **HPACK** | Static 61 + dynamic 62+ (per connection) + fixed Huffman; `GET` = `0x82` |
| **CRIME** | Compressed size leaks secrets; why no gzip for headers |
| **Server push** | Still in RFCs, removed by browsers; replaced by 103 Early Hints |
| **0-RTT** | Request in the first flight; replayable → idempotent only, 425 Too Early |
| **Connection ID** | QUIC identity ≠ address; survives Wi-Fi → LTE; pool + rotate |
| **QPACK** | Encoder stream 0x02, decoder stream 0x03; static 99 from 0; defaults 0/0 |
| **Alt-Svc / HTTPS RR** | Advertise h3 by response header (costs one TCP+TLS) vs DNS type 65 |

**Key numbers:** `9.6 s` (24 objects, HTTP/1.0, 200 ms RTT) · `2.8%` gzip · `6` / `18` connections · `2003 ms vs 3 ms` HOL · `97.5%` repeated headers · `40 ms` delayed ACK · `24`-byte preface · `9`-byte frame header · `174 → 79 → 6` HPACK · `61` / `99` static entries · `3 vs 2 vs 1 RTT` (TCP+TLS 1.3 / QUIC / 0-RTT; TLS 1.2 = 4) · `3–5%` block UDP · `~15%` of requests still HTTP/1.1 · `62` assigned status codes.

---

## 12 · MCQs (mid → advanced)

**Q1.** A page has 10 objects and the RTT is 100 ms. Using the lecture's round-trip model (HTTP/1.0 close = handshake + request per object; keep-alive = 1 handshake + one request at a time), how much time does HTTP/1.1 keep-alive on one connection save over HTTP/1.0?
- A) 1.0 s
- B) 0.9 s
- C) 1.1 s
- D) 0.1 s

> ✅ **Answer: B.** HTTP/1.0 needs 10 × 2 = 20 RTT = 2.0 s; keep-alive needs 1 + 10 = 11 RTT = 1.1 s, so it saves 0.9 s. 1.1 s is the keep-alive total, not the saving.

**Q2.** Six requests are pipelined on one HTTP/1.1 connection at t=0. The first is `/big.txt?delay=2.0`; the other five are tiny files the server finishes in microseconds. When does the client receive the `/style.css` response?
- A) After ~3 ms, because the server finished it almost at once
- B) After ~1 s, half-way through the slow request
- C) Just after ~2 s, right behind `/big.txt`
- D) Never, because servers reject pipelined requests

> ✅ **Answer: C.** HTTP/1.1 responses carry no request ID, so they must come back in request order. Every fast response waits behind the slow one (2003.6 ms on the slide). That is head-of-line blocking.

**Q3.** An interviewer says "HTTP/1.0 suffers from head-of-line blocking." According to the lecture, what is the more precise statement?
- A) Correct: responses queue behind slow ones on the same connection
- B) Not quite: HTTP/1.0 has no pipelining, so nothing can queue; it has a serialisation cost of one round trip per object
- C) Not quite: HTTP/1.0 suffers from TCP-level HOL blocking only
- D) Correct: HOL blocking was removed in HTTP/1.1 by persistence

> ✅ **Answer: B.** HOL blocking needs a queue, and HTTP/1.0 has none. Real HOL blocking arrived in HTTP/1.1 as a side effect of persistence + pipelining, so D has it backwards.

**Q4.** In your calculator server, `POST /add` arrives with `Content-Length: 7` and body `a=1&b=2`. You send 405 but don't read the body. The client then sends `GET /div?a=9&b=3` on the same socket. What happens?
- A) The next request is answered normally, because a 405 tells the kernel to drop the body
- B) The socket closes automatically after any 4xx
- C) The server reads `a=1&b=2GET /div?...` as the next request line, so it answers garbage or errors and request/response pairs go out of sync
- D) The client resends the body, so everything stays in sync

> ✅ **Answer: C.** On a persistent connection you must consume exactly `Content-Length` bytes, because "byte n+1 belongs to somebody else". The unread body becomes the start of the next request.

**Q5.** A request arrives with both `Content-Length` and `Transfer-Encoding: chunked`. What does RFC 9112 §6.3 (2022) allow a server to do?
- A) Ignore Content-Length, use chunked, and keep the connection open for reuse
- B) Always use Content-Length, because it came first
- C) Add the two lengths together
- D) Reject it, or process it by Transfer-Encoding alone, and close the connection after responding either way

> ✅ **Answer: D.** RFC 2616 said "recover, do not reject" (ignore CL). RFC 9112 makes rejection allowed and recovery optional, and **always** destroys the connection. A keeps the connection, which is exactly what 9112 forbids.

**Q6.** A CDN frames requests with `Content-Length`; the origin behind it uses `Transfer-Encoding`. An attacker sends `POST /echo` with `Content-Length: 6`, `Transfer-Encoding: chunked` and body `0\r\n\r\nGET /admin ...`. What is the result?
- A) The origin sees two requests (POST /echo and GET /admin); the CDN never inspected /admin, and a spare response is left on the connection for whoever is routed next
- B) The CDN sees two requests and blocks /admin
- C) Both see one request; only the body length differs
- D) The origin rejects it, because chunked bodies cannot start with 0

> ✅ **Answer: A.** With CL: 6 the body is `0\r\n\r\nG`, one request. With chunked, `0\r\n\r\n` ends the body and `GET /admin` is a second request. That's request smuggling.

**Q7.** Your server builds the ETag from the file's mtime in whole seconds. A file is written twice within the same second. A client cached the first version and now revalidates with `If-None-Match`. What does it get?
- A) 200 with the new content, because the ETag changed
- B) 412 Precondition Failed
- C) 416 Range Not Satisfiable
- D) 304 Not Modified, so it keeps the stale copy

> ✅ **Answer: D.** Both writes have the same 1-second mtime, so the ETag doesn't change: "two writes in one second are one write to every cache". You built a weak, timestamp-like validator, not a fingerprint.

**Q8.** A CDN caches a page whose response has `Vary: User-Agent`. What is the effect?
- A) Correct caching with a handful of variants
- B) Millions of cache keys, so the hit rate drops towards 0
- C) Caching is turned off completely
- D) The CDN ignores Vary for GET requests

> ✅ **Answer: B.** Vary adds the named header to the cache key, and User-Agent strings are almost all different. `Vary: *` is the one that turns caching off; `Vary: Accept-Encoding` is the correct, small one.

**Q9.** A file is 1000 bytes long. The client sends `Range: bytes=-100`. Which response is correct?
- A) 206, `Content-Range: bytes 0-99/1000`
- B) 416, `Content-Range: bytes */1000`
- C) 206, `Content-Range: bytes 900-999/1000`, `Content-Length: 100`
- D) 206, `Content-Range: bytes 900-1000/1000`

> ✅ **Answer: C.** `-100` is a suffix range: the last 100 bytes, as in the slide's `-50` → `201950-201999/202000`. Ranges include both ends, so the last byte is 999, not 1000.

**Q10.** A chunked body arrives as `1a\r\n<data>\r\n10\r\n<data>\r\n0\r\n\r\n`. How many body bytes are there in total?
- A) 42
- B) 36
- C) 26
- D) 28

> ✅ **Answer: A.** Chunk sizes are hex: 0x1a = 26 and 0x10 = 16, so 26 + 16 = 42; the `0` chunk ends the body. 36 comes from wrongly reading `10` as decimal.

**Q11.** A keep-alive server does `write(head); write(body)`. The first request takes 0.9 ms; every later one takes ~41–44 ms. Which fix removes the delay **and** keeps the connection persistent?
- A) Send `Connection: close` on every response
- B) Make the client's delayed ACK wait longer
- C) Open more parallel connections
- D) Build head + body into one write and set `TCP_NODELAY`

> ✅ **Answer: D.** Nagle holds the second small write until the first is ACKed, and the client delays its ACK up to 40 ms. Closing (A) also pushes the data out, which is why HTTP/1.0 never saw this, but it throws away persistence.

**Q12.** A form does `POST /checkout`. The server wants the browser to repeat the **same POST, with its body**, at a temporary new URL. Which status should it send?
- A) 302
- B) 303
- C) 307
- D) 301

> ✅ **Answer: C.** RFC 2068 said 302 keeps the method, but every browser turned the POST into a GET. RFC 2616 renamed 302 "Found" and added 307 for people who meant what 302 originally said.

**Q13.** An HTTP/2 client sends its connection preface to a server that only speaks HTTP/1.1. What happens?
- A) The server parses the binary frames as headers and crashes
- B) The server sees method `PRI`, version `HTTP/2.0`, and fails cleanly with 501
- C) The server upgrades to h2 automatically
- D) The server waits forever for more header lines

> ✅ **Answer: B.** `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` (24 bytes) looks like a complete HTTP/1.x request, so an old server fails cleanly with 501 instead of trying to read the frames behind it as text.

**Q14.** Decode this HTTP/2 frame header: `00 00 1e 01 04 00 00 00 05`.
- A) HEADERS, 30-byte payload, END_HEADERS, stream 5 (client-initiated)
- B) DATA, 30-byte payload, END_STREAM, stream 5
- C) HEADERS, 30-byte payload, stream 4 (server-initiated)
- D) SETTINGS on stream 0

> ✅ **Answer: A.** Length 0x00001e = 30, type 0x01 = HEADERS, flags 0x04 = END_HEADERS (as in the slide's `00 00 4d 01 04 ...`), stream ID 5, which is odd, so client-initiated.

**Q15.** In the HPACK demo the same header set costs 174 bytes as HTTP/1.1 text, 79 bytes the first time and 6 bytes the second time. Why exactly 6?
- A) The second request is gzip-compressed
- B) Huffman coding shrinks every value to one byte
- C) All six fields are now in the static or dynamic table, so each is sent as one one-byte index
- D) The server cached the request and the client sends only a 6-byte hash

> ✅ **Answer: C.** "Six one-byte indices for six header fields." The first request pays to put fields into the dynamic table (from index 62); after that they are just indexed.

**Q16.** A site moves to HTTP/2 but keeps its HTTP/1.1-era domain sharding (img1, img2, img3). What does the lecture say about this?
- A) It helps: 3 × 6 connections give 18 parallel streams
- B) It is neutral: HTTP/2 shares one dynamic table across all hostnames
- C) It helps, because each shard gets its own SETTINGS
- D) It hurts: the HPACK dynamic table is per connection, so every shard starts empty and re-sends headers in full

> ✅ **Answer: D.** "The dynamic table is per connection, which is why HTTP/2 wants ONE. Domain sharding was clever under 1.1 and is actively harmful here."

**Q17.** CRIME broke compressed headers by watching the compressed size. Why doesn't the same attack work on HPACK?
- A) HPACK encrypts each header separately
- B) Its Huffman table is fixed (never adapts to the data), and the dynamic table only matches whole values exactly, so guessing a prefix of the cookie wins nothing
- C) HPACK pads every header block to a fixed size
- D) HPACK uses gzip with a random dictionary per request

> ✅ **Answer: B.** There is no shared adaptive dictionary where attacker text and the secret can meet. RFC 7541 §7.1 also lets sensitive fields be marked "never indexed".

**Q18.** The lecturer's first HTTP/2 server gave every stream ~2005 ms, exactly the HTTP/1.1 pipelined numbers, even though the frames were correctly multiplexed. What was the cause?
- A) Requests were handled inline in the frame-reading loop, so the 2-second `/big.txt` handler blocked all other streams; the fix was a thread per stream plus a lock around the connection
- B) TCP head-of-line blocking from a lost packet
- C) HTTP/2 requires responses in request order
- D) `MAX_CONCURRENT_STREAMS` defaults to 1

> ✅ **Answer: A.** "A protocol can only give you permission." The transport was fine; the code serialised the work itself. C is false (h2 responses arrive out of order on purpose), and D is false (there is no default).

**Q19.** Server push is still in RFC 9113 §8.4 and RFC 9114 §4.6. Why did Chrome remove it, and what replaced it?
- A) RFC 9113 deprecated it; replaced by PRIORITY_UPDATE
- B) PUSH_PROMISE was vulnerable to CRIME; replaced by QPACK
- C) The server doesn't know what is in the client's cache, so it pushes files the client already has; replaced by 103 Early Hints
- D) HTTP/3 dropped push; replaced by 0-RTT

> ✅ **Answer: C.** The only two "deprecat" hits in RFC 9113 are PRIORITY and the h2c Upgrade token, not push. 103 Early Hints informs the client, and the client decides using its own cache.

**Q20.** On an HTTP/2 connection, a TCP segment carrying stream 5's data is lost. Bytes for streams 7 and 9 that came after it are already in the kernel's receive buffer. What happens?
- A) Streams 7 and 9 are delivered at once, because they are different streams
- B) Only stream 5 waits for the retransmission
- C) The HTTP/2 connection is reset with RST_STREAM
- D) The kernel holds back all later bytes until the gap is filled, so streams 7 and 9 stall too

> ✅ **Answer: D.** TCP can't see streams; it must keep its in-order guarantee. The blast radius is "whoever had bytes behind the gap". B describes HTTP/3 over QUIC.

**Q21.** RTT is 100 ms. Roughly how long until the first response for (i) a cold HTTP/2 connection over TCP + TLS 1.2, and (ii) a resumed QUIC connection using 0-RTT?
- A) 400 ms vs ~100 ms
- B) 300 ms vs ~100 ms
- C) 400 ms vs ~200 ms
- D) 300 ms vs 0 ms

> ✅ **Answer: A.** TCP + TLS 1.3 is 3 round trips and "with TLS 1.2 it was four" = 400 ms. With 0-RTT the request goes in the first flight, so the answer comes after ~1 RTT (1.1 measured). 300 ms is the TLS 1.3 number.

**Q22.** A client sends `POST /pay` as 0-RTT early data to server A. An attacker copies the packet and sends it to server B in the same farm. Why can this work, and what is the practical rule?
- A) 0-RTT data is not encrypted; the rule is to use TLS 1.2
- B) The single-use ticket cache is per server, so B has never seen it; allow only idempotent requests in 0-RTT and let the origin refuse with 425 Too Early
- C) It can't work, because connection IDs stop replays
- D) It works only on HTTP/2; the rule is to answer 409 Conflict

> ✅ **Answer: B.** RFC 9001 calls replay protections "imperfect" for this reason. Cloudflare allows only GET and HEAD in 0-RTT and sets `Early-Data` so the origin can send 425.

**Q23.** An HTTP/3 peer uses `QPACK_BLOCKED_STREAMS = 0`. The encoder has just sent a new cookie entry on the encoder stream, and the decoder hasn't acknowledged it yet. What may the request on stream 8 do?
- A) Reference the new entry; the decoder will wait for it
- B) Repeat the insertion inside stream 8's own header block
- C) Not reference the unacknowledged entry; it may only use entries known to be safe, so no stream ever blocks
- D) Close the connection until the ack arrives

> ✅ **Answer: C.** QPACK's encoder only references entries it knows are safe. With 0 blocked streams allowed, referencing an unacknowledged entry (A) would stall the stream, which is HPACK's HOL problem again.

**Q24.** A site advertises HTTP/3 **only** through the `Alt-Svc` header. A browser with a cold cache visits it for the first time. What happens?
- A) The first request goes straight over QUIC
- B) The DNS answer tells the browser to use h3
- C) The browser sends every request over both protocols
- D) The first connection is TCP + TLS; the browser learns `h3=":443"` from the response and uses QUIC later

> ✅ **Answer: D.** Alt-Svc costs "one full TCP+TLS connection, every cold cache". Only the HTTPS DNS record (type 65, RFC 9460) lets the browser know about h3 before the first connection.

**Q25.** A phone switches from Wi-Fi to LTE in the middle of a download. What happens to a TCP connection and to a QUIC connection?
- A) TCP dies because its four-tuple changed; QUIC continues with the same connection ID after path validation, and endpoints rotate IDs from a pool so observers can't link the two paths
- B) Both survive, because the server remembers the client
- C) Only QUIC survives, because UDP keeps the old IP address
- D) Only QUIC survives, because it keeps one fixed, visible connection ID for the whole connection

> ✅ **Answer: A.** In TCP the connection *is* the four-tuple. QUIC separates identity from addressing. D is wrong because a stable visible ID would be a tracking identifier, so IDs must not be correlatable and are rotated.
