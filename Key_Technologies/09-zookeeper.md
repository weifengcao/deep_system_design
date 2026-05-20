# ZooKeeper Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** ZAB consensus, znodes, watches, use cases, leader election, service discovery, when to use etcd instead

---

## What Is ZooKeeper?

Apache ZooKeeper is a **distributed coordination service** that provides a reliable, ordered, and consistent shared data store for distributed systems. It is CP (consistent + partition-tolerant) — it guarantees linearizable reads and writes but may become unavailable during network partitions.

ZooKeeper is the coordination backbone of Apache Kafka (pre-KRaft), Hadoop YARN, HBase, and many other distributed systems.

**ZooKeeper is not a general-purpose database.** It stores small pieces of coordination metadata — leader identities, configuration, service addresses, distributed locks. Total data size is typically < 1 GB.

---

## Core Data Model: ZNodes

ZooKeeper exposes a **hierarchical namespace** (like a filesystem) of **znodes**:

```
/
├── /services
│   ├── /services/payment-service
│   │   ├── /services/payment-service/instance-1  (data: "10.0.0.1:8080")
│   │   └── /services/payment-service/instance-2  (data: "10.0.0.2:8080")
│   └── /services/order-service
│       └── /services/order-service/instance-1    (data: "10.0.0.3:8080")
├── /config
│   └── /config/feature-flags                     (data: JSON blob)
└── /locks
    └── /locks/payment-processor                   (ephemeral — exists while holder is alive)
```

### ZNode Types

| Type | Persistence | Sequence | Use Case |
|------|------------|---------|---------|
| **Persistent** | Survives client disconnect | No | Config, group membership |
| **Ephemeral** | Deleted when client disconnects | No | Leader identity, presence detection |
| **Persistent Sequential** | Survives disconnect | Yes | Queue ordering |
| **Ephemeral Sequential** | Deleted on disconnect | Yes | Leader election |

**Ephemeral znodes** are the key to distributed coordination: when a service crashes, its session expires and its ephemeral znodes are automatically deleted. Other services watching those znodes are immediately notified.

---

## ZAB: ZooKeeper Atomic Broadcast

ZooKeeper's consistency comes from **ZAB (ZooKeeper Atomic Broadcast)**, a Paxos-like consensus protocol:

```
Cluster roles:
  - Leader: 1 node that handles all writes
  - Followers: remaining nodes that replicate and serve reads

Write path:
  1. Client sends write to any node
  2. If not leader, node forwards to leader
  3. Leader assigns a zxid (monotonic transaction ID)
  4. Leader broadcasts PROPOSAL to all followers
  5. Followers write to disk, send ACK
  6. When quorum (N/2+1) ACKs → leader sends COMMIT
  7. All followers apply, respond to client

Read path:
  Client reads from any node (may be stale by one update)
  For linearizable reads: use sync() before read
```

**Quorum requirement:** With 2N+1 nodes, can tolerate N failures:
- 3 nodes → tolerate 1 failure
- 5 nodes → tolerate 2 failures
- 7 nodes → tolerate 3 failures

Always deploy ZooKeeper with **odd number of nodes** (3 or 5 for production).

---

## Watches: Event-Driven Coordination

A **watch** is a one-time trigger: a client registers interest in a znode and is notified when it changes:

```python
# Register a watch on a znode
def leader_changed(event):
    print(f"Leader changed: {event}")
    new_leader = zk.get("/leader", watch=leader_changed)  # re-register

current_leader, stat = zk.get("/leader", watch=leader_changed)
```

Watch semantics:
- Watches fire **once** — client must re-register after each notification
- Watches are **ordered** — guaranteed to fire before the next change is visible
- Watches fire on: data change, child change, node creation/deletion

---

## Core Use Cases

### 1. Leader Election

The canonical ZooKeeper pattern using ephemeral sequential znodes:

```
All candidate nodes create: /election/leader-{seq}
  Node A creates: /election/leader-0000000001
  Node B creates: /election/leader-0000000002
  Node C creates: /election/leader-0000000003

Leader = node with the lowest sequence number = Node A

If Node A dies:
  /election/leader-0000000001 is deleted (ephemeral)
  Node B watches the node just below it in sequence
  Node B is notified → becomes leader

The "watch the next lower sibling" pattern prevents herd effect:
  All nodes watch their predecessor, not all watch the leader
  Only one node wakes up on each leader failure
```

This is used by: Kafka broker leader election, HBase master election, Hadoop ResourceManager.

### 2. Distributed Lock

Using ephemeral znodes:

```python
import kazoo.client

zk = kazoo.client.KazooClient(hosts='zk1:2181,zk2:2181,zk3:2181')
zk.start()

# Try to acquire lock
lock = zk.Lock("/locks/payment-processor", identifier="worker-1")
with lock:
    # Only one process in this block across entire distributed system
    process_payments()
# Lock automatically released when block exits (or on crash — ephemeral)
```

Unlike Redis-based locks, ZooKeeper locks are safe against lock holder crashes — the ephemeral node disappears when the session times out.

### 3. Service Discovery / Registry

```python
# Service registers itself on startup
service_znode = zk.create(
    "/services/payment/instance-",
    value=b"10.0.0.5:8080",
    ephemeral=True,
    sequence=True
)  # Creates: /services/payment/instance-0000000003

# Client discovers all instances
def update_service_list(event=None):
    instances = zk.get_children("/services/payment", watch=update_service_list)
    addresses = [zk.get(f"/services/payment/{i}")[0].decode() for i in instances]
    load_balancer.update(addresses)

update_service_list()
```

When a service instance dies, its ephemeral znode is deleted → watchers are notified → load balancer updates automatically.

### 4. Configuration Management

```python
# Application reads config at startup
config_data, stat = zk.get("/config/app-settings")
config = json.loads(config_data)

# Watch for config changes (hot reload without restart)
def on_config_change(event):
    new_config_data, _ = zk.get("/config/app-settings", watch=on_config_change)
    apply_config(json.loads(new_config_data))

zk.get("/config/app-settings", watch=on_config_change)
```

### 5. Distributed Barrier / Coordination

```python
# All workers must reach a checkpoint before any proceed
# Worker joins barrier
zk.create("/barrier/worker-1", ephemeral=True)

# Worker waits until all N workers have joined
while len(zk.get_children("/barrier")) < N_WORKERS:
    time.sleep(0.1)

# Proceed — all workers past the barrier
```

---

## ZooKeeper vs etcd vs Consul

At staff level, know when to use each:

| | ZooKeeper | etcd | Consul |
|--|-----------|------|--------|
| **Consensus** | ZAB (Paxos-like) | Raft | Raft |
| **Data model** | Hierarchical (znodes) | Flat key-value | Key-value + services |
| **Watches** | One-shot | Watch streams (gRPC) | Long-poll / streaming |
| **Performance** | High write throughput | High | Good |
| **Kubernetes native** | ❌ | ✅ (K8s uses etcd) | ✅ |
| **Service mesh** | ❌ | ❌ | ✅ (Consul Connect) |
| **DNS-based discovery** | ❌ | ❌ | ✅ |
| **Health checking** | ❌ (manual via ephemeral) | ❌ | ✅ Built-in |
| **Ops complexity** | High | Medium | Medium |
| **Best for** | Kafka, Hadoop ecosystem | Kubernetes, cloud-native | HashiCorp stack, service mesh |

**Modern recommendation:**
- New greenfield project: **etcd** (simpler, Raft, gRPC API) or **Consul** (if you want service mesh)
- Existing Kafka/HBase/Hadoop: ZooKeeper (Kafka 3.3+ uses KRaft; migration happening gradually)
- Kubernetes components: etcd (it's already there)

---

## Operational Considerations

### Session Timeout
If a ZooKeeper client loses connectivity for longer than its session timeout (default 30s), its session expires and all ephemeral znodes are deleted. Configure thoughtfully:
- Too short: transient network hiccup → unnecessary failover
- Too long: real failure goes undetected for too long

### ZooKeeper Is Not a Database
- **Max znode data size: 1 MB** (configurable, but small by design)
- **Max total data: a few GB** (all in-memory)
- **Not for high write throughput:** ZooKeeper is for coordination events, not data storage

### Ensemble Sizing
- 3 nodes: tolerate 1 failure. Good for most systems.
- 5 nodes: tolerate 2 failures. Use for critical production systems.
- Never use 2 or 4 nodes (even number → can't form majority after split).

---

## Interview Quick Reference

| Scenario | ZooKeeper Pattern |
|----------|-----------------|
| Elect a leader among N services | Ephemeral sequential znodes + watch predecessor |
| Distributed lock | Ephemeral znode + watch (or use Kazoo Lock) |
| Service registry | Ephemeral znodes per instance + watcher |
| Config hot-reload | Persistent znode + watch |
| Detect service crash | Ephemeral znode disappears on session expiry |
| Coordination barrier | Wait for N znodes to appear under a path |

| Question | Answer |
|----------|--------|
| Why odd number of nodes? | Need N/2+1 quorum; even numbers risk split-brain |
| How does ZooKeeper handle leader failure? | ZAB leader election; quorum must agree on new leader |
| ZooKeeper vs etcd? | etcd is simpler, Raft-based, better for cloud-native; ZK for Kafka/Hadoop legacy |
| Why not Redis for distributed lock? | Redis lock requires Redlock (complex); ZK ephemeral znode auto-releases on crash |

---

*Previous: [08 - Apache Flink](./08-flink.md)*  
*Next: [10 - Time Series Databases](./10-time-series-databases.md)*
