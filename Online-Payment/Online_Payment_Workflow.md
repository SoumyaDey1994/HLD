# Online Payment — End-to-End Workflow

## 1. Overall Flow

```text
Customer
   │
   ▼
Order Portal
   │
   │ 1. Checkout
   ▼
Order Service
   │
   │ 2. Create Order + initiate Payment
   ▼
Payment Service
   │
   │ 3. Create Payment Transaction
   ▼
Payment Service Provider (PSP)
   │
   │ 4. Launch payment execution
   ▼
PSP Payment UI
   │
   │ 5. User selects/executes payment
   │    (UPI / Card / NetBanking / etc.)
   ▼
Bank / UPI / Card Network
   │
   │ 6. Process & authorize payment
   ▼
Payment Service Provider
   │
   ├──────────────► Payment Response
   │
   └──────────────► Webhook
                         │
                         ▼
                  Payment Service
                         │
                         │ 7. Verify + update
                         ▼
                    Payment DB
                         │
                         │ 8. Publish event
                         ▼
                       Kafka
                         │
                         ▼
                    Order Service
                         │
                         │ 9. Confirm order
                         ▼
                  Order = CONFIRMED
```

---

## 2. Step-by-Step Workflow

### Step 1 — Checkout

The customer completes the cart/checkout process in the **Order Portal** and chooses to pay.

```text
Customer → Order Portal → Checkout
```

The Order Portal is owned by the e-commerce/platform application.

---

### Step 2 — Order & Payment Initiation

The **Order Service** creates the order, typically with:

```text
Order Status = PENDING_PAYMENT
```

It then asks the **Payment Service** to initiate payment.

---

### Step 3 — Create Payment Transaction

The Payment Service creates its own payment transaction in the **Payment DB**.

```text
Payment Status = INITIATED
```

Typical information:

```text
payment_id
order_id
amount
currency
payment_method
gateway
status
```

---

### Step 4 — Connect to Payment Service Provider

The Payment Service calls the **Payment Service Provider (PSP)** to create/start the payment session.

The PSP returns the required checkout/session information.

```text
Payment Service → PSP
PSP → Payment Service
```

---

### Step 5 — PSP Payment UI

The customer is presented with the **PSP Payment UI**, where the actual payment is executed.

Examples:

```text
UPI        → QR / UPI Intent
Card       → Card details
NetBanking → Bank selection + authentication
Wallet     → Wallet authorization
```

> The payment-method selection UI can either be part of the Order Portal or be provided by the PSP's hosted checkout, depending on the integration model.

**COD is generally handled by the platform itself and does not require PSP payment execution.**

---

### Step 6 — Bank / Network Processes Payment

The PSP communicates with the appropriate financial network:

```text
PSP
 ↓
Bank / UPI / Card Network
 ↓
Payment Authorization
```

The payment is ultimately **SUCCESS**, **FAILED**, **CANCELLED**, etc.

---

### Step 7 — Webhook Confirms Payment

The PSP asynchronously sends a **Webhook** to the Payment Service.

```text
PSP
 ↓
Webhook
 ↓
Payment Service
```

Payment Service should:

- Verify webhook authenticity/signature.
- Validate payment/order/amount details.
- Handle duplicate webhooks idempotently.
- Update the payment transaction.

```text
INITIATED → SUCCESS
```

The webhook is the important backend confirmation mechanism; the browser/UI response alone should not be treated as the final source of truth.

---

### Step 8 — Publish Payment Event

After successfully updating the Payment DB, Payment Service publishes an event to **Kafka**.

```text
PaymentSucceeded
{
    payment_id
    order_id
    amount
}
```

Kafka decouples Payment Service from downstream consumers.

---

### Step 9 — Confirm Order

The **Order Service** consumes the Kafka event.

```text
PaymentSucceeded
       ↓
Order Service
       ↓
Order = CONFIRMED
```

Other consumers can independently react to the same event:

```text
Kafka
 ├──→ Order Service
 ├──→ Notification Service
 ├──→ Accounting
 └──→ Analytics
```

---

## 3. Responsibility of Each Component

| Component | Responsibility |
|---|---|
| **Order Portal** | Cart, checkout and customer-facing order flow |
| **Order Service** | Owns order lifecycle |
| **Payment Service** | Owns payment lifecycle and payment state |
| **Payment DB** | Persists payment transactions and status |
| **PSP** | Connects platform to payment networks |
| **PSP Payment UI** | Actual payment execution UI |
| **Bank / UPI / Card Network** | Processes and authorizes payment |
| **Webhook** | Async payment status notification from PSP |
| **Kafka** | Publishes payment events to downstream services |
| **Notification Service** | Sends payment/order notifications |

---

## 4. The Two Important Paths

### Synchronous — Payment Initiation

```text
Order Portal
    ↓
Order Service
    ↓
Payment Service
    ↓
PSP
    ↓
PSP Payment UI
    ↓
Customer Payment
```

### Asynchronous — Payment Confirmation

```text
PSP
    ↓
Webhook
    ↓
Payment Service
    ↓
Payment DB
    ↓
Kafka
    ↓
Order Service
    ↓
Order Confirmed
```

---

## 5. Core Principle

> **Order Portal initiates Checkout, Payment Service manages the payment lifecycle, PSP executes the payment, Webhook confirms the PSP result, Payment DB records the payment state, and Kafka propagates the successful payment event to the Order Service and other downstream services.**
