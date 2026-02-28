# System Design Comparison Tables

## System Design Pillars

| Pillar 1: Client Resilience | Pillar 2: System Resilience | Pillar 3: Decision Tradeoffs  |
| --------------------------- | --------------------------- | ----------------------------- |
| State Management            | Traffic Spikes              | Consistency vs. Latency       |
| Offline Strategy            | Service Failure             | Simplicity vs. Scalability    |
| Real-Time Handling          | Data Consistency            | Cost vs. Reliability          |
| Performance Safety          | **Regional Failure**        | Dev Speed vs. Complexity      |
| **Observability**           |                             |                               |

---

## Core 6 Tables To Memorize

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

## 2. Push vs Pull Architecture (Feed/Search)

| Model  | How It Works                                    | Example                              | Pros         | Cons                      |
| ------ | ----------------------------------------------- | ------------------------------------ | ------------ | ------------------------- |
| Push   | On post → write to all followers' inbox tables  | Normal user with 200 followers       | Fast reads   | Heavy write amplification |
| Pull   | On feed open → aggregate posts dynamically      | Celebrity with 10M followers         | Cheap writes | Slow reads                |
| Hybrid | Push for normal users, pull for high-fanout     | Push ≤10K followers, pull for celebs | Balanced     | Complexity                |

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

| Algorithm      | Burst Support | Memory | Accuracy         | Use Case        | Example                                                   |
| -------------- | ------------- | ------ | ---------------- | --------------- | --------------------------------------------------------- |
| Fixed Window   | No            | Low    | Inaccurate edges | Basic API       | 100 req/min; resets at :00 — spike at :59 + :00 slips by |
| Sliding Window | Limited       | Medium | Accurate         | Public API      | GitHub API: 5000 req/hr rolling, no boundary exploit      |
| Leaky Bucket   | No            | Low    | Smooth traffic   | Traffic shaping | Video upload: process 1 frame/sec regardless of bursts    |
| Token Bucket   | Yes           | Low    | Flexible         | Mobile clients  | Stripe API: refill 10 tokens/sec, burst up to 100 tokens  |

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

| Strategy           | How           | Pros              | Cons             | When to Use       | Example                                               |
| ------------------ | ------------- | ----------------- | ---------------- | ----------------- | ----------------------------------------------------- |
| Hash-based         | hash(user_id) | Even distribution | Hard rebalancing | Chat, feed        | Discord: shard messages by hash(channel_id)           |
| Range-based        | ID range      | Easy queries      | Hot shard        | Time-ordered data | Logs: shard 1 = Jan–Mar, shard 2 = Apr–Jun            |
| Geo-based          | Region        | Low latency       | Imbalance        | Global apps       | Uber: US-West shard, EU shard, APAC shard             |
| Consistent Hashing | Ring          | Easy scaling      | Complexity       | Large clusters    | Cassandra: add node → only neighbors rebalance        |

**Interview killer question:** "What happens when one shard becomes hot?"

**Answer:** A hot shard (e.g. a celebrity user or viral post all landing on shard 3) causes CPU/memory spikes while other shards sit idle. Fix it by:
1. **Re-shard** — split the hot shard into smaller ones (expensive, requires migration)
2. **Add a shard key suffix** — append a random suffix (e.g. `user_id:0..9`) to spread one user across 10 sub-shards, then fan-out reads
3. **Cache in front** — put Redis in front of the hot shard to absorb read traffic
4. **Consistent hashing** — minimizes re-migration when adding capacity next time

---

## 7. Real-Time Communication: Polling vs SSE vs WebSocket

| Method       | Direction       | Latency  | Scalability | Best For                    | Watch Out For              |
| ------------ | --------------- | -------- | ----------- | --------------------------- | -------------------------- |
| Polling      | Client → Server | High     | Poor        | Simple dashboards           | Wasted requests            |
| Long Polling | Client waits    | Medium   | Medium      | Legacy chat                 | Thread exhaustion          |
| SSE          | Server → Client | Low      | Good        | Notifications               | No client → server channel |
| WebSocket    | Bi-directional  | Very Low | Needs infra | Chat, trading, merchant app | Stateful scaling           |

For your merchant app case → WebSocket + Redis PubSub fanout.

---

## 8. Strong Ordering vs Best-Effort Ordering (Chat)

| Type              | Guarantee       | Cost   | Example                                                    |
| ----------------- | --------------- | ------ | ---------------------------------------------------------- |
| Global ordering   | Single sequence | High   | Bank ledger: every txn gets a global seq_id, no gaps       |
| Per-room ordering | Partitioned     | Medium | Slack: msg_id increments per channel, not across workspace |
| Best effort       | None            | Low    | YouTube comments: order by likes, not arrival time         |

---

## 9. Read Optimization Strategies

| Strategy          | Example                                                      | Tradeoff    |
| ----------------- | ------------------------------------------------------------ | ----------- |
| Precompute        | Twitter: build user feed on write, not on load               | Write heavy |
| Index             | Messages: compound index on (room_id, created_at) for range  | Storage     |
| Denormalize       | Store sender_name on message row, skip user JOIN on read     | Duplication |
| Pagination cursor | Use `WHERE created_at < $cursor AND id < $id` instead of OFFSET | Complexity  |

---

## 10. Frontend Architecture Comparison (Since you're mobile EM)

| Pattern | Data Flow       | When to Use    |
| ------- | --------------- | -------------- |
| MVC     | Bi-directional  | Legacy         |
| MVP     | Presenter-based | Android legacy |
| MVVM    | Reactive        | Modern Android |
| MVI     | Unidirectional  | Complex state  |

**For interview:** "Why unidirectional data flow?" → easier debugging + predictable state.
