# API Design for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** REST, GraphQL, gRPC, request/response design, pagination, versioning, idempotency

---

## Why API Design Matters

In a system design interview you'll spend roughly 5 minutes on the API step. The goal isn't a perfect spec — it's to show that you can model resources cleanly, communicate how clients interact with the system, and spot API-level risks early (idempotency, pagination, authorization).

For frontend and product roles the bar is higher. For staff-level backend roles the interviewer cares most that your API doesn't create downstream scalability problems and that you can defend your choices.

**Default to REST unless you have a specific reason not to.**

---

## Choosing Your API Style

```
Is this an internal service-to-service call under heavy load?
    └── YES → gRPC

Do diverse clients (mobile, web, partners) need very different data shapes?
    └── YES → GraphQL

Everything else?
    └── REST (default)
```

---

## REST

### Resource Modeling

REST is built around **resources** — the nouns in your system. Your core entities from the requirements phase map almost 1:1 to REST resources.

**Ticketmaster example:**

| Entity | REST Resource |
|--------|--------------|
| Event | `/events` |
| Venue | `/venues` |
| Ticket | `/tickets` |
| Booking | `/bookings` |

**Rules:**
- Resources are always **plural nouns** (`/events`, not `/event` or `/getEvent`)
- Relationships are represented by nesting: `/events/{id}/tickets`
- Operations map to HTTP methods, not to action verbs in the URL

**Bad (RPC-style):**
```
POST /createBooking
POST /cancelBooking
GET  /getUserEvents?userId=123
```

**Good (REST):**
```
POST   /bookings
DELETE /bookings/{id}
GET    /users/{id}/events
```

### HTTP Methods and Idempotency

| Method | Safe | Idempotent | Typical Use |
|--------|------|------------|-------------|
| GET | ✅ | ✅ | Fetch resource |
| HEAD | ✅ | ✅ | Fetch headers only |
| PUT | ❌ | ✅ | Replace entire resource |
| DELETE | ❌ | ✅ | Remove resource |
| PATCH | ❌ | ❌* | Partial update |
| POST | ❌ | ❌ | Create resource |

*PATCH can be made idempotent (set email to X) or not (increment counter). Design accordingly.

**Idempotency is critical for retry safety.** If a client retries a POST (network hiccup), it should not create duplicate bookings. Solutions:
- Use an **idempotency key** sent by the client in the header (`Idempotency-Key: <uuid>`)
- Server stores the key and returns the cached response on duplicate requests
- **Store idempotency keys for a bounded window only** (typically 24h). Storing them forever grows the table unboundedly. After the window, a repeated key is treated as a new request. Stripe uses 24h; document your chosen window in API contracts.

### Passing Data

**Path parameters** — identify a specific resource (required):
```
GET /events/123          # 123 identifies the event
DELETE /bookings/456
```

**Query parameters** — filter, sort, paginate (optional):
```
GET /events?city=NYC&date_from=2025-01-01&sort=date&page=2&limit=20
```

**Request body** — structured payload for create/update:
```http
POST /bookings
Content-Type: application/json

{
  "event_id": "123",
  "tickets": [{"section": "VIP", "quantity": 2}],
  "payment_method_id": "pm_abc"
}
```

**Rule of thumb:** required to identify → path; optional modifier → query string; complex data → body.

### Returning Data

**Status codes:**

| Code | Meaning |
|------|---------|
| 200 OK | Successful GET/PATCH/DELETE |
| 201 Created | Successful POST (include `Location: /resource/{id}` header) |
| 204 No Content | Successful DELETE with no response body |
| 400 Bad Request | Client sent malformed data |
| 401 Unauthorized | Not authenticated |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | State conflict (duplicate idempotency key, optimistic lock failure) |
| 422 Unprocessable | Validation error |
| 429 Too Many Requests | Rate limited |
| 500 Server Error | Unexpected server fault |
| 503 Service Unavailable | Overloaded or in maintenance |

**Response body conventions:**
- Return the created/updated resource in the body (saves a follow-up GET)
- For errors, return a structured body: `{ "error": "validation_failed", "message": "...", "field": "email" }`
- For lists, include pagination metadata

### Pagination

Three main patterns:

**Offset pagination** — simple, but breaks under concurrent inserts and expensive on deep pages:
```
GET /events?offset=200&limit=20
```

**Cursor pagination** — stable under inserts, efficient, but can't jump to arbitrary pages:
```
GET /events?cursor=eyJpZCI6MTIzfQ&limit=20
# Response includes: { "next_cursor": "eyJpZCI6MTQzfQ", "items": [...] }
```

**Keyset pagination** — variant of cursor using the last seen value of a sort key:
```
GET /events?after_id=143&limit=20
```

**Interview recommendation:** propose cursor/keyset for any high-volume list endpoint. Offset pagination is fine for admin UIs or low-traffic endpoints.

### Versioning

APIs change. Two main strategies:

**URL versioning** (most common in practice):
```
/v1/events
/v2/events
```

**Header versioning** (cleaner URLs, harder to test in browser):
```
Accept: application/vnd.myapp.v2+json
```

For staff-level interviews, mentioning deprecation strategy matters: maintain old versions for N months, communicate sunset dates, return `Sunset` headers.

---

## GraphQL

### When to Use It

GraphQL excels when:
1. Multiple clients (iOS, Android, Web, partners) need overlapping but different data
2. The frontend team moves faster than the backend team
3. You want a single endpoint with a rich query language

### Schema-First Design

Define types first:

```graphql
type Event {
  id: ID!
  name: String!
  date: DateTime!
  venue: Venue!
  tickets(section: String): [Ticket!]!
  availableCount: Int!
}

type Venue {
  id: ID!
  name: String!
  city: String!
}

type Ticket {
  id: ID!
  section: String!
  price: Float!
  available: Boolean!
}

type Query {
  event(id: ID!): Event
  events(city: String, dateFrom: String, limit: Int, after: String): EventConnection!
}

type Mutation {
  createBooking(input: BookingInput!): Booking!
  cancelBooking(id: ID!): Booking!
}
```

### The N+1 Problem and DataLoader

Naive GraphQL resolvers cause N+1 queries:

```
# Client asks for 10 events with their venues
# → 1 query for events
# → 10 separate queries for venues (one per event)
```

**Solution — DataLoader (batching + caching per request):**

```
# DataLoader collects all venue IDs within a single tick
# → 1 query: SELECT * FROM venues WHERE id IN (1,2,3,...10)
```

Mention DataLoader in any GraphQL interview discussion. It's the expected answer.

### Subscriptions (Real-Time)

GraphQL Subscriptions use WebSocket to push updates to clients:

```graphql
subscription {
  ticketAvailabilityChanged(eventId: "123") {
    section
    available
    price
  }
}
```

---

## gRPC

### When to Use It

- Internal microservice mesh with high call volume
- Need bidirectional streaming (e.g., log aggregation, real-time ML inference)
- Strongly typed contracts enforced at compile time

### Proto Definition

```protobuf
syntax = "proto3";

service InventoryService {
  rpc GetStock(StockRequest) returns (StockResponse);
  rpc WatchStock(StockRequest) returns (stream StockUpdate);
}

message StockRequest {
  string product_id = 1;
}

message StockResponse {
  string product_id = 1;
  int32  quantity = 2;
  string updated_at = 3;
}

message StockUpdate {
  string product_id = 1;
  int32  delta = 2;
}
```

### Streaming Modes

| Mode | Pattern | Example Use Case |
|------|---------|-----------------|
| Unary | req → resp | Standard RPC call |
| Server streaming | req → stream | Tail logs, price feed |
| Client streaming | stream → resp | Bulk upload |
| Bidirectional | stream ↔ stream | Chat, telemetry ingestion |

### gRPC vs REST Comparison

| Dimension | REST | gRPC |
|-----------|------|------|
| Protocol | HTTP/1.1 (common) | HTTP/2 |
| Payload | JSON (~text) | Protobuf (~binary, 3–10× smaller) |
| Typing | Optional (OpenAPI) | Required (proto schema) |
| Code generation | Optional | Built-in (stubs for 10+ languages) |
| Browser support | Native | Needs grpc-web proxy |
| Learning curve | Low | Medium |
| Latency | Higher (JSON parse) | Lower (binary, multiplexed) |

---

## Rate Limiting

Every public API endpoint needs rate limiting. Without it, one misbehaving client can saturate your servers.

### Algorithms

**Token Bucket** — each client has a bucket of tokens that refills at a fixed rate. Each request consumes one token. Allows bursts up to bucket capacity.

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/sec

Client sends 100 requests instantly → all served (bucket empties)
Client sends 1 more → rejected (429)
After 1 second → 10 tokens refilled → next 10 requests served
```

**Leaky Bucket** — requests enter a queue and are processed at a fixed rate. Smooths bursts into a steady output rate. Good for protecting downstream services from spikes.

**Fixed Window Counter** — count requests per time window (e.g., 1000/min). Simple but has a boundary problem: a client can send 1000 at 12:00:59 and 1000 at 12:01:00, getting 2000 requests in 2 seconds.

**Sliding Window Log** — track timestamps of each request. More accurate, higher memory cost.

**Sliding Window Counter** — hybrid: combines fixed window counts with a weighted fraction of the previous window. Accurate and memory-efficient. Used by Cloudflare and most production rate limiters.

### Implementation

Store counters in Redis with TTL-based expiration:

```python
# Sliding window counter in Redis
key = f"rate:{user_id}:{window_start_minute}"
count = redis.incr(key)
redis.expire(key, 120)  # 2 minutes TTL
if count > LIMIT:
    raise RateLimitExceeded()
```

**Response headers** for rate-limited APIs (follow IETF RFC 6585 / 7235):
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1716220800
Retry-After: 58   # seconds until next request allowed (on 429)
```

### Rate Limit Tiers

Don't apply the same limit everywhere:
- Authenticated users: higher limit
- Premium tier: even higher
- Unauthenticated: lowest (prevent enumeration/scraping)
- Internal services: no limit or very high

---

## Backward Compatibility and API Evolution

APIs are contracts. Breaking changes break clients. At staff level, you're expected to own API evolution without forcing clients to upgrade on your schedule.

### Postel's Law (Robustness Principle)

> Be conservative in what you send, liberal in what you accept.

- Accept extra fields in request bodies without erroring (ignore unknown fields)
- Return all documented fields even if empty/null (don't omit optional fields)
- Never remove or rename a field in a response without a version bump

### Safe vs Breaking Changes

| Change | Safe? | Reason |
|--------|-------|--------|
| Add optional request field | ✅ | Existing clients don't send it |
| Add response field | ✅ | Clients ignore unknown fields |
| Add new endpoint | ✅ | Existing clients don't call it |
| Remove request field | ❌ | Clients may be sending it |
| Remove response field | ❌ | Clients may be reading it |
| Change field type (string→int) | ❌ | Type mismatch |
| Change field semantics | ❌ | Silent data corruption |
| Make optional field required | ❌ | Old clients omit it |

### Deprecation Strategy

1. Add `Deprecation: true` and `Sunset: <date>` response headers
2. Log which clients are calling deprecated endpoints
3. Reach out to high-traffic callers with migration guides
4. After sunset date, return 410 Gone (not 404)

---

### Authentication vs Authorization

- **Authentication** — who are you? (JWT, API key, OAuth token)
- **Authorization** — what can you do? (RBAC, ABAC, ownership checks)

Always validate both at the API layer. Never trust user-supplied IDs without verifying the caller owns that resource.

### Common Mistakes to Call Out in Interviews

1. **Trusting client-supplied user IDs** — extract user from the auth token, not from the request body
2. **Missing rate limiting** — every public API endpoint needs it (return 429 with `Retry-After` header)
3. **Verbose error messages in production** — don't leak stack traces or DB schema info in 500 responses
4. **Over-broad API scopes** — scope tokens to the minimum permissions required

---

## Designing APIs for AI Agent Clients

AI agents calling your API have different needs than human-driven UIs:

| Concern | Recommendation |
|---------|---------------|
| **Discoverability** | Provide OpenAPI spec; agents need to understand available endpoints |
| **Structured errors** | Machine-readable error codes, not just human messages |
| **Idempotency** | Agents retry on failure; all writes must be idempotent |
| **Rate limiting with backoff headers** | Return `Retry-After` so agents can self-throttle |
| **Pagination** | Cursor-based, so agents can iterate large datasets without drift |
| **Webhook / event push** | Agents polling is expensive; push events when state changes |

---

## Interview Template: API Section

A quick format that covers what interviewers want to see in ~5 minutes:

```
1. State your API choice and why:
   "I'll use REST since we have standard CRUD operations and no specialized
    client data-fetching requirements."

2. List the core endpoints (method + path + one-line description):
   POST   /events/{id}/bookings    -- create a booking
   GET    /bookings/{id}           -- retrieve booking
   DELETE /bookings/{id}           -- cancel booking

3. Note one non-obvious decision:
   "Bookings use an Idempotency-Key header so clients can safely retry
    on network failure without double-charging."

4. Call out pagination if relevant:
   "GET /events uses cursor-based pagination to avoid offset instability."
```

---

*Previous: [01 - Networking Essentials](./01-networking-essentials.md)*  
*Next: [03 - Data Modeling](./03-data-modeling.md)*
