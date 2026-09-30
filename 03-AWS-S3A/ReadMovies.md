Example code for accessing S3A

```python
import os
import socket

from pyspark.sql import SparkSession


# ---------------------------------------------------------
# Spark + SeaweedFS S3 configuration
# ---------------------------------------------------------

master_url = os.environ.get(
    "SPARK_MASTER",
    f"spark://{socket.gethostname()}:7077",
)

spark = (
    SparkSession.builder
    .appName("D30-MovieLens-SeaweedFS")
    .master(master_url)

    # SeaweedFS S3 configuration
    .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:9001")
    .config("spark.hadoop.fs.s3a.access.key", "team")
    .config("spark.hadoop.fs.s3a.secret.key", "team1234")
    .config("spark.hadoop.fs.s3a.path.style.access", "true")
    .config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false")
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")

    .getOrCreate()
)

spark.sparkContext.setLogLevel("WARN")


# ---------------------------------------------------------
# Read Bronze data
# ---------------------------------------------------------

movies_path = "s3a://datalake/movielens/bronze/movies/movies.csv"

movies_df = (
    spark.read
    .option("header", "true")
    .option("inferSchema", "true")
    .csv(movies_path)
)


# ---------------------------------------------------------
# Verify
# ---------------------------------------------------------

movies_df.printSchema()
movies_df.show(20, truncate=False)

print("Number of movies:", movies_df.count())

```
