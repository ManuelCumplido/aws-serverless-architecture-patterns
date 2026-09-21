# AWS Glue Connections and Amazon RDS

## What is an AWS Glue Connection?

An AWS Glue Connection stores the information AWS Glue needs to connect to an external data source.

Connections are commonly used with:

- Amazon RDS
- Amazon Aurora
- JDBC-compatible databases
- Amazon Redshift
- Other supported data stores

A simple architecture looks like:

```text
AWS Glue
    |
    | Connection
    v
External Data Source
```

For Amazon RDS:

```text
AWS Glue
    |
    | JDBC
    v
Amazon RDS PostgreSQL
```

---

## Why Do We Need a Connection?

A Glue Job needs to know how to reach the database.

A connection can contain information such as:

- JDBC URL
- Database engine
- Database name
- Port
- Username
- Password or secret reference
- VPC configuration
- Subnet
- Security Group

Instead of hard-coding all connection details in the script, AWS Glue can use a configured connection.

---

## JDBC

JDBC stands for Java Database Connectivity.

It is a standard interface used by applications to connect to relational databases.

AWS Glue uses JDBC to connect to databases such as:

- PostgreSQL
- MySQL
- MariaDB
- SQL Server
- Oracle

Example:

```text
Glue Job
    |
    | JDBC
    v
PostgreSQL Database
```

A PostgreSQL JDBC URL can look like:

```text
jdbc:postgresql://my-database.example.com:5432/analytics
```

The main parts are:

```text
jdbc:postgresql://HOST:PORT/DATABASE
```

---

## Amazon RDS

Amazon RDS is a managed relational database service.

It supports database engines such as:

- PostgreSQL
- MySQL
- MariaDB
- Oracle
- SQL Server

RDS is commonly used for operational or transactional workloads.

For example:

```text
Application
    |
    v
Amazon RDS
    |
    | Relational Tables
    v
Operational Data
```

AWS Glue can extract data from RDS and move it into an analytics pipeline.

Example:

```text
Amazon RDS
    |
    v
AWS Glue Job
    |
    v
Amazon S3
```

---

## Typical RDS to Glue Architecture

A common architecture looks like:

```text
Amazon RDS
    |
    | JDBC
    v
Glue Connection
    |
    v
Glue Job
    |
    | Transform
    v
Amazon S3
```

The Glue Job can:

- Read relational tables
- Filter records
- Join datasets
- Change data types
- Aggregate information
- Write the result to S3

---

## Private RDS Databases

In many environments, RDS is not publicly accessible.

The database can exist inside a private subnet in a VPC.

Example:

```text
VPC
|
├── Private Subnet
|       |
|       v
|   Amazon RDS
|
└── Glue Job Network Interface
        |
        v
     Glue Job
```

For Glue to connect to a private RDS database, network connectivity must be configured correctly.

---

## VPC

A VPC, or Virtual Private Cloud, is an isolated network inside AWS.

Resources such as RDS can be deployed inside a VPC.

Example:

```text
AWS Account
    |
    v
   VPC
    |
    ├── Public Subnet
    |
    └── Private Subnet
            |
            v
        Amazon RDS
```

When Glue needs to access resources inside a VPC, it must be configured with network information that allows it to reach those resources.

---

## Subnets

A subnet is a range of IP addresses inside a VPC.

RDS is commonly deployed in private subnets.

Example:

```text
VPC
|
├── subnet-a
|
├── subnet-b
|
└── subnet-c
```

A Glue Connection can specify the subnet that Glue should use.

This allows Glue to create network interfaces inside the VPC.

---

## Security Groups

Security Groups act as virtual firewalls.

They control which traffic is allowed to enter or leave AWS resources.

For example:

```text
Glue Job
    |
    | TCP 5432
    v
RDS PostgreSQL
```

PostgreSQL normally uses port:

```text
5432
```

MySQL commonly uses:

```text
3306
```

The RDS Security Group must allow inbound traffic from the Security Group used by Glue.

Conceptually:

```text
Glue Security Group
        |
        | Allowed
        v
RDS Security Group
        |
        v
PostgreSQL : 5432
```

A better security practice is to allow traffic from a specific Security Group instead of allowing traffic from a large IP range.

---

## Network Flow

For a private PostgreSQL database, the communication could look like:

```text
Glue Job
    |
    v
Glue Connection
    |
    v
VPC
    |
    v
Subnet
    |
    v
Security Group
    |
    | TCP 5432
    v
RDS PostgreSQL
```

If any part of this network path is incorrect, the connection can fail.

---

## Credentials

Glue needs database credentials to authenticate with RDS.

A simple configuration could use:

```text
Username
Password
```

However, hard-coding credentials in scripts is not recommended.

For example, avoid:

```python
username = "admin"
password = "my-password"
```

A better approach is to store credentials securely.

---

## AWS Secrets Manager

AWS Secrets Manager can be used to securely store database credentials.

Example:

```text
AWS Secrets Manager
        |
        | Credentials
        v
    Glue Job
        |
        | JDBC
        v
   Amazon RDS
```

A secret could contain:

```json
{
  "username": "glue_user",
  "password": "example-password",
  "host": "database.example.com",
  "port": 5432,
  "database": "analytics"
}
```

The Glue IAM Role needs permission to read the secret.

---

## IAM Permissions for Secrets Manager

If a Glue Job uses Secrets Manager, its IAM role needs permission to access the secret.

Example action:

```text
secretsmanager:GetSecretValue
```

Conceptually:

```text
Glue Job
    |
    | IAM Role
    v
Secrets Manager
    |
    | Credentials
    v
RDS
```

The IAM policy should follow the principle of least privilege.

The Glue Job should only be allowed to access the required secret.

---

## Reading Data from RDS

A Glue Job can read relational data through JDBC.

Conceptually:

```text
RDS Table
    |
    v
JDBC
    |
    v
Glue Job
    |
    v
DataFrame / DynamicFrame
```

For example, imagine a table called:

```text
transactions
```

With columns:

```text
transaction_id
customer_id
amount
status
created_at
```

Glue can extract this data and process it.

---

## Reading Through the Glue Data Catalog

A crawler can discover an RDS database and create tables in the Glue Data Catalog.

Example:

```text
Amazon RDS
    |
    v
Glue Crawler
    |
    v
Glue Data Catalog
    |
    v
Glue Job
```

Then the Glue Job can read using the catalog metadata.

Example:

```python
dynamic_frame = glue_context.create_dynamic_frame.from_catalog(
    database="analytics",
    table_name="transactions"
)
```

The Glue Data Catalog contains information about how to access the source.

---

## Reading Directly with JDBC

A Glue Job can also connect directly through JDBC.

Conceptually:

```text
Glue Job
    |
    | JDBC URL
    | Credentials
    v
Amazon RDS
```

Example structure:

```python
df = spark.read \
    .format("jdbc") \
    .option("url", jdbc_url) \
    .option("dbtable", "transactions") \
    .option("user", username) \
    .option("password", password) \
    .load()
```

This creates a Spark DataFrame containing the records from the database table.

---

## Reading a Table vs Reading a Query

A JDBC source can read an entire table.

Example:

```text
transactions
```

But sometimes we only need part of the data.

Instead of extracting everything:

```sql
SELECT *
FROM transactions;
```

we may want:

```sql
SELECT *
FROM transactions
WHERE created_at >= CURRENT_DATE - INTERVAL '1 day';
```

This reduces the amount of data transferred and processed.

A good principle is:

```text
Filter as early as possible
```

Instead of:

```text
RDS
 |
 | Read everything
 v
Glue
 |
 | Filter
 v
Result
```

Prefer:

```text
RDS
 |
 | Read only required data
 v
Glue
 |
 v
Result
```

This can improve performance and reduce resource usage.

---

## JDBC Pushdown

When possible, filtering can be pushed down to the database.

This means the database performs part of the filtering before returning the records to Glue.

Example:

```text
Without Pushdown

RDS
 |
 | 1,000,000 rows
 v
Glue
 |
 | Filter
 v
10,000 rows
```

With filtering at the source:

```text
RDS
 |
 | Filter
 v
10,000 rows
 |
 v
Glue
```

Reducing data movement is usually more efficient.

---

## Parallel Reads

Large relational tables can take a long time to read using a single JDBC connection.

Spark can divide the workload and read data in parallel.

Conceptually:

```text
Large RDS Table
      |
      +--> Partition 1 --> Worker 1
      |
      +--> Partition 2 --> Worker 2
      |
      +--> Partition 3 --> Worker 3
      |
      +--> Partition 4 --> Worker 4
```

Parallel reads can improve performance for large datasets.

They require a suitable partitioning column, such as:

```text
id
```

or another column with a good distribution of values.

---

## Example JDBC Partitioning

A Spark JDBC read can use options such as:

```text
partitionColumn
lowerBound
upperBound
numPartitions
```

Conceptually:

```text
id 1 - 250000
        |
        v
Worker 1

id 250001 - 500000
        |
        v
Worker 2

id 500001 - 750000
        |
        v
Worker 3

id 750001 - 1000000
        |
        v
Worker 4
```

This is useful for large tables, but it is usually unnecessary for small datasets.

---

## Database Load

Glue can create significant load on an operational database.

For example:

```text
Production Application
        |
        v
      RDS
       ^
       |
       |
Glue ETL Job
```

Both systems may compete for database resources.

Because of this, ETL workloads should consider:

- Query size
- Execution frequency
- Indexes
- Database CPU
- Database connections
- Read replicas
- Processing windows

For large analytics workloads, extracting large amounts of data directly from the primary production database can be risky.

---

## RDS Read Replica

A read replica can sometimes be used for analytics extraction.

Conceptually:

```text
Application
    |
    v
Primary RDS
    |
    | Replication
    v
Read Replica
    |
    v
Glue Job
```

This can reduce the analytical workload on the primary database.

Whether this architecture is necessary depends on:

- Data volume
- Query frequency
- Database performance
- Consistency requirements
- Cost

---

## Incremental Extraction

Instead of extracting the complete RDS table every time, a pipeline can process only new or updated records.

Imagine this column:

```text
updated_at
```

The Glue Job could extract:

```sql
SELECT *
FROM transactions
WHERE updated_at > last_processed_time;
```

Conceptually:

```text
RDS
|
├── Old Data
|
└── New Data
      |
      v
   Glue Job
```

This is called incremental extraction.

It can reduce:

- Data transfer
- Processing time
- Compute cost
- Database load

---

## Full Load vs Incremental Extraction

### Full Load

```text
Entire RDS Table
       |
       v
    Glue Job
```

Useful when:

- Dataset is small
- Initial load is required
- Full refresh is necessary

### Incremental Load

```text
New / Changed Records
          |
          v
       Glue Job
```

Useful when:

- Dataset is large
- Pipeline runs frequently
- Only recent changes are required

---

## Common RDS Connection Problems

### Connection Timeout

Example:

```text
Connection timed out
```

Possible causes:

- Incorrect subnet
- Incorrect Security Group
- Incorrect route
- Wrong hostname
- Wrong port
- Database unavailable

---

## Security Group Problem

Example:

```text
Glue Job
    |
    X
    |
RDS
```

The RDS Security Group may not allow inbound traffic from the Glue Security Group.

For PostgreSQL, verify:

```text
Protocol: TCP
Port: 5432
Source: Glue Security Group
```

---

## Authentication Problem

Example:

```text
password authentication failed
```

Possible causes:

- Incorrect username
- Incorrect password
- Incorrect secret
- Database user does not exist
- Credentials were rotated

---

## IAM Problem

Example:

```text
AccessDeniedException
```

This may happen when Glue cannot access:

- Secrets Manager
- S3
- Glue Data Catalog
- KMS-encrypted resources

IAM controls AWS resource access.

Database authentication controls access inside PostgreSQL or MySQL.

These are different security layers.

---

## IAM vs Database Credentials

This distinction is important.

```text
IAM
 |
 | Controls access to AWS resources
 v
Secrets Manager / S3 / Glue
```

While:

```text
Database User
 |
 | Controls database access
 v
PostgreSQL / MySQL
```

A Glue Job may need both:

```text
IAM Permissions
       +
Database Credentials
       |
       v
Successful Connection
```

---

## DNS Problems

Glue must be able to resolve the RDS endpoint.

Example:

```text
my-database.abc123.us-east-1.rds.amazonaws.com
```

If network or DNS configuration is incorrect, Glue may not be able to resolve or reach the endpoint.

---

## Connection Mental Model

A useful way to understand Glue-to-RDS connectivity is:

```text
Who am I?
    |
    v
IAM Role

How do I authenticate?
    |
    v
Database Credentials

Where is the database?
    |
    v
RDS Endpoint

How do I reach it?
    |
    v
VPC + Subnet

Am I allowed through?
    |
    v
Security Groups

How do I communicate?
    |
    v
JDBC
```

All of these pieces must work together.

---

## Typical Glue and RDS Architecture

A complete architecture could look like:

```text
                    AWS Secrets Manager
                            |
                            | Credentials
                            v
Amazon RDS <---------- AWS Glue Job
     ^                     |
     |                     |
     | JDBC                | IAM Role
     |                     |
     +---------------------+
             |
             v
      VPC / Subnet
             |
             v
      Security Groups

AWS Glue Job
     |
     | Transform
     v
Amazon S3
```

---

## Best Practices

When connecting Glue to RDS:

- Avoid hard-coding database credentials.
- Use Secrets Manager when appropriate.
- Use private network connectivity.
- Restrict Security Group rules.
- Follow least-privilege IAM permissions.
- Filter data as early as possible.
- Prefer incremental extraction for large datasets.
- Avoid unnecessary full-table scans.
- Monitor the impact on the operational database.
- Consider a read replica for heavy analytical workloads.
- Use parallel JDBC reads only when needed.
- Monitor Glue Job logs in CloudWatch.

---

## Glue Connection vs Crawler

A Glue Connection and a Crawler solve different problems.

```text
Connection
    |
    v
How Glue reaches the data source
```

```text
Crawler
    |
    v
How Glue discovers the dataset schema
```

They can work together:

```text
RDS
 |
 v
Glue Connection
 |
 v
Glue Crawler
 |
 v
Data Catalog
```

But a Connection does not automatically create tables in the Data Catalog.

---

## Glue Connection vs Glue Job

A Glue Connection contains connectivity information.

A Glue Job performs processing.

```text
Glue Connection
      |
      | Connectivity
      v
    RDS
```

```text
Glue Job
   |
   | Processing
   v
Output Data
```

A Glue Job can use a Glue Connection to reach an external database.

---

## Key Takeaways

- Glue Connections store information required to access external data sources.
- JDBC is commonly used to connect Glue to relational databases.
- Amazon RDS can be used as a source for Glue ETL pipelines.
- Private RDS databases require correct VPC connectivity.
- Glue may need a subnet and Security Group to reach RDS.
- Security Groups control network access between Glue and RDS.
- Database credentials should not be hard-coded.
- Secrets Manager can securely store database credentials.
- IAM permissions and database permissions are separate security layers.
- Filtering at the source can reduce data movement.
- Incremental extraction is often better than full-table extraction for large datasets.
- Parallel JDBC reads can improve performance for large tables.
- ETL workloads can affect production database performance.
- CloudWatch logs are important when troubleshooting connection problems.

---

## Interview Questions

### What is an AWS Glue Connection?

An AWS Glue Connection stores connectivity information that Glue can use to access an external data source such as Amazon RDS.

### How does AWS Glue connect to RDS?

Glue commonly connects to RDS using JDBC.

For private RDS databases, Glue also needs the correct VPC, subnet, and Security Group configuration.

### Why does Glue need a subnet to access private RDS?

Because Glue needs network connectivity inside the VPC where the private database is reachable.

### What Security Group rule is needed for PostgreSQL?

The RDS Security Group should allow inbound TCP traffic on port 5432 from the Security Group used by Glue.

### Should database credentials be stored in the Glue script?

No.

Credentials should generally be stored securely, for example in AWS Secrets Manager.

### What is the difference between IAM permissions and database credentials?

IAM permissions control access to AWS resources.

Database credentials control authentication and authorization inside the relational database.

### What is incremental extraction?

Incremental extraction means reading only new or changed records instead of processing the entire database table every time.

### Why is incremental extraction useful?

It can reduce database load, data transfer, execution time, and Glue processing cost.

### Why should we filter data as early as possible?

Filtering at the source reduces the amount of data transferred to Glue and the amount of data that Glue needs to process.

### Can Glue affect the performance of a production RDS database?

Yes.

Large or frequent queries from Glue can consume database CPU, memory, I/O, and connections.

### When could a read replica be useful?

A read replica can be useful when analytical extraction is heavy and we want to reduce read workload on the primary operational database.

### What is JDBC?

JDBC is a standard interface used to connect applications and data-processing systems to relational databases.