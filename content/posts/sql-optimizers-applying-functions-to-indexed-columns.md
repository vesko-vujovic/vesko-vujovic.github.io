---
title: "The Hidden Pitfall That Sabotages SQL Performance: Functions on Indexed Columns"
draft: true
date: 2025-02-05T15:06:41+02:00
tags:
  - database
  - sql
  - data-engineering
cover:
  image: "/posts/sql-optimizers/sql-optimizers-cover.png"
  alt: sql-optimizers
  caption: sql-optimizers
---

![sql-optimizers-cover](/posts/sql-optimizers/sql-optimizers-cover.png)

## Introduction

As data engineers and analysts, we rely heavily on SQL databases to store and query our data efficiently. To speed up our queries, we often create indexes on frequently filtered columns. However, there's a common gotcha that can cause our queries to run slower than expected, even with appropriate indexes in place. 

In this post, we'll explore how applying functions to indexed columns in the WHERE clause can ditch SQL optimizers from utilizing those indexes effectively.

## The Power of Indexes
Let's start with a quick refresher on indexes. An index is a data structure that allows the database to quickly locate and retrieve specific rows based on the indexed column(s). When you create an index on a column, the database builds a separate, sorted data structure that maps the column values to their corresponding row locations. This enables the database to find matching rows much faster than scanning the entire table.
For example, consider a users table with an index on the email column:

``` sql
CREATE INDEX idx_users_email ON users (email);
```

Now, when you query the table with a filter on the email column, the optimizer can use the index to quickly find the matching rows:

```sql 
SELECT * FROM users WHERE email = 'john@example.com';
```


## The Hidden Pitfall: Functions on Indexed Columns

However, things can go awry when you apply functions to indexed columns in the WHERE clause. Let's say you want to find all users whose email address is in a specific domain:

```sql
SELECT * FROM users WHERE SUBSTRING(email, -11) = '@example.com';
```


In this case, the SQL optimizer may not be able to use the index on the email column effectively. Why? Because the index is built on the raw email values, not the result of applying the SUBSTRING function to those values.

When you apply a function to an indexed column, the optimizer often has to scan the **entire table or index** to evaluate the function for each row. This can be much slower than directly looking up the raw column values in the index.


## Real-World Examples
Let's look at a couple more examples where applying functions to indexed columns can ditch performance:

1. Case-Insensitive Search

Suppose you want to find all users with a specific first name, regardless of case:

```sql
SELECT * FROM users WHERE LOWER(first_name) = 'john';
```

Even if you have an index on the first_name column, the optimizer may not be able to use it efficiently because the index is built on the original case-sensitive values, not the lowercase versions.



2. Date Manipulation

Consider a query that finds all orders placed on a specific day:
```sql
SELECT * FROM orders WHERE DATE_TRUNC('day', order_date) = '2023-06-01';
```

If you have an index on the order_date column, the optimizer may struggle to use it because the index is built on the raw timestamp values, not the truncated date values.



## Workarounds and Solutions

