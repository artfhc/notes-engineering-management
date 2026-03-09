# Design an Instagram Feed

## What the Interviewer Is Testing

They want to see whether you understand:

- Feed generation
- Ranking
- Fanout tradeoffs
- Read-heavy system design
- Latency and freshness

## Clarifying Questions

Ask 3–5 of these:

- Is this home feed or user profile feed?
- Are we optimizing for chronological or ranked feed?
- Do we support images only, or also video/stories?
- Can users follow celebrities with millions of followers?
- Do we need likes/comments in scope?

**A good assumption:** I'll focus on the home feed for followers, supporting ranked results with near-real-time freshness.

## Functional Requirements

- Users can create posts
- Users can follow other users
- Users can open home feed
- Feed shows recent/relevant posts from followed accounts
- Users can like/comment on posts

## Non-Functional Requirements

- Low read latency
- High availability
- Eventual consistency is acceptable for feed updates
- High fanout for celebrity accounts
- Scalable ranking pipeline

## Core Entities

- **User**
- **Post**
- **FollowEdge** (follower_id, followee_id)
- **FeedEntry** (user_id, post_id, score/timestamp)
- **Like / Comment**

## High-Level Architecture

```
Mobile/Web Client
   ↓
API Gateway
   ↓
Post Service  ←→ Object Storage/CDN for media
   ↓
Fanout / Feed Generation Service
   ↓
Feed Store / Cache
   ↓
Ranking Service
   ↓
Home Feed API
```

**Key supporting systems:**

- User Graph Service for follow relationships
- Cache for hot feeds/posts
- Queue / stream for async feed updates
- CDN for media delivery

## Two Classic Approaches

### A. Fanout on Write

When user A posts, push that post into followers' feed buckets immediately.

**Pros:**
- Very fast feed reads
- Good for normal users

**Cons:**
- Expensive for celebrity users
- Huge write amplification

### B. Fanout on Read

When user opens the app, pull posts from followed users and assemble/rank on demand.

**Pros:**
- Better for celebrity accounts
- Lower write cost

**Cons:**
- Higher read latency
- More expensive feed generation during reads

### Best Interview Answer

Use a hybrid design:

- Normal users → fanout on write
- Celebrities / very high follower count → fanout on read

> This is one of the most important tradeoffs to mention.

## Deep Dive: Feed Ranking

A good answer:

1. Candidate posts are retrieved from followed users
2. Lightweight filtering removes hidden/blocked/ineligible content
3. Ranking service scores by:
   - Recency
   - Engagement
   - Relationship strength
   - Predicted relevance
4. Return top N items
5. Cache first page aggressively

## APIs

**Create Post**
```
POST /posts
```

**Get Feed**
```
GET /feed?cursor=...&limit=20
```

**Follow User**
```
POST /follow
```

## Scaling / Tradeoffs

### Hotspots
- Celebrities with millions of followers
- Feed cache invalidation
- Ranking latency

### Solutions
- Hybrid fanout
- Precompute first page
- Cache feed IDs separately from full post objects
- Serve media from CDN
- Async feed generation via queue

### Failure Modes
- Fanout worker lagging
- Ranking service timeout
- Cache miss storms

### Fallbacks
- Fall back to chronological order
- Show slightly stale feed
- Degrade ranking features if downstream systems fail

## 30-Second Summary

I'd model this as a read-heavy feed system with hybrid fanout. Normal users use fanout-on-write for fast reads, while celebrity accounts use fanout-on-read to avoid massive write amplification. A ranking layer orders candidate posts, and feed results are cached aggressively for low latency.

```
                +------------------+
                |   Mobile App     |
                +------------------+
                         |
                         v
                +------------------+
                |   API Gateway    |
                +------------------+
                         |
                         v
                +------------------+
                |   Feed Service   |
                +------------------+
                     /        \
                    v          v
        +----------------+  +------------------+
        |   Feed Cache   |  | Ranking Service  |
        |   (Redis)      |  +------------------+
        +----------------+
                    |
                    v
            +------------------+
            |   Feed Storage   |
            | (Post references)|
            +------------------+
                    |
                    v
             +-----------------+
             |   Post Service  |
             +-----------------+
                    |
                    v
             +------------------+
             |  Media Storage   |
             |  + CDN delivery  |
             +------------------+
```

**Write Flow (Post Creation)**

```
User posts photo
      |
      v
Post Service
      |
      v
Event Queue
      |
      v
Fanout Worker
      |
      v
Update followers' Feed Store
```