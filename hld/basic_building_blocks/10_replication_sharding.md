# 10 — Replication & Sharding

> Two orthogonal techniques for scaling a database. **Replication** = same data on multiple nodes (for HA + read scale). **Sharding** = different data on different nodes (for write scale + storage scale). Real systems use both.

## Replication

### Why Replicate?
- **High availability**: primary crashes → replica takes over.
- **Read scale**: fan out reads to N replicas.
- **Geo-distribution**: replica in each region for local reads.
- **Backup & analytics**: heavy ETL on replica without hitting primary.

### Topologies

**Single-leader (primary-replica)** — the workhorse.
```
Writes → [Primary] ──async──► [Replica 1]
                   ──async──► [Replica 2]
Reads ← [Replicas]
```
- Postgres, MySQL, MongoDB default.
- Simple to reason about; failover promotes a replica.
- **Async replication** = fast writes but stale reads on replicas.
- **Sync replication** = consistent but slower (waits for at least one replica).
- **Semi-sync** = wait for one replica ack, others async (MySQL default in newer versions).

**Multi-leader (multi-primary)** — active-active.
```
[Primary DC1] ⇄ [Primary DC2]
Both accept writes; replicate to each other.
```
- Used for multi-region write locality (e.g., MySQL Group Replication, Cassandra with LOCAL_QUORUM).
- **Conflict resolution needed**: same row modified in two places.
  - Last-Write-Wins (LWW) — data loss possible.
  - CRDTs — mergeable, no loss.
  - Application logic — case by case.

**Leaderless** — any node accepts writes.
```
Client → any node, writes forwarded to N replicas.
Reads ← quorum of nodes.
```
- Cassandra, DynamoDB, Riak.
- **Quorum math**: with N replicas, W writers, R readers → `W + R > N` guarantees consistency.
- Tunable per-query: `LOCAL_QUORUM`, `ALL`, `ONE`.

### Replication Lag

Async replicas trail the primary by ms to seconds. Effects:
- **Read-your-writes anomaly**: user posts, then queries and doesn't see it.
- **Monotonic reads**: two reads may go to different replicas → time appears to go backwards.

**Mitigations:**
- Route recent-writer reads to primary (sticky by userId + time window).
- Client tracks LSN of its write; router waits until a replica catches up.
- Synchronous replication for critical paths (slower writes).

### Failover

Primary crashes:
1. Detector notices (heartbeat missed).
2. Cluster elects new primary (Raft/Paxos, or manual).
3. Router flips writes to new primary.
4. Old replicas re-point to new primary.
5. Old primary comes back → resynced as replica.

**Risks:**
- **Split brain**: two primaries think they're it. Client writes to both → divergent data.
- **Fix**: fencing tokens, quorum leader election, STONITH ("shoot the other node in the head").

### Postgres Replication Modes

- **Streaming replication**: WAL streamed to standby, applied.
- **Logical replication**: publish specific tables; subscribers can transform.
- **CDC via WAL**: Debezium reads WAL for downstream stream processing.

## Sharding (Horizontal Partitioning)

### Why Shard?
- Data doesn't fit on one machine.
- Writes exceed single-node capacity.
- Working set doesn't fit in RAM.
- Want per-tenant / per-region isolation.

### Sharding Strategies

**1. Range sharding**
```
shard1: userId 0000-1FFF
shard2: userId 2000-3FFF
shard3: userId 4000-5FFF
```
- Pro: efficient range queries.
- Con: hot shards (newest users on last shard).

**2. Hash sharding**
```
shard = hash(userId) % N
```
- Pro: even distribution.
- Con: range queries scatter across all shards; resharding is painful when N changes.

**3. Consistent hashing**
```
Ring of 2^32 positions; each node owns an arc.
key → hash → find nearest node clockwise.
```
- Pro: adding/removing a node moves only 1/N of keys.
- Con: slightly uneven balance; fix with **virtual nodes** (each physical node claims many arcs).
- Used by: DynamoDB, Cassandra, memcached rings.
- See [15](15_consistent_hashing.md) for depth.

**4. Directory-based sharding**
```
Lookup service: key → shard
```
- Pro: flexible; move hot keys manually.
- Con: extra hop; lookup service is critical path.
- Used by: Vitess, Foursquare's shard mgmt.

**5. Geographic sharding**
```
US users → US shard
EU users → EU shard
```
- Pro: latency, compliance (GDPR).
- Con: cross-region ops complex; user migration (moving abroad).

### Picking a Shard Key

**Requirements:**
- **High cardinality** — many unique values.
- **Even distribution** — no hot values.
- **Aligned with access pattern** — most queries include the shard key.

**Good keys:**
- `user_id` (typical for user-scoped apps).
- `tenant_id` (SaaS with strong isolation).
- `(user_id, YYYYMM)` — composite for time-scoped data.

**Bad keys:**
- `country` — few values, hot USA/China.
- `created_at` — all new writes hit the newest shard.
- `email_domain` — hot @gmail.com shard.

### Cross-Shard Operations

**Joins across shards** = scatter-gather.
- Query hits all shards, results merged.
- Slow, non-scalable; use only for admin queries.

**Transactions across shards** = 2PC or Saga.
- 2PC: slow, coordinator SPOF, but transactional.
- Saga: compensating actions; see [Pattern 7](../patterns/07_multi_step_processes.md).

**Secondary indexes across shards**:
- Local index (per shard): fast, but query without shard key scatters.
- Global index: a separately sharded index of `(indexed_col → primary_key + shard)`.

### Resharding

Adding a shard:
1. **Provision new shard**.
2. **Dual writes** to old + new for keys that would move.
3. **Backfill** historical data to new shard.
4. **Cut over** reads.
5. **Delete** from old shard.

Automation: Vitess, Citus, CockroachDB. Manual sharding scripts still common.

## Combined — Sharded + Replicated

Real prod:
```
shard 1: primary + 2 replicas
shard 2: primary + 2 replicas
shard 3: primary + 2 replicas
...
```

- **Replication within a shard**: HA + read scale.
- **Sharding across shards**: write + storage scale.
- Coordination via routing layer (ProxySQL, Vitess) or client-side (Cassandra drivers).

## Quorum Math

N replicas, W ack for a write, R ack for a read.

- `W + R > N` → guaranteed to read at least one node with the latest write (**consistent**).
- `W + R ≤ N` → possible stale reads (eventual consistency).

**Common tunings:**
- N=3, W=2, R=2 → consistent, tolerates 1 node down.
- N=3, W=1, R=1 → fast, eventually consistent.
- N=3, W=3, R=1 → fast reads, slow writes.

## Interview Q&A

**Q: When do you shard, and when do you just add replicas?**
- Replicas: read-heavy workload, high availability. Doesn't help writes.
- Shard: writes or storage exceed a single node; there's no other choice.
- Order: vertical scale → replicas → shard.

**Q: What's the biggest risk of sharding?**
- Choosing the wrong shard key. Fixing means downtime or a heroic migration. Sit with the access patterns for a week before committing.

**Q: How do you handle failover in a sharded setup?**
- Each shard has its own primary + replicas + failover logic.
- Router (proxy or client) discovers new primary via service discovery (etcd, Consul).

**Q: What's split-brain and how do you prevent it?**
- Two primaries active simultaneously → divergent writes.
- Prevent: quorum-based election (Raft), fencing tokens, STONITH.

**Q: Explain read-your-writes consistency.**
- User writes → same user's next read must see it.
- Fixes: primary for post-write reads for N seconds; or LSN passing.

**Q: How does DynamoDB shard?**
- Partition key hashed to a physical partition. Each partition = 3000 RCU / 1000 WCU.
- Growing table → adaptive partitioning splits hot ones.
- Hot key that overflows one partition → adaptive capacity or table split.

**Q: Multi-region DB — how do you handle writes?**
- Option A: single primary region, others read-only. Cross-region writes slow.
- Option B: multi-leader with conflict resolution (Cassandra).
- Option C: NewSQL with paxos-style consensus (Spanner, CockroachDB). Slower writes but strong.

**Q: When would you *avoid* sharding?**
- Data + traffic fits on one big box (up to ~10 TB, ~50k QPS with Postgres).
- Access pattern involves rich cross-entity queries.
- Team can't afford the ops complexity.

**Q: What's a hot shard, and how do you fix it?**
- One shard receives disproportionate load (Beyoncé's tweets, one huge tenant).
- Fixes: sub-shard hot keys, move to dedicated shard (directory), aggressive caching, or read replicas within that shard.

## Real-World Examples
- **Twitter**: Manhattan (custom sharded KV) for tweets; range-sharded by userId + time.
- **Discord**: Cassandra + ScyllaDB for messages, sharded by channelId.
- **Instagram**: Postgres shards by userId + Vitess; heavy replication for feeds.
- **Facebook**: MySQL fleet with TAO (graph cache) in front; sharded by object ID.
- **Netflix**: Cassandra multi-region active-active with LOCAL_QUORUM.

## Gotchas
- **Adding a replica in a hot moment**: replica takes time to catch up; premature promotion loses data.
- **Failover races**: two "primaries" for a moment. Fence with tokens.
- **Cross-shard txn creep**: what starts as one shard becomes many after feature growth. Design for future.
- **Backup consistency**: sharded backups need coordination; take snapshots at close-enough LSNs.
- **Migration lock**: schema migration across shards can lock forever if not done in stages.
