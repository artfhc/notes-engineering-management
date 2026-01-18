# WebSocket Subscriptions and Architecture

## High-Level Overview

WebSocket gives you a pipe. The application protocol defines "subscribe", "unsubscribe", and "data".

---

## 1. What WebSocket Itself Does (Transport Only)

### WebSocket Provides

* A persistent TCP connection
* Full-duplex messaging (client ⇄ server)
* Message framing (text / binary)

### WebSocket Does NOT Define

* Subscriptions
* Topics
* Channels
* Message types

**All of that lives on top of WebSocket, in your app protocol.**

---

## 2. Typical Subscription Lifecycle

### Step 1: Open Connection

```text
Client ──(HTTP Upgrade)──> Server
```

Once upgraded, the socket stays open.

### Step 2: Send a Subscription Message

The client explicitly tells the server what it wants.

**Yes — this is usually a distinct message.**

Example (JSON protocol):

```json
{
  "type": "subscribe",
  "channel": "price_updates",
  "symbol": "AAPL"
}
```

**Other common names:**

* `action: "subscribe"`
* `op: "subscribe"`
* `event: "join"`

### Step 3: Server Acknowledges (Optional but Common)

```json
{
  "type": "subscribed",
  "channel": "price_updates",
  "symbol": "AAPL"
}
```

**Why this matters:**

* Confirms success
* Returns a `subscriptionId`
* Allows error handling

### Step 4: Server Pushes Data Messages

Now the server pushes messages without further requests.

```json
{
  "type": "data",
  "channel": "price_updates",
  "symbol": "AAPL",
  "price": 191.42,
  "timestamp": 1700000000
}
```

This continues until:

* Client unsubscribes
* Connection closes
* Server revokes the subscription

### Step 5: Unsubscribe (Optional)

```json
{
  "type": "unsubscribe",
  "channel": "price_updates",
  "symbol": "AAPL"
}
```

---

## 3. Common Message Shapes (Real-World Patterns)

### Pattern A: Explicit Type Field (Most Common)

```json
{ "type": "subscribe", "topic": "orders" }
{ "type": "data", "topic": "orders", "payload": {...} }
```

### Pattern B: Operation-Based (op)

```json
{ "op": "subscribe", "args": ["ticker:AAPL"] }
{ "op": "notify", "data": {...} }
```

### Pattern C: Subscription ID

```json
{ "type": "subscribe", "id": 42, "topic": "chat.room.1" }
{ "type": "event", "id": 42, "payload": {...} }
```

Useful when subscribing to many streams.

---

## 4. Why Subscription & Data Are Separate Messages

Separation gives you:

* **Multiplexing** (many topics over one socket)
* **Selective updates** (only what the client asked for)
* **Backpressure control**
* **Auth + permission checks** per topic
* **Dynamic subscribe/unsubscribe**

Without this, every client would get everything 😬

---

## 5. Examples from Real Systems

### Crypto Exchanges (Binance / Coinbase)

```json
{ "method": "SUBSCRIBE", "params": ["btcusdt@trade"], "id": 1 }
```

### GraphQL Subscriptions

```json
{ "type": "start", "id": "1", "payload": {...} }
```

### Socket.IO

```javascript
socket.emit("join_room", roomId)
socket.on("message", data => ...)
```

(Same concept, different abstraction.)

---

## 6. Mental Model (Important)

Think of WebSocket subscriptions as:

```text
WebSocket = persistent pipe
Protocol = rules for messages
Subscriptions = server-side routing table
```

The server typically keeps:

```text
connection → [subscriptions]
topic → [connections]
```

---

## 7. Minimal Pseudo-Flow

```text
CONNECT
SUBSCRIBE(topic=A)
SUBSCRIBE(topic=B)
← DATA(A)
← DATA(B)
← DATA(A)
UNSUBSCRIBE(A)
← DATA(B)
DISCONNECT
```

---

## 8. When Subscriptions Are NOT Explicit

Some systems:

* **Auto-subscribe based on URL**
  * `wss://example.com/ws/prices/AAPL`
* **Single stream per connection**

This is simpler but does not scale well for many topics.

---

## TL;DR

✔️ **Yes, WebSocket subscription systems almost always have:**

* **Subscription messages** (subscribe / unsubscribe)
* **Data messages** (pushed by server)

**WebSocket itself doesn't define them—the application protocol does.**

---

## The WebSocket Subscription Mental Model (Deep Dive)

### One Sentence Version

> A WebSocket server is basically a message router with a connection table and a subscription table. Everything else is implementation detail.

---

## 9. What Actually Exists on the Server

When clients connect, the server holds in-memory state like this:

### A. Connection Objects

Each open WebSocket connection is an object:

```text
Connection
├── socket
├── userId / auth context
├── subscriptions
├── lastSeen
├── backpressure info
```

**Think:** "This is one browser tab / app instance."

### B. Subscription Registry (The Important Part)

The server typically maintains two linked maps:

**Map 1: Connection → Subscriptions**

```text
conn_123 → [ "prices:AAPL", "orders:open" ]
conn_456 → [ "prices:AAPL" ]
```

Used for:

* Cleanup on disconnect
* Unsubscribe logic
* Permission enforcement

**Map 2: Topic → Connections**

```text
"prices:AAPL" → [ conn_123, conn_456 ]
"orders:open" → [ conn_123 ]
```

Used for:

* Fast fan-out when data arrives

---

## 10. What "Subscribe" Really Does

When the server receives:

```json
{
  "type": "subscribe",
  "topic": "prices:AAPL"
}
```

**The server does not start polling.** It simply updates routing tables.

### Pseudocode

```javascript
subscriptionsByConn[conn].add("prices:AAPL")
connectionsByTopic["prices:AAPL"].add(conn)
```

That's it.

> **Subscribing = "please add me to this routing list"**

---

## 11. What Happens When Data Arrives

Let's say a price update comes from somewhere else:

```text
AAPL price updated → $192.10
```

The WebSocket server does this:

```javascript
for (conn of connectionsByTopic["prices:AAPL"]) {
  conn.send({
    type: "data",
    topic: "prices:AAPL",
    price: 192.10
  })
}
```

**No request. No response. Just routing.**

---

## 12. Why This Model Scales (and Where It Breaks)

### Why It Works Well

* One connection → many topics
* O(1) lookup for fan-out
* Cheap per-message cost

### Where It Breaks

* Millions of topics (memory pressure)
* Hot topics (10k+ connections)
* Slow consumers (backpressure)

This is why systems introduce:

* Sharding by topic
* Pub/Sub brokers (Redis, Kafka)
* Per-topic workers

---

## 13. Unsubscribe & Disconnect (Cleanup Logic)

### Unsubscribe Message

```json
{
  "type": "unsubscribe",
  "topic": "prices:AAPL"
}
```

**Server:**

```javascript
subscriptionsByConn[conn].delete("prices:AAPL")
connectionsByTopic["prices:AAPL"].delete(conn)
```

### Disconnect

When the socket closes:

```javascript
for (topic of subscriptionsByConn[conn]) {
  connectionsByTopic[topic].delete(conn)
}
delete subscriptionsByConn[conn]
```

This is why the `connection → subscriptions` map is mandatory.

---

## 14. Why Subscription IDs Exist

When clients subscribe to many similar streams:

```json
{ "type": "subscribe", "id": 1, "topic": "prices:AAPL" }
{ "type": "subscribe", "id": 2, "topic": "prices:MSFT" }
```

The server stores:

```javascript
conn_123 → {
  1: "prices:AAPL",
  2: "prices:MSFT"
}
```

And sends data like:

```json
{
  "type": "data",
  "id": 1,
  "price": 192.10
}
```

**Why?**

* Smaller payloads
* Faster client routing
* Easier unsubscribe

---

## 15. Authorization Lives in the Mental Model

Auth is not just connection-level.

You usually check on subscribe:

```javascript
if (!canUserSubscribe(conn.user, topic)) {
  conn.send({ type: "error", reason: "forbidden" })
  return
}
```

So the routing table only contains authorized edges.

---

## 16. How This Maps to Pub/Sub Systems

This mental model is identical to:

| System        | Connection | Topic   | Router     |
|---------------|------------|---------|------------|
| WebSocket     | socket     | channel | WS server  |
| Kafka         | consumer   | topic   | broker     |
| Redis PubSub  | client     | channel | redis      |
| GraphQL Sub   | client     | field   | gql server |

WebSocket just keeps it in memory per process.

---

## 17. Common Bugs Explained by This Model

**❌ "User still gets updates after unsubscribe"**
* → Forgot to remove from `connectionsByTopic`

**❌ "Memory leak"**
* → Forgot cleanup on disconnect

**❌ "Wrong user got data"**
* → Topic names too coarse (missing user scoping)

**❌ "Server slows down over time"**
* → One slow connection blocking fan-out

---

## 18. Visual Mental Diagram

```text
          ┌────────────┐
          │ Price Feed │
          └─────┬──────┘
                │
        ┌───────▼────────┐
        │ WS Router       │
        │                 │
        │ prices:AAPL ──┐ │
        │               │ │
        │  conn_1 ◄─────┘ │
        │  conn_2 ◄───────┘
        │                 │
        │ orders:user123 ──► conn_1
        └─────────────────┘
```

---

## 19. The Key Intuition (Remember This)

> **WebSocket subscriptions are just dynamic routing rules. Messages flow through the rules, not through requests.**

---

## Why Systems Introduce Sharding by Topic

### Baseline: No Sharding (Single Routing Domain)

**What it looks like:**

```text
All topics → one routing space → many connections
```

**Pros:**

* Simple mental model
* Easy to implement
* No cross-node coordination

**Cons (why it breaks):**

* ❌ Hot topics overload CPU
* ❌ Large routing tables (memory blow-up)
* ❌ Slow consumers block others
* ❌ One node failure drops everything
* ❌ Can't scale horizontally

This is where sharding becomes inevitable.

---

## Comparison: With vs Without Sharding by Topic

| Dimension            | No Sharding           | Sharding by Topic       |
|----------------------|-----------------------|-------------------------|
| Routing state        | All topics everywhere | Topics split across nodes |
| Memory               | Grows unbounded       | Bounded per shard       |
| CPU fan-out          | Centralized           | Distributed             |
| Hot topic impact     | Global                | Localized               |
| Failure blast radius | Entire system         | Single shard            |
| Horizontal scaling   | Poor                  | Linear                  |
| Ops complexity       | Low                   | Medium                  |
| Rebalancing          | N/A                   | Required                |

---

## Comparison: Sharding by Topic vs Other Scaling Options

### Option 1: Sharding by Connection

```text
Connections split arbitrarily
Topics still global
```

**Why it's worse:**

* Each update must be broadcast to all shards
* Topic fan-out becomes cross-node
* Network amplification
* Good for chat servers
* Bad for high-fan-out topics

### Option 2: Sharding by User

```text
userId % N → shard
```

**Works when:**

* Topics are user-scoped
* Little cross-user overlap

**Breaks when:**

* Global topics exist (prices, live feeds)
* Same topic spans many users

### Option 3: Sharding by Topic ✅

```text
topicHash(topic) → shard
```

**Why systems prefer this:**

* One topic handled by exactly one shard
* Fan-out is local
* Clean ownership
* This matches Kafka partitions, Redis channels, actor systems

---

## Why Topic Sharding Works So Well (Key Intuition)

> **Fan-out cost is per topic, not per user. So you shard the thing that causes fan-out.**

---

## Hot Topic Scenario (Concrete Example)

### Example

* **Topic:** `prices:BTC`
* **Subscribers:** 50,000
* **Updates:** 20/sec

### Without Sharding

* 1 node
* 1M sends/sec
* CPU + GC pressure
* Latency spikes everywhere

### With Topic Sharding

```text
Shard 1 → prices:BTC
Shard 2 → prices:ETH
Shard 3 → prices:SOL
```

**Now:**

* Load isolated
* Other topics unaffected
* Easier autoscaling

---

## Comparison: Static vs Dynamic Topic Sharding

| Aspect              | Static            | Dynamic          |
|---------------------|-------------------|------------------|
| Mapping             | Fixed             | Rebalanced       |
| Complexity          | Low               | Higher           |
| Hot topic handling  | Poor              | Better           |
| Operational cost    | Low               | Medium           |
| Used by             | Simple WS servers | Kafka-like systems |

---

## Why Kafka Is the "Final Form" of Topic Sharding

Kafka:

* Topics → partitions
* Each partition → one consumer
* Ordering guaranteed per shard
* Load scales with partitions

**Topic sharding is the step before Kafka.**

---

## Failure Comparison (Important in Interviews)

| Failure      | No Sharding   | Topic Sharding        |
|--------------|---------------|-----------------------|
| Node crash   | All topics drop | Only shard topics drop |
| Memory leak  | Whole system  | One shard             |
| Backpressure | Global        | Per topic             |

---

## When NOT to Shard by Topic

**Don't shard if:**

* < 10k connections
* Few topics
* Low fan-out
* No "hot" streams

**Sharding adds coordination cost.**

---

## One-Sentence Takeaway (Memorize)

> **Systems shard by topic because fan-out, memory, failures, and backpressure all scale with topics — not connections.**

---

## 20. How WebSocket Subscriptions Map to Pub/Sub Systems

The mental model is identical — only the transport and durability differ.

### Core Mapping

| Concept      | WebSocket                | Pub/Sub (Redis / Kafka) |
|--------------|--------------------------|-------------------------|
| Producer     | backend service / feed   | producer                |
| Broker / Router | WS server process     | Redis / Kafka           |
| Topic        | channel / topic          | channel / topic         |
| Subscriber   | WebSocket connection     | consumer                |
| Subscription | in-memory routing entry  | broker metadata         |
| Delivery     | `conn.send()`            | push / poll             |

### Flow Comparison

**WebSocket (in-memory):**

```text
Producer → WS server → routing table → sockets
```

**Redis Pub/Sub:**

```text
Producer → Redis → subscribers
```

**Kafka:**

```text
Producer → Kafka topic → consumer groups
```

### Same Idea

Data is published to a topic → subscribers receive it

**The difference is where routing state lives:**

* **WebSocket:** memory inside the process
* **Redis/Kafka:** external, shared system

---

## Why People Say "WebSocket Is Just Pub/Sub Over TCP"

Because functionally:

* Topics exist
* Subscribers exist
* Messages are fanned out

But WebSocket is:

* Ephemeral
* Per-process
* Not durable

---

## 21. Why Systems Introduce Sharding by Topic

### The Problem

One server cannot handle:

* 100k+ connections
* Hot topics (e.g., BTC price)
* Unlimited fan-out

### The Solution

Partition responsibility by topic

**Example:**

```text
prices:A* → server 1
prices:B* → server 2
orders:*  → server 3
```

**Now:**

* Each server holds fewer routing entries
* Fan-out load is spread
* Memory and CPU are bounded

> **Sharding = "this server only routes some topics"**

---

## 22. Why Introduce Pub/Sub Brokers (Redis, Kafka)

### Problem #1: Multiple WS Servers

```text
Client → WS server A
Producer → WS server B ❌
```

They don't share memory.

### Solution: External Broker

```text
Producer
   ↓
Kafka / Redis
   ↓
WS server A → clients
WS server B → clients
```

**Now:**

* All WS servers see the same events
* Horizontal scaling works
* Reconnects don't lose data (Kafka)

### Redis vs Kafka (Quick Intuition)

| Redis Pub/Sub      | Kafka               |
|--------------------|---------------------|
| Low latency        | Durable             |
| No replay          | Replayable          |
| In-memory          | Disk-backed         |
| Best for live UI   | Best for correctness|

---

## 23. Why Introduce Per-Topic Workers

### The Problem

Hot topics cause:

* CPU spikes
* Lock contention
* Backpressure

**Example:**

```text
prices:BTC → 50k subscribers
```

If handled inline:

* One slow send can block others
* Latency explodes

### The Solution: Isolate Work Per Topic

```text
Topic → dedicated worker / queue → fan-out
```

**Benefits:**

* Hot topics don't block cold ones
* Rate limiting per topic
* Easier metrics & autoscaling

This mirrors:

* Kafka partitions
* Actor model
* Event loop per shard

---

## 24. One Unified Mental Picture

```text
                Producer
                    │
            ┌───────▼────────┐
            │ Pub/Sub Broker  │
            │ (Kafka / Redis) │
            └───────┬────────┘
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   WS Server A  WS Server B  WS Server C
   (topics 1–3) (topics 4–6) (topics 7–9)
        │           │           │
     clients     clients     clients
```

---

## 25. Final TL;DR (Memorize This)

* **WebSocket subscriptions = in-memory pub/sub**
* **Brokers externalize routing state**
* **Sharding limits blast radius**
* **Per-topic workers isolate hot paths**
* **Kafka formalizes everything WebSockets do informally**
