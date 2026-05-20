# Pattern 03: Multi-Step Processes

The **Multi-Step Processes** pattern is used to handle complex, asynchronous, and failure-prone workflows that span multiple services or external APIs. 

In a distributed microservice architecture, standard database transactions (ACID) are impossible. When an action involves multiple operations (e.g., charging a card, reserving inventory, booking shipping, and sending an email), the system must guarantee eventual consistency and robust recovery, even in the event of hardware crashes or network partitions.

---

## 1. The Distributed Transaction Dilemma

Consider a standard e-commerce order process:

```
+------------+     (1) Charge Card     +-----------------+
|            | ----------------------> | Payment Gateway |
|            |                         +-----------------+
|            |     (2) Reserve Items   +-----------------+
| Order Flow | ----------------------> | Inventory Serv. |
|            |                         +-----------------+
|            |     (3) Book Courier    +-----------------+
|            | ----------------------> | Shipping Serv.  |
+------------+                         +-----------------+
```

If **Step 1** (Payment) and **Step 2** (Inventory) succeed, but **Step 3** (Shipping) fails due to a network timeout, what happens to the user's money and the reserved stock? 
*   **The Baseline Approach:** Single-server synchronous orchestrator. If a step fails, try to call the preceding steps' API rollback endpoints in a catch block.
*   **Why the Baseline Fails:** If the orchestrator itself crashes or loses power mid-execution, the rollback is never triggered, leaving the system in a permanently inconsistent state (e.g., money charged, but no order placed).

---

## 2. Advanced Multi-Step Coordination Patterns

To scale and secure multi-step processes, systems use three main patterns.

### Why Not Two-Phase Commit (2PC)?

Before exploring Sagas, it's worth understanding why the classical distributed transaction protocol — **Two-Phase Commit (2PC)** — is unsuitable for modern microservice architectures.

2PC uses a **coordinator** that orchestrates a two-phase protocol across all participating services:
1.  **Prepare Phase:** The coordinator asks each participant to "prepare" (acquire locks, validate data). Each participant responds with `VOTE_COMMIT` or `VOTE_ABORT`.
2.  **Commit Phase:** If all participants voted commit, the coordinator sends a global `COMMIT`. Otherwise, it sends `ABORT`.

**Why 2PC fails at scale:**

| Problem | Impact |
|---|---|
| **Blocking Protocol** | All participants **hold database locks** during the prepare phase, waiting for the coordinator's decision. If the coordinator is slow or crashes, locks are held indefinitely, blocking other transactions. |
| **Single Point of Failure** | The coordinator is a single point of failure. If it crashes after sending `PREPARE` but before sending `COMMIT`, participants are stuck in an uncertain state (the "in-doubt" problem). |
| **Reduced Availability** | Per the CAP theorem, 2PC sacrifices availability for consistency. Any single participant failure or network partition causes the entire transaction to abort. |
| **Latency** | The protocol requires at minimum 2 round trips (prepare + commit) across all participants, adding significant latency as participants increase. |

> In a system with $n$ participants, a single 2PC transaction requires $2n$ network messages (prepare + commit to each). With 5 services at 50ms per hop, the minimum coordination overhead is $2 \times 5 \times 50\text{ms} = 500\text{ms}$ — and that's without accounting for lock contention.

This is why distributed systems use **Sagas** — which trade strong consistency for availability and partition tolerance, achieving **eventual consistency** through compensating transactions.

### A. The Saga Pattern (Orchestration vs. Choreography)
A Saga is a sequence of local transactions. Each transaction updates the database within a single service. If a step fails, the Saga runs **compensating transactions** (reversal updates) to undo the previous changes.

#### Option 1: Orchestration-based Saga (Centralized)
A dedicated central orchestrator service coordinates the execution of all steps. It instructs services to execute local transactions and is responsible for initiating rollbacks if a step fails.

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Orch as Saga Orchestrator
    participant Pay as Payment Service
    participant Inv as Inventory Service

    Client->>Orch: "Start Order"
    Orch->>Pay: "Charge Card"
    Pay-->>Orch: "Success (Charged $100)"
    Orch->>Inv: "Reserve Stock (Failed!)"
    Inv-->>Orch: "Out of Stock!"
    Note over Orch: Trigger Compensating Transactions
    Orch->>Pay: "Refund Card ($100)"
    Pay-->>Orch: "Success (Refunded)"
    Orch-->>Client: "Return Failure (Out of stock)"
```

*   **Trade-offs:**
    *   **Pros:** Easy to understand; centralizes state and logic; simple to audit.
    *   **Cons:** Central point of failure; orchestrator can become a bottleneck; tight coupling between the orchestrator and sub-services.

---

#### Option 2: Choreography-based Saga (Decentralized/Event-Driven)
There is no central coordinator. Services listen to a message queue and trigger their local transactions in response to events published by other services.

```mermaid
sequenceDiagram
    autonumber
    participant Pay as Payment Service
    participant Queue as Event Broker
    participant Inv as Inventory Service

    Pay->>Pay: "Charge Card"
    Pay->>Queue: "Publish Event: CardCharged"
    Queue->>Inv: "Consume Event: CardCharged"
    Inv->>Inv: "Reserve Stock"
    Note over Inv: Success! Emit Event: StockReserved
```

*   **Trade-offs:**
    *   **Pros:** Highly decoupled; no single point of failure; natural fit for asynchronous, event-driven microservices.
    *   **Cons:** Very hard to debug and visualize; risk of circular event dependencies; complex rollback flows (each service must listen to and handle all potential failure events).

---

### B. Durable Execution Engines (Workflow Engines)
For business-critical, long-running processes, design your system around **Durable Execution Engines** (e.g., **Temporal**, **AWS Step Functions**, or **Netflix Conductor**).

These systems use event sourcing histories to record the execution state of your code at every step. If the host server executing the workflow crashes mid-step, the engine automatically migrates and resumes the execution on another server **exactly** where it left off, maintaining local variables and call stacks.

```mermaid
flowchart TD
    Start[Start Workflow] --> Step1["Step 1: Charge Card"]
    Step1 -->|State Persisted to Journal| Crash{Host Crashes?}
    Crash -->|Yes| Recovery["Engine Restores Call Stack on Server 2"]
    Recovery --> Step2["Step 2: Reserve Inventory"]
    Crash -->|No| Step2
    Step2 --> End[Complete Workflow]
```

*   **Trade-offs:**
    *   **Pros:** Guarantees successful completion of complex code flows; built-in timeout, retries, and sleep operations (can sleep a workflow for 30 days safely); eliminates complex state tracking tables.
    *   **Cons:** Introduces new architectural dependencies; workflow code must be strictly deterministic (cannot use randomized logic or system timestamps directly without wrappers); high infrastructure overhead.

---

### C. Transactional Outbox Pattern

When a service needs to both **write to its database** and **publish an event** to a message broker (e.g., Kafka), it faces the **dual-write problem**: these are two separate I/O operations with no shared transaction boundary. If the DB write succeeds but the Kafka publish fails (or vice versa), the system is left in an inconsistent state.

The **Transactional Outbox** solves this by writing both the business data and the event into the **same database** within a **single ACID transaction**. The event is stored in a dedicated `outbox` table. A separate process (typically a CDC connector like **Debezium**) tails the database's WAL/binlog and publishes outbox rows to Kafka.

```mermaid
sequenceDiagram
    autonumber
    participant Svc as Order Service
    participant DB as Database
    participant CDC as Debezium CDC
    participant Kafka as Kafka
    participant Consumer as Consumer Service

    Svc->>DB: "BEGIN TX: INSERT order + INSERT outbox row"
    DB-->>Svc: "TX COMMIT OK"
    Note over DB: Both rows written atomically
    CDC->>DB: "Tail WAL/binlog for new outbox rows"
    DB-->>CDC: "New outbox row detected"
    CDC->>Kafka: "Publish OrderCreated event"
    Kafka->>Consumer: "Consume OrderCreated event"
    Note over Consumer: Process event and update own state
```

*   **Trade-offs:**
    *   **Pros:** Atomic guarantee — no dual-write risk; works with any relational database; the outbox table serves as an audit log of all published events.
    *   **Cons:** Added latency from CDC polling interval (typically 100ms–1s); the outbox table grows continuously and requires periodic cleanup (DELETE or partitioned table rotation); CDC connectors add operational complexity.

> **Cross-reference:** The Transactional Outbox pattern is also the mechanism behind CDC-based synchronization in [Pattern 04: Scaling Reads](./04_scaling_reads.md) (CQRS section) and is essential for reliable event publishing in [Pattern 05: Scaling Writes](./05_scaling_writes.md).

---

## 3. Idempotency & Retries (The Critical Guardrails)

Every multi-step architecture requires two strict policies to prevent duplicate actions under network retries.

### A. Idempotency Keys
In distributed networks, "at-least-once" delivery guarantees mean duplicate requests **will** occur. If an API times out, the client will retry the request.
*   **The Solution:** The client generates a unique **Idempotency Key** (usually a UUID) for the transaction. The server checks this key before executing the write.

```mermaid
sequenceDiagram
    autonumber
    client->>server: "POST /payments (Idempotency-Key: pay_abc123)"
    Note over server: Key not found. Execute charge, save to DB.
    server-->>client: "200 OK (Charged)"
    Note over client: Network packet lost! Retry request.
    client->>server: "POST /payments (Idempotency-Key: pay_abc123)"
    Note over server: Key found in DB! Skip execution.
    server-->>client: "200 OK (Return cached response)"
```

*   **Implementation Mechanics:**
    *   Store idempotency keys in a fast, ACID-compliant database (e.g., Redis with TTL or Postgres with a unique constraint index).
    *   Include a `status` field: `PENDING`, `COMPLETED`, `FAILED`. If a duplicate request arrives while the status is `PENDING`, reject it or block until the first completes (avoids concurrent race conditions).

### B. Retry with Exponential Backoff and Jitter
When a sub-service is down, retrying immediately can overload the failing service (causing a retry storm). Use exponential backoff (e.g., waiting 1s, 2s, 4s, 8s) combined with random noise (jitter) to distribute the retry load.

---

## 4. Multi-Step Coordination Matrix

| Metric | Monolithic Orchestrator | Saga Choreography | Saga Orchestration | Durable Engines (Temporal) |
|---|---|---|---|---|
| **Coordination** | Synchronous | Event-Driven | Command-Driven | Event Sourced |
| **State Storage** | Application Memory | Decentralized DBs | Centralized Saga DB | Central Engine Log |
| **Complexity** | Low | High | Medium | High |
| **Fault Tolerance** | Low | High | Medium | Extreme |
| **Best Used For** | Simple internally owned APIs | Event-driven microservices | Multi-service updates | Multi-day processes, RAG ingestion pipelines |

---

## 5. Advanced Interview Deep Dives

### Q1: What happens if a Compensating Transaction (Rollback) fails?
If a Saga attempts to refund a credit card, but the payment gateway returns a `500 Server Error`, the system is left in an inconsistent state.
*   **The Solution:**
    1.  **Retry Queues:** Route the failed compensating transaction to a persistent retry queue with a long backoff window.
    2.  **Dead-Letter Queues (DLQs):** If retries are exhausted (e.g., the user's card was cancelled), move the message to a DLQ.
    3.  **Human Operations Console:** Build a dashboard and automated alerting system (e.g., PagerDuty) to notify human operators to resolve the inconsistency manually (e.g., customer support wire transfers).

*   **Compensation Ordering:** Compensations must execute in **reverse order** of the original forward transactions. If the forward saga ran `Payment → Inventory → Shipping`, the compensation must run `Undo Shipping → Undo Inventory → Undo Payment`. Running compensations out of order can violate business invariants (e.g., releasing reserved inventory before cancelling the shipment that depends on it).

*   **Compensation Side Effects:** Be aware that some compensating transactions produce **externally visible side effects** that cannot be undone. For example:
    *   A refund triggers a "Your order has been cancelled" notification email — even if the order was never actually confirmed from the customer's perspective.
    *   A payment reversal may appear on the customer's bank statement as a charge-then-refund pair.
    *   **Mitigation:** Use **semantic locks** — mark the saga step as `PENDING` rather than `COMPLETED` until the entire saga succeeds. Only send customer-facing notifications (emails, push) after the saga reaches a terminal state.

### Q2: How do you handle "Out-of-Order" Saga Events?
Due to network delays, a `CancelOrder` event might arrive at a microservice *before* the corresponding `CreateOrder` event has completed processing.
*   **The Solution (State Machine Guardrails):**
    *   Maintain a state version or timestamp in your record.
    *   If a `CancelOrder` event arrives first, create a record with the status `CANCELLED` (or a sentinel flag).
    *   When the late `CreateOrder` event finally arrives, check the status first: if it is already marked as `CANCELLED`, discard the create event.

---

## 6. Security Considerations

Multi-step processes introduce unique attack surfaces because they span multiple services and persist state over long durations.

### Idempotency Key Spoofing
Idempotency keys must be **scoped to the authenticated user**. Without this, an attacker who discovers or guesses another user's idempotency key can hijack their cached response or block their transaction.
*   **Mitigation:** Store idempotency keys as a composite key of `(user_id, idempotency_key)`. Validate ownership on every request.

### Replay Attacks
Idempotency keys that never expire allow an attacker to replay a captured request indefinitely.
*   **Mitigation:** Enforce a **TTL** on idempotency keys (e.g., 24–72 hours). After expiration, the key is purged, and the same request would be treated as a new operation. Combine with request signing (HMAC) and timestamp validation.

### Saga State Tampering
Saga execution state (current step, completed steps, compensation status) must **never** be trusted from client input. If a client can manipulate saga state, they could skip payment steps or prevent compensations from running.
*   **Mitigation:** Store all saga state **server-side** (in the orchestrator's database or the workflow engine's journal). The client should only hold opaque references (e.g., a saga ID), never the state itself.

### Event Injection
In choreography-based sagas, a malicious actor with access to the event broker could inject fake events (e.g., a forged `PaymentSucceeded` event).
*   **Mitigation:** Sign events with HMAC or use mutual TLS between services. Consumers should validate event signatures before processing.

---

## 7. Observability and Monitoring

Multi-step processes are inherently difficult to debug because failures can occur at any step across multiple services. Robust observability is non-negotiable.

### Distributed Tracing with Correlation IDs
Propagate a unique **correlation ID** (trace ID) across every step of the saga using [OpenTelemetry](https://opentelemetry.io/). This allows you to reconstruct the full execution path of a single transaction across all participating services.
*   Every log line, metric, and event should include the correlation ID.
*   Use structured logging (JSON) so traces can be queried in systems like Jaeger, Datadog, or Grafana Tempo.

### Key Metrics to Track

| Metric | What It Tells You | Alert Threshold (Example) |
|---|---|---|
| **Saga Completion Rate** | Percentage of sagas reaching a terminal success state | < 99% over 5 minutes |
| **Compensation Trigger Rate** | How often compensations are invoked (indicates upstream failure rates) | > 5% of all sagas |
| **Average Saga Duration** | End-to-end latency of the multi-step process | p99 > 30 seconds |
| **DLQ Size** | Number of messages in the Dead Letter Queue (unresolvable failures) | > 0 for critical queues |
| **Idempotency Key Collision Rate** | Frequency of duplicate request detection | Spike may indicate retry storms |

### Circuit Breakers for Downstream Calls
Wrap each downstream service call within a **circuit breaker** (e.g., using [Resilience4j](https://resilience4j.readme.io/) or Hystrix). If a downstream service begins failing consistently, the circuit breaker trips to `OPEN`, and the saga immediately triggers compensations rather than waiting for timeouts.

> **Cross-references:**
> *   For real-time monitoring of saga state changes, see [Pattern 01: Real-Time Updates](./01_realtime_updates.md) — use WebSockets/SSE to push saga status to client dashboards.
> *   For handling write contention when multiple sagas compete for the same resource, see [Pattern 02: Dealing with Contention](./02_dealing_with_contention.md).
> *   For long-running sagas (e.g., multi-day approval workflows), see [Pattern 07: Long-Running Tasks](./07_long_running_tasks.md).
