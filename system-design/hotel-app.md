# Hotel App Search Autocomplete Design

## Overview

Design autocomplete integration in a Hotel app search page requires a balance of performance, responsiveness, and UX clarity. This document covers architecture, data flow strategies, and implementation patterns optimized for scalability and mobile best practices.

---

## Architecture Layers

### 1. UI Layer (View)

- **Search Bar:** TextInput component with a debounce mechanism
- **Suggestions List:** RecyclerView (Android) or ListView (iOS) to show autocomplete results
- **Loading/Empty/Error states:** Handle clearly to improve UX

### 2. ViewModel / Presenter Layer

- **SearchQueryLiveData / StateFlow (Android):** Observe query text changes
- **Debounced Fetch Trigger:** Use debounce (300–500ms) to avoid API spamming
- **UI State Management:** Emit states like Loading, Success(data), Empty, or Error

### 3. Domain Layer (Optional, if using Clean Architecture)

- Use a `FetchAutocompleteSuggestionsUseCase`
- Encapsulate logic like filtering local cache, merging server results, etc.

### 4. Data Layer

- **Repository:** Mediates between local cache and remote API
- **Local Cache:** Store recent searches and popular destinations (e.g., Room DB)
- **Remote Source:** Hit the Autocomplete API (e.g., location, hotels, points of interest)

---

## Core Logic & Flow

```
User types "new yo"
        ↓
Debounced input (500ms)
        ↓
ViewModel emits API call intent
        ↓
Repository:
  1. Return cached results instantly (if available)
  2. Async fetch from server
        ↓
Update UI with real-time suggestions
```

---

## Key Considerations

| Concern | Solution |
|---------|----------|
| **Throttling/Debounce** | Avoid sending a network request for every keystroke |
| **Cancellation** | Cancel previous requests when a new one is made (e.g., with CoroutineScope or RxJava) |
| **Offline Fallback** | Serve recent/pinned searches or local points of interest |
| **Rate Limiting** | Ensure you're within API quota for third-party services |
| **Accessibility** | Support screen readers and proper focus on suggestion lists |

---

## Testing Strategy

- **Unit test** ViewModel logic (debounce, API trigger)
- **Instrument UI tests** for input and suggestion rendering
- **Test edge cases** like typing quickly, going offline, or switching apps mid-search

---

## Tech Stack (Android)

| Layer | Tool/Pattern |
|-------|-------------|
| **UI** | Jetpack Compose / XML |
| **ViewModel** | StateFlow + CoroutineScope |
| **Domain** | UseCase pattern |
| **Data** | Retrofit, Room |
| **Cache** | In-memory + Room |
| **DI** | Hilt / Koin |

---

## Data Flow Strategy

Combining prefetching of popular data with on-demand, user-specific autocomplete gives you responsiveness and personalization.

### 1. Initial Data Loading (Prefetch)

**Goal:** Show something meaningful before user types.

**Data Flow:**

```
App Launch or Search Page Opened
        ↓
Repository.prefetchPopularSearchData()
        ↓
Check Local Cache (Room / DataStore / SharedPrefs)
        ↓                     ↓
Cache Hit? Yes → Return popular suggestions to UI
           No  → Fetch from network, cache locally, return to UI
```

**Sources to Prefetch:**

- Trending hotel searches (from analytics)
- Recently viewed by user (local)
- City-level popular queries (region-specific)
- Popular destinations by season or time of day

---

### 2. Local Query Processing

**Goal:** Instantly filter cached results for fast typing UX (low latency).

**Data Flow:**

```
User types → "new"
        ↓
Debounced input (e.g. 300ms)
        ↓
LocalSearchUseCase.filter(query)
        ↓
In-memory or Room cache → Return filtered list (prefix match, fuzzy, etc.)
        ↓
Display suggestions in UI immediately
```

**Considerations:**

- Use prefix or fuzzy match (e.g., `LIKE 'new%'`)
- Use search index on Room to speed it up
- Optionally show recent user searches at top

---

### 3. Network-Based Refinement (On-Demand API)

**Goal:** Fetch highly relevant, real-time suggestions after local fallback.

**Data Flow:**

```
Same debounced user input → triggers async API request
        ↓
Repository.fetchRemoteSuggestions(query)
        ↓
→ Cancel previous request (token/coroutine job)
→ Execute new request
        ↓
Receive results → Update UI (with shimmer if loading)
        ↓
Cache results for short-term reuse
```

**Merge Strategy:**

```kotlin
val combinedResults = merge(
    localFilteredResults,  // shown immediately
    remoteResults          // shown once available, replacing or appended
)
```

---

## Overall Flow Summary

```
Search Page Loads
   └─ Prefetch popular data (local or network) → UI

User types "new"
   ├─ Local filtering from prefetch cache (instant) → UI
   └─ Debounced network request for refined results → UI
        └─ Result replaces or augments previous local data
```

---

## Smart Enhancements

- **Result Deduplication:** Remove overlaps between local & network responses
- **Adaptive caching:** Cache by region or time to keep data fresh
- **User signal learning:** Promote user-clicked terms in future local filtering
- **Fallback if offline:** Always show cached "last known popular" or recent search terms

---

## Changelist Pattern

### What Is the Changelist Pattern?

It involves:

- Maintaining a local cache (DB) of a list (e.g., hotel search results)
- Fetching only the changes from the server (not the entire list)
- Applying additions, deletions, updates based on change metadata (like added, modified, deleted, timestamp, or version)

### Use Case: Hotel Search Autocomplete

Instead of downloading the full list of popular hotels each time:

- Server sends a list of changes (e.g. new trending hotels or updated city names)
- Client merges these into the local Room DB
- Your UI (e.g. a paginated list or search suggestion) reads directly from local DB

---

## Changelist Flow

### 1. Local Cache Initialization

```
Local DB ← empty
```

### 2. Initial Changelist Sync

```
Client → Server: "Give me all changes since version = 0"
Server → Client: [Hotel A (add), Hotel B (add), Hotel C (add)]
Client applies these to local DB.
→ Now local DB is hydrated.
```

### 3. Incremental Sync

```
Later...
Client → Server: "Give me changes since version = 3"
Server → Client: [Hotel B (modified), Hotel D (add), Hotel A (delete)]
Client updates local DB accordingly.
```

---

## Benefits of Changelist Pattern

| Benefit | Description |
|---------|-------------|
| ✅ Efficient sync | Avoids downloading large payloads repeatedly |
| ✅ Good offline support | UI reads from local DB; syncing happens in background |
| ✅ Works well with paging | Easily integrates with paging libraries (e.g. Android Paging3) |
| ✅ Scalable for large lists | Only syncs deltas, not full snapshots |

---

## Implementation

### On the Server:

- Maintain a change log or versioned data store
- Respond to client with: `item_id`, `operation`, `timestamp/version`, `payload`

### On the Client:

- Store last known `change_token` or `version`
- Periodically (or on trigger) request for changes
- Apply diffs to local DB (using Room or similar)
- Let UI observe local DB (e.g. via Flow or LiveData)

### Android Tech Stack

- Room DB (for cache)
- Coroutines + Paging3 (for streaming paginated data)
- WorkManager (for background syncing)
- ChangeList Sync Layer (e.g. in a Repository)

---

## API Design: GET /autocomplete/changes

### Request

```
GET /autocomplete/changes?since=123
```

**Query Params:**
- `since`: the last known changelist version (e.g. from Room or SharedPrefs)

### Response

```json
{
  "changes": [
    {
      "id": "tokyo",
      "operation": "update",
      "name": "Tokyo",
      "type": "city",
      "country": "Japan"
    },
    {
      "id": "paris",
      "operation": "delete"
    },
    {
      "id": "newyork",
      "operation": "insert",
      "name": "New York",
      "type": "city",
      "country": "USA"
    }
  ],
  "latestVersion": 126
}
```

### Supported Operations

| Operation | Description |
|-----------|-------------|
| `insert` | New item added |
| `update` | Existing item changed |
| `delete` | Item removed from the list |

---

## Client Flow

1. Client stores last known version (e.g., 123)
2. On app start or sync trigger:
   - Calls `/autocomplete/changes?since=123`
3. Parses and applies changes:
   - `insert` → add to DB
   - `update` → update local row
   - `delete` → remove from DB
4. Update stored version to 126

---

## Local Cache Schema (Room)

```kotlin
@Entity(tableName = "autocomplete_suggestions")
data class AutocompleteSuggestion(
    @PrimaryKey val id: String,
    val name: String,
    val type: String,
    val country: String
)

@Entity(tableName = "metadata")
data class ChangelistMetadata(
    @PrimaryKey val key: String = "autocomplete",
    val lastVersion: Long
)
```

**Benefits:**
- Efficient: syncs only deltas, not full list
- Reliable: supports offline mode, retries
- Scalable: handles thousands of items with small payloads

---

## Full Text Search (FTS)

### Core Idea

Instead of doing a naive:

```sql
SELECT * FROM hotels WHERE name LIKE '%new%'
```

FTS uses a specialized index (an inverted index) to match words and phrases, enabling fast, relevant text queries.

---

## How FTS Works

### 1. Tokenization

Input text is broken into words (tokens).

```
"New York Hotel" → ["new", "york", "hotel"]
```

### 2. Inverted Index

For each word, store a mapping of which document (or row) contains it.

**Example:**
```
"new"   → [doc1, doc3]
"york"  → [doc1]
"paris" → [doc2]
```

### 3. Query Parsing

User query is tokenized and optionally transformed (e.g., stemming, lowercasing).

### 4. Search Execution

Find documents that match one or more tokens, rank based on relevance.

---

## FTS in SQLite (Android / iOS)

SQLite provides FTS3, FTS4, and FTS5 extensions for full-text search.

### Example Table:

```sql
CREATE VIRTUAL TABLE hotel_search USING fts4(
  name TEXT,
  city TEXT,
  country TEXT
);
```

### Insert:

```sql
INSERT INTO hotel_search (name, city, country)
VALUES ('New York Marriott', 'New York', 'USA');
```

### Query:

```sql
SELECT * FROM hotel_search WHERE hotel_search MATCH 'new';
```

### Advanced Queries:

```sql
-- Match documents containing both "new" and "york"
MATCH 'new york'

-- Exact phrase
MATCH '"new york"'

-- Partial match (FTS5 only)
MATCH 'new*'
```

---

## FTS Benefits

| Feature | Description |
|---------|-------------|
| 🔎 Fast lookup | Uses index instead of scanning all rows |
| 💬 Phrase search | "new york hotel" vs. separate word match |
| 📊 Ranking | Some engines (like FTS5) support relevance scoring |
| ✂️ Tokenization | Supports language-specific tokenization |

**When To Use:**
- Autocomplete or search suggestions
- Searching hotel names, destinations, descriptions
- Filtering large local datasets efficiently

---

## Alternatives to FTS

### 1. LIKE or ILIKE Queries

```sql
SELECT * FROM hotels WHERE name LIKE '%new%';
```

**Pros:**
- Simple and built-in to all SQL engines
- No need for virtual tables or extensions

**Cons:**
- Slow for large datasets (no index support for `%prefix%` searches)
- No stemming, typo tolerance, or ranking
- Case-sensitive unless handled explicitly

---

### 2. Prefix Matching with Index Support

```sql
SELECT * FROM hotels WHERE name LIKE 'new%';
```

**Pros:**
- Can use B-tree indexes → much faster than `%term%`
- Works well for autocomplete

**Cons:**
- Limited to prefix matching
- No relevance/ranking, no typo tolerance

---

### 3. Local Filtering in Memory

**Flow:**
- Load a reasonable number of rows into memory (e.g. top 1,000 hotels)
- Use in-memory string filtering in Kotlin/Swift/JS

```kotlin
val results = hotelList.filter { it.name.contains("new", ignoreCase = true) }
```

**Pros:**
- Great for small-to-medium datasets
- Fast, responsive UI
- Easy to combine with client-side scoring or sorting

**Cons:**
- Doesn't scale for large datasets
- Must manage memory carefully

---

### 4. Client-side Indexing (Trie or Map)

For advanced local filtering:
- Build a Trie for prefix searches
- Use a `Map<String, List<Hotel>>` where the key is a normalized word

**Pros:**
- Blazing fast lookups
- Works offline
- Full control of logic (e.g., tokenization, stemming)

**Cons:**
- Requires custom code
- Higher memory cost
- Needs regular updates if source data changes

---

### 5. Offload to Server-Side Search

Let the backend handle FTS (e.g., Elasticsearch, Meilisearch, Solr):

```
GET /api/search?q=new
```

**Pros:**
- Powerful ranking, typo correction, language support
- Scalable for large data

**Cons:**
- Requires network and backend infra
- Can't fully work offline

---

## Search Method Comparison

| Method | Fast | Works Offline | Ranked Results | Easy to Implement | Scales |
|--------|------|---------------|----------------|-------------------|--------|
| FTS (SQLite) | ✅ | ✅ | ✅ | ❌ (extra setup) | ✅ |
| LIKE queries | ⚠️ | ✅ | ❌ | ✅ | ⚠️ |
| Prefix + Index | ✅ | ✅ | ❌ | ✅ | ⚠️ |
| In-memory filter | ✅ | ✅ | ❌ | ✅ | ❌ |
| Trie/Map Indexing | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| Server-side Search API | ✅ | ❌ | ✅ | ⚠️ | ✅ |

---

## Trie-like Schema in SQLite

### Option 1: Flat Table with Indexed Text Column (Simplest)

```sql
CREATE TABLE suggestions (
  id INTEGER PRIMARY KEY,
  term TEXT NOT NULL
);

CREATE INDEX idx_term_prefix ON suggestions(term);
```

**Query:**

```sql
SELECT * FROM suggestions WHERE term LIKE 'new%';
```

**Pros:**
- Very simple
- Index on term makes prefix search fast

**Cons:**
- Not a real Trie — you can't do partial-tree traversal or stepwise expansion
- Can't efficiently build search suggestions letter-by-letter for large datasets

---

### Option 2: Trie-like Schema with Parent-Child Node Representation

```sql
CREATE TABLE trie_nodes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  parent_id INTEGER REFERENCES trie_nodes(id),
  char TEXT NOT NULL,
  is_terminal BOOLEAN DEFAULT 0,
  word TEXT -- optional: store full word only at terminal nodes
);
```

**Sample Data (for words: "new", "net", "news"):**

| id | parent_id | char | is_terminal | word |
|----|-----------|------|-------------|------|
| 1 | NULL | n | 0 | NULL |
| 2 | 1 | e | 0 | NULL |
| 3 | 2 | w | 1 | "new" |
| 4 | 2 | t | 1 | "net" |
| 5 | 3 | s | 1 | "news" |

**Step-by-step traversal:**

To find all words starting with "new":
1. Find node chain for n → e → w
2. Traverse all children from node w to find terminal words

**SQL with Recursive CTE:**

```sql
WITH RECURSIVE trie(id, char, word, is_terminal, path) AS (
  SELECT id, char, word, is_terminal, char AS path
  FROM trie_nodes
  WHERE parent_id IS NULL

  UNION ALL

  SELECT n.id, n.char, n.word, n.is_terminal, t.path || n.char
  FROM trie_nodes n
  JOIN trie t ON n.parent_id = t.id
)
SELECT word FROM trie WHERE path LIKE 'new%' AND is_terminal = 1;
```

---

## When to Use Each Approach

| Use Case | Recommended Schema |
|----------|-------------------|
| Small to medium dataset | Flat table with `LIKE 'prefix%'` |
| Large dataset, stepwise traversal | Trie node schema with recursive CTE |
| Server does search | Offload to Elasticsearch or Meili |

---

## Hybrid Model (Recommended for Mobile)

Precompute prefixes and flatten them for speed:

```sql
CREATE TABLE autocomplete_index (
  prefix TEXT PRIMARY KEY,
  word TEXT
);
```

**Example Data:**

| prefix | word |
|--------|------|
| n | new |
| ne | new |
| new | new |
| new | news |

### Why the Hybrid Model Is Best

| Concern | Flat Table | Trie Schema | Hybrid |
|---------|------------|-------------|--------|
| **Speed** | ✅ Fast for small | ⚠️ Slower (joins) | ✅ Fast for prefix |
| **Simplicity** | ✅ Easiest | ❌ Complex | ⚠️ Moderate |
| **Scalability** | ⚠️ Limited | ✅ Scales well | ✅ Scales well |
| **Mobile-Friendly** | ✅ | ❌ Recursive CTEs can be expensive | ✅ Yes |
| **Offline Use** | ✅ | ✅ | ✅ |

---

## Hybrid Approach Implementation

### How It Works

1. Precompute all prefixes for each word in your source data
2. Store each prefix → full word mapping

### Example Table

```sql
CREATE TABLE autocomplete_index (
  prefix TEXT,
  word TEXT
);

CREATE INDEX idx_prefix ON autocomplete_index(prefix);
```

### Sample Data

| prefix | word |
|--------|------|
| n | new |
| ne | new |
| new | new |
| new | newark |
| new | news |
| newy | new york |

### Query

```sql
SELECT DISTINCT word
FROM autocomplete_index
WHERE prefix = 'new'
ORDER BY word
LIMIT 10;
```

---

## Handling Multiple Keywords per Prefix

If each prefix maps to multiple keywords, that's expected in an autocomplete system.

### Example Data

| prefix | keyword |
|--------|---------|
| n | new |
| ne | new |
| new | new |
| new | newark |
| new | news |
| new | new york |
| newy | new york |
| newyo | new york |

### Query Result

```sql
SELECT DISTINCT keyword
FROM autocomplete_index
WHERE prefix = 'new'
ORDER BY keyword
LIMIT 10;
```

**Result:**
```
new
new york
newark
news
```

---

## Optional Enhancements

### 1. Add Weight or Popularity Score

```sql
CREATE TABLE autocomplete_index (
  prefix TEXT,
  keyword TEXT,
  score INTEGER
);

SELECT keyword
FROM autocomplete_index
WHERE prefix = 'new'
ORDER BY score DESC
LIMIT 10;
```

### 2. Support Partial Tokens in Compound Terms

If the keyword is "new york hotel":
- You can generate prefixes for "new", "york", and "hotel" if token-based suggestions are preferred

---

## Reservation Timer Design

### Challenge

Implement a fixed booking window for hotel reservation where:
1. Backend timestamp serves as a single source of truth for tracking reservation holds
2. Client displays the countdown by combining the backend expiration timestamp with local device uptime measurement

---

## Why Use Uptime Instead of System Clock?

### Problem with `System.currentTimeMillis()`

- It can be manipulated by changing the device's system clock
- It can cause jumps forward/backward, breaking the timer

### Solution: Monotonic Clock (Uptime)

Use `SystemClock.elapsedRealtime()` on Android or `ProcessInfo.systemUptime` on iOS:

- It's based on device uptime since boot
- Never changes, not affected by system clock
- Ideal for measuring durations

---

## Timer Implementation

### Step 1: When Backend Responds

```json
{
  "expiresAt": 1721503000000  // in epoch millis (UTC)
}
```

### Step 2: Client Calculates

```kotlin
val expiresAt = 1721503000000L
val receivedAtWallClock = System.currentTimeMillis()
val receivedAtUptime = SystemClock.elapsedRealtime()
```

### Step 3: Compute Expiration Uptime

```kotlin
val delta = expiresAt - receivedAtWallClock
val expiresAtUptime = receivedAtUptime + delta
```

### Step 4: On Each Timer Tick

```kotlin
val remainingMillis = expiresAtUptime - SystemClock.elapsedRealtime()
```

This gives you:
- A precise countdown (immune to wall clock changes)
- Consistency even if the user backgrounded the app

---

## Example Calculation

```
expiresAt = 12:00:00 UTC = 1721502000000
receivedAtWallClock = 11:59:00 UTC = 1721501940000
receivedAtUptime = 100000 ms

val delta = 60000 // 60 seconds
val expiresAtUptime = 100000 + 60000 = 160000

Timer ticks every second:
val remaining = expiresAtUptime - SystemClock.elapsedRealtime()
```

---

## Benefits of This Approach

| Advantage | Why It Matters |
|-----------|---------------|
| Immune to system clock change | Users can't "cheat" the timer |
| Works offline | No need to sync again after loading |
| Precise countdown | Ideal for tight booking expirations |

---

## Summary

### Key Takeaways

1. **Architecture:** Use MVVM with clean separation of concerns
2. **Data Strategy:** Combine prefetching, local filtering, and network refinement
3. **Changelist Pattern:** Sync only deltas for efficiency
4. **Search:** Use FTS or hybrid prefix index for fast autocomplete
5. **Timer:** Use monotonic clock (uptime) for accurate countdowns

### Best Practices

- Debounce user input (300-500ms)
- Cancel previous requests when new ones are made
- Cache aggressively for offline support
- Use WebSocket or polling for real-time updates
- Test edge cases thoroughly
