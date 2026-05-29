# Numbers to Know for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** latency numbers, storage units, throughput benchmarks, capacity estimation, traffic math, cost of operations

---

## Why Numbers Matter

Back-of-envelope calculations are a required skill in system design interviews. They help you:

- **Validate design choices** ("Do I need to shard?")
- **Size infrastructure** ("How many servers do I need?")
- **Spot bottlenecks** ("This queue will fill up in 30 seconds")
- **Signal staff-level thinking** (you don't hand-wave scale)

You don't need exact numbers. You need order-of-magnitude intuition, and you need to do the math confidently and quickly.

---

## Latency Numbers Every Engineer Should Know

*Jeff Dean's famous table, updated for modern hardware.*

```
Operation                                  Latency     Notes
─────────────────────────────────────────────────────────────────────
L1 cache reference                         0.5 ns
Branch misprediction                       5   ns
L2 cache reference                         7   ns
Mutex lock/unlock                          25  ns
Main memory reference                      100 ns
Compress 1K bytes with Snappy             3   µs
Send 1 KB over 1 Gbps network             10  µs
Read 4K randomly from NVMe SSD            100 µs    (fast modern SSD)
Read 1 MB sequentially from memory        50  µs    (~20 GB/s DDR4)
Round trip within same datacenter         500 µs
Read 1 MB sequentially from NVMe SSD     200 µs    (~5 GB/s NVMe; NOT 1ms — that was 2012 SATA)
Read 1 MB sequentially from HDD          10  ms
Send packet US → EU → US (RTT)           150 ms
Send packet US → Australia → US          320 ms
─────────────────────────────────────────────────────────────────────
```

> ⚠️ **Common mistake:** Many sources (including old Jeff Dean tables) list "1 MB sequential SSD = 1 ms". That was for 2012-era SATA SSDs (~500 MB/s). Modern NVMe drives achieve 3–7 GB/s sequential — 1 MB takes ~150–300 µs. Use 200 µs as your estimate.

### Practical Groupings

| Category | Latency | Examples |
|----------|---------|---------|
| **In-CPU** | 1–100 ns | CPU cache, memory access |
| **In-process** | 0.1–1 µs | System call, mutex |
| **Local SSD** | 0.1–1 ms | NVMe random read |
| **In-datacenter** | 0.5–5 ms | Redis read, network hop |
| **Cross-region** | 50–150 ms | API call to remote region |
| **Internet RTT** | 50–300 ms | User to server |

### Key Ratios to Remember

- Memory is **~100× faster** than SSD for sequential reads
- SSD is **~10× faster** than spinning disk
- Same-datacenter network is **~500× faster** than cross-continent
- Redis (~1ms) is **~50× faster** than PostgreSQL (~50ms)

---

## Storage Units and Powers of Two

```
Unit        Value              Approximate
──────────────────────────────────────────
1 Byte      8 bits
1 KB        1,024 bytes        ~1 thousand
1 MB        1,048,576 bytes    ~1 million
1 GB        2^30 bytes         ~1 billion
1 TB        2^40 bytes         ~1 trillion
1 PB        2^50 bytes         ~1 quadrillion
1 EB        2^60 bytes
```

### Practical Storage Sizes

| Data Type | Size |
|-----------|------|
| ASCII character | 1 byte |
| UTF-8 character (Latin) | 1–2 bytes |
| Unicode emoji | 4 bytes |
| UUID | 16 bytes (binary) / 36 bytes (string) |
| Integer (int32) | 4 bytes |
| Integer (int64 / bigint) | 8 bytes |
| Float (double) | 8 bytes |
| IP address (IPv4) | 4 bytes |
| IP address (IPv6) | 16 bytes |
| Timestamp (Unix epoch) | 8 bytes |
| Typical tweet | ~300 bytes |
| Typical user record | ~500 bytes–1 KB |
| Medium image (compressed) | 100 KB–1 MB |
| 1-minute video (1080p) | ~150 MB |
| 1-hour video (1080p) | ~1.5–8 GB |
| LLM embedding (1536 dims, float32) | 6 KB |

---

## Throughput Benchmarks

### Database (Single Instance, No Caching)

```
Operation           Throughput (p50)    p50 latency    p99 latency (under load)
─────────────────────────────────────────────────────────────────────────────
PostgreSQL reads    10K–50K QPS         1–10 ms        50–200 ms
PostgreSQL writes   5K–20K QPS          2–20 ms        100–500 ms
MySQL reads         15K–60K QPS         1–5 ms         30–150 ms
Redis reads         100K–1M QPS         0.1–1 ms       2–5 ms
Redis writes        80K–500K QPS        0.1–1 ms       2–5 ms
Cassandra writes    50K–500K QPS        1–5 ms         20–50 ms
Kafka throughput    1M–10M msgs/sec     —              —
```

> ⚠️ **p99 matters.** Staff engineers think in percentiles, not averages. DB p99 latency under load is typically **5–20× p50**. A "50ms average" database that spikes to 500ms at p99 creates timeouts for 1% of users — at 100K QPS that's 1,000 bad requests per second. Always ask: "what's the p99?"

### Memory Bandwidth (Relevant for LLM Inference)

```
Hardware                Memory Bandwidth    Notes
────────────────────────────────────────────────────────────────
DDR4 (server, 8-channel) ~200 GB/s         CPU server main memory
HBM2 (A100 GPU)          ~2,000 GB/s       Why GPUs dominate LLM inference
HBM3 (H100 GPU)          ~3,350 GB/s
LPDDR5 (Apple M2 Ultra)  ~800 GB/s         Why Apple Silicon is efficient for inference
```

LLM token generation is **memory-bandwidth bound**, not compute-bound (during the autoregressive decode phase). A 7B parameter model in float16 = 14 GB. At H100's 3.35 TB/s bandwidth, you can stream the weights ~240 times per second → theoretical ceiling ~240 tokens/sec per H100. Real throughput is lower due to attention computation and batching overhead.

### Network

```
Connection              Throughput
──────────────────────────────────────
Single TCP connection   10–100 MB/s (depends on distance & window)
1 Gbps NIC              ~125 MB/s
10 Gbps NIC             ~1.2 GB/s
25 Gbps NIC             ~3 GB/s
```

### Compute (Single Core)

```
Operation                   Throughput
───────────────────────────────────────────
JSON serialize/deserialize  1M–5M ops/sec
Hash (SHA-256, 1KB input)   500K ops/sec
AES encrypt (hardware AES)  1–5 GB/sec
HTTP request parsing        100K–500K req/sec
gRPC protobuf encode        1M–5M msgs/sec
```

---

## Traffic Calculations

### Daily Active Users → QPS

The standard formula to convert DAU to requests per second:

```
QPS = DAU × requests_per_user_per_day / 86,400 seconds

Peak QPS ≈ 2–5× average QPS (rule of thumb for bursty traffic)

Example:
  DAU = 10 million
  User opens app 5 times/day, makes 3 requests per open
  requests_per_user/day = 15
  Average QPS = 10M × 15 / 86,400 ≈ 1,736 QPS
  Peak QPS ≈ 5,000–8,000 QPS
```

### Quick Reference Table

| DAU | Avg QPS (5 req/day) | Avg QPS (50 req/day) | Peak (5×) |
|-----|--------------------|--------------------|-----------|
| 100K | 6 | 58 | 30–290 |
| 1M | 58 | 579 | 290–2,900 |
| 10M | 579 | 5,787 | 2,900–28,000 |
| 100M | 5,787 | 57,870 | 29K–290K |
| 1B | 57,870 | 578,703 | 290K–2.9M |

### Storage Growth

```
Storage per year = write_rate × record_size × 86,400 × 365

Example: Twitter-scale posts
  10M posts/day
  300 bytes/post
  Storage/year = 10M × 300 × 365 = 1.1 TB/year
  5-year retention = 5.5 TB
  (Add 2× for indexes, replicas: ~11 TB)
```

---

## Capacity Estimation Template

Use this structure in interviews:

### Step 1: Users and Traffic

```
DAU:              _____ users
Reads/user/day:   _____
Writes/user/day:  _____

Average read QPS:  DAU × reads  / 86,400 = _____
Average write QPS: DAU × writes / 86,400 = _____
Peak QPS:          average × 3–5× = _____
```

### Step 2: Storage

```
New records/day:  write_QPS × 86,400 = _____
Record size:      _____ bytes
Daily storage:    records/day × size = _____ GB/day
Annual storage:   daily × 365 = _____ TB/year
5-year total:     annual × 5 = _____
With replication: × 3 (standard) = _____
```

### Step 3: Bandwidth

```
Read bandwidth:  read_QPS × response_size = _____ MB/s
Write bandwidth: write_QPS × request_size = _____ MB/s
```

### Step 4: Servers

```
Typical server handles:
  CPU-bound API:    1,000–5,000 RPS
  Memory-bound:     10,000–50,000 RPS
  I/O-bound API:    500–2,000 RPS

Servers needed = peak_QPS / RPS_per_server
Add 50–100% buffer for headroom
```

---

## Worked Example: Design Twitter Scale

**Given:** 300M DAU, average user sees 20 tweets and posts 1 tweet per day.

**Step 1: Traffic**
```
Read QPS  = 300M × 20  / 86,400 = 69,000 QPS average → ~350K peak
Write QPS = 300M × 1   / 86,400 = 3,472  QPS average → ~17K peak
```

**Step 2: Storage (tweets)**
```
New tweets/day = 3,472 × 86,400 = 300M tweets/day
Tweet size = 300 bytes text + 100 bytes metadata = 400 bytes
Daily storage  = 300M × 400 bytes = 120 GB/day
Annual storage = 120 GB × 365 = 43.8 TB/year
10-year total  = 438 TB (raw), ~1.3 PB with 3× replication
```

**Step 3: Timeline reads**
```
69,000 read QPS × 20 tweets per read × 400 bytes = ~553 MB/s read bandwidth
With Redis cache (95% hit rate): only 5% hits DB → 3,450 DB read QPS
```

**Step 4: Servers**
```
API servers (I/O bound, 2K RPS each): 350K / 2K = 175 servers → round to 200
Redis: 350K read QPS / 100K per instance = 3.5 → 4 Redis instances
DB reads: 3,450 QPS / 10K per Postgres = 0.35 → 1 primary + 2 read replicas
DB writes: 17K QPS / 5K per Postgres = 3.4 → need to shard (4 shards)
```

**Conclusion:** ~200 API servers, 4 Redis instances, 4 sharded Postgres clusters.

---

## LLM and AI Agent Cost Numbers

Important for staff-level engineers working on AI systems:

### Token Counts (2025 reference)

```
~1 token ≈ 4 characters ≈ 0.75 words (English)

1K tokens ≈ 750 words ≈ 1.5 pages of text
Context window sizes:
  GPT-4o:       128K tokens input
  Claude Sonnet: 200K tokens input
  Gemini 1.5:   1M tokens input
```

### LLM Inference Latency

```
Model type          TTFT (first token)    Throughput
────────────────────────────────────────────────────
Hosted API (GPT-4)  0.5–2 s              40–80 tokens/sec
Hosted API (Claude) 0.3–1.5 s            60–100 tokens/sec
Self-hosted (7B)    50–200 ms            30–100 tokens/sec (1xA100)
Self-hosted (70B)   200–600 ms           5–20 tokens/sec (8xA100)
```

### LLM Cost (approximate, 2025)

```
Model               Input (per 1M tokens)   Output (per 1M tokens)
──────────────────────────────────────────────────────────────────
GPT-4o              $2.50                   $10.00
GPT-4o-mini         $0.15                   $0.60
Claude Sonnet 3.5   $3.00                   $15.00
Claude Haiku 3.5    $0.80                   $4.00
Llama 3.1 70B       $0.20–0.90 (hosted)     $0.20–0.90
```

**Agent cost estimation:**
```
Agent run = avg 5 LLM calls × avg 2K input + 500 output tokens
Cost per run ≈ 5 × (2K × $3/1M + 500 × $15/1M)
            ≈ 5 × ($0.006 + $0.0075)
            ≈ $0.067 per agent run

At 100K runs/day: $6,700/day ≈ $200K/month
→ Caching, smaller models for sub-tasks, and context truncation matter enormously
```

---

## High-Scale Reference Numbers

A few real-world figures to calibrate intuition:

| Service | Scale (approx.) |
|---------|----------------|
| Twitter/X writes | ~6,000 tweets/sec peak |
| Instagram uploads | ~1,000 photos/sec |
| Netflix streaming | >1 TB/sec globally |
| WhatsApp messages | ~100 billion/day |
| Google search queries | ~8.5 billion/day (~100K QPS) |
| Amazon orders | ~1,600 orders/sec |
| Visa transactions | ~24,000 TPS peak |
| Stripe transactions | ~1,000–10,000 TPS |
| Cloudflare CDN | ~46M HTTP requests/sec |

---

## Quick Conversions Cheat Sheet

```
Time
  1 day    = 86,400 seconds  ≈ 10^5 sec
  1 month  ≈ 2.6 million seconds
  1 year   ≈ 31.5 million seconds ≈ 3 × 10^7

Throughput to data
  1 KB/s   = 86 MB/day    = 31 GB/year
  1 MB/s   = 86 GB/day    = 31 TB/year
  1 GB/s   = 86 TB/day    = 31 PB/year

Powers of 10
  1K  = 10^3    (thousand)
  1M  = 10^6    (million)
  1B  = 10^9    (billion)
  1T  = 10^12   (trillion)
```

---

## Interview Tips

1. **State your assumptions loudly.** "I'll assume 50M DAU with 10 requests per user per day." The interviewer can correct you if wrong.

2. **Round aggressively.** 86,400 ≈ 10^5. 48 hours ≈ 2 days. Error within 2–3× is fine.

3. **Use powers of 10.** 50M × 10 req = 500M req/day / 10^5 sec = 5,000 req/sec. Easy mental math.

4. **Drive conclusions from numbers.** "At 5,000 writes/sec, a single Postgres handles this comfortably. I'll add a read replica but don't need sharding yet."

5. **Know when scale is a problem.** If QPS > 50K, caching is critical. If storage > 10 TB, sharding or object storage is needed. If writes > 20K QPS, consider Cassandra.

6. **For AI systems, cost matters.** Token cost math, cache hit rate impact on cost, and model selection (large vs small) are fair game at staff level.

---

*Previous: [08 - Database Indexing](./08-db-indexing.md)*  
*Next: [10 - Security Fundamentals](./10-security-fundamentals.md)*

