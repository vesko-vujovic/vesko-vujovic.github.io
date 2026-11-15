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

In this post we build a CDC pipeline fully on **AWS with Terraform: RDS for PostgreSQL, Debezium running on MSK Connect, Kafka on Amazon MSK, and ClickHouse** at the end. 

You will learn:
 
- What Debezium and MSK Connect do, in plain words
- How replication slots work, and how a stopped connector can fill up your production database's disk
- What to change to make the pipeline highly available
- Why so many companies put ClickHouse at the end of this kind of pipeline

## The Pipeline in One Picture

![CDC pipeline on AWS: RDS PostgreSQL streams changes through Debezium on MSK Connect into Kafka on Amazon MSK, and ClickHouse on EC2 consumes them into a ReplacingMergeTree table](/posts/cdc-postgres-to-clickhouse/aws_cdc_pipeline.png)


- **RDS for PostgreSQL** holds a `transactions` table with dummy card payments. The only setting it needs for CDC is `rds.logical_replication = 1`.
- **Debezium** reads the WAL through a replication slot and turns every row change into a JSON event.
- **Amazon MSK** stores those events in the topic `cdc.public.transactions`.
- **ClickHouse** reads the topic and keeps the latest version of every row, ready for analytical queries.

## Debezium and MSK Connect in Two Minutes

**Debezium** is an open source CDC tool. For PostgreSQL it uses logical decoding with the built-in `pgoutput` plugin, so you install nothing on the database. On the first start it takes a snapshot of the table. After that it streams every change. Each event says what happened (`c` for create, `u` for update, `d` for delete, `r` for snapshot read) and what the row looks like.

Debezium runs inside **Kafka Connect**, the framework for moving data in and out of Kafka. Running Kafka Connect yourself means managing workers, upgrades and scaling. 

**MSK Connect** is the managed version on AWS. You give it a plugin and a config, and AWS runs it.

Three things to know before you start:
 
1. **The plugin must be a ZIP or a JAR.** Debezium ships as a `.tar.gz`, so you download it and zip it again. The demo repo has a script for this.
2. **Pick Kafka Connect 3.7.x.** It runs on Java 17, and Debezium 3.x needs Java 17. On the older 2.7.1 option, a new Debezium connector simply isn't found.
3. **The connector always has one task.** One replication slot has one reader. You scale MSK Connect up (more capacity per worker), not out.



The important part of the connector config:
 
```properties
connector.class=io.debezium.connector.postgresql.PostgresConnector
plugin.name=pgoutput
slot.name=debezium_slot
publication.name=dbz_publication
table.include.list=public.transactions
decimal.handling.mode=string
heartbeat.interval.ms=10000
heartbeat.action.query=UPDATE public.debezium_heartbeat SET ts = now() WHERE id = 1
transforms=unwrap
transforms.unwrap.type=io.debezium.transforms.ExtractNewRecordState
transforms.unwrap.delete.tombstone.handling.mode=rewrite
transforms.unwrap.add.fields=op,lsn,source.ts_ms
```
 
`ExtractNewRecordState` flattens Debezium's before/after envelope into a plain row. A delete becomes a row with `__deleted = true`, which makes life easy for ClickHouse.


You can find all the code for setup here: https://github.com/vesko-vujovic/cdc-postgres-msk-clickhouse