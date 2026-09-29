# Typing Indicator — Slack / WhatsApp Style

## 1. Core Concept

Typing indicator is **ephemeral — temporary and short-lived — real-time state**, not a persistent message.

```text
User types
   ↓
WebSocket
   ↓
Connection Node
   ↓
Typing / Presence Service
   ↓
Redis (short-lived state + TTL)
   ↓
Directly route event to other participants' Connection Nodes
   ↓
WebSocket
   ↓
UI: "Alice is typing…"
```

- No Message DB write.
- Usually no Kafka required.
- State is temporary and best-effort.
- Redis stores typing state with a short TTL, e.g. **3–5 sec**.

---

## 2. Typing Event Lifecycle

Don't send an event for every keystroke.

```text
0s       3s       6s       9s              Send
│        │        │        │                 │
START    TYPING   TYPING   TYPING            STOP
```

Typical approach:

1. First keystroke → `TYPING_START`
2. While continuously typing → heartbeat every **~2–3 sec**
3. Each heartbeat refreshes the Redis TTL
4. User presses **Send** → `TYPING_STOP`
5. If `TYPING_STOP` is lost → TTL automatically removes the temporary typing state after ~3–5 sec

Example:

```json
{
  "type": "TYPING_START",
  "conversation_id": "conv_101"
}
```

---

## 3. 1:1 Chat + Redis Responsibility

For Alice ↔ Bob:

```text
Alice
  │ WebSocket
  ▼
Connection Node A
  │
  ▼
Typing Service
  │
  ├──→ Redis
  │      └── Alice → TTL 5s
  │
  └──→ Bob's Connection Node B
             │
             ▼
          WebSocket
             │
             ▼
            Bob
```

**Important: Connection Service does NOT consume typing events from Redis.**

Redis is a **state store**, not the event-delivery mechanism.

Flow:

1. Typing Service receives `TYPING_START`.
2. Stores/refreshes Alice's typing state in Redis.
3. Looks up Bob's active Connection Node.
4. Directly forwards the typing event to Node B.
5. Node B pushes it through Bob's WebSocket.
6. Redis TTL cleans up the short-lived state if no further heartbeat arrives.

Redis conceptually maintains:

```text
typing:conv_101
└── Alice → expires in 5 sec
```

Bob sees:

> **Alice is typing…**

---

## 4. Group Chat

For:

```text
Alice + Bob + Nick
```

If Alice and Bob are typing simultaneously:

```text
typing:group_101
├── Alice → TTL 5s
└── Bob   → TTL 5s
```

Each typing event is routed/fanned out to the **other active participants**:

```text
                 ┌──→ Bob
Alice ──────────→│
                 └──→ Nick

                 ┌──→ Alice
Bob ────────────→│
                 └──→ Nick
```

Nick's UI can show:

```text
1 person → "Alice is typing…"

2 people → "Alice and Bob are typing…"

3+       → "Alice, Bob and 3 others are typing…"
```

The UI aggregates the currently active typing users.

Typing state remains **per user + conversation**, rather than a single:

```text
conversation_is_typing = true
```

---

## 5. Responsibilities & Interview Summary

| Component | Responsibility |
|---|---|
| **Client** | Sends `TYPING_START`, heartbeat, `TYPING_STOP` |
| **WebSocket / Connection Node** | Receives and pushes real-time events |
| **Typing/Presence Service** | Manages temporary, short-lived typing state and routing |
| **Redis** | Stores typing state + TTL; **does not deliver events** |
| **Message DB** | Not involved |
| **Kafka** | Usually not required for simple typing events |
| **UI** | Displays/aggregates currently typing users |

### Mental Model

> **Typing indicator = lightweight, temporary, short-lived, best-effort real-time state.**

- **1:1:** route the typing event to the other user's Connection Node.
- **Group:** fan it out to other active participants.
- **Redis:** stores the current typing state and handles TTL-based expiry.
- **Typing Service:** performs the actual event routing.
- **Connection Node:** owns the WebSocket and sends the event to the client.

### Key Distinction

```text
Redis = "Who is currently typing?"
Typing Service = "Where should this typing event go?"
Connection Node = "Send it through the WebSocket."
```
