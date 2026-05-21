# Core Concepts — System Design

This section covers the fundamental building blocks for staff+ backend engineering system design interviews. Inspired by Hello Interview's structure, extended with security, AI agent topics, and corrected/deeper coverage.

---

## Articles

| # | Topic | Key Concepts |
|---|-------|-------------|
| 00 | [System Design Playbook](./00-system-design-playbook.md) | 45-minute timeline, requirements gathering, capacity estimation equations, trade-off frameworks, security and resilience guardrails |
| 01 | [Networking Essentials](./01-networking-essentials.md) | OSI layers, TCP vs UDP, HTTP/2/3, REST/GraphQL/gRPC, WebSocket fan-out, mTLS, load balancers, CDN anycast, connection draining |
| 02 | [API Design](./02-api-design.md) | REST resource modeling, HTTP methods, idempotency keys + TTL, pagination, rate limiting algorithms, backward compat / Postel's Law |
| 03 | [Data Modeling](./03-data-modeling.md) | SQL vs NoSQL, schema design, soft deletes, optimistic locking, event sourcing, agent schemas |
| 04 | [Caching](./04-caching.md) | Cache layers, Redis Sentinel vs Cluster, cache-aside/write-through/write-behind, stampede, hot keys, read-your-own-writes |
| 05 | [Sharding](./05-sharding.md) | Shard key selection, hash/range/directory, logical vs physical shards, gh-ost live migration |
| 06 | [Consistent Hashing](./06-consistent-hashing.md) | Hash ring, virtual nodes, replication factor, W+R>N quorum math, Cassandra/DynamoDB/Redis |
| 07 | [CAP Theorem](./07-cap-theorem.md) | CP vs AP, PACELC (with corrected MongoDB classification), quorum consistency, interview framing |
| 08 | [Database Indexing](./08-db-indexing.md) | B-Tree vs LSM-Tree, composite/covering/partial/BRIN indexes, selectivity, planner statistics, vector indexes |
| 09 | [Numbers to Know](./09-numbers-to-know.md) | Corrected latency table (NVMe), p99 vs p50, memory bandwidth for LLM inference, capacity estimation |
| 10 | [Security Fundamentals](./10-security-fundamentals.md) | JWT pitfalls, OAuth2/PKCE, mTLS, secrets management, SQL/NoSQL/prompt injection, audit logging, PII, DDoS |

---

## How to Use This

1. **First pass:** Read each article top-to-bottom for conceptual understanding
2. **Interview prep:** Focus on summary tables, trade-off sections, and end-of-article checklists
3. **Practice:** Apply each concept to a design you know (Ticketmaster, Twitter, WhatsApp)
4. **Security:** Article 10 cross-cuts all others — weave it into each design phase

---

## Reading Order Recommendations

**Security-aware backend role (target role):**
00 → 10 → 07 → 01 → 04 → 02 → 03 → 05 → 06 → 08 → 09

**Distributed systems / infrastructure:**
00 → 06 → 05 → 07 → 08 → 04 → 01 → 09 → 02 → 03 → 10

**Full-stack / product backend:**
00 → 02 → 03 → 04 → 01 → 07 → 08 → 09 → 05 → 06 → 10

---

## Key Themes

- **Defaults matter:** REST, TCP, Postgres, Redis cache-aside, UUID PKs, Argon2id passwords. Know the defaults and when to deviate.
- **Trade-offs are the answer:** No single right choice — it depends on consistency requirements, scale, query patterns, threat model, team capability.
- **Numbers anchor decisions:** Every design choice should be validated with rough math. p99 matters, not just p50.
- **Security is first-class:** AuthN, AuthZ, transport, secrets, audit — surface these proactively.
- **AI agents are first-class:** Every article includes agent-specific considerations. Article 10 covers prompt injection and privilege separation.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| v1.0 | 2025-05 | Initial 9 core concepts |
| v1.1 | 2025-05 | Added Article 10 (Security). Fixed: WebSocket diagram, MongoDB PACELC, NVMe latency, B-Tree depth. Added: mTLS, CDN anycast, connection draining, Redis Sentinel vs Cluster, read-your-own-writes, logical shards, gh-ost, replication factor + quorum math, BRIN index, planner statistics, p99 table, memory bandwidth, idempotency key TTL, rate limiting algorithms, backward compat, soft deletes, optimistic locking, event sourcing. |
