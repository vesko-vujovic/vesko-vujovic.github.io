---
title: "Apache Spark: Beware of Column Ordering and Data Types When Using Apache Spark's Union Function"
date: 2024-10-06T08:06:41+02:00
draft: false
tags:
  - apache-spark
  - data-engineering
  - big-data
  - data-processing
cover:
  image: "/posts/spark-union-function/spark_cover.svg"
  alt: "spark-union-function"
  caption: "spark-union-function"
---

![spark-union-function](/posts/spark-union-function/spark_cover.svg)


# Introduction

In this blog post, we'll zoom into the details of how column ordering and data types can cause issues when using the union function in Apache Spark to combine two dataframes. We'll explore real-world examples that illustrate the problem and provide practical solutions to overcome these challenges. By the end of this post, you'll have a better understanding of how to use union effectively and avoid common pitfalls that can lead to job failures.

So, let's get this party started :tada: :tada: :tada:


# Let's get to know our beloved union function :heart:

The union function in Apache Spark is used to combine two or more DataFrames vertically. It creates a new DataFrame containing all the rows from the input DataFrames. The union function performs a **union based on the position of the columns, not their names.**

Here's a snippet from the Spark code that shows the implementation of the `union` function:

```scala

  /**
   * Returns a new Dataset containing union of rows in this Dataset and another Dataset.
   *
   * This is equivalent to `UNION ALL` in SQL. To do a SQL-style set union (that does
   * deduplication of elements), use this function followed by a [[distinct]].
   *
   * Also as standard in SQL, this function resolves columns by position (not by name):
   *
   * {{{
   *   val df1 = Seq((1, 2, 3)).toDF("col0", "col1", "col2")
   *   val df2 = Seq((4, 5, 6)).toDF("col1", "col2", "col0")
   *   df1.union(df2).show
   *
   *   // output:
   *   // +----+----+----+
   *   // |col0|col1|col2|
   *   // +----+----+----+
   *   // |   1|   2|   3|
   *   // |   4|   5|   6|
   *   // +----+----+----+
   * }}}
   *
   * Notice that the column positions in the schema aren't necessarily matched with the
   * fields in the strongly typed objects in a Dataset. This function resolves columns
   * by their positions in the schema, not the fields in the strongly typed objects. Use
   * [[unionByName]] to resolve columns by field name in the typed objects.
   *
   * @group typedrel
   * @since 2.0.0
   */
  def union(other: Dataset[T]): Dataset[T] = withSetOperator {
    combineUnions(Union(logicalPlan, other.logicalPlan))
  }


}
```

As you can see in the comments of Apache Spark  itself, the `union` function doesn't take care if names of the columns are the same. It just combines two `dataframes` by **order**. 

This part here are teaching us an important lesson. :mouse_trap: A trap that sometimes without thinking I have fallen into. :smile:

``` scala 

  /**
   *   val df1 = Seq((1, 2, 3)).toDF("col0", "col1", "col2")
   *   val df2 = Seq((4, 5, 6)).toDF("col1", "col2", "col0")
   *   df1.union(df2).show
   *
   *   // output:
   *   // +----+----+----+
   *   // |col0|col1|col2|
   *   // +----+----+----+
   *   // |   1|   2|   3|
   *   // |   4|   5|   6|
   * /  // +----+----+----+

```


# The problem :exclamation:

Apache Spark's union function is a powerful tool for combining multiple DataFrames or Datasets into a single DataFrame. It allows you to vertically concatenate data from different sources, making it easier to work with and analyze the combined data. 

However, when using union, it's crucial to be aware of potential **pitfalls that can break your Spark job**.

One common issue arises when the DataFrames being unioned have different column orderings or data types. If you attempt to union such DataFrames and then cast the resulting DataFrame to a Dataset, your job may fail with cryptic errors. This can be frustrating and time-consuming to debug, especially if you have a large dataset. The Feedback loop can be long.


## Example: 

Consider the following example where we have two DataFrames, `df1` and `df2`, with the same schema but different column ordering:

```scala
case class Person(name: String, surname: String, age: Int)

import spark.implicits._

val df1 = Seq(
  ("Alice", "Doe", 23),
).toDF("name", "surname", "age")

val df2 = Seq(
  ("John", 24, "Doe"),
).toDF("name", "age", "surname")
```

If we attempt to union these DataFrames using `df1.union(df2)`, Spark will not raise any errors. However, the resulting DataFrame will have a schema that matches `df1`, and the data from `df2` will be misaligned.

When we try to convert the unioned DataFrame to a Dataset of a case class `Person`, like this:

```scala
val newOne = df1.union(df2).as[Person]
```

Spark will throw :bomb: an error because the columns from `df2` are in the wrong order and have mismatched data types compared to the expected `Person` schema. 

It will output something like this: `[CANNOT_UP_CAST_DATATYPE] Cannot up cast age from "STRING" to "INT"`



## The Solution

To avoid this issue, you have two options:

1. Ensure that the DataFrames being unioned have the same column ordering and data types. You can achieve this by explicitly specifying the schema when creating the DataFrames or by using the `select` function to rearrange the columns before performing the union.




   
For example:

   ```scala
   val df2Reordered = df2.select("name", "surname", "age")
   val dfUnioned = df1.union(df2Reordered)
   val newOne = dfUnioned.as[Person]
   ```

   By reordering the columns in `df2` to match the schema of `df1`, we can safely union the DataFrames and convert the result to a Dataset of `Person` instances.

2. Use the `unionByName` function instead of `union`. The `unionByName` function performs a union based on the column names rather than their positions. It aligns the columns by name before combining the DataFrames. 

 Here's an example:

 ```scala
 val dfUnioned = df1.unionByName(df2)
 val newOne = dfUnioned.as[Person]
 ```

   By using `unionByName`, Spark will match the **columns by their names**, regardless of their positions in the DataFrames. This ensures that the resulting DataFrame has the correct schema and data alignment.


# Final thoughts

When using Apache Spark's `union` function, it's crucial to ensure that the DataFrames being combined **have the same column ordering and data types**. Failing to do so can lead to silent errors and unexpected behavior in your Spark jobs. 

To mitigate this issue, you can either reorder the columns before performing the union or use the `unionByName` function, which matches columns by their names. By understanding how the `union` function works and applying these solutions, you can ensure the reliability and correctness of your Spark jobs.

{{< chat apache-spark-union-function>}}
