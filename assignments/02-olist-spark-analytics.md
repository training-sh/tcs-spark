# Olist Silver-to-Gold Spark Analytics Assignment


Code samples with template provided with a consideration that team members are from admin background, not used with python/sql/data analytical background. 
You may need to repurpose the code for the use case



## Objective

Continue from the completed Bronze-to-Silver Olist pipeline.

The previous assignment produced cleaned Iceberg Silver tables in Apache Polaris:

```text
olist.olist_silver.customers
olist.olist_silver.products
olist.olist_silver.sellers
olist.olist_silver.orders
olist.olist_silver.order_items
olist.olist_silver.order_payments
olist.olist_silver.order_reviews
```

In this assignment, use these Silver tables as the only source for analytical processing.

The objective is to build reusable analytical datasets using:

- Apache Spark
- Spark SQL
- Apache Iceberg
- Apache Polaris
- SeaweedFS S3
- MySQL

The final architecture will be:

```text
Olist Silver Iceberg Tables
        │
        ▼
     Spark SQL
        │
        ▼
Analytical Transformations
        │
        ├──────────────► Polaris / Iceberg Gold
        │                    olist.olist_gold
        │
        └──────────────► MySQL
                             database: olist
```

---

# 1. Input Layer

Do **not** read the original CSV files again.

Do **not** read the HDFS Bronze directories directly.

All analytics must use the Silver Iceberg tables created in the previous assignment.

Input tables:

```text
olist.olist_silver.customers
olist.olist_silver.products
olist.olist_silver.sellers
olist.olist_silver.orders
olist.olist_silver.order_items
olist.olist_silver.order_payments
olist.olist_silver.order_reviews
```

For example:

```python
customers_df = spark.table(
    "olist.olist_silver.customers"
)
```

or:

```python
result_df = spark.sql("""
    SELECT *
    FROM olist.olist_silver.customers
""")
```

---

# 2. Create the Gold Namespace

The Polaris catalog already exists:

```text
olist
```

Create a new namespace for analytical data:

```text
olist_gold
```

The resulting structure should be:

```text
olist
├── olist_silver
│   ├── customers
│   ├── products
│   ├── sellers
│   ├── orders
│   ├── order_items
│   ├── order_payments
│   └── order_reviews
│
└── olist_gold
```

Create the namespace:

```sql
CREATE NAMESPACE IF NOT EXISTS olist.olist_gold;
```

Verify:

```sql
SHOW NAMESPACES IN olist;
```

---

# 3. Gold Layer Purpose

The Silver layer contains cleaned business entities and transaction data.

The Gold layer should contain **analytical results prepared for dashboards, reports, business users, and downstream applications**.

For example:

```text
Silver:
one row per customer
one row per order
one row per order item
one row per payment

Gold:
sales by state
sales by category
customer spending summary
seller performance
monthly sales trends
payment analysis
review analytics
```

Gold tables should therefore normally have an analytical grain rather than simply copying Silver tables.

---

# 4. Important Analytical Rule

Before writing every analytical query, identify the **grain** of the result.

Examples:

```text
one row per customer state
one row per product category
one row per month
one row per seller state
one row per payment type
one row per customer
```

Students must include the grain as a comment in their Spark program.

Example:

```python
# Grain:
# One row per customer_state
```

This is important because joining multiple one-to-many tables incorrectly can multiply rows and produce incorrect revenue totals.

For example:

```text
orders
  ↓
order_items    one-to-many

orders
  ↓
payments       one-to-many
```

Joining `order_items` and `order_payments` directly through orders can multiply rows.

Where necessary, aggregate each dataset to the appropriate grain **before joining**.

---

# 5. Required Gold Analytical Tables

Create the following analytical Iceberg tables under:

```text
olist.olist_gold
```

Required tables:

```text
customer_state_summary
top_customers
payment_type_summary
large_order_summary
seller_state_performance
product_category_performance
review_status_summary
customer_state_freight
city_order_summary
monthly_sales_summary
```

These tables form the minimum Gold analytical layer for this assignment.

---

# 6. Gold Table 1 — Customer State Summary

Create:

```text
olist.olist_gold.customer_state_summary
```

## Grain

```text
One row per customer_state
```

Calculate:

- customer state
- customer record count
- unique customer count

Example output:

| customer_state | customer_count | unique_customers |
|---|---:|---:|
| SP | ... | ... |
| RJ | ... | ... |
| MG | ... | ... |

Reference Spark SQL pattern:

```python
customer_state_df = spark.sql("""
    SELECT
        customer_state,
        COUNT(*) AS customer_count,
        COUNT(DISTINCT customer_unique_id) AS unique_customers
    FROM olist.olist_silver.customers
    GROUP BY customer_state
""")
```

Write the result:

```python
(
    customer_state_df
        .writeTo(
            "olist.olist_gold.customer_state_summary"
        )
        .using("iceberg")
        .createOrReplace()
)
```

---

# 7. Gold Table 2 — Top Customers

Create:

```text
olist.olist_gold.top_customers
```

## Grain

```text
One row per customer_unique_id
```

Use:

```text
customers
orders
order_payments
```

Calculate:

- customer unique ID
- number of distinct orders
- total amount spent
- average payment per order
- first order date
- latest order date

Example target columns:

```text
customer_unique_id
order_count
total_spent
avg_order_payment
first_order_date
latest_order_date
```

Order customers by total spending descending.

Students must avoid incorrect aggregation caused by duplicate joins.

---

# 8. Gold Table 3 — Payment Type Summary

Create:

```text
olist.olist_gold.payment_type_summary
```

## Grain

```text
One row per payment_type
```

Calculate:

- payment type
- payment count
- minimum payment
- maximum payment
- average payment
- total payment amount

Exclude:

```text
not_defined
```

Example analytical pattern:

```sql
SELECT
    payment_type,
    COUNT(*) AS payment_count,
    MIN(payment_value) AS min_payment,
    MAX(payment_value) AS max_payment,
    AVG(payment_value) AS avg_payment,
    SUM(payment_value) AS total_payment
FROM olist.olist_silver.order_payments
WHERE payment_type <> 'not_defined'
GROUP BY payment_type;
```

---

# 9. Gold Table 4 — Large Order Summary

Create:

```text
olist.olist_gold.large_order_summary
```

## Grain

```text
One row per order_id
```

Use:

```text
orders
order_items
```

Include only delivered orders.

Calculate:

- order ID
- customer ID
- item count
- item value
- freight value
- total order value
- purchase timestamp

Where:

```text
total_order_value =
item_value + freight_value
```

Keep only orders containing at least:

```text
5 items
```

Example columns:

```text
order_id
customer_id
item_count
item_value
freight_value
total_order_value
order_purchase_timestamp
```

---

# 10. Gold Table 5 — Seller State Performance

Create:

```text
olist.olist_gold.seller_state_performance
```

## Grain

```text
One row per seller_state
```

Use:

```text
sellers
order_items
```

Calculate:

- seller state
- distinct seller count
- total items sold
- distinct order count
- average item price
- total item revenue

Example columns:

```text
seller_state
seller_count
order_count
items_sold
avg_item_price
item_revenue
```

Students may additionally rank seller states by revenue.

---

# 11. Gold Table 6 — Product Category Performance

Create:

```text
olist.olist_gold.product_category_performance
```

## Grain

```text
One row per product_category_name
```

Use:

```text
products
order_items
```

The product category translation dataset was intentionally excluded from the earlier pipeline.

Therefore, use the native:

```text
product_category_name
```

Calculate:

- product category
- distinct orders
- items sold
- distinct products sold
- average item price
- minimum item price
- maximum item price
- total revenue

Example:

```text
product_category_name
order_count
items_sold
product_count
avg_price
min_price
max_price
revenue
```

Exclude null product categories.

---

# 12. Gold Table 7 — Review Status Summary

Create:

```text
olist.olist_gold.review_status_summary
```

## Grain

```text
One row per order_status
```

Use:

```text
orders
order_reviews
```

Calculate:

- order status
- number of reviews
- minimum review score
- maximum review score
- average review score

Example:

```text
order_status
review_count
min_review_score
max_review_score
avg_review_score
```

Only retain statuses with sufficient review volume.

For this assignment use:

```text
at least 10 reviews
```

---

# 13. Gold Table 8 — Customer State Freight Summary

Create:

```text
olist.olist_gold.customer_state_freight
```

## Grain

```text
One row per customer_state
```

Use:

```text
customers
orders
order_items
```

Include delivered orders.

Calculate:

- state
- distinct order count
- item count
- minimum freight
- maximum freight
- average freight
- total freight

Example:

```text
customer_state
order_count
item_count
min_freight
max_freight
avg_freight
total_freight
```

---

# 14. Gold Table 9 — City Order Summary

Create:

```text
olist.olist_gold.city_order_summary
```

## Grain

```text
One row per customer_state + customer_city
```

Use:

```text
customers
orders
```

For delivered orders calculate:

- customer state
- customer city
- distinct delivered orders
- unique customers

Example:

```text
customer_state
customer_city
delivered_order_count
unique_customers
```

Keep cities having at least:

```text
500 delivered orders
```

---

# 15. Gold Table 10 — Monthly Sales Summary

Create:

```text
olist.olist_gold.monthly_sales_summary
```

This will be one of the most important dashboard datasets.

## Grain

```text
One row per purchase year + purchase month
```

Use:

```text
orders
order_items
```

Include useful sales metrics:

- purchase year
- purchase month
- distinct orders
- items sold
- item revenue
- freight revenue
- combined revenue
- average order value

Example target:

```text
purchase_year
purchase_month
order_count
items_sold
item_revenue
freight_revenue
total_revenue
avg_order_value
```

Example pseudocode:

```python
monthly_sales_df = spark.sql("""
    WITH order_values AS
    (
        SELECT
            o.order_id,
            YEAR(o.order_purchase_timestamp) AS purchase_year,
            MONTH(o.order_purchase_timestamp) AS purchase_month,
            SUM(i.price) AS item_revenue,
            SUM(i.freight_value) AS freight_revenue,
            SUM(i.price + i.freight_value) AS total_order_value,
            COUNT(*) AS item_count
        FROM olist.olist_silver.orders o
        INNER JOIN olist.olist_silver.order_items i
            ON o.order_id = i.order_id
        GROUP BY
            o.order_id,
            YEAR(o.order_purchase_timestamp),
            MONTH(o.order_purchase_timestamp)
    )

    SELECT
        purchase_year,
        purchase_month,
        COUNT(*) AS order_count,
        SUM(item_count) AS items_sold,
        SUM(item_revenue) AS item_revenue,
        SUM(freight_revenue) AS freight_revenue,
        SUM(total_order_value) AS total_revenue,
        AVG(total_order_value) AS avg_order_value
    FROM order_values
    GROUP BY
        purchase_year,
        purchase_month
""")
```

Then write:

```python
(
    monthly_sales_df
        .writeTo(
            "olist.olist_gold.monthly_sales_summary"
        )
        .using("iceberg")
        .createOrReplace()
)
```

---

# 16. Verify Gold Tables

Run:

```sql
SHOW TABLES IN olist.olist_gold;
```

Expected minimum result:

```text
customer_state_summary
top_customers
payment_type_summary
large_order_summary
seller_state_performance
product_category_performance
review_status_summary
customer_state_freight
city_order_summary
monthly_sales_summary
```

Test individual tables.

Example:

```sql
SELECT *
FROM olist.olist_gold.product_category_performance
ORDER BY revenue DESC
LIMIT 10;
```

Example:

```sql
SELECT *
FROM olist.olist_gold.monthly_sales_summary
ORDER BY purchase_year, purchase_month;
```

---

# 17. Why Gold Data Must Be Persisted

Do not treat every Spark analytical result as a temporary DataFrame.

Frequently used business results should be persisted because:

- dashboards should not repeatedly execute expensive joins
- reports can query precomputed analytical tables
- results become reusable across applications
- analytical definitions remain consistent
- Iceberg provides reliable table management
- historical results can later use Iceberg snapshot capabilities
- downstream MySQL publishing becomes simpler

The Gold layer therefore acts as the reusable analytical layer.

---

# 18. MySQL Publishing Layer

In addition to Polaris Gold tables, selected analytical results must be published to MySQL.

Create the MySQL database:

```sql
CREATE DATABASE IF NOT EXISTS olist;
```

The MySQL database represents the **serving layer**.

Conceptually:

```text
Polaris Gold / Iceberg
        │
        ▼
      Spark
        │
        ▼
      MySQL
        │
        ▼
Streamlit / BI / Reporting
```

MySQL should contain compact analytical results rather than the complete raw Olist dataset.

---

# 19. Required MySQL Tables

Create the following tables in database:

```text
olist
```

Required tables:

```text
sales_monthly
sales_by_category
sales_by_state
seller_performance
customer_summary
payment_summary
review_summary
```

These tables should be populated from the Gold analytical results.

---

# 20. MySQL Table — `sales_monthly`

Source:

```text
olist.olist_gold.monthly_sales_summary
```

Target:

```text
olist.sales_monthly
```

Suggested columns:

```text
purchase_year
purchase_month
order_count
items_sold
item_revenue
freight_revenue
total_revenue
avg_order_value
```

This table can directly support dashboard charts such as:

```text
Monthly Revenue Trend
Monthly Order Growth
Monthly Item Sales
Average Order Value
```

---

# 21. MySQL Table — `sales_by_category`

Source:

```text
olist.olist_gold.product_category_performance
```

Target:

```text
olist.sales_by_category
```

Suggested columns:

```text
product_category_name
order_count
items_sold
product_count
avg_price
revenue
```

Possible dashboard use:

```text
Top Selling Categories
Highest Revenue Categories
Average Product Price
Category Contribution
```

---

# 22. MySQL Table — `sales_by_state`

Use relevant Gold state-level analytics.

Target:

```text
olist.sales_by_state
```

Students should derive at least:

```text
customer_state
order_count
customer_count
item_count
revenue
freight_value
```

The exact query may combine existing Gold analytical results or create an additional Gold table if required.

Possible dashboard use:

```text
Regional Sales
Regional Order Volume
Regional Customer Distribution
Freight by State
```

---

# 23. MySQL Table — `seller_performance`

Source:

```text
olist.olist_gold.seller_state_performance
```

Target:

```text
olist.seller_performance
```

Suggested columns:

```text
seller_state
seller_count
order_count
items_sold
avg_item_price
item_revenue
```

---

# 24. MySQL Table — `customer_summary`

Source:

```text
olist.olist_gold.top_customers
```

Target:

```text
olist.customer_summary
```

Suggested columns:

```text
customer_unique_id
order_count
total_spent
avg_order_payment
first_order_date
latest_order_date
```

This table should not necessarily contain only the top 10 customers.

The Gold table may contain the complete customer analytical summary, while dashboards can select:

```sql
ORDER BY total_spent DESC
LIMIT 10;
```

---

# 25. MySQL Table — `payment_summary`

Source:

```text
olist.olist_gold.payment_type_summary
```

Target:

```text
olist.payment_summary
```

Suggested columns:

```text
payment_type
payment_count
min_payment
max_payment
avg_payment
total_payment
```

---

# 26. MySQL Table — `review_summary`

Source:

```text
olist.olist_gold.review_status_summary
```

Target:

```text
olist.review_summary
```

Suggested columns:

```text
order_status
review_count
min_review_score
max_review_score
avg_review_score
```

---

# 27. Writing Spark DataFrames to MySQL

Use Spark JDBC.

Reference pattern:

```python
mysql_url = "jdbc:mysql://<mysql-host>:3306/olist"

mysql_properties = {
    "user": "<username>",
    "password": "<password>",
    "driver": "com.mysql.cj.jdbc.Driver"
}
```

Example:

```python
monthly_sales_df = spark.table(
    "olist.olist_gold.monthly_sales_summary"
)

(
    monthly_sales_df.write
        .mode("overwrite")
        .jdbc(
            url=mysql_url,
            table="sales_monthly",
            properties=mysql_properties
        )
)
```

Equivalent JDBC-style syntax is also acceptable:

```python
(
    monthly_sales_df.write
        .format("jdbc")
        .option(
            "url",
            "jdbc:mysql://<mysql-host>:3306/olist"
        )
        .option(
            "dbtable",
            "sales_monthly"
        )
        .option(
            "user",
            "<username>"
        )
        .option(
            "password",
            "<password>"
        )
        .option(
            "driver",
            "com.mysql.cj.jdbc.Driver"
        )
        .mode("overwrite")
        .save()
)
```

Credentials must not be hard-coded into source code in a real production implementation.

For the lab, use the configuration approach provided by the instructor.

---

# 28. Validate MySQL Results

After publishing, validate from MySQL.

Example:

```sql
USE olist;
```

```sql
SHOW TABLES;
```

Expected minimum tables:

```text
sales_monthly
sales_by_category
sales_by_state
seller_performance
customer_summary
payment_summary
review_summary
```

Check:

```sql
SELECT *
FROM sales_monthly
ORDER BY purchase_year, purchase_month;
```

```sql
SELECT *
FROM sales_by_category
ORDER BY revenue DESC
LIMIT 10;
```

```sql
SELECT *
FROM seller_performance
ORDER BY item_revenue DESC;
```

---

# 29. Row-Count Validation

Students must compare the Gold result and MySQL result.

Example:

```python
gold_count = spark.table(
    "olist.olist_gold.monthly_sales_summary"
).count()

print(gold_count)
```

Then:

```sql
SELECT COUNT(*)
FROM olist.sales_monthly;
```

The numbers should match when the entire Gold table is published.

---

# 30. Currency Handling

Currency calculations must use appropriate numeric types.

Do not convert currency values into formatted strings during analytical processing.

For example:

Correct:

```text
total_revenue = 1234567.89
```

Incorrect:

```text
total_revenue = "1,234,567.89 INR"
```

Formatting belongs in the presentation/dashboard layer.

Round displayed results to two decimal places where appropriate, but avoid unnecessary rounding during intermediate calculations.

---

# 31. Required Spark Scripts

Create independent analytical scripts.

Suggested structure:

```text
01_customer_analytics.py
02_product_analytics.py
03_sales_analytics.py
04_seller_analytics.py
05_payment_analytics.py
06_review_analytics.py
07_publish_gold_to_mysql.py
```

Alternative separation is acceptable if responsibilities remain clear.

Do not build the complete project as one very large Spark script.

---

# 32. Suggested Script Responsibilities

## `01_customer_analytics.py`

Create:

```text
customer_state_summary
top_customers
city_order_summary
```

## `02_product_analytics.py`

Create:

```text
product_category_performance
```

## `03_sales_analytics.py`

Create:

```text
large_order_summary
customer_state_freight
monthly_sales_summary
```

## `04_seller_analytics.py`

Create:

```text
seller_state_performance
```

## `05_payment_analytics.py`

Create:

```text
payment_type_summary
```

## `06_review_analytics.py`

Create:

```text
review_status_summary
```

## `07_publish_gold_to_mysql.py`

Read Gold Iceberg tables and publish required results into:

```text
MySQL → olist
```

---

# 33. Recommended Gold Pipeline Pattern

Each analytical script should follow approximately this structure:

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F


spark = (
    SparkSession.builder
        .appName("olist-gold-analytics")
        .getOrCreate()
)


# ---------------------------------------------------------
# 1. Read Silver
# ---------------------------------------------------------

orders_df = spark.table(
    "olist.olist_silver.orders"
)


# ---------------------------------------------------------
# 2. Perform analytical transformation
# ---------------------------------------------------------

result_df = ...


# ---------------------------------------------------------
# 3. Validate
# ---------------------------------------------------------

result_df.printSchema()
result_df.show(20, truncate=False)

print(
    "Result row count:",
    result_df.count()
)


# ---------------------------------------------------------
# 4. Write Gold Iceberg table
# ---------------------------------------------------------

(
    result_df.writeTo(
        "olist.olist_gold.<table_name>"
    )
    .using("iceberg")
    .createOrReplace()
)


# ---------------------------------------------------------
# 5. Verify Gold
# ---------------------------------------------------------

spark.sql("""
    SELECT *
    FROM olist.olist_gold.<table_name>
    LIMIT 10
""").show(truncate=False)


spark.stop()
```

---

# 34. Do Not Create Gold as Simple Silver Copies

The following is **not** acceptable:

```python
customers_df.writeTo(
    "olist.olist_gold.customers"
)
```

Gold should provide analytical value.

Correct idea:

```text
customers
orders
payments
      ↓
aggregation
      ↓
customer spending analytics
      ↓
olist.olist_gold.top_customers
```

Gold tables must answer a meaningful business question.

---

# 35. Business Questions the Gold Layer Should Answer

At the end of the assignment, the Gold layer should make it easy to answer questions such as:

```text
Which states have the largest customer base?

Which customers spend the most?

Which payment methods generate the most transaction value?

Which product categories generate the highest revenue?

Which seller states contribute the most sales?

How are sales changing month by month?

Which cities produce the largest volume of delivered orders?

What is the average review score for delivered and cancelled orders?

How much freight is associated with each customer state?

Which orders have unusually large numbers of items?
```

These questions should no longer require repeatedly rebuilding complex joins from Silver.

---

# 36. Serving-Layer Purpose

Polaris Gold and MySQL have different responsibilities.

## Polaris Gold

```text
olist.olist_gold
```

Used for:

- durable analytical datasets
- larger analytical tables
- Iceberg-based processing
- future Spark analysis
- recomputation
- historical analytics
- downstream batch processing

## MySQL

```text
olist
```

Used for:

- dashboard serving
- Streamlit queries
- lightweight reporting
- small/medium aggregated analytical results
- fast application queries

Therefore:

```text
Silver
   ↓
Spark
   ↓
Gold Iceberg
   ↓
MySQL serving tables
   ↓
Streamlit
```

---

# 37. Future Airflow Design

These scripts will later be scheduled using Apache Airflow.

Conceptually:

```text
                  Silver Ready
                       │
                       ▼
              customer_analytics
                       │
              product_analytics
                       │
               sales_analytics
                       │
              seller_analytics
                       │
              payment_analytics
                       │
              review_analytics
                       │
                       ▼
                 Gold Ready
                       │
                       ▼
              publish_to_mysql
                       │
                       ▼
                Dashboard Ready
```

The MySQL publishing task must run only after the required Gold analytical tables have been successfully created.

---

# 38. Pipeline Dependency Example

A future Airflow DAG could conceptually contain:

```text
silver_customers ───────┐
silver_orders ──────────┼──► customer_analytics
silver_payments ────────┘

silver_products ────────┐
silver_order_items ─────┼──► product_analytics
                        │
silver_orders ──────────┼──► sales_analytics
                        │
silver_sellers ─────────┼──► seller_analytics
                        │
silver_reviews ─────────┴──► review_analytics

customer_analytics ─────┐
product_analytics ──────┤
sales_analytics ────────┤
seller_analytics ───────┼──► publish_to_mysql
payment_analytics ──────┤
review_analytics ───────┘
```

This is another reason analytical pipelines should remain separated.

---

# 39. Minimum Validation Requirements

For every Gold table:

- verify input tables exist
- verify output grain
- inspect schema
- inspect sample rows
- check row count
- check nulls in important grouping columns
- check duplicate rows at the expected grain
- validate aggregate values
- verify the Iceberg table exists in Polaris

For MySQL:

- verify database exists
- verify table exists
- verify row count
- inspect sample rows
- compare row count against Gold
- verify important numeric columns
- verify no unintended string conversion

---

# 40. Final Required Architecture

The completed project should now have the following flow:

```text
                         OLIST DATA PIPELINE

CSV Source
    │
    ▼
HDFS Bronze
    │
    ▼
Hive External Tables
    │
    ▼
Spark Bronze-to-Silver
    │
    ▼
Polaris Catalog: olist
    │
    ├── olist_silver
    │       │
    │       ├── customers
    │       ├── products
    │       ├── sellers
    │       ├── orders
    │       ├── order_items
    │       ├── order_payments
    │       └── order_reviews
    │
    │
    ▼
Spark SQL Analytics
    │
    ▼
Polaris Catalog: olist
    │
    └── olist_gold
            │
            ├── customer_state_summary
            ├── top_customers
            ├── payment_type_summary
            ├── large_order_summary
            ├── seller_state_performance
            ├── product_category_performance
            ├── review_status_summary
            ├── customer_state_freight
            ├── city_order_summary
            └── monthly_sales_summary
                    │
                    ▼
                  Spark
                    │
                    ▼
             MySQL Database
                  olist
                    │
                    ├── sales_monthly
                    ├── sales_by_category
                    ├── sales_by_state
                    ├── seller_performance
                    ├── customer_summary
                    ├── payment_summary
                    └── review_summary
                            │
                            ▼
                     Streamlit / BI
```

---

# 41. Final Deliverables

Each team must submit the following.

## Polaris Gold Namespace

```text
olist.olist_gold
```

## Gold Iceberg Tables

```text
customer_state_summary
top_customers
payment_type_summary
large_order_summary
seller_state_performance
product_category_performance
review_status_summary
customer_state_freight
city_order_summary
monthly_sales_summary
```

## MySQL Database

```text
olist
```

## MySQL Tables

```text
sales_monthly
sales_by_category
sales_by_state
seller_performance
customer_summary
payment_summary
review_summary
```

## Spark Programs

Approximately:

```text
01_customer_analytics.py
02_product_analytics.py
03_sales_analytics.py
04_seller_analytics.py
05_payment_analytics.py
06_review_analytics.py
07_publish_gold_to_mysql.py
```

---

# 42. Completion Criteria

The assignment is complete when the team can demonstrate:

```text
Polaris Silver
      ↓
Spark SQL Analytics
      ↓
Polaris Gold / Iceberg
      ↓
Spark JDBC
      ↓
MySQL
```

and successfully execute queries such as:

```sql
SELECT *
FROM olist.olist_gold.monthly_sales_summary
ORDER BY purchase_year, purchase_month;
```

and from MySQL:

```sql
SELECT *
FROM olist.sales_monthly
ORDER BY purchase_year, purchase_month;
```

The analytical results in MySQL should be ready to be consumed directly by the Streamlit dashboard developed in the next phase of the project.
