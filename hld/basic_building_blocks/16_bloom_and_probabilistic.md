# 16 — Probabilistic Data Structures

> When exactness is expensive and approximation is cheap, probabilistic data structures let you answer set membership, counts, and quantiles with tiny memory and known error bounds. Everywhere from DB internals to analytics to CDNs.

## The Four to Know

| Structure | Question | Error | Memory | Insert / Query |
|-----------|----------|-------|--------|----------------|
| **Bloom Filter** | Is X in the set? | False positives, no false negatives | ~10 bits per element | O(k) |
| **Count-Min Sketch** | How many times has X appeared? | Overestimates | KB-MB | O(k) |
| **HyperLogLog (HLL)** | How many *distinct* items? | ~2% relative error | 12 KB for billions | O(1) |
| **Cuckoo Filter** | Membership (with deletes) | False positives, supports deletion | Slightly better than Bloom | O(1) amortized |

## Bloom Filter

**Structure**: bit array of size `m`, `k` hash functions.

**Insert** `x`: set bits at positions `h1(x), h2(x), ..., hk(x)`.

**Query** `x`: check those `k` positions. All set → probably present. Any unset → definitely absent.

**False positive rate** (approximately):
```
p ≈ (1 - e^(-kn/m))^k
```
where `n` = items inserted.

**Optimal `k` for given `m/n`**: `k = (m/n) * ln(2)`.

**Rule of thumb**: for 1% false positive rate, use **~9.6 bits per item** with **7 hash functions**.

```
1M items, target 1% FPR:
  m = 1M * 9.6 = 9.6M bits = 1.2 MB
  k = 7 hash functions
```

Compare: storing 1M SHA-1 hashes (20 bytes each) = 20 MB. Bloom is ~15× smaller.

### Use Cases

- **DB read optimization**: LSM engines (Cassandra, RocksDB) check Bloom before hitting disk — skip lookups for keys not in an SSTable.
- **CDN cache**: is this URL cached anywhere? Bloom filter avoids origin roundtrip.
- **Web crawlers**: URL "seen before" check.
- **Chrome Safe Browsing**: bloom filter of malicious URLs shipped to browser; full check only on positive.
- **Spell checkers**: is this word a possible correction?
- **Anti-fraud**: is this user/device in a suspicious set?

### Limitations
- **No deletion** (removing a bit could affect other keys). Counting Bloom filter allows delete (multi-bit counters, more memory).
- **Sizing**: pick `m` upfront. Growing requires rebuild or scalable bloom filters.
- **Positive means "maybe"** — application must handle it (usually fine, just an extra check).

## Cuckoo Filter

Newer alternative to Bloom.
- Supports **deletions**.
- Slightly better space efficiency at same FPR.
- Uses **cuckoo hashing** (two hash functions; kick out existing entries on collision).
- Slower inserts near full; harder to implement correctly.

Use when you need deletion or want ~10-20% memory savings vs Bloom.

## Count-Min Sketch

**Structure**: 2D array `d × w` counters; `d` hash functions.

**Increment** `x`: for each of `d` rows, `counter[i][hi(x) mod w] += 1`.

**Query** `count(x)`: return `min(counter[i][hi(x) mod w])` across rows.

**Error bound**: overestimates by at most `ε × total_count` with probability `1 - δ`, using:
```
w = ⌈e / ε⌉   (columns)
d = ⌈ln(1/δ)⌉ (rows)
```

Example: 10M events, want error ≤ 100 (0.001% of total), δ = 0.001 → `w=2718, d=7` → ~19K counters × 4 bytes = 76 KB.

### Use Cases

- **Heavy hitters**: which URLs / items are trending? Iterate all seen keys with count above threshold.
- **Ad/analytics counters**: approximate view/click counts per user in low memory.
- **DDoS detection**: count requests per source IP without storing every IP.
- **Kafka/Spark analytics** with streaming.

## HyperLogLog (HLL)

Estimates cardinality (# of distinct items) of a huge stream with very little memory.

**Idea**: hash each item; count leading zeros in binary. Longer runs of zeros = rare = more distinct items seen.

**Structure**: `m` "registers" tracking max leading zero count per hash bucket.
- Standard: `m = 2^14 = 16384` registers × 6 bits ≈ 12 KB.
- Estimates up to billions of distinct with ~2% error.

**Union of HLLs** (crucial superpower):
```
HLL_A ⊔ HLL_B = HLL(union of items A and B)
Compute by taking register-wise max.
```
→ **Distributed cardinality** at zero communication cost after combining.

### Use Cases

- **Unique visitors**: Redis `PFADD`, `PFCOUNT`, `PFMERGE`.
- **BigQuery `APPROX_COUNT_DISTINCT`**: HLL under the hood.
- **Twitter, Reddit**: unique user counts on posts.
- **Ad tech**: unique reach across campaigns.

## Skip Lists (Not Probabilistic Membership But Common)

Layered linked list with randomly-added express lanes. Gives O(log n) search / insert / delete with simpler code than balanced trees.

Used by: Redis sorted sets, LevelDB memtable, ConcurrentSkipListMap in Java.

## Top-K Structures

**Space-Saving algorithm**: maintain K counters. On new item:
- If in list → increment.
- Else if space → add with count 1.
- Else → replace the smallest counter with new item, incrementing.

Approximates top-K frequent items in one pass, O(K) memory.

Used by: Redis Stream `XADD` + top-K commands (`TOPK.RESERVE`), analytics tools.

## t-digest / GK Sketch — Quantile Estimation

Approximate percentiles (p50, p95, p99) over a stream in constant memory.

- **t-digest**: clusters of samples, denser near the tails (great for p99).
- **GK** (Greenwald-Khanna): older, error-bounded.

Used by: DataDog, Prometheus (histograms), Elastic percentiles agg.

## Reference — When to Use What

```
"Is X in the set?"             → Bloom filter / Cuckoo filter
"How many times has X been?"   → Count-Min Sketch
"How many distinct items?"     → HyperLogLog
"Top K most frequent?"         → Space-Saving / Frequent
"What's the p99 latency?"      → t-digest / HDR Histogram
```

## Interview Q&A

**Q: Why use a Bloom filter over a HashSet?**
- 10-100× less memory.
- Constant-size regardless of item size (small hash, not full key).
- Accepts small false-positive rate.
- When exactness isn't required or is checked afterward.

**Q: Can Bloom filters have false negatives?**
- No. If all `k` bits are set, item was inserted. But bits can be shared with other items → false positive possible.
- Never false negative → **safe to use as a "pre-check"** before an exact lookup.

**Q: How do you delete from a Bloom filter?**
- You can't (would falsify other keys' bits).
- **Counting Bloom filter**: each position is a counter, increment on add, decrement on delete. Uses 4× more memory.
- Or use **Cuckoo Filter**.

**Q: Why does Cassandra use Bloom filters?**
- Read lookup can hit many SSTables. Bloom per SSTable rejects "definitely not here" without disk I/O.
- Huge speedup for read-mostly workloads.

**Q: Combining HLLs across shards — how?**
- Each shard maintains its own HLL. Merge = register-wise max. Union preserves cardinality estimate.
- Used at scale in analytics: each map task emits an HLL, reduce merges them.

**Q: What's the memory-error tradeoff for HLL?**
- Doubling registers halves relative error (× 1/√2 roughly).
- 4 KB → ~4% error; 16 KB → ~2%; 64 KB → ~1%.

**Q: When is exact counting better than HLL?**
- When you need exact answer (billing, compliance).
- When cardinality is small (< 10k) — a HashSet is fine.
- When you must enumerate elements, not just count.

**Q: Count-Min Sketch overestimates but never underestimates — why?**
- Every item's counter is incremented on insert. Collisions add extras but never subtract.
- Min across rows reduces error but preserves the ceiling.

## Real-World Examples

- **Chrome Safe Browsing**: bloom of malicious URLs; positive hits do exact server check.
- **Bitly**: bloom for existing short-URL keys.
- **Postgres**: Bloom filter extension for skipping table scans.
- **Cassandra / ScyllaDB**: bloom per SSTable for reads.
- **Redis**: `PFADD`/`PFCOUNT` — HLL; `TOPK.*` — top-K.
- **BigQuery / Redshift / Snowflake**: `APPROX_COUNT_DISTINCT` = HLL.
- **DataDog / Prometheus**: t-digest / histograms for latency quantiles.
- **Presto / Trino**: uses HLL for `approx_distinct()`.

## Gotchas
- **False-positive tolerance**: verify downstream can handle it. If a false positive means "expensive lookup", fine. If it means "block a legitimate user", bad — need exact check.
- **Hash function quality**: cheap `String.hashCode()` produces skew. Use MurmurHash3 or xxHash.
- **Sizing upfront**: too small → FPR climbs to useless. Estimate max n × 2 for safety.
- **Serialization**: Bloom filter is fast to check, but ship the *filter*, not the members. Watch schema evolution.
- **Combining sketches** requires same parameters (same `m`, `k`, hash functions). Coordinate across services.
- **Time-decay**: production HLLs / Bloom filters over "unique visitors" fill up over time. Reset periodically or use windowed variants.
