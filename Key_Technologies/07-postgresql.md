# PostgreSQL Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Architecture, ACID/MVCC, indexing recap, replication, connection pooling, extensions, scaling patterns, when Postgres is enough

---

## Why PostgreSQL Is the Default

PostgreSQL is the correct starting point for virtually every system design. It is the most feature-complete, battle-tested, open-source relational database. At staff level, knowing when to stay on Postgres (and scale it correctly) versus moving to a specialized database is a critical judgment call.

**The first question is not "which database?" — it's "can Postgres handle this?"** The answer is often yes, for much longer than engineers expect.

---

## Architecture

### Process Model

Postgres uses a **multi-process** model (not multi-threaded):
- **Postmaster:** Parent process, accepts connections, spawns worker processes
- **Backend process:** One per client connection (default max: 100–200 connections)
- **Background workers:** Autovacuum, WAL writer, checkpoint writer, bgworker

The per-connection process model is why Postgres doesn't handle thousands of raw connections well — you need a connection pooler.

### Storage Engine and MVCC

Postgres uses **MVCC (Multi-Version Concurrency Control)** to handle concurrent reads and writes without locking:

```
Transaction A reads row at timestamp T1:
  → Sees the version of the row that was committed before T1
  → Not blocked by Transaction B that is modifying the row

Transaction B writes row at timestamp T2:
  → Creates a new version of the row with T2
  → Old version (T1) still visible to Transaction A
  → After A commits, old version is dead (to be vacuumed)
```

**Result:** Readers never block writers, writers never block readers. Excellent concurrency.

**Consequence:** Dead tuple accumulation. Old row versions pile up and must be reclaimed by **autovacuum**. A table with high update/delete rate that isn't vacuumed will bloat and slow down. Monitor `pg_stat_user_tables.n_dead_tup`.

### WAL (Write-Ahead Log)

Every change is written to the WAL (sequential disk write) before modifying the data files. This enables:
- **Crash recovery:** Replay WAL after crash to reach a consistent state
- **Streaming replication:** Send WAL records to replicas
- **Point-in-time recovery (PITR):** Restore to any moment in history
- **Logical replication:** Stream individual row changes (used by Debezium, logical decoding)

---

## ACID Guarantees

| Property | How Postgres Implements It |
|---------|--------------------------|
| **Atomicity** | Transaction commits atomically to WAL; on failure, rolled back |
| **Consistency** | Constraints (FK, CHECK, UNIQUE, NOT NULL) enforced at commit |
| **Isolation** | MVCC + configurable isolation levels |
| **Durability** | WAL flushed to disk before commit acknowledged (`fsync=on`) |

### Isolation Levels

```
READ UNCOMMITTED  (not actually different from READ COMMITTED in Postgres)
READ COMMITTED    (default) — sees committed data at statement start
REPEATABLE READ   — sees snapshot at transaction start; prevents non-repeatable reads
SERIALIZABLE      — full serializability; Postgres uses SSI (Serializable Snapshot Isolation)
```

**Interview use:** Mention SERIALIZABLE for financial transactions that absolutely cannot have anomalies. It's available and correct in Postgres, at a latency cost.

---

## Connection Pooling: PgBouncer

Postgres spawns a process per connection. At 1000 raw connections, Postgres overhead is significant. Solution: a **connection pooler** (PgBouncer) that multiplexes many client connections onto a small number of server connections.

```
1000 app connections → PgBouncer → 50 Postgres connections
```

PgBouncer modes:
- **Session pooling:** One server connection per client for duration of session. Limited benefit.
- **Transaction pooling:** Server connection returned to pool after each transaction. **Recommended** — 10–50× connection reduction.
- **Statement pooling:** After each statement. Breaks multi-statement transactions. Rarely used.

**Connection pool sizing formula:**
```
Postgres connections = (num_cores × 2) + effective_spindle_count
Typical: 8-core server → 16-20 Postgres connections is optimal
```

More connections than this degrades throughput (context switching, lock contention). With PgBouncer, serve 10,000 app connections with 20 Postgres connections.

---

## Replication

### Physical Replication (Streaming)

WAL is streamed from primary to replicas in real time. Replicas are byte-for-byte identical copies:

```
Primary → WAL stream → Replica 1 (read replica)
                    → Replica 2 (standby for failover)
```

- **Synchronous replication:** Primary waits for replica WAL ack before committing. No data loss, higher write latency.
- **Asynchronous replication:** Primary doesn't wait. Lower write latency, up to seconds of data loss on failover.
- **Lag monitoring:** `pg_stat_replication.replay_lag` — keep below 1s for hot standbys.

### Logical Replication

Streams individual row-level changes (INSERT/UPDATE/DELETE) in a table-selectable way. Enables:
- Replicating to a different major Postgres version
- Replicating specific tables only
- Replicating to non-Postgres targets (Debezium → Kafka)
- Zero-downtime major version upgrades

### Read Replicas for Scaling Reads

```
Write: Primary (master)
Read:  Replica 1, Replica 2, Replica 3 (behind load balancer)
```

Application routes reads to replicas, writes to primary. Connection-level or statement-level routing via PgBouncer or PgPool-II.

**Lag caveat:** Replicas are eventually consistent (typically < 100ms behind). For "read your own write" consistency, route the write + immediate read to the primary.

---

## Key Extensions

Postgres's extension ecosystem is its killer feature:

| Extension | Purpose |
|-----------|---------|
| **PostGIS** | Geospatial queries (geo_point, proximity search, polygon intersection) |
| **pgvector** | Vector similarity search (cosine, L2 distance) with HNSW/IVFFlat indexes |
| **pg_partman** | Automated table partitioning management |
| **TimescaleDB** | Time-series hypertables, compression, continuous aggregates |
| **Citus** | Horizontal sharding (distributed Postgres) |
| **pg_stat_statements** | Query performance tracking (essential for tuning) |
| **pgcrypto** | Column-level encryption functions |
| **uuid-ossp / gen_random_uuid()** | UUID generation |
| **HStore / JSONB** | Semi-structured data within SQL |

**pgvector** is particularly important for AI workloads — store embeddings and do ANN search directly in Postgres without a separate vector database.

---

## Scaling Patterns

### Vertical Scaling First

Postgres on a large instance (128+ vCPU, 1TB RAM) handles enormous workloads. Before sharding:
- Optimize slow queries (`EXPLAIN ANALYZE`, indexes)
- Add read replicas
- Add connection pooling (PgBouncer)
- Enable table partitioning
- Archive old data to cold storage

### Table Partitioning (Single Node)

```sql
-- Range partition by month
CREATE TABLE events (
    id UUID, user_id UUID, type TEXT, created_at TIMESTAMPTZ
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_01 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE events_2025_02 PARTITION OF events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
```

Benefits: partition pruning (queries only scan relevant partitions), vacuum per partition, drop old partitions instantly.

### Citus (Horizontal Sharding)

Citus extends Postgres with distributed sharding. Choose a **distribution column** (analogous to shard key):

```sql
-- Create distributed table
SELECT create_distributed_table('orders', 'user_id');

-- Co-located tables (same distribution column → same shard)
SELECT create_distributed_table('order_items', 'user_id');

-- Joins between co-located tables are local (fast)
SELECT o.*, i.* FROM orders o JOIN order_items i ON o.id = i.order_id
WHERE o.user_id = 42;  -- all on same shard
```

Citus maintains a metadata table on the coordinator node mapping logical shards to worker nodes. Single SQL interface, distributed execution.

### When to Leave Postgres

Move to a specialized database when:
- **Cassandra/DynamoDB:** Need 500K+ writes/sec, or data doesn't fit the relational model
- **Elasticsearch:** Need full-text search, fuzzy matching, or complex aggregations at scale
- **ClickHouse/Druid:** Need OLAP analytics over billions of rows with sub-second query times
- **Redis:** Need sub-millisecond latency for hot data

---

## JSONB: The Semi-Structured Escape Hatch

When the schema is partially unknown, JSONB allows schema-on-read:

```sql
CREATE TABLE products (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    metadata JSONB  -- arbitrary attributes per product category
);

-- Index a specific JSON key
CREATE INDEX ON products((metadata->>'brand'));

-- GIN index for containment queries
CREATE INDEX ON products USING GIN(metadata);

-- Query by JSON field
SELECT * FROM products WHERE metadata @> '{"brand": "Apple", "color": "silver"}';
SELECT * FROM products WHERE (metadata->>'price')::float < 500;
```

JSONB is binary-stored and indexed — not a document database, but surprisingly capable for mixed-schema use cases.

---

## Performance Tuning Essentials

```sql
-- Find slow queries (requires pg_stat_statements)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Check index usage
SELECT indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0;  -- unused indexes — consider dropping

-- Check for bloat
SELECT tablename, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric/nullif(n_live_tup,0)*100, 2) AS dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Analyze table after bulk load
ANALYZE orders;

-- Rebuild index without locking
REINDEX INDEX CONCURRENTLY idx_orders_user_date;
```

Key `postgresql.conf` parameters to know:
```
shared_buffers = 25% of RAM          # Buffer pool
effective_cache_size = 75% of RAM    # Hint to query planner
work_mem = 64MB                      # Per sort/hash operation
maintenance_work_mem = 1GB           # Vacuum, CREATE INDEX
max_connections = 100                # Use PgBouncer, keep this low
checkpoint_completion_target = 0.9   # Spread checkpoint I/O
```

---

## Postgres for AI Workloads

### pgvector for RAG

```sql
-- Store embeddings alongside content
CREATE TABLE documents (
    id UUID PRIMARY KEY,
    content TEXT,
    embedding vector(1536)  -- OpenAI text-embedding-3-small
);

-- HNSW index for ANN search
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Semantic search query
SELECT content, embedding <=> $1 AS distance
FROM documents
ORDER BY embedding <=> $1  -- cosine distance
LIMIT 5;

-- Hybrid: combine BM25 (full-text) + vector similarity
SELECT content,
       ts_rank(to_tsvector('english', content), to_tsquery('machine & learning')) AS text_score,
       1 - (embedding <=> $1) AS vector_score
FROM documents
WHERE to_tsvector('english', content) @@ to_tsquery('machine & learning')
ORDER BY (text_score + vector_score) DESC
LIMIT 10;
```

---

## Interview Quick Reference

| Question | Answer |
|----------|--------|
| When to use Postgres? | Default choice for most OLTP applications |
| How to scale reads? | Read replicas + connection pooling |
| How to scale writes? | Vertical scale first; Citus for horizontal sharding |
| How does Postgres handle concurrency? | MVCC — readers don't block writers |
| Why need connection pooler? | Process-per-connection model; PgBouncer multiplexes |
| How to do full-text search in Postgres? | GIN index + tsvector/tsquery — good for simple search |
| How to do vector search? | pgvector extension with HNSW index |
| When to move away from Postgres? | Extreme write throughput (Cassandra), search (ES), analytics (ClickHouse) |

---

*Previous: [06 - DynamoDB](./06-dynamodb.md)*  
*Next: [08 - Apache Flink](./08-flink.md)*
