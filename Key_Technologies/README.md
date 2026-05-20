# Key Technologies — System Design

This section covers all technologies from Hello Interview's **Key Technologies** and **Advanced Topics** sections, unified in one folder. Each article goes deeper than the source material, with correct trade-off analysis, security considerations, and AI/agent-specific patterns.

---

## Articles

### Key Technologies (Hello Interview originals)

| # | Technology | Role in System Design | Key Concepts |
|---|-----------|----------------------|-------------|
| 01 | [Redis](./01-redis.md) | Cache, lock, pub/sub, leaderboard, rate limiter | Data structures, Sentinel vs Cluster, hot key, Redlock, Streams |
| 02 | [Elasticsearch](./02-elasticsearch.md) | Full-text search, analytics, log search | Inverted index, mappings, bool query, aggregations, sync patterns |
| 03 | [Kafka](./03-kafka.md) | Event streaming, decoupling, CDC | Partitions, consumer groups, delivery guarantees, DLQ, Flink integration |
| 04 | [API Gateway](./04-api-gateway.md) | Ingress, auth, rate limiting, routing | Request lifecycle, canary, circuit breaker, BFF, AI gateway |
| 05 | [Cassandra](./05-cassandra.md) | Massive write throughput, time-series, IoT | Wide-column, partition key, tunable consistency, quorum, compaction |
| 06 | [DynamoDB](./06-dynamodb.md) | Serverless key-value, AWS-native | Single-table design, GSI/LSI, streams, DAX, on-demand vs provisioned |
| 07 | [PostgreSQL](./07-postgresql.md) | Default OLTP, the right starting point | MVCC, PgBouncer, replication, pgvector, Citus, TimescaleDB |
| 08 | [Apache Flink](./08-flink.md) | Stateful stream processing | Windows, event time, watermarks, exactly-once, vs Kafka Streams |
| 09 | [ZooKeeper](./09-zookeeper.md) | Distributed coordination | ZAB, znodes, leader election, distributed lock, vs etcd |

### Advanced Topics (Hello Interview originals, extended)

| # | Technology | Role in System Design | Key Concepts |
|---|-----------|----------------------|-------------|
| 10 | [Time Series Databases](./10-time-series-databases.md) | Metrics, IoT, monitoring | Columnar storage, compression, InfluxDB, TimescaleDB, ClickHouse, Prometheus |
| 11 | [Data Structures for Big Data](./11-data-structures-big-data.md) | Probabilistic algorithms | Bloom filter, Count-Min Sketch, HyperLogLog, MinHash/LSH, Skip list, Merkle tree |
| 12 | [Vector Databases](./12-vector-databases.md) | Semantic search, RAG | HNSW, IVF-PQ, Pinecone/Weaviate/Qdrant/pgvector, hybrid search, reranking |

---

## How to Read These Articles

Each article follows the same structure:
1. **Why / When** — decision criteria for proposing this technology
2. **How it works** — internal mechanics (what interviewers probe)
3. **Key patterns** — concrete code/config examples
4. **Pitfalls** — what goes wrong and how to fix it
5. **vs Alternatives** — comparison table for trade-off conversations
6. **Security** — security considerations specific to this technology
7. **AI/Agent use cases** — modern applications
8. **Interview quick reference** — tables for fast recall

---

## Technology Selection Guide

### "Which database should I use?"

```
Do you need full-text search?
  → Elasticsearch (or Postgres FTS for simple cases)

Do you need semantic / vector search?
  → pgvector (Postgres, < 5M vectors) or Qdrant/Weaviate/Pinecone (larger scale)

Do you need sub-ms cache or pub/sub?
  → Redis

Do you need millions of writes/second?
  → Cassandra (multi-cloud) or DynamoDB (AWS)

Do you need ACID transactions, flexible queries, joins?
  → PostgreSQL (default — start here)

Do you need stream processing / event CDC?
  → Kafka (message bus) + Flink (stateful processing)

Do you need infrastructure metrics?
  → Prometheus + Grafana

Do you need time-series at extreme scale?
  → InfluxDB or ClickHouse
```

### "Which of these do I actually need to know for my interview?"

**Must know (appear in almost every design):**
- Redis (01)
- Kafka (03)
- PostgreSQL (07)

**Should know (appear in most staff-level designs):**
- Elasticsearch (02)
- API Gateway (04)
- Cassandra or DynamoDB (05 or 06, depending on cloud)

**Good to know (differentiate at staff+ level):**
- Flink (08)
- ZooKeeper (09)
- Vector Databases (12)
- Data Structures for Big Data (11)

**Know basics for AI-focused roles:**
- Vector Databases (12) — essential
- Time Series (10) — for monitoring/observability

---

## Cross-Reference: Technology → Problem

| Interview Problem | Technologies to Know |
|------------------|---------------------|
| Design Twitter/Facebook feed | Redis (cache, pub/sub), Kafka (event pipeline), PostgreSQL |
| Design WhatsApp | Redis (pub/sub), Kafka, Cassandra (message storage) |
| Design Uber | Redis (geo), Kafka, PostgreSQL, API Gateway |
| Design YouTube | Kafka (events), Elasticsearch (search), PostgreSQL, Redis |
| Design ad click aggregator | Kafka, Flink, ClickHouse/Cassandra |
| Design metrics monitoring | Prometheus, InfluxDB/TimescaleDB, Kafka |
| Design search (Yelp, Airbnb) | Elasticsearch, Redis, PostgreSQL |
| Design distributed cache | Redis (Cluster mode), consistent hashing |
| Design leader election | ZooKeeper or etcd |
| Design RAG / AI assistant | Vector DB (pgvector/Qdrant), Kafka, PostgreSQL |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| v1.0 | 2025-05 | All 12 articles created (9 Key Technologies + 3 Advanced Topics) |
