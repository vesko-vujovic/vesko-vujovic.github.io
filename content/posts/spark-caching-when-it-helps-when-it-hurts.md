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

If you see `FileScan` like this on each print of execution plan then spark is scaning same file all over again.

```python

 +- FileScan json [amount#275,id#276L,provider_id#277L,timestamp#278,user_id#279L] 
     Batched: false, DataFilters: [], Format: JSON, 
     Location: InMemoryFileIndex(1 paths)[file:/home/vesko/Documents/Personal/Projects/dummy-data-rust/output/tr..., 
     PartitionFilters: [], PushedFilters: [], 
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
