---
title: "Apache Spark: Why JSON isn't ideal format for your spark job"
date: 2024-08-11T15:06:41+02:00
draft: true
tags:
  - apache-spark
  - data-engineering
  - big-data
  - data-processing
  - file-formats
cover:
  image: "/posts/json-vs-parquet/cover-json-parquet.jpg"
  alt: "json-vs-parquet"
  caption: "json-vs-parquet"
---

![json-vs-parquet](/posts/json-vs-parquet/cover-json-parquet.jpg)


# Introduction

Hi there 👋! In this blog post, we will explore why **JSON** is not suitable as a big data file format. We'll compare it to the widely used **Parquet** format and dig deep to demonstrate, through examples, how the JSON format can significantly degrade the performance of your data processing jobs.
JSON (JavaScript Object Notation) is a popular and versatile data format, but it has limitations when dealing with large-scale data operations. On the other hand, Parquet, an open-source columnar storage format, has become the go-to choice for big data applications.

# Short comparison of JSON vs Parquet

Let's explore the strengths of both JSON and Parquet file formats by looking at them from a few different perspectives.

* ***Basic Concepts***
  * JSON
    * Text-based, human-readable format
    * Originally derived from JavaScript, now language-independent
    * Uses key-value pairs and arrays to represent data
  * Parquet
    * Binary columnar storage format
    * Developed by Apache for the Hadoop ecosystem
    * Optimized for efficient storage and performance on large datasets

* ***Data Structure***
  * JSON
    * Hierarchical structure
    * Flexible, schema-less format
    * Supports nested data structures
    * Each record is self-contained
  * Parquet
    * Columnar structure
    * Strongly typed
    * Supports complex nested data structures
    * Data is organized by columns rather than rows

* ***Storage Efficiency***
  * JSON
    * Typically larger file sizes
    * Repetitive field names
    * No built-in compression
    * Can be compressed externally
  * Parquet
    * Highly efficient storage
    * Built-in compression
    * Supports multiple compression codecs (e.g., Snappy, Gzip, LZO)
    * Can significantly reduce storage costs for large datasets

* ***Read/Write Performance***
  * JSON
    * Fast writes (simple appending of records)
    * Slower reads, especially for large datasets
    * Requires parsing entire records even when only specific fields are needed
  * Parquet
    * Slower writes (due to columnar storage and compression)
    * Much faster reads, especially for analytical queries
    * Supports column pruning and predicate pushdown for efficient querying

* ***Schema Evolution***
  * JSON
    * Easily accommodates schema changes
    * New fields can be added without affecting existing data
  * Parquet
    * Supports schema evolution, but with limitations
    * Can add new columns or change nullability of existing columns
    * Renaming or deleting columns is more challenging

* ***Use Cases***
  * JSON
    * Web APIs and data interchange
    * Document databases (e.g., MongoDB)
    * Logging and event data
    * Scenarios requiring human-readable data
  * Parquet
    * Big data analytics and data warehousing
    * Machine learning model training on large datasets
    * Business intelligence and reporting systems
    * Scenarios prioritizing query performance and storage efficiency

* ***Compatibility and Ecosystem Support***
  * JSON
    * Universally supported across programming languages and platforms
    * Native support in web browsers and many NoSQL databases
    * Easy to work with for developers
    * Doesn't require library for reading and writing
  * Parquet
    * Strong support in big data ecosystems (Hadoop, Spark, Hive, Impala)
    * Strong support in cloud data warehouses (e.g., Amazon Athena, Google BigQuery)
    * Requires specific libraries or tools for reading/writing


# Show time 🌟

In this section, we will showcase all the details about how JSON impacts your Apache Spark job. We will also demonstrate how Apache Spark behaves with the same data written in Parquet on a dataset of ***~60GB***. We will show all the details using [spark web ui](https://spark.apache.org/docs/latest/web-ui.html)


## Execution time

When running the code to read a JSON dataset of approximately ***60GB of transactions***, followed by grouping and summing all transactions by user, the entire job takes _***4.7 minutes to execute***_.


{{< highlight scala "linenos=table,linenostart=1" >}}

    val spark = SparkSession
      .builder()
      .master("spark://localhost:7077")
      .config("spark.driver.bindAddress", "127.0.0.1")
      .config("spark.driver.host", "127.0.0.1")
      .config("spark.memory.offHeap.enabled", "true")
      .config("spark.sql.shuffle.partitions", 500)
      .config("spark.eventLog.enabled", "true")
      .config("park.eventLog.dir", "/tmp/spark-events")
      .config("spark.memory.offHeap.size", "2g")
      .appName("SparkJsonReadTest")
      .getOrCreate()

    val df = spark.read
      .json(
        "~/Downloads/data/transactions.jsonl")

    val sumByUserId = df.groupBy("user_id").sum("amount")
    
    sumByUserId.show()


{{< / highlight >}}


![json-execution](/posts/json-vs-parquet/json_execution_time.png)  

show inout for stage in json job and parquet job
show images for number of tasks 
show shufle write data shufle
mention partition pruning and the fact that with json we cannot skip data

## Input data for stage 

## Number of tasks



# Final thoughts





{{< chat apache-spark-json-vs-parquet>}}