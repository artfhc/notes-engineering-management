# System Design Patterns Cheat Sheet

## Overview

**Mental model:** Pattern = reusable solution to a recurring scaling/reliability problem.

**In interviews:** State the problem → apply the smallest pattern that fixes it → call out tradeoffs.

---

## Core Patterns

### 1. Caching (Speed Up Hot Reads)

**Use when:** Read-heavy, repeated requests, expensive computation, slow downstream

**Example Problem:**
- Product detail page P95 is 800ms
- DB is overloaded during peak

**Solution:**
- Add read-through cache (Redis) keyed by `productId`
- Use TTL + cache invalidation on updates

**Why:**
- Removes repeated DB reads for hot items
- Lowers latency and DB load

**Tradeoffs:**
- Stale data
- Cache stampede
- Invalidation complexity
- Extra infra cost

**Quick add-ons:**
- Request coalescing
- Stale-while-revalidate
- Per-key locks

---

### 2. Replication (Scale Reads + Availability)

**Use when:** Many reads, need higher availability, tolerate slightly stale data

**Example Problem:**
- Search results page causes DB CPU spikes
- Writes are moderate but reads are huge

**Solution:**
- Primary + read replicas
- Route read queries to replicas

**Why:**
- Horizontal read scaling
- Primary handles writes
- Replicas serve read traffic

**Tradeoffs:**
- Replication lag → stale reads
- Failover complexity
- Consistency bugs ("read-your-writes")

**Interview tip:** Mention read-after-write solutions (stickiness to primary, session consistency)

---

### 3. Indexing (Make Queries Fast, Not Bigger)

**Use when:** Queries scan too much data, "slow query" hotspots

**Example Problem:**
- "Orders by user + last 30 days" query does full table scan

**Solution:**
- Add compound index on `(user_id, created_at desc)`

**Why:**
- Turns scan into index seek
- Reduces IO drastically

**Tradeoffs:**
- Slower writes (index maintenance)
- More storage
- Over-indexing hurts performance

**Rule of thumb:** Index your WHERE + ORDER BY columns for the top queries

---

### 4. Load Balancing + Stateless Services (Scale Horizontally)

**Use when:** API tier needs to scale elastically

**Example Problem:**
- Checkout API hits 100% CPU on a single instance

**Solution:**
- Make service stateless
- Add load balancer
- Autoscale

**Why:**
- Easy horizontal scaling
- Isolates failures
- Deploys become safer

**Tradeoffs:**
- State must move to DB/cache
- Distributed tracing becomes important

---

### 5. Queue / Async Processing (Smooth Spikes, Decouple)

**Use when:** Bursts, slow tasks, need resilience from downstream

**Example Problem:**
- Uploading a video blocks request for 30s (transcoding)

**Solution:**
- Write metadata synchronously
- Enqueue job (SQS/Kafka)
- Background workers transcode

**Why:**
- Fast response
- Absorbs spikes
- Retries on failure

**Tradeoffs:**
- Eventual consistency
- Job deduplication
- Monitoring/poison messages

**Interview tip:** Mention idempotency keys and dead-letter queues

---

### 6. Pub/Sub (Fan-out Updates to Many Consumers)

**Use when:** Multiple systems need the same events, avoid point-to-point integrations

**Example Problem:**
- "Order placed" must trigger email, analytics, inventory, fraud checks

**Solution:**
- Emit `OrderPlaced` event to topic
- Each service subscribes

**Why:**
- Decouples producers/consumers
- Easy to add new consumers

**Tradeoffs:**
- Delivery semantics (at-least-once)
- Ordering
- Schema evolution
- Replay costs

**Must say:** Use event versioning + schema registry (or strict JSON contracts)

---

### 7. CQRS (Separate Write Model from Read Model)

**Use when:** Read queries are complex/different shape than writes, need read scalability

**Example Problem:**
- Feed page needs "ranked posts + aggregates + joins"
- Write path is simple

**Solution:**
- Write to OLTP DB
- Publish events
- Build read-optimized view (denormalized store / Elasticsearch)

**Why:**
- Read side can be optimized independently
- Supports fast queries

**Tradeoffs:**
- Eventual consistency
- Backfills
- Dual-store ops complexity

**Interview tip:** Explain how you rebuild the read model from event log

---

### 8. Sharding / Partitioning (Scale Writes + Storage)

**Use when:** Single DB can't handle writes or data size

**Example Problem:**
- Messages table too large
- Write throughput saturates a single DB

**Solution:**
- Shard by `conversation_id` (or `userId`) across N partitions

**Why:**
- Spreads writes + storage
- Improves parallelism

**Tradeoffs:**
- Cross-shard queries are hard
- Resharding is painful
- Hotspot keys

**Tip:** Pick shard key that matches access patterns, handle "celebrity" hotspots

---

### 9. Rate Limiting + Backpressure (Protect the System)

**Use when:** Abusive clients, thundering herds, downstream fragility

**Example Problem:**
- A buggy client retries rapidly and knocks over auth service

**Solution:**
- Token bucket rate limit at edge
- Exponential backoff
- Circuit breaker

**Why:**
- Prevent cascading failures
- Keeps service healthy under stress

**Tradeoffs:**
- Some requests rejected
- Tuning required
- Fairness concerns

---

### 10. Reliability Basics (Failures Are Normal)

**Use when:** Any production system at scale

**Example Problem:**
- A downstream dependency times out
- Requests pile up until everything melts

**Solution:**
- Timeouts
- Retries (bounded)
- Circuit breaker
- Bulkheads

**Why:**
- Contain failure
- Preserve capacity
- Avoid cascading collapse

**Tradeoffs:**
- Retries can amplify load
- Partial outages
- Requires good observability

---

## Interview "Pattern Pitch" Template

> "We're seeing **X** (metric + symptom). Since traffic is mostly **Y** (reads/writes/bursty), I'd introduce **Z** (pattern) at **(layer)**. This improves **A** (latency/throughput/availability) by **B** (mechanism). Tradeoffs are **C**; I'd mitigate with **D**."

---

# Domain-Specific Patterns: Search, Feed, Chat

## How to Talk in Interviews

**Traffic shape → failure mode → pick pattern → tradeoffs + mitigations**

| Domain | Characteristics |
|--------|----------------|
| **Search** | Read-heavy, bursty, latency-sensitive, relevance iteration |
| **Feed** | Read-heavy + ranking, fanout, cacheable, freshness tradeoffs |
| **Chat** | Write-heavy, ordering, real-time delivery, offline sync |

---

## SEARCH (Autocomplete + Results)

### 1. Caching Hot Queries

**Problem:**
- Top 1% queries generate 40% of traffic
- P95 latency spikes at peak

**Solution:**
- Cache results by `(normalized_query, locale, filters)` with short TTL
- Use stale-while-revalidate

**Why:**
- Avoid repeated compute / DB / search engine hits
- Smooth bursty traffic

**Tradeoffs:**
- Stale results
- Cache stampede
- Low hit rate for long-tail queries

**Mitigations:**
- Request coalescing
- Per-key locks
- Tiered cache (in-process + Redis)
- Query normalization

---

### 2. Indexing / Search Engine

**Problem:**
- SQL `LIKE` queries and DB scans can't support prefix matching + relevance ranking

**Solution:**
- Use an inverted index (Elasticsearch/OpenSearch)
- Or specialized suggester for autocomplete

**Why:**
- Search engines are optimized for full-text queries and ranking

**Tradeoffs:**
- Eventual consistency
- Infra and ops cost
- Index tuning complexity

**Mitigations:**
- Async indexing pipeline
- Backfill / reindex strategy
- Dual-read during migration

---

### 3. CQRS for Search

**Problem:**
- Writes are simple catalog updates
- Reads are complex (filters, facets, ranking)

**Solution:**
- Write to OLTP DB → publish events → build read-optimized index and precomputed facets

**Why:**
- Read path scales independently and matches query shape

**Tradeoffs:**
- Data freshness lag
- Rebuild complexity

**Mitigations:**
- Versioned schemas
- Replayable event log
- Reindex tooling

---

### 4. Hedged Requests + Timeouts

**Problem:**
- Tail latency caused by slow shards or replicas

**Solution:**
- Apply timeouts
- Issue a hedged request to another replica after a threshold

**Why:**
- Reduces P95/P99 latency without overprovisioning

**Tradeoffs:**
- Increased load
- Can worsen incidents if misconfigured

**Mitigations:**
- Hedge only idempotent reads
- Cap hedged requests
- Adaptive thresholds

---

## FEED (Home Feed / Recommendations)

### 5. Fanout: Push vs Pull

**Problem:**
- Generating feed on request is slow due to joins and ranking

**Solutions:**

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **Push (Fanout-on-write)** | Precompute per-user timelines | Fast reads | Write amplification, storage cost, hard to rerank |
| **Pull (Fanout-on-read)** | Compute feed on read from candidate stores | Cheaper writes, flexible ranking | Slower reads, high compute cost |

**Why:**
- Push = fast reads
- Pull = cheaper writes and flexible ranking

**Mitigations:**
- Hybrid approach — push followed content, pull recommended modules

---

### 6. Materialized Views / Denormalization

**Problem:**
- Feed rendering requires many joins (post, author, stats)

**Solution:**
- Store feed items with embedded author snapshot and counters

**Why:**
- Fewer round trips and faster rendering

**Tradeoffs:**
- Stale embedded data
- Duplication

**Mitigations:**
- Background refresh
- Clearly defined source-of-truth fields
- Selective embedding

---

### 7. Ranker Pipeline with Fallbacks

**Problem:**
- ML ranking dependency occasionally fails

**Solution:**
- Candidate generation → ranker → post-processing
- With heuristic or cached fallback

**Why:**
- Ensures availability and graceful degradation

**Tradeoffs:**
- Reduced relevance during fallback
- More code paths

**Mitigations:**
- Feature flags
- Shadow evaluation
- Fallback rate monitoring

---

### 8. Pagination Strategy (Cursor-Based)

**Problem:**
- Offset pagination causes duplicates or missing items when feed changes

**Solution:**
- Cursor-based pagination using `(last_rank_score, last_id)`

**Why:**
- Stable pagination under inserts
- Better performance

**Tradeoffs:**
- Hard to jump to arbitrary pages
- Requires deterministic sorting

**Mitigations:**
- Opaque cursor tokens
- Client-side deduplication

---

## CHAT (Realtime + Offline)

### 9. WebSocket + Topic Subscription

**Problem:**
- Need real-time delivery, typing indicators, read receipts

**Solution:**
- WebSocket connection with `SUBSCRIBE conversationId`
- Server pushes events

**Why:**
- Low-latency bidirectional communication

**Tradeoffs:**
- Connection scaling
- Stateful routing
- Mobile network instability

**Mitigations:**
- Sticky sessions or connection registry
- Heartbeats
- Reconnect + resubscribe with backoff

---

### 10. Ordering + Idempotency

**Problem:**
- Retries and reconnects cause duplicate or out-of-order messages

**Solution:**
- Client-generated `messageId`
- Server assigns monotonic `sequenceId` per conversation

**Why:**
- Ensures exactly-once UX and consistent ordering

**Tradeoffs:**
- Hot shards for popular conversations
- Multi-region complexity

**Mitigations:**
- Shard by `conversationId`
- Client buffering for minor reordering

---

### 11. Write Path: Append Log + Async Fanout

**Problem:**
- Message send must be fast but trigger many side effects

**Solution:**
- Synchronously append message
- Asynchronously publish `MessageCreated` event

**Why:**
- Low latency writes with decoupled consumers

**Tradeoffs:**
- Eventual consistency
- Consumer retry complexity

**Mitigations:**
- Idempotent consumers
- Dead-letter queues
- Replay support

---

### 12. Offline Sync (Delta / Changelog)

**Problem:**
- Client reconnects after being offline and needs missed messages

**Solution:**
- Delta API: `GET /messages?afterSeq=XYZ`

**Why:**
- Efficient sync with minimal payload

**Tradeoffs:**
- Requires retention strategy
- Storage growth

**Mitigations:**
- Retention window
- Periodic snapshots + checkpoints

---

## Pattern Picker by Symptom

| Symptom | Recommended Patterns |
|---------|---------------------|
| **High read latency** | Caching, replicas, hedged reads, denormalization |
| **Bursty traffic** | Queues, request coalescing, rate limiting |
| **Too many joins** | Materialized views |
| **Freshness vs speed** | TTL + stale-while-revalidate, hybrid fanout |
| **Realtime issues** | Reconnect logic, idempotency, sequencing |
| **Scaling data** | Sharding, CQRS read models |

---

## Interview Pattern Pitch Template

> "We're seeing **X** (metric + symptom). Since traffic is mostly **Y**, I'd introduce **Z** at the **layer**.
> This improves **A** via **B**. Tradeoffs are **C**, mitigated by **D**."

**Example:**

> "We're seeing **P95 search latency at 800ms** (metric). Since traffic is mostly **read-heavy with hot queries**, I'd introduce **Redis caching** at the **API layer**.
> This improves **latency and DB load** via **avoiding repeated computation**. Tradeoffs are **stale data and cache stampede**, mitigated by **short TTL and request coalescing**."
