# 17 — Rate Limiting

> A hard cap on request volume — per user, per API key, per IP, per plan. Protects your service from abuse, misbehaving clients, and thundering herd. Pairs with [LLD_06_Rate_Limiter](../../java-lld/LLD_06_Rate_Limiter.md) for implementation depth.

## Why Rate Limit

- **Prevent abuse** (scraping, credential stuffing, DDoS).
- **Enforce plans** (free tier vs enterprise).
- **Protect downstream** (DB pool, external APIs).
- **Fair sharing** (one noisy tenant doesn't hog capacity).
- **Cost control** (LLM APIs are expensive per token).

## Algorithms

### 1. Token Bucket

Bucket holds up to `B` tokens; refills at rate `R` tokens/sec.

Each request removes 1 token; refuse if empty.

```
capacity = 100
refill_rate = 10 tokens/sec

Bucket:  [██████████]  100 tokens
Request → -1 token   [████████░░]  99 tokens
No request for 1s → +10   [████████░░] → [█████████░]
```

- Allows **bursts** up to `B`, sustained at `R`.
- Simple, memory-efficient (one number + timestamp per key).
- **Most common in practice** (AWS, Stripe, Cloudflare).

### 2. Leaky Bucket

Requests fill a queue; queue drains at fixed rate `R`. Overflow → reject.

- Smooths bursts — output is always at rate `R`.
- Adds latency (queued requests wait).
- Used for **shaping** outbound traffic to strict APIs.

### 3. Fixed Window Counter

Counter resets every window (`10:00-10:01`, `10:01-10:02`, …).

```
if requests_in_current_minute > 100: reject
else: increment
```

- Simple; **spiky at window boundaries** — a client can send 100 at 10:00:59 and 100 at 10:01:00 = 200 in 2 seconds.

### 4. Sliding Window Log

Store timestamp of every request in a sorted set. On each new request:
- Remove entries older than the window.
- Count remaining. If ≥ limit → reject.

- Accurate.
- Memory: 1 timestamp per request. Expensive for high-QPS clients.

### 5. Sliding Window Counter

Approximates sliding window with two fixed-window counters.

```
now         = 10:00:30
window_size = 60s

count = requests_in_current_window * (elapsed / window)
      + requests_in_previous_window * (1 - elapsed / window)

At 10:00:30, current window elapsed = 30/60 = 50%
Effective = 0.5 * current + 0.5 * previous
```

- **Best balance** of accuracy and memory.
- Used by Cloudflare's "sliding window rate limiter".

## Comparison

| Algorithm | Memory | Burst allowed | Accuracy | Complexity |
|-----------|--------|---------------|----------|------------|
| Token Bucket | O(1) per key | Yes (up to B) | Good | Low |
| Leaky Bucket | O(queue) | No | Good | Low-Mid |
| Fixed Window | O(1) per key | 2× at boundary | Poor at edges | Trivial |
| Sliding Window Log | O(N) per key | Configurable | Perfect | High |
| Sliding Window Counter | O(1) per key | Configurable | Very good | Low |

## Where to Enforce

**Client-side**: helpful but not a defense (client can be modified). Best for "please slow down" UX.

**Edge (CDN / API gateway)**: coarse limits (per IP, per API key). Cheap, keeps traffic off origin.

**Application**: fine-grained (per user, per operation). Enforce with distributed counter (Redis).

**Downstream (DB pool)**: implicit rate limit via connection cap; not user-visible.

**Multi-layer defense**: 5000 req/hour at edge (per IP) + 100 req/min per user at app + 50 conns to DB.

## Distributed Rate Limiting with Redis

For N app instances, all must share counter.

**Token Bucket in Redis (Lua atomic):**
```lua
-- KEYS[1] = user key, ARGV[1] = capacity, ARGV[2] = refill_rate, ARGV[3] = now
local tokens_key = KEYS[1] .. ":tokens"
local ts_key = KEYS[1] .. ":ts"

local last_tokens = tonumber(redis.call("GET", tokens_key)) or ARGV[1]
local last_ts = tonumber(redis.call("GET", ts_key)) or ARGV[3]

local delta = (ARGV[3] - last_ts) * ARGV[2]
local tokens = math.min(ARGV[1], last_tokens + delta)

if tokens < 1 then
  return 0  -- reject
else
  tokens = tokens - 1
  redis.call("SETEX", tokens_key, 3600, tokens)
  redis.call("SETEX", ts_key, 3600, ARGV[3])
  return 1  -- allow
end
```

Atomic via Lua; single round trip.

## Multi-Dimensional Limits

Real APIs usually stack:
- Per IP: 1000/hour (defense against scanners).
- Per API key: 10k/day (billing tier).
- Per user: 100/min (fair use).
- Per operation: 10/min for expensive `POST /transcode`.
- Per resource: 5 concurrent modifications on `PATCH /doc/X`.

Any one of these can reject.

## Response Semantics

**HTTP 429 Too Many Requests** with headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1700000060
Retry-After: 30
```

- Communicates the limit so clients back off gracefully.
- `Retry-After` in seconds; some APIs use HTTP-date.

## Client Behavior — Exponential Backoff with Jitter

After 429, retry with:
```
sleep = min(cap, base * 2^attempt) + rand(0, base)
```

- Doubles delay each attempt (base=1s → 1, 2, 4, 8, ...).
- **Jitter** prevents synchronized retry storms (all clients retry at the same instant).

## Rate Limit vs Throttle vs Quota

- **Rate limit**: instantaneous rate (per second/minute).
- **Throttle**: slow down (queue, delay) instead of reject.
- **Quota**: absolute cap over a long window (e.g., 1M req/month).

Big APIs combine all three: burst limit + steady rate + monthly quota.

## Adaptive Rate Limiting

Static limits waste capacity when system is idle and drop requests when overloaded.

**Adaptive** (Netflix concurrency limits, AIMD):
- Watch p99 latency or error rate.
- Increase limit while healthy.
- Decrease (multiplicatively) when latency rises.

Frameworks: Netflix Hystrix, Resilience4j, Google's SRE token bucket + gradient adaptive.

## Reference — Public API Design

```
Free tier:   100 req/min, 10k req/day
Basic tier:  1000 req/min, 1M req/day
Enterprise:  custom limits, priority queue

Enforcement:
  Layer 1: Cloudflare — per-IP DDoS floor
  Layer 2: API Gateway — per API-key limits (Redis-backed)
  Layer 3: Service — per-user + per-endpoint

Response:
  429 + Retry-After
  X-RateLimit-* headers for observability
```

## Interview Q&A

**Q: Token bucket vs leaky bucket — which do you pick?**
- Token bucket: allows bursts, easier to reason about, more common.
- Leaky bucket: strict smoothing, useful when calling a downstream that hates spikes.

**Q: How to rate-limit in a distributed system?**
- Central store (Redis) with atomic counter operations (Lua, INCR + EXPIRE).
- Local counter with periodic sync — approximate but faster.
- Ephemeral counter at LB / edge for cheap coarse limits.

**Q: What if Redis goes down?**
- **Fail open** (allow all): keeps service up, exposes to abuse.
- **Fail closed** (reject all): protects downstream but breaks users.
- **Fallback local counter**: each instance limits at `total_limit / N` with margin. Works well.

**Q: How do you rate-limit anonymous users?**
- By IP (imperfect — NAT, CGNAT, mobile carriers share IPs).
- By fingerprint (user agent + IP + cookies) — better but privacy concerns.
- By CAPTCHA challenge after N requests.

**Q: How to prevent one big client from starving others?**
- Per-tenant limits ensure fairness.
- Priority queues + per-user concurrency caps (bulkhead pattern).

**Q: How do you rate-limit *background* traffic that shouldn't affect users?**
- Separate limits per tier or endpoint.
- Priority classification at the gateway.
- Reserved capacity for interactive traffic.

**Q: Client sends 1000 requests in parallel and gets rate-limited. Best client behavior?**
- Detect 429, back off with exponential + jitter, respect `Retry-After`.
- Cap concurrent in-flight requests at bounded number.
- Idempotency keys so retries are safe.

## Real-World Examples

- **Stripe API**: token bucket per API key; documented burst + rate limits.
- **GitHub API**: fixed hourly window with `X-RateLimit-*` headers.
- **Cloudflare**: sliding window at edge; multiple dimensions (IP, ASN, country, URL).
- **AWS API Gateway**: token bucket per method; burst + steady rate.
- **Twitter API**: multiple tiers, hourly caps.
- **OpenAI API**: RPM (requests/min) + TPM (tokens/min) — both must be respected.

## Gotchas
- **Fixed window at boundaries**: 2× limit for 2s. Use sliding window if this matters.
- **Clock skew** between app servers: rate limit windows drift. Use Redis server time (`TIME` command).
- **Race conditions**: check-then-set is not atomic across servers. Use Redis Lua / atomic ops.
- **Precision vs cost**: `INCR` per request scales; sorted set per request doesn't at 1M QPS.
- **User-facing vs machine-facing** limits should differ: bots retry aggressively, humans don't.
- **Rate-limit failure mode not tested**: prod incident when Redis latency spikes → all requests wait 5 s on rate-limit check → app slower than no-limit. Add timeout + fail-open fallback.
