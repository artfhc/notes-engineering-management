# System Design Comparison Tables

## 1. Consistency Model Comparison

| Model            | Guarantee                     | Example Use Case            | Pros                    | Cons                      | Interview Trigger                              |
| ---------------- | ----------------------------- | --------------------------- | ----------------------- | ------------------------- | ---------------------------------------------- |
| Strong           | Read always sees latest write | Payments, inventory booking | Simple reasoning        | Latency, low availability | "Double booking must not happen"               |
| Eventual         | Will converge eventually      | Social feed, likes          | Highly scalable         | Stale reads               | "Timeline can be slightly stale"               |
| Causal           | Preserves causality           | Chat threads                | Better UX than eventual | More complex              | "Reply must see original message"              |
| Read-Your-Writes | User sees own writes          | Profile edit                | Good UX                 | Still globally eventual   | "User must see their change immediately"       |
| Monotonic Reads  | Reads never go backwards      | Feed pagination             | Prevent flicker         | Not globally strong       | "Page 2 shouldn't show older data than page 1" |

💡 **Interview hack:**
- If system involves money, booking, quota → Strong
- If system involves content consumption → Eventual + client compensation

---

## 2. Real-Time Communication: Polling vs SSE vs WebSocket

| Method       | Direction       | Latency  | Scalability | Best For                    | Watch Out For              |
| ------------ | --------------- | -------- | ----------- | --------------------------- | -------------------------- |
| Polling      | Client → Server | High     | Poor        | Simple dashboards           | Wasted requests            |
| Long Polling | Client waits    | Medium   | Medium      | Legacy chat                 | Thread exhaustion          |
| SSE          | Server → Client | Low      | Good        | Notifications               | No client → server channel |
| WebSocket    | Bi-directional  | Very Low | Needs infra | Chat, trading, merchant app | Stateful scaling           |

For your merchant app case → WebSocket + Redis PubSub fanout.

---

## 3. Caching Strategy Comparison

| Layer         | Example           | TTL? | Pros           | Cons               | Interview Angle    |
| ------------- | ----------------- | ---- | -------------- | ------------------ | ------------------ |
| Browser Cache | ETag              | Yes  | Free           | Hard to control    | "Reduce bandwidth" |
| CDN           | Cloudflare        | Yes  | Offload origin | Cache invalidation | ISR, static pages  |
| App Cache     | Redis             | Yes  | Fast           | Consistency issues | Feed ranking       |
| DB Cache      | Materialized View | No   | Precomputed    | Expensive refresh  | Heavy aggregation  |

🧠 **Interview pattern:**
- Static content → CDN
- Read-heavy dynamic → Redis
- Expensive join → precompute

---

## 4. Rate Limiting Algorithms

| Algorithm      | Burst Support | Memory | Accuracy         | Use Case        |
| -------------- | ------------- | ------ | ---------------- | --------------- |
| Fixed Window   | No            | Low    | Inaccurate edges | Basic API       |
| Sliding Window | Limited       | Medium | Accurate         | Public API      |
| Leaky Bucket   | No            | Low    | Smooth traffic   | Traffic shaping |
| Token Bucket   | Yes           | Low    | Flexible         | Mobile clients  |

You've asked about token vs leaky before —
- Token bucket = burst allowed
- Leaky bucket = smooth traffic only

---

## 5. Database Choice (Messaging / Feed Context)

| DB             | Strength            | Weakness             | Use Case       |
| -------------- | ------------------- | -------------------- | -------------- |
| MySQL/Postgres | Strong consistency  | Hard to scale writes | Payments       |
| Cassandra      | Massive write scale | Eventual consistency | Chat, feed     |
| DynamoDB       | Serverless scale    | Cost                 | Event streams  |
| Time-series DB | Time-based queries  | Not relational       | Metrics        |
| Elasticsearch  | Search              | Not source of truth  | Product search |

**For messaging:**
- Write-heavy → Cassandra
- Ordered per user → partition by user_id

---

## 6. Sharding Strategy

| Strategy           | How           | Pros              | Cons             | When to Use       |
| ------------------ | ------------- | ----------------- | ---------------- | ----------------- |
| Hash-based         | hash(user_id) | Even distribution | Hard rebalancing | Chat, feed        |
| Range-based        | ID range      | Easy queries      | Hot shard        | Time-ordered data |
| Geo-based          | Region        | Low latency       | Imbalance        | Global apps       |
| Consistent Hashing | Ring          | Easy scaling      | Complexity       | Large clusters    |

**Interview killer question:** "What happens when one shard becomes hot?"

---

## 7. Push vs Pull Architecture (Feed/Search)

| Model  | Example         | Pros         | Cons                      |
| ------ | --------------- | ------------ | ------------------------- |
| Push   | Fanout on write | Fast reads   | Heavy write amplification |
| Pull   | Fanout on read  | Cheap writes | Slow reads                |
| Hybrid | Celebrity pull  | Balanced     | Complexity                |

---

## 9. Strong Ordering vs Best-Effort Ordering (Chat)

| Type              | Guarantee       | Cost   | Example        |
| ----------------- | --------------- | ------ | -------------- |
| Global ordering   | Single sequence | High   | Financial logs |
| Per-room ordering | Partitioned     | Medium | Slack          |
| Best effort       | None            | Low    | Comments       |

---

## 10. Read Optimization Strategies

| Strategy          | Example           | Tradeoff    |
| ----------------- | ----------------- | ----------- |
| Precompute        | Materialized feed | Write heavy |
| Index             | Compound index    | Storage     |
| Denormalize       | Store author name | Duplication |
| Pagination cursor | created_at + id   | Complexity  |

---

## 12. Frontend Architecture Comparison (Since you're mobile EM)

| Pattern | Data Flow       | When to Use    |
| ------- | --------------- | -------------- |
| MVC     | Bi-directional  | Legacy         |
| MVP     | Presenter-based | Android legacy |
| MVVM    | Reactive        | Modern Android |
| MVI     | Unidirectional  | Complex state  |

**For interview:** "Why unidirectional data flow?" → easier debugging + predictable state.
