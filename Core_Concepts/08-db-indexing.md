# Database Indexing for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** B-Tree and LSM-Tree internals, index types, composite indexes, covering indexes, indexing strategy, query planning, trade-offs

---

## Why Indexing Matters

A `SELECT * FROM users WHERE email = 'alice@example.com'` on a table with 100M rows:

- **Without index:** full table scan — reads every row — 10–60 seconds
- **With index on `email`:** B-Tree traversal — ~5–6 page reads (log₃₀₀(100M) ≈ 5 levels) — < 1 ms

Indexes are the single biggest performance lever in relational databases. Poor indexing is the root cause of the majority of database performance problems in production.

---

## How Indexes Work: The B-Tree

PostgreSQL, MySQL, and most SQL databases use a **B-Tree (Balanced Tree)** as the default index structure.

### Structure

```
                    [30 | 60]
                   /    |    \
          [10|20]    [40|50]    [70|80]
          /  |  \    /  |  \    /  |  \
        [8] [15] [25] [35] [45] [55] [65] [75] [90]
        leaf leaf leaf leaf leaf leaf leaf leaf leaf
```

Each internal node contains sorted keys. Each leaf node contains the key plus either:
- **Clustered index:** the actual row data (InnoDB primary key, SQL Server clustered index)
- **Non-clustered index:** a pointer (row ID or primary key value) to the actual row

### Properties

- Tree height is logarithmic: 1M rows → ~3–4 levels deep
- Each node is typically one disk page (4–16 KB)
- A point lookup on 1M rows reads ~3–4 disk pages
- Range scans traverse the leaf level (leaves are linked)

### B-Tree vs B+Tree

Most databases use B+Tree (all data in leaves; internal nodes only contain keys for routing). This means:
- Range scans are efficient — just traverse linked leaf nodes
- Internal nodes can hold more keys (no row data) → shallower tree

---

## How Indexes Work: The LSM-Tree

**LSM-Tree (Log-Structured Merge-Tree)** powers write-optimized storage engines: Cassandra, RocksDB, LevelDB, HBase, Scylla.

### The Problem with B-Trees for Writes

B-Tree writes can cause **write amplification**:
1. Read the existing page into memory
2. Modify the page
3. Write it back to disk
4. Write to WAL (write-ahead log) for durability
5. Potentially rebalance the tree (node splits)

At high write throughput this becomes a bottleneck.

### LSM-Tree Structure

```
Writes ──► MemTable (in-memory, sorted)
                │  when full (e.g., 64MB)
                ▼
           SSTable L0 (immutable, sorted)
                │  compaction
                ▼
           SSTable L1 (larger, sorted)
                │  compaction
                ▼
           SSTable L2 (larger still)
```

1. **MemTable:** Incoming writes go to an in-memory sorted structure (usually a skip list or red-black tree). Very fast.
2. **SSTable:** When the MemTable fills, it's flushed to disk as an immutable Sorted String Table (SSTable).
3. **Compaction:** Background process merges SSTables, removing overwritten/deleted keys.

### Trade-offs vs B-Tree

| | B-Tree | LSM-Tree |
|--|--------|---------|
| Write throughput | Moderate | Very high |
| Read throughput | High (single seek) | Lower (may check multiple levels) |
| Read amplification | Low | High (bloom filters help) |
| Write amplification | High | Lower |
| Compaction overhead | None | Background CPU/IO |
| Best for | Balanced read/write, point queries | Write-heavy, time-series, logs |

---

## Index Types

### 1. Primary Key Index (Clustered)

Every table should have a primary key. In InnoDB (MySQL), the primary key is the clustered index — row data is stored in PK order on disk. Reads by PK require no extra lookup.

In PostgreSQL, the heap is unordered; the PK index stores a pointer (TID) to the heap page. A separate `CLUSTER` command can physically reorder the heap, but it's not maintained automatically.

### 2. Secondary Index (Non-Clustered)

Index on any non-PK column. The index stores the indexed value + a pointer back to the row.

```sql
-- Index for lookups by email
CREATE INDEX idx_users_email ON users(email);

-- Now this is fast:
SELECT * FROM users WHERE email = 'alice@example.com';
```

**Double lookup:** for non-clustered indexes, the database reads the index to get the row pointer, then fetches the actual row. Two disk reads per row.

### 3. Composite Index

Index on multiple columns. **Column order matters critically.**

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
```

A composite index on `(user_id, created_at)` can efficiently answer:
- `WHERE user_id = 42` ✅ (leftmost prefix)
- `WHERE user_id = 42 AND created_at > '2025-01-01'` ✅
- `WHERE user_id = 42 ORDER BY created_at` ✅

It **cannot** efficiently answer:
- `WHERE created_at > '2025-01-01'` ❌ (missing leftmost column)
- `WHERE user_id > 42 AND created_at > '2025-01-01'` ⚠️ (only first column used for range)

**Rule of thumb:** Equality conditions first, then range conditions. The index is useful up to the first range predicate.

### 4. Covering Index

An index that includes all columns needed to satisfy a query, so the database never needs to fetch the actual row.

```sql
-- Query: get status and total for a user's orders
SELECT status, total_cents FROM orders WHERE user_id = 42;

-- Covering index:
CREATE INDEX idx_orders_user_covering ON orders(user_id) INCLUDE (status, total_cents);
-- PostgreSQL syntax: INCLUDE adds non-key columns to the leaf

-- MySQL equivalent:
CREATE INDEX idx_orders_user_covering ON orders(user_id, status, total_cents);
```

With a covering index, the query is satisfied entirely from the index. No heap fetch needed. Massively faster for frequently read columns.

### 5. Partial Index

Index only a subset of rows. Smaller, faster to scan, cheaper to maintain.

```sql
-- Index only active users (not deleted/banned accounts)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';

-- Index only unfulfilled orders
CREATE INDEX idx_pending_orders ON orders(user_id, created_at) WHERE status = 'pending';
```

Partial indexes are underused and very powerful when a query always includes a specific filter.

### 6. Unique Index

Enforces uniqueness and is used by the optimizer like a regular index.

```sql
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
-- This also creates a constraint:
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);
```

### 7. Full-Text Index

Inverted index for text search. Allows efficient keyword lookup across documents.

```sql
-- PostgreSQL GIN full-text index
CREATE INDEX idx_posts_fts ON posts USING GIN(to_tsvector('english', content));

-- Query:
SELECT * FROM posts WHERE to_tsvector('english', content) @@ to_tsquery('database & indexing');
```

Full-text indexing in Postgres is powerful but limited compared to Elasticsearch for complex search (no fuzzy matching, facets, or relevance scoring). Use Postgres FTS for simple keyword search; Elasticsearch for full-featured search.

### 8. Spatial / GIS Index (R-Tree / GiST)

Index for geographic queries.

```sql
CREATE INDEX idx_venues_location ON venues USING GIST(location);

-- Nearest-neighbor query:
SELECT name FROM venues ORDER BY location <-> ST_MakePoint(-122.4, 37.7) LIMIT 10;
```

PostGIS + GiST index makes geo queries fast. Needed for any ride-share, delivery, or location-based system.

### 9. BRIN Index (Block Range Index)

**BRIN** is specifically designed for very large append-only tables where values are naturally correlated with physical storage order — time-series data, log tables, IoT sensor readings.

```sql
-- BRIN on a time-series events table (100M+ rows)
CREATE INDEX ON events USING BRIN(created_at);
```

**How it works:** Instead of indexing every value, BRIN stores the min/max value for each range of disk blocks (e.g., every 128 pages). A query `WHERE created_at > '2025-01-01'` eliminates all block ranges whose max is before that date.

| Property | B-Tree | BRIN |
|----------|--------|------|
| Index size | ~10–20% of table | < 0.1% of table |
| Best for | Arbitrary point/range queries | Naturally ordered append-only data |
| False positives | None | Possible (checks qualifying block ranges) |
| Write overhead | High (maintain sorted structure) | Very low (update min/max only) |

**When to use:** Any large time-series table where `created_at` or `event_time` is always increasing. Dramatically cheaper than B-Tree with similar range-scan performance for time-bounded queries.

---

## Index Selectivity and Query Planner Statistics

The query planner decides *whether* to use your index based on **selectivity** — the fraction of rows an index scan would return.

- High selectivity (UUID lookup, email): index scan is much faster than seq scan → planner uses index
- Low selectivity (boolean, status with 3 values): index + heap fetches can be *slower* than seq scan → planner skips index

**Statistics:** Postgres collects column statistics via `ANALYZE` (run by autovacuum automatically). If statistics are stale, the planner makes bad estimates.

```sql
-- Check statistics age
SELECT schemaname, tablename, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
WHERE tablename = 'orders';

-- Force fresh statistics after bulk load
ANALYZE orders;

-- See what the planner knows about a column
SELECT * FROM pg_stats WHERE tablename = 'orders' AND attname = 'status';
-- n_distinct, most_common_vals, histogram_bounds
```

**Common planner pitfalls:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| Seq scan on indexed column | Stale statistics or low selectivity | `ANALYZE`; check n_distinct |
| Seq scan after bulk insert | New rows not yet analyzed | `ANALYZE` manually post-bulk-load |
| Wrong join strategy | Row count estimate off by 100× | `ANALYZE`; consider `pg_hint_plan` |
| Index scan slower than expected | Index bloat from heavy deletes | `REINDEX CONCURRENTLY` |

The **query planner** decides how to execute a query. You can inspect it:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42 AND created_at > '2025-01-01';
```

Key plan nodes:

| Node | Meaning |
|------|---------|
| `Seq Scan` | Full table scan — no index used |
| `Index Scan` | Index used; heap fetched for each row |
| `Index Only Scan` | Covering index — no heap fetch |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | Multiple conditions merged via bitmap |
| `Nested Loop` | Join: outer row → look up in inner (good for small outer sets) |
| `Hash Join` | Build hash table of one side; probe with other (good for large sets) |
| `Merge Join` | Both inputs pre-sorted; merge (good for sorted inputs) |

**When the planner ignores your index:**
- Table statistics are stale → run `ANALYZE`
- Selectivity is too low (index on a boolean with 50/50 split → seq scan is faster)
- Query doesn't match index prefix
- Table is tiny (seq scan is faster than index + heap fetch for < ~500 rows)

---

## Indexing Strategy

### The Five Questions

Before adding an index, ask:

1. **What query does this support?** Identify the specific query and its `WHERE`, `ORDER BY`, `JOIN` columns.
2. **What's the selectivity?** High cardinality (UUID) = high selectivity = good index candidate. Low cardinality (boolean) = bad.
3. **How often is this column updated?** Every write to an indexed column updates the index too.
4. **How many indexes already exist?** More indexes = slower writes, more storage.
5. **Can I use a covering or partial index?** Reduce index size and heap fetches.

### Index Design Heuristics

```
✅ Always index:
   - Primary keys (automatic)
   - Foreign key columns (if you join or filter by them)
   - Columns in high-frequency WHERE clauses (high selectivity)
   - Columns in ORDER BY for paginated queries

✅ Consider:
   - Composite indexes that cover common multi-condition queries
   - INCLUDE columns for covering index on hot read paths
   - Partial indexes for filtered queries with a stable predicate

❌ Avoid:
   - Indexing every column (write overhead, storage waste)
   - Low-cardinality columns (boolean, status with few values)
   - Columns rarely used in queries
   - Duplicate indexes (same leading columns as existing index)
```

### Index Bloat

Indexes grow over time. Deleted rows leave "dead tuples" in the index. In Postgres, autovacuum reclaims space, but high-churn tables may need manual maintenance:

```sql
-- Rebuild an index without locking (Postgres)
REINDEX INDEX CONCURRENTLY idx_orders_user_date;
```

---

## Index Trade-offs Summary

| Index Type | Read Benefit | Write Cost | Space | Best For |
|-----------|-------------|-----------|-------|----------|
| B-Tree (default) | High | Medium | Medium | Point & range queries |
| Composite B-Tree | High | Medium | Medium | Multi-column WHERE/ORDER BY |
| Covering (INCLUDE) | Very high | Higher | Higher | Eliminate heap fetch |
| Partial | High (smaller) | Lower | Lower | Filtered queries |
| Unique | High + constraint | Medium | Medium | Uniqueness enforcement |
| Full-Text (GIN) | High for text | High | High | Keyword search |
| GiST / Spatial | High for geo | High | High | Geographic queries |

---

## Indexing for Scale: Interview Patterns

### Pagination with Indexes

**Offset pagination** is slow at large offsets:
```sql
SELECT * FROM events ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- Must scan and skip 10,000 rows even with index
```

**Keyset pagination** uses the index efficiently:
```sql
SELECT * FROM events WHERE created_at < '2025-01-15' ORDER BY created_at DESC LIMIT 20;
-- Uses index; jumps directly to the right position
```

Index needed: `CREATE INDEX ON events(created_at DESC)`.

### Composite Index for Feed Queries

A user timeline query: "Get posts by users I follow, sorted by time."

```sql
-- After materializing followed user IDs
SELECT * FROM posts WHERE user_id = ANY(ARRAY[1,2,3,...]) ORDER BY created_at DESC LIMIT 20;

-- Index:
CREATE INDEX ON posts(user_id, created_at DESC);
```

With many followed users, this becomes N separate index scans merged with a bitmap. At high fan-out (thousands of follows), denormalize to a precomputed feed table.

### Index on JSON/JSONB

For semi-structured data in Postgres JSONB columns:

```sql
-- GIN index on JSONB for containment queries
CREATE INDEX ON events USING GIN(metadata);
-- Fast: SELECT * FROM events WHERE metadata @> '{"type": "concert"}';

-- Index a specific JSONB key
CREATE INDEX ON events((metadata->>'venue_id'));
-- Fast: WHERE metadata->>'venue_id' = 'v123'
```

---

## Indexes in AI Agent Systems

### Indexing Agent Run History

```sql
-- Fetch all runs for a user, newest first
CREATE INDEX ON agent_runs(user_id, started_at DESC);

-- Find slow runs for monitoring
CREATE INDEX ON agent_runs(completed_at) WHERE status = 'failed';

-- Covering index for the runs list API
CREATE INDEX ON agent_runs(user_id, started_at DESC) INCLUDE (status, agent_type, total_tokens);
```

### Vector Index for Semantic Memory

For agent long-term memory with embedding search:

```sql
-- pgvector with HNSW index (Postgres)
CREATE INDEX ON agent_memories USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Fast ANN query:
SELECT content FROM agent_memories
WHERE user_id = 42
ORDER BY embedding <=> query_embedding
LIMIT 5;
```

HNSW parameters:
- `m`: number of connections per node (16–64). Higher = better recall, more memory.
- `ef_construction`: build-time quality. Higher = better index, slower build.
- `ef_search`: query-time quality (set at query time). Higher = better recall, slower query.

---

## Checklist for Interviews

When discussing storage in a system design interview:

- [ ] Primary key strategy identified (UUID vs sequential)
- [ ] Foreign key indexes mentioned for join columns
- [ ] Index called out for each major `WHERE` clause in the API
- [ ] Composite index column order justified (equality before range)
- [ ] Pagination strategy uses keyset/cursor with supporting index
- [ ] Partial index mentioned for filtered queries (e.g., active records only)
- [ ] Trade-off acknowledged: indexes speed reads, slow writes

---

*Previous: [07 - CAP Theorem](./07-cap-theorem.md)*  
*Next: [09 - Numbers to Know](./09-numbers-to-know.md)*
