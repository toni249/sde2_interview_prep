# 02 — HTTP Evolution

> HTTP is the lingua franca of the web. Each version fixed a latency bottleneck of the previous. Know what each fixes and what still hurts.

## Version Timeline

| Version | Year | Transport | Killer Feature |
|---------|------|-----------|----------------|
| HTTP/0.9 | 1991 | TCP | GET only, no headers |
| HTTP/1.0 | 1996 | TCP | Headers, status codes, one request per connection |
| HTTP/1.1 | 1997 | TCP | Keep-alive, chunked encoding, `Host` header (virtual hosting) |
| HTTP/2 | 2015 | TCP + TLS | Binary framing, multiplexing, header compression |
| HTTP/3 | 2022 | UDP (QUIC) | 0-RTT, no HoL blocking, connection migration |

## HTTP/1.1 — The Baseline

**Request/response over TCP**:
```
GET /api/users HTTP/1.1
Host: example.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42

{"id":1,"name":"Alice"}
```

Key features:
- **Keep-alive** (default in 1.1): reuse TCP connection for multiple requests.
- **Pipelining**: send multiple requests without waiting for response — rarely worked well in practice due to HoL blocking; disabled in most browsers.
- **Chunked transfer encoding**: send body in chunks when total length unknown (`Transfer-Encoding: chunked`).
- **Virtual hosting**: `Host` header lets one IP serve many domains.

**Pain points:**
- Head-of-line blocking: one slow response blocks all after it on the same connection.
- Browsers open 6 parallel connections per origin as a workaround → costly handshakes.
- Text protocol → parser complexity.
- Repetitive headers on every request (cookies alone can be KB).

## HTTP/2 — Multiplexing over One TCP Connection

Same semantics (methods, headers, status), new wire format.

**Key mechanisms:**
- **Binary framing**: everything is a frame with a stream ID; parser is simpler and faster.
- **Multiplexing**: multiple requests/responses interleave on one TCP connection. No more 6-connection workaround.
- **HPACK header compression**: static + dynamic table dedupes headers across requests. Big win for cookies, auth tokens.
- **Server push**: server sends resources before client asks (deprecated in browsers now — bad cache interaction).
- **Stream prioritization**: browser signals which streams are urgent.

**Still hurts**: TCP head-of-line blocking. If one packet is lost, the entire connection stalls because TCP must deliver in order — even though only one HTTP stream cares about that byte.

## HTTP/3 — QUIC over UDP

Fixes what HTTP/2 couldn't: TCP itself.

**QUIC = a new transport that runs on UDP**, providing:
- Multiplexed streams with **independent loss recovery** (a lost packet on stream A doesn't stall stream B).
- Built-in TLS 1.3 → handshake merged with transport handshake.
- **0-RTT resumption**: client can send app data on the very first packet to a known server (replay-safe methods only).
- **Connection migration**: connection survives when the client's IP changes (mobile → Wi-Fi handoff). Tied to a connection ID, not the 5-tuple.

Downsides:
- UDP is throttled or blocked by some firewalls.
- Higher CPU cost per packet (crypto per-packet vs TCP's per-connection).
- Server ecosystem still maturing (2026-era support is good — Cloudflare, Google, Meta all HTTP/3 in prod).

## The Full Handshake Comparison

```
HTTP/1.1 over TLS 1.2:  DNS + TCP(1) + TLS(2) + Request(1) = ~4 RTTs
HTTP/2 over TLS 1.2:    DNS + TCP(1) + TLS(2) + Request(1) = ~4 RTTs (but reused for many reqs)
HTTP/2 over TLS 1.3:    DNS + TCP(1) + TLS(1) + Request(1) = ~3 RTTs
HTTP/3 (QUIC):          DNS + QUIC(1) + Request                 = ~2 RTTs
HTTP/3 with 0-RTT:      DNS + Request (on first QUIC packet)    = ~1 RTT
```

## Methods & Semantics

| Method | Safe (no side effect) | Idempotent | Body |
|--------|-----------------------|------------|------|
| GET | ✅ | ✅ | No |
| HEAD | ✅ | ✅ | No |
| OPTIONS | ✅ | ✅ | Sometimes |
| PUT | ❌ | ✅ (replace) | Yes |
| DELETE | ❌ | ✅ | No |
| POST | ❌ | ❌ | Yes |
| PATCH | ❌ | ❌ (usually) | Yes |

**Idempotent** = same effect no matter how many times you call it. Critical for retry logic.

## Status Codes You Must Know

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Success |
| 201 | Created | POST created a resource |
| 204 | No Content | Success, empty body (DELETE) |
| 301 | Permanent redirect | URL changed forever |
| 302 / 307 | Temporary redirect | 307 preserves method |
| 304 | Not Modified | Conditional GET, use cache |
| 400 | Bad Request | Client sent malformed data |
| 401 | Unauthorized | No/invalid auth |
| 403 | Forbidden | Auth valid, not allowed |
| 404 | Not Found | Resource missing |
| 409 | Conflict | State conflict (concurrent edit) |
| 429 | Too Many Requests | Rate limited |
| 500 | Internal Server Error | Bug/crash |
| 502 | Bad Gateway | Upstream unreachable |
| 503 | Service Unavailable | Overloaded / maintenance |
| 504 | Gateway Timeout | Upstream too slow |

## Caching Headers

| Header | Purpose |
|--------|---------|
| `Cache-Control: max-age=3600` | Cache for 1 hour |
| `Cache-Control: no-store` | Never cache |
| `Cache-Control: private` | Only browser, not CDN |
| `Cache-Control: public, s-maxage=86400` | CDN cache for 1 day |
| `ETag: "abc123"` | Version tag; conditional GET returns 304 |
| `Last-Modified` | Older weak ETag equivalent |
| `Vary: Accept-Encoding` | Cache separately per header value |
| `stale-while-revalidate=60` | Serve stale while refetching |

**Conditional GET**:
```
GET /file       If-None-Match: "abc123"
→ 304 Not Modified (body omitted)  OR  200 with new body + new ETag
```

## Interview Q&A

**Q: Why is HTTP/2 still susceptible to HoL blocking?**
- Multiplexing is at the *HTTP* layer. Below it, TCP requires in-order byte delivery, so one dropped packet stalls all streams sharing that TCP connection.

**Q: Should we use HTTP/3 in production?**
- Yes for edge/CDN traffic — measurable improvement on mobile networks.
- Internal service-to-service is fine on HTTP/2 (low packet loss inside a DC).

**Q: What's the difference between PUT and PATCH?**
- PUT replaces the entire resource with the body — idempotent.
- PATCH applies a partial change — usually not idempotent (e.g., `{"balance": "+10"}`).
- For updates, PATCH is more common; use PUT when you send the full new state.

**Q: Should POST or PUT be used for create?**
- POST when the server assigns the ID (`POST /users` → `201 Location: /users/42`).
- PUT when the client picks the ID (`PUT /users/42` — replace or create).

**Q: What's connection coalescing in HTTP/2?**
- If two hostnames resolve to the same IP and cert covers both, browser reuses one connection. Reduces handshakes.

**Q: What is `Vary` and why does it matter?**
- Tells the cache to key by additional headers. `Vary: Accept-Language` → English and French versions cached separately. Wrong `Vary` → cross-user cache poisoning or 0 hit rate.

**Q: How does HTTPS work end to end?**
1. DNS resolves domain.
2. TCP handshake (or QUIC handshake).
3. TLS handshake: cert verification, key exchange, symmetric key derived.
4. Encrypted HTTP flows over that key.
5. Connection kept alive with keep-alive; TLS session ticket enables 0-RTT resume next time.

## Gotchas
- **Case-insensitive headers**: `content-type` == `Content-Type`. Watch out with strict frameworks.
- **Cookies aren't scoped by port** (only host + path). Two services on same host share cookie jar.
- **`Content-Length` mismatch** vs actual body → server may close connection or trigger request smuggling attack.
- **HTTP → HTTPS redirect** exposes first request in cleartext. Use HSTS to force HTTPS from browser.
- **HTTP/2 in prod**: some LBs speak HTTP/2 to clients but HTTP/1.1 to backends → keep-alive tuning matters.
