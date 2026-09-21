# Apache Parquet and Partitioning

## What is Apache Parquet?

Apache Parquet is a columnar storage file format commonly used for analytical workloads.

It is designed to store large datasets efficiently and works very well with services such as:

- AWS Glue
- Amazon Athena
- Amazon EMR
- Amazon Redshift Spectrum
- Apache Spark

A typical flow looks like:

```text
Raw Data
   |
   | JSON / CSV
   v
AWS Glue Job
   |
   | Transform
   v
Parquet
   |
   v
Amazon S3
```

---

## Row-Based vs Columnar Storage

Traditional formats such as CSV and JSON are generally row-oriented.

Example:

```text
id | customer | amount | status
1  | Alice    | 100    | completed
2  | Bob      | 200    | pending
3  | Carol    | 300    | completed
```

Conceptually, row-based storage organizes data like:

```text
Row 1 -> id, customer, amount, status
Row 2 -> id, customer, amount, status
Row 3 -> id, customer, amount, status
```

Parquet stores data by column.

Conceptually:

```text
id
|
1
2
3

customer
|
Alice
Bob
Carol

amount
|
100
200
300

status
|
completed
pending
completed
```

This is called columnar storage.

---

## Why Columnar Storage Matters

Analytical queries often need only a few columns.

For example:

```sql
SELECT customer, amount
FROM transactions;
```

If the dataset contains:

```text
id
customer
amount
status
created_at
updated_at
region
payment_method
device_type
```

The query only needs:

```text
customer
amount
```

With a columnar format such as Parquet, the query engine can avoid reading unnecessary columns.

Conceptually:

```text
CSV / JSON

Read complete rows
        |
        v
Ignore unused columns
```

With Parquet:

```text
Parquet

Read only required columns
        |
        v
Less data scanned
```

This can improve:

- Query performance
- Storage efficiency
- Processing time
- Query cost

---

## CSV vs JSON vs Parquet

| CSV | JSON | Parquet |
|---|---|---|
| Row-based | Row-based | Columnar |
| Human-readable | Human-readable | Binary |
| Simple structure | Supports nested data | Optimized for analytics |
| Larger scans | Larger scans | Reads selected columns |
| Limited schema support | Flexible schema | Strong schema support |
| Common for exchange | Common for APIs/events | Common for analytics |

---

## Why Use Parquet with AWS Glue?

AWS Glue frequently processes raw datasets and writes the result as Parquet.

Example:

```text
Amazon S3
    |
    | JSON
    v
Glue Job
    |
    | Clean
    | Filter
    | Cast
    v
Amazon S3
    |
    | Parquet
    v
Amazon Athena
```

This is useful because Parquet is optimized for analytical queries.

---

## Example: JSON to Parquet

Imagine this JSON record:

```json
{
  "transaction_id": "tx-001",
  "customer_id": "customer-100",
  "amount": 125.50,
  "status": "completed"
}
```

A Glue Job could read JSON:

```python
df = spark.read.json(
    "s3://my-data-bucket/raw/"
)
```

Then write Parquet:

```python
df.write.parquet(
    "s3://my-data-bucket/curated/"
)
```

The pipeline becomes:

```text
JSON
 |
 v
Glue Job
 |
 v
Parquet
```

---

## Compression

Parquet supports compression.

Common compression codecs include:

- Snappy
- GZIP
- ZSTD

Compression reduces the amount of storage required.

Conceptually:

```text
Uncompressed Data
      |
      v
Compression
      |
      v
Smaller Files
```

Smaller files can reduce:

- S3 storage usage
- Network transfer
- Data scanned by analytical services

Snappy is commonly used because it provides a good balance between compression and speed.

---

## Parquet and Athena

Amazon Athena can query Parquet files stored in S3.

Example:

```text
Amazon S3
    |
    | Parquet
    v
Glue Data Catalog
    |
    v
Amazon Athena
```

Query:

```sql
SELECT transaction_id, amount
FROM transactions
WHERE status = 'completed';
```

Athena can benefit from Parquet because it can read only the required columns.

---

## What is Partitioning?

Partitioning divides a large dataset into smaller logical groups.

Instead of storing everything in one location:

```text
s3://my-data-bucket/transactions/
```

We can organize the dataset like:

```text
s3://my-data-bucket/transactions/

├── year=2025/
├── year=2026/
└── year=2027/
```

Or with multiple partition keys:

```text
s3://my-data-bucket/transactions/

└── year=2026/
    ├── month=08/
    └── month=09/
```

---

## Why Partition Data?

Imagine we have five years of transaction data.

Without partitioning:

```text
All Transactions
       |
       v
Athena scans everything
```

But a query might only need September 2026:

```sql
SELECT *
FROM transactions
WHERE year = 2026
AND month = 9;
```

With partitions:

```text
transactions/
|
├── year=2025/
|
├── year=2026/
|   |
|   ├── month=08/
|   |
|   └── month=09/   <-- Read
|
└── year=2027/
```

Athena can avoid scanning unrelated partitions.

---

## Partition Pruning

Partition pruning means that the query engine skips partitions that are not required.

For example:

```sql
SELECT *
FROM transactions
WHERE year = 2026;
```

Athena can scan:

```text
year=2026/
```

and ignore:

```text
year=2025/
year=2027/
```

Conceptually:

```text
All Partitions
      |
      v
Partition Filter
      |
      v
Required Partitions Only
```

This can significantly reduce the amount of data scanned.

---

## Writing Partitioned Data with Glue

Spark can write partitioned datasets.

Example:

```python
df.write \
    .partitionBy("year", "month") \
    .parquet(
        "s3://my-data-bucket/transactions/"
    )
```

The generated S3 structure could look like:

```text
transactions/
|
├── year=2026/
│   |
│   ├── month=08/
│   │   ├── part-00000.parquet
│   │   └── part-00001.parquet
│   |
│   └── month=09/
│       ├── part-00000.parquet
│       └── part-00001.parquet
```

---

## Partition Columns

Partition columns should usually represent values commonly used in query filters.

Good examples include:

- Year
- Month
- Day
- Region
- Country
- Environment

For time-based datasets, a common structure is:

```text
year
month
day
```

Example:

```text
year=2026/
month=09/
day=06/
```

---

## Choosing a Good Partition Key

A good partition key should:

- Be commonly used in query filters
- Have a reasonable number of unique values
- Divide data into useful groups
- Avoid extremely small partitions

Example of a good partition key:

```text
transaction_date
```

or:

```text
year / month
```

because analytics queries commonly filter by date.

---

## Bad Partition Keys

Some columns are usually poor partition keys.

For example:

```text
transaction_id
```

If every transaction ID is unique:

```text
transaction_id=tx-001/
transaction_id=tx-002/
transaction_id=tx-003/
...
```

This creates too many partitions.

This is called over-partitioning.

---

## Over-Partitioning

Over-partitioning happens when a dataset is divided into too many small partitions.

Example:

```text
transactions/
|
├── customer_id=customer-001/
├── customer_id=customer-002/
├── customer_id=customer-003/
├── customer_id=customer-004/
├── customer_id=customer-005/
└── ...
```

If there are millions of customers, this can create millions of partitions.

This can cause:

- Metadata overhead
- Slower query planning
- More catalog management
- Small files
- Poor performance

---

## Under-Partitioning

The opposite problem can also happen.

Imagine a large dataset partitioned only by:

```text
year
```

If each year contains multiple terabytes:

```text
year=2026/
```

the partition may still be too large.

A better option could be:

```text
year=2026/
month=09/
day=06/
```

The correct partition strategy depends on the dataset size and query patterns.

---

## Partition Granularity

Partitioning should balance:

```text
Too Few Partitions
      |
      v
Large scans
```

and:

```text
Too Many Partitions
      |
      v
Metadata overhead
```

A good strategy is somewhere in the middle.

---

## The Small Files Problem

Distributed processing systems such as Spark can generate many small files.

Example:

```text
partition/
|
├── part-00001.parquet  10 KB
├── part-00002.parquet  8 KB
├── part-00003.parquet  12 KB
├── part-00004.parquet  9 KB
└── part-00005.parquet  11 KB
```

Having thousands or millions of very small files can reduce query performance.

Why?

Because systems such as Athena and Spark must:

- Open many files
- Read file metadata
- Make many S3 requests
- Schedule more tasks

---

## Better File Sizes

Instead of:

```text
10,000 files
x
10 KB
```

it is generally more efficient to have fewer, larger files.

Conceptually:

```text
Many Tiny Files
      |
      v
Compaction
      |
      v
Fewer Larger Files
```

The ideal file size depends on the workload, but avoiding extremely small files is important.

---

## Controlling Output Files in Spark

Spark provides operations such as:

```python
df.coalesce(4)
```

or:

```python
df.repartition(4)
```

These can affect how many output files are generated.

Example:

```python
df.coalesce(4) \
  .write \
  .parquet("s3://my-data-bucket/output/")
```

Conceptually:

```text
Many Spark Partitions
        |
        v
     coalesce
        |
        v
Fewer Output Files
```

These operations should be used carefully because they can affect performance.

---

## Repartition vs Coalesce

Both operations can change the number of Spark partitions.

### Repartition

```python
df.repartition(10)
```

Can increase or decrease the number of partitions.

It usually causes a shuffle.

### Coalesce

```python
df.coalesce(4)
```

Is commonly used to reduce the number of partitions.

It can avoid a full shuffle in some cases.

A simple mental model is:

```text
repartition
    |
    v
Redistribute data
```

```text
coalesce
    |
    v
Reduce partitions
```

---

## Parquet Schema

Parquet stores schema information inside the file.

For example:

```text
transaction_id -> string
amount         -> double
status         -> string
created_at     -> timestamp
```

This is different from formats such as CSV, which do not inherently store strong type information.

This makes Parquet useful for analytical pipelines where consistent schemas are important.

---

## Nested Data

Parquet can also support nested structures.

Example:

```json
{
  "customer_id": "customer-100",
  "address": {
    "city": "Austin",
    "state": "TX"
  }
}
```

Parquet can preserve nested structures more efficiently than flat formats such as CSV.

---

## Schema Evolution with Parquet

Datasets may evolve over time.

Version 1:

```text
transaction_id
amount
```

Version 2:

```text
transaction_id
amount
status
```

Parquet and Spark can support some types of schema evolution.

However, schema changes should still be managed carefully.

Unexpected changes can affect:

- Glue Jobs
- Athena queries
- Data Catalog tables
- Downstream applications

---

## Raw vs Curated Data

A common data lake structure is:

```text
Amazon S3
|
├── raw/
|
└── curated/
```

The raw layer contains data close to the original source.

Example:

```text
raw/
|
└── transactions/
    |
    └── JSON
```

The curated layer contains cleaned and optimized data.

Example:

```text
curated/
|
└── transactions/
    |
    └── Parquet
```

Typical flow:

```text
Source
  |
  v
Raw S3
  |
  v
Glue Job
  |
  | Clean
  | Validate
  | Transform
  v
Curated S3
  |
  v
Parquet
```

---

## Example Data Lake Layout

A simple S3 structure could look like:

```text
my-data-bucket/
|
├── raw/
│   |
│   └── transactions/
│       |
│       └── json/
|
└── curated/
    |
    └── transactions/
        |
        └── year=2026/
            |
            ├── month=08/
            |
            └── month=09/
```

This separates raw data from analytics-ready data.

---

## Glue Crawler and Parquet

After a Glue Job writes Parquet data:

```text
Glue Job
   |
   v
S3 Parquet
```

A crawler can scan the output:

```text
S3 Parquet
    |
    v
Glue Crawler
    |
    v
Glue Data Catalog
```

Then Athena can query it:

```text
Glue Data Catalog
    |
    v
Amazon Athena
```

Full flow:

```text
Raw Data
   |
   v
Glue Job
   |
   v
Parquet
   |
   v
Glue Crawler
   |
   v
Data Catalog
   |
   v
Athena
```

---

## Partitioning and the Glue Data Catalog

Partitions can also be stored as metadata in the Glue Data Catalog.

For example:

```text
Table: transactions

Partitions:

year=2026/month=08
year=2026/month=09
```

Athena uses this metadata to determine which S3 locations correspond to each partition.

---

## Partition Discovery

If new folders appear:

```text
year=2026/month=10/
```

the Glue Data Catalog needs to know about the new partition.

A crawler can discover it.

Conceptually:

```text
New S3 Partition
       |
       v
Glue Crawler
       |
       v
Data Catalog Updated
```

There are also other approaches for managing partitions, but crawlers are a common option.

---

## Good Partitioning Example

Imagine analytics queries usually filter by date:

```sql
SELECT *
FROM transactions
WHERE year = 2026
AND month = 9;
```

A good structure could be:

```text
transactions/
|
└── year=2026/
    |
    └── month=09/
```

Because the physical layout matches the query pattern.

---

## Poor Partitioning Example

Imagine we partition by transaction ID:

```text
transactions/
|
├── transaction_id=tx-001/
├── transaction_id=tx-002/
├── transaction_id=tx-003/
└── ...
```

If millions of transactions exist, this creates millions of partitions.

That is usually inefficient.

---

## Partitioning Mental Model

A useful way to understand partitioning is:

```text
Large Dataset
     |
     v
Divide by Common Query Filter
     |
     v
Partitions
     |
     v
Query Only What You Need
```

For example:

```text
Large Dataset
     |
     v
Partition by Date
     |
     v
year / month / day
     |
     v
Athena scans selected dates
```

---

## Parquet Mental Model

A simple mental model for Parquet is:

```text
Row-Based Data

Read:
id + name + amount + status + date
```

vs:

```text
Columnar Data

Query:
SELECT amount

Read:
amount only
```

The goal is:

```text
Read Less Data
      |
      v
Faster Analytics
      +
Lower Query Cost
```

---

## Typical Analytics Pipeline

A common architecture is:

```text
Source System
     |
     v
Raw Data
     |
     | JSON / CSV
     v
Amazon S3
     |
     v
AWS Glue Job
     |
     | Clean
     | Transform
     | Partition
     | Convert
     v
Amazon S3
     |
     | Parquet
     v
Glue Data Catalog
     |
     v
Amazon Athena
```

---

## Best Practices

When using Parquet and partitions:

- Prefer Parquet for analytical datasets.
- Use compression where appropriate.
- Partition using columns commonly used in filters.
- Avoid partitioning by high-cardinality unique IDs.
- Avoid creating too many tiny partitions.
- Avoid producing excessive small files.
- Keep raw and curated data separated.
- Filter data as early as possible.
- Monitor query patterns before choosing partition keys.
- Use the Glue Data Catalog to manage schema and partition metadata.
- Review schema evolution carefully.

---

## Key Takeaways

- Parquet is a columnar storage format.
- Columnar storage is well suited for analytics.
- Parquet can reduce the amount of data scanned.
- Parquet supports compression.
- Parquet stores schema information.
- AWS Glue can convert JSON and CSV data into Parquet.
- Partitioning divides datasets into logical groups.
- Athena can use partition pruning to avoid unnecessary scans.
- Good partition keys are commonly used in query filters.
- High-cardinality columns are usually poor partition keys.
- Too many partitions can create metadata overhead.
- Too many small files can reduce performance.
- Raw and curated S3 layers are common in data lake architectures.
- Glue Crawlers can discover Parquet schemas and partitions.

---

## Interview Questions

### What is Apache Parquet?

Apache Parquet is a columnar file format optimized for analytical workloads.

### Why is Parquet commonly used with AWS Glue?

Because it provides efficient storage, compression, schema support, and allows analytical engines to read only the required columns.

### What is the difference between CSV and Parquet?

CSV is row-based and text-based.

Parquet is columnar, binary, compressed, and optimized for analytical queries.

### Why can Parquet reduce Athena query cost?

Athena charges based on the amount of data scanned.

Parquet can allow Athena to read only the required columns instead of complete rows.

### What is partitioning?

Partitioning divides a dataset into smaller logical groups based on specific columns.

### What is partition pruning?

Partition pruning means that a query engine skips partitions that do not match the query filters.

### What makes a good partition key?

A good partition key is commonly used in query filters and has a reasonable number of distinct values.

### Why is a unique ID usually a bad partition key?

Because it can create a very large number of tiny partitions, causing metadata and performance problems.

### What is over-partitioning?

Over-partitioning happens when a dataset is divided into too many small partitions.

### What is the small files problem?

It occurs when a dataset contains a very large number of small files, increasing metadata operations, S3 requests, and query overhead.

### What is the difference between `repartition()` and `coalesce()`?

`repartition()` redistributes data and can increase or decrease the number of Spark partitions.

`coalesce()` is generally used to reduce the number of partitions with less data movement.

### Why separate raw and curated data?

The raw layer preserves source data, while the curated layer contains cleaned and optimized datasets ready for analytics.