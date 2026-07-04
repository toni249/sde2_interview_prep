# 11 — Caching

> Caching trades staleness for speed. It's the highest-leverage lever for latency and load, and the source of the majority of "why is prod broken?" incidents.

## Where You Can Cache

```
Client browser → CDN → App-local (Caffeine) → Distributed (Redis) → DB buffer pool
    fastest, tiny                                                        slowest, biggest
```

Rule: cache at the layer closest to the request where the data is still shared.

| Layer | Latency | Scope |
|-------|---------|-------|
| CPU / L1-L3 | ns | Single core / socket |
| App-in-memory (Caffeine, Guava) | µs | One process |
| Distributed (Redis, Memcached) | 0.5-2 ms | Cluster |
| CDN edge | 5-20 ms | Regional |
| Origin (DB with cache) | 10-50 ms | Global |

## Caching Patterns

### Cache-aside (Lazy Loading)
```
def get(key):
    v = cache.get(key)
    if v is None:
        v = db.get(key)
        cache.set(key, v, ttl=300)
    return v

def update(key, value):
    db.write(key, value)
    cache.delete(key)          # or set
```
- **Most common**. App manages both stores.
- Race: writer deletes cache, reader misses, DB read, reader writes stale value back. Mitigations below.

### Read-Through
- Cache library fetches on miss (cache handles DB call).
- Cleaner code; app sees only cache API.

### Write-Through
- Every write goes to cache and DB atomically.
- Always consistent; slower writes; extra work if cache items aren't read.

### Write-Behind (Write-Back)
- Write to cache; async flush to DB.
- Fastest writes; **can lose data** if cache crashes before flush.
- Used cautiously (Facebook TAO for hot writes).

### Refresh-Ahead
- Cache proactively re-fetches near expiration.
- Prevents miss latency spikes.

## Eviction Policies

Cache is finite. When full, kick something out.

| Policy | Rule | When |
|--------|------|------|
| **LRU** — Least Recently Used | Kick oldest untouched | General purpose |
| **LFU** — Least Frequently Used | Kick fewest accesses | Long-tail with steady stars |
| **FIFO** | Oldest inserted | Rare; simpler |
| **TinyLFU** (Caffeine default) | LFU with sketch admission | State-of-art, near-optimal |
| **Random** | Random victim | Simple, occasionally good |
| **TTL only** | Never explicit evict | Trust TTL to age out |

Redis: `maxmemory-policy allkeys-lru` (most common) or `volatile-lru` (only keys with TTL).

## Invalidation Strategies

**"There are only two hard things in Computer Science: cache invalidation and naming things."** — Phil Karlton.

### 1. TTL only
Simplest. Stale window = TTL. Use short TTLs for volatile data, long for immutable.

### 2. Explicit on write
```
db.update(key)
cache.delete(key)     # or cache.set with new value
```
- Race window: reader between `db.update` and `cache.delete`.
- Better: `cache.set(new value)` — no window.

### 3. Change Data Capture (CDC)
- Debezium reads DB WAL, publishes to Kafka.
- Consumer updates cache.
- Delay: 10-500 ms, but zero application-code coupling.

### 4. Versioned keys
- Include version in key: `user:42:v3`.
- On write, increment version; old key becomes garbage (evicted by LRU).
- Effectively immutable cache entries.

## Cache Pitfalls

### Thundering Herd (Cache Stampede)
Popular key expires → 1000 concurrent requests hit DB simultaneously.

**Fixes:**
- **Single-flight** / **request coalescing**: only one caller refills; others wait.
- **Probabilistic early expiration**: refresh with probability rising as TTL nears end (see: `stale-while-revalidate`).
- **Random jitter** on TTL: `ttl + rand(0, ttl/10)` so keys don't all expire at once.

### Hot Key
One key gets 90% of traffic (celebrity user profile).

**Fixes:**
- **Local L1 cache** in the app (Caffeine) — sub-µs hits, no network.
- **Key sharding**: `user:42:v1`, `user:42:v2`, `user:42:v3` — read random suffix.
- **Read-through with in-flight dedup**.

### Cache Penetration
Missing key repeatedly queried (attack or bad client).

**Fixes:**
- Cache the "negative" result (with shorter TTL).
- Bloom filter (see [16](16_bloom_and_probabilistic.md)) in front to reject lookups for keys known not to exist.

### Big Keys
Single Redis value = 10 MB → blocks other commands (Redis is single-threaded).

**Fixes:**
- Split into hash fields.
- Compress with LZ4/Snappy.
- Move to a proper DB.

### Cold Cache After Deploy
Cache flush + full deploy = DB overload.

**Fixes:**
- Rolling deploy so at any moment most instances have warm cache.
- Pre-warm: replay recent access log against new instance.
- Traffic shifting: 1% → 10% → 100% over minutes.

## Consistency in Cache-Aside — the Nasty Race

```
Time  Reader                 Writer
 T1  cache miss for X
 T2                          DB write X=42
 T3                          cache.delete(X)
 T4  DB read X → 42 (fine)
 T5  cache.set(X, 42)
```
OK. But:
```
Time  Reader                 Writer
 T1  cache miss for X
 T2  DB read X → 10          DB write X=42
 T3                          cache.delete(X)
 T4  cache.set(X, 10)        (stale forever)
```
Reader wrote stale value *after* writer's delete.

**Fixes:**
- **Double-delete**: writer deletes cache, writes DB, sleeps 100 ms, deletes cache again.
- **Redis-based mutex** around read-modify-cache.
- **Write-through**: writer updates cache directly with new value.
- Accept staleness within a small window (usually fine).

## Redis vs Memcached

| | Redis | Memcached |
|---|---|---|
| Data types | Strings, lists, sets, hashes, sorted sets, streams, geo | Strings only |
| Persistence | Yes (RDB/AOF) | No |
| Replication | Master-replica, cluster | External |
| Pub/Sub | Built-in | No |
| Scripts | Lua | No |
| Threads | Single-threaded (mostly) | Multi-threaded |
| Use | Cache + primary for some data | Pure cache |

**Modern default**: Redis, unless you specifically need pure raw KV cache with multi-threaded throughput.

## Multi-Tier Caching

Real large systems:
```
Request → CDN (edge)           ~90% hit
         → App L1 (Caffeine)   ~50% of misses hit
         → Redis (L2)          ~40% of remaining hit
         → DB (source)         final fallback
```

**Cascading TTLs**: CDN 5m → Redis 1h → DB. Each layer shorter than the next.

## Reference Config — Feed Item Cache

```yaml
key_pattern: post:{postId}:v{schemaVersion}
ttl: 300s + jitter(30s)
eviction: allkeys-lru
serialization: MessagePack (2-3x smaller than JSON)
negative_cache: 60s for 404s
size_limit: 100 KB per value
```

Monitor:
- **Hit ratio** (target ≥ 80%).
- **Evictions per second** (spikes = too small).
- **Memory usage** (approaching maxmemory).
- **Command latency p99** (Redis normally < 1 ms).

## Interview Q&A

**Q: When would you NOT cache?**
- Data changes faster than TTL is useful (real-time prices → pub/sub instead).
- Reads are almost never repeated (no reuse).
- Data must be strictly current (financial regulator queries).

**Q: How do you pick TTL?**
- Business tolerance × update frequency.
- User profile: 5-15 min.
- Product listing: 1-5 min.
- Config: 1 h.
- Add jitter.

**Q: How do you invalidate cache across regions?**
- Global Redis (cross-region replication) — added latency.
- Local caches + CDC via Kafka MirrorMaker — eventual across regions.
- Version keys — never invalidate, just bump version.

**Q: What is negative caching?**
- Caching "not found" or errors briefly. Prevents repeated DB pounding for missing keys.
- Short TTL (60 s) so a newly created item is discoverable soon.

**Q: How do you handle cache failure (Redis down)?**
- **Fail open**: bypass to DB. Risk: DB overload.
- **Fail closed**: return error. Safer for DB, bad UX.
- **Bulkhead**: separate Redis clusters per service. One dies, others live.
- Circuit breaker + DB capacity headroom.

**Q: What is a cache warmer?**
- Job that pre-populates cache with high-value keys (top 1000 products, active users) before traffic hits.
- Prevents cold-cache latency spike after deploy or eviction.

**Q: Local (in-process) vs distributed cache?**
- Local: sub-µs, no network, but per-instance (harder to invalidate).
- Distributed: shared state, easy to invalidate, but adds network + serialization.
- Combine: L1 local (small TTL) + L2 Redis.

## Real-World Examples
- **Facebook**: memcached fleet (originally); now TAO (a caching layer over MySQL for the social graph).
- **Netflix**: EVCache (Memcached fork) with cross-AZ replication.
- **Twitter**: Redis-based timeline caches per user.
- **Instagram**: memcached + local caches per app pod.
- **Wikipedia**: Varnish → memcached → MariaDB.

## Gotchas
- **`SELECT *` cached**: schema changes break every entry. Cache DTOs, not rows.
- **Serialization drift**: two service versions cache incompatible objects. Version the key.
- **Session data in Redis**: if Redis dies, everyone gets logged out. Persist Redis or use signed JWTs.
- **Long-running cache miss**: 30 s DB query → user waits 30 s → single-flight actually helps.
- **Cache poisoning via user input**: request path is cache key; user smuggles headers → poisoned entry served to others.
