---
layout: post
title:  "Apache Spark: Dataset vs Dataframe - The Tortoise and Hare"
tags: [big-data, data-engineering, apache-spark]
---

![spark](/assets/img/spark_cover.png)

# Introduction
Hi, welcome to my first blog post. This post is the first one in a series of many that will follow.

Who is **Tortoise** and who is **Hare**? 

Well, In many books about apache spark that I was reading, I didn't found a clear idea of the performance of dataframes compared to the datasets. In this blog post, we will debunk that mystery and show some concrete results and insights regarding this matter.



# Two brothers from the same mother

To prepare ourselves for deep dive into this matter we need to get familiar with the basic concepts of this two API's.

## Dataframe API

Dataframe API is nothing more than data organized in named columns. One good example is the table in a relational database. It's untyped version of Datasets i.e `Dataset[Row]`. We could say that Dataframe is one step ahead of RDD (resilient distributed dataset) because it provides memory management and an optimized execution plan. This is a famous project called [Tungsten](https://jaceklaskowski.gitbooks.io/mastering-spark-sql/content/spark-sql-tungsten.html) which introduced **off-heap memory** which bypassed the overhead of the JVM object model and garbage collection. The off-heap memory is the memory that doesn't belong to JVM, the memory that you can allocate and deallocate at your own will. 

## Dataset API

While the dataframe is untyped the Dataset API is providing a type-safe object-oriented programming interface. Dataset doesn't use. They offer compile-time type safety, which means that you can avoid some runtime errors in comparison to Dataframe API. Their memory is bound to the JVM. This API also enjoys the benefits of Spark SQL’s optimized execution engine.


These two APIs are fundamentally the same except in the fact of how they manage their memory. In the sections below, we will expose some useful insights which one is more performant and when to use one or another API. 



# Who runs at a speed of light?

I tested everything locally with a local spark cluster in standalone mode. I took this [dataset](https://www.kaggle.com/datasets/yuanyuwendymu/airline-delay-and-cancellation-data-2009-2018?fbclid=IwAR1RTFYmc5MqzTaGJO9tOyKJ177_xbZpQbsYBRnOPuI4-4zx4PuZ9eCC7_c) from Kaggle for benchmarking. 2018 year from this dataset. I made chunks from it from `500K` records to `2M` in incremental chunks of 250K.

In the figure bellow, you can see the results of the test.
![results](/assets/img/dataframe_vs_dataset.png)

On this [link](https://github.com/vesko-vujovic/SparkExamples/blob/master/src/main/scala/com/examples/spark/DatasetVsDataFrames.scala) you can find the code for this example.


In this example, we can see that on `2M records` we have a `20%` improvement over the dataset. Please keep in mind that this test is ultra simple we just read the CSV file and then we group and sum, I'm pretty much convinced that improvement in time would be even more than **20%** in `execution time` on large scale applications.

Dataframe API will be the most performant solution because it avoids garbage collection and works directly on binary data which means that we also don't have `serialization/deserialization` of objects which is a really heavy operation when using datasets, a lot of time is spent on this operation. 

Another big problem with Java objects is that they have a big memory overhead. For example, to store a simple `AAAA` string that would take 4 bytes to store in UTF-8 encoding, Java will pack it in `~48 bytes` in total. The characters themselves each will use 2 bytes with a UTF-16 encoding, 12 bytes would be spent on the header and the other 8 bytes are used for hash code. To facilitate a common workloads we will make a trade-off for a larger memory overhead. If you use [Java Object Layout tool](https://openjdk.org/projects/code-tools/jol/) you will get the following output:

```
java.lang.String object internals:
OFFSET  SIZE   TYPE DESCRIPTION                    VALUE
     0     4        (object header)                ...
     4     4        (object header)                ...
     8     4        (object header)                ...
    12     4 char[] String.value                   []
    16     4    int String.hash                    0
    20     4    int String.hash32                  0
Instance size: 24 bytes (reported by Instrumentation API)

```
GC will work well when he's able to predict the lifecycle of transient objects but it will fail short if some transient objects spill into the old generation. since this approach is based on heuristics and estimation squeeizng out the performance from JVM would take dark arts and dozens of parameters to teach JVM about the life cycle of objects. 

To fuel up the performance of the dataframe Project Tungsten uses `sun.misc.Unsafe` the functionality provided by JVM that exposes C-style memory access with explicit allocation, deallocation, and pointer arithmetics.



# Final thougts

If you are ok with the fact that you don't have compile-time safety and sometimes column name change can break your code then `Dataframe API` is the most performant solution. If you want compile-time safety and you are willing to pay the trade-off in terms of memory overhead and bigger execution time then `Dataset API` is the way to go. Choose wisely :) Until the next blog post...
