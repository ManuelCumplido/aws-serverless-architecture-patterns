# GSI Strategies — SmartLocker Domain

**Global Secondary Indexes (GSIs)** in the SmartLocker single-table design.

The core principle:

> A GSI is introduced only when a validated access pattern cannot be satisfied
> efficiently using the primary key structure.

GSIs are powerful but increase cost, complexity, and write amplification.
Therefore, they must be justified by concrete business-driven access patterns.

---

# 1. GSI Design Philosophy

## Rule #1 — Access Pattern First

Never create a GSI “just in case”.

Always:

1. Identify a business access pattern.
2. Write its Query Contract.
3. Attempt to satisfy it using the base table.
4. Introduce a GSI only if inevitable.

---

## Rule #2 — Avoid Over-Indexing

Every GSI:

- Doubles (or more) write cost
- Adds storage cost
- Adds operational complexity
- Introduces eventual consistency

If a query can be materialized instead, prefer materialization.

---

## Rule #3 — Prefer Materialized Views

Instead of:

- Adding GSI for owner → lockers

We materialized:

PK = OWNER#<ownerId>  
SK = LOCKER#<lockerId>

That avoided a GSI entirely.

Materialization is often cheaper than indexing.

---

# 2. When a GSI Becomes Necessary

A GSI is justified when:

- The partition key required by the access pattern differs from the base PK
- The query cannot be satisfied with begins_with or range query
- The access pattern is frequent and business-critical
- Materialization would introduce excessive duplication

---

# 3. Potential GSI Candidates in SmartLocker

Below are realistic future scenarios.

---

## GSI-01 — Query Lockers by Status (Global View)

### Business Requirement

Admin dashboard needs:

- All lockers in MAINTENANCE
- All lockers currently OCCUPIED

### Why Base Table Fails

Primary key groups by:

LOCKER#<lockerId>

There is no way to query by status globally without Scan.

### Proposed GSI

GSI1PK = STATUS#<status>  
GSI1SK = LOCKER#<lockerId>

### Indexed Items

Only index META items.

Example:

GSI1PK = STATUS#AVAILABLE  
GSI1SK = LOCKER#123

### Operation

Query GSI1PK = STATUS#AVAILABLE

### Trade-offs

Pros:
- Efficient global status view
- No Scan

Cons:
- Write amplification on every status update
- Eventual consistency

---

## GSI-02 — Query Reservations by Owner Across Lockers

### Business Requirement

User wants:

“My upcoming reservations”

Reservations are stored under:

PK = LOCKER#<lockerId>

So we cannot query by ownerId across lockers.

### Proposed GSI

GSI2PK = OWNER#<ownerId>  
GSI2SK = RES#<startISO>#<reservationId>

### Indexed Items

Only RESERVATION items.

### Operation

Query GSI2PK = OWNER#999

### Trade-offs

Pros:
- Efficient cross-locker reservation lookup

Cons:
- Extra write cost per reservation
- Must ensure projection only includes required fields

---

# 4. Projection Strategy

When defining a GSI, choose projection carefully.

Projection Types:

- KEYS_ONLY
- INCLUDE
- ALL

## Recommendation

Default to INCLUDE and project only necessary attributes.

Example:

For reservation lookup:
- reservationId
- lockerId
- startAt
- endAt
- status

Avoid projecting large metadata fields.

---

# 5. Cost Implications

Every write to a base item that is indexed:

- Consumes additional WCU for each GSI
- Increases storage
- Increases replication overhead

Example:

If reservation writes = 1000/sec  
And GSI2 exists → total write cost ≈ 2000 writes/sec

---

# 6. Consistency Model of GSIs

Important:

GSIs are ALWAYS eventually consistent.

Strongly consistent reads are not supported.

Implication:

- Do not use GSI for immediately consistent workflows
- Avoid using GSI for state-transition critical paths

---

# 7. Hot Partition Risk in GSIs

Bad Example:

GSI1PK = STATUS#AVAILABLE

If 90% of lockers are AVAILABLE,
that partition becomes hot.

Mitigation Strategies:

- Add sharding suffix
  STATUS#AVAILABLE#<bucketId>
- Or avoid global status queries
- Or paginate carefully

---

# 8. Decision Framework

Before introducing a GSI:

1. Is this access pattern frequent?
2. Is it latency-sensitive?
3. Can it be solved by materialization?
4. What is expected cardinality?
5. What is write amplification impact?
6. What is projected growth?
7. Is eventual consistency acceptable?

If at least 4 of 7 justify it → introduce GSI.

Otherwise → redesign.

---

# 9. Current SmartLocker Decision

For v1:

No GSIs required.

All current access patterns are satisfied by:

- Primary key
- Sort key range queries
- Materialized owner relationship

GSIs are deferred until a new access pattern requires them.

---

# 10. Architectural Maturity

In mature systems:

- GSIs are introduced gradually
- Each GSI is documented with:
  - Justification
  - Access pattern
  - Expected traffic
  - Cost estimate
  - Growth projection

GSIs should be treated as infrastructure commitments,
not convenience shortcuts.

---

# Summary

GSIs are:

- Powerful
- Expensive
- Eventually consistent
- Operationally impactful

Use them when necessary,
but never before defining a strict access pattern and query contract.
