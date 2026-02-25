# DynamoDB Foundations

---

## Why DynamoDB Is Different

DynamoDB is a **fully managed, distributed NoSQL key-value and document database**.
Unlike relational databases, it does not optimize for flexible querying or ad-hoc joins.

Instead, DynamoDB optimizes for:
- Predictable performance at scale
- Horizontal scalability
- High availability
- Simple operational overhead

---

## DynamoDB Is Not SQL Without Joins

A common mistake when adopting DynamoDB is attempting to design tables as if they were relational schemas:

- Multiple tables per entity
- Normalization (3NF)
- Expectation of joins
- Flexible filtering across attributes

In DynamoDB:
- **There are no joins**
- Queries are driven by **primary keys**
- Filters do not reduce read cost
- Table scans do not scale

Trying to replicate relational patterns usually results in:
- High read costs
- Inefficient queries
- Complex application logic
- Poor performance at scale

---

## Access Pattern–Driven Design

The most important concept in DynamoDB modeling is:

> **You do not design tables based on entities.  
> You design tables based on access patterns.**

### What is an access pattern?

An access pattern answers:
- What data do I need?
- How will I query it?
- How often?
- At what scale?

Examples:
- Get a student by studentId
- List all courses a student is enrolled in
- Retrieve the latest activity for a course
- Fetch all lessons for a course
- Query activity logs for a course within a time range

Only after access patterns are clearly defined does table design begin.

---

## Primary Keys as Query Contracts

In DynamoDB, the **partition key (PK)** and **sort key (SK)** form a **query contract**:

- Every efficient query must specify the partition key
- The sort key defines ordering and filtering within the partition
- Queries are predictable and fast when they align with this contract

This means:
- Attributes are not indexed unless explicitly designed
- Data shape is optimized for specific queries
- Flexibility is traded for performance and scalability

---

## Single-Table Design: The Core Idea

Single-table design is not a requirement, but a **design strategy**.

The idea:
- Store multiple entity types in the same table
- Share the same PK/SK namespace
- Co-locate related data in the same partition

Benefits:
- Fewer network calls
- Lower latency
- Lower cost
- Stronger data locality

Trade-offs:
- More upfront design effort
- Less intuitive than relational models
- Requires discipline and documentation

Single-table design works best when:
- Access patterns are well understood
- Relationships are known
- Read efficiency matters more than ad-hoc querying

---

## Secondary Indexes Are Not a Shortcut

Global Secondary Indexes (GSIs) are powerful but expensive.

Common anti-patterns:
- Creating GSIs to compensate for poor primary key design
- Indexing every attribute “just in case”
- Treating GSIs like relational indexes

Best practice:
- Design the primary keys first
- Add GSIs only for **explicit, unavoidable access patterns**
- Treat each GSI as a separate read model

---

## Cost and Performance Are Tightly Coupled

In DynamoDB:
- You pay for what you read and write
- Reading unnecessary attributes costs money
- Poor key design directly impacts cost

Well-designed tables:
- Minimize item size
- Minimize read amplification
- Reduce the need for GSIs
- Scale predictably under load

---

## What Comes Next

With these foundations in place, the next steps are:

1. Define a real-world domain
2. Enumerate concrete access patterns
3. Design a single-table schema
4. Validate it against those access patterns

The next document focuses on the **Single-Table Core Pattern** and introduces a shared domain used throughout this repository.

➡️ See: `dynamodb/01-single-table-design.md`