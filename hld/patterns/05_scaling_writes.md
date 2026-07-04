# Pattern 5 — Scaling Writes

> **Problem:** A single primary node can no longer keep up with write volume. Metrics ingestion, tweet firehose, IoT telemetry, ad clicks — all can push millions of writes per second, well past any single DB.

## The Levers

```
1. Vertical scale         ── bigger box, NVMe, more RAM        (buy time)
2. Batch writes           ── coalesce N writes into 1 flush     (10-100x)
3. Async / queue-buffer   ── absorb spikes, smooth to DB rate
4. Denormalize / drop indexes ── each index costs a write
5. Vertical partition     ── move hot columns to separate table
6. Horizontal shard       ── split by key across many nodes     (the big lever)
7. Different storage      ── LSM (Cassandra, RocksDB) beats B-tree for writes
8. Write-optimized replica topology ── multi-leader, leaderless
```

## 1. Batching

Coalesce writes at the client or an aggregator:
```
Buffer: List<Event>
Flush every 100ms OR when size ≥ 500
   → single bulk INSERT / Kafka produce with batch
```
- Ad servers, metrics pipelines: 100x throughput gain.
- Trade-off: added latency (flush interval), risk of loss if buffer isn't durable.

## 2. Queue as Write Buffer

```
Client → API (fast ack) → Kafka → Consumer → DB
```
- Kafka absorbs 10x spike; DB drains at its own pace.
- Ordering preserved per partition.
- At-least-once delivery → downstream must be idempotent.

## 3. Storage Engine Choice — B-tree vs LSM

| | B-tree (Postgres, MySQL) | LSM (Cassandra, RocksDB, ScyllaDB) |
|---|---|---|
| Write path | Update-in-place, WAL | Append to memtable, flush to SST |
| Write amplification | Low-moderate | High (compaction) but sequential |
| Peak write TPS | ~10-50k/node | 100k-1M/node |
| Read | Fast, direct | Slower, checks multiple SSTs |
| Best for | Balanced OLTP | Write-heavy, time-series |

**Rule of thumb:** if you're doing more inserts than lookups by a wide margin, use LSM (or a time-series DB like InfluxDB / TimescaleDB / VictoriaMetrics).

## 4. Sharding

Split data across N nodes; each node owns a subset.

### Sharding strategies
| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| **Range** | `userId 0-1M → shard 1, 1M-2M → shard 2` | Range scans efficient | Hot shards (new users all land on last shard) |
| **Hash** | `shard = hash(key) % N` | Even distribution | Range scans impossible; resharding painful (see below) |
| **Consistent hashing** | Ring of virtual nodes | Add/remove nodes moves 1/N keys only | Slightly less balanced |
| **Directory** | Lookup service maps key → shard | Flexible, can re-map | Lookup is another hop; SPOF |
| **Geographic** | Shard by user's region | Latency + compliance | Cross-region ops complex |

### Picking a shard key
- **High cardinality** — many unique values.
- **Even distribution** — no hot keys.
- **Aligned with access pattern** — most queries include the shard key (no scatter-gather).
- Examples: `user_id`, `tenant_id`, `(user_id, YYYYMM)`, `region + user_id`.
- Anti-examples: `country` (few values, hot), `timestamp` (all new writes hit last shard).

### Resharding
Moving from N → 2N shards is the hard part:
- **Consistent hashing** minimizes movement.
- **Double-write phase**: write to old and new shard, backfill in background, cut over reads.
- **Vitess, Citus, CockroachDB** automate this for MySQL/Postgres.

### Cross-shard operations
- **Joins** across shards = scatter-gather; keep to a minimum.
- **Transactions** across shards = 2PC (slow, fragile) or Sagas (see pattern 7).
- **Global secondary indexes** need their own sharding.

## 5. Multi-Leader & Leaderless

- **Single-leader** (Postgres primary): all writes go to one node → limited by that box.
- **Multi-leader** (active-active across DCs, Cassandra with LOCAL_QUORUM): writes to any node, replicate to peers. Conflict resolution needed (LWW, CRDTs).
- **Leaderless** (Cassandra, DynamoDB): quorum reads and writes; W + R > N ensures consistency.

## 6. Vertical Partition

Split a wide row into hot/cold columns:
```
user           user_metrics_hot           user_metrics_cold
-----          -----------------          -----------------
id             id, last_login, ...        id, first_login, referrer, ...
name
email
```
Writes to `last_login` don't invalidate cold cache lines.

## Reference Architecture — Metric Ingestion at 1M/sec

```
1M devices → HTTPS → API (Go, stateless, 200 pods)
                       │  batch 1000 in 200ms
                       ▼
                    Kafka  (topic: metrics, 1000 partitions, hashed on deviceId)
                       │
                       ▼
               Consumers (Flink)
                       │  aggregate per (deviceId, metric, 1min bucket)
                       ▼
           TimescaleDB / VictoriaMetrics (hypertable partitioned by time + device)
```

**Why this shape:**
- API is stateless — scale horizontally, no shared state.
- Kafka absorbs spikes and provides replay.
- Aggregation *before* DB reduces row count 10-100x.
- Time-series DB uses columnar compression, drops old chunks efficiently.

## Interview Q&A

**Q: When is it time to shard?**
- Vertical scaling exhausted (largest instance, still saturated).
- Read replicas can no longer help writes.
- Working set no longer fits in RAM on a single box.
- Not before! Sharding is a one-way door — undo is expensive.

**Q: You picked `user_id` as shard key but one user has 100x traffic — now what?**
- **Sub-shard the hot key**: `hash(userId + timestamp)` — trades locality for balance.
- **Move that user to a dedicated shard** (directory-based override).
- **Cache the hot key aggressively** upstream so writes drop.

**Q: How do you keep writes idempotent through a queue?**
- Client sends idempotency key; consumer writes with `IF NOT EXISTS` keyed by it.
- Or use a dedup table with TTL.
- Kafka + Kafka Streams has exactly-once semantics via transactional producers.

**Q: What's the difference between partitioning and sharding?**
- Often used interchangeably. Nuance:
  - Partitioning = splitting a table within a single DB (Postgres partitioned table).
  - Sharding = splitting across multiple *nodes*.

**Q: How does write amplification work in LSM?**
- Data lands in memtable → flushed to L0 SST → compacted to L1 → ... → LN.
- Same data written multiple times as it descends. RocksDB: ~10x amplification typical.
- Trade-off: sequential writes are cheap on SSDs, so still faster than in-place updates.

**Q: How do you handle a schema migration on a sharded DB?**
- **Backward-compatible steps**: add nullable column → deploy code that writes it → backfill → deploy code that reads it → mark NOT NULL.
- Never a single "migrate everything now" step in prod.
- Tools: gh-ost, pt-online-schema-change, Liquibase.

**Q: How do you rate-limit writes upstream to protect the DB?**
- Token bucket per tenant at API layer (see [Rate Limiter LLD](../../java-lld/LLD_06_Rate_Limiter.md)).
- Back-pressure: Kafka lag alarm → auto-scale consumers, throttle producers if lag > threshold.

## Real-World Examples
- **Discord**: MongoDB → Cassandra when they hit 1B messages/day; then ScyllaDB (C++ Cassandra) at trillions.
- **Twitter**: Manhattan (custom sharded k/v) for tweets.
- **Uber**: Schemaless (on top of MySQL shards) then migrated to Docstore.
- **Cloudflare**: Rocks in Rust (Cloudflare Workers KV) for edge writes.
- **DataDog**: Kafka → aggregators → Cassandra for metrics.

## Gotchas
- **Hot shards** — 90% of load on 10% of shards. Monitor per-shard TPS; consider sub-sharding.
- **Rebalance thrash** — adding a node triggers massive data movement; do it in low-traffic windows.
- **Cross-shard transactions** are almost always the wrong tool. Restructure the data model.
- **Backup strategy** — sharded DB backups need coordination; snapshot per shard, but a global consistent snapshot needs freeze-point.
- **Fan-out on write** (pattern 4) is a write-amplification pattern in disguise — 10M followers = 10M writes per post. Consider hybrid.
