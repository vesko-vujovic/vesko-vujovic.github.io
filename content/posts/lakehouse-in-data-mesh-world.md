---
title: "🏗️ Data Mesh Needed an Infrastructure. The Lakehouse Needed a Purpose. Was It Love?"
draft: false
date: 2026-03-22T08:06:41+02:00
tags:
  - Lakehouse
  - Data-Mesh
  - BigData
  - data-engineering
  - data-architecture
cover:
  image: "/posts/lakehouse-in-data-mesh-world/agent-core-cover.png"
  alt: "data-mesh-lakehouse"
  caption: "lakehouse in datamesh"
---

## Introduction

Data mesh has been generating a lot of noise. **Some teams swear by it, others think it's just a rebranding of problems they already had.** 

But underneath the debate, there's a real architectural question worth answering: if you're moving toward domain-owned data, what does your infrastructure actually look like?

The lakehouse keeps coming up as the answer. And for good reason — but it's not the full picture. You still need serving layers, you still have to justify the cost, and you need to understand why a traditional data warehouse doesn't quite fit the mesh model even if it also separates compute from storage these days.
That's what this post covers. No hype, just the architectural logic.

## 🕸️ What Is Data Mesh?
Data mesh is not a technology. You can't install it, you can't buy it from a vendor, and switching to Delta Lake doesn't mean you're doing data mesh. 

**It's an organizational and architectural philosophy, and that distinction matters before we go any further.**

_The core idea is simple: **data should be owned by the teams that produce it, not by a central data engineering team that acts as a middleman for the entire organization.** In a mesh model, the team running the payments service owns the payments data — they define its schema, ensure its quality, and expose it as a product for others to consume._

This sits on four principles introduced by **Zhamak Dehghani: domain ownership, ata as a product, self-serve infrastructure, and federated governance.** 

The first two are about accountability — who owns what and how they expose it. The last two are about making that ownership practical at scale without every domain reinventing the wheel or operating in complete isolation.

The reason this matters architecturally is that it fundamentally changes where decisions get made. In a centralized model, a single team controls the pipelines, the schemas, and the access. In a mesh, that control is distributed. Which means your infrastructure needs to support many independent teams working on the same underlying platform — without creating chaos.


## 😤 Why Organizations Turn to Data Mesh
It usually starts with a bottleneck.
The central data team is good. Maybe even great. But at some point, the business has grown enough that every analytics request, every new pipeline, every schema change has to go through the same small group of people. 

Product teams are waiting weeks for data they need to make decisions. The data team is permanently underwater, context-switching between the marketing team's dashboard and the finance team's reconciliation pipeline.

This is the centralized model breaking under its own weight. And it's not a talent problem — it's a structural one. **You've created a single team responsible for understanding every domain in the company well enough to build reliable data products for it. That doesn't scale.**

The other trigger is data quality ownership. In a centralized setup, **when a pipeline breaks or a metric looks wrong, accountability gets murky fast.** 

The upstream team says the data left their system correctly. The data team says they just moved it. Nobody owns the problem end to end.

**Data mesh forces that conversation by making domain teams responsible for the data they produce — including its quality and reliability.**

It's worth being honest though: data mesh introduces its own complexity. Distributed ownership means distributed governance, which requires **strong platform tooling and organizational discipline.** 

Teams that jump into mesh without that foundation often end up with more fragmentation, not less. The architecture only works if the organization is ready to treat data as a first-class engineering responsibility across every domain or so called **data as a product**.

## 🏠 Where the Lakehouse Fits
If data mesh is the organizational model, the lakehouse is the infrastructure layer that makes it practical.
Here's the core fit: a lakehouse stores data in open formats — Parquet files with Delta Lake, Apache Iceberg, or Hudi on top — sitting in object storage like S3 or GCS.

Any domain team can write their data product there using whatever compute engine they prefer.**Spark, Trino, DuckDB, Flink**— they all speak the same open table format. No central team needs to import your data, transform it into a proprietary format, or grant you access through a single query engine. The storage layer is shared infrastructure, but ownership and write access stays with the domain.

_This is where people raise a fair objection: modern data warehouses like Snowflake and BigQuery also separate compute from storage. So what's actually different?_

**The difference is who controls the storage layer and what format it lives in. In Snowflake, your data lives in Snowflake's internal storage, in Snowflake's format. You can scale compute independently, yes — but you're still going through Snowflake to read or write anything.**

 If another team wants to query your data with a different tool, they can't just point it at S3. They go through Snowflake, which means they're dependent on your Snowflake account, your access controls, and **Snowflake's pricing for every query they run.**

With a lakehouse on open formats, the storage is genuinely neutral. A domain team can write a Delta table to S3, register it in a catalog like **Unity Catalog or AWS Glue**, and any other team with the right permissions can query it with their engine of choice. 

**The data product is truly portable and independently accessible — which is exactly what data mesh requires.**

The lakehouse also fits the mesh model because it handles both large-scale batch processing and increasingly supports streaming and incremental updates through formats like Iceberg and Delta. Domains don't need separate infrastructure for different processing patterns. One storage layer, one format, multiple engines.

## 🍽️ Why You Still Need Serving Databases

The lakehouse handles storage and processing well. It does not handle every query pattern well, and confusing the two is where architectures get expensive and slow.

A lakehouse query engine — whether Spark, Trino, or Athena — is optimized for large analytical workloads. It scans files, applies predicates, aggregates across millions of rows. That's what it's built for. 

**What it's not built for is a dashboard that refreshes every 30 seconds, an API endpoint that needs to return a customer's order history in under 100 milliseconds, or an operational report that ten finance analysts are running simultaneously at 9am on Monday morning.**

These are fundamentally different access patterns. Low latency, high concurrency, predictable response times. For those workloads, you still need a serving layer — something like Postgres, Redshift, DynamoDB, or even a purpose-built OLAP store like ClickHouse or Apache Druid.

The typical pattern is straightforward: **the lakehouse is where data lives and gets processed, and the serving database is where curated, aggregated results land for consumption.**

Think of it as two different jobs. The lakehouse is your processing and storage backbone — it holds the full history, handles the heavy transformations, and is the source of truth across domains. 

The serving database is the last mile — it holds what consumers actually need to query fast and frequently, populated by pipelines running from the lakehouse.

This is not a failure of the lakehouse architecture. **It's just an honest acknowledgment that no single system optimizes for everything.**

Domain teams in a mesh still need to think about what their data product consumers actually need — and sometimes that means materializing results into a serving layer rather than handing someone a raw Iceberg table and wishing them luck.

## 🚫 Why Not Just Use a Traditional DWH?
The traditional data warehouse wasn't designed with distributed ownership in mind. It was designed for the opposite.

In a classic DWH setup, there's a central team that controls the schema, manages the ingestion pipelines, and defines how data from across the organization gets modeled and stored. 

That works well when you have a small number of well-understood data sources and a clear analytical mandate. It breaks down when you have dozens of product teams, hundreds of data sources, and domain knowledge scattered across the organization and unstructured data.

The structural problem is schema ownership. **In a traditional DWH, when the payments team wants to add a new field to their dataset, that change often has to go through a central team — because they own the ingestion pipeline, the transformation logic, and the warehouse schema.**

The payments team knows their data best, but they're blocked waiting for someone else to implement a change on their behalf. This is exactly the bottleneck data mesh is trying to eliminate, and a centralized DWH institutionalizes it.

Vendor lock-in is the other practical concern. Traditional warehouses store data in proprietary formats. Your data is inside Snowflake, or inside Redshift, and getting it out in bulk to use with another tool is slow, expensive, or both.

_For a mesh architecture where different domain teams might legitimately want to use different compute engines, that's a serious constraint. You end up paying for every cross-domain query through the same vendor, which gets costly fast at scale._

There's also a flexibility gap. Domain teams evolve their data products quickly. Schemas change, new sources get added, processing logic gets updated. Traditional DWH pipelines tend to be brittle under that kind of churn — tightly coupled ETL jobs that break when upstream schemas shift. 

Open table formats handle schema evolution more gracefully, which matters a lot when ownership is distributed and you can't coordinate every change through a central team.

None of this means traditional DWHs are obsolete. For smaller organizations with centralized analytical needs and a clear governance model, they're often still the right call. But for organizations scaling toward mesh, the centralized DWH becomes an architectural contradiction — the infrastructure enforces the old model even as the organization tries to move past it.