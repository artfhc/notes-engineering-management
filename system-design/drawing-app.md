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

## 8. Realtime Architecture

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
