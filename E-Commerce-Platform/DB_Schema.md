# Ecommerce Transactional DB Design — Summary

## Core Principle

Design OLTP databases primarily for **correctness, concurrency safety, and efficient writes**, then use selective denormalization where it materially improves read performance or preserves historical snapshots.

---

# 1. User Domain

## `users`

```text
user_id (PK)
email (UNIQUE)
phone (UNIQUE)
password_hash
account_status
created_at
last_login_at
```

## `user_profiles`

```text
user_id (PK, FK → users.user_id)
first_name
last_name
date_of_birth
gender
avatar_url
updated_at
```

## `user_addresses`

```text
address_id (PK)
user_id (FK → users.user_id)

label
full_name
phone
line1
line2
city
state
postal_code
country

is_default
created_at
```

> Orders should store a **shipping/billing address snapshot**, because the user's address may change later.

---

# 2. Product & Seller Domain

## `products`

```text
product_id (PK)
name
brand
category_id
description
created_at
```

## `sellers`

```text
seller_id (PK)
seller_name
rating
created_at
```

## `seller_products`

Many-to-many relationship between sellers and products.

```text
seller_product_id (PK)

seller_id (FK → sellers.seller_id)
product_id (FK → products.product_id)

price
stock_qty
shipping_time
status

created_at
updated_at

UNIQUE (seller_id, product_id)
```

> **Product = WHAT is being sold**  
> **Seller = WHO is selling it**  
> **Seller_Product = HOW it is being sold**

---

# 3. Order Domain

## `orders`

Order header.

```text
order_id (PK)
user_id (FK → users.user_id)

total_amount
total_items

status
payment_status
payment_collected_amount

shipping_address_snapshot
billing_address_snapshot

created_at
updated_at
```

## `order_items`

Individual products purchased in an order.

```text
order_item_id (PK)

order_id (FK → orders.order_id)

product_id
seller_id

product_name_snapshot
seller_name_snapshot

price_at_purchase
quantity
line_total

created_at
```

> Order items contain **snapshots** of important historical information so that future product/seller changes don't alter old orders.

---

# 4. Payment Domain

## `payments`

Represents payment attempts/business state.

```text
payment_id (PK)

order_id (FK → orders.order_id)
user_id (FK → users.user_id)

amount
currency

payment_method
payment_provider

provider_payment_ref
provider_order_ref

status
    created
    authorized
    captured
    failed
    cancelled

failure_reason

created_at
authorized_at
captured_at
```

One order can have multiple payment attempts.

## `payment_transactions`

Stores gateway events/webhook history.

```text
txn_id (PK)

payment_id (FK → payments.payment_id)

event_type
provider_event_id
provider_signature

event_payload (JSON)

created_at
```

Examples:

```text
payment.created
payment.authorized
payment.captured
payment.failed
```

> `payments` = current business state  
> `payment_transactions` = gateway event/audit history

---

# 5. Inventory Domain

## `inventory`

Live stock at seller-product level.

```text
inventory_id (PK)

seller_product_id (FK → seller_products.seller_product_id)

available_qty
reserved_qty
sold_qty

updated_at
```

## `inventory_reservations`

Tracks temporary stock reservations.

```text
reservation_id (PK)

inventory_id (FK → inventory.inventory_id)

order_id
order_item_id

quantity

status
    active
    released
    converted

expires_at
created_at
```

## `inventory_adjustments`

Tracks operational/manual inventory changes.

```text
adjustment_id (PK)

inventory_id (FK → inventory.inventory_id)

change_qty
reason

    restock
    damaged
    return_received
    manual_correction

reference_id

created_at
```

---

# 6. Shipment Domain

## `shipments`

Represents a physical package/shipment.

```text
shipment_id (PK)

order_id (FK → orders.order_id)
seller_id (FK → sellers.seller_id)

warehouse_id

tracking_number
courier_partner

shipment_status
    created
    packed
    shipped
    in_transit
    out_for_delivery
    delivered
    failed

shipped_at
delivered_at
created_at
```

> `seller_id` is intentionally stored here as an FK because shipment ownership is a meaningful business attribute and is frequently queried operationally.

## `shipment_items`

Maps order items to physical shipments.

```text
shipment_item_id (PK)

shipment_id (FK → shipments.shipment_id)
order_item_id (FK → order_items.order_item_id)

quantity_shipped
```

This supports split shipments and partial quantities.

---

# 7. Returns Domain

## `returns`

Return request.

```text
return_id (PK)

order_id (FK → orders.order_id)
user_id (FK → users.user_id)

return_status
    requested
    approved
    pickup_scheduled
    picked
    rejected
    completed

return_reason

created_at
approved_at
completed_at
```

## `return_items`

Individual items being returned.

```text
return_item_id (PK)

return_id (FK → returns.return_id)
order_item_id (FK → order_items.order_item_id)

quantity

resolution_type
    refund
    replacement

item_condition
    unopened
    used
    damaged
```

> Returns are handled at **item level**, not simply at order level.

---

# 8. Refund Domain

## `refunds`

Represents the actual money being returned.

```text
refund_id (PK)

return_id (FK → returns.return_id)
payment_id (FK → payments.payment_id)

refund_amount

refund_status
    initiated
    processing
    success
    failed

gateway_refund_id

created_at
processed_at
```

## `refund_transactions`

Stores refund gateway events/webhook history.

```text
id (PK)

refund_id (FK → refunds.refund_id)

event_type

provider_refund_id

event_payload (JSON)

created_at
```

Examples:

```text
refund.initiated
refund.processed
refund.failed
```

---

# Entity Relationship Overview

## Overall

```text
                         ┌──────────────┐
                         │    USERS     │
                         └──────┬───────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
       USER_PROFILES      USER_ADDRESSES        ORDERS
                                                    │
                           ┌────────────────────────┼───────────────────┐
                           │                        │                   │
                           ▼                        ▼                   ▼
                      ORDER_ITEMS               PAYMENTS            SHIPMENTS
                           │                        │                   │
                           │                        │                   ▼
                           │                        │             SHIPMENT_ITEMS
                           │                        │
                           │                        ▼
                           │               PAYMENT_TRANSACTIONS
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
          RETURN_ITEMS            SHIPMENT_ITEMS
                │
                ▼
             RETURNS
                │
                ▼
             REFUNDS
                │
                ▼
       REFUND_TRANSACTIONS
```

## Product / Seller / Inventory

```text
       PRODUCTS
           │
           │
           ▼
   SELLER_PRODUCTS
           ▲
           │
           │
        SELLERS
           │
           │
           ▼
       INVENTORY
           │
      ┌────┴──────────────┐
      ▼                   ▼
INVENTORY_RESERVATIONS   INVENTORY_ADJUSTMENTS
```

---

# Key Design Principles

## 1. Normalize transactional data

> **OLTP → primarily normalized for correctness and efficient writes.**

Denormalize selectively when a field is:
- frequently queried, or
- a historical snapshot.

## 2. Keep independent state machines independent

```text
Order      → Order Status
Payment    → Payment Status
Shipment   → Shipment Status
Return     → Return Status
Refund     → Refund Status
```

Do not try to represent all of these with one `order_status`.

## 3. Preserve historical truth

Examples:

```text
order_items.price_at_purchase
order_items.product_name_snapshot
orders.shipping_address_snapshot
```

These should not change merely because current catalog/user data changes.

## 4. Money movement ≠ physical movement

```text
Order
 ├── Payment → Refund
 │
 └── Shipment → Return
```

They interact, but should remain separate domains.

## 5. PK recommendation

For a conventional OLTP database:

```text
BIGINT / sequence-based PK
```

is generally preferable for internal database relationships because of index and storage efficiency.

If public IDs need to be non-enumerable or globally unique, a common pattern is:

```text
id         BIGINT PK
public_id  UUID UNIQUE
```

The database joins on BIGINT, while APIs can expose the UUID.
