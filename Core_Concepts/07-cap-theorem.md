# CAP Theorem for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** CAP theorem definition, practical interpretation, CP vs AP systems, PACELC, how to apply it in interviews

---

## What Is CAP Theorem?

CAP theorem (Brewer, 2000) states that a distributed data system can guarantee at most **two** of three properties simultaneously:

| Property | Definition |
|----------|-----------|
| **C**onsistency | Every read returns the most recent write (or an error). All nodes see the same data at the same time. |
| **A**vailability | Every request to a non-failing node returns a response (not an error). |
| **P**artition tolerance | The system continues operating despite network partitions (message loss or delay between nodes). |

> ⚠️ **CAP "consistency" ≠ ACID "consistency"**  
> ACID consistency means data follows integrity constraints (foreign keys, etc.).  
> CAP consistency means all nodes agree on the same value at the same time.  
> They're different properties. Don't conflate them.

---

## The Key Insight: Partition Tolerance Is Not Optional

In any real distributed system, network partitions will happen. Networks drop packets. Data centers lose connectivity. Nodes restart mid-transaction.

**You cannot sacrifice P.** If a partition occurs and your system stops working entirely, that's not a choice — that's a failure.

So in practice, **CAP = choose CP or AP when a partition occurs**:

- **CP (Consistency + Partition tolerance):** During a partition, some nodes refuse to respond rather than serve potentially stale data.
- **AP (Availability + Partition tolerance):** During a partition, nodes keep responding with the data they have, even if it might be stale.

```
                   ┌──────────────────────────────┐
                   │   CAP Theorem (practical)     │
                   │                               │
   Partition       │  Partition                    │
   occurs → choose │  Occurs                       │
                   │      │                        │
           ┌───────┴──────┼───────────────┐        │
           │              │               │        │
           ▼              │               ▼        │
     Refuse to        Return possibly  Accept      │
     respond until    stale data       writes on   │
     partition heals  immediately      all nodes   │
     (CP)             (AP)                         │
                   └──────────────────────────────┘
```

---

## Understanding CP vs AP Through an Example

### Setup

Two database replicas: one in the US, one in Europe. A network partition breaks replication.

User A (US) updates their profile picture → US replica is updated → replication blocked → EU replica still has the old picture.

### CP Choice: Return Error

When User B (EU) queries User A's profile:
- EU replica can't confirm it has the latest data
- Returns an error or "service unavailable"
- Data stays consistent: you never see stale data
- System is less available: some requests fail during the partition

**Result:** correct data, but degraded availability.

### AP Choice: Return Stale Data

When User B (EU) queries User A's profile:
- EU replica returns the old profile picture
- System stays available: no errors
- Data may be stale: briefly shows outdated picture
- Eventually consistent: once the partition heals, EU replica updates

**Result:** always responds, but data may be briefly incorrect.

---

## Choosing CP vs AP: The Right Questions

**Choose CP (consistency over availability) when stale data would cause real harm:**

| System | Why CP | Risk of Stale Data |
|--------|--------|-------------------|
| Financial transactions | Double-spend, overdraft | Account balances must be authoritative |
| Inventory management | Oversell out-of-stock items | $$ refunds, customer anger |
| Ticket booking | Seat sold twice | Operational failure, refunds |
| Distributed locks | Two processes enter critical section | Data corruption |
| Leader election | Two nodes think they're the leader | Split-brain, data loss |

**Choose AP (availability over consistency) when brief staleness is acceptable:**

| System | Why AP | Staleness Impact |
|--------|--------|-----------------|
| Social media feeds | Old posts briefly visible | Negligible |
| Product catalog | Yesterday's description | Negligible |
| Recommendation engine | Slightly outdated suggestions | Negligible |
| DNS | Old IP address cached | Usually fine |
| Shopping cart | Stale item count briefly | Rare, recoverable |
| Analytics dashboards | Metrics slightly behind | Acceptable |

**The test:** "Would it be catastrophic if users briefly saw inconsistent data?" If yes, CP. If no, AP.

---

## Consistency Spectrum: Not Binary

In practice, consistency isn't binary. There's a spectrum of consistency models:

```
Weak ──────────────────────────────────────────────── Strong
 │                                                        │
Eventual    Monotonic Read    Read-Your-Writes    Linearizable
Consistency  Consistency      Consistency         (strongest)

• Eventual: Writes propagate eventually. No ordering guarantee.
• Monotonic Read: Once you read a value, you'll never see an older one.
• Read-Your-Writes: You always see your own writes immediately.
• Linearizable: Global total order. Every operation appears instantaneous.
```

Most "AP" systems offer **eventual consistency** — they promise data will converge but not when.

Many systems let you choose per-operation consistency level:

| System | Consistency Knobs |
|--------|------------------|
| Cassandra | `QUORUM`, `ONE`, `ALL` per query |
| DynamoDB | `STRONG` or `EVENTUAL` per read |
| MongoDB | Read concern (`majority`, `local`, `linearizable`) |
| ZooKeeper | Always linearizable (it's a CP system) |

---

## Database Placement on CAP

| Database | Category | Notes |
|----------|----------|-------|
| PostgreSQL, MySQL | CA (single-node) | No partition tolerance by design; not distributed |
| PostgreSQL (synchronous replication) | CP | Blocks writes if replica unreachable |
| MySQL Galera / Vitess | CP | Multi-master with strong consistency |
| Apache Cassandra | AP | Tunable; defaults to eventual consistency |
| Amazon DynamoDB | AP (default) | Strongly consistent reads available at cost |
| Apache ZooKeeper | CP | Refuses writes without quorum |
| etcd, Consul | CP | Raft-based, linearizable |
| Amazon S3 | AP → CP | Eventual consistency for overwrite-PUTs historically; strong read-after-write now |
| Couchbase, Riak | AP | Conflict resolution (CRDTs, vector clocks) |

> **Interview note:** When asked "Is X consistent?", the answer is almost always "it depends on configuration." Know the defaults.

---

## Beyond CAP: PACELC

CAP only considers behavior during partitions. In practice, latency matters always — not just during failures.

**PACELC** (Daniel Abadi, 2012) extends CAP:

> If there's a **P**artition, choose between **A**vailability and **C**onsistency.  
> **E**lse (normal operation), choose between **L**atency and **C**onsistency.

```
System          Partition behavior    Else (normal ops)   Classification
──────────────────────────────────────────────────────────────────────
DynamoDB        Available             Low latency         PA/EL
Cassandra       Available             Low latency         PA/EL
MongoDB*        Available             Low latency         PA/EL  (default config)
MongoDB†        Consistent            Consistent          PC/EC  (majority write concern)
HBase           Consistent            Consistent          PC/EC
BigTable        Consistent            Consistent          PC/EC
```

> ⚠️ **MongoDB is commonly misclassified.** With default write concern `w:1`, MongoDB is PA/EL — it prioritizes availability and latency. With `w:"majority"` and `readConcern:"majority"`, it becomes PC/EC. This is configuration-dependent, not intrinsic to the system. Always state the configuration when classifying MongoDB.

**Why PACELC matters for interviews:** During normal operation (no partitions — 99.99% of the time), the real trade-off is between *latency* and *consistency*, not between availability and consistency. Synchronous replication costs write latency. Async replication is faster but allows brief inconsistency. PACELC makes this explicit.

---

## Applying CAP in an Interview

CAP should be addressed early in the **non-functional requirements** phase. Frame it as a design constraint that drives architectural decisions.

### Framing the Conversation

```
Interviewer: "Design a seat booking system for concerts."

You: "Before I dive into the design, let me establish some non-functional 
requirements. This is a booking system, so consistency is critical — we 
cannot double-sell seats. I'll prioritize consistency over availability. 
This means during a network partition, some requests may fail rather than 
risk two users booking the same seat. 

Practically, this means I'll use synchronous replication and potentially 
distributed transactions for the booking operation, accepting some write 
latency in exchange for correctness."
```

### CP Design Choices

When you commit to CP:
- Use a strongly-consistent data store (Postgres with synchronous replication, etcd, ZooKeeper)
- Use distributed locks (Redis SETNX, DynamoDB conditional writes) for critical operations
- Use pessimistic locking for inventory/seat reservations
- Accept higher write latency
- Plan for graceful degradation (queue requests, retry after partition heals)

### AP Design Choices

When you choose AP:
- Use eventually consistent stores (Cassandra, DynamoDB with eventual reads)
- Use optimistic concurrency (version numbers, last-write-wins)
- Design for conflict resolution (CRDTs for counters/sets)
- Cache aggressively (stale data is acceptable)
- Return partial data when complete data is unavailable

---

## Common Interview Mistakes

**Mistake 1: "I'll use [database] because it's consistent."**  
Better: "I'll use [database] with synchronous replication and `QUORUM` reads because this operation requires linearizable consistency."

**Mistake 2: Making everything CP.**  
Not all operations in your system need the same consistency level. A booking operation is CP. Displaying the event description is AP. Applying the right model to each operation is a sign of sophistication.

**Mistake 3: Forgetting about partition tolerance.**  
"I'll use a single Postgres node to avoid consistency issues" — that's fine at small scale, but ignores availability (SPOF) and scalability.

**Mistake 4: Conflating CAP consistency with ACID consistency.**  
AP systems can still have ACID-compliant individual nodes. The CAP theorem is about distributed agreement across nodes, not local transaction semantics.

---

## Interview Summary

```
Question: "Does this system need strong consistency?"

Ask yourself:
  Would stale data cause financial, safety, or severe UX harm?
  ├── Yes → CP (strong consistency)
  │         Use synchronous replication, distributed locks, quorum reads
  │         Accept: higher latency, lower availability during partitions
  │
  └── No  → AP (eventual consistency)
            Use async replication, optimistic concurrency, eventual reads
            Accept: brief staleness, need conflict resolution strategy
            Benefit: lower latency, higher availability
```

---

*Previous: [06 - Consistent Hashing](./06-consistent-hashing.md)*  
*Next: [08 - Database Indexing](./08-db-indexing.md)*
