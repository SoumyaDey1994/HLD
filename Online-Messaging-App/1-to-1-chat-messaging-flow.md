# 1:1 Chat — WebSocket-Based Messaging Flow

## Context: Alice & Bob having a conversation

## 1. High-Level Architecture

```text
Alice
  │
  │ WebSocket
  ▼
Load Balancer
  │
  ▼
Connection Node A
  │
  ▼
Message Service
  │
  ├──────────► Message DB
  │
  └──────────► Kafka
                  │
                  ▼
           Delivery Service
                  │
             Redis Lookup
          Bob → Connection Node B
                  │
             Internal RPC
                  ▼
           Connection Node B
                  │
              WebSocket
                  ▼
                 Bob
```

**Core services:**
- **Connection Service** — manages WebSocket connections.
- **Message Service** — persists messages and publishes events.
- **Delivery Service** — routes messages to recipients.
- **Notification Service** — handles offline push notifications.

---

## 2. WebSocket Connection — Connection Service

Users maintain persistent WebSocket connections through the **Load Balancer**.

```text
Alice → LB → Connection Node A
Bob   → LB → Connection Node B
```

Connection Service maintains:

```text
Redis:
Alice → Node A
Bob   → Node B
```

This mapping tells us **which Connection Node currently owns a user's WebSocket**.

---

## 3. Alice Sends Message

Alice sends:

```text
"Hi Bob!"
```

over her existing WebSocket:

```text
Alice → Node A → Message Service
```

Message Service:

1. Authenticates/validates request.
2. Generates `message_id`.
3. Assigns server-side `sequence_no`.
4. Persists message in DB.
5. Publishes `MessageCreated` event to Kafka.

```text
Message Service
     ├──→ Message DB
     └──→ Kafka
```

**Important:** Message is persisted before asynchronous delivery.

---

## 4. Message DB — Schema & Example

### `messages`

| Column | Type | Constraint / Purpose |
|---|---|---|
| `message_id` | UUID | **PK** — unique message |
| `conversation_id` | UUID | **NOT NULL, INDEX** — identifies 1:1 conversation |
| `sender_id` | UUID | **NOT NULL** — sender |
| `receiver_id` | UUID | **NOT NULL** — recipient |
| `content` | TEXT | Message content |
| `message_type` | VARCHAR(20) | `TEXT`, `IMAGE`, `FILE`, etc. |
| `sequence_no` | BIGINT | **NOT NULL** — ordering within conversation |
| `status` | VARCHAR(20) | `SENT`, `DELIVERED`, `READ` |
| `created_at` | TIMESTAMP | Server-side creation time |
| `delivered_at` | TIMESTAMP NULL | Delivery timestamp |
| `read_at` | TIMESTAMP NULL | Read timestamp |
| `edited_at` | TIMESTAMP NULL | Edit timestamp |
| `deleted_at` | TIMESTAMP NULL | Soft-delete timestamp |

**Important index:**

```text
INDEX (conversation_id, sequence_no)
```

This supports efficient **ordered retrieval, pagination and offline synchronization**.

### Example Record

```text
message_id      = 7f4c2a91-...
conversation_id = 9a21bc44-...
sender_id       = alice_101
receiver_id     = bob_202
content         = "Hi Bob!"
message_type    = TEXT
sequence_no     = 1842
status          = READ
created_at      = 2026-09-20 15:10:01
delivered_at    = 2026-09-20 15:10:02
read_at         = 2026-09-20 15:10:05
edited_at       = NULL
deleted_at      = NULL
```

---

## 5. Kafka + Delivery Service

Message Service publishes:

```text
MessageCreated {
    message_id,
    conversation_id,
    sender_id,
    receiver_id,
    content,
    sequence_no
}
```

Delivery Service consumes this event.

Its responsibility:

> **Determine where the recipient is connected and route the message there.**

---

## 6. Recipient Routing

Delivery Service checks Redis:

```text
Redis:
Bob → Node B
```

It then forwards the message to Node B through **internal RPC/gRPC/HTTP**.

```text
Delivery Service
      │
      │ Internal RPC
      ▼
Connection Node B
```

Delivery Service **doesn't directly own or manipulate Bob's WebSocket**.

Connection Node B performs:

```text
socket.send(message)
```

---

## 7. Online vs Offline Handling

### Bob Online

```text
Delivery Service
      ↓
Redis → Bob = Node B
      ↓
Node B
      ↓
WebSocket
      ↓
Bob
```

Message is delivered immediately.

### Bob Offline

```text
Bob → No active WebSocket
```

Message is already safely persisted in DB.

Delivery Service can trigger:

```text
Notification Service
        ↓
    APNs / FCM (Notification mechanism at client-device)
        ↓
   Bob's device
```

When Bob reconnects:

```text
Bob → Connection Service
       ↓
last_seen_sequence
       ↓
Message DB
       ↓
Missed messages
       ↓
Bob
```

---

## 8. Message Status & Acknowledgement

Typical lifecycle:

```text
SENT → DELIVERED → READ
```

- **SENT** → Message successfully persisted.
- **DELIVERED** → Recipient's Connection Node received it.
- **READ** → Recipient opened/read it.

Corresponding timestamps:

```text
created_at
delivered_at
read_at
```

Status updates can be processed asynchronously.

---

## 9. Scalability & Reliability Nuances

### WebSocket Scaling
- Multiple Connection Nodes behind LB.
- Each node maintains thousands of persistent connections.
- Redis stores `user → node` mapping.

### Kafka
- Decouples persistence from delivery.
- Buffers traffic spikes.
- Partitioning can help maintain ordering.

### Connection Node Failure
- User reconnects through LB.
- New node updates Redis mapping.
- Missed messages are recovered using `last_seen_sequence`.

### Idempotency
- Use `message_id` / client idempotency key to prevent duplicate messages during retries.

---

## 10. Core Responsibility Matrix

| Component | Primary Responsibility |
|---|---|
| **Load Balancer** | Distribute WebSocket connections |
| **Connection Service** | Maintain WebSockets + connection ownership |
| **Message Service** | Validate + persist + publish message |
| **Message DB** | Durable message storage |
| **Kafka** | Async event transport / buffering |
| **Delivery Service** | Route message to recipient's node |
| **Redis** | `user → connection node` lookup |
| **Notification Service** | Offline push notifications |
| **WebSocket** | Real-time message delivery |

---

## 11. Interview-Ready Summary

> **Connection Service manages persistent WebSocket connections and maintains user-to-node mapping. Message Service validates and durably stores messages, then publishes a Kafka event. Delivery Service consumes the event, looks up the recipient's Connection Node in Redis, and forwards the message through internal RPC. The Connection Node finally pushes it over WebSocket. If the recipient is offline, the message remains in DB and Notification Service can send a push notification; missed messages are synchronized from DB when the user reconnects.**
