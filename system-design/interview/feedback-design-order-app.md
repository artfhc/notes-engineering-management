# Interview Feedback: Design an Order Tracking App

## 1. First Impression (Interviewer POV)

Your design shows good senior Android instincts:

- ✔ Clear functional requirements
- ✔ Correct separation (VM → Repo → DB/network)
- ✔ Considered SSE vs WebSocket tradeoff
- ✔ Thought about offline + stale data
- ✔ Mentioned UI jank mitigation

But from an E6 / EM level, the interviewer expected more in these areas:

- ❗ State model
- ❗ Realtime data architecture
- ❗ Lifecycle / reconnection handling
- ❗ Map rendering strategy
- ❗ Concurrency + stream merging

These are the actual hard parts of order tracking.

## 2. Requirements Gaps

You listed: order status, contact driver, driver location, item list. Good.

But major real product requirements missing:

**Missing: ETA updates** — ETA constantly changes.

```
driver location -> route -> ETA recalculation
```

Your model treats ETA as static. Better:

```
ETA {
   estimatedArrivalTime
   trafficDelay
   routeDistance
}
```

**Missing: Order state machine** — your status enum is too simple. Real system:

```
PLACED
CONFIRMED
PREPARING
PICKED_UP
DRIVER_EN_ROUTE
ARRIVING
DELIVERED
CANCELLED
FAILED
```

Why this matters: progress bar mapping (`status -> UI step`).

**Missing: Driver identity** — you showed driver name but not model. Need:

```
Driver {
  id
  name
  rating
  vehicleType
  phone
  avatar
}
```

**Missing: Map route** — user experience depends on route. Map shows `restaurant -> driver -> user`. You didn't model route.

## 3. API Design Critique

Your API: `GET /v1/order/{order_id}` — good, but missing critical fields.

**Better response:**

```
Order {
  id
  status
  restaurant
  items[]
  totalPrice
  driver
  deliveryLocation
  driverLocation
  eta
}
```

**Problem: location in separate SSE only.** You split REST → order and SSE → location, but status should also stream. Example: driver picked up order → UI must update instantly.

**Better SSE event types:**

```
DRIVER_LOCATION
ORDER_STATUS
ETA_UPDATE
DRIVER_ASSIGNED
ORDER_CANCELLED
```

Your stream only had `connect`, `disconnect`, `location` — too limited.

## 4. Realtime Architecture (Biggest Gap)

This is the core of the problem.

You discussed SSE vs WebSocket, but missed the real issue: **event fan-out system**.

**Real architecture:**

```
Driver App
    ↓
Location Service
    ↓
Kafka
    ↓
Order Tracking Service
    ↓
SSE / WebSocket
    ↓
Customer App
```

The interviewer hinted this indirectly. Even though the question was "Android", senior candidates usually mention backend assumptions briefly.

## 5. Android Architecture

**Your architecture:**

```
UI
↓
ViewModel
↓
OrderRepo
LocationRepo
↓
Network + Local DB
```

Good, but the interviewer pushed you because this is basic MVVM. He wanted deeper architecture discussion.

**Better architecture:**

```
UI
↓
ViewModel
↓
UseCases
  ├─ ObserveOrderStatus
  ├─ ObserveDriverLocation
  └─ FetchOrderDetails
↓
Repository
↓
DataSources
  ├─ Remote
  └─ Local
```

Why: better separation of concerns.

## 6. Stream Handling (Major Weakness)

**Your solution:** `SSE -> DB -> poll DB -> UI` — this is problematic. You intentionally created DB polling; interviewer questioned UI jank.

**Correct solution:** `SSE -> Flow -> throttle -> UI`

```kotlin
sseFlow
   .sample(2.seconds)
   .collect { updateMap(it) }
```

No DB polling needed.

## 7. Map Rendering Problem

You missed a key Android challenge: updating map marker smoothly.

**Naive approach:**
```kotlin
setMarkerPosition(newLatLng)
// Result: marker jumps
```

**Correct approach:**
```kotlin
animateMarker(from, to)
// Use ValueAnimator
```

Interviewer likely expected this to be mentioned.

## 8. Lifecycle Handling (Important)

You didn't discuss lifecycle. Must handle:

- Activity paused
- Activity resumed
- Process death

**Example:**
```
onResume -> reconnect SSE
onPause -> stop stream
```

Also: retry with exponential backoff.

## 9. Offline Strategy

**Your solution:** store last location in DB — good idea.

**But missing: staleness indicator.**

```
lastUpdateTime

UI rule:
if now - lastUpdate > 60s
   show "location outdated"
```

You mentioned this verbally but didn't model it.

## 10. Race Condition Discussion

You mentioned: driver keeps sending location after delivery. That's not the real race condition.

**Real one:**
```
order delivered event arrives
location event arrives after
```

**Solution:** ignore location events if `order status = DELIVERED`.

## 11. SSE vs WebSocket Answer

Your reasoning: SSE lighter, WebSocket heavier. Interviewers rarely care about this.

**Better answer** — use WebSocket because it supports:
- Status updates
- Location updates
- Chat
- Driver contact

Future extensibility matters more than protocol weight.

## 12. What the Interviewer Was Testing

Key signals from transcript. When interviewer said:

> "I'm interested in the architecture and tricky parts"

He wanted discussion of:

1. Stream management
2. UI state management
3. Map rendering
4. Lifecycle handling

You focused on repository patterns instead.

## 13. Your Best Moment

This part was strong — **debouncing UI updates**.

You said:

> "too many updates could make the map janky"

This is very good Android insight. But the implementation should be Flow debounce, not DB polling.

## 14. Ideal High-Level Answer

**Architecture:**

```
OrderTrackingScreen
       │
       ▼
ViewModel
       │
       ▼
OrderTrackingUseCase
       │
       ▼
Repository
       │
       ├── REST API (order snapshot)
       └── WebSocket/SSE (live updates)
```

**Data streams:**

```kotlin
orderFlow
driverLocationFlow
etaFlow

// Combine:
combine(orderFlow, locationFlow)
```

**UI state:**

```kotlin
TrackingUIState // sealed class
```

## 15. Final Score (Realistic)

If this was DoorDash EM / Staff Android round:

| Area | Score |
|------|-------|
| Requirements | 7/10 |
| API design | 7/10 |
| Android architecture | 7/10 |
| Realtime system | 5/10 |
| Edge cases | 7/10 |
| Communication | 8/10 |

**Overall: ~7/10** — borderline pass depending on bar.

## 16. What to Put on the Whiteboard

You can add a model like this:

```
Order {
  id: String
  items: List<Item>
  subtotal: Money
  deliveryFee: Money
  tax: Money
  total: Money
}

Item {
  name: String
  quantity: Int
  unitPrice: Money
  lineTotal: Money
}

Money {
  amountMinor: Long
  currencyCode: String
}
```

And next to it:
- Client formats using locale
- Server owns final billing math

That would look much stronger than `priceString`.

### What Was Weak in Your Original Answer

You said something close to: "maybe priceString", "maybe backend formats it", "maybe client calculates total" — that felt undecided.

**Stronger version:**
- Backend returns structured `Money`
- Backend computes totals
- Client formats per locale
- No floating point
- Currency code required

---

## 17. How to Upgrade This Answer to 9/10

Add discussion of:

1. **Event streams** — Flow, debounce, combine
2. **Lifecycle** — connect, disconnect, retry
3. **Map rendering** — smooth animation
4. **UI state model** — `sealed class TrackingState`
5. **Stream merging** — status + location + ETA
