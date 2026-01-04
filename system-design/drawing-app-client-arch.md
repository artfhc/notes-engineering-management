# Drawing App Client Architecture

## 1. Core Client Goals

* **Instant local response** (optimistic updates)
* **Deterministic state** (can replay ops)
* **High-performance rendering** (big canvas, many objects)
* **Robust sync** (reconnect, gaps, offline)
* **Great UX** (undo/redo, selection, snapping, zoom/pan)

---

## 2. State Layering (4-Layer Mental Model)

### A) UI / View State (ephemeral)

Not persisted; not synced.

* Current tool (select/pen/text)
* Hover, selection, multi-select box
* Camera: zoom/pan, viewport rect
* In-progress gesture (dragging, resizing)
* Local-only preferences (grid on/off)

### B) Document State (authoritative)

This is "the canvas content."

* `objectsById`: Map of object_id → object
* Z-order / groups
* Assets (image refs)
* Tombstones (deleted objects)

**This is what ops mutate.**

### C) Sync State (network + concurrency)

* `serverRevision` (last committed rev)
* `pendingOps[]` (local edits not yet committed)
* `inflightBatches[]` (sent but not acked)
* `seenOpIds` (dedupe)
* Connection state, last heartbeat time

### D) Persistence State (offline storage)

* Latest snapshot + ops since snapshot
* Pending ops queue (durable)
* Cached assets (images/thumbnails)

**Storage:**
* Web: IndexedDB
* Mobile: SQLite/Room/CoreData

---

## 3. Data Structures (Performance)

### Object Store

Use normalized structures:

* `Map<objectId, Object>`
* Spatial index optional (quadtree / R-tree) when object count grows
* Maintain bbox per object for fast hit-test and viewport culling

### Derived/Render State

**Do NOT recompute everything each frame.**

* Maintain derived caches (e.g., flattened render list, computed transforms)
* Invalidate only what changed (dirty flags)

---

## 4. Input Pipeline (Mouse/Touch/Stylus)

Gesture abstraction layer that outputs semantic intents:

* `PointerDown` → pick tool + hit-test
* `PointerMove` → update in-progress transform (drag/resize/draw path)
* `PointerUp` → finalize

### Key Practices

* Use pointer events unified model on web
* Use platform-native gesture detectors on mobile (pan/scale/rotate)
* Split camera gestures (two-finger pan/zoom) vs object gestures (drag/resize)

**Important:** Avoid producing an op on every pixel movement. Use local transient state while moving, then commit ops intelligently.

---

## 5. Rendering Architecture (60fps)

### Three Common Approaches

**Option 1: Canvas 2D (common)**
* Draw all shapes each frame (with culling)
* Fastest to implement
* Great for "whiteboard" style

**Option 2: SVG / DOM (simpler but can get heavy)**
* Easy selection handles via DOM
* Can struggle with many objects

**Option 3: WebGL / Skia (best scale, more complex)**
* tldraw leans toward performance-optimized rendering patterns
* On mobile you might use Skia (RN Skia) or native drawing surfaces

### Interview Answer

> "Start with Canvas2D + culling; upgrade to WebGL/Skia if object count/complexity demands it."

### Also Mention

* Viewport culling (draw only objects whose bbox intersects viewport)
* Level-of-detail (simplify far-away details)
* Image caching + downscaled thumbnails

---

## 6. Op Generation (Batching + Coalescing)

### During a Drag

* Locally update object position every frame (smooth)
* Coalesce updates into fewer ops to send

### Common Strategy

* Send at ~10–20Hz (every 50–100ms) while dragging
* Always send a final "commit" at pointer up

### Collapse Intermediate Ops

For the same object property:

* **Move updates:** keep only the latest (x,y) for the interval
* **Resize updates:** keep only the latest size
* **Freehand stroke:** accumulate points locally, send chunked segments

### This Reduces

* Network usage
* Server write amplification
* Remote clients' render churn

---

## 7. Sync Integration (Local + Remote Ops)

### Clean Approach: Base + Overlay

* Apply remote committed ops to the **base document**
* Apply local pending ops as an **overlay**
* UI renders `base ⊕ overlay`

### When Remote Ops Arrive

1. Apply remote ops to base
2. Rebase overlay (replay pending ops) to produce new overlay

### This Gives You

* Stable UX (local edits never "flash" away)
* Correct convergence

---

## 8. Undo/Redo (Pragmatic Way)

### Two Kinds

**Local UX undo (most important)**
* User expects undo of their last action
* Keep a local stack of "commands" (semantic ops) for immediate undo

**Implementation:**
* Undo emits compensating ops (inverse patch) into pending queue
* Redo similarly

**Global history / time travel (optional)**
* "Restore to revision R" uses snapshots/op replay
* Often implemented as "create a new revision that matches state at R"

---

## 9. Offline-First Behavior (Mobile Credibility)

### What to Store Locally

* Last snapshot (compressed)
* Ops since snapshot (optional)
* Pending ops queue (must-have)

### Offline Editing

* Keep producing ops locally
* Mark them pending and durable

### Reconnect

1. Catch up remote ops
2. Rebase local pending
3. Resend batches with idempotency keys

**Also mention:** Image assets upload on reconnect; show local placeholder until committed

---

## 10. Client Observability & Debugging (EM Bonus)

* Record "sync timeline" (rev numbers, pending queue size)
* Show connection status + last synced time
* "Export diagnostic bundle" (snapshot + recent ops + device info)
* Feature flags to simulate latency / packet loss

---

## 11. Client Architecture Soundbite

> "The client is local-first: it keeps a normalized object store, applies edits optimistically, batches semantic ops to the server, renders via a fast pipeline with viewport culling, persists snapshots + pending ops for offline use, and merges remote ops deterministically with rebase so all devices converge."

---

## 12. Cross-Platform Architecture

### Core Layer (Platform-Agnostic)

Share as much code as possible across React web and native apps.

**Core modules (same design across platforms):**

**Document Model**
* Object store (`Map<objectId, Obj>`)
* Schema + validation
* Serialization (snapshot)

**Op Engine**
* Apply op (pure function)
* Invert op (undo)
* Compose/coalesce ops

**Sync Engine**
* Revision tracking
* Pending ops queue
* Rebase logic
* Idempotency/dedupe

**Selection & Hit Testing**
* Bbox-based hit testing
* Handles (resize/rotate)
* Snapping rules (optional)

**Command System**
* `MoveCommand`, `ResizeCommand`, `CreateShapeCommand`, etc.
* Produces ops; supports undo/redo cleanly

### Interview Soundbite

> "I keep all correctness logic in a deterministic core: apply ops, rebase, undo/redo, serialization. The UI layer becomes mostly input → commands → render."

---

## 13. Web (React) Architecture

### State Management

Split store into:

* **DocStore** (canvas objects + revision)
* **UIStore** (tool, selection, camera)
* **SyncStore** (pending/inflight, connection status)

**Implementation options:** Zustand / Redux Toolkit / Jotai (any is fine)

**Key:** Doc state updates must be fast and incremental

### Rendering Options

* **Canvas2D** (best for large counts + performance)
* **SVG** (simpler but can get heavy)
* **WebGL** (later)

**Architecture:**
* React for UI chrome (toolbar, panels)
* Canvas rendering loop outside React:
  * `requestAnimationFrame` draws from the current doc state
  * React updates state; renderer reads it

**Big point:** Avoid re-rendering React components for every pointer move

### Input Handling

* Use Pointer Events (mouse + touch + pen)
* Convert pointer events to semantic intents:
  * Drag object
  * Draw stroke
  * Pan/zoom camera
* Use coalescing:
  * Local updates every frame
  * Network ops at 10–20Hz + final commit

---

## 14. Native iOS/Android Architecture

### Shared Mental Model

Keep the same state layers:

* UI state (tool, selection, camera)
* Document state (objects)
* Sync state (revision, pending)
* Persistence state (snapshot, pending ops)

### Rendering Choices

**Option A: Native rendering (recommended)**

* **iOS:** CoreGraphics / Metal (later)
* **Android:** Canvas / RenderNode (later)
* Use your own render loop / invalidate per frame while interacting
* **Pros:** Top perf, best battery
* **Cons:** More code per platform

**Option B: Cross-platform rendering engine**

* Skia-based (mention as alternative)
* Good if you want identical rendering behavior

### Gesture Handling

* **iOS:** UIGestureRecognizers
* **Android:** ScaleGestureDetector + custom touch handling

**Always split:**
* One-finger drag = select/move/draw
* Two-finger = pan/zoom camera
* Stylus = draw precision (if supported)

### Mobile Lifecycle Realities

* App backgrounded → sockets drop
* Process may be killed

**You must persist:**
* Latest snapshot
* Pending ops queue
* Last seen revision

---

## 15. Offline-First Persistence (Web vs Mobile)

### Web: IndexedDB

```text
drawing_id -> snapshot
drawing_id -> pendingOps[]
(optional) cached ops segments
```

### Mobile: SQLite

```sql
snapshots(drawing_id, rev, blob, created_at)
pending_ops(drawing_id, op_id, base_rev, payload, created_at)
assets(asset_id, local_path, status)
```

**Key:** Queue must survive crashes

---

## 16. Sync Engine Details (Stack-Specific)

### Web

* WebSocket for push
* REST for ops commit + catch-up
* Heartbeat/ping to detect disconnect quickly

### Mobile

* WS while foreground
* When backgrounded:
  * Stop WS
  * Optionally periodic sync (background fetch) depending on OS constraints
* On resume:
  * Catch up from `last_seen_rev`
  * Rebase pending ops
  * Resend

**Operational design tip:** Treat WS as optimization; REST is correctness

---

## 17. Asset Pipeline for Images

### When User Inserts an Image

1. Create local placeholder object (`status=uploading`)
2. Request signed upload URL
3. Upload directly to storage
4. Commit asset metadata
5. Send op updating the image object with final URL + dimensions

### On Mobile, You Also Need

* Local caching
* Thumbnail generation

---

## 18. Performance Tactics (Interviewers Like)

### Rendering

* Viewport culling using bbox
* Dirty rectangles / layers (optional)
* Keep "in-progress transform" separate from committed doc state to reduce churn

### State Updates

* Avoid deep clones
* Keep object store normalized
* Patch updates per object

### Network

* Coalesce ops (keep last move/resize per object per tick)
* Chunk strokes
* Compress op payloads (gzip / msgpack later)

---

## 19. Testing Strategy (EM Credibility)

### Core Tests

* Apply-op and rebase logic: pure unit tests

### Golden Tests

* Snapshot + ops replay should match expected state

### Property Tests (Nice-to-Have)

* Applying ops in order yields convergence

### Sync Simulation Tests

* Drop/reorder/duplicate messages

---

## 20. Detailed State Architecture

### Recommended Split (Web + iOS + Android)

**A. Document Store (authoritative)**
* `objectsById: Map<string, TLObject>`
* `order: string[]` (z-order) or per-layer arrays
* `assetsById`
* Tombstones for deletes
* `serverRevision`

**B. UI Store (ephemeral)**
* Tool mode, selection, hover
* Camera zoom/pan, viewport rect
* Current gesture state

**C. Sync Store**
* `pendingOps[]` (not yet committed)
* `inflightBatches[]`
* `lastAckedRevision`
* `connectionStatus`
* `seenOpIds` (dedupe / idempotency)

**D. Persistence Store**
* Snapshot + pending queue in IndexedDB/SQLite
* Asset cache index

### Big Insight Interviewers Like

Don't make React (or SwiftUI/Compose) the source of truth for the canvas.

* UI framework renders controls
* Canvas renderer reads a store designed for high-frequency updates
* **Web:** Zustand/Redux for UI + custom doc store for perf
* **iOS:** ViewModel for UI + core store for doc state
* **Android:** ViewModel + core store

---

## 21. Rendering Pipeline (60fps Without Killing UI Thread)

### Web (React)

**Pattern:** React for chrome; Canvas2D/WebGL draws outside React

Use a `CanvasRenderer` object with:
* `render(state, viewport)`
* `invalidate()` scheduling via `requestAnimationFrame`

**On pointer move:**
* Update a small "interaction state" (not a full rerender)
* Renderer reads from store and draws

**Optimizations:**
* Viewport culling: draw only objects intersecting viewport
* "Dirty" flags: only redraw layers affected
* Text measurement cache (expensive on web)
* Image bitmap caching (`createImageBitmap`, offscreen canvas if available)

### iOS (Native)

* **Rendering:** CoreGraphics (simple) → Metal (advanced)
* Keep drawing on main thread initially; move heavy work to background if needed
* Use cached paths / precomputed glyph layouts for text
* Throttle redraw while panning/zooming

### Android (Native)

* **Rendering:** Android Canvas (simple) → RenderNode/OpenGL (advanced)
* Precompute shape paths
* Avoid allocating objects inside `onDraw()`
* Use `Choreographer` for frame sync

### Soundbite

> "I separate UI rendering from canvas rendering; the canvas loop is optimized with viewport culling, caching, and coalesced updates."

---

## 22. Input & Gestures (Unify Mouse/Touch/Stylus)

### Web

* Pointer Events unify mouse/touch/pen
* Gesture state machine:
  * `down` → hit test → mode (drag / resize / draw / pan)
  * `move` → update in-progress transform (fast)
  * `up` → finalize operation(s)

**Key details:**
* Two-finger gestures: pan/zoom (on touch)
* Pen/stylus: pressure (optional), hover (if supported)

### iOS

* `UIGestureRecognizer` for pan/pinch/rotate
* PencilKit? (mention as optional for freehand)
* **Distinguish:**
  * One-finger = draw/drag
  * Two-finger = camera pan/zoom

### Android

* `ScaleGestureDetector` + custom pointer tracking
* Support multi-pointer transitions cleanly (hard edge case)

### Transform Math (Interviewer Probe)

* Keep all geometry in "world coordinates"
* Camera maps world → screen:
  * `screen = (world - cameraOrigin) * zoom`
* Hit-test in world coords by inverting transform

---

## 23. Op Generation + Batching (Collaboration at Scale)

### Interaction Model: Transient vs Committed

**During drag:**
* Local state updates at 60fps (smooth)
* Network ops:
  * Send at 10–20Hz
  * Final "commit" on pointer up

### Coalescing Rule

For the same object:
* Keep only the latest `move_to(x,y)` during a batch window
* Keep only the latest `resize_to(w,h)`
* For strokes, chunk points (e.g., 200 points per op)

### Why This Matters

* Reduces bandwidth
* Reduces server writes
* Reduces remote clients' work
* Improves battery on mobile

### Soundbite

> "I treat pointer-move as local transient state and emit semantic ops at a throttled cadence with a final commit."

---

## 24. Sync Engine: Overlay Model + Rebase

### Best Practice: Base + Overlay

**Maintain:**
* **Base state:** server-committed objects
* **Overlay:** apply pending ops on top (local intent)

**Render uses `base ⊕ overlay`**

**When remote ops arrive:**
1. Apply to base
2. Rebase overlay (replay pending ops)
3. Continue

This prevents "jitter" where local objects jump around when remote updates come in.

### Reconnect & Catch-Up

* Track `lastSeenRevision`
* If WS drops, keep editing (pending queue)
* On reconnect:
  * `fetch /ops?after=lastSeenRevision`
  * Apply → rebase → resend

### Idempotency

* Each op has `op_id`
* If retry occurs, server dedupes
* Client can safely resend inflight ops after reconnect

---

## 25. Persistence + Offline-First (Reliable Mobile Story)

### Web: IndexedDB

Store per drawing:
* Snapshot blob + `snapshotRevision`
* Pending ops queue
* Optionally recent ops segments for fast reopen

### iOS/Android: SQLite

**Tables:**
* `snapshots(drawing_id, rev, blob)`
* `pending_ops(drawing_id, op_id, base_rev, payload, created_at)`
* `assets(asset_id, local_path, remote_url, status)`

### Offline Behavior

* User continues editing
* Ops queue grows
* On reconnect: catch-up + rebase + resend
* Images: upload later; placeholder until committed

### Mobile-Specific Nuance

* Backgrounding closes WS; must resume from persisted `lastSeenRevision`
* Handle partial uploads safely (resume/retry)

---

## 26. Testing & Debugging (How You Prove It Works)

Interviewers LOVE this.

### Deterministic Core Tests

* Apply-op unit tests
* Replay snapshot + ops yields same result
* Rebase tests: remote ops + pending ops converge

### Sync Simulation

* Drop/reorder/duplicate message tests
* Offline for N minutes then reconnect

### Debug Tools

**"Sync inspector" showing:**
* Current revision
* Pending/inflight counts
* Last ack time
* Export diagnostic bundle (snapshot + last 500 ops)

---

## 27. Implementation Soundbite

> "I build a deterministic core (document model + op engine + sync engine) shared conceptually across web and native. React handles UI chrome; a Canvas renderer reads doc state at 60fps. Native apps use platform canvases with gesture abstraction mapping inputs to semantic commands. All edits produce ops, stored durably in a pending queue for offline. Sync is WS for push plus REST for commit/catch-up, with rebase to preserve local intent."
