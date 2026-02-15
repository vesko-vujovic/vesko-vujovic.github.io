---
title: "🪤 The Fan Trap: Why Your SQL Joins Are Inflating Your Numbers"
date: 2026-02-15T10:30:00+01:00
draft: false
tags:
  - sql
  - data-modeling
  - data-engineering
  - sql-optimization
  - data-quality
cover:
  image: "/posts/fan-trap/fan-trap-cover.png"
  alt: "chasm-trap-sql-aggregation"
  caption: "fan-trap-sql"
---


You run a query to get total revenue per customer. Customer #1 should have $500 in orders. Your query says $1,500. The raw data checks out. So what is wrong here?

You just hit the fan trap — a sneaky SQL join issue that multiplies your numbers without any warning. Let me show you how it happens and how to fix it.

## 🤔 What Is the Fan Trap?

The **fan trap** happens when you join tables along a one-to-many relationship and then aggregate. **The "many" side fans out the rows from the "one" side, duplicating them before your SUM or COUNT ever runs.**

Think of it this way: you have one customer with 3 orders. You join customers to orders — fine, you get 3 rows. But now you join orders to order_items, and each order has 2 items. Suddenly that one customer appears in 6 rows. If you try to SUM the order totals across those 6 rows, you're counting each order multiple times.

The "fan" is the shape of the data — one record fans out into many, and aggregation on the wrong level gives you inflated results.

## 💥 The Problem in Action

Let's say you have these tables:

**CUSTOMERS:**
```
customer_id | name
------------|------
1           | Alice
```

**ORDERS:**
```
order_id | customer_id | order_total
---------|-------------|------------
101      | 1           | 200
102      | 1           | 300
```

**ORDER_ITEMS:**
```
item_id | order_id | product
--------|----------|--------
1       | 101      | Widget A
2       | 101      | Widget B
3       | 102      | Widget C
```

You want Alice's total revenue. Expected: $500 (200 + 300).

Here's the query most people write:

```sql
SELECT 
    c.customer_id,
    c.name,
    SUM(o.order_total) as total_revenue
FROM CUSTOMERS c
JOIN ORDERS o ON c.customer_id = o.customer_id
JOIN ORDER_ITEMS oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.name;
```

**What you get:**
```
customer_id | name  | total_revenue
------------|-------|---------------
1           | Alice | 700
```

Why $700 instead of $500?

Look at the intermediate result after the joins:

```
customer_id | order_id | order_total | item_id
------------|----------|-------------|--------
1           | 101      | 200         | 1
1           | 101      | 200         | 2       ← order 101 duplicated
1           | 102      | 300         | 3
```

Order 101 has 2 items, so it appears twice. When you SUM: 200 + 200 + 300 = 700. The order_items table fanned out the orders, inflating your total.

## ✅ How to Fix It

### Solution 1: Aggregate Before Joining (Recommended)

```sql
WITH order_totals AS (
    SELECT customer_id, SUM(order_total) as total_revenue
    FROM ORDERS
    GROUP BY customer_id
)
SELECT 
    c.customer_id,
    c.name,
    ot.total_revenue
FROM CUSTOMERS c
JOIN order_totals ot ON c.customer_id = ot.customer_id;
```

You aggregate orders first, then join. No fan-out, no inflated numbers.

### Solution 2: Use a Subquery

```sql
SELECT 
    c.customer_id,
    c.name,
    (SELECT SUM(order_total) FROM ORDERS WHERE customer_id = c.customer_id) as total_revenue
FROM CUSTOMERS c;
```

The subquery calculates the total independently, so the order_items table never gets involved.

### What If You Need Item-Level Data Too?

If you need both order totals and item counts, aggregate each level separately:

```sql
WITH order_totals AS (
    SELECT customer_id, SUM(order_total) as total_revenue
    FROM ORDERS
    GROUP BY customer_id
),
item_counts AS (
    SELECT o.customer_id, COUNT(*) as total_items
    FROM ORDER_ITEMS oi
    JOIN ORDERS o ON oi.order_id = o.order_id
    GROUP BY o.customer_id
)
SELECT 
    c.customer_id,
    c.name,
    ot.total_revenue,
    ic.total_items
FROM CUSTOMERS c
JOIN order_totals ot ON c.customer_id = ot.customer_id
JOIN item_counts ic ON c.customer_id = ic.customer_id;
```

Same rule as the chasm trap: **aggregate first, join second**.

## 🎯 Fan Trap vs Chasm Trap

These two get confused a lot. Here's the difference:

**Fan Trap:** One-to-many join fans out rows. You join CUSTOMERS → ORDERS → ORDER_ITEMS, and the items duplicate the order rows. The chain of tables has increasing granularity, and aggregating at the wrong level inflates numbers.

**Chasm Trap:** Two independent tables join to the same shared table. You join SALES and PAYMENTS both through ORDERS, creating a many-to-many Cartesian product between them.

The quick way to tell them apart:

| | Fan Trap | Chasm Trap |
|---|---|---|
| **Cause** | One-to-many fan-out down a chain | Many-to-many Cartesian product |
| **Tables involved** | A → B → C (parent-child-grandchild) | B ← A → C (two tables linked through one shared table) |
| **What duplicates** | Parent rows get repeated by child rows | Both tables cross-join through the shared table |
| **Fix** | Aggregate at the right level before joining | Aggregate each table independently before joining |

The fix is the same idea in both cases: **don't aggregate after a join that multiplies your rows**. Aggregate first, then join the results.

## 🏁 Final Thoughts

The fan trap is one of those bugs that won't throw an error. Your query runs fine. The results look reasonable. But the numbers are wrong, and you might not notice until someone checks the math.

The rule is simple: **if your join increases the number of rows, don't aggregate across that join**. Aggregate first at the right level, then join.

Next time your SUM looks suspiciously high, check your joins. You might have a fan hiding in there.

---

Have you been bitten by the fan trap before? What other SQL join pitfalls have caught you off guard? Let me know in the comments.
