# Drawing App Snapshots & Operation Logs

This document provides interview-friendly timeline diagrams showing how snapshots and operation logs (ops) work together over time.

---

## 1. Basic Timeline: Snapshot + Ops

```text
Time  ─────────────────────────────────────────────────────────────▶

Revision:   100        101      102      103      104      105

            ┌──────────────┐
Snapshot S1 │ Full Canvas   │
(rev=100)   │ State @100    │
            └──────────────┘
                 │
                 ▼
               Op101 ──► move shape A
                 │
               Op102 ──► add text B
                 │
               Op103 ──► resize shape A
                 │
               Op104 ──► delete image C
                 │
               Op105 ──► move text B
```

### Reconstruction Rule (Core Idea)

```text
Current State = Snapshot(100) + Ops[101..105]
```

**This is the key sentence interviewers want to hear.**

---

## 2. Snapshot Creation (Compaction)

After enough ops, the server compacts.

```text
Time  ─────────────────────────────────────────────────────────────▶

Revision:   100        101   102   103   104   105        150

Snapshot S1 |──────── ops ────────────────| Snapshot S2
(rev=100)                                 (rev=150)
                                           ┌──────────────┐
                                           │ Full Canvas   │
                                           │ State @150    │
                                           └──────────────┘
```

### What Happens Internally

* Server replays ops 101–150
* Persists Snapshot S2
* Older ops can be archived or deleted

---

## 3. Client Open Flow (Cold Start)

```text
Client Opens Drawing
        │
        ▼
GET /drawing/meta  ──► latest_revision = 187
        │
        ▼
GET /snapshot?rev<=187 ──► Snapshot @150
        │
        ▼
GET /ops?after=150 ──► [151..187]
        │
        ▼
Rebuild Canvas Locally
        │
        ▼
Connect WebSocket (drawing:123)
```

### Visually

```text
Snapshot @150 + Ops[151..187] = Current State
```

---

## 4. Realtime Editing Timeline (Optimistic UI)

```text
Client Timeline (Optimistic)
─────────────────────────────────────────▶

Local Apply Op188 (move shape)
Local Apply Op189 (move shape)
Local Apply Op190 (move shape)
│
│ send batch
▼
Server assigns:
  Op188 → rev 188
  Op189 → rev 189
  Op190 → rev 190

Server Timeline
─────────────────────────────────────────▶

rev 187 → rev 188 → rev 189 → rev 190
```

**Clients converge because server order is canonical.**

---

## 5. Offline → Reconnect Timeline (Very Important)

```text
Time  ─────────────────────────────────────────────────────────────▶

Server:    rev 200   201   202   203   204

Client A:  ── offline ──┐
                         │ local ops
                         │
                         ▼
Local Ops:              OpA1   OpA2   OpA3
(base_revision=200)

On reconnect:
1. Client fetches missing ops [201..204]
2. Applies them locally
3. Rebases local ops (A1, A2, A3)
4. Resends rebased ops
```

### Final Timeline

```text
rev 205 → A1'
rev 206 → A2'
rev 207 → A3'
```

---

## 6. Undo as Timeline (No History Deletion)

```text
Time  ─────────────────────────────────────────────────────────────▶

rev 300: Op300 (move shape to x=100)
rev 301: Op301 (move shape to x=200)
rev 302: Op302 (UNDO of Op301)
```

### UNDO is Just Another Op

```text
Op302 = update shape { x: 100 }
```

**Benefits:**
* ✔ History remains intact
* ✔ Replays correctly
* ✔ Works in collaboration

---

## 7. One Diagram You Can Re-Draw in an Interview

If you need one diagram to memorize, use this:

```text
[ Snapshot @R ]
      +
[ Ops R+1 .. N ]
      =
[ Current Canvas ]
```

### Or Spoken Aloud

> "We periodically snapshot the canvas and then replay a short, ordered op log on top to reconstruct state."

---

## 8. Why Interviewers Like This Design

This timeline proves you understand:

* **Realtime collaboration**
* **Offline support**
* **Deterministic ordering**
* **Storage efficiency**
* **Failure recovery**
