# Back-of-Envelope Estimation — Baseline Numbers

> The numbers to memorize for HLD interviews. When someone asks "how fast is X?" or "how many QPS can Y handle?", these are the anchors. Real prod varies ±3×; use these as order-of-magnitude.

## The Golden Ladder (memorize this)

```
L1 cache               0.5   ns
L2 cache               7     ns          14×  L1
RAM access             100   ns          14×  L2
SSD random read        100   µs        1000×  RAM
Same-DC round trip     500   µs           5×  SSD
HDD seek               10    ms         100×  SSD
Cross-region RTT       70    ms           7×  HDD
Cross-ocean RTT        100   ms          10×  same-DC
US ↔ India RTT         250   ms           3×  cross-ocean
```

**Rule of thumb**: each rung is roughly **10-100×** the previous. Data locality dominates every design.

## 1. Cache Latency

| Cache tier | Latency | Notes |
|------------|---------|-------|
| CPU L1 | 0.5 ns | Per-core |
| CPU L2 | 7 ns | Per-core |
| CPU L3 | 25 ns | Shared per socket |
| **In-process (Caffeine, Guava)** | **0.1-1 µs** | Just a HashMap lookup |
| **Local Redis (unix socket)** | **50-100 µs** | Same host, no network |
| **Remote Redis (same DC)** | **0.5-2 ms** | 1 RTT + Redis processing |
| **Remote Memcached (same DC)** | **0.3-1 ms** | Multi-threaded; often faster than Redis |
| **Redis cluster with proxy** | **1-3 ms** | Extra hop |
| **Cross-AZ Redis** | **1-3 ms** | ~1 ms AZ latency added |
| **Cross-region Redis** | **60-100 ms** | Avoid for hot path |
| **CDN edge cache hit** | **5-30 ms** | Depends on user distance to PoP |

**Design hint**: local (in-process) cache is 1000× faster than remote Redis. Multi-tier caches pay off.

## 2. Database Latency

### Point Reads (by PK, hitting cache)

| DB | Read latency (p50) | Notes |
|----|-------------------|-------|
| Postgres (buffer pool hit) | 0.1-0.5 ms | RAM read |
| Postgres (SSD miss) | 1-5 ms | Index + row page fetches |
| MySQL InnoDB | 0.1-0.5 ms (hit) / 1-5 ms (miss) | Similar to Postgres |
| MongoDB (WiredTiger) | 0.5-2 ms | Similar profile |
| Redis GET | 0.1-0.5 ms local, 0.5-2 ms remote | In-memory |
| DynamoDB | 5-10 ms (eventually consistent) / 10-20 ms (strong) | Managed hop |
| Cassandra (LOCAL_QUORUM) | 2-10 ms | Multi-node read |
| Elasticsearch (term query) | 5-50 ms | Depends on shard state |

### Range Scans / Complex Queries

| Query | Typical Latency |
|-------|-----------------|
| Postgres index range scan (1k rows) | 5-20 ms |
| Postgres seq scan on 10M row table | 500ms - 5 s |
| Join across 3 tables (indexed) | 5-50 ms |
| Analytical scan on 1B rows (ClickHouse) | 100 ms - few s |
| BigQuery over TB | seconds |

### Writes

| DB | Write latency (p50) | Notes |
|----|--------------------|-------|
| Postgres single-row INSERT | 1-5 ms | fsync WAL |
| Postgres with sync replica | 5-15 ms | Wait for replica ack |
| MySQL single-row INSERT | 1-5 ms | Similar |
| DynamoDB PUT | 10-20 ms | Managed |
| Cassandra write (LOCAL_QUORUM) | 2-10 ms | Log-structured, fast |
| Redis SET | 0.1-0.5 ms local, 0.5-2 ms remote | Memory + AOF flush |
| Kafka produce (acks=1) | 2-10 ms | 1 broker ack |
| Kafka produce (acks=all) | 5-20 ms | Wait for ISR |
| S3 PUT small object | 20-100 ms | HTTPS + storage |
| S3 PUT large multipart | seconds to minutes | Chunked |

**Design hint**: don't do more than ~5 DB calls per user request; each round-trip adds ~5 ms. Batch reads or use JOINs.

## 3. Network Latency (RTT)

| Path | RTT | Notes |
|------|-----|-------|
| Loopback | < 0.1 ms | Same process to same process |
| Same host, different process | 0.05-0.2 ms | Unix socket / localhost |
| Same rack | 0.1-0.5 ms | Same top-of-rack switch |
| Same AZ | 0.5-1 ms | Multiple racks |
| Cross-AZ (same region) | 1-2 ms | E.g., us-east-1a ↔ 1b |
| Cross-region (US-East ↔ US-West) | 60-80 ms | Continental |
| Cross-region (US ↔ EU) | 80-120 ms | Trans-Atlantic |
| Cross-region (US ↔ APAC) | 150-250 ms | Trans-Pacific |
| Global (US ↔ India) | 200-280 ms | Long fiber path |
| Mobile 4G to nearest edge | 30-80 ms | + last-mile jitter |
| Mobile 5G | 10-30 ms | Approaching wired |
| Satellite (Starlink LEO) | 20-40 ms | LEO orbit |
| Traditional satellite (GEO) | 500-700 ms | Geosync altitude |

### Protocol Handshake Overhead

| Handshake | Extra RTTs | Notes |
|-----------|-----------|-------|
| TCP | 1 | SYN, SYN-ACK, ACK |
| TLS 1.2 | 2 | Full handshake |
| TLS 1.3 | 1 | Reduced |
| TLS 1.3 resumed | 0 | Session ticket / PSK |
| QUIC (HTTP/3) | 1 | Combined handshake |
| QUIC 0-RTT | 0 | Same-server reconnect |

**Example**: cold HTTPS request from India to US-East:
- DNS: 50 ms (if cold)
- TCP: 1 RTT = 250 ms
- TLS 1.3: 1 RTT = 250 ms
- Request + response: 1 RTT = 250 ms
- **Total: ~800 ms** before anything renders. Now you know why CDNs and keep-alive matter.

## 4. Message Queue / Streaming Latency

| System | Publish + Consume (p50) | Throughput/broker |
|--------|-------------------------|-------------------|
| Redis Streams (local) | 1-3 ms | 100k msg/s |
| RabbitMQ (persistent) | 5-20 ms | 20-50k msg/s |
| SQS Standard | 20-100 ms | Practically unlimited |
| SQS FIFO | 50-200 ms | 3000/s per group |
| Kafka (acks=1) | 5-20 ms | 100k-1M msg/s |
| Kafka (acks=all) | 10-50 ms | 100k-500k msg/s |
| Kinesis | 50-200 ms | 1 MB/s per shard |
| NATS (core, no persist) | < 1 ms | 1M+ msg/s |

## 5. Storage Latency

| Storage | Read Latency | Write Latency |
|---------|--------------|---------------|
| RAM | 100 ns | 100 ns |
| NVMe SSD (random) | 20-100 µs | 20-100 µs |
| SATA SSD (random) | 100-500 µs | 100-500 µs |
| Spinning HDD (random) | 5-15 ms | 5-15 ms |
| Sequential SSD | Up to 3 GB/s | Up to 3 GB/s |
| Sequential HDD | Up to 250 MB/s | Up to 250 MB/s |
| **EBS gp3** | 0.5-2 ms | 0.5-2 ms |
| **S3 first byte** | 100-200 ms | 20-100 ms |
| **S3 throughput per prefix** | ~5500 GET/sec | ~3500 PUT/sec |
| **Glacier** | Minutes to hours | Fast (async) |

## 6. Throughput Baselines (single node)

| Component | Typical QPS / TPS |
|-----------|-------------------|
| Nginx (static file) | 50-100k RPS |
| Nginx (reverse proxy) | 20-50k RPS |
| Node.js (JSON API, non-blocking) | 5-20k RPS |
| Java Spring (JSON API) | 5-20k RPS |
| Go / Rust HTTP | 50-200k RPS |
| Postgres (mixed OLTP) | 5-20k TPS |
| Postgres (read-only, in RAM) | 50-100k QPS |
| MySQL | Similar to Postgres |
| Redis (single instance) | 100k-1M ops/sec |
| Memcached (single instance) | 200k-1M ops/sec |
| Cassandra (per node) | 10-50k writes/sec |
| DynamoDB (per partition) | 3000 RCU / 1000 WCU |
| Kafka broker | 500k-1M msg/sec |
| Elasticsearch (per node, indexing) | 10-50k docs/sec |

**Design hint**: single Postgres box tops out at ~50k QPS. Beyond that, add replicas → shard.

## 7. Data Size Baselines

### Payload Sizes

| Item | Typical Size |
|------|--------------|
| Simple JSON API response | 500 B - 5 KB |
| HTML page (uncompressed) | 20-200 KB |
| HTML page (gzip'd) | 5-50 KB |
| Small image (thumbnail) | 5-50 KB |
| Full JPEG image (mobile) | 200 KB - 2 MB |
| 4K JPEG | 2-8 MB |
| 1-min HD video | 30-100 MB |
| 1-min 4K video | 200-500 MB |
| Kafka message (typical) | 1-10 KB |
| DB row (typical) | 200 B - 2 KB |
| Log line | 200 B - 1 KB |

### Byte Multipliers

| Unit | Bytes |
|------|-------|
| KB | 10³ |
| MB | 10⁶ |
| GB | 10⁹ |
| TB | 10¹² |
| PB | 10¹⁵ |

### Storage per Record — Rules of Thumb

- User profile: ~1 KB
- Tweet / short message: ~500 B (200 chars + metadata)
- Chat message with attachment metadata: ~2 KB
- Order record with items: ~2-5 KB
- Log line: ~500 B
- Metric point (time-series): ~30-100 B

## 8. Powers of 2 (memory / capacity)

| Power | Value | Rough Meaning |
|-------|-------|---------------|
| 2¹⁰ | 1024 | 1 KiB |
| 2²⁰ | ~1 M | 1 MiB |
| 2³⁰ | ~1 B | 1 GiB |
| 2³² | ~4.3 B | IPv4 space, max int |
| 2⁴⁰ | ~1 T | 1 TiB |
| 2⁵⁰ | ~1 P | 1 PiB |
| 2⁶³ | ~9.2 × 10¹⁸ | long max |

## 9. Time Units in a Day

| Interval | Seconds |
|----------|---------|
| 1 minute | 60 |
| 1 hour | 3,600 |
| 1 day | 86,400 (~10⁵) |
| 1 month | ~2.6 × 10⁶ |
| 1 year | ~3.15 × 10⁷ |

**Memorize**: **1 day ≈ 100k seconds**. Makes MAU → QPS math trivial.

## 10. Throughput ↔ Storage Conversions

### QPS from DAU
```
1M DAU × 10 req/user/day = 10M req/day
10M / 100k sec ≈ 100 QPS average
Peak = 3-10× avg → 300-1000 QPS peak
```

### Storage per day
```
100M events/day × 500 B/event = 50 GB/day
× 365 = ~18 TB/year
+ 3x replication + indexes ≈ 60 TB/year
```

### Bandwidth
```
1M users watching 1080p video (5 Mbps each)
= 5 Tbps = 625 GB/sec
→ CDN mandatory
```

## 11. Worked Examples

### Example A: "Design a URL Shortener like bit.ly"

Assumptions:
- 100M new URLs/month → ~40 URLs/sec write
- Read:Write = 100:1 → 4000 reads/sec avg, ~15k peak
- Each URL row ~500 B → 100M/mo × 500 B = 50 GB/mo → **600 GB/yr**

Latency budget for redirect:
```
DNS   (cached)     : 1 ms
TCP + TLS (cached) : 0
LB                 : 1 ms
Cache lookup       : 1 ms
DB fallback (miss) : 5 ms
Response           : 1 ms
Total              : ~10 ms cache-hit path
```

Cache hit at 90% → avg latency ~5 ms.

### Example B: "Twitter feed"

- 500M DAU × 50 reads/day = 25B reads/day = **~300k reads/sec avg, ~1M peak**
- 500M DAU × 5 tweets/day / 3 (fraction posting) = **~30k writes/sec avg**
- Fan-out to followers (avg 300 followers) → **9M "insert into feed" ops/sec**
- Feed cache: 500M users × 1 KB per feed = **500 GB in Redis** (single-region)

Storage of tweets: 500M users × 5 tweets/day × 500 B × 365 = **~450 TB/year raw**.

### Example C: "Video upload service"

- 1M uploads/day, avg 100 MB → 100 TB/day raw
- Transcoding produces 3-5 additional bitrates → **~500 TB/day** total storage
- Bandwidth for viewers at 10× upload: **50 Gbps sustained** → CDN required
- Transcode compute: 100 MB video takes ~5 min → 5 CPU-hours per video → **200k CPU-hours/day** = ~8000 CPU cores continuous

### Example D: "Real-time chat 1M concurrent users"

Connection overhead:
- 1M WebSockets × 10 KB per connection state = **10 GB total** memory
- Across 100 gateway nodes: 100 MB per node — feasible on 4 GB VM

Message rate:
- Avg 1 message/user/min = 16k msg/sec globally
- Peak 5× = 80k msg/sec
- Each fan-out to ~5 recipients (group chat avg) = **400k deliveries/sec**
- Kafka can handle it on a single topic with 10 partitions

## 12. Common Latency Budget for User-Facing Request

Target end-to-end p99 latency for interactive UI: **~200 ms**.

```
User → CDN edge         : 20 ms (network)
Edge → Origin LB        : 30 ms (network, cross-region)
LB → Service            : 1 ms
Service → Cache         : 2 ms
Service → DB (if miss)  : 5 ms
Service → downstream    : 20 ms (auth, personalization)
Response → back         : 50 ms (network back)
Buffer                  : 70 ms
Total                   : 198 ms
```

Every layer must have a latency budget. When one exceeds, others must compensate — or the UX degrades.

## 13. Estimation Formula Cheatsheet

```
QPS peak       = DAU × actions_per_user_per_day × peak_factor / 100k
                 (peak factor: 3-10× avg)

Storage/year   = events_per_sec × 86400 × 365 × bytes_per_event × replication

Bandwidth      = concurrent_users × bit_rate_per_user

Concurrent conns = QPS × avg_request_duration_sec       (Little's Law)

Servers needed = peak_QPS / QPS_per_server × headroom (1.5-2x)

Cache memory   = hot_items × avg_item_size × 2 (overhead)

Kafka storage  = throughput_MB/s × retention_seconds × replication_factor
```

**Little's Law is a superpower** in interviews:
```
L = λ × W
concurrent_in_flight = arrival_rate × avg_time_in_system
```
E.g., 1000 QPS with 100 ms latency → 100 concurrent requests in flight.

## 14. Common Sanity Checks

| Claim | Reality Check |
|-------|---------------|
| "Postgres can do 500k QPS" | Not on a single box. Max ~50k on huge instance. |
| "Redis has ms latency across regions" | ~100 ms across US-EU. Local is sub-ms. |
| "S3 is fast for small objects" | 20-100 ms per PUT. Bad for many small ops. |
| "Cache hit rate 99%" | Only for narrow, hot data. Broad reads: 70-90% realistic. |
| "10ms DB query" | For point read, yes. For join / aggregation, no. |
| "1 ms network hop" | Same DC. Cross-DC always ≥ 10 ms. |
| "We can do 1M WebSockets on one server" | Only with tuned OS + minimal per-conn work. Realistic 100k. |

## 15. Interview Framework — Estimation Q&A

**Q: How many servers do I need for X QPS?**
```
1. Estimate QPS_per_server (10k for JSON API is decent baseline).
2. Divide: QPS / QPS_per_server = base count.
3. Multiply by 1.5-2× for headroom + resilience.
4. Consider peaks (3-10× avg) — size for peak or auto-scale.
```

**Q: How much RAM for cache?**
```
Working set = hot items × item size × (1 + overhead 30%)
For 90% hit rate target, cache size ≈ top-20% keys by frequency.
```

**Q: How much bandwidth for video?**
```
concurrent_viewers × bitrate = total bandwidth
1M × 5 Mbps = 5 Tbps → CDN mandatory
Origin bandwidth = total / CDN_hit_rate  (if 99% hit → 50 Gbps origin)
```

**Q: How long does write take across replicas?**
```
Single primary: 1 fsync + network to replicas (if sync)
   local write : ~5 ms
   + 1 sync replica same AZ : +2 ms
   + 1 sync replica cross-AZ : +5 ms
Typical: 5-15 ms for durable, replicated write.
```

## 16. Numbers Cheatsheet (Printable)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LATENCY LADDER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
L1 cache            0.5 ns
L2 cache             7  ns
RAM read           100  ns
Mutex lock/unlock   25  ns
Branch mispredict    3  ns
NVMe SSD read     100  µs
Same-DC RTT        0.5 ms
Compressed 1 KB     3  µs (on modern CPU)
HDD seek           10  ms
Cross-region RTT   70  ms
Cross-ocean RTT   100  ms
US ↔ India RTT    250  ms

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
THROUGHPUT (per node)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Nginx static        100k RPS
JSON API server     10k RPS
Postgres           50k QPS (RAM), 5k TPS (write)
Redis              100k-1M ops/s
Kafka broker       500k-1M msg/s

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
SIZE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1 day               ~100k sec
1 year              ~30M sec
JPEG image          ~500 KB avg
Tweet record        ~500 B
User profile        ~1 KB
Log line            ~500 B

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FORMULAS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
QPS = DAU × actions/day / 100k × peak_factor
Storage/yr = QPS × 86400 × 365 × row_size × 3 (repl)
Concurrent = QPS × latency (Little's Law)
Servers    = peak_QPS / QPS_per_server × 1.5-2
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Gotchas
- **p50 vs p99**: quoted latencies usually p50; p99 is 3-10× higher. Design for p99.
- **Cold cache latency spike**: first request per key can be 100× slower.
- **GC pauses**: Java/Go can add 50-200 ms hiccup unpredictably. Budget for them.
- **NUMA effects**: cross-socket memory access on big boxes is 2× slower than local.
- **Cloud networking is not free**: cross-AZ traffic ≈ 1 ¢/GB, cross-region 2-9 ¢/GB. Adds up.
- **Vendor benchmarks are best-case**: real prod is 2-5× slower. Discount claims.
- **Warm up matters**: benchmarks after 10 s look great; first-request from cold pod is much slower.
