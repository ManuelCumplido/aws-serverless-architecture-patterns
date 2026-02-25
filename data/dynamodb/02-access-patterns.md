# Access Patterns (Query Contracts) — SmartLocker Domain

This document formalizes the SmartLocker domain access patterns as **strict query contracts** for DynamoDB single-table design.

---

# Table: SmartLockerTable (Single Table)

## Primary Keys

- PK (string)
- SK (string)

## Common Attributes

- entityType (string)
- createdAt (ISO string or epoch millis)
- updatedAt (ISO string or epoch millis)
- version (number, for optimistic concurrency)

## Key Prefix Conventions

Locker partition:
- LOCKER#<lockerId>

Owner partition:
- OWNER#<ownerId>

Sort key prefixes:
- META
- LOCKER#<lockerId>
- RES#<startISO>#<reservationId>

---

# Access Patterns

---

## AP-01 — Get locker by lockerId

### Goal
Fetch a single locker’s canonical metadata/state.

### Type
Read (hot path)

### Inputs
- lockerId (required)

### DynamoDB Operation
GetItem

### Keys
- PK = LOCKER#<lockerId>
- SK = META

### Expected Cardinality
Exactly 1 item

### Consistency
- Default: Eventually consistent
- Optional: Strongly consistent if immediate read-after-write is required

### Failure Modes
- Item not found → return 404

---

## AP-02 — List lockers for an owner

### Goal
List all lockers owned by an ownerId.

### Type
Read (hot path)

### Inputs
- ownerId (required)
- pageSize (optional, default 25)
- nextCursor (optional)

### DynamoDB Operation
Query

### Key Condition

PK = OWNER#<ownerId>  
AND begins_with(SK, "LOCKER#")

### Ordering
Lexicographic by lockerId

### Pagination
- Use Limit = pageSize
- Return LastEvaluatedKey as nextCursor

### Expected Cardinality
0–200 lockers typical

### Notes
- Denormalize status, lockerAlias, updatedAt into relationship item
- Avoid N+1 queries to META

---

## AP-03 — Update locker state (Optimistic Concurrency)

### Goal
Safely transition locker state (AVAILABLE / OCCUPIED / MAINTENANCE).

### Type
Write (hot path)

### Inputs
- lockerId
- newStatus
- expectedVersion

### DynamoDB Operation
UpdateItem

### Keys
- PK = LOCKER#<lockerId>
- SK = META

### UpdateExpression (Example)

SET #status = :newStatus,  
    updatedAt = :now  

### ConditionExpression

version = :expectedVersion

### Failure Mode
ConditionalCheckFailedException → return 409 Conflict

### Notes
- Enforce allowed state transitions in application layer (state machine)

---

## AP-04 — Create reservation for a locker

### Goal
Create a time-based reservation.

### Type
Write

### Inputs
- lockerId
- reservationId
- ownerId
- startAt
- endAt

### DynamoDB Operation
- PutItem
- Recommended: TransactWriteItems if also updating RES#ACTIVE

### Keys
- PK = LOCKER#<lockerId>
- SK = RES#<startISO>#<reservationId>

### Optional Condition

attribute_not_exists(PK) AND attribute_not_exists(SK)

### Notes
- Time ordering enforced by sort key
- Overlap validation handled at application layer (if required)

---

## AP-05 — List reservations for a locker (time range)

### Goal
Fetch reservations within a time window.

### Type
Read

### Inputs
- lockerId
- startISO
- endISO
- pageSize
- nextCursor

### DynamoDB Operation
Query

### Key Condition

PK = LOCKER#<lockerId>  
AND SK BETWEEN  
"RES#<startISO>"  
AND "RES#<endISO>~"

The "~" ensures lexicographic upper-bound inclusion.

### Ordering
- Ascending by default
- Descending: ScanIndexForward = false

### Pagination
- Use Limit
- Return nextCursor

### Expected Cardinality
Depends on reservation volume per locker

---


## AP-06 — Multi-item consistency (Transactions)

### Goal
Maintain denormalized view consistency.

### Example Use Case
When updating locker status:
- Update LOCKER#<lockerId>/META
- Update OWNER#<ownerId>/LOCKER#<lockerId>

### DynamoDB Operation
TransactWriteItems

### Notes
- Keep transactions small (≤ 25 items)
- Higher cost but strong consistency guaranteed

---

## AP-07 — Transfer locker ownership

### Goal
Move locker to a new owner.

### DynamoDB Operation
TransactWriteItems

### Steps
1. Update LOCKER#<lockerId>/META.ownerId  
2. Delete OWNER#<oldOwnerId>/LOCKER#<lockerId>  
3. Put OWNER#<newOwnerId>/LOCKER#<lockerId>  

### Notes
- Validate authorization
- Handle missing relationship item safely

---

# Pagination Standard

All list endpoints must return:

```json
{
  "items": [],
  "nextCursor": "opaque-string-or-null"
}