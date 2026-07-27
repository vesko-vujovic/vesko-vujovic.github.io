---
title: "Semantic Layer Inside Databricks: A Look at Unity Catalog Metric Views (With an Example)"
date: 206-08-27T15:06:41+02:00
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

# 🎯 Semantic Layer Inside Databricks: A Look at Unity Catalog Metric Views (With an Example)

🤔 The enterprise-wide enigma

Ask three people on your data team what "total revenue" means and you'll get three queries. One filters out returns. One rounds before summing. One groups by order date, another by ship date. All three are technically correct, and none of them match.

This isn't a data quality problem. The data is fine. The problem is that every dashboard, notebook, and BI tool re-implements the aggregation logic from scratch, and nothing stops those implementations from drifting apart from each other.

Standard SQL views don't fix this. A view locks in its GROUP BY and output columns the moment you create it. If someone wants to slice revenue by region instead of by month, they can't just re-group the view. They write a new query against the underlying tables, and the drift starts again.

Unity Catalog metric views take a different approach. They separate what you're measuring from how you're grouping it. Define "total revenue" once, and let people group by whatever field they need at query time. Same metric, same number, no matter who's asking.