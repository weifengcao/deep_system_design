# API Gateway Deep Dive for System Design Interviews

> **Target audience:** Staff+ backend engineers  
> **Covers:** What an API gateway does, cross-cutting concerns, request lifecycle, rate limiting, auth offloading, routing, service mesh vs gateway, security, AI agent gateways

---

## What Is an API Gateway?

An API gateway is the **single entry point** for all client traffic into your backend. It sits between clients (browsers, mobile apps, third-party services) and your microservices, handling cross-cutting concerns so individual services don't have to.

```
                    ┌─────────────────────────────────┐
Clients             │          API Gateway             │
                    │                                  │
Browser ───────────►│ • Authentication / Authorization │─────► User Service
Mobile  ───────────►│ • Rate Limiting / Throttling     │─────► Order Service
Partner ───────────►│ • Request Routing                │─────► Payment Service
                    │ • SSL Termination                │─────► Inventory Service
                    │ • Request/Response Transform     │
                    │ • Logging / Tracing / Metrics    │
                    │ • Circuit Breaking               │
                    └─────────────────────────────────┘
```

Without an API gateway, every microservice must implement auth, rate limiting, logging, and SSL — massive duplication. The gateway centralizes these concerns.

---

## Core Responsibilities

### 1. Authentication and Authorization

The gateway validates tokens before requests reach services, so services can trust that incoming requests are authenticated.

```
Client Request:
  GET /orders/123
  Authorization: Bearer eyJhbG...

Gateway:
  1. Validate JWT signature (public key cached locally)
  2. Check token expiry
  3. Extract user_id and scopes from token claims
  4. Forward request with headers: X-User-Id: 42, X-Scopes: read:orders
  5. Service trusts these headers (only the gateway can set them)
```

**AuthZ options:**
- Gateway enforces coarse-grained rules ("is this token valid?")
- Service enforces fine-grained rules ("does this user own this order?")
- OPA (Open Policy Agent) integration for policy-as-code AuthZ at the gateway

### 2. Rate Limiting and Throttling

Centralized rate limiting protects all downstream services:

```
Per-IP limit:        1000 req/min (unauthenticated)
Per-user limit:      5000 req/min (authenticated)
Per-endpoint limit:  /search: 100 req/min (expensive query)
Global limit:        100K req/sec total (protect entire backend)
```

Implemented via Redis counters shared across gateway instances (see Core Concepts article 04 for algorithms).

Response headers:
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1716220800
```

### 3. Request Routing

Route requests to the correct backend service based on path, host, headers, or method:

```yaml
# Kong / Nginx-style routing rules
routes:
  - path: /api/v1/users
    service: user-service:8080
    
  - path: /api/v1/orders
    service: order-service:8080
    
  - path: /api/v2/orders      # v2 routes to new service
    service: order-service-v2:8080
    
  - host: partner.example.com
    service: partner-api:8080
```

**Advanced routing:**
- **Canary deployments:** Route 5% of traffic to v2, 95% to v1
- **A/B testing:** Route based on user segment or feature flag header
- **Blue/green:** Instant traffic switch between two environments
- **Weighted routing:** Distribute across multiple instances by weight

### 4. SSL/TLS Termination

The gateway decrypts HTTPS at the edge. Backend services communicate over plain HTTP inside the VPC (acceptable) or the gateway re-encrypts for backend (Option C TLS).

```
Internet → [Gateway: TLS termination] → [Services: HTTP or mTLS]
```

Certificate management: integrate with Let's Encrypt, AWS ACM, or Vault PKI for auto-renewal.

### 5. Request/Response Transformation

```
Client sends: GET /users/42?include_orders=true
Gateway transforms:
  → Calls: GET /users/42       (user-service)
  → Calls: GET /orders?user=42 (order-service)
  → Merges responses into single JSON
  → Returns combined response to client
```

This BFF (Backend for Frontend) pattern aggregates multiple service calls into one client request. The gateway handles orchestration, not the individual services.

### 6. Observability

All traffic flows through the gateway → ideal choke point for:
- **Distributed tracing:** Inject trace ID header (`X-Trace-Id`), propagate through services
- **Metrics:** Request rate, error rate, latency percentiles per endpoint and service
- **Access logs:** Every request logged with user ID, endpoint, status, duration

### 7. Circuit Breaking

If a downstream service is failing, the gateway stops routing to it:

```
States:
  CLOSED: requests flow through (normal)
  OPEN:   requests fail fast with 503 (service is broken)
  HALF-OPEN: let one request through; if it succeeds, CLOSE; if fail, stay OPEN
```

Prevents cascade failures: a slow service won't exhaust thread pools and bring down the gateway.

---

## Request Lifecycle

```
Client Request
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                    API Gateway                       │
│                                                     │
│  1. SSL termination                                  │
│  2. Parse request headers                           │
│  3. Rate limit check (Redis counter)                │
│  4. Auth validation (JWT verify / API key lookup)   │
│  5. Route matching (which service?)                 │
│  6. Request transformation (headers, body)          │
│  7. Circuit breaker check (is service healthy?)     │
│  8. Forward to backend service                      │
│  9. Timeout enforcement                             │
│  10. Response received                              │
│  11. Response transformation                        │
│  12. Log request + metrics                          │
│  13. Return response to client                      │
└─────────────────────────────────────────────────────┘
      │
      ▼
Client Response
```

Latency added by gateway: 1–5ms for a well-tuned gateway. Worth the trade-off for all the concerns centralized.

---

## API Gateway Products

| Product | Type | Best For |
|---------|------|---------|
| **Kong** | Self-hosted / Cloud | High flexibility, plugin ecosystem |
| **AWS API Gateway** | Managed | AWS-native, serverless integration |
| **Nginx** | Self-hosted | High-performance reverse proxy, simple routing |
| **Envoy** | Self-hosted | Service mesh, programmable, xDS |
| **Apigee** | Managed | Enterprise API management, analytics |
| **Traefik** | Self-hosted | Kubernetes-native, auto-discovery |
| **Cloudflare Workers** | Edge | Global edge routing, custom logic |

---

## API Gateway vs Service Mesh

A common interview confusion:

| | API Gateway | Service Mesh (Istio/Linkerd) |
|--|------------|------------------------------|
| **Traffic** | North-South (external → internal) | East-West (service → service) |
| **Concerns** | Auth, rate limiting, routing, SSL | mTLS, observability, circuit breaking, retry |
| **Where** | At the perimeter | Sidecar on every pod |
| **Clients** | External users, partners, apps | Internal microservices |
| **Used together?** | Yes — complementary | Yes |

**In practice:** Use both. API Gateway for external traffic; service mesh for internal service-to-service traffic.

---

## Gateway for AI Agents

AI API gateways are an emerging pattern for managing LLM traffic:

### LLM Cost and Rate Limit Management
```
Agent Request → AI Gateway → LLM Provider (OpenAI, Anthropic, etc.)

Gateway responsibilities:
  • Per-user/per-tenant token budget enforcement
  • Rate limiting to LLM provider APIs (RPM, TPM limits)
  • Cost tracking per request (input tokens × rate + output tokens × rate)
  • Model routing: cheap model first, fallback to expensive
  • Retry with exponential backoff on 429/503
  • Response caching (exact match cache for repeated prompts)
```

### Prompt Logging and Audit
```
All LLM requests/responses logged for:
  • Compliance (what did the agent say?)
  • Safety review (detect harmful outputs)
  • Debugging (why did the agent take this action?)
  • Training data collection
```

### Multi-Model Routing
```
Router policy:
  Simple queries (< 100 tokens) → GPT-4o-mini ($0.15/1M)
  Complex reasoning → Claude Sonnet ($3/1M)
  Code generation → GPT-4o ($2.50/1M)
  Fallback on failure → alternative provider
```

Products: LiteLLM, Portkey, Kong AI Gateway, AWS Bedrock Gateway.

---

## Security Hardening

- **Never trust headers from clients.** Only gateway-set headers (X-User-Id, X-Scopes) are trusted by services.
- **Validate request size.** Reject oversized bodies before they reach services.
- **WAF integration.** Web Application Firewall at the gateway layer filters SQLi, XSS, SSRF.
- **IP allowlist for admin APIs.** /admin/* routes only reachable from internal IPs.
- **mTLS for service communication.** After TLS termination, gateway uses mTLS to communicate with services.
- **Secrets rotation.** Gateway's auth keys (JWT public keys, API key hashing secrets) should be rotatable without downtime.

---

## Interview Quick Reference

| Scenario | Gateway Solution |
|----------|-----------------|
| "How do services know who the user is?" | Gateway validates JWT, injects X-User-Id header |
| "How do you prevent one user from hammering the API?" | Rate limiting at gateway with Redis counter |
| "How do you deploy a new version safely?" | Canary routing: 5% traffic to v2 |
| "How do you protect against a failing service?" | Circuit breaker at gateway |
| "Multiple services, single client request?" | BFF aggregation at gateway |
| "How does mobile get different data than web?" | Route to different BFF endpoints at gateway |
| "How do you detect abuse?" | Gateway access logs → anomaly detection |

---

*Previous: [03 - Kafka](./03-kafka.md)*  
*Next: [05 - Cassandra](./05-cassandra.md)*
