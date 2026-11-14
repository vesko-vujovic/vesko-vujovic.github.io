---
title: "CDC on AWS with PostgreSQL, Debezium, Kafka and ClickHouse"
draft: true
description: "Build a CDC pipeline on AWS with Terraform: RDS PostgreSQL, Debezium on Kafka Connect, Kafka on Amazon MSK and ClickHouse. Plus how a replication slot work."
date: 2026-11-14T22:14:14+01:00
tags: ["CDC", "PostgreSQL", "Debezium", "Kafka", "Amazon MSK", "MSK Connect", "ClickHouse", "AWS", "Terraform"]
cover:
  image: "/posts/cdc-postgres-to-clickhouse/cover.png"
  alt: cdc-postgres-to-clickhouse
  caption: cdc-postgres-to-clickhouse
---

## Intro

Your analytics team wants fresh numbers, and your PostgreSQL database is not built for them. You can run heavy queries on production and slow down the app. Or you can copy the data once a night and work with yesterday's numbers.

Or maybe you want to `squeeze` the data in real time (incrementaly) from production system and in parallel build new one. Let's say if client reached all the limits of `vertical scaling`.

Change Data Capture (CDC) solves both challenges. PostgreSQL already writes every insert, update and delete to its write-ahead log (WAL). 

CDC reads that log and streams each change in another database or service. 



