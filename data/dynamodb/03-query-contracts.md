# Query Contracts — SmartLocker Domain

This document formalizes **strict query contracts** for all access patterns in the SmartLocker domain.

While `02-access-patterns.md` defines *what* the system must support,  
this document defines *exactly how each query must behave* at runtime.

Each contract specifies:

- Inputs
- DynamoDB operation
- KeyConditionExpression
- Pagination rules
- Ordering
- Expected cardinality
- Consistency model
- Cost characteristics
- Failure modes
- Idempotency expectations (if applicable)

---

# Contract Principles

All query contracts must:

1. Avoid Scan
2. Avoid filter-based post-query filtering
3. Be bounded (O(1) or limited Query)
4. Define pagination rules
5. Explicitly state consistency requirements
6. Explicitly state error semantics

---

# QC-01 — Get Locker Metadata

## Purpose
Fetch canonical locker metadata/state.

## Inputs
- lockerId (required)

## Operation
GetItem

## Keys
PK = LOCKER#<lockerId>  
SK = META  

## Consistency
- Default: Eventually consistent
- Optional: Strongly consistent if called immediately after write

## Expected Cardinality
Exactly 1 item

## Response Example
{
  "lockerId": "123",
  "ownerId": "999",
  "status": "AVAILABLE",
  "updatedAt": "2026-02-25T10:00:00Z",
  "version": 4
}

## Failure Modes
- Not found → 404
- Unauthorized → 403

## Cost Characteristics
- 1 RCU (eventual)
- 2 RCUs (strong)

---

# QC-02 — List Lockers by Owner

## Purpose
Return paginated list of lockers owned by ownerId.

## Inputs
- ownerId (required)
- pageSize (default 25, max 100)
- nextCursor (optional)

## Operation
Query

## KeyConditionExpression
PK = OWNER#<ownerId>  
AND begins_with(SK, "LOCKER#")

## Ordering
Ascending lexicographic order by lockerId

## Pagination
- Use Limit = pageSize
- Return Base64-encoded LastEvaluatedKey as nextCursor

## Expected Cardinality
- Typical: 0–200 lockers per owner
- Worst-case: higher for organization accounts

## Response Example
{
  "items": [
    {
      "lockerId": "123",
      "status": "AVAILABLE",
      "lockerAlias": "Front Gate"
    }
  ],
  "nextCursor": "opaque-string-or-null"
}

## Cost Characteristics
- Linear in number of items returned
- Efficient partition-level query
- No Scan required

---

# QC-03 — Update Locker State

## Purpose
Change locker status safely using optimistic concurrency.

## Inputs
- lockerId
- newStatus
- expectedVersion

## Operation
UpdateItem

## Keys
PK = LOCKER#<lockerId>  
SK = META  

## ConditionExpression
version = :expectedVersion

## UpdateExpression
SET #status = :newStatus,  
    updatedAt = :now  
ADD version :one  

## Failure Modes
- ConditionalCheckFailedException → 409 Conflict
- Not found → 404

## Idempotency
Not inherently idempotent.  
Client must retry with refreshed version if conflict occurs.

## Cost
1 WCU

---

# QC-04 — Create Reservation

## Purpose
Create a reservation entry for a locker.

## Inputs
- lockerId
- reservationId
- ownerId
- startAt
- endAt

## Operation
PutItem  
Optional: TransactWriteItems if updating RES#ACTIVE pointer

## Keys
PK = LOCKER#<lockerId>  
SK = RES#<startISO>#<reservationId>  

## Idempotency Enforcement
attribute_not_exists(SK)

## Failure Modes
- Duplicate reservationId → 409
- Invalid time range → 400

## Cost
- 1 WCU per item
- Higher if transactional

---

# QC-05 — List Reservations (Time Range)

## Purpose
Return reservations within a given time window.

## Inputs
- lockerId
- startISO
- endISO
- pageSize
- nextCursor

## Operation
Query

## KeyConditionExpression
PK = LOCKER#<lockerId>  
AND SK BETWEEN  
"RES#<startISO>"  
AND "RES#<endISO>~"

The "~" ensures lexicographic upper bound inclusion.

## Ordering
- Ascending by default
- Descending using ScanIndexForward = false

## Pagination
Standard pagination contract

## Expected Cardinality
Depends on reservation frequency and window size.

## Cost Characteristics
Bounded query (no Scan)

---

# QC-06 — Transfer Locker Ownership

## Purpose
Move locker from one owner to another.

## Operation
TransactWriteItems

## Steps
1. Update LOCKER#<lockerId>/META.ownerId  
2. Delete OWNER#<oldOwnerId>/LOCKER#<lockerId>  
3. Put OWNER#<newOwnerId>/LOCKER#<lockerId>  

## Guarantees
Atomic all-or-nothing execution

## Cost
Higher WCU usage due to transactional overhead

---

# Standard Pagination Contract

All list endpoints must return:

{
  "items": [],
  "nextCursor": "opaque-string-or-null"
}

## Cursor Encoding
- Base64-encoded LastEvaluatedKey
- Opaque to clients

## Default Page Size
25

## Maximum Page Size
100

---

# Consistency Model Summary

| Operation Type | Consistency |
|---------------|------------|
| GetItem hot read | Optional strong |
| List queries | Eventual |
| Writes | Atomic per item |
| Transactions | Strong atomicity |

---

# Performance Guarantees

- No Scan operations
- All reads are partition-bounded
- Writes are O(1)
- Hot partition risk limited to high-frequency lockers
- Horizontal scalability via partition key distribution

---

# Design Validation Checklist

Before introducing a new feature:

- Define business access pattern
- Write query contract
- Confirm no Scan required
- Confirm bounded Query
- Estimate cardinality
- Estimate read/write frequency
- Assess hot partition risk
- Evaluate need for GSI
- Document consistency requirement

---

Next:
dynamodb/04-gsi-strategies.md