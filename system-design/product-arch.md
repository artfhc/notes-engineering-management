# Product Architecture Interview Cheat Sheet (1-Page)

## 1. Universal System Design Framework

Always structure your answer like this:

| Step | What to Do | Example Phrases |
|------|-----------|-----------------|
| 1 Clarify | Define scope and assumptions | "I'll start by clarifying requirements." |
| 2 Requirements | Functional + Non-functional | latency, scale, durability |
| 3 Estimate Scale | Users, QPS, storage | "Assume 50M users, 1M DAU." |
| 4 High-Level Design | Main components | client → API → services → DB |
| 5 Data Model | Core entities | User, Post, Message |
| 6 APIs | Key endpoints | POST /messages |
| 7 Deep Dive | Hardest problem | ranking, ordering, geo search |
| 8 Scaling | Bottlenecks | caching, sharding, async queues |
| 9 Failure Modes | Recovery plan | retries, fallback |

**Strong opener:** "I'll clarify requirements, propose a high-level architecture, then dive deeper into the most critical scaling challenge."

---

## 2. Design Instagram / Feed System

### Core Idea

A read-heavy system that aggregates posts from followed users.

### Key Components

```
Client
 ↓
API Gateway
 ↓
Feed Service
 ↓
Feed Cache
 ↓
Post Storage
 ↓
Media CDN
```

### Core Entities

| Entity | Fields |
|--------|--------|
| User | id |
| Post | id, user_id, media_url |
| FollowEdge | follower_id, followee_id |
| FeedEntry | user_id, post_id, score |

### Critical Tradeoff

| Strategy | Pros | Cons |
|----------|------|------|
| Fanout on Write | fast reads | expensive writes |
| Fanout on Read | cheap writes | slower reads |

**Best answer:** Use hybrid fanout: write for normal users, read for celebrities.

### Scaling
- Cache first page of feed
- Async fanout workers
- CDN for media
- Ranking service

### Failure Handling
- Fallback to chronological feed
- Serve slightly stale cache

---

## 3. Design Chat System

### Core Idea

Real-time messaging with persistent history and ordering guarantees.

### Architecture

```
Client
 ↓
WebSocket Gateway
 ↓
Chat Service
 ↓
Message Queue
 ↓
Message Storage
 ↓
Push Notification Service
```

### Core Entities

| Entity | Fields |
|--------|--------|
| Conversation | id |
| Message | id, conversation_id, sender_id |
| Participant | user_id |
| ReadReceipt | message_id, user_id |

### Message Flow

1. Client sends message via WebSocket
2. Server assigns message ID
3. Persist message
4. Publish event
5. Deliver to recipient
6. Push notification if offline

### Critical Design Choice

Guarantee ordering per conversation, not globally.

### Scaling
- Stateless WebSocket servers
- Broker for message distribution
- Shard by `conversation_id`

### Failure Handling
- Idempotent message IDs
- Reconnect + fetch missed messages

---

## 4. Design Ride Matching / Dispatch

### Core Idea

Match riders with nearby drivers using geo-indexed search.

### Architecture

```
Rider App / Driver App
 ↓
API Gateway
 ↓
Trip Service
 ↓
Dispatch Service
 ↓
Geo Index
 ↓
Driver State Store
```

### Core Entities

| Entity | Fields |
|--------|--------|
| Driver | id, status |
| Rider | id |
| Trip | id, rider_id, driver_id |
| LocationUpdate | driver_id, lat, lon |

### Ride Flow

1. Rider requests trip
2. Find nearby drivers
3. Rank candidates
4. Offer trip
5. First accept wins
6. Trip assigned

### Hard Problems

| Problem | Solution |
|---------|----------|
| Nearby search | geohash / spatial index |
| Double assignment | atomic state transition |
| Location spam | TTL + latest location store |

### Scaling
- Partition by region
- In-memory driver state
- Async analytics pipeline

---

## 5. Common Scaling Patterns

| Problem | Solution |
|---------|----------|
| Read heavy | caching |
| Write heavy | async queues |
| Hot users | partitioning |
| Real time | WebSocket |
| Media delivery | CDN |
| Fanout events | pub/sub broker |

---

## 6. High-Impact Interview Phrases

Use these to sound senior and structured:

- "I'll optimize for the critical user path first."
- "This is a read-heavy workload, so caching is important."
- "I'll separate the write path from the read path."
- "Ordering will be guaranteed per conversation."
- "I'd use a hybrid strategy to handle celebrity scale."
- "I'll keep the MVP simple, then scale with partitioning."

---

## 7. Common Bottlenecks

| System | Bottleneck |
|--------|-----------|
| Feed | celebrity fanout |
| Chat | WebSocket connections |
| Ride Matching | location updates |

---

## 8. If You Get Stuck

Say: "Let me step back and restate the problem and focus on the critical path." Then continue.

Interviewers value structured thinking more than perfect answers.

---

## 9. 30-Second Summary Strategy

At the end, summarize:

> "This system separates read and write paths, uses caching and async processing to scale, and focuses on the core user flow to keep latency low."

**Goal:** Memorize the framework, not the exact designs. You can adapt the same structure to almost any product architecture question.
