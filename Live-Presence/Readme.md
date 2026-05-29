# Live Presence System Design — Consolidated Summary
### Date: 29th May, 2026

A **Live Presence System** tracks and broadcasts real-time user activity states like ONLINE, OFFLINE, BUSY, AWAY, DND, or ON_CALL with very low latency and high scalability.

Common examples:

* WhatsApp → Online / Last Seen
* Slack → Active / Away / DND
* Uber → Driver availability

---

# 1. Core Components

| Component           | Responsibility                         |
| ------------------- | -------------------------------------- |
| Load Balancer       | Distribute socket traffic              |
| WebSocket Gateway   | Maintain persistent client connections |
| Presence Service    | Manage presence state transitions      |
| Redis               | Ultra-fast live state store with TTL   |
| Kafka               | Event streaming & decoupled processing |
| Aggregator Consumer | Compute duration analytics             |
| OLAP DB             | Store historical aggregates            |

---

# 2. High-Level Architecture

```text
Client
   ↓
WebSocket Gateway
   ↓
Presence Service
   ├── Redis (live state)
   └── Kafka (state transition events)
            ↓
     Analytics Consumers
            ↓
      ClickHouse / OLAP
```

---

# 3. Presence vs Availability State

| State Type          | Meaning                                |
| ------------------- | -------------------------------------- |
| Connection Presence | ONLINE / OFFLINE based on connectivity |
| Availability Status | BUSY / AWAY / DND manually set by user |

Both are stored independently.

Example:

```json
{
  "connection_state": "ONLINE",
  "availability_state": "BUSY",
  "effective_status": "BUSY"
}
```

---

# 4. Effective Status Resolution

Priority logic:

```text
If availability_status exists:
    effective_status = availability_status
Else:
    effective_status = connection_state
```

Priority:

```text
DND > BUSY > AWAY > ONLINE > OFFLINE
```

---

# 5. Workflow — User Comes Online

```text
User opens app
   ↓
WebSocket connection established
   ↓
Presence Service marks ONLINE
   ↓
Redis updated with TTL
   ↓
Kafka publishes state transition event
   ↓
Subscribers receive update
```

Redis Example:

```bash
SETEX presence:U123 30 ONLINE
```

---

# 6. Workflow — User Sets BUSY/DND/AWAY

```text
User manually sets BUSY
   ↓
Availability state stored separately
   ↓
Effective state recalculated
   ↓
Kafka transition event published
   ↓
Subscribers updated
```

Redis Example:

```bash
HSET profile:U123 availability BUSY
```

---

# 7. Workflow — Heartbeat Processing

```text
Client sends heartbeat every 20–30 sec
   ↓
Redis TTL refreshed
```

Redis Example:

```bash
EXPIRE presence:U123 30
```

Important:

* Heartbeats maintain connectivity only
* They are NOT directly used for analytics

---

# 8. Workflow — OFFLINE Detection

```text
User disconnects OR network drops
   ↓
Heartbeat stops
   ↓
Redis TTL expires automatically
   ↓
Presence Service detects expiration
   ↓
ONLINE → OFFLINE transition generated
   ↓
Kafka publishes OFFLINE event
   ↓
Subscribers notified
```

Redis Example:

```bash
SETEX presence:U123 30 ONLINE
```

If no heartbeat refresh occurs within 30 sec:

* key expires automatically
* user considered OFFLINE

Why TTL approach is powerful:

* No manual cleanup required
* Handles abrupt disconnects
* Works even during app crash/network loss

---

# 9. Workflow — Presence Event Broadcasting

```text
Incoming event
   ↓
Resolve effective status
   ↓
Compare with previous state
   ↓
If changed:
    publish Kafka transition event
```

Only meaningful transitions are emitted:

```text
ONLINE → BUSY
BUSY → ON_CALL
ON_CALL → AVAILABLE
AVAILABLE → OFFLINE
```

---

# 10. Workflow — Duration Aggregation

Goal:
Compute daily duration spent in:

* AVAILABLE
* BUSY
* DND
* ON_CALL

Flow:

```text
Kafka transition event
      ↓
Aggregator Consumer
      ↓
Compute:
current_timestamp - previous_timestamp
      ↓
Update aggregate counters
      ↓
Store in ClickHouse
```

Example:

```json
{
  "user_id": "U123",
  "busy_duration_sec": 1500
}
```

---

# 11. Why Separate Transition Events from Heartbeats

| Heartbeats          | Transition Events  |
| ------------------- | ------------------ |
| Very high frequency | Low frequency      |
| Connectivity only   | Business meaning   |
| Noisy               | Analytics-friendly |
| Expensive at scale  | Efficient          |

Example:

* Heartbeat every 20 sec → billions/day
* Status changes → few/day/user

---

# 12. Recommended Kafka Topics

| Topic                      | Purpose                          |
| -------------------------- | -------------------------------- |
| presence-heartbeats        | Internal connectivity monitoring |
| presence-state-transitions | Business status changes          |
| presence-analytics-events  | Aggregation pipeline             |

---

# 13. Major Design Challenges & Solutions

| Challenge                      | Solution                            |
| ------------------------------ | ----------------------------------- |
| Millions of socket connections | Distributed WebSocket gateways      |
| False offline detection        | Heartbeat + Redis TTL expiry        |
| Excessive Kafka traffic        | Emit only transition events         |
| User intent override           | Separate availability state         |
| Multi-device conflicts         | Aggregate device presence           |
| Out-of-order events            | Kafka partition by user_id          |
| Duplicate events               | Ignore identical consecutive states |
| Midnight open sessions         | Background session closer           |
| Redis overload                 | Lightweight TTL-based presence      |
| Analytics scalability          | OLAP DB like ClickHouse             |

---

# 14. Key Trade-Offs

| Decision                    | Benefit                    | Trade-Off                      |
| --------------------------- | -------------------------- | ------------------------------ |
| Redis for live state        | Ultra-low latency          | Volatile storage               |
| Kafka event-driven model    | Replayability & decoupling | Operational complexity         |
| Transition events only      | Lower infra cost           | Requires state comparison      |
| Separate availability state | Preserves user intent      | Additional coordination        |
| WebSockets                  | Real-time updates          | Persistent connection overhead |

---

# 15. Interview-Friendly Conclusion

> A Live Presence System uses WebSockets, Redis, and Kafka to maintain and broadcast real-time user states at scale. Connectivity state and user-defined availability are managed independently, while Kafka transition events enable efficient real-time analytics and duration computation without processing noisy heartbeat traffic.
