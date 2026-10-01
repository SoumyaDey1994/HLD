# Group Chat System Design — WhatsApp / Slack

> **Scope:** Core group messaging flow for a 4-member group — **Alice, Bob, James & Lisa**.

## 1. Core Requirements

Group:

```text
G1
├── Alice
├── Bob
├── James
└── Lisa
```

Core capabilities:

- Real-time message send/receive
- Conversation-level message ordering
- Online/offline delivery
- Delivery & read acknowledgement
- Durable message storage
- Retry and idempotency
- Horizontal scalability
- Reliable asynchronous fan-out

---

## 2. High-Level Architecture

```text
                         ┌─────────────────┐
                         │  Load Balancer  │
                         └────────┬────────┘
                                  │
                         WebSocket Connections
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
       Connection Node A   Connection Node B   Connection Node C
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  ▼
                          ┌───────────────┐
                          │Message Service│
                          └───────┬───────┘
                                  │
                       ┌──────────┴──────────┐
                       ▼                     ▼
                 ┌───────────┐           ┌───────┐
                 │ Message DB│           │ Kafka │
                 └───────────┘           └───┬───┘
                                             │
                                             ▼
                                      ┌───────────────┐
                                      │Delivery Service│
                                      └───────┬───────┘
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                            Bob             James            Lisa
                              │               │                │
                         WebSocket        Push/Sync        WebSocket
```

Supporting components:

```text
Redis
 ├── User → Connection Node mapping
 └── Presence / temporary state

Push Notification Service
 └── Offline user notifications

Object Storage
 └── Images / files / videos
```

---

## 3. WebSocket Connection

Each participant maintains a persistent WebSocket connection.

Example:

```text
Alice → Connection Node A
Bob   → Connection Node B
James → Connection Node C
Lisa  → Connection Node A
```

Redis can maintain the current connection ownership:

```text
user:Alice → Node A
user:Bob   → Node B
user:James → Node C
user:Lisa  → Node A
```

**Connection Node owns the actual WebSocket.**

Responsibilities:

- Receive client messages
- Send messages/events to clients
- Maintain connection lifecycle
- Update connection ownership on reconnect

---

## 4. Alice Sends a Message

Alice sends:

```json
{
  "type": "MESSAGE_SEND",
  "conversation_id": "G1",
  "client_message_id": "c123",
  "content": "Hello everyone!"
}
```

Flow:

```text
Alice
  │ WebSocket
  ▼
Connection Node A
  │
  ▼
Message Service
  │
  ├── Validate
  ├── Generate message_id
  ├── Assign sequence_no
  ├── Persist
  │
  └── Publish MessageCreated
             │
             ▼
           Kafka
```

The message is persisted before it is considered successfully accepted.

The Kafka event can contain the payload required for delivery:

```json
{
  "message_id": "m123",
  "conversation_id": "G1",
  "sender_id": "Alice",
  "sequence_no": 1001,
  "content": "Hello everyone!"
}
```

Therefore, Delivery Service normally does not need an additional DB read just to deliver the message.

---

## 5. Message Ordering

Ordering is maintained **within a conversation**, not globally.

Example:

```text
G1

1001 → Alice: Hello
1002 → Bob: Hi
1003 → James: Good morning
1004 → Lisa: Hey!
```

Each message gets:

```text
conversation_id = G1
sequence_no     = 1001
```

A typical Kafka strategy:

```text
Kafka partition key = conversation_id
```

This keeps messages for `G1` in the same Kafka partition and preserves Kafka's partition-level ordering.

### Important distinction

```text
Conversation ordering
        ↓
sequence_no

Kafka ordering
        ↓
ordering within a partition
```

Retries should use `message_id` / `client_message_id` for idempotency.

---

## 6. Kafka + Delivery Service

After persistence:

```text
Message Service
      │
      └── MessageCreated
              ↓
            Kafka
              ↓
      Delivery Service
```

Delivery Service obtains the members of `G1`:

```text
G1
├── Alice
├── Bob
├── James
└── Lisa
```

Since Alice is the sender, the message needs to be delivered to:

```text
Bob
James
Lisa
```

For each online user, Delivery Service uses the current connection mapping to find the Connection Node owning that user's WebSocket.

---

## 7. Group Fan-Out

For:

```text
Alice → "Hello everyone!"
```

Fan-out:

```text
Delivery Service
      │
      ├──→ Bob's Connection Node
      │       └──→ WebSocket → Bob
      │
      ├──→ James' Connection Node
      │       └──→ WebSocket → James
      │
      └──→ Lisa's Connection Node
              └──→ WebSocket → Lisa
```

**Delivery Service does not directly manipulate WebSockets.**

It routes the event to the Connection Node that owns each recipient's socket.

Alice can receive a server acknowledgement:

```text
MESSAGE_ACCEPTED
message_id = m123
sequence_no = 1001
```

---

## 8. Online vs Offline + Delivery / Read Status

Suppose:

```text
Alice  → Online
Bob    → Online
James  → Offline
Lisa   → Online
```

Delivery:

```text
Bob   → WebSocket
Lisa  → WebSocket
James → Offline handling
```

The message is already safely stored in Message DB.

For James:

```text
Delivery Service
      │
      └──→ Push Notification Service
                  │
                  ▼
                Device
```

When James reconnects:

```text
James
  │
  ▼
WebSocket
  │
  ▼
Sync(last_seen_sequence)
  │
  ▼
Message DB
  │
  ▼
Missed messages
```

### Delivery / Read Status

Group messages need status **per recipient**.

Example:

```text
Message M1001
Alice → "Hello!"

Bob   → DELIVERED
James → READ
Lisa  → DELIVERED
```

Conceptual schema:

```text
message_recipients
────────────────────────────
message_id
user_id
delivery_status
delivered_at
read_at
```

A single `status` column on `messages` is insufficient because each recipient can have a different delivery/read state.

---

## 9. Reliability, Scalability & Responsibility Matrix

### Reliability

**Idempotency**

```text
client_message_id = c123
```

If Alice retries after a network failure, the server recognizes the duplicate request instead of creating another message.

**Connection failure**

```text
Alice → Connection Node A
          X
          │
          ▼
Alice reconnects
          │
          ▼
Load Balancer → Connection Node D
```

Redis mapping becomes:

```text
Alice → Node D
```

**Kafka failure**

Kafka provides asynchronous buffering between persistence and delivery, allowing Delivery Service to catch up after recovery.

### Scalability

Normal-sized groups:

```text
Persist once
    ↓
Publish once
    ↓
Fan-out to members
```

For very large groups, independent fan-out to every member can become expensive. A hybrid or fan-out-on-read approach can be considered depending on group size and traffic pattern.

### Responsibility Matrix

| Component | Responsibility |
|---|---|
| **Client** | WebSocket communication, retries, acknowledgements |
| **Load Balancer** | Distributes WebSocket connections |
| **Connection Service / Node** | Owns WebSockets and connection lifecycle |
| **Message Service** | Validation, message ID, ordering, persistence |
| **Message DB** | Durable message storage |
| **Kafka** | Async event transport and buffering |
| **Delivery Service** | Group fan-out and recipient routing |
| **Redis** | Connection mapping and short-lived state |
| **Push Service** | Offline notifications |
| **Object Storage** | Images, files, videos |

---

## Interview-Ready Mental Model

```text
Client
  │
  │ WebSocket
  ▼
Connection Node
  │
  ▼
Message Service
  │
  ├────→ Message DB       ← Durable message
  │
  └────→ Kafka            ← Async event
           │
           ▼
    Delivery Service      ← Fan-out
       /    |    \
      ▼     ▼     ▼
    Bob   James   Lisa
      │     │      │
 WebSocket Push  WebSocket
            │
          Sync
       when online
```

> **Core pattern:**  
> **Persist once → publish once → asynchronously fan-out → route to each recipient's Connection Node → deliver through WebSocket → use push/sync for offline users.**
