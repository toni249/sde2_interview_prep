# Pattern 4 — Scaling Reads

> **Problem:** Reads dominate — a social feed, a product catalog, a news homepage. Your DB is the bottleneck long before writes are. Read-to-write ratio ≥ 10:1 is the usual trigger.

## The Ladder (cheapest to costliest)

```
1. Add an index                  ── microseconds saved on query
2. Read replicas                 ── horizontal read capacity
3. In-memory cache (Redis)       ── ms → μs latency, offload DB
4. CDN / edge cache              ── move data close to users
5. Denormalize / materialized view ── precompute the answer
6. Search-optimized store (Elastic) ── secondary read path
7. Sharding                      ── last resort, or use pattern 5
```

Always start at the top. "Cache everything" without an index means you cache slow queries.

## 1. Indexing

- **B-tree** for range + equality (default). Covers `WHERE`, `ORDER BY`, joins.
- **Hash** for equality only.
- **GIN / inverted** for full-text and JSON.
- **Partial index** — `WHERE status='active'` cuts index size 10x on soft-deleted rows.
- **Covering index** — put non-key columns in the leaf to avoid table lookup.

Cost: writes get slower (each index needs updating). Never over-index.

## 2. Read Replicas

Primary handles writes; N replicas serve reads via async streaming replication.

```
        writes → [Primary] ──replication──► [Replica 1]
                             └────────────► [Replica 2]
reads ← LB → { replicas }
```

- **Lag** (ms to seconds) → **stale reads**. Consequences:
  - Read-your-writes: user posts, immediately refreshes, doesn't see it. Route recent-writer reads to primary or use sticky session.
- **Failover**: promote a replica on primary crash — orchestration via Patroni, RDS multi-AZ.
- **Sharding vs replicas**: replicas scale reads, sharding scales writes. Read pattern 5 for sharding.

## 3. Caching

### Cache patterns
| Pattern | Read | Write | Notes |
|---------|------|-------|-------|
| **Cache-aside** (lazy) | check cache → miss → DB → put cache | write DB, invalidate cache | Most common; app manages both stores |
| **Read-through** | cache library fetches from DB on miss | write DB | Simpler code; needs cache library |
| **Write-through** | check cache | write cache and DB atomically | Always consistent; slower writes |
| **Write-behind** | check cache | write cache, async flush to DB | Fast, risk of data loss on cache crash |

### Where to cache
- **Client** (browser, mobile) — `Cache-Control: max-age`.
- **CDN** — static assets, GET responses with `Vary: Accept-Language`.
- **Application** — Guava/Caffeine (per-process, avoids network hop).
- **Distributed** — Redis / Memcached cluster.
- **DB internal** — buffer pool; free but limited.

### Cache key design
- Include *all* varying inputs: `user:123:feed:page:2:lang:en`.
- Version prefix (`v3:...`) so a bad deploy can invalidate all keys.

### Invalidation
- **TTL** — simplest, but stale window = TTL.
- **Explicit delete on write** — race condition risk (delete-then-DB-write can leave stale value if reader interleaves).
- **Set with new value on write** (write-through) — no race.
- **Change data capture (CDC)** — Debezium consumes WAL, updates cache.

### Cache pitfalls
- **Thundering herd**: TTL expires, 1000 requests hit DB simultaneously.
  - *Fix:* single-flight (only one caller refills), or probabilistic early expiration.
- **Hot key**: one key gets 90% of traffic (celebrity user's profile).
  - *Fix:* local L1 cache per app node, or shard the key (`user:123:v1`, `user:123:v2`).
- **Cache stampede on cold start**: pre-warm on deploy or gradually shift traffic.
- **Big keys**: 10 MB Redis value blocks other commands. Split into hash fields.

## 4. CDN / Edge

For anything geo-cacheable: images, videos, static HTML, API GETs with proper headers.
- Cloudflare / Fastly / CloudFront edge PoPs.
- Cache key = URL + `Vary` headers. Watch out for `Cookie` in `Vary` — it kills cache hit ratio.
- **Purge on origin write**: Cloudflare API or `stale-while-revalidate`.

## 5. Denormalization / Materialized Views

Normalized schema → many joins → slow reads. Precompute:
- **Feed table** — per user, list of post IDs, updated by fan-out on write.
- **Follower counts** stored on user row, updated via async job.
- **Materialized view** (Postgres) or **secondary table** in DynamoDB.

Trade-off: writes are now heavier (must update all views); consistency is eventual.

## 6. Search-Optimized Store

Elasticsearch / OpenSearch / Meilisearch for full-text, faceted, ranked queries.
- Populated via CDC from primary DB.
- Keep it a read replica only — never source of truth.

## Reference Architecture — Instagram Feed

```
Post write:
   API → Postgres (posts table)
   API → Kafka (post-created event)
   Fanout worker → for each follower, push post_id into feed:{followerId} (Redis list, capped)

Feed read:
   API → Redis LRANGE feed:{userId} 0 20 → post IDs
   API → Cache-aside (Redis) for each post → miss → Postgres
   Merge in real-time updates from live-followers via SSE
```

**Key trade-offs:**
- **Fan-out on write** = O(followers) writes, O(1) reads. Bad for celebrities with 100M followers.
- **Fan-out on read** = O(follows) reads at feed load. Bad for average user with 500 friends.
- **Hybrid**: celebrity posts fetched on read, normal posts fanned out. (Twitter uses this.)

## Interview Q&A

**Q: When would you *not* cache?**
- Data changes faster than TTL is useful (real-time stock prices — use pub/sub).
- Regulatory reads (must show authoritative value).
- Per-user data with no reuse (cache hit rate ≈ 0).

**Q: Cache-aside race: two writers, one deletes cache, one reader repopulates with stale DB read?**
- Yes — classic race. Fix with **double-delete after DB write** (delete → write → sleep 50ms → delete again), or use CDC-driven invalidation.

**Q: How do you pick TTL?**
- Business tolerance × update frequency. Product prices: 60s. User profile: 5min. Static config: 1h. Add jitter (`ttl + rand(0, ttl/10)`) to avoid stampede.

**Q: Read replicas can't keep up with write load?**
- Chain replicas (cascading replication).
- Filter replication (only some tables).
- Move to sharded write architecture (pattern 5).

**Q: How do you handle read-your-writes with async replicas?**
- Route the writing user's next N reads to primary.
- Or return the write's LSN to client; client sends it as `min_lsn`; router waits until a replica catches up.
- Or use synchronous replication for critical reads (slower writes).

**Q: What if the cache goes down?**
- **Fail-open**: bypass cache, hit DB. Risk: DB overload.
- **Fail-closed**: return error. Safer for DB, bad UX.
- Circuit breaker + capacity headroom on DB. Multi-AZ Redis with failover.

**Q: How big should the cache be?**
- Working set size, not full dataset. 80/20 rule: cache the top 20% keys serving 80% of reads.
- LRU eviction. Monitor hit rate; below 80% means cache is too small or key design is wrong.

## Real-World Examples
- **Instagram**: fan-out on write feed + Cassandra + Memcached tiers.
- **Netflix**: EVCache (Memcached fork), 30+ regional clusters, primary tier + backup tier.
- **Twitter**: hybrid fan-out; Redis home timeline lists per user.
- **Wikipedia**: multi-tier — Varnish CDN → Memcached → MariaDB.
- **Amazon product page**: DAX (DynamoDB accelerator) + CloudFront + backend cache.

## Gotchas
- **Cache hit rate paradox**: high hit rate can *mask* a slow DB. When cache dies, DB melts.
- **Cache key drift** after code refactor: two versions writing incompatible values under same key. Version the key.
- **Session data in Redis** — if you lose Redis you log everyone out. Persist to disk or use sticky sessions.
- **Serialization cost** — JSON parsing in Redis clients can dominate; use binary (MessagePack, Protobuf).
