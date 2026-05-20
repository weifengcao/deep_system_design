# Consistent Hashing for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** the problem with simple modulo hashing, the hash ring, virtual nodes, real-world usage

---

## The Problem: Simple Modulo Hashing Breaks on Resize

When you shard data across N nodes using simple modulo:

```
node = hash(key) % N
```

This works fine — until you add or remove a node.

### Adding a Node (N=3 → N=4)

```
Before (3 nodes):
  key "user:42" → hash = 819 → 819 % 3 = 0 → Node 0
  key "user:99" → hash = 337 → 337 % 3 = 1 → Node 1

After (4 nodes):
  key "user:42" → hash = 819 → 819 % 4 = 3 → Node 3  ← MOVED
  key "user:99" → hash = 337 → 337 % 4 = 1 → Node 1  ← same (lucky)
```

When N changes, most keys map to a different node. For a 3→4 node resize, **~75% of keys move**. For a 10→11 node resize, **~91% of keys move**. Every moved key requires data migration — causing massive I/O spikes and brief unavailability.

### Removing a Node (Failure)

Same problem. A node goes down, N decreases by 1, and again the majority of data needs to be redistributed at exactly the moment your system is already stressed.

**Consistent hashing solves this.**

---

## Consistent Hashing: The Hash Ring

Instead of mapping keys to nodes with modulo, consistent hashing maps both **keys and nodes** onto a circular ring.

### How It Works

**Step 1:** Create a hash ring with a fixed address space (e.g., 0 to 2³²-1). This ring is conceptual — you never allocate all positions.

**Step 2:** Place each node on the ring by hashing the node's identifier (e.g., IP address):

```
hash("node-A") → position 20
hash("node-B") → position 50
hash("node-C") → position 80
hash("node-D") → position 10  (wraps around from 100)

Ring positions (0–100 range for illustration):
  0 ──── 10(D) ──── 20(A) ──── 50(B) ──── 80(C) ──── 100(≡0)
```

**Step 3:** To find which node owns a key, hash the key and walk clockwise until you hit a node:

```
hash("user:42") → position 35 → walk clockwise → hit Node B at 50
hash("user:99") → position 65 → walk clockwise → hit Node C at 80
hash("user:12") → position 95 → walk clockwise → wraps to Node D at 10
```

```
         0
         │
   D(10) ●───────────● A(20)
  ╱                       ╲
●                           ●  user:12 (95) → D
C(80)                      user:42 (35) → B
  ╲                       ╱
   ●───────────────────● B(50)
         user:99 (65) → C
```

### Adding a Node

Add Node E at position 60:

```
Before: user:99 at 65 → Node C at 80
After:  user:99 at 65 → Node E at 60  (E is now closer clockwise)
```

**Only keys between the previous node (50=B) and the new node (60=E) need to move.** Everything else is unaffected. For N nodes, adding one node moves **1/N** of the keys — not most of them.

### Removing a Node

Remove Node B at position 50:

```
Before: user:42 at 35 → Node B at 50
After:  user:42 at 35 → Node C at 80  (next clockwise node)
```

Only keys that were assigned to the removed node need to move to its successor. Again, roughly **1/N** of total keys.

---

## The Problem: Uneven Distribution

With a small number of physical nodes, their positions on the ring are unlikely to be perfectly spaced. Node A might get 40% of the ring, Node B 10%, Node C 30%, Node D 20%.

```
  0 ──── 5(D) ──────────── 35(A) ─── 45(B) ──────────── 75(C) ──── 100
         5% of ring   30% of ring  10% of ring      30% of ring  25% of ring
```

This creates hotspots — some nodes handle far more traffic and data than others.

---

## Virtual Nodes (Vnodes)

**Virtual nodes** solve uneven distribution. Instead of placing each physical node once on the ring, you place it many times under different identifiers:

```
Physical node A → virtual nodes: "A#1", "A#2", "A#3", "A#4" → 4 positions on ring
Physical node B → virtual nodes: "B#1", "B#2", "B#3", "B#4" → 4 positions on ring
```

With enough virtual nodes (typically 100–200 per physical node), the positions are statistically well-spread and each node handles approximately equal share of the keyspace.

```
Ring with 3 physical nodes, 4 vnodes each (12 positions total):
  A#2  B#1  C#3  A#4  B#2  C#1  A#1  B#4  C#2  A#3  B#3  C#4
   │    │    │    │    │    │    │    │    │    │    │    │
   0   10   20   30   40   50   60   70   80   85   90   95
```

Keys in range 0–10 → A#2 → Physical Node A  
Keys in range 10–20 → B#1 → Physical Node B  
… and so on.

### Benefits of Virtual Nodes

1. **Even distribution** — large numbers of random positions average out
2. **Graceful adds/removes** — when removing a node, its virtual positions are spread across many successors, not all piling onto one neighbor
3. **Heterogeneous capacity** — give a more powerful node 2× virtual nodes → it handles 2× the traffic

---

## Consistent Hashing in the Wild

| System | How It Uses Consistent Hashing |
|--------|-------------------------------|
| **Amazon DynamoDB** | Data partitioned across nodes using virtual nodes; each node manages multiple token ranges |
| **Apache Cassandra** | Every node owns one or more token ranges on the ring; vnodes default to 256 per node |
| **Redis Cluster** | 16,384 hash slots distributed across nodes (a variant — slot-based consistent hashing) |
| **Chord DHT** | Academic predecessor; used finger tables for O(log N) lookup |
| **Nginx (consistent_hash)** | Load balance upstream requests to sticky backends |
| **Content Delivery** | Route requests for a URL consistently to the same edge node (cache affinity) |

---

## Replication Factor on the Ring

In production systems, data is not stored on just one node — it's replicated to N nodes for fault tolerance. With consistent hashing, the **replication factor** is implemented by walking clockwise to the next N-1 nodes after the primary:

```
Ring: A(10) → B(30) → C(55) → D(72) → A(10) ...
Replication factor = 3

key "user:42" hashes to position 25 → primary node B
  Replica 1 → next clockwise node: C
  Replica 2 → next clockwise node: D

Data for "user:42" lives on B, C, D
```

**Quorum reads/writes** use this to tune consistency vs availability:

```
N = replication factor (e.g., 3)
W = nodes that must ack a write
R = nodes that must respond to a read

Strong consistency guarantee: W + R > N
  Example: W=2, R=2, N=3 → 2+2=4 > 3 ✅ (overlapping quorum guarantees latest)

High availability, eventual consistency: W=1, R=1
  Faster but no guarantee of reading the latest write

Cassandra defaults: W=1, R=1 (eventual)
Cassandra QUORUM:   W=2, R=2 for N=3 (strong)
```

This is the *mechanism* behind "tunable consistency" in Cassandra and DynamoDB — you're adjusting W and R relative to N.

---

Naively, finding the successor node on the ring requires scanning all positions: O(N).

Real implementations maintain a **sorted array or binary search tree** of positions, enabling **O(log N)** lookup. For 200 vnodes × 100 physical nodes = 20,000 positions, log₂(20,000) ≈ 14 operations — negligible.

---

## Consistent Hashing vs Simple Hash Sharding

| Property | Simple Modulo | Consistent Hashing |
|----------|-------------|-------------------|
| Data moved when adding node | ~(N-1)/N of all keys | ~1/N of all keys |
| Data moved when removing node | ~(N-1)/N of all keys | ~1/N of all keys |
| Even distribution | Good (by design) | Good with vnodes |
| Complexity | Very low | Low–Medium |
| Hotspot resilience | None | Good with vnodes |

---

## When to Mention Consistent Hashing in Interviews

Bring it up whenever you discuss:

1. **Distributed cache clusters** — "I'd use Redis Cluster, which internally uses consistent hashing (16,384 slots) to distribute keys across nodes. Adding nodes causes minimal key movement."

2. **Database sharding** — "For hash-based sharding, I'd use consistent hashing so adding a shard only moves 1/N of the data instead of requiring a full reshuffle."

3. **Load balancing with session affinity** — "Consistent hashing on the session ID ensures the same user always routes to the same application server, maintaining their in-memory state."

4. **CDN routing** — "CDN edge node selection uses consistent hashing on the URL so the same asset is always cached on the same edge, improving cache efficiency."

---

## Implementation Sketch

```python
import hashlib
from bisect import bisect, insort

class ConsistentHashRing:
    def __init__(self, nodes=None, vnodes=150):
        self.vnodes = vnodes
        self.ring = {}      # position → node
        self.sorted_keys = []

        for node in (nodes or []):
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        for i in range(self.vnodes):
            pos = self._hash(f"{node}#{i}")
            self.ring[pos] = node
            insort(self.sorted_keys, pos)

    def remove_node(self, node: str):
        for i in range(self.vnodes):
            pos = self._hash(f"{node}#{i}")
            del self.ring[pos]
            self.sorted_keys.remove(pos)

    def get_node(self, key: str) -> str:
        if not self.ring:
            return None
        pos = self._hash(key)
        idx = bisect(self.sorted_keys, pos) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]
```

---

## Key Takeaways

1. Simple modulo sharding breaks when you resize — consistent hashing limits disruption to 1/N of keys.
2. Virtual nodes solve uneven distribution and enable heterogeneous node capacity.
3. Cassandra, DynamoDB, and Redis Cluster all use variants of consistent hashing in production.
4. In interviews, mention consistent hashing when discussing distributed caches or hash-sharded databases.

---

*Previous: [05 - Sharding](./05-sharding.md)*  
*Next: [07 - CAP Theorem](./07-cap-theorem.md)*
