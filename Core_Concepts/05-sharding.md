# Sharding for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** partitioning vs sharding, shard key selection, sharding strategies, cross-shard operations, rebalancing

---

## The Problem Sharding Solves

A single database has hard ceilings:
- **Storage:** even large cloud instances cap around 64–256 TiB
- **Write throughput:** a single Postgres instance handles ~10K–50K writes/sec under load
- **Read throughput:** read replicas help, but they share the same write bottleneck

When vertical scaling (bigger machine) runs out, you have one option: **split data across multiple machines**.

---

## Partitioning vs Sharding

These terms are often used interchangeably. In practice:

| | Partitioning | Sharding |
|--|-------------|---------|
| Data location | Single machine | Multiple machines |
| Goal | Organizational efficiency, query pruning | Horizontal scale |
| Example | PostgreSQL table partitioned by month | Users split across 8 DB instances |

Both involve dividing a dataset into smaller pieces. Sharding just means those pieces live on different servers.

---

## Step 1: Partitioning Within a Single Node

Before you shard across machines, partition within one. Most production databases support table partitioning natively.

**Horizontal partitioning** — split by rows:

```sql
-- PostgreSQL declarative partitioning by time range
CREATE TABLE orders (
    id          UUID,
    user_id     UUID,
    created_at  TIMESTAMPTZ,
    total_cents INT
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

Query pruning: `WHERE created_at >= '2025-01-01'` scans only `orders_2025`.

**Vertical partitioning** — split by columns. Move rarely-accessed large columns to a separate table:

```sql
-- Core table (hot read path)
CREATE TABLE users (id UUID PRIMARY KEY, email TEXT, name TEXT);

-- Separate table for large/rarely-accessed columns
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY REFERENCES users(id),
    bio     TEXT,
    avatar  BYTEA,
    settings JSONB
);
```

---

## Step 2: Sharding Across Machines

When one machine isn't enough, distribute shards across multiple independent database instances.

```
                    ┌─────────────────┐
   Client ─────────►│  Shard Router   │
                    │ (app / proxy)   │
                    └────────┬────────┘
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         ┌────────┐    ┌────────┐    ┌────────┐
         │ Shard 0│    │ Shard 1│    │ Shard 2│
         │ DB     │    │ DB     │    │ DB     │
         └────────┘    └────────┘    └────────┘
         users 0–1M   users 1M–2M  users 2M–3M
```

Each shard is a complete, independent database. The router decides which shard to query for a given request.

---

## Choosing a Shard Key

The shard key determines which shard holds each record. Choosing well is the most important sharding decision.

### Properties of a Good Shard Key

| Property | Why It Matters |
|----------|---------------|
| **High cardinality** | More unique values → finer-grained distribution |
| **Even distribution** | Prevents hot shards with disproportionate data |
| **Query alignment** | Most queries should hit exactly one shard |
| **Immutable** | Changing a key means moving data across shards |

### Examples

**Good shard keys:**

| Key | Why Good |
|-----|---------|
| `user_id` | High cardinality (millions of users), most queries scoped to one user, even distribution |
| `order_id` (UUID) | High cardinality, orders distribute evenly, queries by order ID = single shard |
| `tenant_id` (multi-tenant SaaS) | Each tenant's data co-located, cross-tenant queries rare |

**Bad shard keys:**

| Key | Why Bad |
|-----|--------|
| `country` | Low cardinality; US shard gets 50%+ of traffic |
| `is_premium` | Only 2 values; unbalanced distribution |
| `created_at` | All new writes go to the latest shard (write hotspot) |
| `status` | Low cardinality (active/inactive/deleted) |

---

## Sharding Strategies

### 1. Hash-Based Sharding (Default)

Apply a hash function to the shard key and use modulo to assign a shard:

```
shard_id = hash(user_id) % num_shards

user_id=42   → hash(42)   % 4 = 2  → Shard 2
user_id=99   → hash(99)   % 4 = 3  → Shard 3
user_id=1337 → hash(1337) % 4 = 1  → Shard 1
```

**Pros:** even distribution; simple routing  
**Cons:** adding/removing shards requires remapping almost all keys (use **consistent hashing** to solve this — see [06 - Consistent Hashing](./06-consistent-hashing.md))

**Range queries are expensive** — "get all users with IDs 1000–2000" may span every shard.

### 2. Range-Based Sharding

Assign contiguous ranges of the key to each shard:

```
Shard 0: user_id 0 – 999,999
Shard 1: user_id 1,000,000 – 1,999,999
Shard 2: user_id 2,000,000 – 2,999,999
```

**Pros:** efficient range scans; adding a shard just adds a new range  
**Cons:** can create hot shards if data or access isn't uniformly distributed (new users all go to the latest shard)

**Best for:** multi-tenant systems where each tenant's workload is self-contained.

### 3. Directory-Based Sharding

A separate lookup service maintains a mapping from key to shard:

```
Lookup table:
  user_id 42   → Shard 2
  user_id 99   → Shard 0  (manually moved for load balancing)
  user_id 1337 → Shard 1
```

**Pros:** maximum flexibility; can rebalance individual records without rehashing everything  
**Cons:** lookup service becomes a critical dependency; every request adds a network round-trip; the lookup table must be highly available and fast

**Best for:** systems that need per-record placement control (very large tenants on dedicated shards).

### Comparison

| Strategy | Distribution | Range Queries | Rebalancing | Complexity |
|----------|-------------|---------------|-------------|------------|
| Hash | Excellent | Poor (scatter-gather) | Hard (consistent hashing helps) | Low |
| Range | Can be uneven | Excellent | Easy (add ranges) | Low |
| Directory | Manual control | Poor (unless designed for it) | Easy (update mapping) | High |

---

## Cross-Shard Operations

Sharding breaks operations that span multiple shards. Understand the costs.

### Scatter-Gather Queries

A query that can't be routed to a single shard must be sent to all shards and the results merged:

```
"Find all users who signed up in the last 7 days"
→ shard by user_id doesn't help (no user_id filter)
→ query all N shards in parallel
→ merge and sort results
→ latency = slowest shard response
```

**Solutions:**
- Accept scatter-gather for rare queries (admin/analytics)
- Maintain a secondary index / search service (Elasticsearch) for non-key queries
- Dual-write to a separate reporting database (read replica, data warehouse)

### Cross-Shard Transactions

ACID transactions spanning multiple shards are complex and expensive:

- **2-Phase Commit (2PC):** Coordinator asks all shards to prepare, then commits. High latency; coordinator is a SPOF.
- **Sagas:** Break the transaction into a sequence of local transactions with compensating actions on failure.
- **Avoid where possible:** Design the shard key so related records live on the same shard.

**Interview tip:** When asked about cross-shard transactions, acknowledge the complexity and propose a design that avoids them (co-locate related entities on the same shard).

### Joins Across Shards

SQL joins across shards require fetching data from multiple nodes and joining in the application:

```python
# Expensive: orders and users on different shards
users = shard_a.query("SELECT * FROM users WHERE id IN (?)", user_ids)
orders = shard_b.query("SELECT * FROM orders WHERE user_id IN (?)", user_ids)
result = join_in_memory(users, orders)
```

**Solution:** shard by a key that keeps related data together. If orders always need the user record, shard both users and orders by `user_id`.

---

## Rebalancing Shards

### The Problem

As data grows, shards become uneven. A popular tenant may overwhelm one shard while others are idle.

### Strategies

1. **Add new shards:** With consistent hashing, adding a shard only moves 1/N of the data. With simple modulo, you need to move almost everything.

2. **Split a hot shard:** Take a large shard and split its key range into two shards. Requires a data migration.

3. **Move individual records (directory-based):** If using directory sharding, move specific hot records to dedicated shards.

### Live Migration Pattern

Moving data without downtime:

```
Phase 1: Dual-write to old and new shard
Phase 2: Backfill historical data from old to new shard
Phase 3: Verify data consistency (checksum comparison)
Phase 4: Switch reads to new shard
Phase 5: Stop writing to old shard
Phase 6: Decommission old shard
```

**Tooling for zero-downtime migrations:**
- **gh-ost** (GitHub) — shadow table + CDC-based MySQL schema changes without locking
- **pglogical / logical replication** — stream changes to a new Postgres shard while backfilling
- **Custom dual-write proxies** — application writes to both old and new; flip reads when new is caught up

### Logical vs Physical Shards

A powerful pattern used by Slack, Discord, and others: **over-shard at creation, but under-provision machines**.

```
Create 1024 logical shards → map to 4 physical DB nodes (256 shards each)

Later, when you need to scale:
  Remap logical shards 512–1023 → 4 new physical DB nodes
  Only ~50% of data moves (physical node's shards migrate to new nodes)
  Application shard routing logic stays the same
```

**Why it's better than physical-only sharding:**
- Adding a physical node is just a config change (remap logical→physical)
- No application code changes — clients route by logical shard ID
- Rebalancing is O(moved_shards) not O(total_data)
- Start with 4 nodes, eventually scale to 1024 nodes without changing the sharding scheme

This is the right answer when an interviewer asks "how would you shard this in a way that's easy to scale later?"

---

## When to Shard

Sharding adds significant operational complexity. Delay it as long as possible.

**First, exhaust these options:**
1. Add read replicas (offload reads)
2. Add caching (Redis)
3. Optimize slow queries and add indexes
4. Upgrade to a larger instance (vertical scaling)
5. Archive old data (delete or cold storage)

**Shard when:**
- Single-node write throughput is insufficient even after optimization
- Data volume exceeds what can fit on one machine
- You're hitting I/O or memory limits even with the largest available instance

**Practical thresholds (rough guidelines):**
- > 5TB of data and growing
- > 50K writes/sec sustained
- > 100K QPS with read replicas saturated

---

## Sharding in AI Agent Systems

Agent systems have unique sharding considerations:

### Sharding Agent State

Shard agent runs by `user_id` — most queries are "get all runs for user X":

```
shard_id = hash(user_id) % num_shards
→ agent_runs for user X all live on shard hash(X)
→ "show me my recent runs" = single-shard query
```

### Sharding Vector Data

For long-term memory with vector embeddings:
- Shard by `user_id` — each user's memories co-located
- Vector search (ANN query) within a shard using indexes like HNSW or IVFFlat
- For global search across all users, use a dedicated vector database (Pinecone, Weaviate) that handles its own sharding

---

## Summary Diagram

```
Is your database struggling?
        │
        ▼
Add read replicas ──► Still slow for reads? ──► Add caching (Redis)
        │
Still struggling under write load or storage?
        │
        ▼
    Partition first (within one node)
    - Range partition by time for time-series
    - Hash partition for even distribution
        │
        ▼
    Shard across nodes (when one node not enough)
    - Hash shard by natural access key (user_id, tenant_id)
    - Use consistent hashing to minimize rebalancing cost
    - Accept scatter-gather or add secondary index for non-key queries
```

---

*Previous: [04 - Caching](./04-caching.md)*  
*Next: [06 - Consistent Hashing](./06-consistent-hashing.md)*
