# 15 — Consistent Hashing

> The clever trick that lets you add or remove a node from a cluster while only moving 1/N of the keys. Foundational for sharded databases, caches, and distributed load balancers.

## The Problem It Solves

Naive sharding: `shard = hash(key) % N`.

If `N` goes 4 → 5, almost every key gets a new shard. Cache: near-total invalidation. Storage: massive data reshuffle.

**Consistent hashing**: adding/removing one node moves *only 1/N* of keys.

## The Ring

1. Imagine a ring of positions [0, 2³²).
2. Each **node** is hashed onto the ring at position `hash(nodeId)`.
3. Each **key** is hashed onto the same ring at `hash(key)`.
4. A key belongs to the **first node clockwise** from its position.

```
                 0
              N1━┓
              ┃  ┃
        k7 ●  ┃  ┃  ● k1
              ┃  ┃
         N3━━━┫  ┣━━━N2
              ┃  ┃
        k5 ●  ┃  ┃  ● k3
              ┃  ┃
              N4━┛
                180

k1 → N2   (nearest CW)
k3 → N4
k5 → N4
k7 → N1
```

Adding N5 near k5's position only reroutes k5 → N5. Everyone else unaffected.

## Virtual Nodes (vnodes)

Naive ring: uneven load — one node may own a big arc, another a tiny one.

**Fix**: each physical node claims K positions on the ring (e.g., K=100-200). Now the load is smooth.

```
N1 → positions H(N1#0), H(N1#1), ..., H(N1#99)
N2 → positions H(N2#0), H(N2#1), ..., H(N2#99)
```

Load balance improves; standard deviation of load drops as K rises. K=100 gives ~10% imbalance; K=1000 gives ~3%.

## Adding / Removing a Node

**Add N5 with 100 vnodes:**
- Each vnode of N5 takes ownership of its arc from the previous owner.
- Data migration: for each new arc, copy the keys from previous owner to N5.
- Total moved: ~1/N of all keys, spread across many source nodes (parallelizable).

**Remove N3:**
- Vnodes of N3 disappear.
- Each ex-arc's clockwise successor now owns those keys.
- Copy data before removal (drain).

## Weighted Consistent Hashing

Nodes with more capacity get more vnodes.
```
N1 (small)  → 50 vnodes
N2 (medium) → 100 vnodes
N3 (big)    → 200 vnodes
```
Load proportional to weight.

## Where It's Used

| System | Purpose |
|--------|---------|
| **DynamoDB** | Partitioning key space across storage nodes |
| **Cassandra** | Token ring for partition placement |
| **Riak** | Same idea |
| **memcached** (client libs like ketama) | Client-side sharding across cache nodes |
| **Envoy / HAProxy** | Ring hashing LB for session affinity |
| **CDN cache selection** | Which PoP holds which content |
| **Distributed systems in general** | Any time you shard by key and expect membership change |

## Example — memcached with ketama

```
Client startup:
  ring = ConsistentHashRing()
  for host in servers:
    for i in 0..99:
      ring.add(hash(host + str(i)) % 2^32, host)

For each request:
  server = ring.get(hash(key))
```

When a cache node is added, only ~1/N of keys move → hit rate stays high during scale-out.

## Rendezvous Hashing (HRW) — Simpler Alternative

For each key, compute `hash(key + nodeId)` for every node, pick the highest.

- No ring, no vnodes.
- Same 1/N property.
- O(N) per lookup — fine for small N.
- Better balance than consistent hashing for small clusters.

Used by: some CDNs, memcached alternatives, ring-less hash routers.

## Jump Consistent Hash (Google, 2014)

Constant-time (no ring), no per-node state, minimal memory.

```java
int JumpConsistentHash(long key, int numBuckets) {
    long b = -1, j = 0;
    while (j < numBuckets) {
        b = j;
        key = key * 2862933555777941757L + 1;
        j = (long)((b + 1) * ((double)(1L << 31) / ((key >>> 33) + 1)));
    }
    return (int) b;
}
```

- 20 lines of code.
- Extremely fast.
- Limitation: only supports adding a node at the "end" (numBuckets++). Can't remove arbitrary node.

## Hot Spots — Consistent Hashing Doesn't Fix Them

If one key is hot (Beyoncé's tweets), it lives on one node → that node melts.

Consistent hashing spreads **keys** evenly, not **load per key**. Solutions:
- **Sub-shard hot keys**: `celeb:beyonce:v0`, `celeb:beyonce:v1`, ..., `celeb:beyonce:v99` — read random suffix.
- **Local caching** (L1) in the app.
- **Detect + reroute** hot keys to dedicated nodes.

## Reference — Cache Ring Behavior During Scale

Cluster of 4 memcached, 100 vnodes each. Hit rate stable at 90%.

**Add a 5th node:**
- ~20% of keys move (1/5).
- Cache hit rate temporarily drops to ~72% (20% become misses briefly).
- Recovers within minutes as new node fills up.

Contrast with `hash % N`: adding one node = 80% miss rate → DB blast → outage.

## Interview Q&A

**Q: Why not just `hash(key) % N`?**
- Adding or removing one node invalidates ~all placements.
- With consistent hashing, only 1/N moves.

**Q: What role do virtual nodes play?**
- Smooth the load distribution. Without vnodes, one node can end up owning a huge arc by chance.
- Also enable weighted distribution (more vnodes = more load).

**Q: How does consistent hashing handle a node failing suddenly (not gracefully)?**
- Clients detect via health check → treat the node as removed from ring.
- Requests reroute to the next node clockwise.
- Data recovery from replicas (Cassandra keeps N replicas; loss of primary → next replica serves).

**Q: Does consistent hashing require every node to know about every other?**
- Yes, membership must be consistent across clients / nodes.
- Cassandra: gossip protocol propagates membership.
- Cache clients: config-driven; membership changes propagated via config service.

**Q: Consistent hashing vs range-based sharding?**
- Consistent hashing: even distribution, no locality (adjacent keys go to different nodes).
- Range: locality (range scans efficient), risk of hot end.

**Q: How to migrate data when adding a node?**
- **Cassandra**: streams SSTables from prior owners to new node.
- **Cache**: no migration; just let old entries expire; new lookups populate new node.
- **Persistent store**: dual-write during migration, backfill, cut over.

**Q: What's rendezvous hashing and when do you use it?**
- Score each node with `hash(key, node)`, pick max.
- Simpler than ring, better small-cluster balance, no vnodes needed.
- O(N) per lookup — fine when N is small (< 100).

## Real-World Examples
- **DynamoDB**: partitioning uses consistent-hash-like scheme (Amazon's Dynamo paper 2007).
- **Cassandra**: token ring with 256 vnodes per node by default.
- **CDN edge assignment**: which PoP handles which origin URL (Akamai's `mapping` layer).
- **Discord's Elixir "guilds"**: guild ID → hashed to a specific Erlang node.
- **Envoy ring hash LB**: sticky session without cookies, via hash of a header.

## Gotchas
- **Bad hash function**: skewed distribution → some nodes overloaded. Use MurmurHash3 or SHA-1 (truncated).
- **Small vnode count**: still uneven. Use ≥ 100.
- **Membership skew**: different clients see different ring → same key routes differently → cache misses. Keep membership consistent.
- **Bootstrapping**: new node must be warmed with data before receiving traffic (avoid cold-node cache stampede).
- **Silent membership changes**: node comes back after being marked out; now belongs to a different arc. Coordinate cleanup.
- **Doesn't fix hot keys** — a common junior mistake. Consistent hashing is about placement, not load.
