# AWS Glue Data Catalog and Crawlers

## What is the AWS Glue Data Catalog?

The AWS Glue Data Catalog is a centralized metadata repository used to store information about datasets.

It does not store the actual data.

Instead, it stores metadata such as:

- Database names
- Table names
- Column names
- Data types
- File formats
- Data locations
- Partitions
- Serialization information

A simple way to think about it is:

```text
Amazon S3
    |
    | Actual data
    v
Files

AWS Glue Data Catalog
    |
    | Metadata
    v
Schema + Location + Format
```

For example:

```text
S3 location:
s3://my-data-bucket/transactions/
```

The Glue Data Catalog could store:

```text
Database: analytics
Table: transactions
Format: Parquet
Location: s3://my-data-bucket/transactions/
```

---

## Glue Database

A Glue Database is a logical container for tables.

It does not represent a traditional relational database.

It is mainly used to organize metadata inside the Glue Data Catalog.

Example:

```text
Glue Data Catalog
|
└── Database: analytics
    |
    ├── transactions
    ├── customers
    └── orders
```

A Glue Database can contain multiple Glue Tables.

---

## Glue Table

A Glue Table represents the metadata definition of a dataset.

A table can describe data stored in locations such as:

- Amazon S3
- Amazon RDS
- JDBC-compatible databases
- Other supported data sources

For an S3 dataset, a Glue Table can contain:

```text
Table: transactions

Columns:
- transaction_id: string
- customer_id: string
- amount: double
- transaction_date: date

Location:
s3://my-data-bucket/transactions/

Format:
Parquet
```

The actual records remain in S3.

The Glue Table only describes how to interpret them.

---

## Schema

A schema describes the structure of a dataset.

For example:

```json
{
  "transaction_id": "tx-001",
  "customer_id": "customer-100",
  "amount": 125.50,
  "status": "completed"
}
```

A detected schema could look like:

```text
transaction_id -> string
customer_id    -> string
amount         -> double
status         -> string
```

This schema can be stored in the Glue Data Catalog.

---

## What is a Glue Crawler?

A Glue Crawler scans a data source and attempts to automatically determine its schema.

It can then create or update metadata inside the Glue Data Catalog.

Typical flow:

```text
Amazon S3
    |
    v
Glue Crawler
    |
    | Detect schema
    v
Glue Data Catalog
    |
    v
Database + Table
```

A crawler can detect information such as:

- Column names
- Data types
- File formats
- Partitions
- Dataset structure

---

## Example: Crawling JSON Data in S3

Imagine the following objects:

```text
s3://my-data-bucket/transactions/

├── transaction-001.json
├── transaction-002.json
└── transaction-003.json
```

Each file contains:

```json
{
  "transaction_id": "tx-001",
  "customer_id": "customer-100",
  "amount": 125.50,
  "status": "completed"
}
```

A Glue Crawler scans the S3 path:

```text
s3://my-data-bucket/transactions/
```

It detects the structure:

```text
transaction_id -> string
customer_id    -> string
amount         -> double
status         -> string
```

Then it creates a table in the Data Catalog:

```text
Database: analytics

Table: transactions

Location:
s3://my-data-bucket/transactions/
```

---

## What Happens When a Crawler Runs?

A simplified crawler execution looks like:

```text
1. Read the configured data source
        |
        v
2. Identify the data format
        |
        v
3. Detect the schema
        |
        v
4. Detect partitions
        |
        v
5. Compare with existing metadata
        |
        v
6. Create or update Glue Tables
```

The crawler does not transform the actual data.

Its main responsibility is metadata discovery.

---

## Crawler vs Glue Job

A Crawler and a Glue Job have different responsibilities.

| Glue Crawler | Glue Job |
|---|---|
| Discovers data | Processes data |
| Detects schemas | Transforms records |
| Creates metadata | Filters and joins data |
| Updates Data Catalog | Writes output datasets |
| Does not transform data | Performs ETL/ELT |

A simple mental model:

```text
Crawler = Understand the data

Job = Transform the data
```

---

## Classifiers

A classifier tells a Glue Crawler how to recognize and interpret a dataset.

AWS Glue includes built-in classifiers for common formats such as:

- JSON
- CSV
- Parquet
- Avro
- XML

Example:

```text
S3 File
   |
   v
Classifier
   |
   | Identify format
   v
Crawler
   |
   v
Schema
```

If the built-in classifiers are not enough, custom classifiers can also be created.

---

## Example: CSV Dataset

Imagine the following CSV file:

```csv
transaction_id,customer_id,amount,status
tx-001,customer-100,125.50,completed
tx-002,customer-101,80.00,pending
```

A Glue Crawler can detect:

```text
transaction_id -> string
customer_id    -> string
amount         -> double
status         -> string
```

However, schema inference is not always perfect.

For example, if the first records contain only integer values:

```csv
amount
100
200
300
```

Glue could infer:

```text
amount -> bigint
```

But later records could contain:

```text
125.50
```

This can create schema inconsistencies.

Because of this, schema discovery should be reviewed instead of blindly trusted.

---

## Schema Evolution

Datasets can change over time.

For example, an initial dataset may contain:

```json
{
  "transaction_id": "tx-001",
  "amount": 100
}
```

Later, a new field could be added:

```json
{
  "transaction_id": "tx-002",
  "amount": 150,
  "status": "completed"
}
```

This is called schema evolution.

A crawler can detect the new field and update the Glue Table.

Example:

```text
Before

transaction_id -> string
amount         -> bigint
```

After a new crawler execution:

```text
transaction_id -> string
amount         -> bigint
status         -> string
```

The behavior depends on the crawler configuration.

---

## Crawler Schema Change Policies

When a crawler detects changes, it can update the Data Catalog.

Typical changes include:

- New columns
- Removed columns
- Changed data types
- New partitions

Crawler configuration determines how these changes should be handled.

In production environments, automatic schema changes should be used carefully.

Unexpected changes in source data can affect:

- Glue Jobs
- Athena queries
- Downstream analytics
- Data quality

---

## Partitions

Partitions allow a large dataset to be divided into smaller logical groups.

Example:

```text
s3://my-data-bucket/transactions/

├── year=2026/
│   ├── month=08/
│   └── month=09/
```

The Glue Data Catalog can store these values as partitions:

```text
year
month
```

A crawler can automatically discover these partition folders.

Example:

```text
S3
|
├── year=2026/month=08/
└── year=2026/month=09/
        |
        v
    Glue Crawler
        |
        v
 Glue Data Catalog
        |
        v
Partitions discovered
```

---

## Why Are Partitions Important?

Without partitions, Athena may need to scan an entire dataset.

For example:

```sql
SELECT *
FROM transactions
WHERE year = 2026
AND month = 9;
```

If the dataset is partitioned by `year` and `month`, Athena can scan only:

```text
year=2026/month=09/
```

Instead of:

```text
All transaction data
```

This can improve:

- Query performance
- Query cost
- Scalability

---

## Data Catalog and Athena

Amazon Athena can use the Glue Data Catalog as its metadata store.

The relationship looks like:

```text
Amazon S3
    |
    | Actual data
    v

Glue Data Catalog
    |
    | Schema
    | Table
    | Partitions
    v

Amazon Athena
    |
    | SQL
    v

Query Results
```

Athena needs to know:

- Where the data is stored
- What format it uses
- What columns exist
- What partitions exist

The Glue Data Catalog provides this information.

---

## Does Athena Require a Crawler?

No.

A crawler is not mandatory.

A table can also be created manually.

For example, Athena can create an external table:

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

This means there are two common approaches:

```text
Option 1

S3
 |
 v
Crawler
 |
 v
Glue Table
 |
 v
Athena
```

or:

```text
Option 2

S3
 |
 v
Manual Table Definition
 |
 v
Glue Data Catalog
 |
 v
Athena
```

Crawlers are useful when automatic schema discovery is valuable.

Manual table definitions provide more control.

---

## When Should I Use a Crawler?

A crawler can be useful when:

- Exploring a new dataset
- The schema is unknown
- Data is coming from multiple sources
- Partitions are created dynamically
- You want automatic metadata discovery

A crawler may not always be necessary when:

- The schema is already known
- Infrastructure is managed as code
- Strong schema control is required
- The dataset structure rarely changes

In those cases, defining Glue Tables explicitly can provide more predictable behavior.

---

## Crawler Mental Model

A useful mental model is:

```text
Data exists somewhere
        |
        v
Crawler inspects it
        |
        v
Classifier identifies it
        |
        v
Schema is discovered
        |
        v
Metadata is stored
        |
        v
Glue Data Catalog
```

And then:

```text
Glue Data Catalog
        |
        +--> Glue Jobs
        |
        +--> Athena
        |
        +--> Redshift Spectrum
        |
        +--> Other analytics services
```

---

## Key Takeaways

- The Glue Data Catalog stores metadata, not the actual data.
- A Glue Database is a logical container for Glue Tables.
- A Glue Table describes the schema and location of a dataset.
- A Glue Crawler discovers schemas automatically.
- Crawlers can create and update Glue Tables.
- Classifiers help crawlers identify data formats.
- Crawlers do not transform data.
- Glue Jobs are responsible for transformations.
- Crawlers can discover partitions.
- Partitions can reduce the amount of data scanned by Athena.
- Athena can use the Glue Data Catalog to query S3 data.
- Crawlers are useful, but they are not mandatory.
- Manually defined schemas can provide more control in production systems.

---

## Interview Questions

### What is the AWS Glue Data Catalog?

The AWS Glue Data Catalog is a centralized metadata repository that stores information about datasets, including schemas, locations, formats, and partitions.

### Does the Glue Data Catalog store the actual data?

No.

It stores metadata.

The actual data can remain in systems such as Amazon S3 or Amazon RDS.

### What is a Glue Crawler?

A Glue Crawler scans a data source, detects its schema, and can create or update tables in the Glue Data Catalog.

### What is the difference between a Crawler and a Glue Job?

A Crawler discovers and catalogs data.

A Glue Job processes and transforms data.

### What is a classifier?

A classifier helps a crawler identify the format and structure of a dataset.

### Do I always need a crawler?

No.

If the schema is already known, tables can be defined manually or through infrastructure as code.

### What is schema evolution?

Schema evolution is when the structure of a dataset changes over time, for example when new columns are added.

### Why are partitions important?

Partitions allow query engines such as Athena to scan only relevant portions of a dataset, which can improve performance and reduce cost.