---
title: "📐 Semantic Layer Inside Databricks: A Look at Unity Catalog Metric Views (With an Example)"
date: 2026-07-26T15:06:41+02:00
draft: false
tags:
  - big-data
  - data-engineering
  - date-modeling
  - AI
  - databricks
  - Genie
cover:
  image: 
  alt: databricks-semantic-layer
  caption: databricks-semantic-layer
---



## 🤔 The enterprise-wide enigma

Ask three people on your data team what "total revenue" means and you'll get three queries. One filters out returns. One rounds before summing. One groups by order date, another by ship date. All three are technically correct, and none of them match.

This isn't a data quality problem. The data is fine. The problem is that every dashboard, notebook, and BI tool re-implements the aggregation logic from scratch, and nothing stops those implementations from drifting apart from each other.

Standard SQL views don't fix this. A view locks in its GROUP BY and output columns the moment you create it. If someone wants to slice revenue by region instead of by month, they can't just re-group the view. They write a new query against the underlying tables, and the drift starts again.

**Unity Catalog metric views take a different approach.** 

_**They separate what you're measuring from how you're grouping it.**_

Define "total revenue" once, and let people group by whatever field they need at query time. Same metric, same number, no matter who's asking.

## 🧱 What a metric view actually is

A metric view is a Unity Catalog object built on five pieces: **a source, optional joins, an optional filter, fields, and measures.**

Source is the base table, view, or SQL query the metric view reads from. Nothing special here, it's just where the data comes from.

**Joins** let you enrich that source with attributes from other tables, most commonly dimension tables. You declare the join condition and the cardinality, and the engine only pulls in the joined table when a query actually needs a field from it.

**Filter** applies to every query against the metric view, no exceptions. If a metric view should only ever show completed orders, you bake that into the filter once instead of hoping everyone remembers to add WHERE status = 'completed'.

**Fields** (also called dimensions) are the things you group and filter by: category, region, order month, status. A field can also be an unaggregated numeric column, like unit price, that gets aggregated later at query time.

**Measures** are the actual metrics: total revenue, order count, average order value. This is the part that trips people up coming from regular SQL, and it's worth being precise about. A measure isn't a **fact table**, and it isn't a precomputed number sitting in a column. The source you point a metric view at is your fact table (or something shaped like one). **A measure is simply the aggregate expressions you'd compute over the fact table's numeric columns**, things like SUM(revenue) or COUNT(order_id), given a name and made reusable instead of retyped slightly differently in every query. It has no fixed grouping level attached to it, and it only gets evaluated when you wrap it in `MEASURE()` in your query.

```sql
SELECT
  `Order Month`,
  MEASURE(`Total Revenue`)
FROM orders_metric_view
GROUP BY ALL
```

## 🏗️ Building a small star schema with dummy data

Databricks ships a samples.tpch dataset that most docs use for metric view examples. It's fine for a syntax reference, but it doesn't really let you see the **"define once, group any way"** payoff, because you don't control the shape of the data. Let's build our own fact table and dimensions instead, so every field and measure below maps directly to something you just created.

```sql
CREATE TABLE dim_product (
  product_id INT,
  product_name STRING,
  category STRING,
  subcategory STRING
);

INSERT INTO dim_product VALUES
  (1, 'Laptop Pro 14', 'Electronics', 'Laptops'),
  (2, 'Laptop Air 13', 'Electronics', 'Laptops'),
  (3, 'Phone X', 'Electronics', 'Phones'),
  (4, 'Phone SE', 'Electronics', 'Phones'),
  (5, 'Wireless Mouse', 'Electronics', 'Accessories'),
  (6, 'Office Chair', 'Home Goods', 'Furniture'),
  (7, 'Standing Desk', 'Home Goods', 'Furniture'),
  (8, 'Coffee Maker', 'Home Goods', 'Kitchen'),
  (9, 'Wool Sweater', 'Apparel', 'Outerwear'),
  (10, 'Running Shoes', 'Apparel', 'Footwear');

CREATE TABLE dim_region (
  store_id INT,
  store_name STRING,
  region STRING,
  country STRING
);

INSERT INTO dim_region VALUES
  (1, 'Store NYC', 'North America', 'USA'),
  (2, 'Store Toronto', 'North America', 'Canada'),
  (3, 'Store London', 'Europe', 'UK'),
  (4, 'Store Berlin', 'Europe', 'Germany'),
  (5, 'Store Tokyo', 'APAC', 'Japan'),
  (6, 'Store Sydney', 'APAC', 'Australia');
```

Ten products across three categories, six stores across three regions. Small enough to eyeball, varied enough that grouping by different fields actually produces different-looking results.

The fact table

This is the part worth being careful about. If every row has the same quantity and the same price, every aggregation you run will look suspiciously clean, and you won't actually be testing anything. So instead of hardcoding a handful of repeated rows, generate a couple thousand with real variance in date, product, store, quantity, and a discount that kicks in on some orders but not others:

```sql
CREATE OR REPLACE TABLE sales_fact AS
SELECT
  id AS order_id,
  date_add('2024-01-01', CAST(id % 365 AS INT)) AS order_date,
  CAST((id % 10) + 1 AS INT) AS product_id,
  CAST((id % 6) + 1 AS INT) AS store_id,
  CAST(((id * 37) % 12) + 1 AS INT) AS quantity,
  ROUND(
    (CAST(((id * 37) % 12) + 1 AS INT)) *
    CASE (id % 10) + 1
      WHEN 1 THEN 1800 WHEN 2 THEN 1200 WHEN 3 THEN 999 WHEN 4 THEN 549
      WHEN 5 THEN 39   WHEN 6 THEN 249  WHEN 7 THEN 459 WHEN 8 THEN 89
      WHEN 9 THEN 79   WHEN 10 THEN 129
    END *
    (1 - (CASE id % 5 WHEN 0 THEN 0.15 WHEN 1 THEN 0.05 WHEN 3 THEN 0.10 ELSE 0.0 END)),
    2
  ) AS revenue
FROM range(1, 2001);

```

That's 2,000 order lines spread across a full year, ten products at realistic price points, and a discount pattern that hits roughly 60% of orders at different rates. sales_fact is your fact table here, in the **Kimball sense: it's the thing metric view measures will aggregate over.**


```sql
CREATE OR REPLACE VIEW sales_metric_view WITH METRICS LANGUAGE YAML AS
$$
version: 1.1
comment: 'Sales KPIs across product and region dimensions'
source: sales_fact

joins:
  - name: product
    source: dim_product
    'on': source.product_id = product.product_id
    rely:
      at_most_one_match: true
  - name: store_region
    source: dim_region
    'on': source.store_id = store_region.store_id
    rely:
      at_most_one_match: true

fields:
  - name: Order Month
    expr: DATE_TRUNC('MONTH', source.order_date)
    comment: 'Month of the order'
  - name: Category
    expr: product.category
    comment: 'Product category'
  - name: Subcategory
    expr: product.subcategory
    comment: 'Product subcategory'
  - name: Region
    expr: store_region.region
    comment: 'Sales region'
  - name: Country
    expr: store_region.country
    comment: 'Country of sale'

measures:
  - name: Total Revenue
    expr: SUM(source.revenue)
    comment: 'Sum of all order revenue'
  - name: Units Sold
    expr: SUM(source.quantity)
    comment: 'Total quantity sold'
  - name: Order Count
    expr: COUNT(1)
    comment: 'Number of order line items'
  - name: Average Order Value
    expr: SUM(source.revenue) / COUNT(1)
    comment: 'Average revenue per order line'
$$;
```

## ⚡ Where the power shows up

Here's the part that actually justifies building a metric view instead of just writing four separate queries: every query below reuses the same Total Revenue, Units Sold, and Average Order Value measures. Nobody redefines the aggregation logic. Nobody risks the region cut drifting from the category cut.

```sql
SELECT
  Region,
  MEASURE(`Total Revenue`),
  MEASURE(`Units Sold`),
  MEASURE(`Order Count`)
FROM sales_metric_view
GROUP BY ALL
ORDER BY Region;
```

![metric-view](/posts/semantic-view-dbx/metric_view_v1.png)


Slice 2: by product category

```sql
SELECT
  Category,
  MEASURE(`Total Revenue`),
  MEASURE(`Average Order Value`)
FROM sales_metric_view
GROUP BY ALL
ORDER BY Category;
```

Slice 3: by month, trended over the year

```sql
SELECT
  `Order Month`,
  MEASURE(`Total Revenue`),
  MEASURE(`Order Count`)
FROM sales_metric_view
GROUP BY ALL
ORDER BY `Order Month`;
```

Slice 4: region and category together

```sql
SELECT
  Region,
  Category,
  MEASURE(`Total Revenue`),
  MEASURE(`Units Sold`)
FROM sales_metric_view
GROUP BY ALL
ORDER BY Region, Category;
```

Four completely different shapes of output, one region-only, one category-only, one time-trended, one crossed two ways, and not one line of aggregation logic was rewritten to get there. Total Revenue is defined exactly once, in the metric view. Every query just tells it how to group.

Run these against your own `sales_metric_view` and you'll see the actual numbers, they'll depend on the exact dummy data you generated. What matters isn't the specific figures, it's that switching from a region cut to a category cut took zero SQL changes to the underlying math. Try doing that with four copy-pasted `GROUP BY` queries against `sales_fact` directly, and you'll either end up hand-maintaining four slightly different `SUM(revenue)` expressions, or you'll write a fifth query to double check the first four agree with each other.

That's the whole pitch of a semantic layer in one paragraph: the grouping is disposable, the metric definition isn't.

Ready for Section 5 (the query mechanics, `MEASURE()`, and the CTE workaround for joining metric views to other tables) when you are.