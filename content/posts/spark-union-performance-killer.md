---
title: "Speed Up Your Spark Jobs: The Hidden Trap in Union Operations"
date: 2024-11-29T15:06:41+02:00
draft: false
tags:
  - big-data
  - data-engineering
  - apache-spark
  - data-processing
cover:
  image: /posts/tortoise_smaller.jpeg
  alt: tortoise
  caption: tortoise
description:
---

![optimized-union-cover](/posts/spark-union-performance/cover-union-performance.png)

# The Problem: Union function isn't as Simple as it Seems

Picture this: You have a large dataset that you need to process in different ways, so you:

- Split it into smaller pieces
- Transform each piece differently
- Put them back together using union

Sounds straightforward, right? Well, there's a catch that most developers don't know about.

# The Hidden Performance Killer 🐌

Here's what's actually happening behind the scenes when you use `union`:

```scala

    // Create two dataframes from 1 to 1000000
    val df1 = spark.range(10000000).toDF("value")
    val df2 = spark.range(10000000).toDF("value")

    // Perform inner join
    val df = df1.join(df2, Seq("value"), "inner")

    // Split into odd and even numbers
    val dfOdd = df.filter(col("value") % 2 === 1)
    val dfEven = df.filter(col("value") % 2 === 0)

    val dfOddAdded = dfOdd.withColumn("add_value", col("value") + 1)
    val dfEvenDivided = dfEven.withColumn("divide_value", col("value") / 2)

    // Union and count
    val result = dfOddAdded.union(dfEvenDivided).count()

    // Print the result
    println(s"Total count: $result")

    // Optional: Show the execution plan
    dfOddWithMagic.union(dfEvenDivided).explain("formatted")

```

When Spark sees this code, it doesn't realize it can reuse data. Instead, it goes back to the beginning and processes your entire pipeline again for each part of the union!

Think of it like running an entire production line twice to make identical toys, just to paint half of them red and half blue. Instead, you could run the production line once and split the toys for different paint jobs at the end!

Look at this execution plan, we see **twice the same part of the plan** i.e. **union operation will trigger the recomputation**:

![recompute-image](/posts/spark-union-performance/union-recompute.png)

# The Simple Fix: Cache is Your Friend!⚡

Here's how to make your unions lightning fast:

```scala 

    val df = df1.join(df2, Seq("value"), "inner")
    df.cache()

```


By adding `cache()`, you're telling Spark: "Hey, keep this data in memory - we'll need it again soon!"

### Now imagine these scenarios:
- You have **billions** of records to process 😧
- Your job is running on AWS Glue where you pay per DPU hour 💰
- Without caching, you're essentially:
  - Processing those billions of records twice
  - Doubling your computation time
  - Doubling your AWS costs 🔥

The cost impact is real:
- 2x processing time = 2x DPU hours
- 2x DPU hours = 2x your bill

This is why understanding caching isn't just about performance—it's about your bottom line! 💡

### Let's see the optimzied execution plan:

![optimized-union-image](/posts/spark-union-performance/optimized-union.png)

Let me explain how we can see caching in action in this execution plan:

1. The key indicator is `InMemoryRelation` at the top. This shows that Spark is using a cached version of the data with these specifications:
   - `StorageLevel(disk, memory, deserialized, 1 replicas)`: Data is stored both in memory and disk
   - `CachedRDDBuilder`: Shows it's using the cached RDD version

2. The execution plan shows:
   - Instead of recomputing the Range and Project operations multiple times
   - It's reusing the cached data for subsequent operations
   - `BroadcastHashJoin` is working with the cached data, not recomputing from scratch

3. Without caching, you would see:
   - Duplicate Range operations (0 to 1000000)
   - Multiple Project operations
   - More complex execution plan with repeated computations


### Caveat

Here's a good caveat about caching and disk spillage:

When using `cache()`, be careful! If you don't have enough memory, Spark will "spill" the cached data to disk, which could actually make your job **slower** than not caching at all. Why? Because now you're adding:
- Disk I/O operations (writing to and reading from disk)
- Data serialization/deserialization overhead
- Network traffic if using distributed storage
- Additional storage space management

It's like ordering takeout but your fridge is full, so you have to store the food in the basement. Now every time you want to eat (access the data), you have to walk down to the basement, bring the food up, heat it up (deserialize), and then do this all over again. Sometimes it's faster to just order fresh food (recompute the data) than deal with all that overhead! 🏃‍♂️🔄

The key is to be strategic about what you cache and always monitor your memory usage. Remember: not all data needs to be cached! 💡


Here's a compelling conclusion for your blog post about Spark caching and performance:

# Final Thoughts

Understanding Spark's union operation and caching mechanism isn't just about technical optimization—it's about being a thoughtful data engineer. Let's recap the key takeaways:

1. **Know Your Data Pipeline**: 
   - Before adding `cache()`, understand if and where your data is being reused
   - Monitor your execution plans to identify redundant computations
   - Be strategic about what you cache—not everything needs to be cached

2. **Consider the Trade-offs**:
   - Caching can significantly improve performance when used correctly
   - But remember: insufficient memory leads to disk spillage, which can make things worse

Remember, the goal isn't to cache everything—it's to cache smartly. Whether you're processing millions of records or working with complex transformations, understanding these concepts will help you build more efficient and cost-effective data pipelines.


If you missed my previous deep dive into Spark's union operation, you can catch up [here](https://blog.veskovujovic.me/posts/spark-union-function/). It complements this caching discussion nicely!

Happy Spark-ing! 🚀

---
*Found this helpful? Follow me for more data engineering tips and tricks*




