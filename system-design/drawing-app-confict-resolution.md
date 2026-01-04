# Rebase Strategy for Operation-Based Sync

> **Purpose**: Explain how rebase works in a collaborative drawing app (tldraw-like)
>
> **Audience**: Engineering Manager system design interviews
>
> **Context**: Operation-based realtime + offline synchronization

---

## 1. What Is Rebase?

In an operation-based collaborative system, **rebase** is the process of reconciling local edits that were created on top of an outdated document state with newer changes that arrived from the server or other clients.

In simple terms:

> *Apply missing remote operations first, then replay local intent-based operations on top.*

Rebase is a **client-side responsibility**. The server only enforces ordering and durability.

---

## 2. Why Rebase Is Necessary

Concurrent edits are the default case in collaborative systems.

Example scenario:

* Revision 10 is the latest state on the server.
* Client A and Client B both start editing at revision 10.
* Client A sends an update → server assigns revision 11.
* Client B, unaware of revision 11, sends edits based on revision 10.

Without rebase, one client’s intent would overwrite the other’s changes or cause divergence.

---

## 3. Detecting the Need for Rebase

Each client operation batch includes:

```text
base_revision = R
```

The server compares this with:

```text
latest_revision = L
```

If `R < L`, the client is behind and must rebase.

---

## 4. Rebase Flow (Step-by-Step)

### Step 1: Fetch Missing Remote Operations

The client requests all operations after its base revision:

```text
GET /drawings/{id}/ops?after=R
```

---

### Step 2: Apply Remote Operations

The client applies remote operations in **server revision order**, updating its local state to match the authoritative server state.

---

### Step 3: Replay Local Pending Operations

The client maintains a local queue of **pending (unacknowledged) operations**.

After applying remote ops, the client replays its own operations on top of the updated state.

This preserves the user’s intent while maintaining convergence.

---

### Step 4: Resend Rebased Operations

The client sends the rebased operations with an updated base revision:

```text
base_revision = latest_server_revision
```

The server can now safely assign new revisions.

---

## 5. Why Operations Must Be Intent-Based

Rebase only works if operations represent **intent**, not raw diffs.

### ❌ Diff-Based Operation (Bad)

```json
{ "dx": 50, "dy": 50 }
```

This compounds incorrectly if the object already moved.

### ✅ Intent-Based Operation (Good)

```json
{ "type": "move", "to": { "x": 200, "y": 200 } }
```

Intent-based ops are:

* Deterministic
* Idempotent
* Safe to replay

---

## 6. Rebase Granularity

Rebase happens **per object**, not per canvas:

* Ops affecting different object IDs never conflict
* Only overlapping object mutations need reconciliation

This keeps rebasing efficient and predictable.

---

## 7. Conflict Resolution Rules

Typical MVP rules:

* **Last-write-wins per object property**, based on server revision
* Non-overlapping property updates are merged

These rules are deterministic and easy to reason about in interviews.

---

## 8. Text Editing Considerations

Text editing is more complex than shapes or images.

### MVP Approach

* Treat text box updates as atomic operations
* Optionally soft-lock a text box while editing

### Advanced (Deferred)

* CRDT-based text (Yjs, Automerge)
* OT-based collaborative text engine

Explicitly deferring CRDT is a reasonable trade-off in interviews.

---

## 9. Offline Rebase (Mobile-First)

On mobile devices:

* Pending ops are persisted locally (SQLite / Room)
* Client may be offline for minutes or hours

On reconnect:

1. Fetch remote ops
2. Apply them
3. Replay local ops
4. Resend

This enables true offline-first behavior.

---

## 10. Failure Safety & Guarantees

A correct rebase implementation must be:

* **Deterministic**: Same inputs → same output
* **Idempotent**: Replay-safe
* **Bounded**: Limited ops replay via snapshots

If rebase fails, the client can:

* Reset to latest snapshot
* Reapply user intent

---

## 11. One-Sentence Interview Summary

> **Rebase means applying missing server-ordered operations first, then replaying local intent-based operations on top, so concurrent edits converge without losing user intent.**

---

## 12. Why This Works Well for Drawing Apps

* Shapes and images are naturally intent-based
* Conflicts are rare and visually obvious
* Users expect last-write behavior
* Offline support is critical on mobile

This makes rebase a practical and scalable strategy for collaborative drawing systems.
