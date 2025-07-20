---
title: "💡 Spark Caching: When It Helps and When It Hurts Your Performance 🔧"
date: 2025-07-20T08:06:41+02:00
draft: false
tags:
  - apache-spark
  - data-engineering
  - caching
  - performance-optimization
  - big-data
  - spark-optimization
cover:
  image: "/posts/spark-caching-performance/spark_caching_cover.png"
  alt: "spark-caching-performance"
  caption: "spark-caching-performance"
---

Ever had a Spark job that keeps re-reading the same data over and over? You might need caching. But cache at the wrong time, and you'll actually slow things down.
Here's when caching helps, when it doesn't, and how to use it right. This blog will be short but sweet!

# 🔍 What is Caching?
Think of caching like keeping your frequently used files on your desk instead of walking to the filing cabinet every time you need them.

## Here's what happens without caching in apache spark:

``` python

# Spark reads the file from disk every single time
df = spark.read.parquet("large_file.parquet")

result1 = df.filter(col("status") == "active").count()  # Reads file
result2 = df.groupBy("category").count()                # Reads file AGAIN
result3 = df.select("id", "name").collect()             # Reads file AGAIN

```

If you see `FileScan` without `InMemoryTableScan` section like this on each print of execution plan then spark is scaning same file all over again.

```python

== Physical Plan ==
*(1) Filter (isnotnull(amount#58) AND (amount#58 > 5000.0))
+- FileScan json [amount#58,id#59L,provider_id#60L,timestamp#61,user_id#62L] Batched: false, DataFilters: [isnotnull(amount#58), 
 (amount#58 > 5000.0)], Format: JSON, 
  Location: InMemoryFileIndex(1 paths)[file:/home/vesko/Documents/Personal/Projects/dummy-data-rust/output/tr..., 
  PartitionFilters: [], PushedFilters: [IsNotNull(amount), GreaterThan(amount,5000.0)], 
  ReadSchema: struct<amount:double,id:bigint,provider_id:bigint,timestamp:string,user_id:bigint>

```



With caching:

```python

# Read once, keep in memory
df = spark.read.parquet("large_file.parquet").cache()
result1 = df.filter(col("status") == "active").count()  # Reads file + caches
result2 = df.groupBy("category").count()                # Uses cached data
result3 = df.select("id", "name").collect()             # Uses cached data
```

You will see something like this:

```python

  == Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- Filter (isnotnull(amount#58) AND (amount#58 > 100000.0))
   +- InMemoryTableScan [amount#58, id#59L, provider_id#60L, timestamp#61, user_id#62L], [isnotnull(amount#58), (amount#58 > 100000.0)]
         +- InMemoryRelation [amount#58, id#59L, provider_id#60L, timestamp#61, user_id#62L], StorageLevel(disk, memory, deserialized, 1 replicas)
               +- FileScan json [amount#58,id#59L,provider_id#60L,timestamp#61,user_id#62L] Batched: false, DataFilters: [], Format: JSON, Location: InMemoryFileIndex(1 paths)[file:/home/vesko/Documents/Personal/Projects/dummy-data-rust/output/tr..., PartitionFilters: [], PushedFilters: [], ReadSchema: struct<amount:double,id:bigint,provider_id:bigint,timestamp:string,user_id:bigint>

```

From these execeution plan we can see that:

- **Plan 1:** Reads ALL data from file → loads into memory → then filters (AQE will be active in the next phase of exectuion)
- **Plan 2:** Applies filters while reading the file, so less data enters Spark + AQE is enabled and **WholeCodeGeneration** is active (optimization at max)

### The difference? 

Instead of reading scaning from disk three times, you read once and reuse the data from memory. This can turn a **10-minute** job into a **2-minute** job.

# ✅ When to Cache (The Good Stuff)

You're Using the Same Data Multiple Times
This is the most common scenario. If you're running several operations on the same DataFrame, cache it.

```python
# Perfect use case for caching
sales_data = spark.read.csv("sales.csv").cache()

# All of these reuse the cached data
daily_sales = sales_data.groupBy("date").sum("amount")
top_products = sales_data.groupBy("product").count()
regional_summary = sales_data.groupBy("region").agg(avg("amount"))
```

## Expensive Operations You'll Need Again
Got a complex join or heavy transformation? Cache the result if you'll use it multiple times.

```python

# Expensive join operation
enriched_data = transactions \
    .join(customers, "customer_id") \
    .join(products, "product_id") \
    .withColumn("profit", col("revenue") - col("cost")) \
    .cache()  # Cache this expensive result

# Now use it multiple times without recomputing
high_value = enriched_data.filter(col("profit") > 1000)
monthly_trends = enriched_data.groupBy("month").sum("profit")

```

## Interactive Data Exploration
Working in a Jupyter notebook? Cache your base dataset so you can explore it quickly.