## 1. What “realtime” really means here

Realtime ≠ “every pixel moves instantly.”

For a drawing app, realtime means:

* **Low-latency propagation of accepted operations** (≈100–500ms perceived)
* **Eventual convergence** across all clients
* **Optimistic UX** even when the network is slow
* **Graceful degradation** when realtime is unavailable

The key design decision:

> **Realtime is a delivery mechanism for ops, not the source of truth.**

---

## 2. High-level architecture (mental diagram)

```text
Client (Web / Mobile)
   |
   |  REST (auth, snapshots, ops fetch)
   |
API Gateway
   |
Drawing Service  <——>  Persistent Store (DB + Object Storage)
   |
   |  publish(ops)
   v
Realtime Service (WebSocket / SSE)
   |
   |  fan-out
   v
Other Clients
```

* **REST APIs** = correctness, durability, recovery
* **WebSocket** = speed and push
* They are complementary, not interchangeable

---

## 3. Why WebSocket (and not only polling)

### WebSocket benefits

* Bidirectional, persistent connection
* Low overhead per message
* Natural fit for collaborative sessions

### Why not *only* WebSocket

* Connections drop
* Mobile backgrounding kills sockets
* Messages can be missed

Hence:

> **WebSocket is best-effort delivery; REST is the safety net.**

---

## 4. Why REST for Writes, WebSocket for Reads

### TL;DR (One Sentence)

We use REST for writing ops because writes must be durable, retryable, and ordered, and WebSocket for subscribing because reads benefit from low-latency push but don't need durability.

### Why REST API for Writing Ops

Writes are high-value operations (they mutate state), so we want strong guarantees.

**REST gives us:**

**Durability**
* Ops are persisted to DB before success is returned
* Survives server crashes or restarts

**Retry + Idempotency**
* Client can safely retry `POST /ops`
* `op_id` ensures exactly-once application

**Clear ordering**
* Server assigns a monotonic revision
* Single source of truth for op order

**Better failure handling**
* HTTP status codes (409, 401, 429)
* Load balancers, retries, timeouts are well understood

> **In short: REST = correctness + safety**

### Why WebSocket for Subscribing to Ops

Subscriptions are read-only, best-effort, and latency-sensitive.

**WebSocket gives us:**

**Low latency**
* Server can push ops immediately
* No polling delay

**Efficient fan-out**
* One op → many connected clients
* Much cheaper than polling

**Bidirectional & real-time**
* Natural for collaboration
* Presence, cursors, live updates

**Ephemeral is OK**
* If a message is missed → client detects revision gap
* Client recovers via REST (`GET /ops?after=R`)

> **In short: WebSocket = speed + UX**

### Why NOT Use WebSocket for Writes?

Because WebSocket lacks guarantees:

* Messages can be dropped
* No built-in retry semantics
* Harder idempotency & ordering
* Mobile backgrounding kills sockets
* Debugging + observability is worse

**Using WS for writes risks:**
* ❌ Lost ops
* ❌ Inconsistent state
* ❌ Complex recovery logic

### Why NOT Use REST for Subscribing?

Because it's inefficient:

* Polling adds latency
* High QPS under load
* Wasteful when nothing changes
* Worse UX for collaboration

### The Key Design Principle (Say This in Interviews)

> "REST is the source of truth; WebSocket is a delivery optimization."

Or:

> "Writes need correctness guarantees; reads need low latency."

### One-Line Mental Model

```text
REST = commit the truth
WS   = broadcast the truth
```

---

## 5. Realtime data flow (step-by-step)

### Step 1: Client opens a drawing

* Fetch metadata (`latest_revision`)
* Fetch snapshot + ops
* Render canvas
* Open WebSocket: `drawing:{id}`

### Step 2: Client makes an edit

* Update canvas locally immediately (optimistic)
* Create ops
* Send ops via REST (`POST /ops`)

  * REST ensures durability & ordering
* **Not** directly through WebSocket

Why?

* WebSocket messages may be lost
* REST provides retries + idempotency

---

### Step 3: Server processes ops

* Validate + authorize
* Assign revision
* Persist to op log
* Publish event:

```json
{
  "drawing_id": "d1",
  "revision": 121,
  "ops": [...]
}
```

---

### Step 4: Realtime service fans out

* Broadcast to all connected clients in that drawing room
* Includes the originating client (so it can confirm)

---

### Step 5: Clients receive remote ops

* Apply ops in revision order
* Update canvas state
* Acknowledge committed ops (clear pending queue)

---

## 6. Presence vs data updates (important distinction)

### Data (must be durable)

* Object create / update / delete
* Goes through **REST + op log**

### Presence (ephemeral)

* Cursor position
* Selection highlight
* “User X is editing text”

Presence can:

* Go directly over WebSocket
* Be dropped safely
* Never persisted

This keeps the system simpler and scalable.

---

## 7. Handling dropped connections

### Detection

* Revision gaps:

  * “Last seen rev = 120, incoming rev = 125”

### Recovery

* Pause rendering
* Fetch missing ops via REST
* Resume realtime stream

This is why revision numbers are critical.

---

## 8. Scaling the realtime layer

### Problem

* Millions of concurrent WebSocket connections
* Hot drawings with many editors

### Solutions

* **Stateless WebSocket servers**
* Shard by `drawing_id`
* Sticky routing at the load balancer
* Redis / Kafka pub-sub for cross-node fan-out

```text
WS Node A  ——\
WS Node B  ——>  Pub/Sub  ——>  All WS nodes
WS Node C  ——/
```

---

## 9. Mobile-specific realities (interview gold)

* App backgrounding → socket closed
* OS may kill process anytime
* Network switches (WiFi ↔ LTE)

Design implications:

* Treat realtime as **opportunistic**
* Always support REST-based catch-up
* Resume from `last_known_revision`

---

## 10. Latency optimization techniques

* Batch pointer-move ops (e.g. 60–100ms)
* Coalesce intermediate updates
* Send semantic ops (move-to, resize-to)
* Viewport-based rendering

This keeps both network and rendering cheap.

---

## 11. Failure modes & guarantees

### Guarantees

* At-least-once delivery via REST
* Exactly-once application via `op_id` dedupe
* Eventual consistency

### Failure scenarios

| Failure           | Handling                     |
| ----------------- | ---------------------------- |
| WS disconnect     | Poll `/ops`                  |
| Server restart    | Clients resync               |
| Message loss      | Revision gap detection       |
| Partial broadcast | REST remains source of truth |

---

## 12. How to say this crisply in interviews (memorize)

> “We treat realtime as a low-latency fan-out channel for server-ordered operations. All mutations go through a durable REST path, and WebSocket is used only to push already-committed ops and ephemeral presence.”

---

## 13. How this maps to tldraw / Figma

* Same fundamental architecture
* tldraw keeps ops extremely small and semantic
* Heavy emphasis on local-first + replay
* Realtime enhances UX but never owns correctness
