# DoorDash Interview Prep

## Interview Structure

### 1. Technical Screen (60 min)
- No coding
- High-level system design
- Domain + tradeoffs discussion
- Client ↔ Server interaction
- Data modeling + scaling decisions

### 2. Leadership Interview (45 min)
- Hiring strategy
- Team building
- Scaling org
- Mentorship
- Feedback
- Rapid growth environment

---

## Part 1 — System Design Prep

### Likely Prompts (DoorDash Context)
- Design a food delivery system
- Design order dispatch system
- Design real-time order tracking
- Design merchant order management app
- Design notification system
- Design recommendations for restaurants
- Design promotion engine
- Design availability system for drivers
- Design Instagram news feed
- Design a customer food item review system
- Design a donation service (see below)

### Answer Framework

1. **Clarify Requirements** — functional + non-functional (scale, latency, consistency, availability)
2. **High-Level Architecture** — major services, client-server interactions, data flow
3. **Data Modeling** — core entities, storage type, indexing
4. **Scale Strategy** — sharding, caching, async processing, rate limiting
5. **Consistency Model** — strong vs eventual, where and why
6. **Failure Modes** — what breaks, how do we recover
7. **Trade-offs**

---

## Mock Scenario: Real-Time Order Tracking System

### Requirements

**Functional:**
- Customer places order
- Restaurant accepts order
- Driver assigned
- Real-time location updates
- Customer sees live tracking
- Status updates

**Non-Functional:**
- Millions of concurrent users
- <1s latency for tracking
- Highly available
- Fault tolerant
- Mobile-first

### High-Level Architecture

```
Client Apps (Customer / Merchant / Driver)
        ↓
API Gateway
        ↓
Core Services:
  - Order Service
  - Dispatch Service
  - Tracking Service
  - Notification Service
        ↓
Databases + Cache + Event Bus
```

Additional layers:
- WebSocket layer for real-time
- Kafka / PubSub for events
- Redis for hot state

### Data Modeling

| Entity | Storage | Why |
|--------|---------|-----|
| Order | SQL (Postgres) | transactional consistency |
| Driver location | Redis | high write frequency |
| Order events | Kafka | async processing |
| Restaurant menu | NoSQL | flexible schema |

Key decisions:
- Orders need strong consistency
- Driver location is ephemeral
- Event log enables replay + analytics

### Tradeoffs

**Redis vs DB for driver location:**
- DB too slow for high-frequency updates
- Redis supports TTL
- Can shard by `driver_id`

**Polling vs WebSocket:**
- Polling expensive at scale
- WebSocket reduces overhead
- Needs broker layer for multi-node scaling

### Scaling Strategy

**Horizontal scale:**
- Stateless API servers
- DB read replicas

**Shard by:** `order_id`, `region`, `restaurant_id`

**Caching:** menu cache, order status cache, driver availability cache

**Backpressure:** token bucket rate limiter, circuit breakers

### Failure Discussion

- Idempotency keys
- Exactly-once vs at-least-once
- Retry strategy
- Dead letter queue
- Partial system outage fallback
- **Senior signal:** mention blast radius containment

### Advanced Talking Points

- ML-driven dispatch optimization
- Feature flags for experimentation
- Latency budget between mobile and backend
- Observability (SLIs, SLOs, error budgets)
- Mobile UX tradeoffs when backend is slow

---

## Mock Scenario: Instagram News Feed

### Requirements

**Functional:**
- User posts a photo/video with caption
- User sees a feed of posts from accounts they follow
- Feed is roughly reverse-chronological (with ranking)
- Like, comment on posts

**Non-Functional:**
- Read-heavy (feed reads >> post writes)
- <500ms feed load
- Eventual consistency acceptable for feed ordering
- Highly available

### Key Design Decisions

**Fan-out strategy:**
- Fan-out on write (push): precompute feed inbox per user on post creation
  - Fast reads, slow writes; breaks for celebrities with millions of followers
- Fan-out on read (pull): compute feed at read time from followee post lists
  - Always fresh, but expensive at read time
- **Hybrid (recommended):** fan-out on write for normal users, fan-out on read for celebrities (> N followers threshold)

**Storage:**
| Entity | Storage | Why |
|--------|---------|-----|
| Posts | SQL (Postgres) | structured, queryable |
| Feed inbox | Redis (sorted set by timestamp) | O(log n) range reads |
| Media (photos/video) | Object store (S3) + CDN | large blobs, global delivery |
| Follow graph | Graph DB or denormalized SQL | efficient follower lookups |

**Architecture:**
```
Client
  ↓
API Gateway
  ↓
Feed Service ──► Redis feed inbox (per user)
Post Service ──► Fanout Worker (Kafka) ──► pushes to follower inboxes
Media Service ──► S3 + CDN
```

**Scaling:**
- Shard feed inbox by `user_id`
- CDN for media (cache at edge)
- Read replicas for post metadata
- Rank/ML scoring layer applied at read time on top of raw feed

**Failure modes:**
- Fanout queue backup during traffic spike → use Kafka with backpressure
- Feed inbox stale if worker lags → acceptable (eventual consistency)
- Cache miss on cold start → fallback to pull from DB

---

## Mock Scenario: Customer Food Item Review System

### Requirements

**Functional:**
- Customer submits a text review + star rating for a food item after delivery
- Other users can view reviews for a food item
- Reviews can be flagged/moderated

**Non-Functional:**
- Write once, read many
- Reviews visible within seconds of submission (eventual ok)
- Scale: thousands of reviews/min during peak, millions of food items

### Key Design Decisions

**Data model:**

| Field | Type | Notes |
|-------|------|-------|
| `review_id` | UUID | primary key |
| `item_id` | FK | food item being reviewed |
| `user_id` | FK | reviewer |
| `rating` | int (1–5) | |
| `text` | text | |
| `created_at` | timestamp | |
| `status` | enum | `pending`, `approved`, `flagged` |

**Aggregated rating:** maintain a denormalized `avg_rating` + `review_count` on the food item row, updated via async worker after each review write (not inline).

**Architecture:**
```
Client ──► Review Service ──► DB (writes)
                          ──► Kafka ──► Rating Aggregation Worker ──► item table
                          ──► Moderation Service (async)
Read path: Review Service ──► Cache (item reviews, paginated) ──► DB on miss
```

**Scaling:**
- Shard reviews by `item_id`
- Cache top reviews per item (TTL ~5 min)
- Cursor-based pagination (not OFFSET)
- Moderation can be async (ML classifier + human review queue)

**Tradeoffs:**
- Inline rating update vs async worker: inline is simpler but adds write latency; async is eventually consistent but decoupled
- Full text search on reviews: add Elasticsearch if needed, not required for MVP

---

## Mock Scenario: Donation Service

### Context

DoorDash sponsors a 3-day charity event (Friday–Sunday). Users donate to 1 of 10 charities via a form (name, email, card info, charity, amount). Frontend (iOS/Android/web) is already built. Payment via Braintree (POST card info → 201 Created). All funds land in one account; CFO manually disburses afterward. **The interviewer gives no guidance — all choices are "up to you."**

**Scale:** millions of donations over 3 days, ~$100M total. **Reliability is the primary concern.**

### Requirements

**Functional:**
- Accept donation form submission
- Charge card via Braintree
- Record donation in DB
- Confirm to user

**Non-Functional:**
- Every donation must be recorded exactly once (no double-charges)
- No donation silently lost
- Available for all 3 days with no maintenance window
- Braintree is a 3rd party — must handle their failures

### Architecture

```
Client (iOS/Android/Web)
  ↓
API Gateway (rate limiting, TLS termination)
  ↓
Donation Service
  ├──► Idempotency check (Redis: client-generated idempotency key)
  ├──► Braintree API (POST charge)
  ├──► DB write (Postgres): donation record with status
  └──► Confirmation response to client
```

### Reliability Design (Primary Focus)

**Idempotency:**
- Client generates a UUID `idempotency_key` per submission
- Server checks Redis for existing key before calling Braintree
- If key exists, return cached response — no double charge
- Key TTL: 24h

**Saga / two-phase write:**
1. Write donation record with `status = pending`
2. Call Braintree
3. On success: update to `status = completed`
4. On failure: update to `status = failed`, surface error to user

**Braintree failure handling:**
- Timeout: do NOT retry blindly — check idempotency key first
- Braintree 5xx: retry with exponential backoff (max 3 attempts)
- Never charge if uncertain — mark as `status = requires_review` and alert ops

**No silent failures:**
- Dead letter queue for any donation that fails all retries
- Ops alert on DLQ messages
- Reconciliation job: compare Braintree transaction log vs internal DB daily

**Scaling for 3-day burst:**
- Stateless Donation Service — horizontal scale behind load balancer
- Postgres with connection pooling (PgBouncer); single-region is fine for 3 days
- Redis for idempotency key store (replicated)
- Rate limit at API Gateway: protect Braintree from burst (they likely have their own limits)

### Data Model

| Field | Type | Notes |
|-------|------|-------|
| `donation_id` | UUID | primary key |
| `idempotency_key` | UUID | unique, client-provided |
| `first_name`, `last_name`, `email` | string | |
| `charity_id` | int (1–10) | |
| `amount_cents` | int | store in cents, never floats |
| `braintree_txn_id` | string | returned by Braintree on success |
| `status` | enum | `pending`, `completed`, `failed`, `requires_review` |
| `created_at` | timestamp | |

### Key Interview Signals

- **Store amount in cents** (never floats for money)
- **Idempotency key** on every payment call — mention this proactively
- **Pending → completed pattern** before calling 3rd party (not after)
- **Reconciliation job** to catch mismatches between Braintree and internal DB
- **DLQ + ops alert** so no donation silently disappears
- **Don't over-engineer:** no need for microservices, multi-region, or Kafka — it's 3 days

---

## Part 2 — Leadership Interview

DoorDash is scaling headcount. They care about: hiring bar, scaling managers, organizational design, culture preservation, execution under hypergrowth.

### Leadership Narrative — 3 Pillars

**1. Craftsmanship + Quality Ownership**
- Incidents happened → team blamed luck → raised the bar
- Instituted better QA + release discipline, built debugging tools
- Shows: standards, accountability, ownership

**2. Building High-Leverage Teams**

*Hiring strategy:*
- Hire for slope > y-intercept
- Structured interviews, bar raisers, cross-functional calibration

*Partnering with Recruiting:*
- Clear intake doc, weekly sync, fast feedback loop, employer branding

**3. Growing Engineers**

*Common questions:*
- How do you mentor underperformers?
- How do you promote?
- How do you handle conflict?
- How do you give tough feedback?

*Use:* specific frameworks, 30/60/90 plans, growth plans, skip-level check-ins, clear leveling expectations

### Hard Questions

**"How do you hire fast without lowering the bar?"**
- Parallel pipelines, clear rubric, debrief discipline, reject decisively

**"What's your biggest hiring mistake?"**
- Be vulnerable. Show reflection. Show system improvement.

**"How do you scale culture?"**
- Written principles, interview calibration, leadership modeling behavior

**"How do you handle a strong performer who's toxic?"**
- DoorDash values intensity + kindness
- Address behavior, protect team, set expectation, escalate if needed

### Strategic Positioning

Strengths to leverage: mobile leadership, Search UX ownership, cross-ML collaboration, platform thinking, experimentation, production incident experience.

> "A systems-oriented engineering leader who builds durable teams that ship high-quality, ML-powered customer experiences."

---

## Practice Drills

**Technical:** "Design a real-time dispatch system for drivers."
- Walk through: requirements, matching algorithm, geo-indexing (geohash), latency constraints, scaling, failure handling

**Leadership:** "You need to hire 10 engineers in 6 months. What's your plan?"
- Structure answer: define team topology, define skill mix, partner with recruiting, structured interview panel, calibration, onboarding system

---

## Homework Before Interview

**Technical:**
- Practice 2 full mock designs
- Review: caching strategies, sharding, Pub/Sub, consistency models, rate limiting

**Leadership — Prepare 5 stories:**
- Incident recovery
- Hiring success
- Hiring mistake
- Tough feedback
- High performer growth
