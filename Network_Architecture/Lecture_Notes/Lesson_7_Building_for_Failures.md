# 📓 Building for Failures
### *CN at Scaler — Lesson 7 (Session 7 of 8)* · "fail on purpose"

> Timeouts, retries, backoff, circuit breakers — and what a video player does when the network lies to it.
> **classify → stop if structural → retry if transient → trip the breaker → degrade, don't die**

---

## ⭐ WHAT THIS LECTURE IS ABOUT

A request never fails "in general". It fails at a **specific stage**, and the stage decides whether a retry can help. Before you retry, ask: *"will anything be different next time?"* If yes, retry carefully (capped attempts, exponential backoff, full jitter, obey `Retry-After`). If a dependency keeps failing, stop calling it (circuit breaker). If nothing good is left, fail over, degrade, or serve stale — **anything but nothing**. The case study is a **video player**.

> 💡 **Simple words mein:** Dukaan band hai toh baar baar darwaza khatkhatane se nahi khulegi (don't retry); bas bheed hai toh thoda ruk ke aao (retry); aur kuch na mile toh doosri dukaan se le lo (fallback).

---

## 📋 Table of contents

1. [Recap: the code from earlier sessions](#1--recap-the-code-from-earlier-sessions)
2. [Failure taxonomy: one request, six places to fail](#2--failure-taxonomy-one-request-six-places-to-fail)
3. [HTTP status, revisited with one new question](#3--http-status-revisited-with-one-new-question)
4. [Definitions: is this actually a retry?](#4--definitions-is-this-actually-a-retry)
5. [When to retry: classify before you decide](#5--when-to-retry-classify-before-you-decide)
6. [Idempotency: when blind retry is safe](#6--idempotency-the-question-that-decides-if-blind-retry-is-safe)
7. [How to retry: the naive loop, backoff, jitter](#7--how-to-retry-mechanics)
8. [When the server tells you how long to wait (429 / 503 / 408)](#8--when-the-server-tells-you-how-long-to-wait)
9. [The circuit breaker](#9--the-circuit-breaker)
10. [Case study: a video player's 96 reasons to retry](#10--case-study-a-video-players-96-reasons-to-retry)
11. [Systems that survive](#11--systems-that-survive)
12. [Synthesis: one flowchart](#12--synthesis-one-flowchart)
13. [Assignments & homework — with solutions](#13--assignments--homework-with-solutions)
14. [Cheat-sheet](#14--cheat-sheet)
15. [MCQs (mid → advanced)](#15--mcqs-mid--advanced)

---

## 1 · RECAP: THE CODE FROM EARLIER SESSIONS

The recap slides re-show two programs from Session 1 (full code and walkthrough: **Lesson 1**, the `echo_server.c` and SIGPIPE-demo sections):
- **`echo_server.c`** — "Send it anything. It sends it back, once." (TCP server on port 2026.)
- **`sigpipe_client.c`** — "`SO_LINGER {1, 0}` sends **RST** instead of FIN." (abortive close)

Then: *"Revise the client/server code and the HTTP status vocabulary from Sessions 1–5 — then place every failure on the timeline of a single request."*

---

## 2 · FAILURE TAXONOMY: ONE REQUEST, SIX PLACES TO FAIL

```
   1            2               3               4               5                   6
 DNS lookup → TCP handshake → TLS handshake → Request write → Server processing → Response read
 name → IP    SYN/SYN-ACK/ACK  cert + key      bytes leave     their problem,      bytes arrive,
                               exchange        your socket     now yours too       or stop arriving
```

1. *"Every failure this session talks about lives on this line. Nothing fails generically — it fails at a specific stage, and the stage decides whether a retry can possibly help."*
2. *"A DNS timeout and a read timeout are both 'timeouts' in your logs and completely different problems: one never reached a server, the other reached one that might have half-done the work."*

---

## 3 · HTTP STATUS, REVISITED WITH ONE NEW QUESTION

*(callback: Session 5 — HTTP semantics)*

| Class | Meaning | What actually happened | Retry? |
|---|---|---|---|
| **1xx** | Informational | Interim — keep listening, not a final answer | n/a |
| **2xx** | Success | The server did the thing | no need |
| **3xx** | Redirection | Ask again, somewhere else, maybe with a different method | depends |
| **4xx** | Client error | You sent something the server will refuse **every time you send it unchanged** | usually not |
| **5xx** | Server error | The server admits it failed, but says nothing about whether it changed anything | usually yes |

- Session 5 taught what each status *means*. The new question: **does receiving it tell you whether trying again will change the outcome?**
- *"That single question — 'will anything be different next time?' — is the entire retry decision. Everything from slide 9 onward is that question applied code by code."*

---

## 4 · DEFINITIONS: IS THIS ACTUALLY A RETRY?

> 💡 **Simple words mein:** Retry = wahi form, wahi window pe dobara jama karna. Handling = form ki galti theek karke ya doosri window pe jaana — woh retry nahi hai.

| RETRY | NOT A RETRY |
|---|---|
| Trying the **exact same step again, with nothing changed**. | A **valid, different action** that looks similar. |
| Same request. Same method, same body, same headers that matter. You're betting that the **SAME** action will get a **DIFFERENT** result, because you believe the failure was **transient** — the network blinked, the server was briefly overloaded, a load balancer picked an unhealthy backend. | Refreshing an expired token and re-sending the request is **not** a retry — it's a different request (new `Authorization` header) responding to information the server just gave you. Auth failing outright is usually not something a retry fixes at all: it needs a different credential, not another attempt with the old one. |

> **The test:** *if you changed nothing about the request and only waited, would you call the second attempt identical to the first? If the answer is no — new token, different endpoint, altered payload — you are not retrying, you are handling.*

---

## 5 · WHEN TO RETRY: CLASSIFY BEFORE YOU DECIDE

Classify the failure by **nature and severity** before you touch a retry counter:

| # | Question | What it distinguishes |
|---|---|---|
| 1 | **Where in the request did it fail?** | Before a connection existed (DNS, handshake) vs after (mid-response) |
| 2 | **Is the server admitting fault?** | 5xx says "I failed". 4xx says "you failed". |
| 3 | **Is the action idempotent?** | Same effect no matter how many times it runs? |
| 4 | **Is this transient or structural?** | Will waiting 200 ms change anything, or is the resource actually gone? |

### 5.1 No point retrying *(mindmap: "no point in retrying")*

| Failure | Why retrying is useless |
|---|---|
| **404 — the resource is not there** | If it doesn't exist now, it won't exist 200 ms from now — *unless* you have specific reason to believe otherwise (the CDN-propagation exception, slide 22 → §10.3). |
| **Before the handshake completes** | DNS never resolved, or TCP/TLS never finished — you never reached a server, so nothing about "the server" failed. Retrying the **same broken path** just burns time twice. |
| **Geo-blocks** | Refused by **policy**, not by accident. No amount of retrying changes geography. |
| **Preflight failures** | The browser/client already decided the real request isn't allowed (**CORS**, **mixed content**). The real request was never sent. |
| **Auth failing outright** | Not "don't handle it" — handle it with a **token refresh**, a different, valid flow, not a retry (slide 8 → §4). |

### 5.2 Worth retrying — "nothing was decided against you"

| SERVER ADMITS FAULT | ENVIRONMENT SHIFTED UNDER YOU |
|---|---|
| 5xx from the server (**500 / 502 / 503 / 504**) | **Network switch mid-flight** (4G → 5G, wifi → cellular) |
| **Connection timeout** — you never got a socket | **CDN edge / load balancer** momentarily can't take the request |
| **Read timeout** — you had one, then it went quiet | **Cache not found** (an upstream fetch will repopulate it) |
| **DNS server not responding** (not "no record" — no answer at all) | **TLS handshake failure** on an otherwise-healthy host |

---

## 6 · IDEMPOTENCY: THE QUESTION THAT DECIDES IF BLIND RETRY IS SAFE

*(callback: RFC 9110 §9.2 — Session 5)*

> 💡 **Simple words mein:** Lift ka button 5 baar dabao, lift ek hi baar aati hai (idempotent). ATM se ₹500 do baar nikalo toh ₹1000 gaye (not idempotent). Idempotency key = bank ka token number: same token dobara aaye toh kaam dobara nahi hota.

| Method | Idempotent? | Retry after timeout? | Why |
|---|---|---|---|
| **GET** | yes | safe, blindly | reading twice changes nothing |
| **PUT** | yes | safe, blindly | "set it to X" twice = set to X |
| **DELETE** | yes | safe, blindly | already gone → already gone |
| **POST** | **no** | **only with an idempotency key** | "charge the card" twice = two charges |

- *You didn't get a response — you don't know if the server finished the work before the connection died.*
- *For a non-idempotent action, blind retry can double-execute. The fix isn't "don't retry", it's "make the server able to tell attempt #2 was the same as attempt #1" — an **idempotency key** sent with the request, checked and deduplicated server-side.*

---

## 7 · HOW TO RETRY: MECHANICS

> *"Build the actual retry loop live — count, backoff, jitter, and the circuit breaker that stops it from making things worse."*

### 7.1 The retry loop everyone writes first — and why it's dangerous

```python
while True:
    try:
        return call_upstream(request)
    except TimeoutError:
        continue   # try again. immediately. forever.
```

| Missing piece | What goes wrong |
|---|---|
| **No max count** | A permanently-down dependency becomes an **infinite loop**, not a fast failure. |
| **No delay** | Every client hammers the upstream **at wire speed** the instant it starts failing. |
| **No coordination across clients** | When the upstream recovers, every retry loop fires **on the same tick**. This is the **thundering herd**, and it can re-crash a server that had just come back up. |

### 7.2 Exponential backoff, with jitter *(built with the class)*

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

**THREE KNOBS, THREE JOBS:**

| Knob | Job |
|---|---|
| **MAX ATTEMPTS** | caps total damage — a dead dependency fails in **seconds**, not forever |
| **EXPONENTIAL** `base * 2^attempt` | spaces retries out so load on the upstream **DECAYS** instead of holding steady |
| **FULL JITTER** `uniform(0, delay)`, *not* `delay ± a bit` | **decorrelates clients**. Without it, every client computes the SAME delay and they all retry in lockstep — the herd, just delayed |

> 💡 **Simple words mein:** Sabko "1 ghante baad aana" bologe toh sab ek saath aa jayenge aur counter pe phir bheed. Jitter = har customer ko alag random time dena, taaki line ek saath na lage.

---

## 8 · WHEN THE SERVER TELLS YOU HOW LONG TO WAIT

| Code | Name | What it means | Retry strategy |
|---|---|---|---|
| **429** | Too Many Requests | Rate limit hit — the server is fine, **you're asking too fast** | Honor the `Retry-After` header; don't back off blindly on top of it |
| **503** | Service Unavailable | Often paired with `Retry-After` during planned maintenance / overload shedding | Same — trust the header over your own backoff curve |
| **408** | Request Timeout | Server gave up waiting for **you** to finish sending | Retry is fine — nothing about your request was rejected |

- *Connection limits and 5/10-second interval caps (mindmap: "how do we retry") exist for exactly this reason — clients that ignore `Retry-After` and back off on their own schedule can still cumulatively overwhelm a server that told them, explicitly, to slow down.*
- ***Rule of thumb:** a server that hands you a number is smarter about its own load than your exponential curve is. Use the number.*

---

## 9 · THE CIRCUIT BREAKER

*"Stop asking a question you already know the answer to."* *(callback: Michael Nygard, Release It! (2007))*

> 💡 **Simple words mein:** Dost ka phone 5 baar switched off aaya? Ab har minute call mat karo (OPEN). Thodi der baad ek baar try karo (HALF-OPEN); lag gaya toh normal (CLOSED), nahi laga toh phir ruko.

```
  CLOSED ──failure threshold crossed──▶ OPEN ──cooldown elapses──▶ HALF-OPEN
     ▲                                   ▲                             │
     └────────── probe succeeds ─────────┼─────────────────────────────┤
                                         └──────── probe fails ────────┘
```

| State | Behaviour |
|---|---|
| **CLOSED** | Requests flow through normally. Failures are counted. |
| **OPEN** | Failure threshold crossed. Requests **fail instantly, locally** — no call is even attempted. |
| **HALF-OPEN** | After a cooldown, let **ONE** request through as a probe. Success → close. Failure → re-open. |

1. **Backoff protects the upstream from ONE client retrying too fast. A circuit breaker protects the upstream from an ENTIRE fleet** still calling a service everyone already knows is down — **and protects the caller** from paying the full timeout cost on every one of those doomed calls.
2. *"This is the max-retry-count idea, one level up: instead of capping retries **per request**, cap failures **per DEPENDENCY**, and remember the verdict for a while."*

---

## 10 · CASE STUDY: A VIDEO PLAYER'S 96 REASONS TO RETRY

> *"Real production taxonomy, condensed — what changes when the client is a player, not a browser tab."*

### 10.1 Why a player's retry logic is harder than an API client's

- An API client fails **visibly** — an exception, a 5xx in a log. A player that mishandles a failure fails **silently**, as a frozen frame or a spinner the user stares at.
- One "play" session makes **hundreds of requests** — manifest, key, and dozens of segments — each with a **different retry policy**, on **one continuous user-visible timeline**.
- Some failures **aren't network failures at all**: a DRM key can be denied, a codec can be unsupported, a decoder can crash — the correct response is often not "retry" but "**fall back to a different stream entirely**."
- The retry budget is a **human** one: a viewer tolerates a few seconds of rebuffering, not a few minutes of exponential backoff playing out correctly.

### 10.2 Network issues — same taxonomy, tighter clock *(mindmap: "network issues")*

| Failure | What's actually different for a player | Retry? |
|---|---|---|
| **DNS server not responding** | Same as any client — but a player often has a **fallback CDN with a different hostname** to try instead | retry, or switch CDN |
| **Connection / read timeout** | **Differential timeouts by request type** — a manifest fetch and a segment fetch don't get the same budget | yes, tuned per type |
| **Network switch (4G→5G)** | **IP address can change mid-download** — the in-flight request is **dead, not slow** | yes, fresh request |
| **Wifi off mid-playback** | **Buffer keeps playing while offline** — retry only needs to start once buffer runs low | yes, deferred |
| **CDN edge not responding** | **Multi-CDN** players fail over to a second CDN rather than retry the same one | switch, don't retry |

### 10.3 When 404 doesn't mean "give up"

| THE EXCEPTION: 404 on a live segment | CUSTOM CODES: 419 / 474 / 475 — not in the RFC |
|---|---|
| General rule (slide 11): 404 means don't retry. **Exception:** a live stream's newest segment can 404 for a real reason — the origin is **still encoding** it, or the CDN **hasn't finished propagating** it. It **WILL** exist in a few hundred milliseconds. | Real deployments layer application-specific codes on top of the HTTP registry: **419** (token/session expiry — handle like auth, not a retry) and **474/475** (internal codes seen when a segment is retried in a way that returns a **mismatched, partial result**). |
| The fix isn't "retry blindly" — it's a **short, bounded retry (2-3 attempts, small fixed delay)** specifically for **live-edge segment requests, and nowhere else**. A 404 on an **on-demand** asset still means give up. | The lesson isn't the numbers — it's that **your OWN system's status codes deserve the same classify-before-retry discipline as RFC ones.** Document what they mean and whether they're retryable. |

### 10.4 Sometimes the answer isn't retry — it's fall back *(mindmap: "codec/DRM fallback", "stream issue")*

| Failure | What to do |
|---|---|
| **DRM key denied / L1 vs L3** | Retrying the same license request returns the same denial. **Fall back** to a lower DRM security level or a different key system. |
| **Decoder / codec unsupported** | **Not transient** — the device genuinely cannot decode this. Fall back to an **alternate codec rendition** in the manifest. |
| **Master manifest failure → MPD to HLS** | If the DASH manifest fails, some players fall back to an **HLS rendition of the same content** rather than retrying DASH. |
| **Child manifest / segment / init failure** | Retryable in the network sense (the §7 backoff applies) — but a repeated failure **escalates to a fatal, coded error** (e.g. 8018, 8022), not an infinite retry loop. |
| **Ad failure or timeout** | Falls back to **content**, not to a retried ad — the user's time budget for an ad failure is near zero. |
| **iOS ResetMedia** | A platform-specific recovery: **tear down and rebuild the entire AVPlayer pipeline**, because the failure is in **device media state**, not in a request at all. |

### 10.5 Not every retry decision is a network decision

- **Business-rule-based retries** — e.g. retry a payment-gated stream start **only if the user's entitlement check itself times out**, not if it cleanly returns "not entitled". The HTTP layer succeeded; the **business layer made a decision.**
- **Change player behaviour based on information in the response** — the response **body**, not just the status code, can carry a signal ("**degraded, use lower bitrate**", "**maintenance, try again in N minutes**") that tells the player what to do far more precisely than a status code.
- **The general lesson** — the earlier sections classified failures using **transport-layer signals** (status code, timeout type). Real systems also carry **APPLICATION-layer signals**, and a mature retry policy reads **both**.

---

## 11 · SYSTEMS THAT SURVIVE

> *"What survives isn't the request — it's the system."* Failover, graceful degradation, and the thundering herd that shows up right when everyone thinks the outage is over.

### 11.1 "49 minutes, and the thing that was still serving traffic" *(callback: Session 6 — the edge, revisited)*

| 85% | ~1 min | 49 min |
|---|---|---|
| of Fastly's network returning errors | to detect the fault | to 95% recovered — **8 Jun 2021** |

- **Session 6's closing question:** today's edge survives your origin dying, because a cache can serve a **stale hit**. *What does your system do when the edge — or a whole region — is the thing that dies?*
- **`stale-if-error` (RFC 5861)** is the same idea pointed the other way: a cache configured to **keep serving its last good copy** even after it can no longer reach the origin to revalidate it. **Not a retry at all — a decision to serve something known-stale rather than nothing.**
- **The lesson generalizes past caches:** a system that can answer with **slightly-wrong data** survives outages that a **perfectly-fresh-or-nothing** system does not.

> 💡 **Simple words mein:** Taaza samosa khatam? Thoda purana garam karke de do (stale) ya chai-biscuit de do (degrade) — customer ko khaali haath mat bhejo.

### 11.2 Failover, feature flags, graceful degradation

| FAILOVER — *Primary → backup* | FEATURE FLAGS — *Turn off the expensive part* | GRACEFUL DEGRADATION — *Answer smaller, not "no"* |
|---|---|---|
| A **circuit breaker** (slide 18) trips OPEN on the primary and **routes to a backup** — different region, different provider, different CDN. The client doesn't retry a dead dependency; **it stops asking it and asks someone else.** | During an incident, a flag can **disable a non-essential feature** (recommendations, personalization) to **shed load off the critical path** (playback itself), **without a deploy**. Graceful degradation as a **lever you can pull in seconds**. | **Lower bitrate** instead of full quality. **Cached recommendations** instead of live ones. A **slightly stale price** instead of a spinner. Every case study today has one: the player's own **buffer**, the CDN's **stale-if-error**, a **lower DRM tier**. |

### 11.3 The second outage: everyone recovering at once *(callback: slide 16 — jitter, at fleet scale)*

- **The moment a dependency comes back up is the most dangerous moment of the whole incident** — not the failure itself.
- Every client that was retrying, backing off, or waiting on a breaker's cooldown now tries again, and **without jitter they tend to be synchronized** — they all started failing around the same moment, so their backoff schedules line up.
- The recovering service, at **1% of normal capacity** while it warms caches and re-establishes connections, receives **close to 100% of normal traffic** in its first second — and can fall right back over.
- **Mitigations:** **jitter** (slide 16) · **staggered circuit-breaker half-open probes** rather than all-at-once · **slow-start** on the recovering side (accept traffic at a ramped rate, not instantly at full capacity).

> 💡 **Simple words mein:** Bijli wapas aate hi poore mohalle ke AC aur fridge ek saath on → transformer phir ud gaya. Isliye load dheere dheere badhao.

---

## 12 · SYNTHESIS: ONE FLOWCHART

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

---

## 13 · ASSIGNMENTS & HOMEWORK (with solutions)

### 13.1 The HTTP/1.1 calculator that stays on the line

See Lesson 5 §8 for this assignment and its solution.

### 13.2 Course project — HTTP in binary (pairs)

See Lesson 5 (Course project section) for the project brief.

### 13.3 "Things to try" — wrap a flaky client in a policy *(due before Session 8)*

**Task:** put a client behind a resilient-call wrapper you write from scratch. It must (1) **classify** the failure, (2) **back off with full jitter**, capped at a max attempt count, (3) **honor `Retry-After`**, (4) **open a circuit after N consecutive failures** and fail fast while open. Test it against your own misbehaving server (`/sometimes-500`, `/always-503-with-retry-after`, `/always-down`). Deliverable: wrapper + server + a log of request timestamps showing the backoff curve, the circuit opening, and it closing again after the server recovers.

**✅ Solution** — two files, Python standard library only.

**`flaky_server.py`** (the failure fraction is `?p=`, and `/always-down` "recovers" 15 s after the server starts, so you can see the circuit close):

```python
# flaky_server.py — python3 flaky_server.py   (prints one log line per request it receives)
import random, time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse, parse_qs

START, DOWN_FOR = time.time(), 15

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        url = urlparse(self.path)
        p = float(parse_qs(url.query).get("p", ["0.5"])[0])      # failure fraction
        if url.path == "/sometimes-500" and random.random() < p:
            return self.reply(500, b"boom\n")
        if url.path == "/always-503-with-retry-after":
            return self.reply(503, b"busy\n", {"Retry-After": "2"})
        if url.path == "/always-down" and time.time() - START < DOWN_FOR:
            self.close_connection = True       # drop the socket without replying
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

**`policy_client.py`** (classify → backoff + full jitter → Retry-After → circuit breaker):

```python
# policy_client.py — python3 policy_client.py /always-down 10
import random, sys, time, urllib.request, urllib.error

RETRYABLE = {408, 429, 500, 502, 503, 504}       # other 4xx = "you failed" → don't retry
T0 = time.monotonic()

class Retryable(Exception):
    def __init__(self, why, retry_after=None):
        super().__init__(why)
        self.retry_after = retry_after

class CircuitOpen(Exception):
    pass

class Breaker:                                   # CLOSED → OPEN → HALF-OPEN
    def __init__(self, n=5, cooldown=5.0):
        self.n, self.cooldown, self.fails, self.opened_at = n, cooldown, 0, None
    def state(self):
        if self.opened_at is None:
            return "CLOSED"
        return "OPEN" if time.monotonic() - self.opened_at < self.cooldown else "HALF_OPEN"
    def call(self, fn):
        if self.state() == "OPEN":
            raise CircuitOpen("fail fast")       # no call is even attempted
        try:
            result = fn()
        except Retryable:
            self.fails += 1
            if self.state() == "HALF_OPEN" or self.fails >= self.n:
                self.opened_at = time.monotonic()  # trip (or re-open after a failed probe)
            raise
        self.fails, self.opened_at = 0, None     # success → CLOSED
        return result

breaker = Breaker()

def attempt(url):                                # 1. classify
    try:
        with urllib.request.urlopen(url, timeout=2) as r:
            return r.status
    except urllib.error.HTTPError as e:
        if e.code in RETRYABLE:
            raise Retryable(e.code, e.headers.get("Retry-After"))
        return e.code                            # structural: return the answer, no retry
    except (urllib.error.URLError, ConnectionError, TimeoutError) as e:
        raise Retryable(type(e).__name__)        # timeout / reset / dropped socket

def log(n, what, sleep=0.0):
    print(f"{time.monotonic()-T0:6.2f}s attempt={n} {what} sleep={sleep:.2f} breaker={breaker.state()}")

def call(url, max_attempts=5, base=0.2, cap=8.0):
    for n in range(max_attempts):
        try:
            status = breaker.call(lambda: attempt(url))
            log(n, f"-> {status}")
            return status
        except CircuitOpen:
            log(n, "circuit OPEN, failing fast")
            raise
        except Retryable as e:
            if n == max_attempts - 1:            # 2. cap attempts
                log(n, f"{e}, giving up")
                raise
            ra = e.retry_after                   # 3. honor Retry-After, else full jitter
            delay = float(ra) if ra and ra.isdigit() else random.uniform(0, min(cap, base * 2**n))
            log(n, e, delay)
            time.sleep(delay)

if __name__ == "__main__":
    path, runs = sys.argv[1], int(sys.argv[2]) if len(sys.argv) > 2 else 1
    for _ in range(runs):
        try:
            call("http://127.0.0.1:8082" + path)
        except (Retryable, CircuitOpen) as e:
            print("   FAILED:", type(e).__name__)
        time.sleep(1)
```

**Run it** (terminal A: `python3 flaky_server.py`; terminal B: the commands below). Sample output (sleeps are random):

```text
$ python3 policy_client.py "/sometimes-500?p=0.6" 1
  0.01s attempt=0 500 sleep=0.01 breaker=CLOSED          ← backoff: random sleeps,
  0.02s attempt=1 500 sleep=0.35 breaker=CLOSED            ceiling doubles each time
  0.37s attempt=2 500 sleep=0.59 breaker=CLOSED
  0.97s attempt=3 -> 200 sleep=0.00 breaker=CLOSED

$ python3 policy_client.py /always-503-with-retry-after 1
  0.01s attempt=0 503 sleep=2.00 breaker=CLOSED          ← Retry-After: 2 used instead of the curve
  2.01s attempt=1 503 sleep=2.00 breaker=CLOSED
  ...
  8.03s attempt=4 503, giving up sleep=0.00 breaker=OPEN

$ python3 policy_client.py /always-down 9               (restart the server first)
  0.01s attempt=0 RemoteDisconnected sleep=0.10 breaker=CLOSED
  ...
  1.67s attempt=4 RemoteDisconnected, giving up sleep=0.00 breaker=OPEN   ← circuit opens
  2.67s attempt=0 circuit OPEN, failing fast sleep=0.00 breaker=OPEN      ← no server log line
  ...
  6.69s attempt=0 -> 200 sleep=0.00 breaker=CLOSED                        ← probe OK → closes
```

**What to hand in / observe:** (1) the growing, random gaps between attempts = the backoff curve (plot `timestamp` vs `attempt`); (2) exactly 2 s gaps for the 503 = `Retry-After` honored; (3) while OPEN the client prints "failing fast" but the **server prints nothing** = "no call is even attempted"; (4) after the cooldown, one probe succeeds and the state returns to CLOSED.

### 13.4 Homework — "Five, do at least one" (all five solved)

#### HW 1 — 1,000 clients vs a rate-limited server: naive loop vs backoff + jitter

**Task:** simulate 1,000 clients against a rate-limited server, count total requests in the first second under each loop, explain the gap.

**✅ Solution:**

```python
# herd.py — python3 herd.py
import heapq, random

N, K, RTT = 1000, 100, 0.01   # 1,000 clients; server accepts 100 req/s, rejects the rest; 10 ms per reply

def simulate(next_delay):
    q = [(0.0, c, 0) for c in range(N)]          # (time, client, attempt#): all start at t=0
    sent = accepted = 0
    while q:
        t, c, n = heapq.heappop(q)
        if t >= 1.0:                             # only count the first second
            break
        sent += 1
        if accepted < K:                         # within the rate limit → success, client done
            accepted += 1
            continue
        d = next_delay(n)                        # rejected → maybe retry
        if d is not None:
            heapq.heappush(q, (t + RTT + d, c, n + 1))
    return sent

naive = lambda n: 0.0                            # while True: continue
def backoff(n, base=0.2, cap=8.0, max_attempts=5):
    return None if n == max_attempts - 1 else random.uniform(0, min(cap, base * 2**n))

print("naive loop    :", simulate(naive))
print("backoff+jitter:", simulate(backoff))
```

Output (jitter number varies slightly per run):
```text
naive loop    : 90100
backoff+jitter: 3715
```

**Explanation:** the naive loop has no delay, so each of the 900 rejected clients retries every 10 ms (as fast as a failure comes back) → ~100 requests each → **~90,000 requests/s** at a server that can take 100. Backoff + jitter caps each client at 5 attempts, the waits grow (0.2, 0.4, 0.8 s ceilings), and jitter spreads them out → **~3,700**, about **24× less load**, and no synchronized spikes.

#### HW 2 — three case-study items that aren't retries

**✅ Solution** (slide 8's test: anything that changes the request is *handling*, not retrying):
1. **CDN edge not responding → switch to a second CDN:** this is **failover** — a different endpoint, not the same request again.
2. **DRM key denied (L1 vs L3) → lower security level / other key system:** this is a **fallback** — a different license request, because the same one returns the same denial.
3. **419 token/session expiry → refresh and resend:** this is **auth handling** — a new credential, i.e. a different request.

(Also valid: codec fallback, MPD → HLS, ad → content, iOS ResetMedia = rebuild the whole player pipeline.)

#### HW 3 — idempotency key scheme for a POST endpoint

**✅ Solution** (endpoint: `POST /orders`):
- **Client:** creates `Idempotency-Key: <uuid4>` **once per checkout click** and sends the same key on every retry of that order.
- **Server stores seen keys** in the **database**, table `idempotency_keys(user_id, key, request_hash, status_code, response_body, created_at)` with `UNIQUE(user_id, key)`, **inserted in the same transaction** that creates the order (so two parallel retries can't both succeed).
  - key seen + same body → **return the stored response**, don't create a second order;
  - key seen but first attempt still running → `409 Conflict` ("in progress");
  - key seen with a different body → `422` (client bug, not a retry).
- **How long:** **24 hours** (longer than any client's whole retry window), then a nightly job deletes older rows. (Simpler alternative: Redis `SET key response NX EX 86400`.)

#### HW 4 — map the Fastly postmortem onto the five-step framework

**✅ Solution** (Fastly's "Summary of June 8 outage", times UTC):

| Time | Event | Step |
|---|---|---|
| 09:47 | A valid customer config change triggers a latent bug; 85% of the network returns errors | — |
| 09:48 | Fastly monitoring detects it (**~1 min**) | 1. Classify — fast |
| 09:58 | Status page posted | — |
| 10:27 | Engineers identify the customer configuration that triggered it (~40 min after detection) | 2. Structural (a bug: retries can't fix it) |
| 10:36 | Config disabled, services recover (**49 min**, 95% recovered) | 5. No good answer left |
| later | Permanent bug fix deployed | — |

**Answer:** detection was fast, so the time was lost in **mitigation**. **Step 5** (fail over / degrade / serve stale) would have shortened the 49 minutes: for **Fastly**, a ready "turn this off" lever (feature-flag style, pulled in seconds); for **customers**, a **failover to a second CDN** (multi-CDN), as their breakers tripped. Retrying (step 3) could not help — every retry hit the same broken edge.

#### HW 5 — add classify + backoff + jitter + cap to a real client (the one to do)

**✅ Solution** — wrap the HTTP call your daily script makes in this function (Python stdlib):

```python
# fetch.py — python3 fetch.py https://example.com/
import random, time, urllib.request, urllib.error

RETRYABLE = {408, 429, 500, 502, 503, 504}

def fetch(url, max_attempts=5, base=0.5, cap=8.0):
    for n in range(max_attempts):
        try:
            with urllib.request.urlopen(url, timeout=5) as r:   # a timeout, or it hangs forever
                return r.read()
        except urllib.error.HTTPError as e:                     # classify
            if e.code not in RETRYABLE:
                raise                                           # 4xx: structural → stop now
            err = e
        except (urllib.error.URLError, ConnectionError, TimeoutError) as e:
            err = e                                             # network blip → transient
        if n == max_attempts - 1:                               # cap
            raise err
        d = random.uniform(0, min(cap, base * 2**n))            # backoff + full jitter
        print(f"attempt {n} failed ({err}); retrying in {d:.2f}s")
        time.sleep(d)

if __name__ == "__main__":
    import sys
    print(fetch(sys.argv[1])[:80])
```

**Break it on purpose and observe:**

| Test | Command | What you see |
|---|---|---|
| Server that never answers | terminal A: `nc -l 8081` · terminal B: `python3 fetch.py http://127.0.0.1:8081/` | `attempt 0 failed (timed out)`, then quick retries, then a clean error — **no hang** |
| Port blocked / nothing listening | `python3 fetch.py http://127.0.0.1:9/` | `Connection refused` ×4 with growing random waits, then it gives up |
| Kill the wifi | run against a real URL, turn wifi off, turn it back on within ~5 s | a few failed attempts, then the page is printed — **it recovers instead of hanging** |

Note: the `timeout=5` is what turns a "hang" into a failure the retry loop can see.

---

## 14 · CHEAT-SHEET

| Term | One-line meaning |
|---|---|
| **Six failure stages** | DNS → TCP handshake → TLS handshake → request write → server processing → response read |
| **DNS timeout vs read timeout** | one never reached a server; the other reached one that may have half-done the work |
| **Connection timeout** | you never got a socket |
| **Read timeout** | you had a socket, then it went quiet |
| **Retry** | the *exact same* request again, nothing changed, betting the failure was transient |
| **Handling** | a different, valid action (new token, other endpoint, altered payload) — not a retry |
| **Four questions** | where did it fail? · server admitting fault? · idempotent? · transient or structural? |
| **Structural (don't retry)** | 404, before handshake completes, geo-block, preflight (CORS/mixed content), auth failing outright |
| **Worth retrying** | 500/502/503/504, connection/read timeout, DNS not responding, network switch, CDN/LB blip, cache miss, TLS failure on healthy host |
| **4xx / 5xx** | "you failed" — usually don't retry / "I failed" — usually retry |
| **Idempotent** | same effect however many times it runs (GET, PUT, DELETE; not POST) |
| **Idempotency key** | sent with the request; server checks it and deduplicates, so attempt #2 = attempt #1 |
| **Naive retry loop** | no max count, no delay, no coordination → infinite loop, hammering, thundering herd |
| **Exponential backoff** | delay = `min(cap, base · 2^attempt)` → load on the upstream decays |
| **Full jitter** | sleep `uniform(0, delay)`, not `delay ± a bit` → decorrelates clients (AWS 2015) |
| **Max attempts** | caps total damage; dead dependency fails in seconds |
| **429 / 503 / 408** | too fast, honor `Retry-After` / maintenance-overload, trust `Retry-After` / server gave up waiting for you, retry fine |
| **Retry-After rule** | the server's number beats your exponential curve — use it |
| **Circuit breaker** | cap failures per dependency and remember the verdict (Nygard, *Release It!*, 2007) |
| **CLOSED / OPEN / HALF-OPEN** | normal & counting / fail fast locally / one probe after cooldown |
| **Thundering herd** | every client retrying on the same tick — worst right when a service recovers |
| **Recovery mitigations** | jitter · staggered half-open probes · slow-start |
| **Live-edge 404** | newest live segment not ready → 2–3 short fixed-delay retries, only there |
| **Custom codes (419/474/475)** | non-RFC codes; classify and document them like real ones |
| **Player fallbacks** | DRM L1→L3, alternate codec, MPD→HLS, ad→content, iOS ResetMedia |
| **Fatal coded error** | repeated segment/manifest failure ends in a coded error (e.g. 8018, 8022), not an infinite loop |
| **Business-rule retry** | retry entitlement check only on timeout, not on a clean "not entitled" |
| **stale-if-error (RFC 5861)** | cache serves its last good copy when the origin can't be reached — not a retry |
| **Failover / feature flag / degradation** | ask a backup / turn off non-essential features without a deploy / answer smaller, not "no" |
| **Fastly, 8 Jun 2021** | 85% of network erroring, ~1 min to detect, 49 min to 95% recovered |
| **5-step synthesis** | classify → structural? stop → transient? retry well → breaker → fail over/degrade/serve stale |

---

## 15 · MCQs (mid → advanced)

**Q1.** A client logs a **connect timeout** (TCP handshake never completed) on `POST /orders`. Which statement is correct?
- A) It is unsafe to retry, because POST is not idempotent
- B) The request never reached a server, so nothing ran there; retrying cannot double-execute it
- C) It is a client-side refusal like a 4xx, so it should not be retried
- D) The client must refresh its auth token before sending again

> ✅ **Answer: B.** A connection timeout means "you never got a socket" — the failure happened before any server saw the request. The idempotency worry (A) only applies once the server may have acted (e.g. a read timeout).

**Q2.** `POST /payments` hits a **read timeout**, and the client sends no idempotency key. What is the right move?
- A) Retry immediately — read timeouts are on the "worth retrying" list
- B) Retry with exponential backoff and full jitter; spacing the attempts out prevents a double charge
- C) Tell the user the payment failed, since no 2xx came back
- D) Don't retry blindly — the server may already have charged; the fix is an idempotency key the server deduplicates

> ✅ **Answer: D.** After a read timeout you don't know if the work finished, and POST is not idempotent, so a blind retry can charge twice. Backoff (B) changes *when* you retry, not *whether* it double-executes. C may be false, because the charge may have gone through.

**Q3.** You are writing a generic retry policy for HTTP status codes. Using the lecture's rules, which set should it retry?
- A) 408, 429, 500, 502, 503, 504
- B) 400, 401, 404, 500, 503
- C) 403, 404, 429, 500
- D) Only 500 and 503

> ✅ **Answer: A.** The 5xx codes (500/502/503/504) are "server admits fault", and 408/429 are the 4xx codes that are retryable. 404, geo-blocks (often 403) and auth failures (401) are structural — nothing changes on the next try.

**Q4.** A request fails with 401 because the token expired. The client refreshes the token and sends the request again. How should this be counted?
- A) As retry #1 of 3, with a backoff sleep before re-sending
- B) As a retry, because the URL, method and body are the same
- C) Not as a retry — it is handling (a new `Authorization` header), so no backoff and no retry budget
- D) As structural — give up at once without refreshing

> ✅ **Answer: C.** Slide 8's test: the second attempt is not identical (new token), so it is handling. Waiting would not fix an expired token. D is wrong because the slide says to handle auth with a token refresh.

**Q5.** A player asks for the **newest segment of a live stream** and gets 404. What does the lecture recommend?
- A) Give up — a 404 is never retried
- B) A short, bounded retry (2–3 attempts, small fixed delay), only for live-edge segment requests
- C) Exponential backoff with full jitter, up to the 8 s cap
- D) Keep retrying until it returns 200, because the segment will exist

> ✅ **Answer: B.** The segment may still be encoding or propagating through the CDN, so it will exist in a few hundred ms. Retrying must stay bounded (not D) and fast (not C — the viewer can't wait). A 404 on an on-demand asset still means give up.

**Q6.** `retry(fn, max_attempts=6, base=0.5, cap=4.0)` from slide 16 runs, and every attempt fails. What is the **maximum** total time spent sleeping?
- A) 7.5 s
- B) 15.5 s
- C) 11.5 s
- D) 31.5 s

> ✅ **Answer: C.** Sleeps happen after attempts 0–4 only (the last attempt raises): 0.5 + 1 + 2 + 4 + min(4, 8) = 11.5 s. 15.5 s forgets the cap; 31.5 s also adds a sleep after the final attempt.

**Q7.** With the slide defaults (`max_attempts=5, base=0.2, cap=8.0`) and an upstream that always fails, what is the **average** total sleep?
- A) 3.0 s
- B) 1.5 s
- C) 0.75 s
- D) 8.0 s

> ✅ **Answer: B.** The ceilings are 0.2 + 0.4 + 0.8 + 1.6 = 3.0 s. Full jitter picks `uniform(0, delay)`, whose average is half the ceiling, so ~1.5 s. 3.0 s is the worst case, not the average.

**Q8.** Someone raises `max_attempts` to 10 with `base=0.2, cap=8.0`. After the attempt with index 6 fails, what is `delay` **before** jitter is applied?
- A) 12.8 s
- B) 6.4 s
- C) a random value between 0 and 12.8 s
- D) 8.0 s

> ✅ **Answer: D.** `base * 2**6` = 12.8, and `min(cap, 12.8)` = 8.0. Only after that does `uniform(0, 8.0)` add the jitter. 6.4 s is the value for index 5.

**Q9.** A service uses the naive `while True: ... continue` loop. The upstream goes down; when it comes back, it is knocked over again within a second. Which missing piece is mainly to blame for the **re-crash on recovery**?
- A) No coordination across clients — every loop fires on the same tick (thundering herd)
- B) No max count
- C) No delay
- D) Catching only `TimeoutError`

> ✅ **Answer: A.** The slide ties the recovery spike to the lack of coordination: all clients retry together the moment it is back. "No delay" (C) explains the hammering while it is down, and "no max count" (B) explains the infinite loop.

**Q10.** A 429 arrives with `Retry-After: 30`. Your backoff curve says to wait 0.4 s. What should the client do?
- A) Wait 0.4 s — your own curve controls the retries
- B) Wait the 30 s the server asked for
- C) Wait 30 s and then your own backoff on top, to be extra safe
- D) Don't retry — 4xx means a client error

> ✅ **Answer: B.** "A server that hands you a number is smarter about its own load than your exponential curve is. Use the number." The 429 row says honor `Retry-After` and don't back off blindly on top of it (C). 429 is a retryable 4xx (D).

**Q11.** A server returns **408 Request Timeout**. What does it mean, and should you retry?
- A) The server took too long to process the request; don't retry
- B) The request was malformed; fix it before sending again
- C) The server gave up waiting for you to finish sending; retrying is fine because nothing about the request was rejected
- D) You hit a rate limit; honor `Retry-After`

> ✅ **Answer: C.** 408 is about the client being slow to send, not about the content being bad, so it is one of the retryable 4xx codes. A is the confusion with 504 (gateway timeout); D describes 429.

**Q12.** 1,000 clients all started failing at the same moment. Why does the slide want `uniform(0, delay)` and **not** `delay ± a bit`?
- A) `± a bit` makes each client send more retries in total
- B) `uniform(0, delay)` lowers the number of attempts per client
- C) Clients that failed together compute the same delay, so `± a bit` keeps them nearly in lockstep (the herd, just delayed); spreading over the whole range decorrelates them
- D) `uniform(0, delay)` guarantees each client waits at least `delay`

> ✅ **Answer: C.** Jitter does not change the number of attempts (A, B). Its job is to spread clients out in time. D is backwards — full jitter can wait anywhere from 0 up to `delay`.

**Q13.** 10,000 clients each use correct exponential backoff but **ignore** `Retry-After: 120` on a 503 during maintenance. What risk does the slide name?
- A) None — backoff alone protects the server
- B) The clients will hit their max attempt count too early
- C) POST requests will be executed twice
- D) Together they can still overwhelm a server that explicitly told them to slow down

> ✅ **Answer: D.** That is why connection limits and 5/10-second interval caps exist: each client's curve may look polite, but the total of clients on their own schedules can still swamp a server that gave a number.

**Q14.** A circuit breaker opens after **3 consecutive failures** and has a **10 s cooldown**. Calls: t=0 fail, t=1 fail, t=2 success, t=3 fail, t=4 fail, t=5 fail. What happens to a call at **t=12**?
- A) It fails instantly and locally — no network call is made
- B) It goes through as the half-open probe
- C) It goes through normally, because the success at t=2 reset everything
- D) It waits until t=15 and is then sent

> ✅ **Answer: A.** The success at t=2 reset the count, then t=3, 4, 5 are 3 consecutive failures → OPEN at t=5. The cooldown runs until t=15, so at t=12 the breaker is still OPEN and fails fast. It does not queue the call (D).

**Q15.** Same breaker: at t=16 the half-open probe **fails**. When is the earliest a request can reach the upstream again?
- A) Immediately, on the next call at t=16
- B) After 3 more failures are counted
- C) At t=26
- D) Never — once re-opened, it stays open until a restart

> ✅ **Answer: C.** "Failure → re-open": the breaker goes back to OPEN at t=16 and a new 10 s cooldown starts, so the next probe can go at t=26. While OPEN, no calls are made, so no more failures can be counted (B).

**Q16.** A breaker sits in front of a user-profile service. During a bot attack that asks for **non-existent profiles (404s)**, it trips OPEN and blocks every real user. What is the bug?
- A) The threshold is too high
- B) It counts 404s as dependency failures — a 4xx means "you failed", not that the dependency is unhealthy
- C) The cooldown is too short
- D) It should be one global breaker instead of one per dependency

> ✅ **Answer: B.** A breaker caps failures **per dependency** — it should count signs that the service is down (timeouts, 5xx). A 404 is a structural "the resource isn't there" answer from a healthy server.

**Q17.** What does a circuit breaker protect against that backoff does **not**?
- A) One client retrying too fast
- B) Duplicate execution of non-idempotent requests
- C) An entire fleet still calling a service everyone knows is down — and the caller paying the full timeout on every doomed call
- D) The need to honor `Retry-After`

> ✅ **Answer: C.** Backoff protects the upstream from ONE client (A). The breaker works one level up: it caps failures per dependency and remembers the verdict, so calls fail fast and locally.

**Q18.** After an outage the upstream comes back, receives close to 100% of normal traffic while at ~1% capacity, and falls over again — even though clients already use jitter. Which further mitigations from the slides fit?
- A) Raise every client's max attempt count
- B) Remove the breaker cooldown so breakers close sooner
- C) Switch all clients to a fixed retry delay
- D) Stagger the circuit breakers' half-open probes, and use slow-start on the recovering side

> ✅ **Answer: D.** The slide lists jitter, staggered half-open probes instead of all at once, and slow-start (accept traffic at a ramped rate). A and B add load sooner; a fixed delay (C) re-synchronizes clients.

**Q19.** Mid-download, a phone switches from 4G to 5G and its IP address changes. What should the player do with the in-flight segment request?
- A) Abandon it and send a fresh request right away — it is dead, not slow
- B) Wait for the read timeout, then back off
- C) Fail over to a second CDN
- D) Treat it as structural and show an error

> ✅ **Answer: A.** The slide's row: "IP address can change mid-download — the in-flight request is dead, not slow" → "yes, fresh request". Waiting for a timeout (B) just wastes the viewer's tiny time budget, and the CDN is fine (C).

**Q20.** `DELETE /photos/7` gets a read timeout. The client retries, and the retry returns **404**. How should it be read?
- A) The retry failed; report an error to the user
- B) It is consistent with the first attempt having worked — DELETE is idempotent ("already gone → already gone"), so treat it as done
- C) The photo never existed, so the first attempt must have failed
- D) DELETE needed an idempotency key; the photo was deleted twice

> ✅ **Answer: B.** The end state (photo gone) is what idempotency promises. The 404 on the retry most likely means the first DELETE succeeded before the connection died. Only POST needs a key (D).

**Q21.** A developer adds an idempotency key to `POST /payments` but generates a **new UUID inside the retry loop**, one per attempt. What happens?
- A) Nothing is wrong — any key makes POST safe
- B) The server rejects every attempt after the first
- C) Each attempt looks like a new operation to the server, so it can't tell attempt #2 is the same as #1 — double charges are still possible
- D) The UUID makes POST idempotent by itself

> ✅ **Answer: C.** The key's whole point is to let the server "tell attempt #2 was the same as attempt #1". That only works if every attempt of one operation carries the **same** key.

**Q22.** Which request is safe to retry **blindly** after a timeout, according to the method table?
- A) `POST /payments`
- B) `POST /orders` without a key
- C) `POST /cart/items`
- D) `PUT /users/7/email` with body `{"email": "a@b.com"}`

> ✅ **Answer: D.** PUT is "set it to X" — doing it twice still leaves X. All the POSTs are non-idempotent and need an idempotency key before they can be retried.

**Q23.** A payment-gated stream start calls an entitlement service. Case 1: the call **times out**. Case 2: it returns **200 `{"entitled": false}`**. What is the right policy?
- A) Retry case 1 only; case 2 is a business decision, so stop (e.g. show a subscribe screen)
- B) Retry both — neither started the stream
- C) Retry neither — entitlement is structural
- D) Retry case 2, because the body says it failed

> ✅ **Answer: A.** In case 2 the HTTP layer succeeded and the **business layer made a decision**, so asking again gives the same "no". A timeout means nothing was decided, so it is worth retrying.

**Q24.** A device with only Widevine **L3** asks for the license of a title that needs **L1**, and the license server denies it. What should the player do?
- A) Retry the license request with backoff + jitter
- B) Fall back to a lower DRM security level or a different key system
- C) Retry three times, then raise fatal error 8018
- D) Wait for `Retry-After` and try again

> ✅ **Answer: B.** "Retrying the same license request returns the same denial." The denial is structural, so the answer is a fallback, not a retry. The coded fatal error (C) is for repeated segment/manifest network failures.

**Q25.** A cache holds an expired copy with `stale-if-error` set, and the origin cannot be reached. What happens, and how does the lecture classify it?
- A) The cache retries the origin with backoff until it gets a fresh copy
- B) The cache returns 504, because a stale copy must never be served
- C) The cache keeps serving its last good copy — not a retry, but a decision to serve known-stale data rather than nothing
- D) The cache fails over to another region's origin

> ✅ **Answer: C.** `stale-if-error` (RFC 5861) lets the cache keep serving the last good copy when it can't revalidate. Nothing is sent again, so it is not a retry. It is the "slightly-wrong data beats nothing" lesson.
