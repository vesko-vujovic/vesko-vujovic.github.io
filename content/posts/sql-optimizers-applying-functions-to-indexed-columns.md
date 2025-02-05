---
title: "🚨 The Hidden Pitfall That Sabotages SQL Performance: Functions on Indexed Columns 📉"
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

_In this post, we'll explore how applying functions to indexed columns in the WHERE clause can ditch SQL optimizers from utilizing those indexes effectively._

## 💪 The Power of Indexes
Let's start with a quick refresher on indexes. An index is a data structure that allows the database to quickly locate and retrieve specific rows based on the indexed column(s). When you create an index on a column, the database builds a separate, sorted data structure that maps the column values to their corresponding row locations. This enables the database to find matching rows much faster than scanning the entire table.
For example, consider a users table with an index on the email column:

``` sql
CREATE INDEX idx_users_email ON users (email);
```

Now, when you query the table with a filter on the email column, the optimizer can use the index to quickly find the matching rows:

```sql 
SELECT * FROM users WHERE email = 'john@example.com';
```


## 🕵️‍♀️ The Hidden Pitfall: Functions on Indexed Columns

However, things can go awry when you apply functions to indexed columns in the WHERE clause. Let's say you want to find all users whose email address is in a specific domain:

```sql
SELECT * FROM users WHERE SUBSTRING(email, -11) = '@example.com';
```


In this case, the SQL optimizer may not be able to use the index on the email column effectively. Why? Because the index is built on the raw email values, not the result of applying the SUBSTRING function to those values.

When you apply a function to an indexed column, the optimizer often has to scan the **entire table or index** to evaluate the function for each row. This can be much slower than directly looking up the raw column values in the index.


## 🌍 Real-World Examples
Let's look at a couple more examples where applying functions to indexed columns can ditch performance:

1. **🔍 Case-Insensitive Search**

Suppose you want to find all users with a specific first name, regardless of case:

```sql
SELECT * FROM users WHERE LOWER(first_name) = 'john';
```

Even if you have an index on the first_name column, the optimizer may not be able to use it efficiently because the index is built on the original case-sensitive values, not the lowercase versions.



2. **📅 Date Manipulation**

Consider a query that finds all orders placed on a specific day:
```sql
SELECT * FROM orders WHERE DATE_TRUNC('day', order_date) = '2023-06-01';
```

If you have an index on the order_date column, the optimizer may struggle to use it because the index is built on the raw timestamp values, not the truncated date values.



## Workarounds and Solutions


So, what can you do when you need to apply functions to indexed columns in your queries? Here are a few strategies:

**1. Create Function-Based Indexes**

Some databases, like **Oracle and PostgreSQL**, support function-based indexes. These indexes are built on the result of applying a function to a column, rather than the raw column values. For example:


```sql
CREATE INDEX idx_users_lower_first_name ON users (LOWER(first_name));

 ```


With this index, the optimizer can efficiently handle queries that use LOWER(first_name) in the WHERE clause.

_However, function-based indexes have limitations. They are not supported by all databases, and they can be less flexible than regular indexes because they are tailored to specific function use cases._

2. **Rewrite Queries**

In some cases, you can rewrite your queries to avoid applying functions to indexed columns. For example, instead of using **LOWER(first_name)**, you could store a separate lowercase version of the first_name column and create a regular index on it:

```sql
ALTER TABLE users ADD COLUMN first_name_lower VARCHAR(255);
UPDATE users SET first_name_lower = LOWER(first_name);
CREATE INDEX idx_users_first_name_lower ON users (first_name_lower);
```

Then, you can rewrite the query to use the first_name_lower column directly:

```sql
SELECT * FROM users WHERE first_name_lower = 'john';
```

_This approach requires additional storage space and maintenance, but it can enable the optimizer to use the index efficiently._


3. **Use Optimizer Hints**

Some databases provide optimizer hints that allow you to force the use of a specific index or join order. While hints should be used sparingly and with caution, they can be helpful in cases where the optimizer struggles to choose the best execution plan.


In **PostgreSQL**, you can use the `pg_hint_plan` extension to provide optimizer hints. First, you need to install the extension:

```sql
CREATE EXTENSION pg_hint_plan;
```

Then, you can use hints like `SeqScan`, `IndexScan`, or `IndexOnlyScan` to suggest the optimizer to use a specific scan method. For example:

```sql
SELECT /*+ INDEX(users idx_users_email) */ * 
FROM users 
WHERE SUBSTRING(email, -11) = '@example.com';
```
However, optimizer hints are not a silver bullet. They can make your queries less portable and more brittle to schema changes. It's generally better to let the optimizer make its own decisions based on statistics and cost estimates.


## 🐘  Demonstrating the Performance Impact with PostgreSQL and Docker

Let's see the performance impact in action using a PostgreSQL Docker container. Follow these steps:

1. 🐳  Pull the PostgreSQL Docker image and start a container:

```bash
docker pull postgres
docker run --name my-postgres -e POSTGRES_PASSWORD=password -d postgres
```

2. 🔌 Connect to the PostgreSQL container:

```bash
docker exec -it my-postgres psql -U postgres
```

3. 🔧 Execute the SQL statements to create a table, insert sample data, and create an index:

```sql

CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255)
);

INSERT INTO users (email)
SELECT 'user' || num || '@example.com'
FROM generate_series(1, 1000000) AS num;

CREATE INDEX idx_users_email ON users (email);

```

4. 🚀 Query using the indexed column directly:

```sql
EXPLAIN ANALYZE
SELECT * FROM users
WHERE email = 'user500000@example.com';
```

We get this:

```sql
 Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=26) (actual time=0.192..0.193 rows=1 loops=1)
   Index Cond: ((email)::text = 'user500000@example.com'::text)
 Planning Time: 1.118 ms
 Execution Time: 0.273 ms
(4 rows)

```
Let's conclude that the optimizer **efficiently uses the index** to find the matching row.


5. 🐌 Query using a function on the indexed column:

```sql
EXPLAIN ANALYZE
SELECT * FROM users
WHERE SUBSTRING(email, 5, 6) = '500000';
```

```sql
 Gather  (cost=1000.00..15102.00 rows=5000 width=26) (actual time=77.663..79.179 rows=1 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users  (cost=0.00..13602.00 rows=2083 width=26) (actual time=62.086..74.125 rows=0 loops=3)
         Filter: ("substring"((email)::text, 5, 6) = '500000'::text)
         Rows Removed by Filter: 333333
 Planning Time: 0.267 ms
 Execution Time: 79.274 ms
(8 rows)

```

From the query plan we can all see that last query is **290 times** slower. ❗❗❗

Notice that applying the SUBSTRING function prevents the optimizer from using the index effectively, resulting in a sequential scan of the entire table.

The performance difference becomes significant as the table size grows. Applying functions to indexed columns can lead to much slower query execution compared to leveraging indexes directly. ⏰

The query that uses `hints` will perform the same like the query from the forth step. 

## Conclusion
As data engineers and analysts, it's crucial to understand how SQL optimizers work and to be aware of potential performance gotchas. 
By keeping an eye out for function application on indexed columns and choosing appropriate strategies to address it, we can ensure that our queries run efficiently and our databases perform at their best. 💪🚀 



