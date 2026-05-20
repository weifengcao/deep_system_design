# Networking Essentials for System Design Interviews

> **Target audience:** Staff+ backend engineers preparing for system design interviews  
> **Covers:** OSI model, TCP/UDP, HTTP/HTTPS, REST, GraphQL, gRPC, DNS, load balancers, CDN

---

## Why Networking Matters

Virtually every system you design is a collection of independent processes communicating over a network. Understanding how that communication works — what guarantees it provides, what costs it introduces — lets you make sharper trade-off decisions in interviews and on the job.

Infrastructure and distributed-systems interviews probe this most deeply, but every system design benefits from knowing the fundamentals.

---

## The Layered Model (OSI)

Networks are built in layers. Each layer abstracts away the complexity below it, so application developers can reason about requests and responses without worrying about voltages on wires.

```
┌───────────────────────────────────────────┐
│  Layer 7 – Application (HTTP, DNS, WS)    │
├───────────────────────────────────────────┤
│  Layer 4 – Transport (TCP, UDP, QUIC)     │
├───────────────────────────────────────────┤
│  Layer 3 – Network (IP)                   │
├───────────────────────────────────────────┤
│  Layer 2 – Data Link (Ethernet, WiFi)     │
├───────────────────────────────────────────┤
│  Layer 1 – Physical (cables, radio)       │
└───────────────────────────────────────────┘
```

For system design interviews, three layers dominate: **Network (L3)**, **Transport (L4)**, and **Application (L7)**.

### Layer 3 — Network (IP)

IP handles routing and addressing. Every internet-reachable device has a **public IP address** allocated by a Regional Internet Registry. The internet's routing infrastructure knows where these address blocks live and routes packets accordingly.

### Layer 4 — Transport (TCP, UDP, QUIC)

Transport protocols provide end-to-end communication between applications. They decide what guarantees to offer on top of IP's best-effort delivery.

### Layer 7 — Application (HTTP, DNS, WebSocket …)

Application protocols define how applications communicate. They run in user-space and are the layer most developers actually touch.

---

## A Concrete Walk-Through: HTTP over TCP

When you type `https://example.com` into a browser:

```
Client                          DNS Server
  │──── DNS query (example.com?) ────▶│
  │◀─── IP address returned ──────────│

Client                          Server (resolved IP)
  │──── SYN ──────────────────────────▶│   TCP
  │◀─── SYN-ACK ───────────────────────│   handshake
  │──── ACK ──────────────────────────▶│   (3-way)
  │                                     │
  │──── TLS ClientHello ───────────────▶│   TLS
  │◀─── TLS ServerHello + Cert ─────────│   handshake
  │──── TLS Finished ─────────────────▶│
  │                                     │
  │──── HTTP GET /index.html ──────────▶│   Application
  │◀─── HTTP 200 OK + body ─────────────│
  │                                     │
  │──── FIN ──────────────────────────▶│   TCP teardown
  │◀─── FIN-ACK ────────────────────────│
```

**Three takeaways that matter for interviews:**

1. **Stateful connections have overhead.** Every new TCP connection costs a round-trip before data can flow. Features like HTTP keep-alive and HTTP/2 multiplexing amortize that cost.
2. **Higher layers = more latency.** An L4 load balancer that routes by IP/port is faster than an L7 load balancer that parses HTTP headers — but less flexible.
3. **One logical request, many packets.** DNS, TCP handshake, TLS negotiation, and data transfer are all separate roundtrips. Distance and packet loss amplify this.

---

## Transport Layer: TCP vs UDP

### TCP — Reliable, Ordered, Stateful

TCP provides a **stream** abstraction: bytes you write on one end arrive in the same order at the other end, with retransmission on loss.

| Property | Value |
|---|---|
| Connection setup | 3-way handshake |
| Delivery guarantee | Yes (retransmit on loss) |
| Ordering | Yes |
| Flow control | Yes (receiver window) |
| Congestion control | Yes (slow-start, AIMD) |
| Header overhead | 20–60 bytes |

**Use TCP for:** APIs, databases, file transfer — anything where correctness matters.

### UDP — Fast, Stateless, Unreliable

UDP adds almost nothing to IP: a source port, destination port, length, and checksum. No connection, no retransmit, no ordering.

| Property | Value |
|---|---|
| Connection setup | None |
| Delivery guarantee | No |
| Ordering | No |
| Header overhead | 8 bytes |

**Use UDP for:** live video/audio, online gaming, DNS, IoT telemetry — cases where losing a packet is better than waiting for a retransmit.

> **Browser note:** Browsers don't support raw UDP beyond WebRTC. If you propose a UDP-based design, think about the browser fallback path.

### QUIC — Modern Replacement for TCP+TLS

QUIC (used by HTTP/3) runs over UDP but re-implements TCP's reliability in user-space. It eliminates TCP's head-of-line blocking and reduces handshake RTTs by combining TLS 1.3 negotiation. Increasingly common at the edge (Cloudflare, Google), but most internal microservice traffic still uses TCP.

### Decision Guide

```
Is data loss catastrophic (APIs, DB, file sync)?
    ├── YES → TCP
    └── NO → Is latency more important than reliability?
                 ├── YES → UDP (games, live streams, VoIP, telemetry)
                 └── MAYBE → QUIC/HTTP3 if modern stack is available
```

---

## Application Layer Protocols

### DNS

DNS translates human-readable names (e.g., `api.example.com`) into IP addresses. A full resolution chain: browser cache → OS cache → recursive resolver → root nameserver → TLD nameserver → authoritative nameserver.

**TTL matters for system design.** Low TTLs (60s) enable fast failover but increase DNS query load. High TTLs reduce load but make traffic shifting slow during incidents.

### HTTP / HTTPS

HTTP is a stateless request-response protocol. Each request carries a method, URL, headers, and optional body; each response carries a status code, headers, and body.

**Key methods and their semantics:**

| Method | Idempotent? | Safe? | Use |
|--------|------------|-------|-----|
| GET | Yes | Yes | Fetch resource |
| POST | No | No | Create resource |
| PUT | Yes | No | Replace resource |
| PATCH | No | No | Partial update |
| DELETE | Yes | No | Remove resource |

**Common status codes to know:**

| Range | Meaning | Examples |
|-------|---------|---------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Permanent, 302 Temporary, 304 Not Modified |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server error | 500 Internal Error, 502 Bad Gateway, 503 Service Unavailable |

**HTTP/1.1 → HTTP/2 → HTTP/3 evolution:**

```
HTTP/1.1  One request per TCP connection (keep-alive helps, but head-of-line blocking remains)
HTTP/2    Multiplexed streams on one TCP connection; header compression (HPACK)
HTTP/3    Multiplexed streams over QUIC (UDP-based); no TCP head-of-line blocking
```

HTTPS adds TLS on top of TCP. Contents are encrypted in transit, but your API should still validate all input — an attacker can forge valid HTTPS requests.

---

## API Design Patterns

### REST

REST models your system as **resources** identified by URLs and manipulates them with HTTP methods. It maps naturally onto your core entities.

```
GET    /users/{id}              → return user
POST   /users                   → create user
PUT    /users/{id}              → replace user
PATCH  /users/{id}              → partial update
DELETE /users/{id}              → delete user
GET    /users/{id}/posts        → list user's posts
POST   /events/{id}/bookings    → book tickets for event
```

**Rule of thumb:** path parameters identify a specific resource (required); query parameters filter, paginate, or sort (optional).

**When to default to REST:** Most interviews. It's well-understood, tooling is excellent, and it works for 90%+ of use cases.

### GraphQL

GraphQL exposes a single endpoint that accepts typed queries. Clients specify exactly which fields they need, eliminating over-fetching and under-fetching.

```graphql
query {
  event(id: "123") {
    name
    date
    venue { name, address }
    tickets { section, price, available }
  }
}
```

**Choose GraphQL when:** diverse clients (mobile, web, partners) need different data shapes, or the frontend team needs to iterate without backend API changes.

**Watch out for:** the N+1 problem — a list query that fetches 100 events may issue 100 sub-queries for venues unless you use a DataLoader/batching pattern.

### gRPC

gRPC uses Protocol Buffers (binary) and HTTP/2. It's designed for **internal service-to-service** communication where throughput and latency matter.

```proto
service PermissionService {
  rpc CheckPermission(PermissionRequest) returns (PermissionResponse);
}
```

**Choose gRPC when:** microservice mesh with high call volume, you need bi-directional streaming, or you want strongly-typed contracts with codegen.

**Tradeoff:** browser support requires a proxy (grpc-web). Not a good fit for public-facing APIs.

### Comparison Table

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol | HTTP/1.1+ | HTTP/1.1+ | HTTP/2 |
| Payload format | JSON | JSON | Protobuf (binary) |
| Typing | Informal (OpenAPI) | Strong (schema) | Strong (proto) |
| Streaming | SSE / WebSocket (separate) | Subscriptions | Native bidirectional |
| Browser support | Full | Full | Needs proxy |
| Best for | Public APIs, CRUD | Flexible queries, multi-client | Internal microservices |

---

## Real-Time Protocols

Standard HTTP is request-response; for server-pushed updates you need a different primitive.

### Long Polling

Client makes a request; server holds it open until an event occurs (or timeout), then responds. Simple to implement, high overhead.

### Server-Sent Events (SSE)

Server streams newline-delimited events over a persistent HTTP connection. Unidirectional (server → client). Good for feeds, live dashboards.

### WebSockets

Full-duplex TCP connection upgraded from HTTP. Suitable for chat, collaboration, real-time gaming. Stateful — every WebSocket connection must live on a specific server (or use a pub/sub broker to fan out).

**Critical: clients A and B connect to *different* WebSocket servers.** The pub/sub broker (Redis) is what bridges them. A common mistake is drawing both clients hitting the same server — at scale there are hundreds of WS servers and the broker decouples them.

```
Sequence: Chat message delivery (two different WS servers)

Client A        WS Server 1       Redis Pub/Sub      WS Server 2       Client B
   │──send msg──▶│                      │                  │               │
   │             │──PUBLISH chat:42────▶│                  │               │
   │             │                      │──PUSH to subs───▶│               │
   │             │                      │   (Server 2      │──push msg────▶│
   │             │                      │    subscribed)   │               │
```

**Fan-out problem:** If a chat room has 100,000 subscribers spread across 500 WS servers, one message triggers 500 Redis deliveries. At very high fan-out, a dedicated message bus (Kafka) or a hierarchical pub/sub is needed.

### WebRTC

Peer-to-peer audio/video/data over UDP. Signaling (ICE candidates, SDP) happens over WebSocket or HTTP; media flows directly between peers.

---

## Load Balancers

A load balancer distributes incoming requests across a pool of backend servers. Two main layers:

### L4 (Network) Load Balancer

Routes based on IP and TCP port. Doesn't inspect application data. Very fast (often implemented in hardware or kernel bypass). Use when raw throughput matters more than request-level routing.

### L7 (Application) Load Balancer

Parses HTTP headers, cookies, paths. Enables:
- **Path-based routing** (`/api` → service A, `/web` → service B)
- **Host-based routing** (multi-tenant)
- **Sticky sessions** (route same user to same backend)
- **SSL termination** (decrypt once at the LB)
- **Health check awareness** (drain unhealthy backends)

**Common algorithms:**

| Algorithm | How it works | Best for |
|-----------|-------------|----------|
| Round Robin | Rotate through servers | Homogeneous servers, uniform request cost |
| Least Connections | Route to server with fewest active connections | Variable request duration |
| IP Hash | Hash client IP → consistent server | Sticky sessions without cookies |
| Weighted | Assign more traffic to powerful nodes | Mixed server capacity |

**Connection draining:** When removing a backend (deploy, scale-down), the LB stops sending *new* requests but lets *in-flight* requests finish (typically 30–60s grace period). Without draining, users see mid-request 502s during every deploy.

---

## mTLS and Service Mesh Security

In microservice architectures, TLS between services is not enough — you need to verify *which service* is calling you, not just that the connection is encrypted. This is **mutual TLS (mTLS)**.

```
Service A                          Service B
    │──── ClientHello ─────────────────▶│
    │◀─── ServerHello + Cert (B) ───────│
    │──── Client Cert (A) ──────────────▶│   ← A proves its identity
    │         both sides verified         │
    │──── encrypted traffic ────────────▶│
```

In a service mesh (Istio, Linkerd), mTLS is enforced transparently via sidecar proxies (Envoy). Services get short-lived certificates from a mesh CA. This enables:
- **Zero-trust networking:** no implicit trust between services, even inside the datacenter
- **Fine-grained AuthZ:** "Service A is allowed to call Service B's `/payments` endpoint"
- **Observability:** every service call is traced and metered by the proxy

At staff level, knowing *why* mTLS matters (lateral movement prevention after a breach) is as important as knowing it exists.

---

## CDN (Content Delivery Network)

A CDN is a geographically distributed cache at the edge. Requests are served from the nearest **PoP (Point of Presence)** rather than your origin.

```
User (India) ─────▶ CDN Edge (Mumbai) ─── cache hit ──▶ Response (20ms)
                             │ cache miss
                             └──▶ Origin (Virginia) ──▶ Response (280ms)
                                  (CDN stores copy for next request)
```

**What CDNs cache well:** static assets (JS, CSS, images), video segments, public API responses with Cache-Control headers.

**What they don't:** private user data, highly dynamic responses, POST requests.

**How CDNs route to the nearest PoP — Anycast:** CDNs advertise the same IP address from every PoP via BGP anycast. The internet's routing infrastructure naturally delivers each user's packet to the topologically closest PoP — no DNS tricks needed. This also makes CDNs highly effective at absorbing DDoS: a 1 Tbps attack gets spread across hundreds of PoPs, each absorbing a fraction.

**Invalidation strategies:**
- **TTL-based**: Set `Cache-Control: max-age=3600`. Simple but can serve stale.
- **Versioned URLs**: `/app.abc123.js` — old URL expires naturally; new URL always misses.
- **Purge API**: Explicitly invalidate a path on deploys. Most CDNs support this.

---

## Interview Cheat Sheet

| Scenario | Recommendation |
|----------|---------------|
| Default API protocol | REST over HTTPS |
| Real-time bidirectional | WebSocket + pub/sub broker |
| Server → client updates only | SSE |
| Internal microservice RPC | gRPC |
| Multi-client different data needs | GraphQL |
| Media / live streaming | UDP or QUIC (WebRTC for browser) |
| Static asset delivery | CDN |
| Route traffic to multiple backends | L7 load balancer |
| Ultra-high throughput routing | L4 load balancer |

---

## Key Trade-offs to Articulate in Interviews

1. **TCP vs UDP**: Reliability vs latency. UDP makes sense only when the application layer can tolerate or correct for loss.
2. **L4 vs L7 LB**: Speed vs flexibility. L7 opens the HTTP envelope (cost) but enables smart routing (benefit).
3. **CDN**: Dramatically reduces latency for static/cacheable content. Adds complexity for invalidation and doesn't help with personalized/dynamic data.
4. **REST vs gRPC**: Developer ergonomics and broad compatibility vs maximum throughput and strong typing.
5. **WebSocket vs SSE**: Full-duplex (needed for chat) vs simpler server-push (sufficient for feeds).

---

*Next: [02 - API Design](./02-api-design.md)*
