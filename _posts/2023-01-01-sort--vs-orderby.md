---
layout: post
title:  "Apache Spark: sort() vs orderBy() - Are they the same?"
tags: [apache spark, data engineering]
---

![spark](/assets/img/sortvsorderby.jpeg)


# Introduction

In this blog post, we will discuss the two most commonly used functions to sort the data on one or more columns. Those functions are `sort()` and `orderBy()`. In many blog posts and spark documentation about sort() and orderby() I have found that they state that `sort()` function will sort data on the level of the partition. In this blog post, we will deep dive into this topic and debunk the myth.


Before we dive into the topic let's first create the test data needed for this example. I'm using local standalone cluster with two worker nodes with 2G of ram memory. Version of spark is __3.3.1__


```scala

val employees =
  Seq(
    ("Mark", "Sales", "AL", 80000, 24, 90000),
    ("John", "Sales", "AL", 76000, 46, 10000),
    ("Alex", "Sales", "CA", 71000, 20, 13000),
    ("Chris", "Finance", "CT", 80000, 42, 13000),
    ("Oliver", "Finance", "DE", 89000, 30, 14000),
    ("Emma", "Finance", "DL", 73000, 46, 29000),
    ("Olivia", "Finance", "IL", 88000, 63, 25000),
    ("Mia", "Marketing", "IN", 90000, 35, 28000),
    ("Henry", "Marketing", "RI", 82000, 40, 11000),
    ("Sam", "Marketing", "RI", 81000, 40, 12000),
    ("Bob", "Finance", "IL", 120000, 40, 91000),
    ("Peter", "Finance", "IL", 120001, 51, 95000),
  ).toDF("employee_name", "department", "state", "salary", "age", "bonus")

employees.show(truncate = false)


+-------------+----------+-----+------+---+-----+
|employee_name|department|state|salary|age|bonus|
+-------------+----------+-----+------+---+-----+
|Mark         |Sales     |AL   |80000 |24 |90000|
|John         |Sales     |AL   |76000 |46 |10000|
|Alex         |Sales     |CA   |71000 |20 |13000|
|Chris        |Finance   |CT   |80000 |42 |13000|
|Oliver       |Finance   |DE   |89000 |30 |14000|
|Emma         |Finance   |DL   |73000 |46 |29000|
|Olivia       |Finance   |IL   |88000 |63 |25000|
|Mia          |Marketing |IN   |90000 |35 |28000|
|Henry        |Marketing |RI   |82000 |40 |11000|
|Sam          |Marketing |RI   |81000 |40 |12000|
|Bob          |Finance   |IL   |120000|40 |91000|
|Peter        |Finance   |IL   |120001|51 |95000|
+-------------+----------+-----+------+---+-----+
```


# sort() function

In some blog posts on Stackoverflow and spark documentation, I have found that they state that the `sort()` function will sort the data on the partition level. I cannot confirm the same thing. Let's unroll our mystery and make a conclusion based on examples. See this [excerpt](https://spark.apache.org/docs/3.3.1/sql-ref-syntax-qry-select-sortby.html#content) from the spark documentation:

> The SORT BY clause is used to return the result rows sorted within each partition in the user specified order. When there is more than one partition SORT BY may return result that is partially ordered. This is different than ORDER BY clause which guarantees a total order of the output.


After we repartition and sort our data we cannot state that the data is sorted within __one partition__ .

```scala

    employees
      .repartition(4)
      .sort(col("salary"))
      .show(truncate = false)

+-------------+----------+-----+------+---+-----+
|employee_name|department|state|salary|age|bonus|
+-------------+----------+-----+------+---+-----+
|Alex         |Sales     |CA   |71000 |20 |13000|
|Emma         |Finance   |DL   |73000 |46 |29000|
|John         |Sales     |AL   |76000 |46 |10000|
|Chris        |Finance   |CT   |80000 |42 |13000|
|Mark         |Sales     |AL   |80000 |24 |90000|
|Sam          |Marketing |RI   |81000 |40 |12000|
|Henry        |Marketing |RI   |82000 |40 |11000|
|Olivia       |Finance   |IL   |88000 |63 |25000|
|Oliver       |Finance   |DE   |89000 |30 |14000|
|Mia          |Marketing |IN   |90000 |35 |28000|
|Bob          |Finance   |IL   |120000|40 |91000|
|Peter        |Finance   |IL   |120001|51 |95000|
+-------------+----------+-----+------+---+-----+

```

After a while, I started to doubt my conclusion and decided to dig even further and then I discovered a lot of interesting things. If we open the spark codebase and find the `sort()` function we find this: 

```scala

  @scala.annotation.varargs
  def sort(sortExprs: Column*): Dataset[T] = {
    sortInternal(global = true, sortExprs)
  }

```

Then I was intrigued what the `global` param means.:thinking: Then I followed the breadcrumbs along the road and found this.

``` scala
  private def sortInternal(global: Boolean, sortExprs: Seq[Column]): Dataset[T] = {
    val sortOrder: Seq[SortOrder] = sortExprs.map { col =>
      col.expr match {
        case expr: SortOrder =>
          expr
        case expr: Expression =>
          SortOrder(expr, Ascending)
      }
    }
    withTypedPlan {
      Sort(sortOrder, global = global, logicalPlan)
    }
  }



/**
 * @param order  The ordering expressions
 * @param global True means global sorting apply for entire data set,
 *               False means sorting only apply within the partition.
 * @param child  Child logical plan
 */
case class Sort(
    order: Seq[SortOrder],
    global: Boolean,
    child: LogicalPlan) extends UnaryNode {
  override def output: Seq[Attribute] = child.output
  override def maxRows: Option[Long] = child.maxRows
  override def outputOrdering: Seq[SortOrder] = order
  final override val nodePatterns: Seq[TreePattern] = Seq(SORT)
  override protected def withNewChildInternal(newChild: LogicalPlan): Sort = copy(child = newChild)
}

```
The sort function will every time call the `sortInternal` function with param `global = true` which implies that the whole dataset will be sorted across the cluster which means across all partitions.

### Spark sql

I was not lazy and tried to prove that I'm maybe wrong. I decided to create a temp view and query the data with spark.sql context. See the results below.


``` scala
spark
  .sql("SELECT /*+ REPARTITION(4) */ employee_name," +
    " department, state, salary, age, bonus FROM employees  SORT BY salary")
  .show(truncate = false)

+-------------+----------+-----+------+---+-----+
|employee_name|department|state|salary|age|bonus|
+-------------+----------+-----+------+---+-----+
|Mia          |Marketing |IN   |90000 |35 |28000|
|Peter        |Finance   |IL   |120001|51 |95000|------# end of 1st partition
|Alex         |Sales     |CA   |71000 |20 |13000|
|Chris        |Finance   |CT   |80000 |42 |13000|
|Henry        |Marketing |RI   |82000 |40 |11000|
|Bob          |Finance   |IL   |120000|40 |91000|------# end of 2nd partition
|John         |Sales     |AL   |76000 |46 |10000|
|Olivia       |Finance   |IL   |88000 |63 |25000|
|Oliver       |Finance   |DE   |89000 |30 |14000|------# end of 3rd partition
|Emma         |Finance   |DL   |73000 |46 |29000|
|Mark         |Sales     |AL   |80000 |24 |90000|
|Sam          |Marketing |RI   |81000 |40 |12000|------# end of 4th partition
+-------------+----------+-----+------+---+-----+

```
:bulb: :bulb: :bulb: Ha :exclamation: :exclamation: Wow, indeed the data is sorted within the partition and we can say that the statement from spark documentation is really true i.e `SORT BY may return a result that is partially ordered.`

Within this section, we can conclude that `SORT BY` from the spark documentation and the `sort()` function are not the same thing. `SORT BY` will not guarantee the total order of the data, while `sort()` function will return the total order of the data even if in some blogs and on stackoverflow they state that the `sort()` function is more efficient because it sorts the data on the level of partition which is not true.

Let's move on to our next section.


# orderby() function


This section will be short since there are no myths to debunk. The `ORDER BY` expression and `orderBy()` function will return the same result. And we can say that they both guarantee the total ordering of the data. 

We can :100: percent state that this excerpt taken form [spark](https://spark.apache.org/docs/3.3.1/sql-ref-syntax-qry-select-orderby.html) documentation is true: 

> The ORDER BY clause is used to return the result rows in a sorted manner in the user specified order. Unlike the SORT BY clause, this clause guarantees a total order in the output.

For this given peace of code we get the same result.sql

```scala

employees
  .repartition(4)
  .orderBy(col("salary"))
  .show(truncate = false)

spark
  .sql("SELECT /*+ REPARTITION(4) */ employee_name," +
    " department, state, salary, age, bonus FROM employees  ORDER BY salary")
  .show(truncate = false)

+-------------+----------+-----+------+---+-----+
|employee_name|department|state|salary|age|bonus|
+-------------+----------+-----+------+---+-----+
|Alex         |Sales     |CA   |71000 |20 |13000|
|Emma         |Finance   |DL   |73000 |46 |29000|
|John         |Sales     |AL   |76000 |46 |10000|
|Chris        |Finance   |CT   |80000 |42 |13000|
|Mark         |Sales     |AL   |80000 |24 |90000|
|Sam          |Marketing |RI   |81000 |40 |12000|
|Henry        |Marketing |RI   |82000 |40 |11000|
|Olivia       |Finance   |IL   |88000 |63 |25000|
|Oliver       |Finance   |DE   |89000 |30 |14000|
|Mia          |Marketing |IN   |90000 |35 |28000|
|Bob          |Finance   |IL   |120000|40 |91000|
|Peter        |Finance   |IL   |120001|51 |95000|
+-------------+----------+-----+------+---+-----+

```

If you are not again convinced then this line of code from spark codebase tells more than 1000 words :smiley:

```
  @scala.annotation.varargs
  def orderBy(sortExprs: Column*): Dataset[T] = sort(sortExprs : _*)

```

Internally it will call the `sort()` function which will guarantee the total ordering of data.


# Final thoughts

In this blog post, we have concluded that `SORT BY` expression and `sort()` function are not the same. If you need to sort data within a partition you can use the `sortWithinPartitions()` or `SORT BY` expression which will do the trick. On the other hand `ORDER BY` expression and the `orderBy()` function are the same, they both return the total ordering of the data. From what we saw from the spark codebase `orderBy()` function is just an alias for the `sort()` function. 

:bulb: :bulb: :bulb: *Global sorting is really spark job killer since all the data needs to be passed to one machine in order to check that all data is correctly sorted. If you have a lot of data then :boom:. You get it :). When you can choose `sortWithinPartitions()` or `SORT BY` would be more efficient indeed.*

All the code used in this post can be found [here](https://github.com/vesko-vujovic/SparkExamples/blob/master/src/main/scala/com/examples/spark/SortVsOrderBy.scala)



Until the next post... :wave:










