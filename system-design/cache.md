# Cache: Common Hard Problems

## Cache Stampede (Thundering Herd)

**Problem:** hot key expires → thousands of misses hit DB simultaneously

**Fixes:**
- Singleflight lock per key
- Early refresh (soft TTL)
- Add jitter to TTL
- Serve stale while revalidating

---

## Cache Penetration

**Problem:** repeated requests for nonexistent keys bypass cache entirely

**Fixes:**
- Cache negative results (short TTL)
- Bloom filter before DB

---

## Cache Inconsistency

**Problem:** DB updated but cache still serves old data

**Fixes:**
- Invalidate on write
- Use versioned keys (e.g., `user:123:v7`)
- Event-driven invalidation (CDC/Kafka)

---

## Choosing Patterns by System

### Feed
- Cache-aside for post objects / user profiles
- Often hybrid TTL + refresh
- Avoid strict invalidation across many derived keys

### Search
- Cache-aside for query results (short TTL)
- Cache: top queries, facets, spell/auto-complete
- Use result caching + per-doc caching separately

### Chat
- Cache recent messages per room/user
- Read-through can be nice for "conversation summary state"
- Writes usually go to DB/log first; cache is acceleration

---

## Cache Invalidation vs TTL-only

### A) TTL-only
- You accept staleness up to TTL
- Best for: feeds, recommendations, non-critical product metadata

### B) Explicit Invalidation
- On write, delete/update related keys
- Best for: profile edits, pricing-ish, inventory-ish (still careful)
- **Gotcha:** invalidation is hard with fanout keys (e.g., feed inboxes)

---

## Cache Pattern Comparison

| Pattern | Read Path | Write Path | Consistency | When to Use |
|---------|-----------|------------|-------------|-------------|
| Cache-aside | Cache → DB on miss | DB then invalidate | Eventual-ish | Most apps |
| Read-through | Cache fetches DB on miss | App writes DB (varies) | Depends | Cleaner app code |
| Write-through | Cache then DB sync | Cache+DB sync | Stronger | Must-read-your-writes |
| Write-behind | Cache now, DB later | Async flush | Eventual | Very high write / buffering |
