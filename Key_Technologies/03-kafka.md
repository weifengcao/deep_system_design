# Kafka Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Architecture, topics/partitions/offsets, producers/consumers, delivery guarantees, stream processing, when to use vs alternatives

---

## Why Kafka

Apache Kafka is a **distributed commit log** designed for high-throughput, durable, ordered event streaming. Used by 80%+ of Fortune 100 companies, it is the backbone of modern event-driven architectures.

**Use Kafka when:**
- You need to decouple producers from consumers at high throughput (millions of events/sec)
- Multiple independent consumers need to read the same stream
- You need replay — consumers can re-read historical events
- Events must be processed in order within a logical partition
- You need durable, fault-tolerant message delivery

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Kafka Cluster                           │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│  │  Broker 1  │   │  Broker 2  │   │  Broker 3  │           │
│  │            │   │            │   │            │           │
│  │ Topic A    │   │ Topic A    │   │ Topic A    │           │
│  │  P0 (lead) │   │  P1 (lead) │   │  P2 (lead) │           │
│  │  P1 (rep)  │   │  P2 (rep)  │   │  P0 (rep)  │           │
│  └────────────┘   └────────────┘   └────────────┘           │
│                    ↑ ZooKeeper / KRaft manages metadata      │
└─────────────────────────────────────────────────────────────┘
        ▲ produce                          ▼ consume
  ┌─────────────┐                  ┌─────────────────────┐
  │  Producers  │                  │  Consumer Groups    │
  └─────────────┘                  └─────────────────────┘
```

### Topics

A **topic** is a named, append-only log of events. Think of it as a category or channel. Topics are durable — messages are retained even after consumption (configurable retention period or size).

### Partitions

Each topic is split into **partitions**. A partition is an ordered, immutable sequence of records stored on disk. This is the fundamental unit of parallelism and ordering.

```
Topic "orders" with 3 partitions:

Partition 0: [msg0] [msg3] [msg6] [msg9] ...   (offset 0, 1, 2, 3...)
Partition 1: [msg1] [msg4] [msg7] [msg10] ...
Partition 2: [msg2] [msg5] [msg8] [msg11] ...
```

**Key properties:**
- Ordering is guaranteed only **within a partition**
- More partitions = more parallelism (but more overhead)
- Partition count is fixed at topic creation (can increase, hard to decrease)

### Offsets

Every message in a partition has a monotonically increasing **offset** — its position in the partition log. Consumers track their progress using offsets. This enables:
- **Replay:** reset consumer offset to any past position
- **Independent consumers:** each consumer group tracks its own offset
- **At-least-once recovery:** restart from last committed offset after failure

### Brokers

Kafka brokers are servers that store partition replicas. Each partition has one **leader** (handles all reads/writes) and N-1 **followers** (replicate from leader). If the leader fails, one follower is elected as the new leader.

### KRaft (Replacing ZooKeeper)

Kafka 3.3+ ships with KRaft — Kafka's own Raft-based metadata consensus, replacing ZooKeeper. Simpler ops, faster startup, eliminates a dependency. In interviews, mention ZooKeeper for context but note KRaft is the modern default.

---

## Producers

### Partition Assignment

When a producer sends a message:
1. **With key:** `partition = hash(key) % num_partitions`. Same key always goes to same partition → ordering guaranteed for that key.
2. **No key:** Round-robin or sticky partitioner (batch to same partition for efficiency, rotate periodically).

**Choosing the right key is critical:**
```
# Order events: key = order_id → all events for an order stay ordered
producer.send("orders", key=order_id, value=event_json)

# User activity: key = user_id → all events for a user stay ordered  
producer.send("activity", key=user_id, value=activity_json)

# No ordering needed: no key → even distribution
producer.send("metrics", value=metric_json)
```

### Delivery Acknowledgment (acks)

```
acks=0  → Fire and forget. No confirmation. Possible data loss. Max throughput.
acks=1  → Leader confirms write. Follower lag = potential loss on leader failure.
acks=-1 → All in-sync replicas (ISR) confirm. No data loss. Lower throughput.
```

**Interview default:** `acks=-1` with `min.insync.replicas=2` for durability. Accept the throughput trade-off.

### Idempotent Producer
Enable with `enable.idempotence=true`. Kafka assigns each producer a PID and sequence number. Duplicate retries are deduplicated at the broker. Required for exactly-once semantics.

### Batching and Compression

Producers batch messages before sending to reduce network overhead:
- `linger.ms=5` — wait up to 5ms to accumulate a batch
- `batch.size=16384` — max batch size in bytes
- `compression.type=snappy` — compress batches (typically 4–5× compression)

---

## Consumers and Consumer Groups

### Consumer Groups

Each consumer in a group reads from a **disjoint set of partitions**. This is the mechanism for parallel consumption.

```
Topic: 6 partitions
Consumer Group A (2 consumers):
  consumer-a1 → reads P0, P1, P2
  consumer-a2 → reads P3, P4, P5

Consumer Group B (6 consumers):
  consumer-b0 → reads P0
  consumer-b1 → reads P1
  ...          (one consumer per partition — max parallelism)

Consumer Group B (7 consumers):
  consumer-b0 → reads P0
  ...
  consumer-b5 → reads P5
  consumer-b6 → IDLE (no partitions to assign — can't exceed partition count)
```

**Rule:** Max useful consumers per group = number of partitions.

### Offset Management

Consumers commit offsets back to Kafka (stored in `__consumer_offsets` topic):

```python
# Auto-commit (simple but risk of data loss or duplication)
consumer = KafkaConsumer(enable_auto_commit=True, auto_commit_interval_ms=5000)

# Manual commit (after processing — safer)
for msg in consumer:
    process(msg)
    consumer.commit()  # commit after successful processing
```

**At-least-once vs exactly-once:**
- Default (at-least-once): commit offset after processing. If crash between process and commit → message reprocessed.
- Exactly-once: use Kafka transactions (`transactional.id`) + idempotent consumers (check if already processed).

### Rebalancing

When a consumer joins or leaves a group, Kafka **rebalances** partition assignments. During rebalance, all consumers in the group pause. Minimize rebalance impact with:
- `session.timeout.ms` — how long before a consumer is considered dead
- `max.poll.interval.ms` — max time between poll calls before considered dead
- Cooperative/incremental rebalancing (Kafka 2.4+) — only revoke necessary partitions, avoids full stop-the-world rebalance

---

## Delivery Guarantees

| Guarantee | How | Trade-off |
|-----------|-----|----------|
| At-most-once | Commit offset before processing | Fast, possible data loss |
| At-least-once | Commit offset after processing | Some duplicates, idempotent consumers needed |
| Exactly-once | Idempotent producer + transactions | Complex, lower throughput |

**Interview default:** propose at-least-once with idempotent consumers (check deduplication key in DB before acting). Exactly-once is rarely needed and complex to implement.

---

## Retention and Compaction

### Time/Size Retention
Default: retain messages for 7 days or until disk limit. Good for event streams.

### Log Compaction
For topics where only the latest value per key matters (like a changelog or state store), enable log compaction. Kafka retains only the most recent message per key.

```
Before compaction:  [k1:v1] [k2:v1] [k1:v2] [k3:v1] [k2:v2]
After compaction:   [k1:v2] [k2:v2] [k3:v1]
```

Use for: database CDC streams (keep latest row state), user profile change events.

---

## Stream Processing Patterns

### Kafka Streams
Java library for stateful stream processing on top of Kafka. No separate cluster needed.

```java
KStream<String, Order> orders = builder.stream("orders");
KTable<String, Long> orderCounts = orders
    .groupByKey()
    .count(Materialized.as("order-counts"));
orderCounts.toStream().to("order-count-results");
```

### Kafka + Flink (More Powerful)
For complex CEP (Complex Event Processing), windowed aggregations, joins across streams. Flink reads from Kafka, processes with exactly-once guarantees, writes results back to Kafka or a database.

### Dead Letter Queue (DLQ)
For messages that consistently fail processing:
```python
try:
    process(msg)
    consumer.commit()
except ProcessingError:
    # Send to DLQ topic for manual inspection/retry
    producer.send("orders-dlq", value=msg.value, headers={"error": str(e)})
    consumer.commit()  # don't block the pipeline
```

---

## Kafka vs Alternatives

| Feature | Kafka | Redis Streams | RabbitMQ | AWS SQS/SNS |
|---------|-------|--------------|----------|------------|
| Throughput | ✅ Millions/sec | ✅ High | ⚠️ Moderate | ⚠️ Moderate |
| Replay / rewind | ✅ Configurable retention | ⚠️ Limited | ❌ | ❌ |
| Multiple consumers | ✅ Consumer groups | ✅ | ✅ Exchanges | ✅ SNS fan-out |
| Ordering | ✅ Per-partition | ✅ Per-stream | ⚠️ Per-queue | ⚠️ FIFO queues only |
| Durability | ✅ Replicated, disk | ⚠️ AOF/RDB | ✅ | ✅ |
| Ops complexity | High | Low | Medium | Zero (managed) |
| At-exactly-once | ✅ (complex) | ❌ | ❌ | ❌ |
| Use for | Event streaming, analytics | Lightweight queues | Task queues | Serverless, AWS-native |

---

## Security Considerations

- **Authentication:** SASL/PLAIN, SASL/SCRAM, mTLS for broker-to-broker and client-to-broker
- **Authorization:** Kafka ACLs — control which producers/consumers can read/write which topics
- **Encryption in transit:** TLS between clients and brokers, between brokers
- **Encryption at rest:** Broker disk encryption (OS-level or cloud provider)
- **Audit:** Enable log4j audit logging for access tracking

---

## AI Agent Use Cases

### Streaming Agent Events
```
Agent Step Completed → Kafka topic "agent-events"
  → Consumer Group A: Audit logger → writes to S3
  → Consumer Group B: UI streamer → SSE to browser
  → Consumer Group C: Metrics aggregator → Prometheus
```

### Training Data Pipeline
```
User interactions → Kafka "user-feedback" topic
  → Batch consumer → S3 → Training job reads from S3
```

### Tool Result Streaming
```python
# Agent publishes tool call results for downstream systems
producer.send("tool-results",
    key=run_id,  # all events for one run on same partition → ordered
    value={"tool": "web_search", "result": data, "run_id": run_id}
)
```

---

## Capacity Sizing

```
Throughput needed: 100K messages/sec, average 1KB/message
= 100 MB/sec ingress

With 3× replication: 300 MB/sec disk write throughput needed
NVMe SSD: 1–3 GB/sec → 3 brokers with NVMe handles this easily

Retention: 7 days × 100 MB/sec × 86400 sec/day = ~60 TB
With 3× replication: 180 TB total storage
→ 3 brokers × 60 TB each (consider 6 brokers for headroom)

Partition count: aim for consumer parallelism needed
  If 10 consumer instances → at least 10 partitions
  Rule of thumb: partitions = 2-4× max consumer count
```

---

## Interview Quick Reference

| Scenario | Choice |
|----------|--------|
| Decouple microservices | Kafka topic per event type |
| Fan-out to multiple consumers | One topic, multiple consumer groups |
| Order events for one entity | Key = entity ID |
| Retry failed events | DLQ topic + separate retry consumer |
| Replay historical events | Reset consumer group offset |
| High durability | acks=-1, min.insync.replicas=2 |
| Max throughput | acks=1, linger.ms=5, compression |
| Exactly-once | Idempotent producer + transactional API |
| CDC sync to ES/DB | Debezium → Kafka → consumers |

---

*Previous: [02 - Elasticsearch](./02-elasticsearch.md)*  
*Next: [04 - API Gateway](./04-api-gateway.md)*
