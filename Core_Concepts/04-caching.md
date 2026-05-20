# Caching for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** where to cache, cache architectures, eviction policies, cache pitfalls, interview patterns

---

## Why Caching Matters

A read from PostgreSQL typically takes 5–50 ms (network + disk). A read from Redis takes ~0.1–1 ms (memory, in-process network). That's a 50–500× latency improvement.

Caches do two things:
1. **Reduce latency** — serve frequently accessed data from memory
2. **Reduce backend load** — shield databases from repeated identical queries

Both matter at scale. A 95% cache hit rate means your database sees only 5% of read traffic.

---

## Where to Cache

```
Browser / Mobile
        │ HTTP cache (Cache-Control headers)
        ▼
CDN Edge Node
        │ Cached static assets and public API responses
        ▼
Load Balancer / API Gateway
        │
        ▼
Application Server
        │ In-process cache (L1: hot objects in memory)
        ▼
External Cache (Redis / Memcached)      ← most interview discussions live here
        │
        ▼
Primary Database
```

### 1. External Cache (Redis / Memcached)

Standalone cache service shared by all application instances. The canonical interview answer.

**Use Redis when:** you need rich data types (sorted sets, lists, streams), persistence, cluster mode, or pub/sub.  
**Use Memcached when:** you need simple key-value with multiple threads and horizontal memory scaling.

For most designs: **default to Redis**.

#### Redis HA: Sentinel vs Cluster — Know the Difference

These are frequently confused and interviewers notice:

| | Redis Sentinel | Redis Cluster |
|--|--------------|--------------|
| **Purpose** | High availability for a single shard | Horizontal scaling across multiple shards |
| **Data distribution** | All data on one primary (+ replicas) | Data split across 16,384 hash slots across N primaries |
| **Failover** | Sentinel monitors primary; promotes replica on failure | Cluster nodes gossip; auto-promote replica on primary failure |
| **Capacity** | Limited to one node's memory | Scales horizontally (add shards) |
| **Client complexity** | Simple (one endpoint via Sentinel) | Client must understand slot routing |
| **Use when** | You need HA but data fits on one node | Data exceeds one node, or write throughput requires sharding |

At staff level, the right answer for large-scale caches is **Redis Cluster**. For smaller systems where HA matters but scale doesn't, **Sentinel** is simpler.

### 2. CDN

Geographically distributed edge caches. Best for static assets and public content.

```
Without CDN: User in India → origin in Virginia → 280ms
With CDN:    User in India → Mumbai PoP → 20ms (cache hit)
```

Use when: the system serves images, videos, JS/CSS, or public API responses that are the same for all users.

Don't use CDN for: personalized content, private data, or responses that require authentication.

### 3. In-Process Cache (Application-Level)

Cache data inside the process memory (e.g., a dictionary, `functools.lru_cache`, Guava Cache). Zero network overhead.

**Best for:** small, frequently accessed data that rarely changes:
- Feature flags
- Config values
- Hot reference data (currency exchange rates, country codes)
- Per-request deduplication (DataLoader pattern)

**Limitation:** each application instance has its own copy. Cache invalidation becomes a broadcast problem across all instances.

### 4. Client-Side Cache

Browser HTTP cache, mobile app local DB, Redis client cluster-slot cache.

Reduces network calls but limits invalidation control. Use Cache-Control headers (`max-age`, `ETag`, `Last-Modified`) for browser caching.

---

## Cache Architectures

### Cache-Aside (Lazy Loading) — Default

Application manages the cache explicitly.

```
Read path:
  1. Check cache → hit → return data
  2. Cache miss → query database → write to cache → return data

Write path:
  1. Write to database
  2. Invalidate (delete) cache entry
```

```
┌──────────┐    1. GET key?    ┌───────────┐
│   App    │ ────────────────► │   Cache   │
│  Server  │ ◄──────────────── │  (Redis)  │
└──────────┘    2. cache miss  └───────────┘
     │
     │ 3. query
     ▼
┌──────────┐
│    DB    │
└──────────┘
     │ 4. result
     │
     └─► 5. SET key value EX 300 (back into cache)
```

**Pros:** cache only contains what's actually needed; resilient (cache failure degrades gracefully to DB)  
**Cons:** cold start penalty (first request per key always hits DB); potential for stampede

**This is the right default for interviews.**

### Write-Through

Application writes to cache; cache synchronously writes to DB.

```
Write path:
  1. Write to cache
  2. Cache writes to DB (synchronous)
  3. Acknowledge to client
```

**Pros:** cache always has latest data; no stale reads after a write  
**Cons:** write latency higher (must wait for DB); cache fills with data that may never be read; dual-write failure risk

Use when: reads must always return fresh data and you can tolerate slower writes.

### Write-Behind (Write-Back)

Application writes to cache; cache asynchronously batches writes to DB.

```
Write path:
  1. Write to cache
  2. Acknowledge immediately to client
  3. Cache flushes to DB asynchronously (batched)
```

**Pros:** very fast writes; good for write-heavy analytics  
**Cons:** data loss risk if cache crashes before flush; harder to reason about consistency

Use when: high write throughput, eventual persistence acceptable (metrics, analytics, gaming leaderboards).

### Read-Through

Cache manages DB reads itself. Application talks only to the cache.

```
Read path:
  1. Application asks cache for key
  2. Cache hit → return
  3. Cache miss → cache fetches from DB, stores, returns
```

CDNs are a form of read-through cache. Less common for application-level caching because it requires specialized cache infrastructure.

### Architecture Comparison

| Pattern | Write Path | Read Path | Consistency | Complexity |
|---------|-----------|-----------|-------------|------------|
| Cache-Aside | DB only, then invalidate | Cache → miss → DB | Eventual | Low |
| Write-Through | Cache+DB sync | Cache always fresh | Strong | Medium |
| Write-Behind | Cache only, async DB | Cache always fresh | Eventual | High |
| Read-Through | Varies | Cache → auto-fill | Eventual | Medium |

---

## Eviction Policies

Caches have finite memory. When full, they must evict something.

### LRU (Least Recently Used)

Evict the item accessed least recently. Implemented as a doubly-linked list + hash map.

```
Access order: A → B → C → D → A → E (cache full, evict B)
Cache state:  [E, A, D, C]  (B was LRU)
```

**Default choice.** Works well when recent access predicts future access (temporal locality).

### LFU (Least Frequently Used)

Evict the item accessed least often. Maintains frequency counters.

Good for: workloads where popularity is stable over time (top YouTube videos, popular product pages).  
Downside: new items get evicted quickly before they accumulate frequency.

### FIFO (First In First Out)

Evict oldest-inserted item regardless of access pattern. Simple but ignores hotness. Rarely used in practice.

### TTL (Time To Live)

Not an eviction policy per se, but a key expiration mechanism. Every key has a maximum age. Expired keys are evicted lazily (on access) or actively (background sweep).

**Always set TTLs.** Unbounded caches grow until OOM. Typical TTL values:
- User session: 24h–7 days
- API response cache: 1–5 minutes  
- Reference data (currencies, config): 1–60 minutes
- Hot database rows: 30s–5 minutes

---

## Cache Problems and Solutions

### Cache Stampede (Thundering Herd)

**Problem:** A popular key expires. All concurrent requests miss simultaneously and hammer the database.

```
t=0:00  Key expires
t=0:01  1000 simultaneous requests all miss → 1000 DB queries
        Database melts
```

**Solutions:**

1. **Request coalescing / single-flight:** Allow only one request to rebuild the cache. Others wait for the result.

```python
# Go singleflight pattern (pseudo-code)
result, err = sf.Do(key, func() {
    return db.FetchUser(userId)
})
# Only one DB call, everyone gets the result
```

2. **Jitter on TTL:** Randomize expiry across similar keys to stagger evictions:
```python
ttl = base_ttl + random.randint(0, base_ttl * 0.1)  # ±10%
```

3. **Probabilistic early refresh (XFetch):** Start refreshing a key before it expires with probability inversely proportional to remaining TTL.

4. **Cache warming:** Proactively refresh popular keys before expiry using a background job.

### Stale Data / Cache Consistency

**Problem:** Cache shows old data after a write.

```
t=0: User updates email to new@example.com
t=1: DB updated to new@example.com
t=2: Cache still shows old@example.com (TTL=5min)
```

**Solutions:**

1. **Invalidate on write:** After updating DB, delete the cache key. Next read repopulates.
2. **Short TTLs:** Accept brief staleness (e.g., 30s for non-critical data).
3. **Write-through caching:** Update cache and DB atomically (adds complexity).
4. **Event-driven invalidation:** DB change event → cache invalidation message → subscriber deletes key.

**There is no perfect solution.** Choose based on how much staleness your application can tolerate.

#### The Read-Your-Own-Writes Problem

A subtle consistency bug under load balancing: your write hits App Server A (which invalidates its in-process L1 cache), but your *next read* is routed to App Server B (which still has the stale value in its own process cache). You write `new@email.com` and immediately see `old@email.com`.

Solutions:
- Rely only on the **shared external cache (Redis)** — no L1 process cache — for data that must reflect writes immediately
- **Sticky sessions for writes** — route the user to the same server for a short window after a write
- **Per-request dirty flag** — mark a key as invalidated in a request-scoped set; skip L1 cache for those keys

### Hot Keys

**Problem:** A single cache key receives disproportionate traffic, saturating the memory/CPU on one Redis shard.

```
During a product launch, millions of requests all ask for:
  GET product:iphone16   → this key is on Redis node 3 → node 3 overloaded
```

**Solutions:**

1. **Local replica cache:** Copy the hot key into each application server's in-process cache. Reduces Redis load.
2. **Key sharding:** Replicate the value under multiple keys (`product:iphone16:shard0` … `shard9`), and randomly read one.
3. **CDN for public data:** If the hot key is the same for all users, cache it at the CDN level.

### Cache Penetration

**Problem:** Requests for keys that don't exist in DB (e.g., invalid IDs) always miss the cache and hit the DB.

```
Attacker floods: GET /users/nonexistent_id_1, /users/nonexistent_id_2 ...
→ Every request misses cache → DB overwhelmed
```

**Solutions:**

1. **Cache negative results:** Store a null sentinel (`SET user:99999 NULL EX 60`). Future misses return null without hitting DB.
2. **Bloom filter:** A probabilistic data structure that can tell you whether a key definitely doesn't exist. Check bloom filter first; only query DB on potential hits.

```
Bloom filter for valid user IDs:
  Query user:99999 → bloom filter says "definitely not in set" → return 404 without DB query
```

---

## Choosing Cache Scope

| Data Type | Where to Cache | TTL | Invalidation |
|-----------|---------------|-----|--------------|
| User profile | Redis | 5 min | Delete on update |
| User session | Redis | 24h–7d | Delete on logout |
| Product catalog | Redis + CDN | 5 min | Delete on update |
| Static assets | CDN | 1 week–1 year | Version in URL |
| Search results | Redis | 1 min | TTL only |
| Rate limit counters | Redis | Per window (1s–1h) | Auto-expire |
| Feature flags | In-process | 30s | Re-fetch periodically |

---

## Cache Sequence: Cache-Aside Pattern

```
Client          App Server          Redis          PostgreSQL
  │                  │                 │                │
  │── GET /user/42 ──►│                 │                │
  │                  │── GET user:42 ──►│                │
  │                  │◄── (miss) ───────│                │
  │                  │──────────── SELECT * FROM users WHERE id=42 ──►│
  │                  │◄─────────────────────────────────────────── row│
  │                  │── SET user:42 <data> EX 300 ──►│                │
  │◄── 200 + data ───│                 │                │
  │                  │                 │                │
  │── GET /user/42 ──►│   (2nd request) │                │
  │                  │── GET user:42 ──►│                │
  │                  │◄── (hit) data ───│                │
  │◄── 200 + data ───│                 │                │
```

---

## Cache in AI Agent Systems

Caching is critical for reducing LLM inference costs and latency:

### Prompt / Semantic Cache

Cache LLM responses for semantically similar inputs:
- Exact-match cache: hash the prompt → return cached response
- Semantic cache: embed the prompt → find nearest neighbor → if similarity > threshold, return cached response (Redis with vector search)

### KV Cache (Model-Level)

Modern LLMs maintain an internal key-value cache of attention activations for the input context. Designing prompts with stable prefixes (system prompt, few-shot examples) allows the model to reuse KV cache across requests, cutting latency and cost.

### Tool Result Cache

Agent tools (web search, code execution, API calls) are often idempotent for a given input within a time window. Cache tool results to avoid redundant calls:

```python
cache_key = f"tool:{tool_name}:{hash(json.dumps(tool_input, sort_keys=True))}"
ttl = 300  # 5 minutes for most tool calls
```

---

## Interview Cheat Sheet

| Question | Answer |
|----------|--------|
| Default cache solution | Redis (external cache-aside) |
| When to mention CDN | Static assets, media, public API responses |
| When cache-aside breaks | Stampedes; use singleflight + jitter |
| Cache invalidation strategy | Delete-on-write for strong consistency; TTL for eventual |
| Cache hit rate target | >95% for hot-path reads |
| How to handle hot keys | In-process cache replica + key sharding |
| Cache for non-existent keys | Negative caching or Bloom filter |

---

*Previous: [03 - Data Modeling](./03-data-modeling.md)*  
*Next: [05 - Sharding](./05-sharding.md)*
