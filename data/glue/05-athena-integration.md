# Amazon Athena Integration with AWS Glue

## What is Amazon Athena?

Amazon Athena is a serverless interactive query service that allows you to analyze data stored in Amazon S3 using standard SQL.

Athena does not require you to provision or manage database servers.

A simple architecture looks like:

```text
Amazon S3
    |
    | Data
    v
Glue Data Catalog
    |
    | Metadata
    v
Amazon Athena
    |
    | SQL Query
    v
Query Results
```

Athena is commonly used together with:

- Amazon S3
- AWS Glue Data Catalog
- AWS Glue Crawlers
- Apache Parquet

---

## How Athena Works

Athena queries data directly from Amazon S3.

It does not move the data into a separate database before querying it.

Conceptually:

```text
Amazon S3
    |
    | Files
    v
Amazon Athena
    |
    | Read
    | Filter
    | Aggregate
    v
Results
```

Athena needs metadata to understand the structure of the data.

This metadata is commonly stored in the AWS Glue Data Catalog.

---

## Athena and the Glue Data Catalog

The Glue Data Catalog stores metadata such as:

- Database names
- Table names
- Column names
- Data types
- S3 locations
- File formats
- Partitions

Athena can use this metadata to query the underlying S3 data.

Example:

```text
Amazon S3
    |
    | transactions.parquet
    v
Glue Data Catalog
    |
    | Database: analytics
    | Table: transactions
    v
Amazon Athena
```

The Glue Data Catalog stores the definition of the dataset.

Amazon S3 stores the actual data.

Athena uses both.

---

## Glue Catalog vs S3 vs Athena

A useful mental model is:

```text
Amazon S3
    |
    | Stores the data
    v

Glue Data Catalog
    |
    | Describes the data
    v

Amazon Athena
    |
    | Queries the data
    v

SQL Results
```

In simple terms:

```text
S3 = Data

Glue Catalog = Metadata

Athena = Query Engine
```

---

## Example Dataset

Imagine we have Parquet files stored in:

```text
s3://my-data-bucket/transactions/
```

The Glue Data Catalog contains:

```text
Database: analytics

Table: transactions

Columns:
- transaction_id: string
- customer_id: string
- amount: double
- status: string
- year: int
- month: int

Format:
Parquet

Location:
s3://my-data-bucket/transactions/
```

Athena can now query the dataset.

Example:

```sql
SELECT *
FROM analytics.transactions;
```

---

## Simple Athena Queries

### Select All Data

```sql
SELECT *
FROM analytics.transactions;
```

---

### Select Specific Columns

```sql
SELECT
    transaction_id,
    customer_id,
    amount
FROM analytics.transactions;
```

Selecting only required columns is especially useful with Parquet.

---

### Filtering

```sql
SELECT *
FROM analytics.transactions
WHERE status = 'completed';
```

---

### Aggregation

```sql
SELECT
    status,
    COUNT(*) AS total
FROM analytics.transactions
GROUP BY status;
```

Possible result:

```text
completed | 1200
pending   | 300
failed    | 50
```

---

## Athena with Partitioned Data

Imagine the S3 dataset is organized as:

```text
transactions/
|
└── year=2026/
    |
    ├── month=08/
    |
    └── month=09/
```

The Glue Table contains partition columns:

```text
year
month
```

A query can filter by partition:

```sql
SELECT *
FROM analytics.transactions
WHERE year = 2026
AND month = 9;
```

Athena can scan only:

```text
year=2026/month=09/
```

instead of scanning the complete dataset.

---

## Partition Pruning

Partition pruning allows Athena to skip data that does not match the partition filters.

Example:

```sql
SELECT *
FROM transactions
WHERE year = 2026
AND month = 9;
```

Conceptually:

```text
All Data
   |
   v
Partition Filter
   |
   v
Only Required Partition
```

Without partition pruning:

```text
Scan Everything
```

With partition pruning:

```text
Scan Only Relevant Data
```

This can reduce:

- Query execution time
- Amount of data scanned
- Query cost

---

## Athena Pricing Model

Athena commonly charges based on the amount of data scanned by a query.

Conceptually:

```text
More Data Scanned
       |
       v
Higher Cost
```

and:

```text
Less Data Scanned
       |
       v
Lower Cost
```

Because of this, data organization matters.

Common optimizations include:

- Parquet
- Compression
- Partitioning
- Selecting only required columns
- Avoiding unnecessary `SELECT *`

---

## SELECT * vs Selecting Columns

Imagine a dataset contains:

```text
transaction_id
customer_id
amount
status
created_at
region
payment_method
device_type
metadata
```

This query:

```sql
SELECT *
FROM transactions;
```

may read all columns.

But this query:

```sql
SELECT
    transaction_id,
    amount
FROM transactions;
```

only needs two columns.

With Parquet, Athena can avoid reading unrelated columns.

A good practice is:

```text
Select only what you need
```

---

## Athena and Parquet

Athena works especially well with Parquet.

Example:

```text
JSON
 |
 v
Glue Job
 |
 | Convert
 v
Parquet
 |
 v
Amazon S3
 |
 v
Athena
```

Parquet can provide:

- Column pruning
- Compression
- Smaller scans
- Better query performance

This makes the combination common in analytics architectures.

---

## Query Results Location

Athena stores query results in Amazon S3.

For example:

```text
Athena Query
    |
    v
Query Result
    |
    v
Amazon S3
```

A result location might look like:

```text
s3://my-athena-results/
```

This location must be configured before running queries.

---

## Athena Does Not Store the Main Dataset

Athena is not a traditional database.

It does not normally store the source dataset itself.

Instead:

```text
Amazon S3
    |
    | Data
    v
Athena
```

Athena is the query engine.

The source data stays in S3.

---

## External Tables

Athena tables are often external tables.

An external table describes data stored outside Athena.

Example:

```sql
CREATE EXTERNAL TABLE transactions (
    transaction_id string,
    customer_id string,
    amount double,
    status string
)
STORED AS PARQUET
LOCATION 's3://my-data-bucket/transactions/';
```

This creates metadata describing the S3 dataset.

The actual files remain in S3.

---

## Do I Need a Glue Crawler?

No.

Athena can query a table created manually.

There are two common approaches.

### Using a Glue Crawler

```text
S3
 |
 v
Glue Crawler
 |
 v
Glue Data Catalog
 |
 v
Athena
```

### Defining the Table Manually

```text
S3
 |
 v
CREATE EXTERNAL TABLE
 |
 v
Glue Data Catalog
 |
 v
Athena
```

A crawler is useful for automatic schema discovery.

Manual definitions provide more control.

---

## Creating a Table with a Crawler

A common flow is:

```text
S3 Dataset
    |
    v
Glue Crawler
    |
    | Detect Schema
    v
Glue Table
    |
    v
Athena Query
```

Example:

```text
s3://my-data-bucket/transactions/
```

Crawler discovers:

```text
transaction_id string
customer_id string
amount double
status string
```

Then Athena can query:

```sql
SELECT *
FROM analytics.transactions;
```

---

## Athena and Schema Changes

Imagine the original schema is:

```text
transaction_id
amount
```

Later, the data changes to:

```text
transaction_id
amount
status
```

The Data Catalog may need to be updated before Athena can correctly use the new field.

This could happen through:

- Glue Crawler
- Manual table update
- Infrastructure as Code

Schema evolution should be controlled carefully.

---

## Partition Metadata

Having partition folders in S3 is not always enough.

The Glue Data Catalog also needs information about those partitions.

Example S3:

```text
year=2026/
month=09/
```

The Glue Catalog should know:

```text
year=2026
month=09
```

Then Athena can use the partition metadata.

---

## Discovering New Partitions

Imagine a new partition appears:

```text
year=2026/month=10/
```

The Data Catalog needs to learn about it.

One option is to run a Glue Crawler.

```text
New S3 Partition
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

## MSCK REPAIR TABLE

For Hive-style partitions, Athena can also discover partitions using:

```sql
MSCK REPAIR TABLE transactions;
```

This scans compatible S3 directory structures and attempts to add missing partitions.

Example structure:

```text
transactions/
|
└── year=2026/
    |
    └── month=09/
```

However, this is not always the best strategy for very large partitioned datasets.

---

## Partition Projection

Athena also supports partition projection.

Partition projection allows Athena to calculate partition locations instead of storing every partition explicitly in the catalog.

Conceptually:

```text
Traditional

S3 Partitions
     |
     v
Store Every Partition
     |
     v
Glue Data Catalog
```

With partition projection:

```text
Partition Rules
      |
      v
Athena Calculates
Partition Locations
```

This can be useful for datasets with many predictable partitions.

This is a more advanced optimization and is not required for basic Athena usage.

---

## Athena Views

Athena can create views.

A view stores a SQL query as a reusable logical object.

Example:

```sql
CREATE VIEW completed_transactions AS
SELECT
    transaction_id,
    customer_id,
    amount
FROM transactions
WHERE status = 'completed';
```

Then:

```sql
SELECT *
FROM completed_transactions;
```

Views can simplify common analytical queries.

---

## CTAS

CTAS stands for:

```text
CREATE TABLE AS SELECT
```

Athena can create a new dataset based on a query.

Example:

```sql
CREATE TABLE completed_transactions
WITH (
    format = 'PARQUET',
    external_location = 's3://my-data-bucket/completed/'
) AS
SELECT *
FROM transactions
WHERE status = 'completed';
```

Conceptually:

```text
Existing Dataset
       |
       v
Athena Query
       |
       v
New Dataset
       |
       v
Amazon S3
```

CTAS can be useful for:

- Data transformation
- Converting formats
- Creating optimized datasets
- Preparing analytical tables

---

## Glue Job vs Athena Transformation

Both Glue and Athena can transform data.

Example Glue:

```text
Source
 |
 v
Glue Job
 |
 | PySpark
 v
S3
```

Example Athena:

```text
S3
 |
 v
Athena SQL
 |
 | CTAS
 v
S3
```

Glue is usually better suited for:

- Complex transformations
- Large ETL pipelines
- Spark processing
- Complex joins
- Programmatic data processing

Athena is often useful for:

- SQL-based transformations
- Ad hoc queries
- Simple analytical transformations
- Exploration

---

## Athena vs RDS

Athena and RDS serve different purposes.

| Amazon Athena | Amazon RDS |
|---|---|
| Analytics | Transactional workloads |
| Queries S3 | Stores relational data |
| Serverless query engine | Managed database |
| Schema-on-read | Structured database schema |
| Large analytical scans | Low-latency application queries |
| Pay per data scanned | Database instance/storage pricing |

Example RDS:

```text
Application
    |
    v
RDS
    |
    v
Insert / Update / Delete
```

Example Athena:

```text
S3
 |
 v
Athena
 |
 v
Analyze millions of records
```

---

## Athena vs Glue

Athena and Glue also have different responsibilities.

| AWS Glue | Amazon Athena |
|---|---|
| Data integration | Data querying |
| ETL / ELT | SQL analytics |
| PySpark | SQL |
| Transform datasets | Query datasets |
| Catalog metadata | Uses catalog metadata |
| Batch pipelines | Interactive queries |

A common architecture uses both:

```text
Raw Data
    |
    v
Glue Job
    |
    v
Curated Data
    |
    v
Athena
```

---

## Common Athena Query Problems

### Too Much Data Scanned

Example:

```sql
SELECT *
FROM transactions;
```

on a very large dataset.

Possible improvements:

- Select only required columns
- Use Parquet
- Filter partitions
- Compress data

---

## Missing Partition

A query may not return recently added data if the new partition is not registered.

Example:

```text
S3:
year=2026/month=10/
```

but Data Catalog only knows:

```text
year=2026/month=09/
```

Athena may not query the new partition until the metadata is updated.

---

## Schema Mismatch

Example:

```text
Catalog:
amount -> double

Actual Data:
amount -> string
```

This can cause query errors.

The schema in the catalog must be compatible with the stored data.

---

## Wrong S3 Location

If a Glue Table points to:

```text
s3://wrong-bucket/transactions/
```

Athena will query the wrong location or return no records.

Always verify the table location.

---

## Permissions

Athena requires access to several resources.

Depending on the architecture, permissions may include:

- Read access to source S3 data
- Write access to Athena result location
- Access to the Glue Data Catalog
- KMS permissions for encrypted data

Conceptually:

```text
Athena
 |
 +--> Read Source S3
 |
 +--> Read Glue Catalog
 |
 +--> Write Query Results
```

---

## Athena Query Optimization

A good Athena query strategy usually follows:

```text
Read Less Data
      |
      v
Faster Query
      +
Lower Cost
```

Ways to achieve this include:

- Use Parquet
- Use compression
- Partition data
- Filter partition columns
- Select only required columns
- Avoid unnecessary full scans
- Keep reasonable file sizes

---

## Example Optimized Query

Less optimized:

```sql
SELECT *
FROM transactions;
```

More optimized:

```sql
SELECT
    transaction_id,
    amount
FROM transactions
WHERE year = 2026
AND month = 9
AND status = 'completed';
```

This benefits from:

```text
Column Pruning
      +
Partition Pruning
```

---

## Typical Glue + Athena Pipeline

A common architecture is:

```text
Source System
     |
     v
Raw S3
     |
     | JSON / CSV
     v
AWS Glue Job
     |
     | Clean
     | Transform
     | Convert
     v
Curated S3
     |
     | Parquet
     | Partitioned
     v
Glue Data Catalog
     |
     v
Amazon Athena
     |
     | SQL
     v
Analytics
```

---

## Athena Mental Model

A useful way to think about Athena is:

```text
Where is my data?
        |
        v
       S3

What does it look like?
        |
        v
Glue Data Catalog

What do I want to know?
        |
        v
      SQL

Who executes the query?
        |
        v
     Athena
```

Or simply:

```text
S3 + Metadata + SQL
         |
         v
       Athena
```

---

## Best Practices

When using Athena:

- Store analytical data in Parquet when appropriate.
- Compress datasets.
- Partition large datasets.
- Filter using partition columns.
- Avoid unnecessary `SELECT *`.
- Select only required columns.
- Store query results in a dedicated S3 location.
- Keep the Glue Data Catalog schema accurate.
- Monitor schema evolution.
- Avoid excessive small files.
- Use CTAS when SQL-based dataset transformation is appropriate.
- Use Glue Jobs for more complex ETL processing.
- Follow least-privilege IAM permissions.

---

## Key Takeaways

- Amazon Athena is a serverless SQL query service.
- Athena queries data directly from Amazon S3.
- Athena commonly uses the AWS Glue Data Catalog for metadata.
- S3 stores the actual data.
- The Glue Data Catalog describes the data.
- Athena executes SQL queries over the data.
- Athena works especially well with Parquet.
- Partition pruning can reduce the amount of data scanned.
- Selecting only required columns can reduce scans.
- Athena query cost is strongly related to the amount of data scanned.
- Crawlers are useful but not mandatory.
- Athena tables can be defined manually.
- Athena query results are stored in S3.
- CTAS can create new datasets using SQL.
- Glue and Athena are complementary services.

---

## Interview Questions

### What is Amazon Athena?

Amazon Athena is a serverless interactive query service used to analyze data stored in Amazon S3 using SQL.

### Does Athena store the source data?

No.

The source data remains in Amazon S3.

### What is the relationship between Athena and the Glue Data Catalog?

Athena can use the Glue Data Catalog to obtain metadata such as table schemas, locations, formats, and partitions.

### Why is Parquet good for Athena?

Parquet is columnar and compressed, allowing Athena to read only required columns and potentially scan less data.

### How is Athena priced?

Athena is commonly priced based on the amount of data scanned by queries.

### How can you reduce Athena query cost?

Common techniques include using Parquet, compression, partitioning, filtering partitions, and selecting only required columns.

### What is partition pruning?

Partition pruning allows Athena to skip partitions that do not match the query filters.

### Does Athena require Glue Crawlers?

No.

Tables can also be defined manually or through infrastructure as code.

### Where does Athena store query results?

Athena stores query results in an Amazon S3 location.

### What is CTAS?

CTAS means `CREATE TABLE AS SELECT`.

It allows Athena to create a new table and dataset using the result of a SQL query.

### What is the difference between Athena and RDS?

RDS is designed primarily for relational transactional workloads.

Athena is designed for analytical SQL queries over data stored in S3.

### What is the difference between Athena and Glue?

Glue is mainly used for data integration and ETL processing.

Athena is mainly used to query analytical datasets using SQL.