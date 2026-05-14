# LLD: Rate Limiter

> Frequency: Very High | Difficulty: High
> Tests: Multiple algorithm knowledge, concurrency, trade-off analysis

---

## Step 1 — Clarifying Questions

- Per-user or per-IP or global rate limiting?
- What should happen when limit is exceeded? (reject, queue, degrade)
- Sliding window or fixed window?
- Distributed system or single machine?
- What's the granularity? (per second, per minute, per hour)

---

## Step 2 — Four Algorithms (Know All of Them)

```
1. Fixed Window Counter   — simplest, burst problem at window edges
2. Sliding Window Log     — accurate, high memory usage
3. Sliding Window Counter — memory-efficient approximation of sliding log
4. Token Bucket           — smooth rate + burst capacity
5. Leaky Bucket           — constant output rate, no burst
```

---

## Algorithm 1: Fixed Window Counter

```java
public class FixedWindowRateLimiter {
    private final int maxRequests;
    private final long windowMs;
    private long windowStart;
    private int requestCount;

    public FixedWindowRateLimiter(int maxRequests, long windowMs) {
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
        this.windowStart = System.currentTimeMillis();
        this.requestCount = 0;
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();

        // Reset window if expired
        if (now - windowStart >= windowMs) {
            windowStart = now;
            requestCount = 0;
        }

        if (requestCount < maxRequests) {
            requestCount++;
            return true;
        }
        return false;
    }
}
```

**Problem:**
```
Limit: 10 requests per minute
At t=0:59 → 10 requests (end of window 1)
At t=1:00 → window resets
At t=1:01 → 10 requests (start of window 2)
→ 20 requests in 2 seconds! Burst at window boundary.
```

---

## Algorithm 2: Token Bucket (Most Common Interview Answer)

**Concept:** A bucket holds tokens (capacity = burst limit). Tokens are added at a fixed rate. Each request consumes a token. If no tokens → reject.

```java
public class TokenBucketRateLimiter {
    private final long capacity;          // max tokens (burst limit)
    private final double refillRatePerMs; // tokens added per millisecond
    private double tokens;
    private long lastRefillTime;

    // Per-user map (for multi-user rate limiting)
    private final ConcurrentHashMap<String, UserBucket> userBuckets = new ConcurrentHashMap<>();

    public TokenBucketRateLimiter(long capacity, double tokensPerSecond) {
        this.capacity = capacity;
        this.refillRatePerMs = tokensPerSecond / 1000.0;
        this.tokens = capacity;
        this.lastRefillTime = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        refill();
        if (tokens >= 1.0) {
            tokens -= 1.0;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastRefillTime;
        double newTokens = elapsed * refillRatePerMs;
        tokens = Math.min(capacity, tokens + newTokens);
        lastRefillTime = now;
    }

    // ─── Per-user rate limiting ───
    private static class UserBucket {
        double tokens;
        long lastRefillTime;

        UserBucket(long capacity) {
            this.tokens = capacity;
            this.lastRefillTime = System.currentTimeMillis();
        }
    }

    public boolean allowRequest(String userId, long capacity, double tokensPerSecond) {
        UserBucket bucket = userBuckets.computeIfAbsent(userId, k -> new UserBucket(capacity));

        synchronized (bucket) {  // lock per user — fine-grained!
            long now = System.currentTimeMillis();
            double refillRatePerMs = tokensPerSecond / 1000.0;
            double newTokens = (now - bucket.lastRefillTime) * refillRatePerMs;
            bucket.tokens = Math.min(capacity, bucket.tokens + newTokens);
            bucket.lastRefillTime = now;

            if (bucket.tokens >= 1.0) {
                bucket.tokens -= 1.0;
                return true;
            }
            return false;
        }
    }
}
```

---

## Algorithm 3: Sliding Window Log

```java
public class SlidingWindowLogRateLimiter {
    private final int maxRequests;
    private final long windowMs;
    // Per-user deque of timestamps
    private final ConcurrentHashMap<String, Deque<Long>> userLogs = new ConcurrentHashMap<>();

    public SlidingWindowLogRateLimiter(int maxRequests, long windowMs) {
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
    }

    public boolean allowRequest(String userId) {
        long now = System.currentTimeMillis();
        long windowStart = now - windowMs;

        Deque<Long> log = userLogs.computeIfAbsent(userId, k -> new ArrayDeque<>());

        synchronized (log) {
            // Remove timestamps outside the window
            while (!log.isEmpty() && log.peekFirst() <= windowStart) {
                log.pollFirst();
            }

            if (log.size() < maxRequests) {
                log.addLast(now);
                return true;
            }
            return false;
        }
    }
}
```

**Problem:** Memory grows with request volume. 1M users × 100 requests/min = 100M timestamps.

---

## Algorithm 4: Sliding Window Counter

**Best of both worlds** — O(1) memory, approximate sliding window.

```java
public class SlidingWindowCounterRateLimiter {
    private final int maxRequests;
    private final long windowMs;
    private final ConcurrentHashMap<String, long[]> userCounters = new ConcurrentHashMap<>();
    // long[0] = current window count, long[1] = current window start time

    public SlidingWindowCounterRateLimiter(int maxRequests, long windowMs) {
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
    }

    public boolean allowRequest(String userId) {
        long now = System.currentTimeMillis();
        long[] data = userCounters.computeIfAbsent(userId, k -> new long[]{0, now});

        synchronized (data) {
            long elapsed = now - data[1];  // time since window start
            double overlap = 1.0 - (double) elapsed / windowMs;  // fraction of prev window still relevant

            // Weighted count: fraction of previous window + current window count
            double weightedCount = overlap * data[0]; // simplified — real impl keeps two windows
            // For full sliding window: keep prev_count and curr_count with window rotation

            // Simple implementation: if window expired, reset
            if (elapsed >= windowMs) {
                data[0] = 1;
                data[1] = now;
                return true;
            }

            if (data[0] < maxRequests) {
                data[0]++;
                return true;
            }
            return false;
        }
    }
}
```

---

## Full Production-Grade Rate Limiter (Interface + Multiple Algorithms)

```java
// ─── INTERFACE ───
public interface RateLimiter {
    boolean allowRequest(String key);
    RateLimitInfo getInfo(String key);
}

public class RateLimitInfo {
    public final boolean allowed;
    public final int remaining;
    public final long retryAfterMs;

    public RateLimitInfo(boolean allowed, int remaining, long retryAfterMs) {
        this.allowed = allowed;
        this.remaining = remaining;
        this.retryAfterMs = retryAfterMs;
    }
}

// ─── TOKEN BUCKET IMPLEMENTATION ───
public class TokenBucketLimiter implements RateLimiter {
    private final long capacity;
    private final double refillRatePerMs;
    private final ConcurrentHashMap<String, Bucket> buckets = new ConcurrentHashMap<>();

    public TokenBucketLimiter(long capacity, double requestsPerSecond) {
        this.capacity = capacity;
        this.refillRatePerMs = requestsPerSecond / 1000.0;
    }

    @Override
    public boolean allowRequest(String key) {
        return getInfo(key).allowed;
    }

    @Override
    public RateLimitInfo getInfo(String key) {
        Bucket bucket = buckets.computeIfAbsent(key, k -> new Bucket(capacity));

        synchronized (bucket) {
            bucket.refill(refillRatePerMs, capacity);

            if (bucket.tokens >= 1.0) {
                bucket.tokens -= 1.0;
                return new RateLimitInfo(true, (int) bucket.tokens, 0);
            }

            // Calculate when next token will be available
            long retryAfterMs = (long) ((1.0 - bucket.tokens) / refillRatePerMs);
            return new RateLimitInfo(false, 0, retryAfterMs);
        }
    }

    private static class Bucket {
        double tokens;
        long lastRefill;

        Bucket(long capacity) { tokens = capacity; lastRefill = System.currentTimeMillis(); }

        void refill(double ratePerMs, long cap) {
            long now = System.currentTimeMillis();
            tokens = Math.min(cap, tokens + (now - lastRefill) * ratePerMs);
            lastRefill = now;
        }
    }
}

// ─── RATE LIMITER FACTORY ───
public class RateLimiterFactory {
    public enum Algorithm { TOKEN_BUCKET, FIXED_WINDOW, SLIDING_WINDOW_LOG }

    public static RateLimiter create(Algorithm algo, int maxRequests, long windowMs) {
        return switch (algo) {
            case TOKEN_BUCKET -> new TokenBucketLimiter(maxRequests, maxRequests * 1000.0 / windowMs);
            case FIXED_WINDOW -> new FixedWindowRateLimiterAdapter(maxRequests, windowMs);
            case SLIDING_WINDOW_LOG -> new SlidingWindowLogAdapter(maxRequests, windowMs);
        };
    }
}

// ─── USAGE IN HTTP MIDDLEWARE ───
public class RateLimitMiddleware {
    private final RateLimiter limiter;

    public RateLimitMiddleware(RateLimiter limiter) { this.limiter = limiter; }

    public boolean processRequest(HttpRequest request) {
        String key = request.getUserId();  // or IP address
        RateLimitInfo info = limiter.getInfo(key);

        if (!info.allowed) {
            System.out.println("429 Too Many Requests. Retry after: " + info.retryAfterMs + "ms");
            return false;
        }
        return true;
    }
}
```

---

## Algorithm Comparison

| Algorithm | Memory | Burst | Accuracy | Use Case |
|---|---|---|---|---|
| Fixed Window | O(1) | Yes (at boundary) | Approximate | Simple APIs |
| Sliding Window Log | O(n) per user | No | Exact | Low traffic, high accuracy |
| Sliding Window Counter | O(1) | Moderate | Good approximation | Most APIs |
| Token Bucket | O(1) | Yes (controlled) | Good | APIs with burst tolerance |
| Leaky Bucket | O(1) | No (constant drain) | Good | Stream processing |

---

## Concurrency Follow-up Questions

**Q: The `allowRequest()` on `TokenBucket` uses `synchronized(bucket)`. Why not `synchronized(this)`?**
> `synchronized(this)` would lock the entire `TokenBucketLimiter` — all users' requests would be serialized. `synchronized(bucket)` locks only ONE user's bucket. User A's requests don't block User B's requests. This is the same "striped locking" principle as the Parking Lot. Throughput scales with number of users.

**Q: Is `ConcurrentHashMap.computeIfAbsent` + `synchronized(bucket)` truly race-free?**
> Almost. `computeIfAbsent` is atomic — two threads calling it for the same key will only create one `Bucket`. The returned reference is then synchronized. But there's a subtle issue: what if thread A gets the bucket reference and is preempted before entering `synchronized`, then thread B creates a different bucket for the same key (impossible with `computeIfAbsent`), then thread A enters synchronized on the OLD bucket?
>
> With `ConcurrentHashMap`, `computeIfAbsent` always returns the same instance for the same key. So all threads synchronize on the same object. This is safe.

**Q: Token Bucket — what happens when the server restarts? All tokens are reset to full capacity.**
> This means users who were rate-limited get full capacity after a restart — potential burst attack window. Fix: Persist token count and last refill time to Redis/cache on each update. On startup, load state from Redis.

**Q: How do you rate limit in a distributed environment (multiple servers)?**
> In-memory rate limiting (above) only works per server. With 3 servers, a user can make 3× the limit.
>
> **Redis-based solution:**
> ```
> INCR user:{userId}:count  (atomic increment)
> EXPIRE user:{userId}:count 60  (set TTL for window)
> Check if count > maxRequests
> ```
> Redis INCR is atomic. Lua scripts for read-increment-check atomically. Redis Cluster for horizontal scale.

**Q: What is the "thundering herd" problem with rate limiters?**
> When the rate limit resets (e.g., at second boundary), all queued clients may simultaneously retry. This creates a spike that overwhelms the backend.
>
> Fix: Add jitter to retry-after time:
> ```java
> long retryAfterMs = baseRetryAfterMs + (long)(Math.random() * 500);  // ±500ms jitter
> ```
> Distributes the retry burst over time.

**Q: Design a rate limiter that allows different rates for different API tiers.**
> ```java
> public class TieredRateLimiter {
>     private final Map<String, RateLimiter> tierLimiters = new HashMap<>();
>
>     public TieredRateLimiter() {
>         tierLimiters.put("FREE", new TokenBucketLimiter(10, 1.0));     // 1 req/sec, burst 10
>         tierLimiters.put("BASIC", new TokenBucketLimiter(100, 10.0));  // 10 req/sec, burst 100
>         tierLimiters.put("PRO", new TokenBucketLimiter(1000, 100.0));  // 100 req/sec, burst 1000
>     }
>
>     public boolean allowRequest(User user) {
>         RateLimiter limiter = tierLimiters.getOrDefault(user.getTier(),
>                                   tierLimiters.get("FREE"));
>         return limiter.allowRequest(user.getId());
>     }
> }
> ```
