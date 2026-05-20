# Apache Flink Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Stream vs batch, Flink architecture, windowing, stateful processing, exactly-once, when to use vs Kafka Streams vs Spark

---

## What Is Apache Flink?

Apache Flink is a **unified stream and batch processing engine** for stateful computations over unbounded (streaming) and bounded (batch) data sets. It provides:

- **True stream processing** (not micro-batching like early Spark Streaming)
- **Stateful computations** with exactly-once guarantees
- **Event-time processing** with out-of-order event handling
- **High throughput and low latency** (millions of events/sec, sub-second)

**Use Flink when:**
- You need complex stateful aggregations over event streams (windowed counts, joins, CEP)
- You require exactly-once semantics for critical business logic
- You're building real-time analytics pipelines (revenue metrics, fraud detection)
- You need to join a stream with another stream or a batch dataset

---

## Stream Processing Fundamentals

### Unbounded vs Bounded Data

```
Unbounded (stream): Events arrive continuously, no end
  [order_1] [order_2] [order_3] → ... → [order_∞]
  Question: "Revenue per hour, updated every minute"

Bounded (batch): Fixed dataset, has a beginning and end
  [orders_jan] [orders_feb] → done
  Question: "Total revenue in January 2025"
```

Flink handles both with the same API. Historically, companies ran separate batch and streaming systems ("Lambda architecture") — Flink enables a unified approach ("Kappa architecture").

### Event Time vs Processing Time

This distinction is critical for correct results:

```
Event time:      When the event actually happened (embedded in event payload)
Processing time: When Flink processes the event (wall clock time)

Problem: Network delay, mobile offline → events arrive out of order
  Event at 10:00:01 arrives at Flink at 10:00:05 (4s late)
  Processing-time window: event lands in 10:00–10:01 window ✓ (correct)
  Event-time window:      event belongs to 10:00–10:01 window ✓ (also correct)

BUT:
  Event at 09:59:55 arrives at Flink at 10:00:03 (8s late, crossed minute boundary)
  Processing-time window: wrongly places it in 10:00–10:01 window ✗
  Event-time window:      correctly places it in 09:59–10:00 window ✓
```

**Always use event time** for business metrics. Use watermarks to handle late arrivals.

### Watermarks

A **watermark** is a timestamp that asserts "all events with event_time ≤ watermark have been received." This tells Flink when it's safe to close a window and emit results.

```
Events arrive: [t=10:00:01] [t=10:00:03] [t=09:59:55 — LATE!] [t=10:00:07]
Watermark: max_seen_event_time - 10s = allowable lateness

When watermark passes window end time → window fires and results are emitted
Late events after watermark: handled by late data side output or discarded
```

---

## Flink Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Flink Cluster                             │
│                                                                  │
│  ┌──────────────────┐                                           │
│  │   JobManager      │  ← Receives job, schedules tasks,        │
│  │   (coordinator)   │    manages checkpoints, handles failures  │
│  └────────┬─────────┘                                           │
│           │ distribute tasks                                     │
│  ┌────────┴─────────────────────────────────────────────┐      │
│  │                  TaskManagers                         │      │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │      │
│  │  │  TM 1    │  │  TM 2    │  │  TM 3    │           │      │
│  │  │ [Task]   │  │ [Task]   │  │ [Task]   │           │      │
│  │  │ [Task]   │  │ [Task]   │  │ [Task]   │           │      │
│  │  │ [State]  │  │ [State]  │  │ [State]  │           │      │
│  │  └──────────┘  └──────────┘  └──────────┘           │      │
│  └──────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
        ▲ read                                    ▼ write
   Kafka / Kinesis                         Kafka / DB / S3
```

- **JobManager:** Coordinates the distributed execution. Manages checkpointing. In HA mode, backed by ZooKeeper/Kubernetes for leader election.
- **TaskManagers:** Execute tasks. Each has a fixed number of task slots (typically #cores). Tasks run in parallel across task slots.
- **State backend:** Where operator state is stored. Options: in-heap (fast, limited), RocksDB (large state, spills to disk).

---

## Core Abstractions

### DataStream API

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Order> orders = env
    .addSource(new FlinkKafkaConsumer<>("orders", schema, props))
    .assignTimestampsAndWatermarks(
        WatermarkStrategy.<Order>forBoundedOutOfOrderness(Duration.ofSeconds(10))
                         .withTimestampAssigner((e, ts) -> e.getEventTime())
    );

DataStream<RevenueResult> revenue = orders
    .keyBy(order -> order.getProductCategory())    // partition by key
    .window(TumblingEventTimeWindows.of(Time.hours(1)))  // 1-hour window
    .aggregate(new RevenueAggregator());           // stateful aggregation

revenue.addSink(new FlinkKafkaProducer<>("revenue-results", schema, props));

env.execute("Revenue Per Category Per Hour");
```

### Windowing

Windows group events for bounded computation:

```
Tumbling window:   [0-1h] [1-2h] [2-3h]    No overlap, fixed size
Sliding window:    [0-1h] [0.5-1.5h] [1-2h] Overlap, fixed size, fixed slide
Session window:    [events within 30min gap] Variable size, gap-based
Global window:     All events in one window  Manual trigger needed
```

**Example — sliding window (real-time fraud detection):**
```java
transactions
    .keyBy(tx -> tx.getCardId())
    .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
    .aggregate(new SpendingAggregator())
    .filter(result -> result.getTotalSpend() > FRAUD_THRESHOLD);
```

### Stateful Operations

Flink operators can maintain state across events. State is partitioned by key:

```java
public class FraudDetectionFunction extends KeyedProcessFunction<String, Transaction, Alert> {
    private ValueState<Double> rollingSum;
    private ValueState<Long> windowStart;

    @Override
    public void processElement(Transaction tx, Context ctx, Collector<Alert> out) {
        Double sum = rollingSum.value() == null ? 0.0 : rollingSum.value();
        sum += tx.getAmount();
        rollingSum.update(sum);

        if (sum > 10000.0) {
            out.collect(new Alert(tx.getCardId(), sum));
        }

        // Register a timer to clear state after 1 hour
        ctx.timerService().registerEventTimeTimer(tx.getEventTime() + 3600_000);
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Alert> out) {
        rollingSum.clear();  // Reset window
    }
}
```

State types: `ValueState`, `ListState`, `MapState`, `ReducingState`, `AggregatingState`.

### Stream-Stream Joins

Join two streams within a time window:

```java
// Join orders with payments within 5-minute window
orders
    .join(payments)
    .where(order -> order.getOrderId())
    .equalTo(payment -> payment.getOrderId())
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .apply((order, payment) -> new FulfilledOrder(order, payment));
```

---

## Exactly-Once Semantics

Flink achieves exactly-once end-to-end via **distributed snapshots (Chandy-Lamport algorithm)**:

```
1. JobManager triggers a checkpoint
2. Injects "barrier" marker into all source streams
3. Barrier flows through the DAG like a normal record
4. Each operator:
   a. Receives barriers from all upstream inputs (barrier alignment)
   b. Snapshots its current state to durable storage (S3/HDFS)
   c. Forwards barrier downstream
5. When sink receives barrier: it commits (e.g., Kafka transaction commit)
6. JobManager records checkpoint as complete

On failure:
  Restore all operator states from last checkpoint
  Replay source from checkpoint offset
  Result: each record processed exactly once end-to-end
```

**Requirements for exactly-once:**
- Source must be replayable (Kafka ✅, random HTTP call ❌)
- Sink must support transactions or idempotency (Kafka ✅, most DBs ✅, HTTP API ❌)

---

## Flink vs Kafka Streams vs Spark Streaming

| Dimension | Flink | Kafka Streams | Spark Streaming |
|-----------|-------|--------------|----------------|
| Processing model | True streaming | True streaming | Micro-batch |
| Latency | Sub-second | Sub-second | Seconds |
| State management | ✅ Excellent (RocksDB) | ✅ Good (local KV) | ⚠️ Limited |
| Exactly-once | ✅ End-to-end | ✅ Within Kafka | ✅ With config |
| Cluster needed | ✅ Yes | ❌ Runs in app | ✅ Spark cluster |
| Cross-stream joins | ✅ | ⚠️ Limited | ✅ |
| Batch + streaming | ✅ Unified | ❌ Streaming only | ✅ |
| Learning curve | High | Low | Medium |
| Use when | Complex CEP, large state | Kafka-only pipeline | Spark ecosystem |

**Interview guidance:**
- Simple Kafka → transform → Kafka: Kafka Streams (zero extra infra)
- Complex stateful, cross-stream joins, CEP: Flink
- Batch analytics, already on Spark: Spark Structured Streaming

---

## Common Flink Use Cases in Interviews

### Real-Time Metrics Dashboard
```
Kafka (events) → Flink (windowed aggregations per minute) → Redis/Kafka → Dashboard
```

### Fraud Detection
```
Kafka (transactions) → Flink (stateful pattern detection) → Kafka (alerts) → Notification service
Pattern: 3 transactions in 1 minute from different countries
```

### ETL / Change Data Capture
```
Debezium (DB changes) → Kafka → Flink (transform, enrich, filter) → S3 / Data warehouse
```

### Real-Time Recommendation
```
Kafka (user clicks) → Flink (sliding window feature extraction) → Feature store → ML model
```

---

## Interview Quick Reference

| Question | Answer |
|----------|--------|
| Flink vs Kafka Streams? | Flink for complex stateful/cross-stream; Kafka Streams for simple Kafka pipelines |
| How does Flink achieve exactly-once? | Distributed snapshots (barriers) + transactional sinks |
| Event time vs processing time? | Always use event time for correct business metrics; processing time is simpler but wrong for late data |
| What's a watermark? | Assertion that all events up to time T have arrived; triggers window computation |
| How does Flink handle large state? | RocksDB state backend (disk-backed, incremental checkpoints) |
| Where does Flink state live? | TaskManager local storage; snapshotted to S3/HDFS on checkpoint |

---

*Previous: [07 - PostgreSQL](./07-postgresql.md)*  
*Next: [09 - ZooKeeper](./09-zookeeper.md)*
