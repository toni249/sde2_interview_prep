# 13 — CAP & Consistency Models

> CAP is the two-sentence tweet of distributed systems. The full story — consistency models, PACELC, quorum math — is what you actually need.

## CAP Theorem (Brewer)

> In the presence of a network **P**artition, a distributed system must choose between **C**onsistency and **A**vailability. You cannot have all three.

- **Consistency (C)**: every read returns the most recent successful write (or an error). *Linearizability*, in academic terms.
- **Availability (A)**: every request receives a (non-error) response.
- **Partition tolerance (P)**: system continues operating despite network splits.

Since real networks *will* partition, the practical choice is **CP** or **AP**.

- **CP** (Consistency + Partition tolerance): during a partition, refuse some requests to preserve consistency. Examples: ZooKeeper, etcd, HBase, MongoDB (with majority writes), Spanner.
- **AP** (Availability + Partition tolerance): during a partition, serve possibly-stale data. Examples: Cassandra (default), DynamoDB (default), Riak, Couchbase.

CAP's "C" is **strong** consistency. Many "eventually consistent" systems are actually AP but converge quickly.

## PACELC — CAP's Better Cousin

> When there IS no partition, you STILL trade **L**atency vs **C**onsistency. When there is one, you trade C vs A.

**PACELC** = if Partition (P) then A or C, Else (E) L or C.

| System | Partition | Normal |
|--------|-----------|--------|
| Cassandra (default) | PA | EL |
| DynamoDB | PA | EL |
| Spanner | PC | EC |
| MySQL (single primary) | PC | EC |
| Cosmos DB | Tunable | Tunable |

**Insight**: Cassandra chooses low latency over strong consistency *even when the network is fine*. That's a design decision, not a partition emergency.

## Consistency Models — the Real Spectrum

Not binary. From strongest to weakest:

| Model | Guarantee | Example |
|-------|-----------|---------|
| **Linearizability** | Reads see writes in real-time order; behaves as if there's one copy | Spanner, etcd |
| **Sequential consistency** | All processes see same order, but not necessarily real-time | Rare in practice |
| **Causal consistency** | Related writes seen in cause-effect order; unrelated may differ | COPS, some Riak configs |
| **Read-your-writes** | You see your own writes; others may not yet | Session-consistent stores |
| **Monotonic reads** | Once you've seen a value, you won't see older | Common tuning |
| **Bounded staleness** | Reads at most K seconds / N updates stale | Cosmos DB |
| **Eventual consistency** | Given no new writes, replicas converge | Cassandra, DynamoDB default |

**Strong consistency** casually means "linearizable" but is often loose. Ask what the guarantee actually is.

## Common Anomalies

**Read-your-writes violation** — you write "hello", refresh, and see empty. (Async replica lag.)

**Monotonic reads violation** — you see comment count 5, then 4. (Reads hit different replicas.)

**Lost update** — two clients read, modify, write; last write silently overwrites the other. (Missing OCC/pessimistic lock.)

**Dirty read** — one txn reads uncommitted data from another. (Isolation level too low.)

**Non-repeatable read** — same query in one txn returns different data. (No repeatable-read.)

**Phantom read** — same range query returns different rows. (Missing gap locks / serializable isolation.)

## Isolation Levels (Single-Node RDBMS)

| Level | Prevents |
|-------|----------|
| Read Uncommitted | (nothing extra) |
| Read Committed | Dirty reads |
| Repeatable Read | + non-repeatable reads |
| Serializable | + phantoms; behaves as if txns run serially |

Snapshot Isolation ≈ Repeatable Read for most workloads; Postgres implements Serializable with SSI (serializable snapshot isolation).

## Quorum Math

With N replicas, W ack writers, R ack readers.

- **W + R > N** → guaranteed overlap → **consistent reads**.
- **W = N, R = 1** → fast reads, slow writes.
- **W = 1, R = N** → fast writes, slow reads.
- **W = R = ⌈(N+1)/2⌉** → majority quorum → typical balanced choice.
- **W + R ≤ N** → may read stale (eventual consistency).

Cassandra: `LOCAL_QUORUM` = majority in local DC. `EACH_QUORUM` = majority in every DC (slow, cross-region).

## Consensus vs Consistency

- **Consensus** (Raft, Paxos) — mechanism for a group of nodes to agree on a value.
- **Consistency** — property observed by clients.

Consensus is *how* CP systems achieve strong consistency. See [14](14_consensus.md).

## Practical Consistency Trade-Offs

**Bank balance**: strong consistency mandatory. Use CP DB or strict quorum.

**Social feed**: eventual OK — user sees a friend's post seconds later, no big deal.

**Product inventory** (last item): strong consistency for the "decrement" (avoid oversell); eventual OK for the display counter.

**Distributed lock**: strong consistency; use etcd / ZooKeeper with fencing tokens.

**Follower count**: eventual — approximate is fine.

## Multi-Region Strong Consistency — What It Costs

- Every write must be acknowledged by a quorum across regions.
- If regions are 100 ms apart, every write costs 100 ms minimum.
- Failover works but is slower.
- Spanner: uses atomic clocks + TrueTime to bound uncertainty; still latency-heavy for cross-region writes.

**Alternative**: single primary region + eventually-consistent regional replicas + route writes to primary. Cheaper but async.

## Reference — Making a Consistency Decision

Ask:
1. **What's the cost of a wrong answer?** (Money → strong; opinion → eventual.)
2. **What's the tolerance for staleness?** (Seconds vs minutes vs hours.)
3. **How often is the field written vs read?** (Read-heavy → cache more aggressively.)
4. **Is there a user-visible action after read?** (User writes and re-reads → read-your-writes needed.)
5. **Multi-region?** (Adds latency budget to strong consistency.)

## Interview Q&A

**Q: Is CAP still useful in modern systems?**
- The letter of CAP is outdated (all real systems are P). The spirit — "you can't have all three" — remains a mental hook. Use PACELC for a fuller picture.

**Q: Is Kafka CP or AP?**
- With `acks=all` and in-sync-replica requirement: CP-ish (won't ack when quorum lost).
- With `acks=1`: AP-ish (can lose data on broker crash).
- Really depends on config.

**Q: How can DynamoDB be both eventually consistent AND offer strong reads?**
- Default reads are eventually consistent (fast, cheap).
- Add `ConsistentRead=true` for strong reads (slower, 2x cost).
- Under the hood: reads from any replica normally; from leader when strong requested.

**Q: What does "strong consistency" actually mean?**
- Usually linearizability: any read sees the effect of any prior successful write.
- Some vendors mean "sequential consistency" — weaker but often called "strong".

**Q: How do you handle read-your-writes on an async replica setup?**
- Route the writer's next reads to primary (sticky by userId).
- Track LSN and wait until replica catches up.
- Or use synchronous replication for critical writes.

**Q: What's causal consistency and where is it used?**
- If A causes B, everyone sees A before B. Unrelated events (D, E) can be seen in any order.
- Comments: seeing a reply implies you saw the parent.
- Facebook TAO, some Riak configs.

**Q: When would you accept eventual consistency in payments?**
- Never for the actual money movement.
- Yes for reporting/UI (dashboard shows balance updated within seconds is OK).
- Even Stripe's internal ledger is strongly consistent, but user-facing dashboards may lag.

**Q: Explain CAP through Cassandra vs MongoDB.**
- Cassandra AP: writes accepted anywhere; conflicts resolved by timestamp (LWW); replicas converge.
- MongoDB CP (with majority writes): primary needed; if partitioned, minority becomes read-only.

## Real-World Examples

- **Google Spanner**: linearizable across the globe using TrueTime.
- **DynamoDB**: eventually consistent default, strong on request; multi-region strong via Global Tables 2.0.
- **Cassandra at Netflix**: LOCAL_QUORUM per DC, multi-DC async replication.
- **Amazon DynamoDB (2007 paper)**: showed AP can be practical at scale.
- **ZooKeeper / etcd**: CP by design; used for locks and config, not high-throughput data.

## Gotchas
- **Assuming "strong" without checking** — many DBs weaken it under the hood.
- **W+R>N is not always enough**: read-repair may still return stale on failure paths.
- **Clock skew** breaks LWW (Cassandra timestamps). Use NTP + monotonic clocks; don't trust wall time for causality.
- **Split-brain** during partition: two minority partitions each accept writes → divergence. Requires manual reconciliation.
- **Isolation level default varies**: Postgres = Read Committed, MySQL InnoDB = Repeatable Read. Know which you have.
