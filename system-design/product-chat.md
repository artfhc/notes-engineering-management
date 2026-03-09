# Design a Chat System

## What the Interviewer Is Testing

They want to see whether you understand:

- Real-time messaging
- WebSocket / long-lived connections
- Message delivery semantics
- Ordering
- Unread state / sync across devices

## Clarifying Questions

Ask:

- One-to-one only, or group chat too?
- Need online presence / typing indicator?
- Need read receipts?
- Need message history?
- Need cross-device sync?

**A good assumption:** I'll design a chat system that supports one-to-one and small group chat, with persistent history, delivery acknowledgements, and multi-device sync.

## Functional Requirements

- Send/receive messages in real time
- Store chat history
- Support online/offline users
- Sync across multiple devices
- Show delivery/read status

## Non-Functional Requirements

- Low latency
- High availability
- Durable storage
- Ordered delivery per conversation
- Eventual consistency acceptable for read receipts/presence

## Core Entities

- **User**
- **Conversation**
- **Message**
- **Participant**
- **DeliveryReceipt**
- **ReadReceipt**

## High-Level Architecture

```
Client
   ↓
API Gateway / Auth
   ↓
WebSocket Gateway
   ↓
Chat Service
   ↓
Message Queue / Event Bus
   ↓
Message Store
   ↓
Notification Service
   ↓
Push Notification Provider
```

### Core Flow

When sender sends a message:

1. Client sends message through WebSocket
2. Chat service validates auth and conversation membership
3. Message gets assigned server-side ID and timestamp
4. Persist message in durable store
5. Publish event to delivery pipeline
6. If recipient is online, deliver via WebSocket
7. If offline, trigger push notification
8. Update unread/read states asynchronously

## Important Design Choices

### A. WebSocket for Real-Time Delivery

**Why:**
- Low-latency bidirectional communication
- Better than polling for chat

### B. Persistent Storage

Need durable message history.

**Choices:**
- Relational DB for metadata
- Wide-column / NoSQL store for high-scale message history

**Good interview answer:** I'd separate metadata from message history if scale grows large.

## Hard Part: Message Ordering

True global ordering is hard and unnecessary.

**Better answer:**
- Guarantee per-conversation ordering
- Use monotonically increasing sequence numbers per conversation
- Server assigns final ordering key

> That's a very strong answer.

## Delivery Semantics

**Typical design:**
- At least once delivery internally
- Dedupe on client/server using message IDs
- Ack states: `sent` → `delivered` → `read`

**Mention idempotency:** Client-generated temporary IDs plus server-issued canonical IDs help retries without duplicates.

## Multi-Device Sync

If user has phone + laptop:

- All active sessions subscribe to same user channel
- New message events fan out to all sessions
- Unread state tracked at account/conversation level

## APIs / Events

**Send Message**
```
WS: send_message(conversation_id, client_msg_id, payload)
```

**Fetch History**
```
GET /conversations/{id}/messages?cursor=...
```

**Read Receipt**
```
POST /conversations/{id}/read
```

## Scaling / Tradeoffs

### Challenges
- Millions of open WebSocket connections
- Message persistence throughput
- Presence updates can be noisy

### Solutions
- Stateless WebSocket gateways behind load balancer
- External pub/sub or broker to distribute events across nodes
- Shard conversations by `conversation_id`
- Do not over-engineer presence accuracy

> **Strong line:** Presence is typically approximate, while message durability and ordering matter much more.

### Failure Modes
- WebSocket node failure
- Duplicated sends due to retries
- Push notification delay
- Broker lag

### Fallbacks
- Reconnect and backfill from last seen cursor
- Idempotent message IDs
- Fetch missed messages from history store

## 30-Second Summary

I'd use WebSockets for real-time delivery, durable message storage for history, and a broker to fan out events across chat servers. Ordering would be guaranteed per conversation rather than globally, and retries would be handled idempotently using message IDs.

               +--------------------+
               |     Mobile App     |
               +--------------------+
                        |
                        v
               +--------------------+
               |   API Gateway      |
               |   (Auth)           |
               +--------------------+
                        |
                        v
               +--------------------+
               | WebSocket Gateway  |
               +--------------------+
                        |
                        v
               +--------------------+
               |    Chat Service    |
               +--------------------+
                 /        |        \
                v         v         v
     +---------------+ +------------+ +------------------+
     | Message Queue | | Message DB | | Presence Service |
     | (Kafka etc)   | | (history)  | | online/offline   |
     +---------------+ +------------+ +------------------+
                |
                v
        +----------------------+
        | Push Notification    |
        | Service (APNs/FCM)   |
        +----------------------+

Message Flow
Sender → WebSocket → Chat Service
      → Store Message
      → Publish Event
      → Deliver to Recipient
      → Push notification if offline

Key Talking Points

WebSocket persistent connections

Per-conversation ordering

Message queue for fanout

Multi-device sync