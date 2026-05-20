# Cassandra Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Architecture, data model, consistent hashing ring, replication, tunable consistency, CQL, when to use, pitfalls

---

## Why Cassandra

Apache Cassandra is a **wide-column, distributed, AP database** designed for massive write throughput, linear horizontal scaling, and no single point of failure. It is the go-to choice when:

- You need **millions of writes per second** (IoT telemetry, time-series, activity logs)
- Data is **naturally partitioned** by a high-cardinality key (user_id, device_id, sensor_id)
- Queries are known in advance and access data by partition key
- You can tolerate **eventual consistency** (or tune toward stronger consistency for some operations)
- Global multi-datacenter replication is needed

---

## Architecture

### Peer-to-Peer Ring (No Master)

Cassandra has no primary/secondary distinction — all nodes are peers:

```
         Node A (tokens 0–25)
        /                    \
Node D                        Node B
(tokens 76–100)         (tokens 26–50)
        \                    /
         Node C (tokens 51–75)
```

Every node can accept reads and writes. Clients can connect to any node (the **coordinator**). The coordinator routes requests to the correct replica nodes using the consistent hash ring. No SPOF.

### Data Distribution: Consistent Hashing + Virtual Nodes

Data is partitioned using a consistent hash ring (same principles as covered in Core Concepts 06). Cassandra uses **virtual nodes (vnodes)** — each physical node owns many tokens on the ring, ensuring even distribution and graceful rebalancing.

```
Partition key → murmur3 hash → token → vnode → physical node
```

### Replication

Each piece of data is replicated to N nodes (replication factor). The `NetworkTopologyStrategy` places replicas across racks/datacenters to maximize fault tolerance:

```
RF=3, 3 nodes: data lives on node A, B, C (clockwise on ring)
If node A fails → reads/writes still work from B and C
```

For multi-datacenter:
```
Keyspace: RF=3 per datacenter
  DC1: 3 replicas (US-East)
  DC2: 3 replicas (EU-West)
→ Survives entire datacenter failure
```

---

## Data Model: Design for Queries

Cassandra's fundamental rule: **design your data model around your queries, not your entities**.

### Tables

```cql
CREATE TABLE user_messages (
    user_id     UUID,
    sent_at     TIMESTAMP,
    message_id  UUID,
    content     TEXT,
    sender_id   UUID,
    PRIMARY KEY (user_id, sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC);
```

- **Partition key:** `user_id` — determines which nodes store this row. All rows with the same partition key are stored together on disk in sorted order.
- **Clustering columns:** `sent_at, message_id` — determine sort order within the partition. Enables efficient range queries within a partition.
- **Result:** "Get last 50 messages for user X" = single partition scan. Fast.

### Partition Key Selection

The partition key must:
1. **Be high cardinality** — millions of distinct values to spread load evenly
2. **Align with the primary access pattern** — the most common query filters by this key

**Good partition keys:**
- `user_id` for user-centric data
- `device_id` for IoT telemetry
- `(tenant_id, date)` for multi-tenant time-series

**Bad partition keys:**
- `status` (low cardinality → hot partitions)
- `country` (uneven distribution → US partition overwhelmed)
- `created_at` (time-based → all writes to same partition)

### Wide Rows and Data Locality

All rows in a partition are stored contiguously on disk. This makes range scans within a partition extremely efficient:

```cql
-- Efficient: single partition scan
SELECT * FROM user_messages
WHERE user_id = ? AND sent_at > ? AND sent_at < ?;

-- Inefficient: requires scatter-gather across all partitions
SELECT * FROM user_messages WHERE sent_at > ?;  -- full table scan! Avoid.
```

### Secondary Indexes and Materialized Views

When you need to query by a non-partition-key column, you have options:

**Cassandra Secondary Index (SAI):** Stored locally on each node. Queries must scatter-gather across all nodes. Acceptable for low-cardinality columns on moderate data volumes. Not for high-cardinality or high-throughput queries.

**Materialized Views:** Cassandra maintains a separate table with a different partition key, synced automatically. Useful but has performance implications and consistency caveats.

**Recommended approach:** Create a separate "query table" (denormalized) with the desired partition key, and dual-write to both tables:

```cql
-- Primary table: user_id partitioned
CREATE TABLE messages_by_user (user_id UUID, sent_at TIMESTAMP, ..., PRIMARY KEY (user_id, sent_at));

-- Query table: room_id partitioned (for "get messages in a chat room")
CREATE TABLE messages_by_room (room_id UUID, sent_at TIMESTAMP, ..., PRIMARY KEY (room_id, sent_at));
```

Write to both tables in the same Cassandra batch (logged batch for atomicity, though this has performance cost — use sparingly).

---

## Tunable Consistency

Cassandra's consistency is configurable **per operation**, not per table. This is its most powerful (and most misunderstood) feature.

```
N = replication factor
W = write quorum (nodes that must ack a write)
R = read quorum (nodes that must respond to a read)

Strong consistency when: W + R > N
```

| Consistency Level | Meaning | Use When |
|------------------|---------|---------|
| `ONE` | 1 node must respond | Max throughput, tolerate stale reads |
| `QUORUM` | N/2+1 nodes must respond | Strong consistency, balanced perf |
| `LOCAL_QUORUM` | Quorum within local DC only | Multi-DC, low cross-DC latency |
| `ALL` | All N nodes must respond | Strongest, least available |
| `LOCAL_ONE` | 1 node in local DC | Max throughput, local DC only |

**Example (RF=3):**
```
W=QUORUM (2), R=QUORUM (2): 2+2=4 > 3 → strongly consistent
W=ONE (1), R=ONE (1): 1+1=2 < 3 → eventual consistency

Multi-DC with LOCAL_QUORUM:
  Write acks from 2/3 nodes in local DC before returning
  Read from 2/3 nodes in local DC
  → Strong consistency within DC, eventual across DCs
```

**Interview tip:** Cassandra being "AP" doesn't mean you can't get strong consistency — you can with QUORUM. It means during a partition, you choose: sacrifice C or A. With QUORUM, you sacrifice some A (may block if too many nodes are down).

---

## Write Path

Cassandra is optimized for writes:

```
Write request arrives at coordinator
    │
    ▼
Coordinator writes to commit log (WAL, disk, sequential write)
AND writes to memtable (in-memory sorted structure)
    │
    ▼ (async, when memtable full)
SSTable flush to disk (immutable sorted file)
    │
    ▼ (background compaction)
Merge SSTables, remove tombstones, compact
```

- **Commit log:** Crash recovery. Sequential disk writes = very fast.
- **Memtable:** In-memory write buffer. All writes go here first.
- **SSTable:** Immutable on-disk sorted file. Never modified after flush.
- **Compaction:** Merges SSTables, reclaims space from deleted/expired data.

**Why Cassandra writes are fast:** Writes are always appends (commit log + memtable). No random disk I/O, no B-Tree rebalancing. This is the same LSM-Tree design as RocksDB.

## Read Path

Reads are more expensive than writes:

```
Read request arrives
    │
    ▼
Check memtable (in-memory)
    │ miss
    ▼
Check row cache (if enabled)
    │ miss
    ▼
Check Bloom filter per SSTable (skip SSTables that definitely don't have the key)
    │
    ▼
Read from SSTable(s) on disk
    │
    ▼
Merge results (latest wins across SSTables)
```

**Read repair:** If replicas disagree, Cassandra resolves conflict using the most recent timestamp and asynchronously repairs the inconsistent replica.

---

## Deletions and Tombstones

Cassandra doesn't delete data immediately. Instead, it writes a **tombstone** — a marker with a timestamp indicating the record is deleted. During compaction, tombstones and their target data are removed.

**Tombstone accumulation is a serious operational issue.** If data is deleted frequently and compaction can't keep up, tombstones pile up and slow reads dramatically. Mitigation:
- Set appropriate `gc_grace_seconds` (default 10 days — how long tombstones survive)
- Tune compaction strategy (TWCS for time-series, STCS for general, LCS for read-heavy)
- Use TTL instead of explicit deletes: `INSERT ... USING TTL 86400`

---

## Cassandra vs Alternatives

| Dimension | Cassandra | DynamoDB | MongoDB | PostgreSQL |
|-----------|-----------|----------|---------|------------|
| Write throughput | ✅ Extreme | ✅ High | ⚠️ Moderate | ⚠️ Moderate |
| Horizontal scale | ✅ Linear | ✅ Managed | ✅ | ⚠️ Complex |
| Consistency | Tunable | Tunable | Configurable | ACID |
| Query flexibility | ⚠️ Limited to pk | ⚠️ Limited | ✅ Flexible | ✅ Best |
| Ops complexity | High | Low (managed) | Medium | Medium |
| Multi-DC replication | ✅ Built-in | ✅ Global Tables | ⚠️ Atlas | ⚠️ Complex |
| Time-series | ✅ With TWCS | ✅ | ⚠️ | ⚠️ With BRIN |
| Best for | Massive write scale, IoT | AWS-native apps | Flexible schemas | General purpose |

---

## When to Propose Cassandra in an Interview

✅ **Good fit:**
- Messaging systems (WhatsApp-style inbox: partitioned by user)
- IoT/telemetry (millions of device writes/sec)
- Time-series (sensor data, metrics — use TWCS compaction)
- Activity feeds (partitioned by user_id, clustered by timestamp)
- Global multi-region data with low-latency local reads

❌ **Bad fit:**
- Complex multi-table joins
- Ad-hoc analytics (use a data warehouse instead)
- Strong ACID transactions across multiple partition keys
- When your team doesn't have Cassandra expertise (ops is hard)

---

## Interview Quick Reference

| Question | Answer |
|----------|--------|
| How does Cassandra scale? | Add nodes to the ring; vnodes redistribute automatically |
| How is data distributed? | Consistent hashing on partition key; vnodes per node |
| How do you get strong consistency? | W=QUORUM + R=QUORUM with RF≥3 |
| What's the #1 data modeling rule? | Design tables around queries, not entities |
| Why are reads more expensive than writes? | LSM-Tree: writes are appends; reads may merge multiple SSTables |
| What kills Cassandra performance? | Hot partitions, tombstone accumulation, scatter-gather queries |
| How do you handle time-series? | Partition by entity+time-bucket, cluster by timestamp, use TWCS |

---

*Previous: [04 - API Gateway](./04-api-gateway.md)*  
*Next: [06 - DynamoDB](./06-dynamodb.md)*
