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