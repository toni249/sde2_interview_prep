# 12 — Message Queues & Streaming

> Async messaging decouples producers from consumers in time and rate. Two broad families: **queues** (per-message, consumed once) and **streams** (append log, replayable).

## Queues vs Streams — The Core Distinction

| | Queue (RabbitMQ, SQS) | Stream (Kafka, Kinesis, Pulsar) |
|---|---|---|
| Data model | Discrete messages, deleted on ack | Append-only log; retention days-forever |
| Delivery | Once (or few) times | Multiple consumers each read independently |
| Ordering | Per-queue FIFO | Per-partition |
| Throughput | 10-100k/sec | 10M+/sec |
| Replay | Not natively | Rewind to any offset |
| Consumer model | Compete for messages | Consumer groups; each partition to one consumer |
| Use for | Work distribution, RPC-like | Event streaming, data pipelines |

**Rule**: if you want "one worker handles this task", queue. If you want "many independent systems react to this event, possibly replaying it", stream.

## RabbitMQ (Queue Family)

**Model**: producer → **exchange** → **binding** → **queue** → consumer.

- **Direct exchange**: exact routing key match.
- **Topic exchange**: wildcard match (`order.*.paid`).
- **Fanout exchange**: broadcast to all bound queues.
- **Headers exchange**: match on header values.

**Guarantees:**
- Publisher confirms (ack from broker).
- Consumer acks (broker keeps until acked).
- Persistent messages + durable queues survive restart.

**Pros**: rich routing, low latency, mature.
**Cons**: throughput ceiling ~50k msgs/sec/node; scaling is nontrivial.

## SQS (Cloud Queue)

- **Standard queues**: at-least-once, best-effort ordering, unlimited throughput.
- **FIFO queues**: exactly-once (5-min dedup window), strict order per MessageGroupId, up to 3000 TPS/group with batching.

**Model:**
- Send → invisible from other consumers.
- Receive → visibility timeout starts.
- Delete on success (implicit ack).
- If not deleted in time → message becomes visible again.

**DLQ (Dead Letter Queue)**: after N receive attempts, poison messages moved off main queue.

**Pros**: fully managed, autoscaling, cheap.
**Cons**: no ordering (standard), no complex routing, latency ~10-100 ms.

## Kafka (Stream Family)

**Model**: producer → **topic (partitioned, replicated log)** → consumer.

```
Topic "orders"
├── partition 0:  [msg-0 msg-1 msg-2 msg-3 ...]
├── partition 1:  [msg-0 msg-1 msg-2 ...]
└── partition 2:  [msg-0 msg-1 msg-2 msg-3 msg-4 ...]
```

- **Partition = ordered log** on disk; each message has a monotonically increasing offset.
- **Producer** picks partition by key (hash) or round-robin.
- **Consumer group**: partitions distributed to consumers in the group. Each partition → at most one consumer.
- **Retention**: time (7 days default) or size, per topic. Then old segments deleted (or compacted).

**Guarantees:**
- Per-partition ordering.
- At-least-once by default; exactly-once with transactional producer + consumer + idempotent producer.
- Configurable replication (default 3 replicas; `acks=all` waits for all in-sync replicas).

**Pros**: crazy throughput (millions/sec/broker), replay, durability, ecosystem (Streams, Connect, ksqlDB).
**Cons**: operational complexity (until KRaft, ZooKeeper too); ordering only per-partition; latency 10-100 ms typical.

## Pulsar — The New Kid

Apache Pulsar splits storage (BookKeeper) from serving (broker). Compute-storage separation.
- Multi-tenant natively.
- Geo-replication built-in.
- Supports queue *and* stream semantics.
- Higher operational burden than Kafka (managed offerings improve this).

## Delivery Semantics

**At-most-once**: fire and forget. Fast, may lose messages.
**At-least-once**: retry until ack. Duplicates possible → consumers must be idempotent.
**Exactly-once**: no dupes, no loss. Hard to achieve end-to-end; usually at-least-once + idempotent consumer.

Kafka "exactly-once semantics" (EOS) = transactional writes across topic+offset commit, within Kafka. Outside Kafka (writing to a DB), you're back to idempotency.

## Idempotent Consumers

Every consumer must handle duplicate delivery. Techniques:
- **Idempotency key**: message has `event_id`; consumer stores `processed_event_ids` with TTL.
- **Conditional writes** (`INSERT ... ON CONFLICT DO NOTHING`).
- **Natural idempotency**: `SET x=10` vs `INCR x`. Prefer the former.

## Ordering

- **RabbitMQ**: FIFO per queue.
- **SQS Standard**: no ordering.
- **SQS FIFO**: per MessageGroupId.
- **Kafka**: per partition. Keyed messages → same key → same partition → ordered.

**Common trap**: partition by userId for per-user ordering; want global ordering → single partition → no scale.

## Backpressure

Producer faster than consumer → messages pile up.

- **Kafka**: unbounded (until disk fills). Monitor `consumer lag`.
- **RabbitMQ**: publisher gets `Flow` (throttled), can drop or NACK.
- **SQS**: unbounded (up to 120k in-flight per queue).

**Response**:
- Scale consumers.
- Drop old messages (Kafka topic-level `max.message.bytes`, `retention.ms`).
- Add capacity headroom.

## Fanout Patterns

**Fanout across services**: Kafka topic — every consumer group gets full copy.

**Fanout across queues**: RabbitMQ fanout exchange → N queues.

**SNS → SQS fanout**: SNS topic notifies N SQS queues → decoupled fanout on AWS.

## Ordering vs Parallelism Trade-Off

Ordered per key → parallelism = # of distinct keys.

If 90% of traffic is one key → single-consumer bottleneck, other consumers idle.

Fixes:
- **Sub-partition** hot key by adding a random suffix (breaks strict order but distributes).
- **Batch and merge** downstream.
- Accept that hot keys serialize.

## Kafka Advanced Features

- **Log compaction**: keep only the latest value per key. Effectively a KV store snapshot in a log.
- **Kafka Streams**: build stream-processing apps (state stores, windowed joins).
- **Kafka Connect**: pluggable source/sink connectors (Debezium for CDC, S3 sink, JDBC, etc.).
- **ksqlDB**: SQL over Kafka streams.

## Reference — E-commerce Event Bus

```
OrderService  →  Kafka topic "orders"
                     │
                     ├──► InventoryService (consumer group)
                     ├──► PaymentService
                     ├──► FulfillmentService
                     ├──► AnalyticsPipeline → Snowflake
                     └──► SearchIndexer → Elasticsearch
```

Each downstream is independent, can replay from offset zero for backfill.

## Common Bugs

**Consumer lag creeping**: consumer slower than producer. Alarm at 30-minute lag; auto-scale on that metric.

**Rebalance storms**: consumer joins/leaves triggers partition reassignment; all consumers pause briefly. Tune `session.timeout.ms` and use static membership.

**Poison message loops**: bad message crashes consumer, is redelivered forever. DLQ + alerting.

**Unbounded retention**: forgot to set `retention.ms`; disk fills. Monitor.

**Missing ack**: consumer completes work, crashes before ack → message replayed → duplicate side effect. Idempotency is mandatory.

## Interview Q&A

**Q: Kafka vs SQS?**
- SQS: managed, decoupled, single-consumer semantics. Great for work queues, background jobs.
- Kafka: multi-consumer, event streaming, replay, higher throughput. Great for data pipelines, event sourcing.
- If in doubt on AWS with simple work queue → SQS.
- If multiple systems react to same events → Kafka.

**Q: When would you use a queue over a stream?**
- Job distribution (transcode this video). One worker only.
- Batch retries with visibility timeout semantics.
- Simple pub-sub without replay needs.

**Q: How does Kafka achieve high throughput?**
- Sequential disk writes (SSD sequential > RAM random on some benchmarks).
- Zero-copy transfer (`sendfile`) from disk to network.
- Batching (produce and consume in batches).
- Partitioning for parallelism.

**Q: What is exactly-once really?**
- End-to-end EOS is a myth for cross-system.
- Inside Kafka: transactional producer + `read_committed` consumer + atomic offset commit.
- Across systems: at-least-once + idempotent consumer = effectively once.

**Q: How to guarantee ordering across a partition change?**
- You can't. Ordering is per partition.
- Design keys so related events share a partition (`userId`, `orderId`).

**Q: Kafka message size limits?**
- Default 1 MB (`message.max.bytes`). Larger → store in S3, put pointer in Kafka.
- Similar for SQS (256 KB) and RabbitMQ.

**Q: What's the outbox pattern?**
- App writes to DB + outbox table in one transaction.
- Poller / CDC publishes outbox to Kafka.
- Guarantees "event published iff DB write committed" — see [Pattern 7](../patterns/07_multi_step_processes.md).

**Q: How do you migrate from RabbitMQ to Kafka?**
- Bridge: Kafka Connect consumer reads from RabbitMQ, publishes to Kafka.
- Dual-publish period; downstreams switch one by one.
- Verify parity, then decommission RabbitMQ.

## Real-World Examples
- **LinkedIn**: invented Kafka; billions of events/day for data infra.
- **Uber**: Kafka for logs, metrics, and event pipeline (later Peloton for stream processing).
- **Netflix**: Kafka + Flink for real-time recommendations.
- **Stripe**: RabbitMQ for internal async work.
- **Airbnb**: mix of SQS (jobs), Kafka (events), Airflow (batch orchestration).

## Gotchas
- **Consumer group + subscription**: adding/removing consumers rebalances all partitions. Costly during peak.
- **Kafka retention vs consumer lag**: if lag exceeds retention, messages deleted before read → data loss.
- **Message ordering with retries**: retrying a failed message later breaks strict order. Design for it.
- **DLQ silence**: nobody watches. Alert on non-zero DLQ depth.
- **Serialization schemas**: Avro/Protobuf schema evolution across versions. Use Schema Registry.
- **Cost**: Kafka storage in cloud is not free. Set retention wisely.
