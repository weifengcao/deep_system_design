# Elasticsearch Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Inverted index, sharding/replication, query types, aggregations, sync patterns, pitfalls, AI/vector search

---

## Why Elasticsearch

Elasticsearch (ES) is the go-to solution whenever a system needs **full-text search, faceted filtering, or aggregations at scale**. It handles queries that relational databases handle poorly: "find all documents containing 'machine learning' ranked by relevance, filtered to the last 30 days, grouped by author."

**Use Elasticsearch when:**
- Full-text search with relevance ranking
- Faceted/multi-attribute filtering (e-commerce: price + brand + rating + category)
- Log/event analytics (the ELK stack)
- Autocomplete / type-ahead suggestions
- Fuzzy matching / typo tolerance

**Do NOT use as primary store:** ES has no ACID transactions, eventual consistency between replicas, and complex failure modes. Always sync from a primary database.

---

## How Elasticsearch Works

### The Inverted Index

ES's core data structure. For each unique term in a field, it stores a list of documents containing that term — the inverse of a row-based DB.

```
Text corpus:
  doc1: "Redis is fast and in-memory"
  doc2: "Kafka is fast and distributed"
  doc3: "Redis is distributed and persistent"

Inverted index for field "body":
  "redis"       → [doc1, doc3]
  "fast"        → [doc1, doc2]
  "in-memory"   → [doc1]
  "kafka"       → [doc2]
  "distributed" → [doc2, doc3]
  "persistent"  → [doc3]

Query: "redis distributed"
  → intersect [doc1, doc3] ∩ [doc2, doc3] = [doc3]
  → OR: [doc1, doc2, doc3] scored by term frequency
```

**Text analysis pipeline** (happens at index time and query time):
```
"Machine Learning!" 
    → tokenize → ["Machine", "Learning"]
    → lowercase → ["machine", "learning"]
    → stem      → ["machin", "learn"]
    → remove stopwords → ["machin", "learn"]  (no stopwords here)
    → store in inverted index
```
This is why "Machines" matches "machine learning" — they share the same stem. The analyzer must be the same at index and query time.

### Documents, Indices, Shards

```
Elasticsearch Cluster
├── Index: "products"          (like a database table)
│   ├── Primary Shard 0        (one chunk of data, one Lucene instance)
│   ├── Primary Shard 1
│   ├── Primary Shard 2
│   ├── Replica of Shard 0     (copy on different node)
│   ├── Replica of Shard 1
│   └── Replica of Shard 2
└── Index: "orders"
    └── ...
```

- **Document:** A JSON object, the atomic unit (like a row)
- **Index:** A collection of documents with a shared mapping (schema)
- **Shard:** A Lucene index. Primary shards hold data; replica shards are copies for HA and read scaling
- **Node:** A JVM process, can host many shards

**Shard count is fixed at index creation.** Choosing the wrong shard count is a common operational mistake:
- Too few shards → can't distribute to all nodes, hot shards
- Too many shards → overhead per query (scatter-gather), memory pressure

Rule of thumb: aim for 10–50 GB per shard. For 1 TB of data, 20–50 shards is reasonable.

### Cluster Architecture

```
┌──────────────────────────────────────────┐
│             Coordinating Node             │
│  (receives query, fans out to shards,    │
│   merges results, returns to client)     │
└──────────────┬───────────────────────────┘
               │ scatter
    ┌──────────┼──────────┐
    ▼          ▼          ▼
  Node 1     Node 2     Node 3
 [P0][R1]  [P1][R2]   [P2][R0]
```

Every search fans out to all primary (or replica) shards, each returning their top-N hits. The coordinating node then merges and re-ranks. This "scatter-gather" is efficient for low shard counts but expensive as shards multiply.

---

## Indexing Documents

### Mapping (Schema)

Define field types before indexing. ES will infer types if you don't, but this often leads to wrong behavior (e.g., numbers stored as strings).

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name":        { "type": "text", "analyzer": "english" },
      "brand":       { "type": "keyword" },
      "price":       { "type": "float" },
      "rating":      { "type": "float" },
      "description": { "type": "text", "analyzer": "english" },
      "tags":        { "type": "keyword" },
      "created_at":  { "type": "date" },
      "location":    { "type": "geo_point" }
    }
  }
}
```

**`text` vs `keyword`:**
- `text`: analyzed, tokenized, supports full-text search but NOT exact match, aggregation, or sorting
- `keyword`: stored as-is, supports exact match, aggregation, sorting, and faceting
- For "search and filter": use `fields` to index as both:
```json
"brand": {
  "type": "text",
  "fields": { "keyword": { "type": "keyword" } }
}
```

### Indexing a Document

```json
POST /products/_doc/123
{
  "name": "MacBook Pro 16",
  "brand": "Apple",
  "price": 2499.00,
  "rating": 4.8,
  "tags": ["laptop", "apple", "pro"]
}
```

ES writes to primary shard, replicates to replicas. Write is acknowledged when replicas confirm (configurable: `wait_for_active_shards`).

**Near-real-time:** ES buffers writes in memory (the "translog") and flushes to a new Lucene segment every 1 second by default. Documents are searchable within 1 second of indexing, not immediately — this is "near-real-time" (NRT), not real-time.

---

## Query Types

### Match Query (Full-Text)
```json
GET /products/_search
{
  "query": {
    "match": {
      "description": "fast in-memory database"
    }
  }
}
```
Analyzes the query string, searches the inverted index. Returns documents ranked by BM25 relevance score.

### Term Query (Exact Match)
```json
{ "query": { "term": { "brand": "Apple" } } }
```
No analysis — use for `keyword` fields. Never use on `text` fields (won't match analyzed tokens).

### Range Query
```json
{ "query": { "range": { "price": { "gte": 100, "lte": 500 } } } }
```

### Bool Query (Combine Multiple)
```json
{
  "query": {
    "bool": {
      "must":    [{ "match": { "description": "laptop" } }],
      "filter":  [
        { "term":  { "brand": "Apple" } },
        { "range": { "price": { "lte": 3000 } } }
      ],
      "should":  [{ "term": { "tags": "pro" } }],
      "must_not":[{ "term": { "tags": "refurbished" } }]
    }
  }
}
```
- `must`: must match, contributes to score
- `filter`: must match, **no score** (cached, faster)
- `should`: nice-to-have, boosts score
- `must_not`: must not match

**Performance rule:** Always put exact-match and range conditions in `filter` (not `must`) to leverage filter caching.

### Aggregations (Analytics)
```json
{
  "aggs": {
    "by_brand": {
      "terms": { "field": "brand", "size": 10 }
    },
    "avg_price": {
      "avg": { "field": "price" }
    },
    "price_histogram": {
      "histogram": { "field": "price", "interval": 100 }
    }
  }
}
```
Returns: top 10 brands by count, average price, price distribution. This is what powers e-commerce faceted navigation.

### Fuzzy / Typo-Tolerant Search
```json
{ "query": { "fuzzy": { "name": { "value": "labtop", "fuzziness": "AUTO" } } } }
```
`fuzziness: AUTO` allows 1 edit for strings ≤5 chars, 2 edits for longer. Catches common typos.

---

## Sync Patterns: Keeping ES and Primary DB in Sync

ES is never the source of truth. The primary DB is. ES is a read-optimized projection. The critical challenge is keeping them in sync.

### Pattern 1: Synchronous Dual-Write (Simple, Fragile)
```
Write to DB → Write to ES → Return to client
```
**Problem:** If the ES write fails, DB and ES diverge. Not atomic. Avoid for anything important.

### Pattern 2: Async via Message Queue (Recommended)
```
Write to DB → Publish event to Kafka → Consumer reads → Write to ES
```
- DB and Kafka write can be atomic (transactional outbox pattern)
- ES write is eventually consistent (typically < 1s lag)
- Kafka consumer retries on ES failure
- **Interview default**

```
┌──────────┐    write    ┌──────┐    publish   ┌───────┐
│  Client  │────────────►│  DB  │──────────────►│Kafka  │
└──────────┘             └──────┘              └───┬───┘
                                                   │ consume
                                               ┌───▼───────┐
                                               │ES Indexer │
                                               └───────────┘
                                                   │ index
                                               ┌───▼───┐
                                               │  ES   │
                                               └───────┘
```

### Pattern 3: CDC (Change Data Capture)
Use Debezium or similar to stream DB WAL changes → Kafka → ES. Fully decoupled from application write path. Best for large existing datasets or when you can't change application code.

---

## Operational Pitfalls

### Mapping Explosion
Don't index arbitrary JSON keys (user-defined field names). Each unique key becomes a field in the mapping. 10,000 unique fields → massive memory overhead and cluster instability.
**Fix:** Use `dynamic: false` and define mappings explicitly. Store arbitrary data in a `text` field if searchable, or `object` with `enabled: false` if not.

### Split-Brain / Cluster Failures
ES uses a quorum-based master election. With 3 master-eligible nodes, you need 2 to form a quorum. With 2 nodes, a network partition gives each node a majority of 1 — split-brain. Always use odd numbers of master-eligible nodes (3 or 5).

### Heap Size
ES should use no more than 50% of available RAM for heap (the other 50% goes to the OS page cache for Lucene files). Cap at 26–30 GB (above this, G1GC becomes inefficient). For 64 GB RAM server: 30 GB heap, 34 GB page cache.

### Index vs Field Count
ES clusters struggle beyond ~10,000 indices or 1,000 shards per node. Use Index Lifecycle Management (ILM) to roll over and delete old indices (crucial for log data).

---

## Elasticsearch vs Alternatives

| Capability | Elasticsearch | Postgres FTS | Algolia | Typesense |
|-----------|--------------|-------------|---------|-----------|
| Full-text search | ✅ Best-in-class | ✅ Good | ✅ Excellent | ✅ Good |
| Fuzzy/typo tolerance | ✅ | ❌ Limited | ✅ | ✅ |
| Faceted filtering | ✅ | ⚠️ Slow at scale | ✅ | ✅ |
| Analytics/aggregations | ✅ | ⚠️ | ❌ | ❌ |
| Log analytics | ✅ (ELK stack) | ❌ | ❌ | ❌ |
| Self-hosted | ✅ | ✅ | ❌ | ✅ |
| Managed | ✅ (Elastic Cloud) | ✅ | ✅ SaaS | ✅ |
| Ops complexity | High | Low | Zero | Low |
| Interview default | ✅ Yes | Simple search only | Small teams | Small teams |

**Rule of thumb:** Propose Elasticsearch in interviews for any serious search problem. Use Postgres FTS only if the team already uses Postgres and search is not the primary feature.

---

## Vector Search in Elasticsearch (AI Use Cases)

ES 8.x supports dense vector fields and k-NN (approximate nearest neighbor) search via HNSW:

```json
PUT /documents
{
  "mappings": {
    "properties": {
      "content": { "type": "text" },
      "embedding": {
        "type": "dense_vector",
        "dims": 1536,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}

GET /documents/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 10,
    "num_candidates": 100
  }
}
```

**Hybrid search (keyword + vector):** Combine BM25 text score and vector similarity score using `rrf` (Reciprocal Rank Fusion) or custom weighting. Critical for RAG (Retrieval-Augmented Generation) systems where exact keyword matches matter alongside semantic similarity.

---

## Interview Quick Reference

| Question | Answer |
|----------|--------|
| Why not Postgres for search? | No BM25 ranking, no fuzzy match, poor aggregation performance at scale |
| How is ES consistent with primary DB? | Eventually consistent via async Kafka pipeline |
| How do you scale ES? | More shards (set at index time), more nodes, read replicas |
| ES as primary store? | Never — no ACID, complex failure modes |
| How does text matching work? | Analyzer → tokenize → lowercase → stem → inverted index lookup |
| Latency for search? | 10–100ms for typical queries on well-sized clusters |
| What's near-real-time? | 1s refresh interval — writes searchable within ~1s |

---

*Previous: [01 - Redis](./01-redis.md)*  
*Next: [03 - Kafka](./03-kafka.md)*
