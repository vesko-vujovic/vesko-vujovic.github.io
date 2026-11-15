---
title: "CDC on AWS with PostgreSQL, Debezium, Kafka and ClickHouse"
draft: false
description: "Build a CDC pipeline on AWS with Terraform: RDS PostgreSQL, Debezium on Kafka Connect, Kafka on Amazon MSK and ClickHouse. Plus how a replication slot work."
date: 2026-11-14T22:14:14+01:00
tags: ["CDC", "PostgreSQL", "Debezium", "Kafka", "Amazon MSK", "MSK Connect", "ClickHouse", "AWS", "Terraform"]
cover:
  image: "/posts/cdc-postgres-to-clickhouse/cover.png"
  alt: cdc-postgres-to-clickhouse
  caption: cdc-postgres-to-clickhouse
---

![cdc-postgres-to-clickhouse-cover](/posts/cdc-postgres-to-clickhouse/cover.png)

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


## Replication Slots: How CDC Can Take Down Production
 
### What a replication slot is
 
PostgreSQL writes every change to the **WAL (write ahead log)** before it touches the table. 

The WAL is a long list of 16 MB files, and every position in it has an address called an **LSN** (log sequence number). 

Once PostgreSQL has safely written changes to the tables, it recycles (deletes) the old WAL files to save disk.
 
A **replication slot** is an object inside the database that remembers how far one reader has read.

`The slot is a sticky note on a page that says "someone still needs everything after this page." The cleaner can't throw away anything after the note until the reader moves it forward.`
 
A slot tracks two positions:
 
- `confirmed_flush_lsn`: the last change the reader has confirmed it has stored safely.
- `restart_lsn`: the oldest WAL the database must keep, so the reader can continue from there after a restart.

A **logical** slot, the kind Debezium uses, does one more thing. It decodes the raw WAL into row changes (insert this row, update that one) through an output plugin, here `pgoutput`. The **publication** decides which tables go into that stream.
 
For our pipeline it works like this:
 
1. On its first start, Debezium creates `debezium_slot` and takes a snapshot of the table.
2. It reads changes from the slot and sends them to Kafka.
3. Each time Kafka Connect commits its offsets, Debezium tells PostgreSQL "I have everything up to this LSN." The slot moves forward, and PostgreSQL can recycle the older WAL.

Two rules matter for the rest of this section. A slot has **one reader** at a time. And it **lives in the database**, not in the connector, so it survives restarts, crashes and even deleting the connector. You can see every slot in `pg_replication_slots`.
 
### How it breaks
 
That is what makes CDC reliable. If Debezium restarts, it continues from the slot and loses nothing. It is also the danger:
 
- The connector stops, crashes, or someone deletes it.
- The slot stays. It is not attached to the connector, it lives in the database.
- **The application keeps writing. PostgreSQL keeps every WAL file the slot hasn't confirmed.**
- The disk fills up. On RDS, the instance goes into `storage-full` and stops accepting writes.

Nothing is wrong with the database or the data. A forgotten `replication_slot` is enough to stop production.
 
There is an RDS twist too. RDS writes to its own internal tables in the background. If your captured tables are quiet, Debezium gets no events and never confirms a newer position, so WAL piles up anyway. That is why the config has a **heartbeat**: every 10 seconds Debezium updates a small table, gets an event back and moves the slot forward.

### The initial snapshot can hurt too
 
The first load is the other risky moment. With `snapshot.mode=initial`, Debezium creates the slot first and then reads the whole table in one long transaction. On a table with hundreds of millions of rows, that can run for hours, and three things happen at once:
 
- **The slot holds WAL for the whole snapshot.** It was created before the snapshot started, and it can't move forward until the snapshot is done. A busy database keeps writing WAL the entire time.
- **The long transaction blocks vacuum.** PostgreSQL can't clean up dead rows that the open transaction might still see, so tables and indexes bloat and queries get slower.
- **The full table scan competes with the application** for CPU, disk I/O and memory, right on the production primary.

So a snapshot that takes too long fills the disk the same way a stopped connector does, and it also slows down everything else on the way.

What helps:
 
- **Use incremental snapshots for big tables.** Start with `snapshot.mode=no_data`, so Debezium only streams new changes, then trigger an incremental snapshot through a signal table. Debezium reads the table in small chunks between streaming events, with no long transaction, and the slot keeps moving.
- **Run the first load when traffic is low**, and watch slot lag and `FreeStorageSpace` while it runs.
- **Make sure the disk has room** for all the WAL the database writes during the snapshot window.
 
How to protect yourself:
 
- **Alarm on `OldestReplicationSlotLag`** and `FreeStorageSpace` in CloudWatch. The demo creates both.

- **Check slots from SQL** when something looks wrong:
```sql
SELECT slot_name, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```
 
- **Consider `max_slot_wal_keep_size`.** It caps how much WAL one slot can hold. The trade-off: when the cap is hit, PostgreSQL invalidates the slot and Debezium has to snapshot the whole table again. A full disk is worse than a re-snapshot, so for most teams the cap is worth it.
- **Don't count on storage autoscaling.** It hides the problem until the bill shows up.

## Run the Demo
 
The whole setup is in the repo: [github.com/vesko-vujovic/cdc-postgres-msk-clickhouse](https://github.com/vesko-vujovic/cdc-postgres-msk-clickhouse). You need Terraform, the AWS CLI and an AWS account.
 
**1. Build it**
 
```bash
make infra       # VPC, RDS, MSK, ClickHouse box (~30 min, MSK is the slow part)
make init        # table, debezium user, publication, ClickHouse tables, 10k rows
make connector   # start Debezium: snapshot first, then streaming
```
 
The connector starts in a separate step because the table, user and publication must exist before Debezium connects.

`The RDS instance`
![The cdc-demo RDS for PostgreSQL instance available in the AWS console after make infra: db.t4g.micro in eu-west-1a](/posts/cdc-postgres-to-clickhouse/rds.png)

`The MSK Kafka instance`
![The cdc-demo Amazon MSK cluster active in the AWS console after make infra: provisioned, Kafka 3.8.x, 2 brokers](/posts/cdc-postgres-to-clickhouse/kafka_msk.png)
 
**2. Watch changes flow**
 
```bash
make compare          # same row count and total in PostgreSQL and ClickHouse
make churn N=2000     # settle, refund and delete transactions
make compare          # matches again a few seconds later
make latency          # p50 / p99 lag from commit to ClickHouse
```
 
 
**3. Break it**
 
```bash
make stop-connector   # deletes the connector, the slot stays behind
make flood N=1000000  # 1M inserts plus an update of every row
make slot             # active = f, retained_wal keeps growing
```
 
**4. Fix it**
 
```bash
make start-connector  # resumes from the stored offset, no new snapshot
make slot             # retained_wal goes back down as Debezium catches up
```
 
The recovery works without a new snapshot because the worker config sets a fixed `offset.storage.topic`. By default, every new MSK Connect connector gets its own offset topic, so a recreated connector forgets where it was and starts over.

When you are done, run `make destroy`. **MSK, MSK Connect, RDS and the NAT gateway** all bill by the hour.

## Making It Highly Available
 
The demo runs on two AZs to keep costs down. For a real workload with a lot of data:
 
- **Kafka:** 3 brokers in 3 AZs, replication factor 3 and `min.insync.replicas = 2`. You can lose one AZ without losing data or stopping writes. In the demo this is one variable: `az_count = 3`.
- **MSK Connect:** you can't add tasks, so give the worker more capacity (MCUs) when throughput grows. If the worker dies, MSK Connect restarts it and Debezium continues from its offset. Keep the fixed offset topic so this also holds when you recreate the connector.
- **PostgreSQL:** the slot exists only on the primary. Without extra setup, a failover means a new slot and a new snapshot. PostgreSQL 17 on RDS can sync logical slots to a read replica, so the connector can continue after promotion. The slot has to be created with failover enabled, so check that your Debezium version supports it.
- **Many tables:** split them across a few connectors instead of one huge one. Each connector has its own slot, so a problem in one doesn't block the others. Every slot is one more thing to monitor, though.
- **ClickHouse:** use replicated tables on a cluster, or ClickHouse Cloud, instead of a single node.


## Why ClickHouse at the End
 
PostgreSQL stores data row by row, which is right for an app that reads and writes single orders. 

Analytics does the opposite. "Total refunds per merchant last month" reads two columns across millions of rows. 

`ClickHouse stores each column separately and compresses it, so that query reads a small part of the data and comes back in milliseconds to seconds.`
 
That is why companies use it as the destination for CDC data. It handles `real-time dashboards, user-facing analytics and reporting` without touching the production database. It's also open source, so you can run it yourself, as in this demo, or use the managed cloud version.
 
Two ClickHouse features make the CDC part simple:
 
1. **The Kafka table engine** reads the topic directly, and a materialized view inserts every batch into the real table. You don't need a second connector.
2. **`ReplacingMergeTree(_version, _is_deleted)`** keeps only the newest version of each row. The PostgreSQL log position (LSN) is the version, because it only goes up. A delete sets `_is_deleted = 1`, and the row disappears.
```sql
CREATE TABLE cdc.transactions
(
    id          UInt64,
    merchant    LowCardinality(String),
    amount      Decimal(12, 2),
    status      LowCardinality(String),
    _version    UInt64,
    _is_deleted UInt8
    -- other columns left out
)
ENGINE = ReplacingMergeTree(_version, _is_deleted)
ORDER BY id;
```

If you wonder what is `ReplacingMergeTree` it  is a specialized table engine in ClickHouse that extends the standard MergeTree engine to automatically remove duplicate rows and handle updates
 
One thing to remember: ClickHouse removes old versions in the background, not on insert. Add `FINAL` to queries that must see exactly one row per id:
 
```sql
SELECT merchant, sum(amount) AS total
FROM cdc.transactions FINAL
WHERE status = 'settled'
GROUP BY merchant
ORDER BY total DESC;
```


## Conclusion
 
- **CDC reads the database log**, so your application doesn't change and production gets no extra queries. Debezium on MSK Connect plus Kafka on Amazon MSK gives you that as a managed setup.
- **The replication slot is the part that can hurt you.** A stopped connector makes PostgreSQL keep WAL until the disk is full. Alarm on slot lag, use a heartbeat, and think about `max_slot_wal_keep_size`.
- **ClickHouse is a good place to end up.** The Kafka engine and `ReplacingMergeTree` turn a stream of changes into a table you can query, without another connector.
Clone the repo, run `make up`, then stop the connector and watch the slot grow. Seeing it happen in a demo is a lot cheaper than seeing it happen in production.