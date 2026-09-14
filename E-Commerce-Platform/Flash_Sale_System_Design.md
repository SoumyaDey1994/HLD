# Flash Sale Order Processing — High-Concurrency System Design

## 1. Use Case

**Scenario:**

- 1M concurrent users click **Buy Now**
- Only **100 / limited stock** is available
- System must:
  - Prevent overselling
  - Avoid database connection exhaustion
  - Handle massive traffic spikes
  - Provide reliable order confirmation

---

## 2. Why the Normal Design Fails

### Usual Flow

```text
Client
  ↓
Order Service
  ↓
Inventory DB → Check Stock
  ↓
Create Order
  ↓
Update Stock
```

### Problems During Flash Sale

| Problem | Reason |
|---|---|
| Overselling | Multiple requests read the same available stock |
| DB overload | Millions of concurrent stock checks hit DB |
| Connection exhaustion | DB connection pool gets exhausted |
| High latency | DB becomes the bottleneck |
| Traffic spike | All users are allowed into the order-processing path |

**Key principle:**

> Don't let 1M users compete directly for 100 available item.

---

# 3. Production-Style Architecture

```text
1M Clients
    │
    ▼
┌──────────────────┐
│   API Gateway    │
│  Rate Limiter    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Order Service  │
└────────┬─────────┘
         │
         ▼
┌─────────────────────────┐
│ Redis Stock Counter      │
│ Atomic Check + Decrement │
│      (Lua Script)        │
└────────┬────────────────┘
         │
      Token Acquired
         │
         ▼
┌──────────────────┐
│      Kafka       │
│ OrderRequested   │
└────────┬─────────┘
         │
         ▼
┌─────────────────────────┐
│ Order Processing Worker │
└────────┬────────────────┘
         │
         ├── Create/Update Order
         ├── Payment
         └── Final Status
                  │
                  ▼
             Order DB
```

---

# 4. End-to-End Flow

## Step 1 — Client Sends Buy Request

```text
Client → POST /orders
```

1M users may send the request simultaneously.

The request first reaches the **API Gateway**.

---

## Step 2 — API Gateway / Rate Limiter

### Responsibility

Protect the backend from a massive request burst.

Example:

```text
1M requests
     ↓
Rate Limiter
     ↓
10K requests/sec allowed
     ↓
Remaining requests → rejected / throttled
```

The exact limit depends on system capacity.

### Important

Rate limiting is **not the stock allocation mechanism**.

It only prevents the entire backend from being overwhelmed.

---

# 5. Step 3 — Order Service

Requests allowed through the gateway reach the **Order Service**.

### Responsibilities

- Validate basic request information
- Generate `orderId`
- Ask Redis for stock/token
- Publish an order request to Kafka
- Return an initial `PENDING` response

The Order Service should remain **lightweight**.

It should **not** synchronously perform:

- Payment
- Heavy DB processing
- Inventory DB locking
- Long-running business operations

---

# 6. Step 4 — Redis as High-Speed Stock Gate

Before the sale begins:

```text
Redis:
product:phone123:stock = 100
```

Redis maintains the **real-time sale availability counter**.

Instead of querying the Inventory DB for every request:

```text
1M requests → Redis
```

rather than:

```text
1M requests → Inventory DB
```

---

# 7. Step 5 — Atomic Counter / Lua Script

A simple `GET → CHECK → DECREMENT` is not safe because multiple requests can execute concurrently.

Use an **atomic Redis operation**, commonly a Lua script.

Conceptually:

```text
IF stock > 0
    DECREMENT stock
    RETURN SUCCESS
ELSE
    RETURN FAILURE
```

### Example

Initial:

```text
stock = 100
```

1000 concurrent requests arrive.

Redis atomically processes them:

```text
Request #100 → SUCCESS → stock = 0
Request #101 → FAILURE
Request #102 → FAILURE
...
Request #1000 → FAILURE
```

Only **one request acquires the stock/token**.

### Responsibility

**Redis + atomic Lua logic = concurrency gate**

It decides:

> "Who is allowed to proceed?"

---

# 8. Step 6 — Exhausted Token / Stock

Once:

```text
stock = 0
```

new Buy Now requests should be rejected immediately.

```text
Client
  ↓
Order Service
  ↓
Redis
  ↓
stock = 0
  ↓
"Sold Out"
```

The UI can also disable **Buy Now** after the backend reports that the item is sold out.

### Important

The UI disabling is only a **UX optimization**.

The backend Redis check remains the authoritative admission control.

A malicious/client request cannot bypass it.

---

# 9. Step 7 — Publish to Kafka

If Redis successfully grants the token:

```text
Order Service
      ↓
Kafka
      ↓
OrderRequested
```

The event should contain information such as:

```text
orderId
userId
productId
quantity
request/fare details
timestamp
idempotencyKey
```

### Why Kafka?

Kafka acts as a **buffer** between the extremely fast admission phase and slower order processing.

Instead of making the Order Service wait:

```text
Client → Order Service → Payment → DB → Response
```

we do:

```text
Client
  ↓
Order Service
  ↓
Redis admission
  ↓
Kafka
  ↓
Return PENDING
```

---

# 10. Step 8 — Order Processing Worker

Kafka consumers/processors asynchronously consume:

```text
OrderRequested
```

### Worker responsibilities

1. Create/persist order as `PENDING`
2. Initiate payment
3. Handle payment result
4. Update order status
5. Perform compensation if necessary
6. Persist final order state

Example:

```text
PENDING
   │
   ├── Payment Success ──→ CONFIRMED
   │
   ├── Payment Failure ──→ FAILED
   │                         ↓
   │                     Release token
   │
   └── Timeout ───────────→ EXPIRED
                             ↓
                         Release token
```

---

# 11. Saving Order Details Asynchronously

The initial Buy request should not wait for all DB operations.

Instead:

```text
Client
  ↓
Order Service
  ↓
Redis admission
  ↓
Kafka
  ↓
Worker
  ↓
Order DB
```

The client receives something like:

```json
{
  "orderId": "ORD123",
  "status": "PENDING"
}
```

The actual order persistence and processing happen asynchronously.

### Benefit

The critical Buy path remains extremely lightweight.

---

# 12. Payment Processing

Worker calls:

```text
Order Worker
      ↓
Payment Service
      ↓
External Payment Gateway
```

The Payment Service should preferably use a **webhook/event-based response** from the external payment provider rather than having the worker continuously poll.

```text
Payment Gateway
      ↓ webhook
Payment Service
      ↓
PaymentCompleted / PaymentFailed
      ↓
Kafka
      ↓
Order Worker
```

---

# 13. Successful Payment

```text
Payment Success
      ↓
Order = CONFIRMED
      ↓
Persist final order/payment details
```

The Redis token has already been consumed during admission.

The system should **not decrement Redis again**.

Similarly, avoid blindly decrementing inventory DB a second time.

---

# 14. Failed Payment / Timeout

If payment fails:

```text
Payment Failed
      ↓
Order = FAILED
      ↓
Release reserved token
      ↓
Redis INCR
```

Conceptually:

```text
Redis stock:
0 → 1
```

This allows another user to attempt the purchase.

### Important

The release operation must be **idempotent**.

A retry or duplicate Kafka event must not execute:

```text
INCR
INCR
INCR
```

for the same order.

---

# 15. Payment Already Charged but Order Failed

If payment succeeded but the order ultimately cannot be completed:

```text
Payment Success
      ↓
Order Processing Failure
      ↓
Compensating Transaction
      ↓
Refund
```

Refund is required only when money was actually captured/debited.

---

# 16. Client Order Status

The client receives an `orderId` immediately and can query:

```text
GET /orders/{orderId}
```

### Recommended status flow

```text
PENDING
   │
   ├── CONFIRMED
   ├── FAILED
   └── EXPIRED
```

Response:

| Status | Client Behaviour |
|---|---|
| `PENDING` | Show "Processing..." |
| `CONFIRMED` | Show order confirmation |
| `FAILED` | Show failure |
| `EXPIRED` | Show timeout / retry |

For very high polling traffic, `orderId → status` can also be cached in Redis.

However:

> **DB remains the durable source of truth; Redis is the read cache.**

---

# 17. Idempotency

Flash sales generate many retries and duplicate messages.

Idempotency must exist at multiple levels.

### Order Idempotency

Use an idempotency key such as:

```text
userId + productId + sale/session
```

or a client-generated idempotency key.

Ensure the same request does not create multiple orders.

### Kafka Consumer Idempotency

Kafka can deliver messages more than once.

Worker should safely handle:

```text
OrderRequested
OrderRequested   ← duplicate
```

without creating another order or charging twice.

### Payment Idempotency

Payment requests should carry a unique:

```text
paymentId / idempotencyKey
```

so retries don't result in multiple charges.

### Stock Release Idempotency

Release stock only once:

```text
if stockReleaseAlreadyDone:
    ignore
else:
    INCR Redis
    mark release completed
```

---

# 18. Redis Crash During Flash Sale

Redis is extremely fast, but it should not be treated as the **only durable source of truth**.

### Recovery strategy

When Redis becomes unavailable:

```text
Redis Failure
     ↓
Enter DEGRADED MODE
     ↓
Stop / disable Buy Now
     ↓
Reconcile durable state
     ↓
Rebuild Redis stock
     ↓
Resume sale
```

### Reconciliation

Reconstruct availability using durable information such as:

```text
Available Stock
    =
Initial Stock
    - Confirmed Orders
    - Active Reservations
```

Also reconcile payment records for cases such as:

```text
Payment SUCCESS
but
Order DB not updated
```

Then reload:

```text
Redis stock = reconciled stock
```

and resume Buy Now.

---

# 19. Redis High Availability

To minimize downtime:

- Redis replication
- Automatic failover
- AOF/RDB persistence where appropriate
- Monitoring and alerting
- Controlled degraded mode
- Durable Kafka events for replay/reconciliation

The goal is:

> **Redis failure should temporarily stop new admissions, not corrupt the final business state.**

---

# 20. Responsibility Summary

| Component | Primary Responsibility |
|---|---|
| **Client** | Send Buy request and check order status |
| **API Gateway** | Routing, authentication, rate limiting, traffic protection |
| **Rate Limiter** | Control request admission rate |
| **Order Service** | Validate request, acquire Redis token, generate orderId, publish event |
| **Redis** | Fast real-time stock/token gate |
| **Lua / Atomic Counter** | Prevent race conditions and overselling |
| **Kafka** | Buffer and reliably transport order events |
| **Order Worker** | Async order processing/orchestration |
| **Payment Service** | Communicate with external payment provider |
| **Payment Gateway** | Execute actual payment |
| **Order DB** | Durable order state/history |
| **Inventory DB** | Durable inventory records / reconciliation |
| **Client Status API** | Return latest order state |

---

# 21. Final End-to-End Picture

```text
                ┌──────────────┐
                │   1M Clients │
                └──────┬───────┘
                       │
                       ▼
              ┌─────────────────┐
              │   API Gateway   │
              │  Rate Limiter   │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Order Service  │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │      Redis      │
              │ Atomic Lua Check│
              └────────┬────────┘
                       │
              Token Acquired
                       │
                       ▼
              ┌─────────────────┐
              │      Kafka      │
              │ OrderRequested  │
              └────────┬────────┘
                       │
                       ▼
          ┌─────────────────────────┐
          │ Order Processing Worker │
          └────────────┬────────────┘
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
       Payment Service        Order DB
             │
             ▼
      External Gateway
             │
             ▼
       Payment Event
             │
             ▼
          Worker
             │
       ┌─────┴─────┐
       ▼           ▼
  CONFIRMED     FAILED
                   │
                   ▼
             Release Token
               Redis INCR
```

---

# 🎯 Core Design Principles

### 1. Rate Limiter
**Protect the system from traffic explosion.**

### 2. Redis
**Fast admission control for scarce inventory.**

### 3. Atomic Lua Counter
**Guarantee that stock allocation is race-condition safe.**

### 4. Kafka
**Buffer the admitted requests and decouple ingestion from processing.**

### 5. Order Worker
**Perform heavy asynchronous business processing.**

### 6. DB
**Durably record the final business outcome.**

### 7. Idempotency
**Make retries and duplicate events safe.**

### 8. Redis Recovery
**Stop admissions → reconcile durable state → rebuild Redis → resume.**

---

## 🧠 The Mental Model

```text
Rate Limiter → "How many can enter?"
Redis        → "Who gets the scarce stock?"
Kafka        → "Who waits for processing?"
Worker       → "Process the order."
Payment      → "Did money move?"
DB           → "What is the durable outcome?"
Client       → "What is my order status?"
```

> **The fundamental idea: don't allow 1M users to reach the expensive order-processing path. Use rate limiting + atomic Redis admission + asynchronous Kafka processing to reduce millions of concurrent requests into a small, controlled set of actual order attempts.**