# Redis Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Data structures, infrastructure configs, capabilities, Pub/Sub, Streams, distributed locks, hot key handling, security, AI agent use cases

---

## Why Redis Is Indispensable

Redis appears in more system designs than almost any other single technology. Its combination of **sub-millisecond latency**, **rich data structures**, **atomic operations**, and **versatile communication primitives** makes it the Swiss-army knife of backend infrastructure.

At staff level, knowing *when* to use Redis (and when not to) and being able to articulate the trade-offs of each pattern is what separates a good candidate from a great one.

---

## Core Architecture

Redis is an **in-memory, single-threaded** data structure server written in C.

- **In-memory:** All data lives in RAM → µs-range read latency, ~100K–1M ops/sec throughput
- **Single-threaded command execution:** No lock contention, no race conditions between commands — atomic operations are trivial
- **Durability is opt-in:** Not a default guarantee. Choose your trade-off:

| Persistence Mode | Mechanism | Data Loss Risk | Performance Cost |
|-----------------|-----------|---------------|-----------------|
| None | No persistence | All data on crash | Zero |
| RDB (snapshot) | Periodic fork + dump | Up to last snapshot interval | Low (fork overhead) |
| AOF (append-only file) | Log every write | Near zero (configurable fsync) | Medium |
| AOF + RDB | Both | Minimal | Higher |
| MemoryDB (AWS) | Write-ahead log to S3 | None | ~30% latency increase |

**Interview tip:** Always mention that Redis is not the source of truth — the primary DB is. Redis loss is survivable; rebuild from DB. If you need durability *and* Redis semantics, propose AWS MemoryDB.

---

## Data Structures

Redis exposes named data structures as first-class primitives. Each has specific use cases:

### Strings
The basic key-value type. Supports atomic increment/decrement.
```
SET user:42:name "Alice"
GET user:42:name          → "Alice"
INCR page:views:home      → atomic counter
SETEX session:abc123 3600 "{...}"  → key with TTL
```

### Hashes
Like a dictionary inside a key. Efficient for objects.
```
HSET user:42 name "Alice" email "a@x.com" plan "pro"
HGET user:42 plan          → "pro"
HGETALL user:42            → all fields
```
Use instead of JSON strings when you need to update individual fields without deserializing the whole object.

### Lists
Ordered, doubly-linked. Efficient push/pop from both ends.
```
LPUSH queue:emails "msg1"
RPOP queue:emails          → dequeue
LRANGE feed:user:42 0 49   → paginate a feed
```
Use for: task queues, activity feeds, recent-item lists.

### Sets
Unordered, unique members. O(1) membership test.
```
SADD online:users user:42
SISMEMBER online:users user:42  → 1
SINTER followers:A followers:B  → mutual followers
```
Use for: online presence, deduplication, set operations.

### Sorted Sets (ZSets)
Members with float scores, ordered by score. O(log N) operations.
```
ZADD leaderboard 1500.0 "player:99"
ZADD leaderboard 2300.0 "player:42"
ZREVRANGE leaderboard 0 9 WITHSCORES  → top 10
ZRANK leaderboard "player:99"          → rank (0-indexed)
```
Use for: leaderboards, priority queues, rate limiting windows, top-K.

### Bloom Filters
Probabilistic set membership. Space-efficient, allows false positives, no false negatives.
```
BF.ADD seen:urls "https://example.com"
BF.EXISTS seen:urls "https://example.com"  → 1 (possibly)
```
Use for: web crawler deduplication, cache penetration defense, spam filtering.

### Geospatial Index
GeoHash-based spatial index built on top of sorted sets.
```
GEOADD drivers:nyc -74.006 40.7128 "driver:42"
GEOSEARCH drivers:nyc FROMLONLAT -73.99 40.72 BYRADIUS 5 km ASC COUNT 10
```
Use for: ride-share driver lookup, delivery proximity, location-based features.

### Streams
Append-only log with consumer groups. Redis's answer to lightweight Kafka.
```
XADD events * type "order_placed" order_id "123"
XREADGROUP GROUP workers consumer1 COUNT 10 STREAMS events >
XACK events workers <message-id>
```
Use for: work queues with delivery guarantees, event sourcing, audit logs.

### HyperLogLog
Probabilistic cardinality estimation. ~12 KB memory for any cardinality, ±0.81% error.
```
PFADD dau:2025-05-20 user:1 user:2 user:3
PFCOUNT dau:2025-05-20  → ~3
PFMERGE dau:week dau:2025-05-18 dau:2025-05-19 dau:2025-05-20
```
Use for: DAU/MAU counting, unique visitors, cardinality at scale without full sets.

---

## Infrastructure Configurations

### Single Node
Development, low-traffic. No HA.

### Sentinel (High Availability, Single Shard)
One primary + N replicas. Sentinel processes monitor health and auto-promote a replica if the primary fails.

```
Primary ──replicates──► Replica 1
       └──replicates──► Replica 2

Sentinel 1 ─┐
Sentinel 2 ─┼── monitor all three, vote on failover
Sentinel 3 ─┘
```

- Clients connect via Sentinel to discover the current primary
- Failover takes ~30s (configurable)
- **No horizontal scaling** — all data on one node
- Use when: data fits in RAM of one node, need HA, simple ops

### Cluster (Horizontal Scaling + HA)
Data split across 16,384 hash slots distributed across primary nodes. Each primary has replicas.

```
Node A (slots 0–5460)     → Replica A'
Node B (slots 5461–10922) → Replica B'
Node C (slots 10923–16383)→ Replica C'
```

- Clients cache the slot→node map locally
- `MOVED` redirect when slot moves during rebalancing
- **All keys in a multi-key command must be on the same slot** (use hash tags: `{user:42}:profile`)
- Use when: data exceeds single-node RAM, need write-scale

**Interview note:** Know the Sentinel vs Cluster distinction cold. A common mistake is saying "Redis Cluster" when you mean "Redis with replication" — they're different things.

---

## Key Capabilities and Patterns

### 1. Cache (Cache-Aside)

```
function getUser(id):
  data = redis.GET(f"user:{id}")
  if data: return deserialize(data)
  data = db.query("SELECT * FROM users WHERE id = ?", id)
  redis.SETEX(f"user:{id}", 300, serialize(data))  # 5-min TTL
  return data
```

**Write invalidation:**
```
function updateUser(id, payload):
  db.update(users, id, payload)
  redis.DEL(f"user:{id}")  # invalidate, let next read repopulate
```

**Cache stampede defense — singleflight pattern:**
```
# Only one goroutine/thread rebuilds; others wait for the result
result = singleflight.Do(f"user:{id}", lambda: db.query(...))
```

### 2. Distributed Lock (Redlock)

Simple lock with timeout:
```
# Acquire lock (NX = only if not exists, EX = TTL in seconds)
acquired = redis.SET("lock:order:123", "worker-uuid", NX=True, EX=30)
if not acquired:
    raise LockBusy()

try:
    # critical section
    process_order(123)
finally:
    # Only release if we still own it (Lua for atomicity)
    redis.eval("""
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      end
    """, 1, "lock:order:123", "worker-uuid")
```

**Redlock (multi-node quorum lock):**
For strong guarantees, acquire the lock on N/2+1 independent Redis nodes. A lock is valid only if acquired on the majority within a short time window. Handles single-node failure without releasing the lock erroneously.

**Caution:** Even Redlock has edge cases under GC pauses and clock skew. For financial operations, prefer database-level pessimistic locks. Use Redis locks for lower-stakes coordination (rate limiting, task deduplication).

### 3. Rate Limiter

**Fixed-window:**
```
key = f"rate:{user_id}:{current_minute}"
count = redis.INCR(key)
redis.EXPIRE(key, 60)  # reset after window
if count > LIMIT: raise TooManyRequests()
```

**Sliding window with sorted set:**
```lua
-- Lua script (atomic)
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)
if count < limit then
  redis.call('ZADD', key, now, now)
  redis.call('EXPIRE', key, window)
  return 1  -- allowed
end
return 0  -- rejected
```

### 4. Leaderboard

```
# Score a user
ZADD game:leaderboard 9850 "user:42"

# Top 10
ZREVRANGE game:leaderboard 0 9 WITHSCORES

# User's rank (0-indexed from top)
ZREVRANK game:leaderboard "user:42"

# Users within score range
ZRANGEBYSCORE game:leaderboard 9000 10000
```

Handles millions of entries with O(log N) per operation.

### 5. Pub/Sub (Real-Time Fan-out)

```
# Publisher (server receiving event)
redis.spublish("chat:room:42", json.dumps({"user": "Alice", "msg": "hi"}))

# Subscriber (WebSocket server)
pubsub = redis.pubsub()
pubsub.ssubscribe("chat:room:42")
for message in pubsub.listen():
    broadcast_to_ws_connections(message)
```

**Critical properties:**
- Messages are **not persisted** — offline subscribers miss messages
- Delivery is **at-most-once**
- Clients hold **one connection per cluster node** (not per channel) — scales efficiently
- Sharded Pub/Sub (SPUBLISH/SSUBSCRIBE) routes channels to specific cluster nodes

**When Redis Pub/Sub is not enough:** Use Kafka or Streams when you need message persistence, replay, or at-least-once delivery guarantees.

### 6. Streams (Durable Queue)

```
# Producer
XADD orders * order_id 123 amount 5000 user_id 42

# Consumer group setup (once)
XGROUP CREATE orders workers $ MKSTREAM

# Consumer reads
XREADGROUP GROUP workers worker-1 COUNT 10 BLOCK 2000 STREAMS orders >

# Acknowledge processed message
XACK orders workers <msg-id>

# Reclaim messages from dead workers (pending > 60s)
XCLAIM orders workers worker-2 60000 <msg-id>
```

Streams give you: ordering, consumer groups, message acknowledgment, dead-letter handling, and replay. Think lightweight Kafka within the Redis stack.

---

## Hot Key Problem and Solutions

A "hot key" is a single Redis key receiving disproportionate traffic, overwhelming one node.

**Example:** Product detail for a viral item gets 500K req/s — all hitting the same Redis node.

### Solutions (in order of preference):

**1. In-process (L1) cache:** Cache the hot value in application memory for 1–5 seconds. Eliminates Redis round-trips entirely for the hottest keys.
```python
@lru_cache(maxsize=1000, ttl=2)  # 2-second in-process TTL
def get_product(id): return redis.get(f"product:{id}")
```

**2. Key replication (fan-out reads):**
```python
# Write to N shards
for i in range(10):
    redis.set(f"product:{id}:shard:{i}", data)

# Read from random shard
shard = random.randint(0, 9)
redis.get(f"product:{id}:shard:{shard}")
```

**3. CDN for public data:** If the hot key serves the same content to all users (product detail, configuration), cache at the CDN edge. Zero Redis load.

**4. Read replicas with dynamic scaling:** Add read replicas behind a load balancer, scale based on CPU metrics.

---

## Security Considerations

- **No auth by default:** Always enable `requirepass` in production. Use Redis ACLs (6.0+) for fine-grained command/key-level access control.
- **No TLS by default:** Enable `tls-port` and configure certs. Use stunnel or a proxy if your Redis version doesn't support native TLS.
- **Never expose Redis to the public internet.** Bind to private IP only (`bind 127.0.0.1 10.0.0.5`). Countless Redis instances have been compromised via open ports.
- **Command renaming:** Disable dangerous commands in production: `rename-command FLUSHALL ""` and `rename-command CONFIG ""`.
- **Lua script injection:** Validate all EVAL arguments. Lua runs as root within Redis — a malicious script can access any key.

---

## Redis vs Alternatives

| Use Case | Redis | Memcached | Kafka | RabbitMQ |
|----------|-------|-----------|-------|----------|
| Simple cache | ✅ Best | ✅ Simpler | ❌ | ❌ |
| Rich data structures | ✅ | ❌ | ❌ | ❌ |
| Pub/Sub (ephemeral) | ✅ | ❌ | ✅ | ✅ |
| Durable message queue | ⚠️ Streams | ❌ | ✅ Best | ✅ |
| High-throughput streaming | ⚠️ | ❌ | ✅ Best | ❌ |
| Distributed lock | ✅ | ❌ | ❌ | ✅ |
| Geo search | ✅ | ❌ | ❌ | ❌ |
| Leaderboard | ✅ | ❌ | ❌ | ❌ |
| Persistence guarantee | ⚠️ MemoryDB | ❌ | ✅ | ✅ |

---

## AI Agent Use Cases

### Prompt / Semantic Cache
```python
# Exact match: hash the prompt
cache_key = f"llm:{sha256(prompt)}"
if result := redis.get(cache_key):
    return json.loads(result)
result = llm.generate(prompt)
redis.setex(cache_key, 3600, json.dumps(result))

# Semantic match: store embedding → use Redis VSS (Vector Similarity Search)
redis.hset(f"prompt:{id}", mapping={
    "text": prompt,
    "embedding": embedding.tobytes(),
    "response": response
})
```

### Agent Session / State
```python
# Store agent working memory per run (auto-expire after 1h)
redis.setex(f"agent:state:{run_id}", 3600, json.dumps(state))

# Pub/Sub for streaming agent steps to UI
redis.publish(f"agent:stream:{run_id}", json.dumps(step_event))
```

### Tool Result Cache
```python
cache_key = f"tool:{tool_name}:{sha256(json.dumps(args, sort_keys=True))}"
if cached := redis.get(cache_key):
    return json.loads(cached)
result = execute_tool(tool_name, args)
redis.setex(cache_key, 300, json.dumps(result))  # 5-min TTL
```

---

## Interview Quick Reference

| Scenario | Redis Pattern | Key Commands |
|----------|-------------|-------------|
| Cache with TTL | String + SETEX | SET/GET/SETEX/DEL |
| Session store | Hash or String | HSET/HGET/EXPIRE |
| Rate limiter | INCR + EXPIRE or ZSet | INCR/EXPIRE/ZADD/ZCARD |
| Distributed lock | SET NX EX + Lua | SET NX EX / EVAL |
| Leaderboard | Sorted Set | ZADD/ZREVRANGE/ZRANK |
| Online users | Set | SADD/SISMEMBER/SCARD |
| Task queue | List or Stream | LPUSH/RPOP / XADD/XREADGROUP |
| Pub/Sub fan-out | Pub/Sub | SPUBLISH/SSUBSCRIBE |
| Geo proximity | Geo Index | GEOADD/GEOSEARCH |
| Cardinality count | HyperLogLog | PFADD/PFCOUNT |
| Bloom filter | BF module | BF.ADD/BF.EXISTS |
| Hot key mitigation | Key sharding + L1 cache | Multiple keys + local dict |

---

*Next: [02 - Elasticsearch](./02-elasticsearch.md)*
