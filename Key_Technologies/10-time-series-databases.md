# Time Series Databases for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** What makes time-series special, storage engines, InfluxDB, TimescaleDB, ClickHouse, Prometheus, design patterns, AI/ML use cases

---

## What Is a Time Series?

A **time series** is a sequence of data points indexed by time, where:
- Each point has a **timestamp**, one or more **measurements** (values), and optional **tags/labels** (metadata)
- Data arrives in roughly chronological order
- Queries are almost always time-bounded ("last hour", "yesterday", "past 30 days")
- Recent data is queried far more often than old data
- Data is append-only — historical records are never updated

```
Metric: cpu_usage
  timestamp=2025-01-01T10:00:00Z, host=web-01, region=us-east, value=72.3
  timestamp=2025-01-01T10:00:10Z, host=web-01, region=us-east, value=74.1
  timestamp=2025-01-01T10:00:10Z, host=web-02, region=us-east, value=31.2
  timestamp=2025-01-01T10:00:20Z, host=web-01, region=us-east, value=76.8
```

---

## Why Not Just Use PostgreSQL?

Postgres (and most relational databases) handle time-series poorly at scale:

| Problem | Why It Happens | TSDB Solution |
|---------|---------------|--------------|
| Write throughput | B-Tree insert random I/O | LSM-Tree / columnar append |
| Storage size | 8-byte float stored with full row overhead | Delta encoding, compression (10–100× smaller) |
| Query speed | Sequential scan for range queries on unpartitioned tables | Pre-partitioned by time, columnar layout for vectorized aggregation |
| Downsampling | No built-in aggregation policies | Continuous aggregates, rollup jobs |
| Data retention | No built-in TTL or tier | Automatic TTL, tiered storage (hot/warm/cold) |

**At small scale (< 1M points/day):** Postgres with TimescaleDB or BRIN index is fine.  
**At large scale (> 100M points/day, multi-TB):** Dedicated TSDB.

---

## Core TSDB Concepts

### Columnar Storage

TSDBs store data column-by-column, not row-by-row:

```
Row storage:        Columnar storage:
[ts, host, val]     timestamps: [t1, t2, t3, t4, t5...]
[t1, h1, 72.3]      hosts:      [h1, h1, h2, h1, h2...]
[t2, h1, 74.1]      values:     [72.3, 74.1, 31.2, 76.8, 33.1...]
[t3, h2, 31.2]
```

For `SELECT avg(value) WHERE timestamp BETWEEN t1 AND t100`, columnar reads only the value column — massively less I/O.

### Delta + Run-Length Encoding

Time-series values are often monotonically increasing or slowly varying. This enables extreme compression:

```
Timestamps: [1000000, 1000010, 1000020, 1000030]
→ Delta encode: [1000000, +10, +10, +10]
→ Run-length: [1000000, (+10 × 3)]

Values: [72.3, 72.4, 72.3, 72.5, 72.3]
→ XOR float encoding + Gorilla compression (Facebook's algorithm): ~1.5 bytes/value vs 8 bytes raw
```

Production TSDBs achieve **10–40× compression** over raw float64 storage.

### Downsampling and Rollups

Raw data (10s intervals) accumulates fast. Queries over months of data at 10s granularity are expensive. TSDBs automatically downsample:

```
Raw:      10s intervals → retain 7 days
1-minute: 1m aggregates  → retain 30 days
1-hour:   1h aggregates  → retain 1 year
1-day:    1d aggregates  → retain forever
```

Query engine automatically selects the appropriate rollup tier for the requested time range.

---

## Key TSDB Products

### InfluxDB

The most widely known purpose-built TSDB. Uses InfluxQL or Flux query language.

**Data model:**
- **Measurement:** Like a table (e.g., `cpu_usage`)
- **Tags:** Indexed string key-value pairs (host, region) — used in WHERE clauses
- **Fields:** Numeric measurements (value, count) — stored but not indexed
- **Time:** Implicit timestamp

```influxql
-- Write
cpu_usage,host=web-01,region=us-east value=72.3 1705651200000000000

-- Query: 5-minute averages for last hour
SELECT mean(value) FROM cpu_usage
WHERE time > now() - 1h AND host='web-01'
GROUP BY time(5m)
```

**InfluxDB 3.0:** Rewrote the storage engine in Apache Arrow / DataFusion. Columnar storage, SQL support, much better compression and query performance.

**Strengths:** Purpose-built, excellent write throughput, native downsampling policies  
**Weaknesses:** Proprietary, cardinality explosion with many tag combinations, InfluxDB Cloud pricing

### TimescaleDB

PostgreSQL extension that turns Postgres into a time-series database via **hypertables** (automatically partitioned tables):

```sql
-- Create a hypertable (automatic time-based partitioning)
SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '1 day');

-- Compression policy (compress chunks older than 7 days)
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- Continuous aggregate (auto-maintained rollup)
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
       host, avg(value) AS avg_val, max(value) AS max_val
FROM metrics
GROUP BY bucket, host;

-- Query (uses continuous aggregate automatically for historical data)
SELECT bucket, avg_val FROM metrics_hourly
WHERE bucket > now() - INTERVAL '30 days' AND host = 'web-01';
```

**Strengths:** Full SQL, Postgres ecosystem (pgvector, PostGIS, pgcrypto), familiar ops, joins with other tables, native compression (10–20×)  
**Weaknesses:** Not as fast as purpose-built TSDBs for extreme write rates (> 10M points/sec)

**Interview recommendation:** Propose TimescaleDB when the team already uses Postgres. Propose InfluxDB or ClickHouse for pure time-series at extreme scale.

### ClickHouse

Columnar OLAP database (not exclusively TSDB but excellent for time-series analytics):

```sql
CREATE TABLE metrics (
    timestamp DateTime,
    host LowCardinality(String),
    metric_name LowCardinality(String),
    value Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (host, metric_name, timestamp);

-- Fast aggregation query
SELECT host, avg(value) AS avg_cpu
FROM metrics
WHERE timestamp >= now() - INTERVAL 1 HOUR
  AND metric_name = 'cpu_usage'
GROUP BY host
ORDER BY avg_cpu DESC;
```

ClickHouse achieves **billions of rows/sec scan throughput** using vectorized SIMD instructions on columnar data. Used by Cloudflare, Uber, ByteDance at petabyte scale.

**Strengths:** Extreme query speed, rich SQL, great compression, horizontal scaling  
**Weaknesses:** Complex writes (batch inserts required), not optimized for single-row lookups, steep learning curve

### Prometheus

Pull-based metrics system with its own TSDB. The standard for infrastructure monitoring:

```
Architecture:
  Applications expose /metrics endpoint (Prometheus format)
  Prometheus scrapes endpoints every 15s
  Stores in local TSDB (2-hour blocks, then compacts)
  Grafana queries Prometheus via PromQL
```

```promql
# CPU usage per instance, 5-minute rate
rate(process_cpu_seconds_total[5m]) * 100

# Alert: any instance > 80% CPU for 5 minutes
ALERT HighCPU
  IF rate(process_cpu_seconds_total[5m]) * 100 > 80
  FOR 5m
```

**Strengths:** Excellent for infra metrics, huge ecosystem, Kubernetes-native (kube-state-metrics, node-exporter), AlertManager  
**Weaknesses:** Short retention (typically 15 days), single-node storage only → use Thanos or Cortex for long-term distributed storage  
**Not for:** Business metrics, high-cardinality labels, non-numeric data

### OpenTSDB / Druid

- **OpenTSDB:** Stores time-series in HBase. Scalable but operationally complex. Legacy at this point.
- **Apache Druid:** Columnar OLAP for real-time analytics on event streams. Fast for pre-aggregated queries; not for raw time-series.

---

## Design Patterns

### Cardinality Management

The #1 operational problem in TSDBs: **cardinality explosion**.

```
Metric: http_requests_total
Tags: {method, status_code, path, user_id}

Good: {method="GET", status_code="200", path="/api/users"} → low cardinality
Bad:  {user_id="42"} → 10M users = 10M unique series → index OOM
```

**Rules:**
- Tags must have bounded cardinality (< 10K unique values per tag)
- Never use user_id, request_id, or any unique identifier as a tag
- Put high-cardinality values in fields (unindexed) or a separate table

### Write Batching

TSDBs are optimized for batched writes, not individual inserts:

```python
# BAD: one write per data point
for metric in metrics:
    tsdb.write(metric)

# GOOD: batch many points per write
batch = []
for metric in metrics:
    batch.append(metric)
    if len(batch) >= 5000:
        tsdb.write_batch(batch)
        batch = []
```

### Tiered Storage

```
Hot tier  (SSD, last 7 days):    full resolution, fast queries
Warm tier (HDD, last 90 days):   1-minute aggregates
Cold tier (S3, years):           1-hour aggregates

Query engine selects appropriate tier based on time range requested
```

---

## Time-Series for AI/ML Systems

### Model Performance Monitoring
```
Metrics:
  model_latency_ms{model="gpt4o", tier="p99"} = 1200
  model_throughput_tps{model="gpt4o"} = 150
  model_error_rate{model="gpt4o", error_type="rate_limit"} = 0.002
  token_cost_usd{model="gpt4o", customer="tenant-42"} = 0.0045
```

### Training Metrics
```
Loss, accuracy, learning rate per step → logged to InfluxDB or W&B
Query: "Show me loss curve for all runs in the last 24 hours"
Alert: "Loss spiked > 2σ from baseline"
```

### Feature Store Time-Series
```
User features updated every 5 minutes:
  user_avg_session_duration_7d = 12.5 min
  user_purchase_count_30d = 3
  user_last_active_hours_ago = 2
```

---

## Interview Quick Reference

| Scenario | Recommendation |
|----------|---------------|
| Infrastructure metrics | Prometheus + Grafana (standard stack) |
| Business metrics at scale | InfluxDB or ClickHouse |
| Time-series + SQL ecosystem | TimescaleDB (PostgreSQL extension) |
| IoT sensor data at petabyte scale | ClickHouse or Cassandra with TWCS |
| Long-term Prometheus retention | Thanos or Cortex on top of Prometheus |

| Question | Answer |
|----------|--------|
| Why not Postgres for metrics? | Write throughput, storage (no compression), slow range aggregations |
| What is downsampling? | Aggregate high-res data into lower-res for long-term storage and fast queries |
| What is cardinality? | Number of unique time series; cardinality explosion is the main TSDB failure mode |
| How do TSDBs compress data? | Delta encoding on timestamps, XOR/Gorilla compression on floats → 10–40× |

---

*Previous: [09 - ZooKeeper](./09-zookeeper.md)*  
*Next: [11 - Data Structures for Big Data](./11-data-structures-big-data.md)*
