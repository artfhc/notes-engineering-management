# Design Ride Matching / Dispatch

This could also be framed as: Uber matching, DoorDash delivery assignment, driver-rider dispatch, or courier matching.

## What the Interviewer Is Testing

They want to see whether you understand:

- Real-time location systems
- Matching algorithms
- Geo indexing
- State machines
- Marketplace tradeoffs

## Clarifying Questions

Ask:

- Are we designing for rides or food delivery?
- Single city or global?
- Real-time driver location updates in scope?
- Need ETA estimation?
- Does dispatch optimize for nearest driver only, or also supply balancing?

**A good assumption:** I'll focus on real-time ride matching in one region, where riders request rides and nearby drivers are matched based on proximity, ETA, and availability.

## Functional Requirements

- Riders request rides
- Drivers publish live location and availability
- System finds suitable nearby drivers
- Driver can accept/reject request
- Rider sees trip status updates

## Non-Functional Requirements

- Very low latency
- High availability
- High write throughput for location updates
- Eventually consistent maps are acceptable
- Matching should avoid duplicate assignment

## Core Entities

- **Driver**
- **Rider**
- **Trip**
- **LocationUpdate**
- **DriverStatus** (available, busy, offline)
- **MatchCandidate**

## High-Level Architecture

```
Rider App / Driver App
   ↓
API Gateway
   ↓
Trip Service
   ↓
Dispatch / Matching Service
   ↓
Geo Index / Nearby Search
   ↓
Driver State Store
   ↓
Notification Service
```

**Supporting systems:**

- Location ingestion pipeline
- ETA service / routing service
- Event stream for trip state updates

## Core Flow

### Ride Request Flow

1. Rider submits pickup/dropoff
2. Trip service creates trip request
3. Dispatch service queries nearby available drivers
4. Rank candidates by ETA / distance / driver quality / marketplace rules
5. Offer trip to one or more drivers
6. First accepted driver gets assignment
7. Trip state changes to `matched`
8. Notify rider and driver

## Hard Parts

### Hard Part #1: Nearby Driver Search

Do not say "scan all drivers."

Use:
- Geohash
- S2 cells
- Quadtrees / spatial index

**Good answer:** Driver locations can be bucketed into geo cells so we can quickly find nearby drivers without scanning the whole fleet.

### Hard Part #2: Prevent Double Assignment

Critical point — two drivers must not both get the same trip.

**Solutions:**
- Trip state machine
- Optimistic concurrency / compare-and-swap
- Distributed locking only if needed

> **Strong answer:** I'd model trip assignment as an atomic state transition from `REQUESTED` → `ASSIGNED`, guarded by transactional update or compare-and-swap semantics.

### Hard Part #3: Frequent Location Updates

Drivers constantly send updates.

**Design choice:**
- High-throughput, ephemeral location store
- Only latest location matters for dispatch
- Archive selected events asynchronously for analytics

> This shows maturity.

## APIs

**Request Ride**
```
POST /trips
```

**Driver Location Update**
```
POST /drivers/{id}/location
```

**Get Trip Status**
```
GET /trips/{id}
```

**Driver Accept**
```
POST /trips/{id}/accept
```

## Matching Logic

Candidate score can include:

- ETA to pickup
- Distance
- Driver state
- Driver quality / acceptance rate
- Marketplace balancing

**You can mention:** Initially I'd optimize for nearby ETA and reliability, then later extend with marketplace efficiency and fairness constraints.

## Scaling / Tradeoffs

### Challenges
- Heavy location write traffic
- Hot downtown regions
- Duplicate offers / stale driver state
- Dispatch latency

### Solutions
- Partition by geo region
- In-memory or fast key-value store for active driver state
- Periodic heartbeats / TTL for driver availability
- Async analytics pipeline separate from dispatch path

### Failure Modes
- Stale location data
- Driver accepts but network drops
- Duplicate offer race
- Notification delay

### Fallbacks
- Expire old driver state with TTL
- Re-offer if acceptance not confirmed
- Periodic rider status refresh
- Region failover if possible

## 30-Second Summary

I'd design ride matching around a fast geo-indexed dispatch service. Drivers continuously update location into a low-latency state store, and dispatch queries nearby available drivers, ranks them, and atomically assigns the trip using a protected state transition to avoid double booking.

```
              +-------------------+
              | Rider App         |
              | Driver App        |
              +-------------------+
                        |
                        v
              +-------------------+
              |   API Gateway     |
              +-------------------+
                        |
                        v
              +-------------------+
              |   Trip Service    |
              +-------------------+
                        |
                        v
              +-------------------+
              | Dispatch Service  |
              +-------------------+
                 /            \
                v              v
     +----------------+   +------------------+
     | Geo Index      |   | Driver State DB  |
     | (geohash/S2)   |   | location/status  |
     +----------------+   +------------------+
                |
                v
       +-------------------+
       | Notification      |
       | Service           |
       +-------------------+
                |
                v
      +---------------------+
      | Driver accepts trip |
      +---------------------+
```

**Driver Location Update**

```
Driver App
   |
   v
Location Service
   |
   v
Geo Index + Driver State Store
```

**Key Talking Points**

- Geo-index for nearby search
- Atomic trip assignment
- Frequent location updates
- Marketplace optimization