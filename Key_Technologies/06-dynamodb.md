# DynamoDB Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** Single-table design, partition/sort keys, GSI/LSI, consistency, capacity modes, streams, DAX, when to use

---

## Why DynamoDB

Amazon DynamoDB is a **fully managed, serverless, key-value and document database** with single-digit millisecond performance at any scale. It eliminates ops burden (no clusters to manage, auto-scaling, built-in HA) and integrates deeply with the AWS ecosystem.

**Use DynamoDB when:**
- You're already in AWS and want zero ops overhead
- Access patterns are known and simple (lookup by key, range scans on sort key)
- You need predictable single-digit ms latency at any scale
- Auto-scaling from zero to millions of requests/sec is required
- You want Global Tables for multi-region active-active replication

---

## Core Concepts

### Items, Tables, and Keys

```
Table: "Orders"
┌────────────────────────────────────────────────────────────┐
│ PK (Partition Key)  │ SK (Sort Key)      │ Attributes      │
├────────────────────────────────────────────────────────────┤
│ USER#42             │ ORDER#2025-01-01-1 │ status, amount  │
│ USER#42             │ ORDER#2025-01-15-2 │ status, amount  │
│ USER#99             │ ORDER#2025-01-10-1 │ status, amount  │
│ ORDER#2025-01-01-1  │ ITEM#sku-123       │ qty, price      │
│ ORDER#2025-01-01-1  │ ITEM#sku-456       │ qty, price      │
└────────────────────────────────────────────────────────────┘
```

- **Partition key (PK):** Determines which partition stores the item. Must be unique if no sort key.
- **Sort key (SK):** Optional. Combined with PK forms the composite primary key. Items with the same PK are stored together, sorted by SK. Enables range queries within a partition.
- **Attributes:** Schema-free — each item can have different attributes.

### Single-Table Design

DynamoDB's most important and most misunderstood pattern. Instead of one table per entity (like relational design), you put **multiple entity types in one table**, using composite keys to encode relationships.

```
# User entity
PK: USER#42        SK: METADATA
  name: "Alice", email: "alice@x.com"

# User's orders
PK: USER#42        SK: ORDER#2025-01-01
  status: confirmed, total: 5000

# Order items (separate entity, same table)
PK: ORDER#2025-01-01  SK: ITEM#sku-123
  qty: 2, price: 1500

# Product catalog entry
PK: PRODUCT#sku-123  SK: METADATA
  name: "Widget", category: "tools"
```

**Why single-table?**
- All related data fetched in one request (`Query` by PK)
- Avoids cross-table joins (DynamoDB has no joins)
- Cost and performance efficiency

**The downside:** Harder to understand, requires upfront query design, hard to evolve.

### Access Patterns Drive Everything

Before writing a single line of DynamoDB schema, list all your access patterns:
```
1. Get user profile by user_id
2. Get all orders for a user (newest first)
3. Get items in an order
4. Get order by order_id (for status check)
5. Get all orders with status "pending" (admin view)
```

Each pattern maps to a DynamoDB Query/GetItem. Patterns 1–4 work naturally with the single-table composite key. Pattern 5 needs a GSI.

---

## Secondary Indexes

### GSI (Global Secondary Index)

A completely separate index with a different partition key and optional sort key. Spans the entire table.

```
Table: PK=USER#42, SK=ORDER#...
GSI "by-status": PK=status, SK=created_at

Query: "Get all PENDING orders sorted by date"
→ Query GSI by-status where PK=PENDING ORDER BY created_at
```

**GSI costs:** Writes to the GSI happen asynchronously. The GSI can lag behind the main table (eventual consistency only — no strongly consistent reads on GSIs).

### LSI (Local Secondary Index)

Same partition key as the table, but different sort key. Must be defined at table creation. Only queries within a partition.

```
Table: PK=USER#42, SK=ORDER#date-id
LSI: PK=USER#42, SK=status

Query: "Get user 42's orders with status=PENDING"
→ Query LSI where PK=USER#42, SK begins_with "PENDING"
```

LSIs are less commonly needed since GSIs are more flexible. Use LSIs when you need strongly consistent reads on the index.

---

## Capacity Modes

### Provisioned Capacity

You specify Read Capacity Units (RCU) and Write Capacity Units (WCU):
- 1 RCU = 1 strongly consistent read of ≤ 4KB/sec (or 2 eventually consistent)
- 1 WCU = 1 write of ≤ 1KB/sec

**Auto-scaling:** DynamoDB can auto-scale RCU/WCU within min/max bounds. Takes ~60–90 seconds to scale up — not instant enough for sudden spikes.

**Use when:** Predictable, steady traffic. Cost-optimized for known workloads.

### On-Demand Capacity

Pay per request. DynamoDB handles any traffic level instantly.

- **Reads:** $0.25 per million read request units
- **Writes:** $1.25 per million write request units

**Use when:** Unpredictable traffic, new applications, dev/test environments.

**Cost trap:** On-demand is 5–7× more expensive than provisioned at sustained high load. Switch to provisioned once traffic patterns are known.

---

## Consistency Models

### Eventually Consistent Reads (default)
Read from any replica. May return data up to ~1 second stale. Costs 0.5 RCU.

### Strongly Consistent Reads
Read from the leader replica. Always up-to-date. Costs 1 RCU. Not available on GSIs.

### Transactions
`TransactWriteItems` / `TransactGetItems` — ACID across up to 100 items in a single transaction (can span different tables):

```python
dynamodb.transact_write_items(
    TransactItems=[
        {
            "Update": {
                "TableName": "Inventory",
                "Key": {"PK": "PRODUCT#sku-123"},
                "UpdateExpression": "SET quantity = quantity - :qty",
                "ConditionExpression": "quantity >= :qty",
                "ExpressionAttributeValues": {":qty": 2}
            }
        },
        {
            "Put": {
                "TableName": "Orders",
                "Item": {"PK": "ORDER#xyz", "SK": "METADATA", "status": "confirmed"}
            }
        }
    ]
)
```

Transactions cost 2× the normal RCU/WCU. Use sparingly — only when atomicity across items is truly needed.

---

## DynamoDB Streams and Change Data Capture

DynamoDB Streams capture every item-level change (insert, update, delete) in an ordered log, retained for 24 hours.

```
Table Change → DynamoDB Stream → Lambda trigger or Kinesis

Use cases:
  • Sync DynamoDB to Elasticsearch (for search)
  • Invalidate cache on update
  • Send notifications when order status changes
  • Audit logging
  • Cross-region replication (Global Tables use streams internally)
```

```python
# Lambda triggered on DynamoDB Stream
def handler(event, context):
    for record in event['Records']:
        if record['eventName'] == 'MODIFY':
            new_item = record['dynamodb']['NewImage']
            # Sync to Elasticsearch
            es.index(index='orders', id=new_item['PK']['S'], body=new_item)
```

---

## DAX (DynamoDB Accelerator)

In-memory cache purpose-built for DynamoDB. Provides microsecond read latency (vs ms for DynamoDB):

```
Client → DAX Cluster → DynamoDB
         (cache hit: ~µs)  (cache miss: ~ms, stores in DAX)
```

- Fully managed, transparent to application (same DynamoDB API)
- Item cache + query cache
- Write-through: writes go to DynamoDB first, then DAX cache is updated
- Use for: read-heavy workloads where ms latency isn't fast enough

**When to use DAX vs Redis:**
- DAX: DynamoDB-specific, transparent cache, managed, no code change
- Redis: multi-source data, richer data structures, cross-service sharing

---

## Pitfalls

### Hot Partitions
If all traffic hits the same partition key (e.g., all writes to `STATUS#pending`), one partition becomes a bottleneck. Solutions:
- Use high-cardinality keys (user_id, UUID)
- Shard hot keys: `STATUS#pending#0` through `STATUS#pending#9` (randomize suffix, scatter reads)
- Use on-demand capacity (DynamoDB absorbs spikes within a partition more gracefully)

### Scan Operations
`Scan` reads every item in the table. Almost always wrong in production:
```python
# BAD: O(table size)
response = table.scan(FilterExpression=Attr('status').eq('pending'))

# GOOD: O(result size) using GSI
response = table.query(
    IndexName='by-status',
    KeyConditionExpression=Key('status').eq('pending')
)
```

### Item Size Limit
Each DynamoDB item is limited to 400KB. For large payloads (documents, blobs), store the payload in S3 and keep only the S3 key in DynamoDB.

### Transaction Limitations
Transactions span at most 100 items and incur 2× cost. Don't design systems that require large transactions — restructure the data model instead.

---

## DynamoDB vs Cassandra Comparison

| Dimension | DynamoDB | Cassandra |
|-----------|----------|-----------|
| Operations | Zero (fully managed) | High (self-managed or DataStax) |
| Scaling | Auto-scaling, serverless | Manual node addition |
| Multi-region | Global Tables (easy) | Multi-DC (complex config) |
| Query flexibility | Limited (PK + SK + GSI) | Similar limitation |
| Consistency | Tunable per read | Tunable per operation |
| Transactions | ✅ Up to 100 items | ✅ Lightweight transactions (LWT) |
| Cloud portability | ❌ AWS only | ✅ Any cloud/on-prem |
| Cost at scale | Higher at extreme scale | Lower (hardware cost only) |
| Use case fit | AWS ecosystem, serverless | High write throughput, multi-cloud |

---

## Interview Quick Reference

| Scenario | DynamoDB Pattern |
|----------|-----------------|
| "Get user and their recent orders" | Single-table, Query PK=USER#id |
| "Find all orders with status=pending" | GSI with PK=status |
| "Update inventory atomically with order creation" | TransactWriteItems |
| "Sync to Elasticsearch" | DynamoDB Streams → Lambda → ES |
| "Cache DynamoDB reads" | DAX for µs latency |
| "Hot partition on popular product" | Shard key: PRODUCT#id#shard0..9 |
| "Store large documents" | S3 for payload, DynamoDB for metadata + S3 key |

---

*Previous: [05 - Cassandra](./05-cassandra.md)*  
*Next: [07 - PostgreSQL](./07-postgresql.md)*
