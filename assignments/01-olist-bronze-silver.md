# Olist Bronze-to-Silver Data Engineering Assignment

## Dataset

OList data to download. 

https://gopalakrishnan-my.sharepoint.com/:f:/g/personal/gs_training_sh/IgAyLtwyhFVZSa_aHESUJys3AQLon4o_g19UOz8oYppHMxw?e=vsK479

## Objective

Build a Bronze-to-Silver data pipeline for the Olist e-commerce dataset using:

- HDFS for Bronze/raw data storage
- Apache Hive for Bronze external tables
- Apache Spark / PySpark for data processing
- Apache Polaris as the Iceberg catalog
- Apache Iceberg for Silver tables
- SeaweedFS S3 as the physical storage for Iceberg data
- Airflow later for scheduling the individual pipelines

The objective of this assignment is not to build one large Spark program.

Each dataset must have its own independent processing script so that the pipelines can later be scheduled, monitored, retried, and maintained individually using Apache Airflow.

---

# 1. Datasets in Scope

Use the following Olist datasets.

| Dataset | Source file | Type |
|---|---|---|
| Customers | `olist_customers_dataset.csv` | Dimension |
| Products | `olist_products_dataset.csv` | Dimension |
| Sellers | `olist_sellers_dataset.csv` | Dimension |
| Orders | `olist_orders_dataset.csv` | Fact / Transaction |
| Order Items | `olist_order_items_dataset.csv` | Fact / Transaction |
| Order Payments | `olist_order_payments_dataset.csv` | Fact / Transaction |
| Order Reviews | `olist_order_reviews_dataset.csv` | Fact / Event |

The following files are **excluded from this assignment**:

- `olist_geolocation_dataset.csv`
- `product_category_name_translation.csv`

Therefore, the assignment contains **7 Bronze datasets and 7 Silver pipelines**.

---

# 2. Bronze Layer — HDFS

Create the root Bronze directory in HDFS:

```bash
hdfs dfs -mkdir -p /olist/bronze
```

Create one directory for each type of incoming dataset.

```text
/olist/bronze/
├── customers/
├── products/
├── sellers/
├── orders/
├── order_items/
├── order_payments/
└── order_reviews/
```

Create the directories using HDFS commands.

```bash
hdfs dfs -mkdir -p /olist/bronze/customers
hdfs dfs -mkdir -p /olist/bronze/products
hdfs dfs -mkdir -p /olist/bronze/sellers

hdfs dfs -mkdir -p /olist/bronze/orders
hdfs dfs -mkdir -p /olist/bronze/order_items
hdfs dfs -mkdir -p /olist/bronze/order_payments
hdfs dfs -mkdir -p /olist/bronze/order_reviews
```

---

# 3. Upload the Source Files

Place each CSV file in its corresponding Bronze directory.

Example:

```bash
hdfs dfs -put olist_customers_dataset.csv \
    /olist/bronze/customers/

hdfs dfs -put olist_products_dataset.csv \
    /olist/bronze/products/

hdfs dfs -put olist_sellers_dataset.csv \
    /olist/bronze/sellers/

hdfs dfs -put olist_orders_dataset.csv \
    /olist/bronze/orders/

hdfs dfs -put olist_order_items_dataset.csv \
    /olist/bronze/order_items/

hdfs dfs -put olist_order_payments_dataset.csv \
    /olist/bronze/order_payments/

hdfs dfs -put olist_order_reviews_dataset.csv \
    /olist/bronze/order_reviews/
```

The resulting structure should resemble:

```text
/olist/bronze/
├── customers/
│   └── olist_customers_dataset.csv
│
├── products/
│   └── olist_products_dataset.csv
│
├── sellers/
│   └── olist_sellers_dataset.csv
│
├── orders/
│   └── olist_orders_dataset.csv
│
├── order_items/
│   └── olist_order_items_dataset.csv
│
├── order_payments/
│   └── olist_order_payments_dataset.csv
│
└── order_reviews/
    └── olist_order_reviews_dataset.csv
```

Verify the files.

```bash
hdfs dfs -ls -R /olist/bronze
```

---

# 4. Important Bronze Storage Rule

Do **not** design the directories assuming that there will always be exactly one CSV file.

For example, today:

```text
/olist/bronze/orders/
    olist_orders_dataset.csv
```

Later the same directory may contain additional batch files:

```text
/olist/bronze/orders/
    olist_orders_dataset.csv
    orders_2026_10_07.csv
    orders_2026_10_08.csv
    orders_2026_10_09.csv
```

All files placed inside a dataset directory must follow the same expected schema.

This design is important because future batches may be delivered daily or periodically.

The Spark/Airflow pipeline should process the dataset location rather than being tightly coupled to one manually specified filename.

---

# 5. Bronze Hive Database

Create an Apache Hive database named:

```text
olist_bronze
```

Example:

```sql
CREATE DATABASE IF NOT EXISTS olist_bronze;
```

Verify:

```sql
SHOW DATABASES;
```

---

# 6. Bronze Hive External Tables

Create one **EXTERNAL Hive table** for every Bronze dataset.

Required tables:

```text
olist_bronze.olist_customers
olist_bronze.olist_products
olist_bronze.olist_sellers
olist_bronze.olist_orders
olist_bronze.olist_order_items
olist_bronze.olist_order_payments
olist_bronze.olist_order_reviews
```

Each external table must point to the corresponding HDFS directory.

For example:

```text
olist_bronze.olist_customers
        ↓
/olist/bronze/customers/
```

and:

```text
olist_bronze.olist_orders
        ↓
/olist/bronze/orders/
```

---

# 7. Example Hive External Table

The following is only a reference pattern.

Students must inspect the CSV and create the appropriate definitions for all seven datasets.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS olist_bronze.olist_customers
(
    customer_id STRING,
    customer_unique_id STRING,
    customer_zip_code_prefix STRING,
    customer_city STRING,
    customer_state STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/olist/bronze/customers'
TBLPROPERTIES (
    'skip.header.line.count'='1'
);
```

You must create corresponding external tables for:

```text
customers
products
sellers
orders
order_items
order_payments
order_reviews
```

Bronze tables should remain reasonably close to the source data.

Do not perform major business transformations while defining Bronze tables.

---

# 8. Validate the Bronze Layer

Run basic checks on every Bronze table.

Example:

```sql
SELECT COUNT(*)
FROM olist_bronze.olist_customers;
```

```sql
SELECT *
FROM olist_bronze.olist_customers
LIMIT 10;
```

Students should validate at minimum:

- table exists
- columns are readable
- header is not appearing as a normal row
- row count is reasonable
- important identifiers are populated
- numeric columns can later be converted correctly
- timestamp fields contain expected values

Perform these checks for all seven datasets.

---

# 9. Polaris Catalog

Create an Apache Polaris catalog named:

```text
olist
```

The physical storage for this catalog must use the configured **SeaweedFS S3-compatible storage**.

Conceptually:

```text
Polaris Catalog
       │
       └── olist
             │
             └── SeaweedFS S3 storage
```

The Polaris catalog configuration and SeaweedFS credentials/endpoints will be provided as part of the lab environment.

Students do not need to invent a different catalog name.

---

# 10. Create the Silver Namespace

Inside the `olist` Polaris catalog, create a namespace named:

```text
olist_silver
```

The final logical structure should therefore be:

```text
olist
└── olist_silver
```

Using Spark SQL, the operation will look similar to:

```sql
CREATE NAMESPACE IF NOT EXISTS olist.olist_silver;
```

Verify:

```sql
SHOW NAMESPACES IN olist;
```

In Iceberg terminology this is normally called a **namespace**.

It is conceptually similar to a database/schema used to organize related tables.

---

# 11. Expected Silver Tables

The following Iceberg tables must eventually exist:

```text
olist.olist_silver.customers
olist.olist_silver.products
olist.olist_silver.sellers
olist.olist_silver.orders
olist.olist_silver.order_items
olist.olist_silver.order_payments
olist.olist_silver.order_reviews
```

These tables must be:

- registered through Polaris
- stored as Apache Iceberg tables
- physically stored in SeaweedFS S3
- readable using Spark

---

# 12. Dimension Pipelines

The following datasets will be treated primarily as dimensions:

```text
customers
products
sellers
```

For this exercise, read these datasets **directly from their Bronze HDFS locations using Spark**.

Example:

```python
customers_df = (
    spark.read
        .option("header", "true")
        .option("inferSchema", "false")
        .csv("hdfs:///olist/bronze/customers/")
)
```

Inspect the data:

```python
customers_df.printSchema()
customers_df.show(10, truncate=False)
```

---

# 13. Dimension Transformation Example

The following is reference pseudocode close to working PySpark.

```python
from pyspark.sql import functions as F

source_df = (
    spark.read
        .option("header", "true")
        .csv("hdfs:///olist/bronze/customers/")
)

silver_df = (
    source_df
        .select(
            F.trim("customer_id").alias("customer_id"),
            F.trim("customer_unique_id").alias("customer_unique_id"),
            F.col("customer_zip_code_prefix")
                .cast("int")
                .alias("customer_zip_code_prefix"),
            F.trim("customer_city").alias("customer_city"),
            F.upper(F.trim("customer_state")).alias("customer_state")
        )
        .dropDuplicates(["customer_id"])
)
```

This is only a pattern.

Students must decide appropriate transformations for each dataset.

Typical Silver operations include:

- trimming strings
- converting blank values to null
- casting numeric columns
- casting date/timestamp columns
- validating required identifiers
- eliminating accidental duplicate records
- selecting useful columns
- using meaningful column names and data types

---

# 14. Write Dimension Data to Polaris/Iceberg

Write the transformed DataFrame into the Polaris catalog as an Iceberg table.

Example pattern:

```python
(
    silver_df.writeTo(
        "olist.olist_silver.customers"
    )
    .using("iceberg")
    .createOrReplace()
)
```

Depending on the processing strategy, students may use:

```python
.create()
```

or:

```python
.createOrReplace()
```

For later incremental pipelines, append/merge strategies may be required.

Do not blindly use `overwrite` without understanding the effect.

---

# 15. Verify the Iceberg Table

Example:

```python
spark.sql("""
    SELECT *
    FROM olist.olist_silver.customers
    LIMIT 10
""").show(truncate=False)
```

Also verify:

```sql
SHOW TABLES IN olist.olist_silver;
```

---

# 16. Repeat the Dimension Pattern

Implement the same Bronze-to-Silver approach for:

## Customers

```text
HDFS:
/olist/bronze/customers/

↓

Iceberg:
olist.olist_silver.customers
```

## Products

```text
HDFS:
/olist/bronze/products/

↓

Iceberg:
olist.olist_silver.products
```

## Sellers

```text
HDFS:
/olist/bronze/sellers/

↓

Iceberg:
olist.olist_silver.sellers
```

---

# 17. Fact / Transaction Pipelines

The following datasets should be processed as fact/event datasets:

```text
orders
order_items
order_payments
order_reviews
```

For these datasets, use the Bronze **Hive external tables** as the Spark source.

For example:

```python
orders_df = spark.table(
    "olist_bronze.olist_orders"
)
```

or:

```python
orders_df = spark.sql("""
    SELECT *
    FROM olist_bronze.olist_orders
""")
```

This requires the Spark session to have access to the Hive catalog/metastore used by the exercise environment.

---

# 18. Example Fact Transformation

Example for orders:

```python
from pyspark.sql import functions as F

bronze_df = spark.table(
    "olist_bronze.olist_orders"
)

silver_df = (
    bronze_df
        .select(
            F.trim("order_id").alias("order_id"),
            F.trim("customer_id").alias("customer_id"),
            F.trim("order_status").alias("order_status"),

            F.to_timestamp(
                "order_purchase_timestamp"
            ).alias("order_purchase_timestamp"),

            F.to_timestamp(
                "order_approved_at"
            ).alias("order_approved_at"),

            F.to_timestamp(
                "order_delivered_carrier_date"
            ).alias("order_delivered_carrier_date"),

            F.to_timestamp(
                "order_delivered_customer_date"
            ).alias("order_delivered_customer_date"),

            F.to_timestamp(
                "order_estimated_delivery_date"
            ).alias("order_estimated_delivery_date")
        )
)
```

You may derive useful processing columns where appropriate.

For example:

```python
silver_df = (
    silver_df
        .withColumn(
            "purchase_year",
            F.year("order_purchase_timestamp")
        )
        .withColumn(
            "purchase_month",
            F.month("order_purchase_timestamp")
        )
)
```

---

# 19. Write Fact Data into Polaris

Example:

```python
(
    silver_df.writeTo(
        "olist.olist_silver.orders"
    )
    .using("iceberg")
    .partitionedBy(
        "purchase_year",
        "purchase_month"
    )
    .createOrReplace()
)
```

The exact partitioning strategy must make sense for the dataset.

Do not partition every small table unnecessarily.

Large transaction/event datasets are better candidates for partitioning than small dimensions.

---

# 20. Order Items Special Requirement

`order_items` does not directly contain the order purchase timestamp.

Therefore, derive the purchase period using the corresponding order.

Conceptually:

```text
order_items
     +
orders
     ↓
order_purchase_timestamp
     ↓
purchase_year
purchase_month
```

Example:

```python
items_df = spark.table(
    "olist_bronze.olist_order_items"
)

orders_df = spark.table(
    "olist_bronze.olist_orders"
)

orders_dates_df = (
    orders_df
        .select(
            "order_id",
            F.to_timestamp(
                "order_purchase_timestamp"
            ).alias("order_purchase_timestamp")
        )
)

silver_items_df = (
    items_df
        .join(
            orders_dates_df,
            on="order_id",
            how="left"
        )
        .withColumn(
            "purchase_year",
            F.year("order_purchase_timestamp")
        )
        .withColumn(
            "purchase_month",
            F.month("order_purchase_timestamp")
        )
)
```

The Silver `order_items` table can then be partitioned by:

```text
purchase_year
purchase_month
```

---

# 21. Required Pipeline Scripts

Create **seven independent PySpark scripts**.

Suggested filenames:

```text
01_customers_bronze_to_silver.py
02_products_bronze_to_silver.py
03_sellers_bronze_to_silver.py

04_orders_bronze_to_silver.py
05_order_items_bronze_to_silver.py
06_order_payments_bronze_to_silver.py
07_order_reviews_bronze_to_silver.py
```

Do **not** put all transformations into one giant script.

---

# 22. Why Separate Scripts?

The pipeline is deliberately divided by dataset because later these programs will become individual Airflow tasks.

For example:

```text
Airflow DAG
   │
   ├── process_customers
   ├── process_products
   ├── process_sellers
   │
   ├── process_orders
   ├── process_order_items
   ├── process_order_payments
   └── process_order_reviews
```

This provides several advantages.

### Independent Scheduling

Different datasets may arrive at different times.

For example:

```text
customers       → daily
products        → weekly
orders          → hourly/daily
payments        → hourly/daily
reviews         → daily
```

### Independent Retry

If `order_reviews` fails, Airflow should be able to retry only:

```text
order_reviews
```

rather than rebuilding every Silver table.

### Independent Monitoring

Airflow can report:

```text
customers       SUCCESS
products        SUCCESS
sellers         SUCCESS
orders          SUCCESS
order_items     SUCCESS
payments        SUCCESS
reviews         FAILED
```

### Easier Incremental Processing

Future datasets may arrive as batches.

Example:

```text
/olist/bronze/orders/orders_2026_10_07.csv
/olist/bronze/orders/orders_2026_10_08.csv
/olist/bronze/orders/orders_2026_10_09.csv
```

A dedicated orders pipeline can later be changed from:

```text
full refresh
```

to:

```text
incremental append / MERGE
```

without affecting the other pipelines.

---

# 23. Recommended Script Structure

Every script should approximately follow the same structure.

```python
# ---------------------------------------------------------
# 1. Imports
# ---------------------------------------------------------

from pyspark.sql import SparkSession
from pyspark.sql import functions as F


# ---------------------------------------------------------
# 2. Create Spark Session
# ---------------------------------------------------------

spark = (
    SparkSession.builder
        .appName("olist-customers-bronze-to-silver")
        .getOrCreate()
)


# ---------------------------------------------------------
# 3. Read Bronze
# ---------------------------------------------------------

# HDFS directly OR Hive Bronze table
source_df = ...


# ---------------------------------------------------------
# 4. Validate Source
# ---------------------------------------------------------

source_df.printSchema()
print("Source count:", source_df.count())


# ---------------------------------------------------------
# 5. Transform
# ---------------------------------------------------------

silver_df = ...


# ---------------------------------------------------------
# 6. Data Quality Checks
# ---------------------------------------------------------

# Check keys, nulls, duplicates, casts, etc.


# ---------------------------------------------------------
# 7. Write Iceberg Silver Table
# ---------------------------------------------------------

(
    silver_df.writeTo(
        "olist.olist_silver.<table>"
    )
    .using("iceberg")
    .createOrReplace()
)


# ---------------------------------------------------------
# 8. Validate Target
# ---------------------------------------------------------

spark.sql("""
    SELECT COUNT(*)
    FROM olist.olist_silver.<table>
""").show()


# ---------------------------------------------------------
# 9. Stop Spark
# ---------------------------------------------------------

spark.stop()
```

Students should maintain approximately the same structure across all seven programs.

---

# 24. Bronze vs Silver Responsibilities

## Bronze

Bronze represents the source as received.

```text
CSV
 ↓
HDFS
 ↓
Hive External Tables
```

Characteristics:

- raw or near-raw data
- source schema preserved
- minimal transformation
- useful for replay/reprocessing
- original files retained

## Silver

Silver represents cleaned and typed analytical data.

```text
Bronze
 ↓
Spark transformation
 ↓
Iceberg
 ↓
Polaris
 ↓
SeaweedFS S3
```

Characteristics:

- proper Spark/SQL data types
- cleaned strings
- parsed timestamps
- numeric columns stored numerically
- invalid/null values handled
- duplicate checks
- reusable analytical tables
- Iceberg table metadata
- suitable for later Gold transformations

---

# 25. Target Architecture

The completed architecture should look like:

```text
                Olist CSV Files
                      │
                      ▼
              HDFS Bronze Layer
                      │
       /olist/bronze/<dataset>/
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
      Direct Spark Read    Apache Hive
       for dimensions     External Tables
             │                 │
             │              Fact Data
             │                 │
             └────────┬────────┘
                      │
                      ▼
                  PySpark
               Transformations
                      │
                      ▼
               Apache Iceberg
                      │
                      ▼
               Polaris Catalog
                     olist
                      │
                      ▼
                olist_silver
                      │
                      ▼
              SeaweedFS S3
```

---

# 26. Dataset Processing Matrix

| Dataset | Bronze location | Spark source | Silver table |
|---|---|---|---|
| Customers | `/olist/bronze/customers/` | HDFS CSV | `olist.olist_silver.customers` |
| Products | `/olist/bronze/products/` | HDFS CSV | `olist.olist_silver.products` |
| Sellers | `/olist/bronze/sellers/` | HDFS CSV | `olist.olist_silver.sellers` |
| Orders | `/olist/bronze/orders/` | Hive | `olist.olist_silver.orders` |
| Order Items | `/olist/bronze/order_items/` | Hive | `olist.olist_silver.order_items` |
| Order Payments | `/olist/bronze/order_payments/` | Hive | `olist.olist_silver.order_payments` |
| Order Reviews | `/olist/bronze/order_reviews/` | Hive | `olist.olist_silver.order_reviews` |

---

# 27. Minimum Data-Quality Checks

Each script must perform appropriate checks before writing Silver data.

At minimum investigate:

- source row count
- target row count
- null primary/business keys
- duplicate business keys
- incorrect numeric conversions
- incorrect timestamp conversions
- blank strings
- negative monetary values where inappropriate
- malformed state codes
- orphan foreign keys where applicable

Example:

```python
null_ids = (
    silver_df
        .filter(F.col("customer_id").isNull())
        .count()
)

assert null_ids == 0, "customer_id contains NULL values"
```

Example duplicate check:

```python
duplicates = (
    silver_df
        .groupBy("customer_id")
        .count()
        .filter(F.col("count") > 1)
        .count()
)

assert duplicates == 0, "Duplicate customer_id detected"
```

A data-quality failure should cause the pipeline to fail rather than silently publishing incorrect Silver data.

---

# 28. Validation after Writing

Students must demonstrate that the Silver tables are registered in Polaris.

```sql
SHOW TABLES IN olist.olist_silver;
```

Expected tables:

```text
customers
products
sellers
orders
order_items
order_payments
order_reviews
```

Run counts:

```sql
SELECT COUNT(*)
FROM olist.olist_silver.customers;
```

```sql
SELECT COUNT(*)
FROM olist.olist_silver.orders;
```

Inspect sample records:

```sql
SELECT *
FROM olist.olist_silver.order_items
LIMIT 20;
```

---

# 29. Verify Iceberg / SeaweedFS Storage

Students must understand that Polaris stores/manages the catalog metadata while the actual Iceberg table data and metadata files are stored in the configured object storage.

Conceptually:

```text
Spark
   │
   ▼
Polaris REST Catalog
   │
   ▼
Iceberg metadata
   │
   ▼
SeaweedFS S3
```

Verify that Iceberg metadata/data files are being created in the configured SeaweedFS storage.

Do not manually copy Parquet files into the Silver S3 location.

The Iceberg writer must manage the Silver table storage.

---

# 30. Final Deliverables

Each team must submit:

### HDFS Bronze structure

```text
/olist/bronze/
├── customers/
├── products/
├── sellers/
├── orders/
├── order_items/
├── order_payments/
└── order_reviews/
```

### Hive

Database:

```text
olist_bronze
```

Seven external tables:

```text
olist_customers
olist_products
olist_sellers
olist_orders
olist_order_items
olist_order_payments
olist_order_reviews
```

### Polaris

Catalog:

```text
olist
```

Namespace:

```text
olist_silver
```

Seven Iceberg tables:

```text
customers
products
sellers
orders
order_items
order_payments
order_reviews
```

### PySpark

Seven independent scripts:

```text
01_customers_bronze_to_silver.py
02_products_bronze_to_silver.py
03_sellers_bronze_to_silver.py
04_orders_bronze_to_silver.py
05_order_items_bronze_to_silver.py
06_order_payments_bronze_to_silver.py
07_order_reviews_bronze_to_silver.py
```

---

# 31. Assignment Completion Criteria

The assignment is considered complete only when the team can demonstrate the complete flow:

```text
CSV
 ↓
HDFS Bronze
 ↓
Hive External Tables
 ↓
Spark Transformation
 ↓
Iceberg Silver
 ↓
Polaris Catalog
 ↓
SeaweedFS S3
```

and successfully query:

```sql
SELECT *
FROM olist.olist_silver.customers;
```

as well as:

```sql
SELECT *
FROM olist.olist_silver.orders;
```

The seven pipelines must remain separate because the next stage of the project will schedule these programs as independent tasks using **Apache Airflow**.
