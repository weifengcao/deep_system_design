# Data Modeling for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** SQL vs NoSQL decision, schema design, keys & relationships, normalization trade-offs, choosing the right database

---

## What Interviewers Want to See

Data modeling comes up twice in a system design interview:

1. **Requirements phase** — identify *core entities* (users, orders, posts…). These become your tables/collections.
2. **High-level design** — sketch a schema alongside each storage component. Show key fields, relationships, and how you'd index for the main query patterns.

You don't need a fully normalized ER diagram. You need a schema that is *correct enough* to not create problems later (missing foreign keys, wrong cardinality, index-unfriendly query patterns).

---

## Step 1: Identify Core Entities

Before choosing a database, list the things your system manages. For a ticket-booking system:

| Entity | Description |
|--------|-------------|
| `User` | Anyone with an account |
| `Event` | A concert, game, or show |
| `Venue` | Physical location |
| `Ticket` | A seat at an event |
| `Booking` | A user's reservation of one or more tickets |
| `Payment` | Financial record for a booking |

Each entity usually becomes one table (SQL) or one collection (NoSQL).

---

## Step 2: Choose a Database Model

### Relational (SQL) — The Default

Relational databases organize data into tables with fixed schemas. Foreign keys express relationships. Transactions (ACID) guarantee consistency.

**Use SQL when:**
- Data has clear relationships (users → orders → line items)
- Strong consistency is required (payments, inventory, bookings)
- You need flexible ad-hoc queries (joins, aggregations)
- You're unsure — PostgreSQL is the right default for most systems

**Example schema (Ticketmaster):**

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       TEXT NOT NULL UNIQUE,
    name        TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE venues (
    id          UUID PRIMARY KEY,
    name        TEXT NOT NULL,
    city        TEXT NOT NULL,
    capacity    INT NOT NULL
);

CREATE TABLE events (
    id          UUID PRIMARY KEY,
    venue_id    UUID NOT NULL REFERENCES venues(id),
    name        TEXT NOT NULL,
    starts_at   TIMESTAMPTZ NOT NULL,
    total_seats INT NOT NULL
);

CREATE TABLE tickets (
    id          UUID PRIMARY KEY,
    event_id    UUID NOT NULL REFERENCES events(id),
    section     TEXT NOT NULL,
    row         TEXT,
    seat_number TEXT,
    price_cents INT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'available' -- available | reserved | sold
);

CREATE TABLE bookings (
    id              UUID PRIMARY KEY,
    user_id         UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'pending', -- pending | confirmed | cancelled
    total_cents     INT NOT NULL
);

CREATE TABLE booking_tickets (
    booking_id  UUID NOT NULL REFERENCES bookings(id),
    ticket_id   UUID NOT NULL REFERENCES tickets(id),
    PRIMARY KEY (booking_id, ticket_id)
);
```

### Document (NoSQL) — For Flexible or Hierarchical Data

Document databases (MongoDB, DynamoDB, Firestore) store JSON-like documents. Good when:
- Schema is rapidly evolving
- Data is naturally hierarchical (a blog post with embedded comments)
- Read patterns always access the document as a whole

**When to suggest it in interviews:** product catalogs with varied attributes, user-generated content, configuration stores, event logs.

**Caution:** joins are expensive or unsupported. Denormalize intentionally and document why.

### Wide-Column — For Time-Series and High Write Throughput

Cassandra and DynamoDB (when used with single-table design) excel at:
- High write throughput (millions/s)
- Time-ordered data (sensor readings, activity logs)
- Queries by a known partition key

**Design around access patterns, not normalization.** Define queries first, then design tables.

```
Table: user_activity
Partition key: user_id
Sort key: timestamp DESC

→ "Get last 100 events for user X" is efficient
→ "Get all events of type 'purchase'" requires a full scan — bad
```

### Key-Value — For Caching and Simple Lookups

Redis, DynamoDB (simple access). One key maps to one value. No relationships, no joins. Use for:
- Session storage
- Cache layer
- Rate limiting counters
- Leaderboards (Redis sorted sets)

### Graph — For Relationship-Heavy Traversals

Neo4j, Amazon Neptune. Model nodes and edges explicitly. Use when:
- Queries are relationship traversals (friends-of-friends, shortest path)
- Relationship attributes matter (weight, timestamp on an edge)

Rarely the right answer in interviews unless the problem is explicitly about graph traversal.

### Search — For Full-Text and Faceted Queries

Elasticsearch, OpenSearch. Use alongside your primary database when:
- Full-text search over unstructured text
- Faceted filtering (filter by multiple attributes simultaneously)
- Fuzzy matching, autocomplete

**Never make Elasticsearch your primary store.** It lacks ACID guarantees and is hard to update atomically.

---

## SQL vs NoSQL Decision Matrix

| Consideration | Favor SQL | Favor NoSQL |
|--------------|-----------|------------|
| Data relationships | Complex, normalized | Flat, hierarchical |
| Schema stability | Stable | Evolving |
| Transaction needs | Strong ACID | Eventual OK |
| Query patterns | Flexible / ad-hoc | Known at design time |
| Scale axis | Read replicas, then shard | Horizontal from day 1 |
| Team experience | Most engineers know SQL | Requires NoSQL expertise |

---

## Keys and Relationships

### Primary Keys

| Key Type | Pros | Cons |
|----------|------|------|
| Auto-increment int | Compact, sequential | Leaks record count; hard to shard |
| UUID (v4, random) | Global uniqueness, shard-friendly | Larger (16 bytes); random = index fragmentation |
| UUID v7 (time-ordered) | Shard-friendly + sequential | Newer, less library support |
| Snowflake ID | Time-ordered, shardable, 64-bit | Requires a generator service |

**Interview recommendation:** use UUID v4 or v7 unless you have a specific reason for auto-increment. Avoids enumeration attacks and works across shards.

### Relationship Types

```
One-to-One:   users (1) — (1) user_profiles
One-to-Many:  events (1) — (N) tickets
Many-to-Many: bookings (N) — (N) tickets  →  junction table: booking_tickets
```

In SQL, foreign keys enforce referential integrity. Decide per-relationship whether `ON DELETE CASCADE` or `ON DELETE RESTRICT` is appropriate (e.g., cascade-delete booking_tickets when a booking is deleted; restrict deletion of an event if tickets exist).

---

## Normalization vs Denormalization

**Normalized:** each piece of data stored once. Updates are easy; reads require joins.

**Denormalized:** data duplicated across records. Reads are fast; updates require updating multiple places.

| | Normalized | Denormalized |
|--|-----------|-------------|
| Read performance | Slower (joins) | Faster (one read) |
| Write complexity | Simple | Complex (update multiple places) |
| Storage | Compact | Larger |
| Consistency | Automatic (FK) | Must maintain manually |

**Rule of thumb:** start normalized. Denormalize specific hot paths when you have evidence of a performance bottleneck. Always document *why* a field is denormalized.

**Common denormalization patterns:**
- Store `username` on `comments` table (avoid join to `users` on every comment read)
- Store `like_count` on `posts` table (avoid `COUNT(*)` on `likes` per post)
- Materialize a `user_feed` table (precomputed timeline)

---

## Indexing Strategy (Overview)

*(Covered in depth in [08 - Database Indexing](./08-db-indexing.md))*

In your schema sketch, annotate indexes:

```sql
-- For "get all tickets for an event"
CREATE INDEX ON tickets (event_id);

-- For "get upcoming events in a city, sorted by date"
CREATE INDEX ON events (city, starts_at);

-- For "look up booking by confirmation code"
CREATE UNIQUE INDEX ON bookings (confirmation_code);
```

**Rules of thumb:**
- Always index foreign keys (unless you never query child→parent)
- Index columns in `WHERE` clauses on hot queries
- Composite indexes: order matters — put equality conditions before range conditions
- Every index costs write throughput and storage — don't over-index

---

## Soft Deletes

Hard deletes (`DELETE FROM users WHERE id = ?`) are irreversible and lose audit history. Most production systems use **soft deletes** instead:

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- "Delete" a user
UPDATE users SET deleted_at = now() WHERE id = ?;

-- All queries must filter out deleted rows
SELECT * FROM users WHERE deleted_at IS NULL AND id = ?;
```

**Pros:** recoverable, preserves audit trail, referential integrity stays intact  
**Cons:** every query needs `WHERE deleted_at IS NULL`; table grows without bounds; indexes must include the filter

**Partial index solves the performance problem:**
```sql
-- Only indexes non-deleted rows — much smaller, queries stay fast
CREATE INDEX ON users(email) WHERE deleted_at IS NULL;
```

**Pitfall:** unique constraints on soft-deleted tables. If email must be unique among *active* users:
```sql
-- Won't work: deleted user's email blocks re-registration
CREATE UNIQUE INDEX ON users(email);  ❌

-- Correct: unique only among active users
CREATE UNIQUE INDEX ON users(email) WHERE deleted_at IS NULL;  ✅
```

---

## Optimistic Locking (Concurrency Control)

When two transactions try to update the same row, you need a strategy. Two options:

**Pessimistic locking** — lock the row on read, block other writers until done. Safe but reduces throughput. Good for short, high-contention operations (seat booking).

```sql
BEGIN;
SELECT * FROM tickets WHERE id = ? FOR UPDATE;  -- acquires row lock
UPDATE tickets SET status = 'sold' WHERE id = ?;
COMMIT;
```

**Optimistic locking** — don't lock; instead, detect conflicts at write time using a version counter:

```sql
ALTER TABLE users ADD COLUMN version INT NOT NULL DEFAULT 1;

-- Read
SELECT id, email, version FROM users WHERE id = 42;
-- Returns: {id: 42, email: "old@x.com", version: 7}

-- Write (fails if someone else updated since your read)
UPDATE users
SET email = 'new@x.com', version = version + 1
WHERE id = 42 AND version = 7;
-- If 0 rows updated → conflict → retry
```

**Pros:** no locking overhead; high throughput for low-contention workloads  
**Cons:** requires application-level retry logic; under high contention, retry storms can occur

**Rule of thumb:** use optimistic locking for typical update flows (profile edits, settings); use pessimistic locking for inventory/financial operations where contention is expected and conflicts are expensive.

---

## Event Sourcing

Instead of storing current state, **event sourcing** stores the full history of events. Current state is derived by replaying events.

```
Traditional (state):
  users table: {id: 1, email: "new@x.com", plan: "pro"}

Event sourced:
  events table:
    {user_id: 1, type: "UserRegistered",   data: {email: "old@x.com"}, ts: t1}
    {user_id: 1, type: "EmailChanged",     data: {email: "new@x.com"}, ts: t2}
    {user_id: 1, type: "PlanUpgraded",     data: {plan: "pro"},        ts: t3}
```

**Benefits:**
- Complete audit trail — every state change is preserved
- Time-travel queries — "what did this user's account look like on Jan 1?"
- Event replay — rebuild read models, fix bugs by replaying with corrected logic
- Natural fit for CQRS (Command Query Responsibility Segregation)

**Costs:**
- Query complexity — current state requires event replay or a materialized view (projection)
- Storage growth — events accumulate forever (mitigated with snapshots)
- Eventual consistency between event store and projections

**When to use it:**
- Financial systems (ledgers, transactions — every cent must be traceable)
- Compliance-heavy domains (healthcare, legal)
- AI agent step logs — naturally event-sourced; each tool call and LLM response is an immutable event

```sql
-- Event sourcing schema for agent runs
CREATE TABLE agent_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id      UUID NOT NULL REFERENCES agent_runs(id),
    seq         INT NOT NULL,             -- ordering within a run
    event_type  TEXT NOT NULL,            -- 'llm_call' | 'tool_use' | 'memory_write'
    payload     JSONB NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (run_id, seq)
);
```

---

SQL schemas require migrations. For production systems:

1. **Additive migrations only** — adding nullable columns and new tables is safe; dropping or renaming columns requires a multi-phase deploy
2. **Never lock the table** — use `ALTER TABLE ... ADD COLUMN` which is usually instant in Postgres; avoid `NOT NULL` without a default on large tables
3. **Use a migration tool** — Flyway, Liquibase, Alembic, golang-migrate

**Safe rename pattern:**
```
Phase 1: Add new column `email_address`, backfill data, write to both columns
Phase 2: Read from new column; stop writing to old column
Phase 3: Remove old column `email`
```

---

## Data Modeling for AI Agent Systems

Agent systems introduce new modeling challenges:

### Agent Run / Trace

```sql
CREATE TABLE agent_runs (
    id              UUID PRIMARY KEY,
    user_id         UUID REFERENCES users(id),
    agent_type      TEXT NOT NULL,          -- 'customer_support', 'code_review'
    status          TEXT NOT NULL,          -- 'running' | 'completed' | 'failed'
    input           JSONB NOT NULL,
    output          JSONB,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    total_tokens    INT,
    cost_usd        NUMERIC(10,6)
);

CREATE TABLE agent_steps (
    id              UUID PRIMARY KEY,
    run_id          UUID NOT NULL REFERENCES agent_runs(id),
    step_number     INT NOT NULL,
    step_type       TEXT NOT NULL,          -- 'llm_call' | 'tool_use' | 'memory_read'
    input           JSONB NOT NULL,
    output          JSONB,
    latency_ms      INT,
    created_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (run_id, step_number)
);
```

### Memory / Long-Term Context

Agents need memory across sessions. Two common patterns:

**Episodic memory** — store conversation summaries:
```sql
CREATE TABLE agent_memories (
    id          UUID PRIMARY KEY,
    user_id     UUID REFERENCES users(id),
    content     TEXT NOT NULL,
    embedding   VECTOR(1536),               -- for semantic retrieval
    created_at  TIMESTAMPTZ NOT NULL
);
CREATE INDEX ON agent_memories USING ivfflat (embedding vector_cosine_ops);
```

**Tool state** — agents writing to external systems need idempotency:
```sql
CREATE TABLE tool_invocations (
    id              UUID PRIMARY KEY,
    run_id          UUID REFERENCES agent_runs(id),
    idempotency_key TEXT NOT NULL UNIQUE,   -- prevent duplicate tool calls
    tool_name       TEXT NOT NULL,
    status          TEXT NOT NULL,
    result          JSONB,
    created_at      TIMESTAMPTZ NOT NULL
);
```

---

## Interview Checklist

Before moving past the data modeling step, verify:

- [ ] All core entities have a table/collection
- [ ] Each entity has a primary key strategy (UUID preferred)
- [ ] Many-to-many relationships have junction tables
- [ ] Foreign key relationships are explicit
- [ ] At least one index called out per major query pattern
- [ ] Database type justified relative to consistency and query requirements
- [ ] Schema supports the non-functional requirements (sharding key present if scale is a concern)

---

*Previous: [02 - API Design](./02-api-design.md)*  
*Next: [04 - Caching](./04-caching.md)*
