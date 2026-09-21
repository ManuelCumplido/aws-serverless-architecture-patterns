# AWS Glue Foundations

## What is AWS Glue?

AWS Glue is a serverless data integration service used to discover, prepare, transform, and move data between different data sources.

It is commonly used to build ETL and ELT pipelines without managing servers.

A typical architecture looks like:

```text
Data Source
    |
    v
AWS Glue
    |
    +--> Data Catalog
    |
    +--> ETL Job
    |
    v
Amazon S3
    |
    v
Amazon Athena
```

---

## ETL vs ELT

### ETL

ETL stands for:

```text
Extract
   |
Transform
   |
Load
```

In an ETL process, data is extracted from a source, transformed, and then loaded into the destination.

Example:

```text
Amazon RDS
    |
    v
AWS Glue
    |
    | Transform
    v
Amazon S3
```

The transformation happens before the data reaches the final destination.

### ELT

ELT stands for:

```text
Extract
   |
Load
   |
Transform
```

In an ELT process, raw data is loaded first and transformed later.

Example:

```text
Amazon RDS
    |
    v
Amazon S3
    |
    v
Athena / Redshift
    |
    v
Transformation
```

---

## Core AWS Glue Components

AWS Glue is composed of several components that work together.

```text
AWS Glue
│
├── Data Catalog
├── Crawlers
├── Jobs
├── Connections
└── Triggers / Workflows
```

### Data Catalog

The AWS Glue Data Catalog is a centralized metadata repository.

It stores information about datasets such as:

- Table names
- Column names
- Data types
- File formats
- Data locations
- Partitions

The Data Catalog stores metadata, not the actual data.

Example:

```text
Database: analytics
Table: transactions
Location: s3://my-bucket/transactions/
Format: Parquet
```

### Crawlers

Glue Crawlers scan data sources and try to automatically detect their schema.

Example:

```text
Amazon S3
    |
    v
Glue Crawler
    |
    v
Glue Data Catalog
```

A crawler can detect:

- Columns
- Data types
- File formats
- Partitions

Crawlers are mainly used for schema discovery.

### Glue Jobs

Glue Jobs execute data transformations.

They are commonly implemented using Apache Spark and PySpark.

Typical transformations include:

- Filtering
- Selecting columns
- Renaming columns
- Joining datasets
- Aggregating data
- Cleaning invalid records
- Converting data formats

Example:

```text
Amazon RDS
    |
    v
Glue Job
    |
    | Filter
    | Join
    | Transform
    | Aggregate
    v
Amazon S3
```

### Connections

Glue Connections allow AWS Glue to connect to external data sources.

For example:

```text
AWS Glue
    |
    | JDBC
    v
Amazon RDS PostgreSQL
```

A connection can contain configuration such as:

- JDBC URL
- Database information
- Credentials
- VPC
- Subnet
- Security Group

---

## Glue Mental Model

A simple way to think about AWS Glue is:

```text
Where is my data?
        |
        v
    Data Source
        |
        v
How do I describe it?
        |
        v
Data Catalog / Crawler
        |
        v
How do I transform it?
        |
        v
     Glue Job
        |
        v
Where do I store the result?
        |
        v
       S3
        |
        v
How do I query it?
        |
        v
     Athena
```

## When Should I Use AWS Glue?

AWS Glue is a good option when:

- Building ETL pipelines
- Processing large datasets
- Transforming data stored in Amazon S3
- Reading data from relational databases
- Converting JSON or CSV into Parquet
- Joining multiple datasets
- Preparing data for analytics
- Integrating with Athena or Redshift

Glue may not be the best option for:

- Low-latency APIs
- Simple event processing
- Very small transformations
- Request/response application logic

For those scenarios, AWS Lambda may be more appropriate.

---

## Glue vs Lambda

| AWS Glue | AWS Lambda |
|---|---|
| Data engineering | Application logic |
| ETL / ELT | Event-driven processing |
| Large datasets | Small payloads |
| Apache Spark / PySpark | Python / Node.js / Java |
| Batch processing | Short-running execution |
| Analytics pipelines | APIs and events |

---

## Key Takeaways

- AWS Glue is a serverless data integration service.
- Glue is commonly used to build ETL and ELT pipelines.
- The Glue Data Catalog stores metadata about datasets.
- Glue Crawlers discover schemas.
- Glue Jobs transform data.
- Glue Connections provide access to external data sources.
- Amazon S3 is commonly used as the storage layer.
- Amazon Athena can query datasets registered in the Glue Data Catalog.