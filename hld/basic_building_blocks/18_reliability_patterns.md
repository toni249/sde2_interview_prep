# 18 — Reliability Patterns: Circuit Breaker, Retries, Bulkhead, Idempotency

> Distributed systems fail. These patterns let you fail *well*: contain damage, recover automatically, keep the system alive during storms. Combine them in every service.

## The Failure Modes

- **Slow dependency**: response takes 30 s instead of 100 ms → threads pile up → OOM.
- **Failing dependency**: 100% 500s → retries flood → dependent service melts.
- **Cascading failure**: one slow service takes down its callers → they take down theirs.
- **Retry storm**: everyone retries at the same instant.
- **Metastable failure**: system enters degraded state, can't recover without intervention.

Reliability patterns are pre-designed answers to these.

## Retry with Exponential Backoff + Jitter

Most transient failures resolve if you wait. But naive retry destroys.

**Rules:**
- Retry only on **transient** errors (5xx, timeouts, connection reset). Never on 4xx (bug on your side).
- **Cap** retries (3-5 usually enough).
- **Exponential backoff**: `delay = base × 2^attempt`.
- **Jitter**: `delay = base × 2^attempt + random(0, base)` — prevents synchronized retries.
- **Budget**: total time across retries bounded (`totalTimeout`).

```java
int attempt = 0;
long base = 100;   // ms
while (attempt < 5) {
    try {
        return call();
    } catch (RetryableException e) {
        long delay = base * (1L << attempt) + ThreadLocalRandom.current().nextLong(base);
        Thread.sleep(Math.min(delay, 10_000));
        attempt++;
    }
}
throw new ExhaustedException();
```

**AWS SDK, gRPC, HTTP clients** have this built-in — configure, don't reimplement.

**Retry storm math**: 100 clients × 5 retries × 3 hops deep = 1500 requests from one user-visible failure. Retries must be budgeted globally.

## Circuit Breaker

Modeled after electrical fuses.

**States:**
- **Closed**: calls flow through; failures counted.
- **Open**: threshold exceeded → immediately fail all calls without hitting downstream.
- **Half-Open**: after cool-down, allow one trial call → if success, close; if fail, re-open.

```
    ┌─── closed ───┐
    │      │       │
success   fail     │
    ▲     count    │
    │       │      │
    │       ▼      │
    │  threshold?  │
    │       │      │
    │  yes  ▼      │
    │       │      │
    │   open ──────┤
    │      │       │
    │  timeout     │
    │      ▼       │
    │  half-open   │
    │  trial call  │
    │   fail       │
    │      │       │
    │      ▼───────┘
    │   
    └─success── closed
```

**Config:**
- `failureRateThreshold`: e.g., 50% failures over last 100 calls trip.
- `slowCallDurationThreshold`: also trip on slow calls (not just errors).
- `waitDurationInOpenState`: how long before trial (default 60 s).
- `permittedNumberOfCallsInHalfOpenState`: 3-10 trial calls.

**Where to place:**
- **Client-side** (in the caller): trips fast, immediate fallback.
- **Sidecar/mesh** (Envoy, Istio): declarative, no code change.
- **Both**: mesh for defaults, client for specialized fallbacks.

**Libraries**: Netflix Hystrix (deprecated but influential), Resilience4j, Polly (.NET), Go/Java kits.

## Bulkhead

Ship analogy — one flooded compartment doesn't sink the ship.

**Isolate resources per dependency** so a hung dependency doesn't consume all your threads/connections.

```
Thread pool for PaymentService: 20 threads
Thread pool for SearchService:  20 threads
Thread pool for AuthService:    10 threads
```

If PaymentService hangs, its 20 threads block, but SearchService still works.

**Types:**
- **Thread pool bulkhead**: fixed pool per dependency.
- **Semaphore bulkhead**: N concurrent calls allowed; simpler, no extra threads.
- **Connection pool per dependency** (DB, HTTP client).

Combines with circuit breaker: bulkhead limits *concurrent* damage, breaker limits *repeated* damage.

## Timeout

Every network call must have a timeout.

**Rules:**
- **Client timeout < server timeout < LB timeout**: prevents zombie work continuing after client gave up.
- **Deadline propagation** (gRPC): each hop inherits and can shorten the deadline.
- **Total request timeout** (5 s) different from per-attempt (1 s) — retries fit inside.

**Antipattern**: no timeout → thread blocks forever → thread pool exhaustion → cascading failure.

## Idempotency

Any operation the client might retry must be safe to run twice.

**Techniques:**
- **Idempotency key** header: server dedupes by key (Stripe pattern).
```
POST /charge   Idempotency-Key: 7c3d...
```
- **Conditional writes**: `INSERT ... ON CONFLICT DO NOTHING`, `PUT If-None-Match`.
- **Natural idempotency**: `SET x=10` vs `INCR x`. Prefer SET.
- **Result table**: store `key → result`; on retry return same result.

**Idempotency key TTL**: 24 h is common. Store it in Redis + backing DB.

Not all HTTP methods are naturally idempotent — PUT, DELETE, GET are (per HTTP spec); POST, PATCH are not by default.

## Fallback

When downstream is broken and circuit is open, what do you return?

- **Static default**: empty list, cached last-known-good value.
- **Degraded feature**: return without recommendations, without pricing, without images.
- **Fail hard**: error to caller. Only if failure is safer than incorrect data.

Design fallbacks for **every** critical dependency in advance.

## Health Checks

- **Liveness**: am I alive? Any answer → yes. Restart on failure.
- **Readiness**: am I ready to serve traffic? Include downstream health.
- **Deep**: check downstream — but risk cascading failure if downstream blips.
- **Shallow**: just check process. Safer default; use deep sparingly for specific readiness.

Kubernetes distinguishes liveness/readiness probes; use both.

## Graceful Degradation

Prioritize what's critical:
- Payment must work; recommendations can go dark.
- Search must return results; personalization can be generic.
- Reads must work; writes may reject temporarily.

Explicit degrade modes with flags → operator can turn off features to save the core.

## Load Shedding

When overloaded, drop *some* traffic to save the rest.

**Prioritization:**
- Reject low-priority (`analytics/`, `search/*`) first.
- Keep high-priority (`checkout/`, `login/`).
- Reject unauthenticated requests before authenticated.

**Techniques:**
- Return 503 with `Retry-After` for shed requests.
- Adaptive: shed when queue depth or p99 latency exceeds threshold.
- Netflix "adaptive concurrency" — dynamically finds the sweet spot.

## Chaos Engineering

Proactively inject failures to verify reliability:
- Kill random pods (Chaos Monkey).
- Inject network latency (Toxiproxy, Gremlin).
- Blackhole a dependency.
- Fill disk.
- Slow down DB.

Run in staging weekly; graduate to prod (with safety) — Netflix runs Chaos Monkey in prod.

## Combined — Defense-in-Depth Example

Service A calling Service B:

```
                Retry (3, backoff+jitter)
                    │
                Circuit Breaker (50% fail rate → open 60s)
                    │
                Bulkhead (20-thread pool for B)
                    │
                Timeout (500 ms per call, 1500 ms total)
                    │
                Fallback (cached last-known-good)
                    │
                    ▼
             Service B (with idempotency check)
```

Every layer catches a different failure mode.

## Observability for Reliability

- **Metrics per dependency**: success rate, latency p50/p99, circuit breaker state.
- **Distributed tracing**: which hop caused the timeout.
- **Structured logs**: `dep=payment status=open reason=high_error_rate`.
- **Alerts**: breaker stays open > 5 min, thread pool at 100% utilization, retry rate > 10%.

## Interview Q&A

**Q: Why is retry alone insufficient?**
- Blind retry on a struggling downstream amplifies the outage.
- Need circuit breaker to stop retrying once broken, and bulkhead to prevent the caller from being consumed.

**Q: Where do you put the circuit breaker — client or server?**
- Client-side, in the caller. The point is to *not* call a broken downstream. A server-side breaker (on B) can't stop A from calling.

**Q: What's the difference between timeout and circuit breaker?**
- Timeout: per-call limit.
- Circuit breaker: stops calls entirely after threshold, giving downstream time to recover.
- Complementary, not alternative.

**Q: How to design idempotent create?**
- Client-supplied `Idempotency-Key: uuid` header.
- Server stores `(key → result)` with TTL.
- On retry with same key: return stored result without re-executing.

**Q: What's a metastable failure?**
- System enters a degraded state that persists even after the initial trigger is removed. Example: retries create load, load causes latency, latency causes retries. Won't recover without dropping load. Fix: **load shedding**, backpressure, coordinated retry limits.

**Q: How to handle a dependency being slow (not down)?**
- Timeout aggressively.
- Circuit breaker on slow-call threshold (`slowCallDurationThreshold`).
- Bulkhead prevents slow calls from consuming all threads.
- Hedging (send duplicate request to another replica after p50 latency; use first response).

**Q: What is hedging?**
- After waiting p50 latency, send a second request in parallel. Use whichever returns first. Cancel the loser.
- Reduces tail latency at the cost of extra requests.
- Google BigTable, Envoy support this.

**Q: How do you decide retry vs fallback vs fail?**
- Retry: probably-transient (timeout, 503, network blip). Limited attempts.
- Fallback: dependency essential but downtime possible; return safe default.
- Fail: no meaningful fallback; propagate error with context.

**Q: What is graceful degradation vs failover?**
- Degradation: keep the service up with reduced functionality.
- Failover: switch to a healthy instance/region entirely.
- Real prod uses both.

## Real-World Examples

- **Netflix**: Hystrix pioneered the pattern; every service call is wrapped.
- **AWS**: SDKs implement exponential backoff + jitter; documented "AWS retry mode".
- **Stripe**: idempotency keys on every write endpoint; industry standard.
- **Google SRE**: coined "error budget", "graceful degradation", "load shedding" terminology.
- **Envoy / Istio**: circuit breaking, retry, timeout, outlier detection as sidecar config.

## Gotchas
- **Timeout too high**: everything else waits for it — thread pool fills.
- **Timeout too low**: legitimate slow-but-successful requests killed.
- **Retry without idempotency**: duplicate side effects (double charge).
- **No jitter**: 1000 clients all retry at second 5 → thundering herd.
- **Circuit breaker on transient error only**: business errors (4xx) shouldn't trip it.
- **Fallback that itself calls downstream**: fallback fails when the primary fails. Fallbacks must be local (cache, static).
- **Not testing degraded modes**: assumed fallback works but never verified. Chaos-test it.
- **Total retry budget across microservices**: A calls B calls C, all retry 3x → 27 total tries for one request. Compose limits.
