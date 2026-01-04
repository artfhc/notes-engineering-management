# Design a Collaborative Drawing App (tldraw-like)

> **Context**: Engineering Manager system design interview
>
> **Focus**: Frontend & Mobile credibility, with solid Backend & Realtime fundamentals

---

## 1. Problem Statement

Design a collaborative drawing application similar to **tldraw**, supporting web and mobile clients, realtime collaboration, offline mode, and version history at scale.

---

## 2. Scope & Assumptions

### Platforms

* Web
* Mobile Web
* iOS
* Android

### Scale Assumptions

* ~10M Daily Active Users (DAU)
* Mostly read-heavy, few active collaborative sessions at a time
* Hot drawings << total drawings

---

## 3. Functional Requirements

### Core

* User authentication (login / logout)
* Create, edit, delete drawings
* Draw **shapes, text, images** on a canvas
* Persist drawings across sessions & devices
* Version history (undo, redo, restore)
* Share drawings with other users

  * Read access
  * Edit access

### Collaboration

* Realtime sync across clients (<1s latency)
* Optimistic local updates

### Offline

* Fully usable offline
* Sync when connection is restored

---

## 4. Non-Functional Requirements

* **Latency**: Near-realtime sync (<1s)
* **Consistency**: Eventual consistency, server-ordered ops
* **Availability**: Drawing should work even if realtime service is down
* **Security**: Access control on every read/write
* **Scalability**: Horizontal scaling of realtime connections
* **Durability**: No data loss
* **Abuse Protection**: Rate limiting, DDoS protection

---

## 5. Data Model

### Drawing

```text
Drawing
- drawing_id
- owner_id
- title
- created_at
- updated_at
- latest_revision
- acl: [{ user_id, role: read | edit }]
```

### Objects (Logical Model)

#### Text

```text
- object_id
- type: text
- text: char_64
- font
- font_size
- coordinates: { x, y, width, height }
```

#### Image

```text
- object_id
- type: image
- url
- coordinates: { x, y, width, height }
```

#### Shape

```text
- object_id
- type: shape
- shape_type: circle | rectangle
- coordinates: { x, y, width, height, radius }
```

### Operation Log (Core Sync Primitive)

```text
Operation
- revision_id (monotonic, server-assigned)
- drawing_id
- client_id
- base_revision
- ops: [ add | update | delete ]
- timestamp
```

---

## 6. Versioning Strategy

### Chosen Approach: Snapshot + Operation Log

* Periodic **snapshots** of drawing state
* Append-only **operation log** between snapshots

**Why**:

* Efficient sync
* Enables history & replay
* Supports offline edits

---

## 7. API Design

### Authentication

```
POST /login
POST /login/oauth
POST /logout
```

### Drawings

```
POST /drawings
GET  /drawings/{drawing_id}
GET  /drawings/{drawing_id}/snapshot?rev=R
GET  /drawings/{drawing_id}/ops?after=R
POST /drawings/{drawing_id}/ops
```

### Objects (Optional)

```
PUT|DELETE /drawings/text/{id}
PUT|DELETE /drawings/image/{id}
PUT|DELETE /drawings/shape/{id}
```

### Sharing

```
POST   /drawings/{drawing_id}/sharing
DELETE /drawings/{drawing_id}/sharing/{user_id}
```

---

## 8. Operation Log (Core Sync Mechanism)

The **operation log (op log)** is the backbone of realtime collaboration, offline support, and version history. Instead of persisting the full canvas on every change, the system records a sequence of small, ordered operations that mutate the drawing state over time.

### Why an Operation Log?

* Efficient network usage (send ops, not full state)
* Realtime collaboration across clients
* Offline edits with later replay
* Version history, auditability, and recovery

---

### Operation Schema

Each operation represents an intent to mutate part of the drawing.

```json
{
  "op_id": "client42:891",
  "drawing_id": "d1",
  "client_id": "client42",
  "base_revision": 120,
  "type": "update",
  "object_id": "shape7",
  "patch": { "x": 120, "y": 80 },
  "timestamp": 1700000000
}
```

```json
{
  "drawing_id": "d1",
  "client_id": "c42",
  "base_revision": 120,
  "ops": [
    {"op_id":"c42:891", "type":"update", "object_id":"s7", "patch":{"x":12,"y":40}},
    {"op_id":"c42:892", "type":"update", "object_id":"s7", "patch":{"x":13,"y":41}}
  ]
}
```


**Key Fields**:

* `op_id`: Client-generated unique id for idempotency
* `base_revision`: Revision the client believes is current
* `patch`: Minimal delta to apply (property-level)

---

### Server Responsibilities

1. **Validation**

   * Authorization (edit access)
   * Schema validation
   * Rate limiting and payload size checks

2. **Assign Total Order**

   * Server assigns a **monotonic revision number** per drawing
   * This establishes a single canonical order of operations

3. **Durable Persistence**

   * Append operation to the op log (append-only)
   * Update drawing metadata (`latest_revision`)

4. **Broadcast**

   * Publish accepted ops to all WebSocket subscribers of `drawing:{id}`

---

### Client Responsibilities

#### On Open

1. Fetch drawing metadata (latest revision)
2. Fetch nearest snapshot
3. Fetch ops after snapshot revision
4. Reconstruct state
5. Connect to WebSocket channel

#### On Edit (Optimistic Updates)

1. Apply change locally immediately
2. Append op to local pending queue
3. Batch and send ops to server

#### On Acknowledgement

* Server returns assigned revision
* Client marks ops as committed
* Advances local revision pointer

---

### Concurrency Handling

#### Base Revision Mismatch

When multiple clients edit concurrently, a client may submit ops based on a stale revision.

**Option A (MVP – Simpler)**

* Server still accepts ops
* Conflicts resolved via **last-write-wins per object property**

**Option B (More Correct)**

* Server rejects with `409 Conflict`
* Client fetches missing ops, rebases local ops, retries

---

### Rebase Strategy

* Client applies missing remote ops
* Replays local pending ops on top
* Deterministic conflict resolution (revision wins)

**Text Editing Note**:

* MVP: treat text box updates as atomic
* Future: CRDT or OT for rich collaborative text

---

### Snapshots & Compaction

To prevent unbounded log growth:

* Generate a snapshot every N ops or T seconds
* Store only recent ops for replay
* Archive or compact older ops

**Storage Options**:

* Postgres for recent ops
* Object storage (S3/R2) for archived segments

---

### Idempotency & Exactly-Once Semantics

* Each op has a unique `op_id`
* Server enforces uniqueness (dedupe)
* Retries do not cause duplicate mutations

---

### Undo / Redo

* Undo is **user-intent based**, not global rewind
* Implemented by sending compensating ops
* Restore version = load snapshot at revision R and create a new revision

---

### Failure Modes

* WebSocket disconnect → fallback to polling ops
* Missed messages → detect revision gap and catch up
* Server crash → persist ops before broadcast
* Network partition → offline ops queued and rebased on reconnect

---

## 9. Realtime Architecture

### Transport

* WebSocket (primary)
* SSE as fallback

### Flow

1. Client opens drawing
2. Fetch snapshot + ops
3. Connect to WebSocket room: `drawing:{id}`
4. Client emits ops optimistically
5. Server:

   * Validates
   * Assigns revision
   * Persists
   * Broadcasts to subscribers

### Conflict Resolution (MVP)

* Server-ordered operations
* Last-write-wins per object
* Text treated as atomic updates

---

## 9. Client Architecture (Frontend / Mobile)

### State Layers

* UI State (selection, tool, zoom)
* Canvas State (objects)
* Sync State (revision, pending ops)

### Offline Support

* Local persistent store (IndexedDB / SQLite / Room)
* Pending ops queue
* Rebase local ops after server sync

---

## 10. Backend Architecture

### Services

* **API Gateway**
* **Auth Service**
* **Drawing Service**
* **Realtime (WebSocket) Service**
* **Logging / Audit Service**

### Storage

* Metadata DB: PostgreSQL
* Cache: Redis
* Blob Storage: S3 / R2 (snapshots, images)

---

## 11. Scalability Considerations

* Shard WebSocket connections by `drawing_id`
* Hot drawings cached in Redis
* Batch ops during drag interactions
* CDN for static assets & images

---

## 12. Security & Permissions

* JWT or session-based auth
* ACL enforced on every API
* Signed URLs for image upload/download
* Rate limiting per user / drawing

---

## 13. Observability & Ops

* Structured logs per revision
* Metrics:

  * Active drawings
  * Ops/sec
  * WebSocket connections
* Alerts on sync lag or dropped ops

---

## 14. Trade-offs & Future Enhancements

### Deferred

* CRDT for rich text
* Cursor presence & avatars
* Fine-grained object locking
* Infinite canvas spatial indexing (quadtrees)

### Explicit Design Choices

* Server-ordered ops over CRDT (simplicity)
* Snapshot cadence over full replay

---

## 15. Summary

This design uses an **operation-based realtime architecture** with server-ordered revisions, enabling:

* Realtime collaboration
* Offline support
* Version history
* Scalable backend

It balances frontend responsiveness with backend correctness and operational simplicity.
