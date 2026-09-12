# E-commerce Order Processing Workflow

## 1. Scope

This document describes a compact, production-oriented order workflow covering:

**User → Order → Merchant/Product Validation → Inventory → Payment → Order Confirmation → Fulfillment/Notification**

### Example Order

| Item | Quantity |
|---|---:|
| Item 1 | 2 |
| Item 2 | 3 |

---

## 2. High-Level Flow

```text
User
  │
  ▼
API Gateway
  │
  ▼
Order Service
  │
  ├──► Merchant / Product Service
  │       └── Validate items, price & availability
  │
  ├──► Inventory Service
  │       └── Reserve stock (SYNC)
  │
  └──► Payment Service
          └── Initiate payment (ASYNC)
                    │
                    ▼
             Payment Gateway
                    │
                    │ Webhook
                    ▼
             Payment Service
                    │
                    │ PAYMENT_SUCCESS event
                    ▼
              Order Service
                    │
                    ├──► Confirm Order
                    │
                    └──► ORDER_CONFIRMED event
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
             Inventory Service    Merchant/Fulfillment
             Deduct stock         Process shipment
                    │
                    ▼
              Confirmation /
              Notification Service
```

---

# 3. Step-by-Step Workflow

## Step 1 — User Places Order

The client sends an order request through the API Gateway.

```json
{
  "userId": "U123",
  "items": [
    { "itemId": "ITEM1", "quantity": 2 },
    { "itemId": "ITEM2", "quantity": 3 }
  ],
  "addressId": "ADDR1",
  "paymentMethod": "CARD"
}
```

**Order Service**
- Authenticates/validates the request.
- Validates quantities and required fields.
- Generates `orderId`.
- Creates the order with status `PENDING`.
- Stores the order before calling downstream services.

**Important:** Use an `idempotencyKey` so client retries do not create duplicate orders.

---

## Step 2 — Merchant / Product Validation

**Order Service → Merchant/Product Service**

Validate:
- Item exists and is orderable.
- Current selling price.
- Product/merchant availability.
- Merchant information required for fulfillment.

The Order Service stores a **price/product snapshot** in the order so later product-price changes do not alter an already-created order.

Example:

```text
ITEM1 → ₹100 × 2 = ₹200
ITEM2 → ₹200 × 3 = ₹600
--------------------------------
Order Total = ₹800
```

---

## Step 3 — Inventory Stock Check & Reservation

**Order Service → Inventory Service**

Request reservation:

```text
ITEM1 → Reserve 2
ITEM2 → Reserve 3
```

Inventory performs an atomic availability check and reservation.

### Recommended behavior

**Synchronous**

```text
Order Service
      │
      │ Reserve stock
      ▼
Inventory Service
      │
      ├── SUCCESS → continue
      └── FAILURE → order cannot proceed
```

Why synchronous?
- Gives immediate stock confirmation.
- Prevents overselling.
- Establishes a stock reservation before payment.

Order status can move to:

```text
PENDING → INVENTORY_RESERVED
```

**Do not permanently deduct stock yet.**

---

## Step 4 — Initiate Payment

**Order Service → Payment Service**

Payment Service creates a payment transaction for the order.

```text
orderId   = O123
amount    = ₹800
status    = PENDING
```

Payment Service interacts with the external Payment Gateway.

### Payment is generally asynchronous

The gateway may require:
- User interaction.
- 3-D Secure/authentication.
- Redirects.
- Additional processing time.

Therefore, the Order Service should not assume payment success merely because payment initiation succeeded.

Order status:

```text
INVENTORY_RESERVED → PAYMENT_PENDING
```

The client can receive something like:

```json
{
  "orderId": "O123",
  "status": "PAYMENT_PENDING"
}
```

---

## Step 5 — Payment Gateway Confirmation

The **Payment Gateway → Payment Service** sends the final payment result through a webhook.

```text
Payment Gateway
      │
      │ Webhook
      ▼
Payment Service
```

Payment Service:
1. Validates the webhook.
2. Updates `payment_transactions`.
3. Ensures webhook processing is idempotent.
4. Publishes an internal event such as:

```text
PAYMENT_SUCCESS
```

or

```text
PAYMENT_FAILED
```

### Important distinction

```text
Gateway → Payment Service = Webhook
Payment Service → Order Service = Internal Event/API
```

The external payment webhook should terminate at the Payment Service, not directly at the Order Service.

---

# 4. Payment Success Path

## Step 6 — Order Service Receives Payment Success

Order Service consumes `PAYMENT_SUCCESS`.

It verifies:
- Correct `orderId`.
- Correct payment amount.
- Expected order state.
- Payment transaction is successful.

Then the order can transition toward confirmation.

```text
PAYMENT_PENDING
      ↓
PAYMENT_SUCCESS
```

---

## Step 7 — Final Inventory Deduction

The reserved stock now needs to become a permanent stock deduction.

```text
RESERVED → DEDUCTED
```

### Recommended architecture

Publish:

```text
ORDER_CONFIRMED
```

Inventory Service consumes the event and performs the deduction asynchronously.

```text
Order Service
     │
     │ ORDER_CONFIRMED
     ▼
Inventory Service
     │
     └── Reserved stock → Deducted stock
```

Why async?
- Stock was already reserved.
- Overselling risk has already been controlled.
- Avoids making final order confirmation dependent on another synchronous network call.
- Retries can handle temporary Inventory Service failures.

The deduction operation **must be idempotent**.

---

# 5. Order Confirmation

## Step 8 — Confirm the Order

After successful payment validation, Order Service changes the order state:

```text
PAYMENT_PENDING
      ↓
CONFIRMED
```

The order now represents a successfully placed/paid order.

Example:

```json
{
  "orderId": "O123",
  "status": "CONFIRMED",
  "paymentStatus": "SUCCESS",
  "inventoryStatus": "RESERVED"
}
```

Inventory may subsequently update its own state to `DEDUCTED`.

---

# 6. Merchant / Fulfillment Interaction

## Step 9 — Notify Merchant / Fulfillment

Order Service publishes:

```text
ORDER_CONFIRMED
```

Consumers may include:
- Merchant Service
- Fulfillment/Shipment Service
- Inventory Service
- Notification Service
- Analytics Service

Merchant/Fulfillment uses the event to:
- Accept/process the order.
- Prepare items.
- Generate shipment information.
- Start fulfillment.

This interaction should generally be **asynchronous**.

---

# 7. Customer Confirmation

## Step 10 — Send Final Confirmation

Notification/Confirmation Service consumes:

```text
ORDER_CONFIRMED
```

It sends:
- Order confirmation email.
- SMS/push notification.
- Order ID.
- Ordered items and quantities.
- Amount paid.
- Expected delivery information.

Example:

```text
Order O123 confirmed successfully.

Item1 × 2
Item2 × 3

Total Paid: ₹800
```

Notification failure should normally **not roll back the order**. It should be retried independently.

---

# 8. Complete State Flow

```text
                    ┌─────────────────────┐
                    │   User Places Order │
                    └──────────┬──────────┘
                               ▼
                           PENDING
                               │
                               ▼
                    Product/Merchant Valid
                               │
                               ▼
                    INVENTORY_RESERVED
                               │
                               ▼
                     PAYMENT_PENDING
                               │
                  ┌────────────┴────────────┐
                  │                         │
             PAYMENT_SUCCESS          PAYMENT_FAILED
                  │                         │
                  ▼                         ▼
              CONFIRMED                  CANCELLED
                  │                         │
                  │                         ▼
                  │                  Release Reservation
                  │
                  ├──► Deduct Inventory (Async)
                  │
                  ├──► Merchant/Fulfillment
                  │
                  └──► Customer Confirmation
```

---

# 9. Failure / Compensation Flow

The workflow is a **Saga-style distributed transaction**. There is no single DB transaction spanning Order, Inventory, Payment and Merchant services.

| Failure | Action |
|---|---|
| Product validation fails | Cancel/fail order |
| Inventory reservation fails | Cancel/fail order |
| Payment fails | Release inventory reservation + cancel order |
| Payment remains pending | Keep order in `PAYMENT_PENDING` |
| Inventory deduction temporarily fails | Retry asynchronously; use DLQ/reconciliation |
| Merchant processing fails | Apply business-specific retry/cancellation workflow |
| Notification fails | Retry notification; do not cancel paid order |

---

# 10. Sync vs Async — Recommended Split

| Operation | Recommended | Reason |
|---|---|---|
| Create Order | **SYNC** | Return `orderId` immediately |
| Product/Merchant validation | **SYNC** | Validate order before committing |
| Inventory reservation | **SYNC** | Prevent overselling |
| Payment initiation | **ASYNC** | External gateway can be slow/interactive |
| Payment confirmation | **ASYNC** | Gateway webhook/event |
| Final order confirmation | **SYNC within Order Service** | State transition is local |
| Inventory deduction | **ASYNC** | Reservation already protects stock |
| Merchant fulfillment | **ASYNC** | Independent downstream workflow |
| Customer notification | **ASYNC** | Should not block order confirmation |

---

# 11. Critical Design Rules

### 1. Inventory Reservation ≠ Inventory Deduction

```text
Before payment:
Available → Reserved

After successful payment:
Reserved → Deducted
```

### 2. Payment Webhook Belongs to Payment Service

```text
Payment Gateway
      ↓ webhook
Payment Service
      ↓ event
Order Service
```

### 3. Every Async Consumer Must Be Idempotent

Events may be delivered more than once.

Use:
- Event ID / transaction ID.
- Unique constraints.
- Processed-event tracking where appropriate.

### 4. Use an Outbox Pattern

When Order/Payment services update their DB and publish an event, use a transactional outbox to avoid:

```text
DB updated
    +
Event lost
```

The outbox allows reliable event publication.

### 5. Use Compensation Instead of Distributed Transactions

Example:

```text
Inventory Reserved
      ↓
Payment Failed
      ↓
Release Inventory
      ↓
Order Cancelled
```

---

# 12. Final Mental Model

```text
CREATE
  ↓
VALIDATE
  ↓
RESERVE STOCK  ← synchronous
  ↓
INITIATE PAYMENT
  ↓
WAIT FOR PAYMENT WEBHOOK
  ↓
PAYMENT SUCCESS
  ↓
CONFIRM ORDER
  ↓
PUBLISH ORDER_CONFIRMED
  ├──► DEDUCT INVENTORY  ← async
  ├──► MERCHANT/FULFILLMENT
  └──► CUSTOMER CONFIRMATION
```

**Core principle:**

> **Reserve synchronously for correctness; process payment asynchronously for reliability; confirm the order after successful payment; then use events for downstream fulfillment, inventory deduction, and notifications.**
