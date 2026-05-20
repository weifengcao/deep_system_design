# Pattern 01: Real-time Updates

The **Real-time Updates** pattern addresses the architectural challenge of pushing immediate notifications and data changes from a server to clients as events occur, rather than waiting for the client to ask for them.

---

## 1. The Core Architecture: The Two Hops of Real-Time

When designing a real-time system, think of the system as having **two distinct communication hops**:

1.  **Hop 1: Source to Server (Ingestion):** How the system receives the event or change from the source (e.g., a driver's GPS device posting coordinate updates, or a chat client writing a message).
2.  **Hop 2: Server to Client (Push):** How the server routes and pushes that update to the specific active user(s) listening for it.

```
+---------------+     Hop 1 (Ingestion)      +------------------+     Hop 2 (Push)      +----------------+
| Ingestion Src |  =======================>  | Stateful Servers |  ==================>  | Active Clients |
| (e.g. Writer) |    Stateless REST/gRPC     |  (Gateway Nodes)  |  WebSockets/SSE/Poll |  (Web/Mobile)  |
+---------------+                            +------------------+                       +----------------+
```

---

## 2. Client-Server Connection Protocols (Hop 2 Options)

The choice of protocol determines how the server pushes data to the client. Let's explore the 5 primary options.

### A. Simple Polling (Short Polling)
*The Baseline (Pull instead of Push)*

The client repeatedly sends traditional HTTP requests to the server on a fixed schedule (e.g., every 5 seconds).

```mermaid
sequenceDiagram
    autonumber
    participant client as Client
    participant server as Server
    client->>server: "GET /notifications (HTTP/1.1)"
    server-->>client: "200 OK (No new notifications)"
    Note over client,server: Sleep 5 seconds
    client->>server: "GET /notifications (HTTP/1.1)"
    server-->>client: "200 OK (1 new notification!)"
```

*   **Trade-offs:**
    *   **Pros:** Extremely simple; stateless; works natively with standard HTTP/1.1 and Layer 7 load balancers; no persistent connections to maintain.
    *   **Cons:** Enormous resource waste (thousands of empty requests/responses); high database load; high latency (up to the sleep interval).
    *   **Resource Equation:**
        $$\text{Total Requests/sec} = \frac{\text{Active Users}}{\text{Polling Interval (seconds)}}$$
        *Example: 10,000 active users polling every 1 second = 10,000 req/sec hitting your backend, even if 99% return no data!*

---

### B. Long Polling
*The "Hanging GET" (Semi-Push)*

The client sends an HTTP request, but the server **holds the request open** (hangs) until new data is available or a timeout occurs. Once the client receives a response, it immediately opens a new request.

```mermaid
sequenceDiagram
    autonumber
    participant client as Client
    participant server as Server
    client->>server: "GET /notifications (HTTP/1.1)"
    Note over server: No updates. Hold request open...
    Note over server: Event occurs!
    server-->>client: "200 OK (Notification Data)"
    client->>server: "GET /notifications (HTTP/1.1)"
    Note over server: Hold request open...
```

*   **Trade-offs:**
    *   **Pros:** Lower latency than short polling; fallback protocol for environments blocking WebSockets.
    *   **Cons:** Server-side resource drain (each open request holds an operating system thread or file descriptor); complicated timeout and retry logic; browser limits (maximum 6 connections per domain).

---

### C. Server-Sent Events (SSE)
*The Unidirectional Stream*

A persistent, one-way connection from server to client over standard HTTP. The client initiates a connection, and the server keeps it open, sending data in a lightweight, text-based stream (`text/event-stream`).

```mermaid
sequenceDiagram
    autonumber
    participant client as Client
    participant server as Server
    client->>server: "GET /stream (Accept: text/event-stream)"
    server-->>client: "200 OK (Connection established)"
    Note over client,server: Persistent Unidirectional HTTP/2 Connection
    server->>client: "data: {msg: New notification!}"
    server->>client: "data: {msg: Another update!}"
```

*   **Trade-offs:**
    *   **Pros:** Lightweight and simple to implement; built-in automatic browser reconnection with last-event IDs; works over standard HTTP/2 (allowing multiplexing); built-in backpressure.
    *   **Cons:** Unidirectional only (client cannot send data back over this connection); limited browser support outside of web environments (e.g., standard iOS/Android sockets require custom wrappers); vulnerable to HTTP/1.1 connection exhaustion limits if HTTP/2 is not supported.

---

### D. WebSockets
*The Full-Duplex Champion*

A persistent, bidirectional, full-duplex connection between client and server. It starts as an HTTP request and "upgrades" to a stateful WebSocket TCP connection.

```mermaid
sequenceDiagram
    autonumber
    participant client as Client
    participant server as Server
    client->>server: "GET /ws (Upgrade: websocket)"
    server-->>client: "101 Switching Protocols"
    Note over client,server: Stateful Bidirectional TCP Tunnel
    client->>server: "ping (Keep-alive)"
    server-->>client: "pong"
    client->>server: "User typing..."
    server->>client: "Message received"
```

*   **Trade-offs:**
    *   **Pros:** Ultra-low latency; bidirectional communication; highly efficient framing (1-2 bytes overhead per message vs. hundreds of bytes of HTTP headers).
    *   **Cons:** Statefulness prevents simple horizontal scaling; requires Layer 4 load balancing (TCP stream routing); no automatic reconnection (must be implemented in client code); bypasses standard HTTP caching and security middleware.

---

### E. WebRTC
*The Peer-to-Peer Solution*

Direct browser-to-browser (peer-to-peer) communication. A server is only required during signaling (initial connection setup) and STUN/TURN traversal (bypassing NAT/firewalls).

```mermaid
sequenceDiagram
    autonumber
    participant clientA as Client A
    participant sig as Signaling Server
    participant clientB as Client B
    clientA->>sig: "Send SDP Offer"
    sig->>clientB: "Forward SDP Offer"
    clientB->>sig: "Send SDP Answer"
    sig->>clientA: "Forward SDP Answer"
    Note over clientA,clientB: P2P UDP Connection Established
    clientA->>clientB: "Direct media/data streaming (Bidirectional)"
```

*   **Trade-offs:**
    *   **Pros:** Lowest possible latency (no intermediate server hop); zero server bandwidth cost for raw media streams.
    *   **Cons:** High implementation complexity; hard to scale for multi-user scenarios (requires Selective Forwarding Units (SFUs)); firewalls and symmetric NATs block direct connections in up to 20% of cases, requiring expensive TURN relay servers.

---

## 3. Protocol Comparison Cheat Sheet

| Metric | Simple Polling | Long Polling | Server-Sent Events (SSE) | WebSockets | WebRTC |
|---|---|---|---|---|---|
| **Direction** | Client $\rightarrow$ Server | Client $\rightarrow$ Server | Server $\rightarrow$ Client | Bidirectional | Peer-to-Peer |
| **Protocol** | Standard HTTP | Standard HTTP | Standard HTTP (HTTP/2) | Custom TCP (`ws://`) | UDP Data Channel |
| **Latency** | High | Medium | Low | Ultra-Low | Extreme Low |
| **Scaling Complexity** | Trivial | Easy | Hard (Stateful) | Very Hard (Stateful) | Extreme (Signaling/STUN/TURN) |
| **Reconnection** | Native | Manual | Native | Manual | Manual |
| **Ideal Use Case** | Trivial dashboard | Fallback socket | Tickers, live score feeds, ChatGPT streams | Chat, multiplayer gaming, collaborative editor | Video/audio calls, P2P file sharing |

---

## 4. Server-Side Push Routing & Scale (The Complex Part)

If you select a stateful protocol (WebSockets or SSE), you cannot scale horizontally by simply adding stateless server instances. If **Client A** is connected to **Server 1**, and **Server 2** receives the message intended for **Client A**, how does the message get routed to the correct server?

### Pattern A: Pushing via Pub/Sub (Broker Fan-Out)
We use a message broker (e.g., Redis Pub/Sub, Kafka, or RabbitMQ) to broadcast messages across all stateful server nodes.

1.  Each server subscribes to a shared message broker cluster.
2.  When a write occurs, the message is published to a channel (e.g., `user_123_channel`).
3.  Every server receives the broadcast, but **only** the server that holds the active TCP connection to `user_123` pushes the update down the socket.

```mermaid
flowchart TD
    Write[Write Path API] -->|Publish Msg to channel| Broker[Redis Pub/Sub]
    Broker -->|Broadcast| G1[Gateway Server 1]
    Broker -->|Broadcast| G2[Gateway Server 2]
    
    G1 -.->|No active connection| C1[Client A]
    G2 ==>|Active TCP Connection| C2[Client B]
    
    style G2 stroke:#299a8d,stroke-width:3px
    style C2 stroke:#299a8d,stroke-width:3px
```

*   **Pros:** Simple to implement; highly decoupled.
    *   **Cons:** "O(N * M)" scaling bottleneck where $N$ is the number of servers and $M$ is the number of messages. As servers grow, broadcasting wastes substantial server-to-server bandwidth.

---

### Pattern B: Pushing via Consistent Hash Routing
Instead of broadcasting to all servers, we route messages directly to the specific server holding the connection, using a Consistent Hash Ring (based on `user_id`).

1.  A Consistent Hash Ring assigns both gateway servers and `user_id`s to a 360-degree virtual ring.
2.  Both the client (when establishing a WebSocket connection) and the write-path service (when routing messages) use the same hash ring to locate the target Gateway Server.

```mermaid
flowchart TD
    HashRing{Consistent Hash Ring}
    ClientA[Client A ID] -->|Hash to Ring| HashRing
    Write[Write API] -->|Target User ID| HashRing
    
    HashRing -->|Route| G1[WebSocket Server 1]
    HashRing -->|Route| G2[WebSocket Server 2]
```

*   **Pros:** Zero broadcast overhead; highly efficient point-to-point routing ($O(1)$).
    *   **Cons:** Server node additions or removals trigger connection rebalancing, which can drop sockets and cause reconnection storms.

---

## 5. Common Deep Dives & Edge Cases

### Q1: How do you handle connection failures and reconnection storms?
When a stateful WebSocket/SSE server restarts or crashes, millions of clients will disconnect simultaneously. If they all immediately try to reconnect, they will trigger a **Distributed Denial of Service (DDoS)** attack on your gateway servers (a **Reconnection Storm**).
*   **The Solution:**
    1.  **Exponential Backoff with Jitter:** Clients must delay reconnection attempts using an exponentially increasing delay, modified by a random value (jitter) to distribute the load over time:
        $$T_{\text{wait}} = 2^{\text{attempt}} \times \text{Base Delay} + \text{Random Jitter}$$
    2.  **L4 Load Balancer Connection Throttling:** Place rate limiters at the TCP layer to reject bursty connection attempts gracefully.
    3.  **State Recovery Buffers:** Use client-side event sequences (e.g., `last_event_id`) and keep a short-term message log in Redis so clients can fetch missed updates immediately after reconnecting.

### Q2: What happens when a single user has millions of followers (The Celebrity Problem)?
If a celebrity like Cristiano Ronaldo posts an update, sending real-time pushes to 500 million followers simultaneously will crush the gateway infrastructure (a massive **Fan-Out** bottleneck).
*   **The Solution:**
    *   **Hybrid Push/Pull Model:** 
        *   **Standard Users:** Use push channels (WebSockets/SSE) for immediate delivery.
        *   **Celebrity Followers:** Do **not** push the update. Instead, when the active follower opens their feed, have the client fetch (pull) the celebrity update via an HTTP GET request, leveraging cached CDN responses.

### Q3: How do you maintain message ordering across distributed servers?
In a distributed real-time chat application, packets may take different network paths and arrive out of order (e.g., a reply arriving before the question).
*   **The Solution:**
    1.  **Distributed Sequencer:** Generate unique, monotonically increasing message IDs using a centralized system like **Snowflake IDs** or a Redis atomic counter per channel.
    2.  **Client-Side Buffering:** Have clients buffer incoming messages and sort them by their sequence ID or logical clock (e.g., Lamport Timestamps) rather than the physical time the packet was received.
