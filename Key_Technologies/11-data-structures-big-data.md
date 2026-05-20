# Data Structures for Big Data

> **Target audience:** Staff+ backend engineers  
> **Covers:** Bloom filters, Count-Min Sketch, HyperLogLog, MinHash/LSH, Skip lists, Merkle trees — when to use each and why

---

## Why Probabilistic Data Structures?

At big-data scale, exact computation becomes impossible or economically impractical:

- **Counting distinct users:** Exact set membership for 10B users requires 10B × 8 bytes = 80GB of RAM
- **Checking URL seen before (web crawler):** Hash set of 1B URLs = ~40GB
- **Frequency of each word in 1TB of text:** HashMap grows unboundedly

**Probabilistic data structures** trade a small, bounded error rate for dramatic reductions in memory and time. They answer approximate questions ("about how many?", "probably seen this?") using a tiny fraction of the memory.

---

## Bloom Filter

### What It Does

A Bloom filter answers: **"Is this element in the set?"**
- "Definitely NOT in set" — 100% accurate (no false negatives)
- "Probably in set" — may have false positives (tunable false positive rate)

Never modified after elements are added (no deletion in basic form).

### How It Works

```
Bit array of m bits (all 0 initially)
k hash functions

ADD "hello":
  h1("hello") = 3 → set bit 3
  h2("hello") = 7 → set bit 7
  h3("hello") = 12 → set bit 12

CHECK "hello":
  h1("hello") = 3 → bit 3 is 1 ✓
  h2("hello") = 7 → bit 7 is 1 ✓
  h3("hello") = 12 → bit 12 is 1 ✓
  → "Probably in set"

CHECK "world" (not added):
  h1("world") = 3 → bit 3 is 1 ✓  (coincidence)
  h2("world") = 2 → bit 2 is 0 ✗
  → "Definitely NOT in set"
```

### Sizing

```
n = expected elements
p = desired false positive rate

Optimal bit array size: m = -(n × ln(p)) / (ln(2)²)
Optimal hash functions: k = (m/n) × ln(2)

Example: 1 billion URLs, 1% false positive rate
  m = -(1B × ln(0.01)) / (ln(2)²) = 9.59 bits/element ≈ 10 bits
  Total: 10B bits = 1.25 GB  (vs 40 GB for exact hash set)
  k = 7 hash functions
```

### Use Cases

**Web crawler deduplication:**
```python
bloom = BloomFilter(capacity=1_000_000_000, error_rate=0.01)

def should_crawl(url):
    if url in bloom:
        return False   # probably seen (1% false positive rate)
    bloom.add(url)
    return True
```

**Cache penetration defense:**
```python
# Bloom filter of all valid user IDs
# Any ID not in bloom → 100% not in DB → return 404 without DB query
if user_id not in valid_users_bloom:
    return 404
```

**Cassandra / HBase read optimization:** Each SSTable has a Bloom filter. On a point lookup, check the Bloom filter first — if key is definitely not in this SSTable, skip it entirely. Dramatically reduces I/O on read path.

**Database existence checks (Rocksdb, Cassandra):**
Built into most LSM-Tree based storage engines to skip SSTables on point reads.

### Counting Bloom Filter
Allows deletions by using counters instead of bits. Trade-off: 4× more memory. Use when you need to delete elements from the filter.

---

## Count-Min Sketch

### What It Does

Answers: **"Approximately how many times have I seen X?"** — frequency estimation over a data stream.

Memory usage: O(ε⁻¹ × ln(δ⁻¹)) — independent of number of distinct elements.

### How It Works

```
2D array: w columns × d rows
d hash functions (one per row)

ADD "redis" (count=1):
  h1("redis") = 3 → sketch[0][3] += 1
  h2("redis") = 7 → sketch[1][7] += 1
  h3("redis") = 1 → sketch[2][1] += 1

ADD "kafka" (count=1):
  h1("kafka") = 3 → sketch[0][3] += 1  (collision with "redis"!)
  h2("kafka") = 2 → sketch[1][2] += 1
  h3("kafka") = 5 → sketch[2][5] += 1

QUERY "redis" frequency:
  h1("redis") = 3 → sketch[0][3] = 2  (overcounted due to "kafka" collision)
  h2("redis") = 7 → sketch[1][7] = 1
  h3("redis") = 1 → sketch[2][1] = 1
  → take MINIMUM = 1 (correct!)
```

The minimum across all rows cancels out overcount from hash collisions.

### Use Cases

**Top-K heavy hitters (trending topics, popular products):**
```python
# Track word frequencies in a streaming text corpus
sketch = CountMinSketch(width=1000, depth=5)
for word in word_stream:
    sketch.add(word)

# Query frequency
freq = sketch.query("redis")  # approximate frequency
```

**Network traffic analysis:** Count packets per source IP over a sliding window without storing all IPs.

**Rate limiting per endpoint:** Approximate per-user request count without a full hash map.

**Ad click frequency:** Count ad impressions per user without exact user state.

---

## HyperLogLog

### What It Does

Answers: **"How many distinct elements have I seen?"** (cardinality estimation)

Error rate: ~0.81% with 12 KB of memory, regardless of cardinality (works for billions of distinct elements).

### How It Works

The intuition: in a uniformly random hash, the probability of a string of k leading zeros is 2⁻ᵏ. If the maximum leading zeros seen is k, you've seen approximately 2ᵏ distinct elements.

```
Hash elements → observe maximum number of leading zeros in binary hash
  "user:1" → hash = 0b00110101... → 2 leading zeros → seen at most 4 distinct
  "user:2" → hash = 0b00001101... → 4 leading zeros → seen at most 16 distinct
  "user:3" → hash = 0b10110101... → 0 leading zeros → seen at most 1 distinct
  ...
  Max = 4 leading zeros → estimate 2⁴ = 16 distinct elements

HyperLogLog: uses multiple registers (m=2^14 = 16384) and harmonic mean for accuracy
```

### Use Cases

**Daily/Monthly Active Users:**
```python
# Redis HyperLogLog
redis.pfadd(f"dau:{today}", user_id)  # add user to today's HLL
redis.pfcount(f"dau:{today}")          # approximate DAU count

# Merge across days for MAU
redis.pfmerge("mau:jan", *[f"dau:2025-01-{d:02d}" for d in range(1,32)])
redis.pfcount("mau:jan")
```

**Unique visitors per page:**
```python
redis.pfadd(f"visitors:{page_id}:{hour}", ip_address)
redis.pfcount(f"visitors:{page_id}:{hour}")
```

**Distinct query count in analytics:** How many distinct SQL queries hit the database in the last hour?

### HyperLogLog vs Exact Count

| | Exact Set | HyperLogLog |
|--|----------|------------|
| Memory (1B elements) | ~8 GB | 12 KB |
| Error | 0% | ~0.81% |
| Merge | Union (expensive) | Merge HLLs (O(m)) |
| Use when | Need exact count | Cardinality estimation OK |

---

## MinHash / LSH (Locality-Sensitive Hashing)

### What It Does

Answers: **"Which of these documents are similar to each other?"** — approximate similarity search.

Finds near-duplicates in a large corpus without comparing every pair (O(n²) → O(n)).

### Jaccard Similarity

```
Set A = {"the", "cat", "sat", "on", "mat"}
Set B = {"the", "cat", "sat", "on", "floor"}

Jaccard(A, B) = |A ∩ B| / |A ∪ B| = 4 / 6 = 0.667

MinHash approximates Jaccard without computing exact sets
```

### How MinHash Works

```
1. Apply k different hash functions to each document's shingles
2. For each hash function, record the MINIMUM hash value (the "signature")
3. Signature = [min_h1, min_h2, ..., min_hk] — fixed-length vector

Similarity estimate:
  P(min_h(A) == min_h(B)) ≈ Jaccard(A, B)
  Compare signature vectors: fraction of matching min-hash values ≈ Jaccard
```

### LSH (Locality-Sensitive Hashing)

Group similar items into the same "bucket" without comparing all pairs:

```
1. Compute MinHash signatures for all documents
2. Divide signatures into bands of r rows
3. Documents in the same band that hash to same bucket → candidate pair
4. Check only candidate pairs exactly

With b bands × r rows, probability of detection:
  P(candidate) = 1 - (1 - s^r)^b  where s = Jaccard similarity

Tune b and r to control the similarity threshold
```

### Use Cases

**Near-duplicate detection** (spam filters, content deduplication):
```
Web crawl: 10B pages → find near-duplicate content
Without LSH: O(n²) = 100 quintillion comparisons (impossible)
With LSH: O(n) comparison of candidates only
```

**Recommendation systems:** Find users with similar purchase histories without O(n²) pairwise comparison.

**Plagiarism detection:** Flag documents with Jaccard similarity > 0.7.

---

## Skip List

### What It Does

A probabilistic data structure that provides **sorted key-value storage** with O(log N) average search, insert, and delete — without the rebalancing complexity of balanced BSTs (AVL, Red-Black trees).

### How It Works

```
Level 3: head ─────────────────────────── 50 ────────────────── tail
Level 2: head ─────────── 20 ──────────── 50 ─────────── 80 ─── tail
Level 1: head ──── 10 ─── 20 ─── 30 ───── 50 ─── 60 ─── 80 ─── tail
Level 0: head ─ 5 ─ 10 ─ 20 ─ 30 ─ 40 ─ 50 ─ 60 ─ 70 ─ 80 ─ 90 ─ tail

Search for 60:
  Level 3: head → 50 (too small) → tail (overshot) → drop
  Level 2: 50 → 80 (overshot) → drop
  Level 1: 50 → 60 ✓ found!
```

Each node is promoted to the next level with probability 0.5 (coin flip). Expected height: O(log N).

### Use Cases

**Redis Sorted Sets:** Redis uses a skip list as the primary data structure for sorted sets (ZSets). Efficient O(log N) insert/delete/rank queries.

**Database indexes (in-memory):** Some in-memory databases use skip lists because they support concurrent modification more naturally than B-Trees (no rebalancing = fewer lock contention points).

**Range queries:** Because the base level is sorted, range scans are efficient — traverse linked list from start to end of range.

---

## Merkle Tree

### What It Does

A **Merkle tree** (hash tree) enables efficient and secure verification that two datasets are identical — or finding exactly which parts differ.

### How It Works

```
Data blocks: [A] [B] [C] [D]

Leaf hashes:    H(A)       H(B)       H(C)       H(D)
                  \         /           \         /
Level 1:         H(H(A)+H(B))          H(H(C)+H(D))
                        \                   /
Root:                H(H(AB) + H(CD))

Root hash = "fingerprint" of entire dataset
```

To check if datasets match: **compare root hashes** (O(1)).  
To find which block differs: **traverse the tree** (O(log N)).  
To prove a block is in the dataset: **Merkle proof** (a path of sibling hashes, O(log N) size).

### Use Cases

**Git:** Every commit is a Merkle tree of file trees. `git clone` verifies data integrity via hashes.

**Blockchain:** Bitcoin and Ethereum use Merkle trees for transaction verification. The block header contains the Merkle root — any transaction change invalidates the root.

**Cassandra anti-entropy (read repair):** Each node computes a Merkle tree over its data. Nodes exchange Merkle roots periodically. If roots differ, they traverse to find divergent sub-trees and sync only those segments. Without Merkle trees, nodes would have to send/compare all data.

**Dynamo-style databases:** Used for replica sync to efficiently identify which key ranges need repair after a node recovers from failure.

**IPFS / distributed storage:** Content-addressed chunks form a Merkle DAG. Any node can verify it received uncorrupted data by checking hashes.

---

## Summary Comparison

| Structure | Problem Solved | Memory | Error | Supports Delete |
|-----------|---------------|--------|-------|-----------------|
| Bloom Filter | Set membership | Very small | False positives | No (basic) |
| Count-Min Sketch | Frequency estimation | Small | Overcount only | No |
| HyperLogLog | Cardinality estimation | Tiny (12KB) | ~0.81% | No |
| MinHash / LSH | Similarity search | Proportional to signatures | Approximate Jaccard | Rebuild |
| Skip List | Sorted KV store | O(N log N) | None (exact) | Yes |
| Merkle Tree | Data integrity / diff | O(N) hashes | None (exact) | Rebuild |

---

## Interview Application

| System Design Problem | Data Structure | Reason |
|----------------------|---------------|--------|
| Web crawler (seen URLs?) | Bloom filter | 10B URLs in 1.25GB |
| Trending topics (top-K words) | Count-Min Sketch + min-heap | Streaming frequency |
| DAU/MAU metric | HyperLogLog | Cardinality without full set |
| Duplicate content detection | MinHash + LSH | O(n) vs O(n²) comparisons |
| Leaderboard (top scores) | Skip list (Redis ZSet) | O(log N) rank queries |
| Distributed replica sync | Merkle tree | Find divergent ranges efficiently |
| Database SSTable reads | Bloom filter | Skip SSTables that don't have the key |

---

*Previous: [10 - Time Series Databases](./10-time-series-databases.md)*  
*Next: [12 - Vector Databases](./12-vector-databases.md)*
