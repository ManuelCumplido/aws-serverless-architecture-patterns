# AWS Glue Jobs

## What is an AWS Glue Job?

An AWS Glue Job is a data processing workload used to extract, transform, and load data.

Glue Jobs are commonly used for ETL and ELT pipelines.

A simple flow looks like:

```text
Data Source
    |
    v
Glue Job
    |
    | Read
    | Transform
    | Validate
    | Aggregate
    v
Destination
```

A common example is:

```text
Amazon S3
    |
    | JSON
    v
Glue Job
    |
    | PySpark
    | Filter
    | Rename
    | Cast
    v
Amazon S3
    |
    | Parquet
    v
Analytics
```

---

## Glue Job Types

AWS Glue supports different types of jobs.

The most common ones are:

- Spark jobs
- Python shell jobs
- Ray jobs

For large-scale ETL workloads, Spark jobs are commonly used.

---

## Spark Jobs

Spark jobs use Apache Spark for distributed data processing.

They are useful when working with:

- Large datasets
- Multiple files
- Joins
- Aggregations
- Complex transformations
- Distributed workloads

Example:

```text
Large Dataset
     |
     v
Apache Spark
     |
     | Distributed Processing
     v
Processed Dataset
```

AWS Glue manages the infrastructure required to run Spark.

This means we do not need to manually create or manage Spark clusters.

---

## PySpark

PySpark is the Python API for Apache Spark.

It allows us to write Spark transformations using Python.

Example:

```python
df = spark.read.parquet("s3://my-bucket/input/")

filtered_df = df.filter(df.status == "completed")

filtered_df.write.parquet("s3://my-bucket/output/")
```

In this example:

```text
Read
 |
 v
Filter
 |
 v
Write
```

PySpark operations can be distributed across multiple workers.

---

## GlueContext

AWS Glue provides a class called `GlueContext`.

`GlueContext` extends the capabilities of a Spark context and adds AWS Glue-specific functionality.

A typical Glue script starts with:

```python
import sys

from awsglue.context import GlueContext
from awsglue.job import Job

from pyspark.context import SparkContext

sc = SparkContext()
glue_context = GlueContext(sc)

spark = glue_context.spark_session

job = Job(glue_context)
```

The relationship looks like:

```text
SparkContext
     |
     v
GlueContext
     |
     v
Spark Session
```

`GlueContext` provides methods to:

- Read from the Glue Data Catalog
- Work with DynamicFrames
- Write data to supported destinations
- Integrate Glue-specific features

---

## Reading Data

A Glue Job can read data from different sources.

Common sources include:

- Amazon S3
- Glue Data Catalog
- Amazon RDS
- JDBC-compatible databases
- Amazon Redshift

---

## Reading from S3

A Spark DataFrame can read data directly from S3.

Example:

```python
df = spark.read.json(
    "s3://my-data-bucket/raw/"
)
```

For Parquet:

```python
df = spark.read.parquet(
    "s3://my-data-bucket/curated/"
)
```

---

## Reading from the Glue Data Catalog

Glue can use metadata stored in the Data Catalog.

Example:

```python
dynamic_frame = glue_context.create_dynamic_frame.from_catalog(
    database="analytics",
    table_name="transactions"
)
```

This means Glue does not need the schema and location to be hard-coded in the script.

The Data Catalog already contains that information.

```text
Glue Job
    |
    v
Data Catalog
    |
    | Schema
    | Location
    v
Amazon S3
```

---

## Transformations

The main responsibility of a Glue Job is to transform data.

Common transformations include:

- Filtering rows
- Selecting columns
- Renaming columns
- Casting data types
- Joining datasets
- Aggregating records
- Removing duplicates
- Handling null values
- Calculating new columns

---

## Filtering

Example:

```python
filtered_df = df.filter(
    df.status == "completed"
)
```

Input:

```text
tx-001 | completed
tx-002 | pending
tx-003 | completed
```

Output:

```text
tx-001 | completed
tx-003 | completed
```

---

## Selecting Columns

Example:

```python
selected_df = df.select(
    "transaction_id",
    "amount",
    "status"
)
```

This removes unnecessary columns from the processing pipeline.

---

## Renaming Columns

Example:

```python
renamed_df = df.withColumnRenamed(
    "transaction_date",
    "created_at"
)
```

---

## Casting Data Types

Data types are important when processing analytical datasets.

Example:

```python
from pyspark.sql.functions import col

converted_df = df.withColumn(
    "amount",
    col("amount").cast("double")
)
```

This converts the `amount` column to a double.

---

## Adding Columns

New columns can be created during a transformation.

Example:

```python
from pyspark.sql.functions import current_timestamp

result_df = df.withColumn(
    "processed_at",
    current_timestamp()
)
```

---

## Joining Datasets

Glue Jobs can join multiple datasets.

Example:

```python
result_df = transactions_df.join(
    customers_df,
    transactions_df.customer_id == customers_df.customer_id,
    "inner"
)
```

Conceptually:

```text
Transactions
      |
      |
      +------+
             |
             v
            Join
             ^
             |
      +------+
      |
Customers
```

The result could contain data from both datasets.

---

## Aggregations

Spark can aggregate large datasets.

Example:

```python
from pyspark.sql.functions import count, sum

summary_df = df.groupBy(
    "status"
).agg(
    count("*").alias("total_transactions"),
    sum("amount").alias("total_amount")
)
```

Input:

```text
completed | 100
completed | 200
pending   | 50
```

Output:

```text
completed | 2 | 300
pending   | 1 | 50
```

---

## Writing Data

After transformations are completed, the result must normally be stored somewhere.

A common destination is Amazon S3.

Example:

```python
result_df.write.mode("overwrite").parquet(
    "s3://my-data-bucket/curated/"
)
```

Common write modes include:

- `overwrite`
- `append`
- `ignore`
- `error`

---

## Writing Parquet

Parquet is commonly used for analytics workloads.

Example:

```python
result_df.write.parquet(
    "s3://my-data-bucket/curated/transactions/"
)
```

The flow becomes:

```text
JSON / CSV
    |
    v
Glue Job
    |
    | Transform
    v
Parquet
    |
    v
Amazon S3
```

---

## Partitioned Writes

Data can also be written using partitions.

Example:

```python
result_df.write.partitionBy(
    "year",
    "month"
).parquet(
    "s3://my-data-bucket/transactions/"
)
```

The resulting S3 structure could look like:

```text
transactions/
|
├── year=2026/
│   ├── month=08/
│   └── month=09/
```

Partitioning can help reduce the amount of data scanned by analytical queries.

---

## Job Parameters

Glue Jobs can receive runtime parameters.

This allows the same script to be reused across different environments or datasets.

Example:

```python
import sys

from awsglue.utils import getResolvedOptions

args = getResolvedOptions(
    sys.argv,
    [
        "JOB_NAME",
        "SOURCE_BUCKET",
        "DESTINATION_BUCKET"
    ]
)
```

The values can then be used in the script:

```python
source_bucket = args["SOURCE_BUCKET"]
destination_bucket = args["DESTINATION_BUCKET"]
```

Instead of hard-coding:

```python
source_bucket = "my-dev-bucket"
```

This makes the job more reusable.

---

## Job Initialization

A Glue Job should normally be initialized.

Example:

```python
job.init(
    args["JOB_NAME"],
    args
)
```

And after the processing finishes:

```python
job.commit()
```

A simplified script structure looks like:

```python
# Initialize

job.init(...)

# Read

# Transform

# Write

# Finish

job.commit()
```

---

## IAM Role

A Glue Job requires an IAM role.

The role defines which AWS resources the job can access.

For example, a job may require permissions to:

```text
Read from S3
Write to S3
Read Glue Data Catalog
Access Secrets Manager
Connect to RDS
Write CloudWatch Logs
```

Example permissions could include actions such as:

```text
s3:GetObject
s3:PutObject
glue:GetTable
glue:GetDatabase
logs:CreateLogStream
logs:PutLogEvents
```

The principle of least privilege should be used.

A Glue Job should only receive the permissions it actually needs.

---

## Workers and Compute

Glue Jobs run using managed compute resources.

For Spark jobs, AWS Glue provides worker types.

A job can use multiple workers to process data in parallel.

Conceptually:

```text
Dataset
   |
   v
Glue Job
   |
   +--> Worker 1
   |
   +--> Worker 2
   |
   +--> Worker 3
   |
   +--> Worker 4
```

More workers can provide more processing capacity, but they also increase cost.

The correct configuration depends on:

- Dataset size
- Transformation complexity
- Join operations
- Memory requirements
- Execution time

---

## DPUs

A DPU stands for Data Processing Unit.

Glue uses DPUs as a measurement of processing capacity.

A DPU provides a combination of:

- CPU
- Memory
- Processing resources

The amount of compute required depends on the worker type and number of workers.

A simple mental model is:

```text
More Compute
    |
    +--> Potentially faster processing
    |
    +--> Higher cost
```

---

## Job Bookmarks

Job Bookmarks help Glue track which data has already been processed.

This is useful for incremental processing.

Imagine S3 contains:

```text
day-01.json
day-02.json
day-03.json
```

First execution:

```text
day-01
day-02
day-03
   |
   v
Processed
```

Later:

```text
day-01
day-02
day-03
day-04
```

With Job Bookmarks enabled, Glue can avoid processing the same data again and focus on new data.

```text
Already Processed
|
├── day-01
├── day-02
└── day-03

New
|
└── day-04
     |
     v
Process
```

Job Bookmarks are useful for incremental ETL pipelines.

---

## Full Load vs Incremental Load

A full load processes the entire dataset.

```text
All Data
   |
   v
Glue Job
```

An incremental load processes only new or changed data.

```text
New Data
   |
   v
Glue Job
```

Incremental processing can reduce:

- Execution time
- Compute usage
- Cost

Job Bookmarks are one mechanism Glue provides to support incremental processing.

---

## Retry and Timeout

Glue Jobs can be configured with retry and timeout settings.

Retries can help when failures are temporary.

Examples:

- Temporary network problem
- Dependency unavailable
- Database connection issue

Timeout prevents a job from running indefinitely.

Example:

```text
Glue Job
   |
   | Failure
   v
Retry
   |
   v
Success
```

Retry settings should be used carefully.

A retry does not fix deterministic problems such as:

- Invalid code
- Invalid schema
- Missing permissions
- Incorrect configuration

---

## Monitoring with CloudWatch

Glue integrates with Amazon CloudWatch.

CloudWatch can be used to inspect:

- Job execution logs
- Errors
- Runtime information
- Execution duration

A common debugging flow is:

```text
Glue Job Failed
      |
      v
CloudWatch Logs
      |
      v
Find Error
      |
      v
Fix Job
```

Useful information may include:

- Python exceptions
- Spark errors
- Permission failures
- Connection errors
- Schema errors

---

## Common Glue Job Failures

Common problems include:

### Permission Errors

Example:

```text
AccessDenied
```

Possible cause:

```text
Glue IAM Role
     |
     X
     |
S3 Bucket
```

The job does not have permission to access the resource.

---

### Schema Problems

Example:

```text
Expected:
amount -> double

Received:
amount -> string
```

This can cause transformations or writes to fail.

---

### Network Problems

When accessing a private RDS database:

```text
Glue Job
    |
    X
    |
Private RDS
```

Possible causes include:

- Incorrect subnet
- Security Group configuration
- Routing issues
- DNS issues

---

### Memory Problems

Large joins or transformations can consume significant memory.

Example:

```text
Large Dataset A
       |
       +------+
              |
             Join
              |
       +------+
       |
Large Dataset B
```

Possible solutions may include:

- Increasing workers
- Filtering data earlier
- Reducing columns
- Improving partitioning
- Optimizing join strategy

---

## Glue Job vs Crawler

A crawler discovers metadata.

A job processes records.

```text
Crawler
   |
   v
Schema Discovery
   |
   v
Data Catalog
```

```text
Glue Job
   |
   v
Data Processing
   |
   v
Output Dataset
```

A crawler does not replace a Glue Job.

A Glue Job does not necessarily require a crawler.

---

## Glue Job vs Lambda

Glue and Lambda solve different types of workloads.

| AWS Glue Job | AWS Lambda |
|---|---|
| Data processing | Application processing |
| Large datasets | Small payloads |
| Distributed processing | Single execution environment |
| Spark / PySpark | Python / Node.js / Java |
| Batch ETL | Event-driven |
| Analytics | APIs and events |

Example Lambda workload:

```text
API Request
    |
    v
Lambda
    |
    v
DynamoDB
```

Example Glue workload:

```text
Millions of Records
        |
        v
    Glue Job
        |
        v
Processed Dataset
```

---

## Typical Glue Job Pipeline

A common Glue pipeline looks like:

```text
Source
  |
  v
Extract
  |
  v
Glue Job
  |
  | Clean
  | Filter
  | Join
  | Aggregate
  | Validate
  v
Transform
  |
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

## Glue Job Mental Model

A useful mental model is:

```text
What data do I need?
        |
        v
      READ
        |
        v
What changes do I need?
        |
        v
   TRANSFORM
        |
        v
Is the data correct?
        |
        v
    VALIDATE
        |
        v
Where should it go?
        |
        v
      WRITE
```

Or simply:

```text
Read
 |
 v
Transform
 |
 v
Write
```

---

## Key Takeaways

- Glue Jobs are used to process and transform data.
- Spark jobs are commonly used for large-scale ETL workloads.
- PySpark allows Spark transformations to be written in Python.
- `GlueContext` provides AWS Glue-specific functionality.
- Glue Jobs can read data from S3, the Data Catalog, RDS, JDBC sources, and other systems.
- Common transformations include filtering, joining, casting, and aggregating.
- Data is commonly written to S3 in Parquet format.
- Job parameters make scripts reusable across environments.
- Glue Jobs require an IAM role.
- Workers provide distributed processing capacity.
- More workers can improve performance but also increase cost.
- Job Bookmarks can help with incremental processing.
- CloudWatch is important for monitoring and troubleshooting Glue Jobs.
- Glue Jobs and Crawlers have different responsibilities.
- Glue is generally better suited than Lambda for large-scale batch data processing.

---

## Interview Questions

### What is an AWS Glue Job?

An AWS Glue Job is a managed data processing workload used to perform ETL or ELT transformations.

### What is PySpark?

PySpark is the Python API for Apache Spark and is commonly used in Glue for distributed data processing.

### What is GlueContext?

`GlueContext` is an AWS Glue abstraction built on top of Spark that provides Glue-specific capabilities such as integration with the Data Catalog and DynamicFrames.

### Can a Glue Job read directly from S3?

Yes.

A Glue Job can read data directly from S3 or use metadata stored in the Glue Data Catalog.

### What types of transformations can a Glue Job perform?

Common transformations include filtering, joining, aggregating, renaming columns, casting data types, and removing invalid data.

### What are Job Bookmarks?

Job Bookmarks allow Glue to track previously processed data and can help implement incremental ETL pipelines.

### What is the difference between a full load and an incremental load?

A full load processes the complete dataset.

An incremental load processes only new or changed data.

### Why does a Glue Job need an IAM role?

The IAM role defines which AWS resources the job is allowed to access.

### How do you monitor a Glue Job?

Glue Job logs and execution information can be monitored using Amazon CloudWatch.

### What is the difference between a Glue Job and a Crawler?

A Crawler discovers schemas and metadata.

A Glue Job processes and transforms the actual data.

### When would you use Glue instead of Lambda?

Glue is generally better suited for large datasets, batch processing, complex transformations, and distributed ETL workloads.

Lambda is generally better suited for short-running, event-driven application logic.