# 14 — Consensus (Raft, Paxos, ZAB)

> Consensus = getting a group of nodes to agree on one value even when some fail. It's the "how" behind every CP system: strongly consistent DBs, distributed locks, config stores, leader election.

## Why We Need It

Distributed system must decide *something*:
- Who's the primary?
- What's the next log entry?
- Is this transaction committed?

Naive "just check with everyone" fails when nodes crash, messages drop, or the network partitions. Consensus algorithms give provable agreement under bounded failures.

## The Requirements (Safety + Liveness)

- **Agreement**: no two correct nodes decide different values.
- **Validity**: the decided value was proposed by someone.
- **Termination**: eventually a decision is reached (given a majority alive).

FLP impossibility (1985): in a fully async network with even one crash, no deterministic consensus algorithm can guarantee termination. Practical algorithms sidestep this with timeouts (assume partial synchrony).

## Paxos (Lamport, 1998)

The original. Correct but famously hard to explain.

**Roles:**
- **Proposer**: proposes a value.
- **Acceptor**: votes yes/no.
- **Learner**: learns the decided value.

**Two phases:**
1. **Prepare/Promise**: proposer picks number `n`, asks acceptors to promise not to accept lower `n`.
2. **Accept/Accepted**: if majority promised, proposer sends `(n, value)`; if majority accepts, value is decided.

Handles concurrent proposers via ballot numbers; the highest ballot wins.

**Multi-Paxos**: run Paxos repeatedly for a log of decisions; optimize by keeping a stable leader to skip phase 1.

**Where used**: Google Chubby, Spanner, some in-house systems. Rarely implemented from scratch — the paper is short, the correctness proofs are dense.

## Raft (Ongaro & Ousterhout, 2014)

Designed to be **understandable**. Same guarantees as Paxos, more approachable.

**Three roles:**
- **Leader**: single leader at a time, handles all client requests.
- **Follower**: passive, replicates from leader.
- **Candidate**: transitional state during election.

**Two RPCs:**
- **RequestVote**: candidate asks for votes.
- **AppendEntries**: leader replicates log entries (or heartbeats).

**Timing:**
- Followers have randomized election timeouts (150-300 ms).
- Miss heartbeat → become candidate → request votes.
- Win majority → become leader → send heartbeats.

**Log Replication:**
```
Client → Leader
Leader appends to its log
Leader sends AppendEntries to all followers
Followers append, ack
Once majority acked → leader commits, applies to state machine, replies to client
Followers commit on next heartbeat
```

**Safety** via:
- **Election restriction**: only vote for candidates whose log is at least as up-to-date as yours.
- **Log matching**: same index+term → same commands throughout log.
- **Commit rule**: leader can only commit entries from its own term.

**Where used**: etcd, Consul, CockroachDB, TiKV, RethinkDB, InfluxDB clustering, Kafka Raft (KRaft — replaces ZooKeeper).

## ZAB (ZooKeeper Atomic Broadcast)

Paxos-family; used by ZooKeeper. Optimized for single leader (called "primary") that broadcasts totally-ordered proposals.

Phases: leader election, discovery, synchronization, broadcast.

**Where used**: ZooKeeper itself (Kafka's original coordination, Hadoop, Solr).

## Comparison

| | Paxos | Raft | ZAB |
|---|---|---|---|
| Understandability | Hard | Easy | Medium |
| Leader | Optional (Multi-Paxos has one) | Explicit | Explicit |
| Log-based | Multi-Paxos | Yes | Yes |
| Primary use | Chubby, Spanner | etcd, Consul, CockroachDB | ZooKeeper |
| Correctness proof | 2 pages of dense math | Whole thesis | Paper |

For interviews: **know Raft cold**. Paxos gets a mention. ZAB rarely.

## Where Consensus Hides in Production

You often use consensus without realizing:

| Product | Consensus |
|---------|-----------|
| Kubernetes | etcd (Raft) for cluster state |
| Kafka (KRaft) | Raft for controller quorum |
| Kafka (pre-KRaft) | ZooKeeper (ZAB) |
| Cassandra LWT | Paxos for compare-and-set |
| Spanner | Paxos + TrueTime |
| CockroachDB | Raft per range |
| Elasticsearch (7.x+) | Custom Zen (Raft-like) |
| Consul | Raft |
| Vitess | Semi-sync MySQL + orchestrator |
| Google Chubby | Paxos |

## Leader Election

Simplest use of consensus: pick one node as leader, everyone else follows.

- **Raft**: built-in.
- **etcd/ZooKeeper**: register ephemeral node with your ID; lowest ID wins; watch for changes.
- **Redis Sentinel**: quorum-based leader election with fencing.

**Fencing tokens**: after election, new leader gets a monotonically increasing token. Downstream systems reject writes with lower tokens → prevents zombie leader from writing.

## Distributed Locks

Consensus enables correct distributed locks:
```
etcd: put with lease + revision → try-acquire.
     If revision incremented from your view → someone else holds.
Held for lease TTL; auto-release if client crashes.
```

Redlock (Redis) is famously fragile — use etcd, ZooKeeper, or a proper coordination service.

## Byzantine Fault Tolerance (BFT)

Raft/Paxos assume nodes crash but never lie. Byzantine assumes nodes may behave arbitrarily (malicious or corrupted).

- **PBFT** (Practical Byzantine Fault Tolerance): tolerates ⌊(N-1)/3⌋ Byzantine nodes.
- **Blockchains** (Bitcoin PoW, Ethereum PoS, Tendermint) are BFT consensus at internet scale.

Rarely relevant to internal enterprise systems (you trust your own nodes). Relevant for cross-organization consensus.

## Reference — Raft in Action (Kubernetes)

```
API Server writes to etcd:
  Client → etcd leader → propose in Raft log
  Leader → 2 followers: AppendEntries (majority = 2 of 3)
  Followers ack
  Leader commits, applies to state machine (BoltDB)
  Leader responds to API Server
  API Server responds to client
```

Every kubectl action goes through Raft. This is why etcd size and health are critical for cluster stability.

## Performance

- **Latency**: one round-trip to majority. LAN: ~1 ms. Cross-region: ~50-100 ms.
- **Throughput**: leader-bottlenecked. etcd: ~10k writes/sec. CockroachDB per-range: similar.
- **Read**: can be served from leader (linearizable) or any node (with lease + timestamp, "follower read").

**Optimizations:**
- **Batching** proposals into one round-trip.
- **Pipelining** — leader sends next entry before previous is acked.
- **Leader leases** — leader can serve reads without consensus round-trip for lease duration.
- **Follower reads** with timestamp: read from any replica at a bounded-stale time.

## Interview Q&A

**Q: What is Raft in one paragraph?**
- Raft elects a single leader via randomized timeouts and majority vote. All writes go through the leader, which appends to its log and replicates to followers. Once a majority has the entry, it's committed. If the leader dies, followers time out and start a new election. Safety properties (log matching, election restriction) ensure no two leaders in the same term and no committed entry ever changes.

**Q: Why is there always a leader in Raft?**
- Simplifies the protocol — no concurrent proposals, no ballot collisions.
- Read performance — leader can serve reads locally (with a lease).
- Bottleneck: leader is a single point of contention; scale by having multiple Raft groups (one per shard).

**Q: How does Raft ensure only one leader per term?**
- Each node votes at most once per term. Candidate needs majority. Two candidates in the same term can't both get majority (majority intersects).

**Q: What happens during a network partition?**
- Minority side can't elect a leader (no majority) → becomes unavailable for writes.
- Majority side elects a new leader (if old leader was on the minority side) → continues to serve.
- On heal: minority nodes sync missing entries from majority.

**Q: Why is majority the magic number (N/2 + 1)?**
- Any two majorities intersect → at least one node has the latest committed state → new leader can be found who knows it.

**Q: Can Raft be used for storing millions of QPS?**
- Not through a single Raft group (leader is bottleneck).
- Solution: many Raft groups, each per shard (CockroachDB, TiKV do this).

**Q: What is a lease read?**
- Leader believes it's still leader for the next N ms (lease). It serves reads locally without contacting followers. Correct as long as no new leader can be elected within the lease period (guaranteed by election timeout > lease).

**Q: Raft vs Paxos — is there a real difference in guarantees?**
- No. Both provide the same safety and liveness. Difference is understandability and structure.

## Real-World Examples
- **Kubernetes' etcd**: brains of every cluster. Raft ensures every node sees the same cluster state.
- **CockroachDB**: one Raft group per data range (~64 MB). Reshard = add/split ranges.
- **Kafka KRaft**: replaces ZooKeeper. Controller quorum uses Raft for metadata.
- **Consul**: service discovery + KV store, Raft-backed.
- **Google Spanner**: Paxos per data partition, TrueTime for external consistency.

## Gotchas
- **Etcd disk latency**: Raft commits require fsync. Slow disk → cluster latency. Use SSDs.
- **Quorum loss**: majority nodes down → cluster read-only. Backup + restore procedure critical.
- **Cluster size**: 3 or 5 nodes standard; more = higher latency per commit. 7+ rarely worth it.
- **Log growth**: unbounded log fills disk; take snapshots regularly.
- **Members change**: adding/removing nodes uses joint consensus; buggy implementations can lose safety.
- **Wall clock trust**: never use wall time for correctness in a Raft/Paxos state machine.
