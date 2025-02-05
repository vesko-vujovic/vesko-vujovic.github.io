---
title: "The Hidden Pitfall That Sabotages SQL Performance: Functions on Indexed Columns"
draft: true
tags:
  - database
  - sql
  - data-engineering
cover:
  image: /posts/sql-optimizers/sql-optimizers-cover.png
  alt: sql-optimizers
  caption: sql-optimizers
---



As data engineers and analysts, we rely heavily on SQL databases to store and query our data efficiently. To speed up our queries, we often create indexes on frequently filtered columns. However, there's a common gotcha that can cause our queries to run slower than expected, even with appropriate indexes in place. In this post, we'll explore how applying functions to indexed columns in the WHERE clause can hinder SQL optimizers from utilizing those indexes effectively.