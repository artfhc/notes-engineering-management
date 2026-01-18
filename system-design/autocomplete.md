# Autocomplete System Design

## Overview

Autocomplete is a high-performance, distributed client-side system that requires careful orchestration of state, caching, and async operations. This guide covers the key architectural and implementation considerations for building production-grade autocomplete.

---

## 1. Autocomplete is a System, not a UI Widget

At a high level, autocomplete is a mini distributed system on the client:

```
Input → Controller → Cache / Network → Results UI
```

**Key insight:**
> The controller is the brain, not the input or the popup.

---

## 2. Separation of Responsibilities

### Core Components

| Component | Responsibility |
|-----------|---------------|
| **Input Field** | Captures keystrokes, focus/blur, keyboard events |
| **Controller** | Orchestrates everything: cache vs network, handles race conditions |
| **Cache** | Stores historical query → results, enables instant responses |
| **Results UI (Popup)** | Renders results, handles selection & navigation |

**Interview takeaway:**
> Never let UI components talk to the network directly.

---

## 3. API Design for Reusability

Autocomplete is often built as a shared component / SDK, so API design matters.

### Basic API Concepts

**Configuration:**
- `apiUrl` - Endpoint for fetching suggestions
- `limit` - Maximum number of results
- `minQueryLength` - Minimum characters before triggering

**Event hooks:**
- `onInput` - Fired on every keystroke
- `onSelect` - Fired when user picks a suggestion
- `onFocus` - Fired when input gains focus

**Rendering customization:**
- Theming tokens
- CSS class injection
- Render callbacks (React-style inversion of control)

**Key insight:**
> This mirrors server-driven UI & platform thinking.

---

## 4. Performance is the Core Constraint

Autocomplete is latency-sensitive and keystroke-driven.

### Key Performance Techniques

| Technique | Purpose |
|-----------|---------|
| **Debouncing** | Avoid firing a request per keystroke |
| **Minimum query length** | Don't query on 1–2 characters |
| **Client-side caching** | Near-zero latency for repeat queries |

**Golden rule:**
> Users perceive autocomplete as "broken" if it's slow by even ~100–200ms.

---

## 5. Handling Race Conditions

### The Problem

```
User types: f → fo → foo
Network returns responses out of order
```

### Correct Solution

1. **Key results by query string**
2. **Always render results matching current input value**
3. **Cache all responses** (don't discard)

### What to Avoid

- ❌ Relying on request order
- ❌ Aborting requests (server already did the work)

**Key insight:**
> This is a real-world concurrency problem, not just frontend trivia.

---

## 6. Cache Design is the Deep-Dive Topic

This is where senior-level answers stand out.

### Cache Strategies

#### Option 1: Query → Results Map

```javascript
{
  "new": [{id: 1, name: "New York"}, ...],
  "new yo": [{id: 1, name: "New York"}]
}
```

**Pros:**
- Simple, fast

**Cons:**
- Memory-heavy (duplicate data)

---

#### Option 2: Flat List

```javascript
[
  {id: 1, name: "New York"},
  {id: 2, name: "Newark"},
  ...
]
```

**Pros:**
- No duplication

**Cons:**
- Expensive client-side filtering

---

#### Option 3: Normalized Store (Best Tradeoff)

```javascript
{
  entities: {
    1: {id: 1, name: "New York"},
    2: {id: 2, name: "Newark"}
  },
  queries: {
    "new": [1, 2],
    "new yo": [1]
  }
}
```

**Pros:**
- Results stored once
- Queries store ID lists
- Memory efficient
- Fast lookups

**Interview framing:**
> Cache design is a space vs time tradeoff, influenced by app lifetime.

---

## 7. Initial Results (Zero-Query UX)

Autocomplete doesn't always start with typing.

### Examples

- **Trending searches** (Google)
- **Recent searches** (Facebook)
- **User history** (e-commerce, travel)

### Implementation Trick

Treat empty string (`""`) as a valid cache key:

```javascript
cache[""] = getTrendingResults()
```

**Key insight:**
> This directly improves conversion & engagement.

---

## 8. Offline & Failure Handling

Good autocomplete handles:

| Scenario | Strategy |
|----------|----------|
| **Network loss** | Fallback to cache |
| **Request failures** | Retries with backoff |
| **Loading states** | Show spinners |
| **Empty states** | Display "No results" |

**Key insight:**
> This is where product polish shows.

---

## 9. Scalability on the Client

### Large Result Sets

- **Virtualized lists**
  - Only render visible rows
  - Avoid DOM explosion
  - Use libraries like `react-window` or `RecyclerView`

### Memory Management

- **Cache eviction**
  - TTL (Time To Live)
  - LRU (Least Recently Used)
- **Idle-time cleanup**
  - Clear cache when app is idle

**Key insight:**
> Especially important for long-lived mobile sessions.

---

## 10. Accessibility Is Not Optional

Autocomplete is a keyboard-first control.

### Key ARIA Attributes

```html
<input
  role="combobox"
  aria-expanded="true"
  aria-autocomplete="list"
  aria-controls="suggestions-list"
/>

<ul id="suggestions-list" role="listbox">
  <li role="option">...</li>
</ul>
```

### Keyboard Navigation

| Key | Action |
|-----|--------|
| **Arrow Down** | Move to next suggestion |
| **Arrow Up** | Move to previous suggestion |
| **Enter** | Select current suggestion |
| **Escape** | Dismiss popup |

### Follow Standards

- [WAI-ARIA Combobox patterns](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/)

**Key insight:**
> Many companies fail here—calling it out is a plus.

---

## 11. UX Details That Matter in Practice

### Critical UX Considerations

- **Autofocus** - Only when intent is high
- **Long strings** - Ellipsis, wrapping, or truncation
- **Mobile tap targets** - Minimum 48dp/44pt touch areas
- **Popup positioning** - Render above input if near screen bottom
- **Disable browser autocomplete**
  ```html
  <input autocomplete="off" autocorrect="off" autocapitalize="off" />
  ```

---

## 12. What Interviewers Are REALLY Testing

### Core Competencies Being Evaluated

| Area | What They're Looking For |
|------|-------------------------|
| **State orchestration** | Can you manage complex async flows? |
| **Async & race conditions** | Do you understand concurrency? |
| **Caching tradeoffs** | Can you optimize for performance vs memory? |
| **API design for reuse** | Can you build reusable, configurable components? |
| **UX + performance coupling** | Do you understand how they affect each other? |

### The Real Question

**Autocomplete is a proxy for:**
> "Can you design interactive, high-performance client systems?"

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│                  User Input                      │
└────────────────┬────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────┐
│              Controller                          │
│  • Debounce logic                               │
│  • Cache check                                   │
│  • Network request orchestration                │
│  • Race condition handling                       │
└────────┬──────────────────────┬─────────────────┘
         │                      │
         ▼                      ▼
    ┌────────┐          ┌──────────────┐
    │ Cache  │          │   Network    │
    │ Store  │          │   API Call   │
    └────┬───┘          └──────┬───────┘
         │                     │
         └──────┬──────────────┘
                │
                ▼
    ┌───────────────────────┐
    │   Results Merger      │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │   Results UI/Popup    │
    │   • Rendering         │
    │   • Keyboard nav      │
    │   • Selection         │
    └───────────────────────┘
```

---

## Implementation Checklist

### Must-Haves

- [ ] Debouncing (300-500ms)
- [ ] Minimum query length (2-3 chars)
- [ ] Client-side caching
- [ ] Race condition handling
- [ ] Loading states
- [ ] Empty states
- [ ] Error handling
- [ ] Keyboard navigation
- [ ] ARIA attributes

### Nice-to-Haves

- [ ] Zero-query results (trending/recent)
- [ ] Virtualized list rendering
- [ ] Cache eviction strategy
- [ ] Offline fallback
- [ ] Analytics/tracking
- [ ] Custom theming
- [ ] Mobile optimizations

---

## Common Pitfalls to Avoid

1. **Making network requests on every keystroke** - Use debouncing
2. **Not handling race conditions** - Always check if response matches current query
3. **Ignoring accessibility** - Add proper ARIA attributes and keyboard support
4. **Poor cache design** - Use normalized store for efficiency
5. **No offline handling** - Always have a cache fallback
6. **Forgetting mobile UX** - Ensure proper touch targets and positioning
7. **Coupling UI to network** - Keep controller as separate orchestration layer

---

## Summary

### Key Takeaways

1. **Architecture:** Separate controller from UI for clean orchestration
2. **Performance:** Debounce + cache = responsive UX
3. **Race conditions:** Key by query string, always render for current input
4. **Cache:** Normalized store is the best tradeoff
5. **UX:** Accessibility and offline support are not optional

### Interview Success Formula

```
Strong autocomplete answer =
  System thinking +
  Performance optimization +
  Concurrency handling +
  API design +
  UX polish
```
