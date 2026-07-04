# 09 — Databases: Types & When to Pick Which

> The DB choice is the single biggest architectural decision. Know the taxonomy, the CAP trade-offs each family makes, and match to access patterns.

## The Big Split

| Family | Data Model | Examples | Best For |
|--------|-----------|----------|----------|
| **Relational (SQL)** | Rows in tables with schema | Postgres, MySQL, SQL Server, Oracle | Transactions, complex queries, joins |
| **Key-Value** | `key → blob` | Redis, DynamoDB, Memcached, RocksDB | Simple lookups, caching, session |
| **Document** | `key → JSON/BSON` | MongoDB, CouchDB, DocumentDB | Flexible schemas, nested data |
| **Wide-Column** | Sparse table, wide rows | Cassandra, ScyllaDB, HBase, Bigtable | High write throughput, time-series |
| **Graph** | Nodes + edges | Neo4j, Neptune, JanusGraph | Relationships, recommendations, fraud |
| **Search** | Inverted index | Elasticsearch, OpenSearch, Solr | Full-text, ranked, faceted |
| **Time-Series** | Time-indexed metrics | InfluxDB, TimescaleDB, VictoriaMetrics, Prometheus | Metrics, IoT, logs |
| **Columnar / OLAP** | Column-oriented, batch analytics | BigQuery, Snowflake, Redshift, ClickHouse, DuckDB | Analytics, aggregations over billions of rows |
| **NewSQL** | SQL + horizontal scale | CockroachDB, Spanner, Vitess, TiDB, Yugabyte | Global SQL, strong consistency |
| **Vector** | High-dim vectors, ANN search | Pinecone, Weaviate, Milvus, pgvector | ML embeddings, semantic search |

## SQL (Relational) — The Default

**Strengths:**
- ACID transactions.
- Rich query language (joins, window functions, CTEs).
- Mature ecosystem, tooling, ops experience.
- Strong consistency out of the box.
- Referential integrity (foreign keys).

**Weaknesses:**
- Vertical scaling has a ceiling (~64 vCPU, 500 GB RAM for a big box).
- Sharding painful (Vitess, Citus help).
- Schema migrations require care in prod.

**When:** OLTP, financial data, anything where a wrong answer costs money. **Default choice unless you have a specific reason otherwise.**

## Key-Value

**Model**: `PUT key value; GET key; DELETE key`.

**In-memory** (Redis, Memcached): µs latency, capacity limited by RAM.
**On-disk** (DynamoDB, RocksDB): TB scale, ms latency.

**When:**
- Session store.
- Cache (see [11](11_caching.md)).
- Feature flags, rate-limit counters.
- User preferences by ID.

**Downside**: no queries beyond key lookup (secondary indexes possible but limited). No joins.

## Document

**Model**: JSON blobs keyed by ID; some support secondary indexes and queries into nested fields.

```json
{
  "_id": "user42",
  "name": "Alice",
  "orders": [{"id": 1, "total": 100}, {"id": 2, "total": 50}],
  "prefs": {"theme": "dark"}
}
```

**When:**
- Schema evolves rapidly (early-stage product).
- Natural nested structure (product with variants).
- Read the whole document at once.

**Downside**: joins across documents are manual and slow. Consistency across documents is application logic.

**MongoDB trend note**: modern Mongo supports transactions and even sharding, blurring the line with relational.

## Wide-Column (Cassandra family)

**Model**: table with primary key = `(partition_key, clustering_key)`. Very fast reads/writes if you query by partition key.

**When:**
- Massive write throughput (100k+ writes/sec/node).
- Time-series or event data partitioned by ID + time.
- Multi-DC replication with tunable consistency.
- Netflix viewing history, Discord messages, Instagram feed.

**Downside**: query patterns must be known upfront (design schema around queries). No ad-hoc analytics. Secondary indexes are a footgun.

## Graph

**Model**: nodes and edges as first-class citizens.

```
(User)-[FOLLOWS]->(User)
(User)-[PURCHASED]->(Product)
(Product)-[SIMILAR_TO]->(Product)
```

**When:**
- Traversals of depth 3+ (friend-of-friend, fraud rings).
- Recommendations based on shared connections.
- Knowledge graphs.

**Downside**: rarely the primary DB; often augments a relational store.

## Search

**Model**: inverted index (`term → posting list of doc IDs`), TF-IDF or BM25 ranking, faceting.

**When:**
- Full-text search (product catalog, docs, logs).
- Complex filters + relevance ranking.
- Log aggregation (ELK stack).

**Downside**: eventual consistency (indexing lag), not source of truth.

## Time-Series

**Model**: `(timestamp, series_id, value)` — highly compressed, downsampled at older tiers.

**When:**
- Metrics (Prometheus).
- IoT sensor data.
- Financial ticks.
- Application observability.

**Downside**: writes only append (updates rare), not a general-purpose DB.

## OLAP / Columnar

**Model**: columns stored contiguously → aggregations skip unread columns, compress well.

**When:**
- Analytics (dashboards, reports).
- Ad-hoc queries over billions of rows.
- Batch pipelines.

**Downside**: high latency for point reads; bad for OLTP. Usually a **secondary** store fed by CDC from primary DB.

## NewSQL

**Idea**: give me SQL semantics with horizontal scale.

- **Spanner** (Google): globally strong consistency using TrueTime.
- **CockroachDB**: Spanner-inspired, open-source.
- **Vitess**: MySQL sharding at YouTube scale.
- **TiDB / Yugabyte**: hybrid HTAP.

**When:** SQL + scale beyond what Postgres/MySQL can vertically deliver, but you still want transactions across shards.

**Downside**: young; ops still learning; more expensive than plain Postgres.

## Vector DB

**Model**: high-dimensional vectors (embeddings from an ML model), approximate nearest-neighbor (ANN) search using HNSW, IVF, or ScaNN.

**When:**
- Semantic search over text/images.
- RAG (retrieval-augmented generation) for LLM apps.
- Recommendation systems.

**Options**: dedicated (Pinecone, Weaviate, Milvus) or extensions (pgvector, Elastic dense_vector).

## OLTP vs OLAP

| | OLTP | OLAP |
|---|---|---|
| Workload | Many small txns | Few large scans |
| Users | End users | Analysts / BI |
| Row model | Row-oriented | Column-oriented |
| Consistency | Strong | Eventual OK |
| Examples | Postgres, MySQL, DynamoDB | Snowflake, BigQuery, ClickHouse |

**Rule:** never run analytics on your OLTP DB. Use CDC (Debezium, Kafka Connect) to stream to an analytics store.

## ACID vs BASE

**ACID** (traditional RDBMS):
- **A**tomic — all or nothing.
- **C**onsistent — respects constraints.
- **I**solated — concurrent txns look serial.
- **D**urable — committed data survives crashes.

**BASE** (many NoSQL):
- **B**asically **A**vailable.
- **S**oft state — may change without input.
- **E**ventually consistent — converges eventually.

## Choosing Framework

Ask:
1. **Access pattern**: point reads, range scans, aggregations, full-text?
2. **Scale**: rows, QPS, size in bytes?
3. **Consistency need**: transactional (money) or eventual (feed)?
4. **Availability need**: multi-region? Cross-region latency budget?
5. **Query flexibility**: known ahead vs ad-hoc?
6. **Schema volatility**: rigid vs evolving?

Then:
- Money involved + complex queries? → SQL.
- Simple key lookups at insane scale? → DynamoDB / Cassandra.
- Search? → Elastic beside primary DB.
- Analytics? → columnar beside primary DB.
- Metrics? → Prometheus / VictoriaMetrics.
- Nested docs, no fixed schema? → Mongo.
- Graph traversals? → Neo4j.

## Common Multi-Store Architecture

```
Primary OLTP (Postgres)
   │
   │  CDC (Debezium → Kafka)
   ├───► Elasticsearch  (search)
   ├───► Redis          (cache)
   ├───► Snowflake      (analytics)
   ├───► Vector DB      (semantic search)
   └───► S3 data lake   (archival)
```

## Interview Q&A

**Q: SQL vs NoSQL — how do you decide?**
- Start SQL. Move to NoSQL only when specific reasons force it: scale beyond single node, schema flexibility, or access pattern that SQL indexes poorly.
- Not either/or; most systems have both.

**Q: When does Postgres stop being enough?**
- Sustained > 10-50k writes/sec on a single primary.
- Working set no longer fits in RAM (~500 GB).
- Multi-region strong consistency (Postgres is single-primary).
- Then consider: read replicas → sharded Postgres (Citus) → different DB per hot data (Cassandra, DynamoDB) → NewSQL (CockroachDB).

**Q: DynamoDB vs Cassandra?**
- DynamoDB: managed, pay-per-request, no ops. Limits: item size 400 KB, no ad-hoc queries.
- Cassandra: self-hosted (or ScyllaDB), multi-DC replication, tunable consistency.

**Q: When would you use MongoDB over Postgres?**
- Truly nested document access with sub-doc updates and no need for joins.
- Rapid schema evolution.
- Otherwise Postgres with JSONB columns is often enough.

**Q: What's the difference between horizontal and vertical scaling for DB?**
- Vertical: bigger box (more CPU, RAM, disk). Bounded by hardware limits.
- Horizontal: split data across nodes. Unbounded but adds complexity (sharding, cross-shard queries).

**Q: What are secondary indexes and their cost?**
- Extra data structure so queries by non-primary fields are fast.
- Every write updates the base row and every index → write amplification.
- In distributed DBs (DynamoDB, Cassandra), secondary indexes can be **local** (per partition, fast) or **global** (across all partitions, expensive).

**Q: How do you migrate from one DB to another with zero downtime?**
1. **Dual write**: app writes to old and new (with error tolerance on new).
2. **Backfill**: batch job copies historical data.
3. **Dual read**: read from new, compare to old, log diffs.
4. **Cut over**: read primarily from new; old becomes fallback.
5. **Decommission**: stop dual writes, remove old.

## Gotchas
- **"NoSQL means no schema"** — false. Schema is in application code; still exists.
- **JSONB in Postgres**: often good enough vs going to Mongo. Try first.
- **Cross-partition queries in Cassandra**: full scatter-gather → slow. Schema must fit the query.
- **DynamoDB hot partition**: > 1000 WCU on one partition throttles. Choose partition key wisely.
- **Elastic as source of truth**: never. Rebuild from real DB via CDC.
- **Choosing based on hype**: pick what your team can operate. A "worse" DB you understand beats a "better" one you can't debug at 3 AM.
