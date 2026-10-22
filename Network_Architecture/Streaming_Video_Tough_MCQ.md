# Streaming Video at Large Scale — Toughest MCQs (Lesson 8)

> Try each question first, then check the answer right below it.

---

### 1. TTL Stacking with Jitter
Mid-tier TTL = 8 s, edge TTL = 6 s, and the client polls every 20 s with ±20% jitter. What is the worst-case staleness a viewer can see?
- A. 34 s
- B. 38 s
- C. 28 s
- D. 24 s

> ✅ **Answer: B. 38 s** — Worst case = longest poll gap + every TTL. The jittered poll can stretch to 20 × 1.2 = 24 s, so 24 + 8 + 6 = 38 s.

---

### 2. Little's Law — Pull vs Push
30M viewers poll through a CDN every 20 s, and each request spends 0.04 s at the edge. How many requests are in flight at the edge, and how many sockets would push hold for the same audience?
- A. 1.5M in flight vs 30M sockets
- B. 60,000 in flight vs 1.5M sockets
- C. 60,000 in flight vs 30M sockets
- D. 1.2M in flight vs 30M sockets

> ✅ **Answer: C.** λ = 30M / 20 = 1.5M/s, and L = 1.5M × 0.04 = 60,000. Push holds one socket per viewer for the whole match (W = hours), so 30M.

---

### 3. Load-testing Held Sockets
You want 400,000 test connections against a single server IP:port. The client uses Linux's default ephemeral range (32768–60999). What is the minimum number of client IPs?
- A. 7
- B. 14
- C. 15
- D. 13

> ✅ **Answer: C. 15** — 60999 − 32768 + 1 = 28,232 ports per client IP per destination. 400,000 / 28,232 = 14.17, which rounds up to 15.

---

### 4. Keepalive Tax
20M phones are idle on MQTT with a keepalive of 80 s. How many PINGREQs per second hit the brokers before the first ball?
- A. 250,000/s
- B. 333,000/s
- C. 160,000/s
- D. 1.6M/s

> ✅ **Answer: A. 250,000/s** — 20M / 80 s = 250k pings a second, all of them "keepalives with nothing to say".

---

### 5. Master Playlist BANDWIDTH
Using the class rule, a rendition has 2000k video + 128k audio. What BANDWIDTH goes in `#EXT-X-STREAM-INF`?
- A. 2,128,000
- B. 2,340,800
- C. 2,200,000
- D. 2,328,000

> ✅ **Answer: B. 2,340,800** — (2000k + 128k) × 1.1 = 2,128,000 × 1.1 = 2,340,800.

---

### 6. -g vs Frame Rate
The ladder command uses `-g 120 -keyint_min 120 -sc_threshold 0 -hls_time 4` on a **25 fps** source. What segment length appears in `#EXTINF`?
- A. 4.0 s
- B. 4.8 s
- C. 5.0 s
- D. 8.0 s

> ✅ **Answer: B. 4.8 s** — 120 frames / 25 fps = 4.8 s between keyframes. The cut lands on the first keyframe at or after 4 s, which is 4.8 s.

---

### 7. The Honest Benchmark
`ffmpeg -re -i in.mp4 -codec: copy -hls_time 6 -f hls index.m3u8`. The source is a 25 fps file encoded with x264's default keyframe interval. What segment length do you get?
- A. 6 s
- B. 8.33 s
- C. 12 s
- D. 10 s

> ✅ **Answer: D. 10 s** — `copy` can only cut at existing keyframes. x264's default GOP is 250 frames, which is 10 s at 25 fps (the 8.33 s from class was at 30 fps).

---

### 8. RFC 8216 Segment Retention
Target duration = 6 s and `-hls_list_size 8`. RFC 8216 says a removed segment MUST stay available for its duration plus the playlist's duration. What is the smallest `-hls_delete_threshold` that satisfies this?
- A. 1
- B. 8
- C. 9
- D. 12

> ✅ **Answer: C. 9** — 6 + (8 × 6) = 54 s must be covered, and 54 / 6 = 9 segments. The class used 12 for 4 + 40 = 44 s, which included some margin.

---

### 9. Playlist Reload Rules
A player reloads a live media playlist. `MEDIA-SEQUENCE` is the same as last time, but the URI at that sequence number is different. Per RFC 8216, the player:
- A. Waits half a target duration and reloads
- B. Waits one full target duration and reloads
- C. Switches to a lower rendition
- D. SHOULD halt playback — the playlist is lying

> ✅ **Answer: D.** Same sequence number with a different URI means segment N used to be something else. RFC 8216 says the player SHOULD halt.

---

### 10. Latency Budget
6 s segments, encode ≈ 2 s, CDN ≈ 2 s, and standard player hold-back. How far behind the stadium is the viewer?
- A. ≈ 28 s
- B. ≈ 22 s
- C. ≈ 16 s
- D. ≈ 34 s

> ✅ **Answer: A. ≈ 28 s** — 2 (encode) + 6 (segment fills) + 2 (CDN) + 3 × 6 (hold-back) = 28 s. Hold-back is the biggest bar, and LL-HLS shrinks it.

---

### 11. Break-even Object Size
A CDN charges $0.0100 per 10,000 requests and $0.10/GB. Below roughly what object size does the request fee dominate the bill?
- A. 1 KB
- B. 10 KB
- C. 100 KB
- D. 1 MB

> ✅ **Answer: B. 10 KB** — The fee per request is $0.000001. Size = $0.000001 / $0.10 per GB = 10⁻⁵ GB ≈ 10 KB.

---

### 12. Playlist RPS at Peak
40M peak viewers and target duration 6 s. Players follow the RFC (reload every target duration when the playlist changes). What is the peak media-playlist request rate?
- A. 40M req/s
- B. 13.3M req/s
- C. 6.67M req/s
- D. 1.33M req/s

> ✅ **Answer: C. 6.67M req/s** — 40M / 6 s. A playlist is billed per request just like the scorecard, which is why playlists are the hard part.

---

### 13. Spot the Dangerous Cache Rule
Which of these is the WORST rule for a live HLS stream (target duration 4 s)?
- A. `seg_*.ts` → 200 cached for 10 minutes
- B. `master.m3u8` → cached for 5 minutes
- C. media `index.m3u8` → TTL 2 s, but `stale-if-error` for 60 s
- D. 503 from origin → cached for 2 s

> ✅ **Answer: C.** A stale index served for 60 s lists segments that `delete_segments` already removed, so players get 404, the 404 gets cached, and playback stalls. A playlist should be served stale for one target duration at most.

---

### 14. Collapsing Silently Disabled
20,000 phones miss `score.json` at the same edge in 50 ms. The origin's framework adds a `Set-Cookie` header. On Fastly, what happens?
- A. The waiters are released one by one to the origin, and the object is marked hit-for-pass (collapsing skipped) for ~2 minutes
- B. All 19,999 waiters get the first response, but the cookie is stripped
- C. The edge returns 503 to the waiters
- D. Nothing changes — collapsing ignores response headers

> ✅ **Answer: A.** Collapsing only works when the first answer can be shared. A private response or `Set-Cookie` dequeues the waiters serially, and Fastly skips collapsing for that object for ~2 min.

---

### 15. Versioned Polling Done Right
To let late clients catch up on missed balls, which pull design keeps the CDN hit rate high?
- A. `score.json?since=<version>` so each client gets only its diff
- B. `score.json?user=<id>` so the CDN can track each client
- C. Long polling `score?after=<version>` through the CDN
- D. One URL whose body contains the current version plus the last few changes

> ✅ **Answer: D.** The query string is part of the cache key, so `?since=` gives every client its own miss. One shared file with recent history works exactly like a sliding HLS playlist.

---

### 16. Framing — Both Answers at Once
Session 2 said a message can be framed by **length** or by **delimiter**. Which format uses BOTH a fixed length and a sync marker?
- A. Server-Sent Events
- B. MPEG-TS
- C. WebSocket frames
- D. MQTT packets

> ✅ **Answer: B. MPEG-TS** — Every packet is exactly 188 bytes (length) and starts with the 0x47 sync byte (marker). SSE uses only a delimiter, while WebSocket and MQTT use only a length.

---

### 17. Why WebSocket Client Frames Are Masked
Why must every client-to-server WebSocket frame be masked?
- A. To encrypt the payload when TLS is not used
- B. To compress repeated payloads
- C. So attacker-controlled bytes can never look like an HTTP request to a transparent caching proxy (cache poisoning)
- D. So the server can tell text frames from binary frames

> ✅ **Answer: C.** Masking stops a script from crafting bytes like `GET /evil` that a transparent proxy might cache. It isn't encryption.

---

### 18. WebSocket Frame Size
A browser sends a single, unfragmented 200-byte text message over WebSocket. What is the total frame size on the wire (excluding TCP/IP)?
- A. 202 bytes
- B. 204 bytes
- C. 206 bytes
- D. 208 bytes

> ✅ **Answer: D. 208** — 1 (FIN/opcode) + 1 (MASK/len = 126) + 2 (16-bit extended length, since 200 > 125) + 4 (mask key) = 8 bytes of header, plus 200 bytes of payload.

---

### 19. Valid MPEG-TS Segment
Which of these file sizes could be a correctly written `.ts` segment?
- A. 940,000 bytes
- B. 1,000,000 bytes
- C. 823,000 bytes
- D. 512,000 bytes

> ✅ **Answer: A. 940,000** — A segment must be an exact multiple of 188, and 940,000 = 188 × 5,000. None of the others divide evenly.

---

### 20. Throughput-based ABR
The last segment (1.2 MB) took 4 s to download. The player applies a 0.8 safety factor. The ladder's BANDWIDTHs are 3,440,800 / 1,755,600 / 950,400. Which rendition does it pick next?
- A. 720p
- B. 480p
- C. 360p
- D. It stalls until the buffer refills

> ✅ **Answer: B. 480p** — 1.2 MB × 8 / 4 s = 2.4 Mbps, and × 0.8 = 1.92 Mbps. 720p (3.44M) doesn't fit, so it picks the highest that does: 480p (1.76M).

---

### 21. S3 Hot Key
22M viewers poll one `score.json` directly from S3 every 20 s. S3 promises ≥ 5,500 GET/s per prefix. What happens, and why doesn't spreading across prefixes fix it?
- A. Fine — S3 auto-scales instantly to 1.1M GET/s
- B. ~20× over, fixed by adding 20 prefixes
- C. ~200× over, and 503 Slow Down; one key lives under one prefix, so it can't be sharded
- D. ~200× over, but the GET fee is so low that it doesn't matter

> ✅ **Answer: C.** 22M / 20 = 1.1M GET/s, which is 200× the per-prefix limit. Spreading across prefixes doesn't help because it is one hot key under one prefix.

---

### 22. What a 304 Doesn't Save
A team adds ETags to the 1 KB scorecard, expecting its CDN bill to drop ~90%. Most polls now return 304. What actually happens to the bill?
- A. It drops ~90%, because the body is no longer sent
- B. It drops to zero, because a 304 is free
- C. It goes up, because 304s can't be cached
- D. It barely moves — ~92% of the bill is per-request fees, and the request still happens

> ✅ **Answer: D.** A 304 saves the body, the parse and the render, but not the request fee, the round trip or the headers. For a 1 KB object, the request fee is ~92% of the bill.

---

### 23. LL-HLS Mapping — Find the WRONG One
Which mapping of an LL-HLS feature to an earlier lecture idea is WRONG?
- A. Parts ↔ HTTP/2 Server Push
- B. Blocking playlist reload (`_HLS_msn`) ↔ long polling
- C. Preload hints ↔ request collapsing / 103 Early Hints
- D. Delta updates (`EXT-X-SKIP`) ↔ sending diffs instead of the whole card

> ✅ **Answer: A.** Parts are just smaller pieces. HTTP/2 Server Push was the original LL-HLS delivery method that **preload hints replaced** (Jan 2020).

---

### 24. Spoiler Fix Arithmetic
The segment on screen has `PROGRAM-DATE-TIME 03:16:40.000`. It starts at 120.0 s on the player timeline, and the playhead is at 122.5 s. Which score update may be shown now?
- A. `"at": "03:16:43.0"`
- B. `"at": "03:16:42.1"`
- C. `"at": "03:16:44.1"`
- D. None — wait for the next segment

> ✅ **Answer: B.** Wall clock on screen = 03:16:40.000 + (122.5 − 120.0) = 03:16:42.5. Only updates with `at` ≤ 42.5 may be shown, so 42.1 is the only one.

---

### 25. EVENT Playlists
`-hls_playlist_type event -hls_list_size 5 -hls_time 4` runs for 60 s. How many segments are listed at the end?
- A. 5, as a sliding window
- B. 10, because EVENT doubles the window
- C. All of them (~15) — EVENT playlists are append-only and ignore the window
- D. 0, because EVENT writes `#EXT-X-ENDLIST` immediately

> ✅ **Answer: C.** An EVENT playlist is append-only, so `hls_list_size` is ignored and 60 / 4 ≈ 15 segments are listed. For a sliding window, leave out `-hls_playlist_type`.
