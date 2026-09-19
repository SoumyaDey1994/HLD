# Document Access Control — DB Design

## 1. Overview

The system manages:

- Users
- Documents and their owners
- Access permissions granted to users for specific documents
- Audit history of access changes

### Core relationship

```text
User
 ├── owns ────────────────> Documents
 │
 └── receives access ─────> Document Access
                              │
                              └── Access Level
```

### Example

```text
User A owns Doc1

A ──owns──> Doc1

A ──shares READ──> B
A ──shares WRITE─> C
```

---

# 2. `users`

Stores system users.

| Column | Type | Constraints / Notes |
|---|---|---|
| `user_id` | BIGINT | **PK** |
| `name` | VARCHAR(255) | NOT NULL |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE |
| `created_at` | TIMESTAMP | NOT NULL |

### Indexes

```text
PK(user_id)
UNIQUE(email)
```

### Example

| user_id | name | email |
|---:|---|---|
| 101 | A | a@example.com |
| 102 | B | b@example.com |
| 103 | C | c@example.com |

---

# 3. `documents`

Stores documents and their ownership.

| Column | Type | Constraints / Notes |
|---|---|---|
| `document_id` | BIGINT | **PK** |
| `title` | VARCHAR(255) | NOT NULL |
| `content` | TEXT / JSON / separate storage reference | Depends on document architecture |
| `owner_id` | BIGINT | **FK → users.user_id**, NOT NULL |
| `created_at` | TIMESTAMP | NOT NULL |

### Constraints

```text
PK(document_id)

FK(owner_id)
    → users.user_id
```

Every document has exactly one owner.

### Indexes

```text
PK(document_id)

INDEX(owner_id)
```

`owner_id` should be indexed because queries such as:

```text
"Give me all documents owned by user A"
```

are common.

### Example

| document_id | title | owner_id |
|---:|---|---:|
| 1001 | Doc1 | 101 |
| 1002 | Doc2 | 101 |
| 1003 | Doc3 | 103 |

Here:

```text
Doc1 → owned by A
Doc2 → owned by A
Doc3 → owned by C
```

---

# 4. `access_levels`

Master/reference table containing the available access permissions.

| Column | Type | Constraints / Notes |
|---|---|---|
| `access_level_id` | INT | **PK** |
| `name` | VARCHAR(50) | NOT NULL, UNIQUE |
| `priority` | INT | Optional |
| `description` | VARCHAR(255) | Optional |

### Example

| access_level_id | name | priority | description |
|---:|---|---:|---|
| 1 | READ | 1 | Can view document |
| 2 | WRITE | 2 | Can modify document |
| 3 | DELETE | 3 | Can delete document |

### Important caveat about `priority`

`priority` should only be used if your application's permissions have a genuine hierarchy.

For example:

```text
READ < WRITE
```

may make sense.

But:

```text
DELETE > WRITE
```

does **not automatically mean** that someone with DELETE permission should receive WRITE permission.

So don't blindly derive permissions from `priority`.

If permissions are independent capabilities, model them explicitly instead.

---

# 5. `document_access`

This is the **main access-control mapping table**.

It represents the current access granted to a user for a document.

| Column | Type | Constraints / Notes |
|---|---|---|
| `id` | BIGINT | **PK** |
| `document_id` | BIGINT | **FK → documents.document_id**, NOT NULL |
| `user_id` | BIGINT | **FK → users.user_id**, NOT NULL |
| `access_level_id` | INT | **FK → access_levels.access_level_id**, NOT NULL |
| `granted_by` | BIGINT | **FK → users.user_id**, NOT NULL |
| `granted_at` | TIMESTAMP | NOT NULL |

### Important constraint

```text
UNIQUE(document_id, user_id)
```

This guarantees:

> One user can have only one current access entry for a particular document.

### Indexes

```text
PK(id)

UNIQUE(document_id, user_id)

INDEX(user_id)

INDEX(document_id)
```

The unique composite index on:

```text
(document_id, user_id)
```

is particularly important for the common access-check query:

```text
Does user B have access to Doc1?
```

---

# 6. Example — Sharing a Document

Suppose:

```text
A owns Doc1
B gets READ access
C gets WRITE access
```

`documents`:

| document_id | owner_id |
|---|---:|
| 1001 | 101 |

`document_access`:

| id | document_id | user_id | access_level_id | granted_by |
|---:|---:|---:|---:|---:|
| 1 | 1001 | 102 | 1 (READ) | 101 |
| 2 | 1001 | 103 | 2 (WRITE) | 101 |

So:

```text
A ──owner──> Doc1

A ──READ───> B
A ──WRITE──> C
```

---

# 7. Owner Access

The owner does **not** need a row in `document_access`.

Ownership is represented by:

```text
documents.owner_id
```

Therefore:

```text
user_id == documents.owner_id
        ↓
    Owner access
```

The application can treat the owner as having all required document-management permissions.

This avoids duplicating ownership information in the access table.

---

# 8. What Does "No Access" Mean?

For the current-state `document_access` table:

```text
No row
   ↓
No explicit access
   ↓
DENY
```

There should normally be **no `NO_ACCESS` row**.

For example:

```text
Doc1 + B + READ
```

exists initially.

If A revokes B's access:

```text
DELETE document_access
WHERE document_id = Doc1
AND user_id = B
```

After deletion:

```text
Doc1 + B
   ↓
No document_access row
   ↓
No access
```

---

# 9. Grant / Update / Revoke Operations

## 9.1 Grant Access

If B currently has no access:

```text
INSERT
```

Example:

```text
Doc1 → B → READ
```

creates:

```text
document_id = 1001
user_id = 102
access_level_id = READ
granted_by = A
```

---

## 9.2 Change Access

Suppose:

```text
B currently has READ
```

and A changes it to:

```text
WRITE
```

Update the existing row:

```text
READ → WRITE
```

Do **not** create a second row because:

```text
UNIQUE(document_id, user_id)
```

allows only one current permission.

---

## 9.3 Revoke Access

Suppose:

```text
B has WRITE access
```

and A revokes it.

Recommended current-state operation:

```text
DELETE document_access row
```

Result:

```text
No row = No access
```

This keeps the current-state table clean and simple.

---

# 10. `document_access_audit`

`document_access` represents **current state**.

If we delete a row during revocation, however, we lose the historical information.

Therefore, if audit/history is required, maintain a separate audit table.

| Column | Type | Constraints / Notes |
|---|---|---|
| `audit_id` | BIGINT | **PK** |
| `document_id` | BIGINT | **FK → documents.document_id** |
| `user_id` | BIGINT | **FK → users.user_id** |
| `old_access_level_id` | INT | **FK → access_levels.access_level_id**, NULL allowed |
| `new_access_level_id` | INT | **FK → access_levels.access_level_id**, NULL allowed |
| `action` | VARCHAR(20) | GRANTED / UPDATED / REVOKED |
| `performed_by` | BIGINT | **FK → users.user_id** |
| `performed_at` | TIMESTAMP | NOT NULL |

### Indexes

```text
PK(audit_id)

INDEX(document_id, user_id)

INDEX(performed_at)

INDEX(performed_by)
```

The `(document_id, user_id)` index helps retrieve the complete access history for a particular user/document pair.

---

# 11. Audit Examples

## Grant

B receives READ access.

| document_id | user_id | old | new | action | performed_by |
|---:|---:|---|---|---|---:|
| 1001 | 102 | NULL | READ | GRANTED | 101 |

Meaning:

```text
No previous access → READ
```

---

## Permission Change

B's access changes:

```text
READ → WRITE
```

| document_id | user_id | old | new | action | performed_by |
|---:|---:|---|---|---|---:|
| 1001 | 102 | READ | WRITE | UPDATED | 101 |

---

## Revoke

B's WRITE access is revoked.

| document_id | user_id | old | new | action | performed_by |
|---:|---:|---|---|---|---:|
| 1001 | 102 | WRITE | NULL | REVOKED | 101 |

### Important

For a revoke operation:

```text
old_access_level_id = WRITE
new_access_level_id = NULL
```

Do **not** store:

```text
new_access_level_id = NO_ACCESS
```

because the audit record represents the **state transition**:

```text
WRITE → No access
```

rather than treating `NO_ACCESS` as an actual permission.

---

# 12. Final Schema Relationship

```text
                    ┌──────────────┐
                    │    users     │
                    │──────────────│
                    │ user_id (PK) │
                    └──────┬───────┘
                           │
             ┌─────────────┴──────────────┐
             │                            │
          owner_id                    user_id
             │                            │
             ▼                            ▼
      ┌──────────────┐            ┌─────────────────┐
      │  documents   │            │ document_access │
      │──────────────│            │─────────────────│
      │ document_id  │◄───────────│ document_id     │
      │ owner_id     │            │ user_id         │
      └──────────────┘            │ access_level_id │
                                  │ granted_by      │
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌────────────────┐
                                  │ access_levels  │
                                  │────────────────│
                                  │ access_level_id│
                                  │ name           │
                                  └────────────────┘


              Historical changes
                       │
                       ▼
             ┌──────────────────────┐
             │ document_access_audit│
             │──────────────────────│
             │ audit_id             │
             │ document_id          │
             │ user_id              │
             │ old_access_level_id  │
             │ new_access_level_id  │
             │ action               │
             │ performed_by         │
             │ performed_at         │
             └──────────────────────┘
```

---

# 13. Complete Schema — Compact View

### `users`

```text
user_id       BIGINT          PK
name          VARCHAR(255)    NOT NULL
email         VARCHAR(255)    NOT NULL, UNIQUE
created_at    TIMESTAMP       NOT NULL
```

**Indexes**

```text
PK(user_id)
UNIQUE(email)
```

---

### `documents`

```text
document_id   BIGINT          PK
title         VARCHAR(255)    NOT NULL
content       TEXT/JSON       Optional
owner_id      BIGINT          FK → users.user_id, NOT NULL
created_at    TIMESTAMP       NOT NULL
```

**Indexes**

```text
PK(document_id)
INDEX(owner_id)
```

---

### `access_levels`

```text
access_level_id   INT          PK
name              VARCHAR(50)  NOT NULL, UNIQUE
priority          INT          Optional
description       VARCHAR(255) Optional
```

**Indexes**

```text
PK(access_level_id)
UNIQUE(name)
```

---

### `document_access`

```text
id                BIGINT       PK
document_id       BIGINT       FK → documents.document_id
user_id           BIGINT       FK → users.user_id
access_level_id   INT          FK → access_levels.access_level_id
granted_by        BIGINT       FK → users.user_id
granted_at        TIMESTAMP    NOT NULL
```

**Constraints**

```text
UNIQUE(document_id, user_id)
```

**Indexes**

```text
PK(id)
UNIQUE(document_id, user_id)
INDEX(user_id)
INDEX(document_id)
```

---

### `document_access_audit`

```text
audit_id              BIGINT        PK
document_id           BIGINT        FK → documents.document_id
user_id               BIGINT        FK → users.user_id
old_access_level_id   INT           FK → access_levels.access_level_id, NULL
new_access_level_id   INT           FK → access_levels.access_level_id, NULL
action                VARCHAR(20)   NOT NULL
performed_by          BIGINT        FK → users.user_id
performed_at          TIMESTAMP     NOT NULL
```

**Indexes**

```text
PK(audit_id)
INDEX(document_id, user_id)
INDEX(performed_at)
INDEX(performed_by)
```

---

# 14. Recommended Operating Model

```text
                  CURRENT STATE
                       │
                       ▼
             ┌──────────────────┐
             │ document_access  │
             └──────────────────┘
                       │
            ┌──────────┼──────────┐
            │          │          │
          GRANT      UPDATE     REVOKE
            │          │          │
         INSERT      UPDATE     DELETE
            │          │          │
            └──────────┼──────────┘
                       │
                       ▼
             ┌──────────────────┐
             │ access_audit     │
             │ INSERT event     │
             └──────────────────┘
```

### Core rule

> **`document_access` = current permission state**  
> **`document_access_audit` = historical state transitions**

---

# 15. Important Caveats

### 1. Don't store `NO_ACCESS` by default

Use:

```text
No document_access row = No access
```

This keeps the current-state table compact.

---

### 2. Keep audit separate

Deleting an access row should not mean losing history.

```text
DELETE current access
+
INSERT REVOKED audit event
```

Ideally these operations happen within the same transaction.

---

### 3. Don't duplicate owner access

Don't create:

```text
Doc1 → A → OWNER
```

in `document_access` if ownership is already represented by:

```text
documents.owner_id = A
```

Otherwise, ownership and permission state can become inconsistent.

---

### 4. `granted_by` and `performed_by` have different meanings

`document_access.granted_by`:

> Who created/granted the current access?

`document_access_audit.performed_by`:

> Who performed this particular access change?

---

### 5. Explicit DENY is different from NO ACCESS

For the simple design:

```text
No row = No access
```

But if the system later supports:

```text
User access
+ Group access
+ Folder inheritance
+ Organization access
```

then an explicit `DENY` may become necessary.

Example:

```text
Group → READ
User B → DENY
```

Here, merely having no user-level row cannot distinguish:

```text
"No specific permission"
```

from:

```text
"Explicitly denied despite inherited permission"
```

At that point, a `DENY`/`NO_ACCESS` concept can be introduced deliberately.

---

# Final Design Principle

For the current simple document-sharing system:

```text
Owner
  → documents.owner_id

Shared access
  → document_access

Permission definition
  → access_levels

Access history
  → document_access_audit

No current access
  → absence of document_access row
```

This gives a clean separation between **ownership**, **current authorization**, **permission definitions**, and **historical audit information**.