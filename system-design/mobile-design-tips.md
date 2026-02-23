# The 3 Pillars Model

When the interviewer says "let's go deeper," think:

1. **Client Resilience**
2. **System Resilience**
3. **Decision Tradeoffs**

Everything fits under those 3.

---

## Pillar 1 — Client Resilience

> "How does the Android app stay correct, fast, and safe under stress?"

### The 5 Things to Always Cover

**1. State Management**
- Single source of truth
- ViewModel owns state
- Immutable UI state
- Avoid duplicate emissions

**2. Offline Strategy**
- Local DB (Room)
- Pending action queue
- WorkManager sync
- Idempotent APIs

**3. Real-Time Handling**
- WebSocket lifecycle-aware
- Reconnect with backoff
- Push fallback
- Deduplication by order_id

**4. Performance Safety**
- Background threads (Coroutines)
- DiffUtil for list updates
- Avoid full list rebind
- Throttle updates on burst traffic

**5. Observability**
- Crash monitoring
- Order delivery latency metric
- WebSocket reconnect metric

**Mental checklist:** State → Offline → Real-time → Performance → Observability

---

## Pillar 2 — System Resilience

> "What breaks at scale?"

### The 4 Failure Domains

**1. Traffic Spikes** (e.g. lunch rush)
- Horizontal scaling
- Queue buffering (Kafka)
- Rate limiting

**2. Service Failures** (e.g. Notification Service down)
- Retry with backoff
- Dead-letter queue
- Circuit breaker
- Fallback polling

**3. Data Inconsistency** (e.g. duplicate order acceptance)
- Idempotent APIs
- Order versioning
- Strong consistency on order state

**4. Regional Failure** (e.g. one region crashes)
- Geo partitioning
- Regional failover
- Replication

**Mental checklist:** Traffic → Service failure → Data consistency → Regional failure

---

## Pillar 3 — Decision Tradeoffs

Senior signal is not knowing answers. It's showing decision thinking.

Every tradeoff fits one of these 4 dimensions:

**1. Consistency vs. Latency**
- Strong consistency = slower (use for order state)
- Eventual consistency = faster (acceptable for analytics)

**2. Simplicity vs. Scalability**
- Monolith is easier to build
- Microservices scale better

**3. Cost vs. Reliability**
- WebSocket infra is expensive but reliable
- Push notifications are cheaper but less reliable

**4. Dev Speed vs. Operational Complexity**
- NoSQL is flexible and fast to iterate
- SQL is rigid but safer for transactions

When stuck, say: *"The tradeoff here is between X and Y."* That sentence alone signals senior thinking.

---

## Quick Recall

```
Client Resilience      System Resilience       Decision Tradeoffs
  State                  Traffic spikes          Consistency vs latency
  Offline                Service failure         Simplicity vs scalability
  Real-time              Data consistency        Cost vs reliability
  Performance            Regional failure        Dev speed vs complexity
  Observability
```

---

## Why This Framework Works for EMs

Interviewers want to see:
- Can you think in failure modes?
- Can you think in resilience?
- Can you articulate tradeoffs?
- Can you protect revenue?

Not: can you name every Android API.
