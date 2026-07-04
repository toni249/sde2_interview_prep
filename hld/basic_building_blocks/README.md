# Basic Building Blocks — HLD Core Concepts

The foundational primitives every system-design interview assumes you know cold. Each file is a standalone deep-dive with comparisons, trade-offs, and interview follow-ups.

## Curriculum

### Networking & Protocols
| # | Topic | What you'll learn |
|---|-------|-------------------|
| 01 | [Networking Basics](01_networking_basics.md) | IP, TCP, UDP, DNS, TLS, latency numbers every SDE should know |
| 02 | [HTTP Evolution](02_http_evolution.md) | HTTP/1.1 → 2 → 3, keep-alive, multiplexing, QUIC, HTTPS handshake |
| 03 | [API Styles](03_api_styles.md) | REST, GraphQL, gRPC, RPC — when to pick which |
| 04 | [Real-Time Protocols](04_realtime_protocols.md) | Polling, long-poll, SSE, WebSockets, WebRTC |

### Traffic Layer
| # | Topic | What you'll learn |
|---|-------|-------------------|
| 05 | [Load Balancers](05_load_balancers.md) | L4 vs L7, algorithms, health checks, sticky sessions |
| 06 | [Proxies](06_proxies.md) | Forward, reverse, transparent — where each fits |
| 07 | [API Gateway & BFF](07_api_gateway_and_bff.md) | Gateway responsibilities, Backend-for-Frontend pattern |
| 08 | [CDN](08_cdn.md) | How CDNs work, cache keys, edge compute |

### Data Layer
| # | Topic | What you'll learn |
|---|-------|-------------------|
| 09 | [Databases Overview](09_databases.md) | SQL vs NoSQL, KV/doc/wide-col/graph, picking a store |
| 10 | [Replication & Sharding](10_replication_sharding.md) | Leader-follower, quorum, sharding strategies |
| 11 | [Caching](11_caching.md) | Patterns, eviction, tiers, invalidation |

### Async & Messaging
| # | Topic | What you'll learn |
|---|-------|-------------------|
| 12 | [Message Queues & Streaming](12_message_queues_streaming.md) | SQS/RabbitMQ vs Kafka; delivery semantics |

### Distributed Systems Theory
| # | Topic | What you'll learn |
|---|-------|-------------------|
| 13 | [CAP & Consistency Models](13_cap_consistency.md) | CAP, PACELC, strong / eventual / causal, quorum math |
| 14 | [Consensus](14_consensus.md) | Raft, Paxos, Zab — where they hide in prod |
| 15 | [Consistent Hashing](15_consistent_hashing.md) | The ring, virtual nodes, rebalancing |
| 16 | [Probabilistic Data Structures](16_bloom_and_probabilistic.md) | Bloom filter, HyperLogLog, Count-Min Sketch |

### Reliability
| # | Topic | What you'll learn |
|---|-------|-------------------|
| 17 | [Rate Limiting](17_rate_limiting.md) | Token bucket, leaky bucket, sliding window, where to enforce |
| 18 | [Reliability Patterns](18_reliability_patterns.md) | Circuit breaker, retries + backoff, bulkhead, idempotency |

## Suggested Reading Order

1. **Foundations first** — 01, 02, 05, 09, 11 (protocols, LB, DB, cache)
2. **Comm styles** — 03, 04 (API styles, real-time)
3. **Traffic layer depth** — 06, 07, 08 (proxies, gateway, CDN)
4. **Data at scale** — 10 (replication/sharding)
5. **Async** — 12 (queues, streaming)
6. **Theory** — 13, 14, 15, 16 (CAP, consensus, hashing, probabilistic)
7. **Reliability** — 17, 18 (rate limits, circuit breakers)

## Related Study
- [System Design Patterns](../patterns/README.md) — how to combine these blocks into solutions
- [Rate Limiter LLD](../../java-lld/LLD_06_Rate_Limiter.md)
- [LRU Cache LLD](../../java-lld/LLD_07_LRU_Cache.md)
