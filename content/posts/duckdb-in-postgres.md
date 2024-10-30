---
title: "DuckDB Inside Postgres: The Unlikely Duo Supercharging Analytics"
date: 2024-10-30T15:06:41+02:00
draft: true
tags:
  - postgresql
  - data-engineering
  - big-data
  - data-processing
  - duckdb
  - database
cover:
  image: "/posts/duckdb-postgres/duckdb-postgres-cover.png"
  alt: "duckdb"
  caption: "duckdb-postgresql"
---

![duckdb-postgresql](/posts/duckdb-postgres/duckdb-postgres-cover.png)


# DuckDB Inside Postgres: The Unlikely Duo Supercharging Analytics 

If you work in data engineering, you know that the field moves at a dizzying pace. New tools and technologies seem to pop up daily, each promising to revolutionize how we store, process and analyze data. Amidst this constant change, two names have remained stalwarts: Postgres, the tried-and-true relational database, and DuckDB, the talented new kid on the block for analytics workloads.

But what if I told you that these two data powerhouses could work together - not just side by side, but DuckDB *inside* Postgres? Sounds wild, right? Let's explore what this dynamic duo can do.

## Why DuckDB and Postgres Are a Match Made in Data Heaven

Postgres has long been a go-to for data storage. It's reliable, feature-rich, and ubiquitous across data platforms. But when it comes to analytics on large datasets, especially those requiring complex aggregations, **Postgres can start to strain under the load**. 

Enter DuckDB. **This embedded analytical database engine shines where Postgres struggles**. DuckDB was built from the ground up to excel at OLAP (online analytical processing) workloads. It's lightning-fast, thanks to its columnar storage and vectorized query execution.

Marrying Postgres' robust data management with DuckDB's analytics prowess is a stroke of genius. It's like having a dependable workhorse that can suddenly sprout wings and soar when it needs to crunch some serious numbers.

## Putting DuckDB Inside Postgres to the Test

Of course, the proof is in the pudding (or the query execution time, as it were). Using a handy Docker image provided by the DuckDB team, I spun up a Postgres instance with the DuckDB extension installed. I then generated a **hefty dataset - 100 million records** - and loaded it into a Postgres table. Complete guide on how to do it you can find below.


## Testing DuckDB Inside Postgres: A Step-by-Step Guide

Curious to see DuckDB inside Postgres in action? Here's how you can run your own tests:

1. **Spin up a Postgres-DuckDB Docker container:**
   
   ```bash
   docker run -d -e POSTGRES_PASSWORD=duckdb pgduckdb/pgduckdb:16-main
   ```

2. **Generate a test dataset.** I used a custom Rust tool that I made to generate a **100 million record CSV 16GB in size**, but you can use your preferred method. Ensure your CSV matches this schema:

   ```json
      {
      "columns": [
        {
          "name": "id",
          "type": "integer",
          "description": "The unique identifier for the transaction."
        },
        {
          "name": "user_id",
          "type": "integer",
          "description": "The identifier of the user associated with the transaction."
        },
        {
          "name": "provider_id",
          "type": "integer",
          "description": "The identifier of the provider associated with the transaction."
        },
        {
          "name": "amount",
          "type": "float",
          "description": "The  amount of the transaction."
        },
        {
          "name": "timestamp",
          "type": "string",
          "description": "The timestamp indicating when the transaction occurred."
        }
      ]
     }
   ```

3. **Load the data into Postgres:**
   
   - Copy the CSV into the Docker container:
     ```bash
     docker cp your_data.csv yourDockerId:/tmp/

     // Successfully copied 16.2GB to 4222c43adc85:/tmp/
     ```
   - Connect to Postgres:
     ```bash
     docker exec -it 4222c43adc85 psql
     ```
   - Create a table matching your schema:
     ```sql
     CREATE TABLE transactions (
          id BIGINT PRIMARY KEY,
          user_id BIGINT NOT NULL,
          provider_id BIGINT NOT NULL,
          amount DOUBLE PRECISION NOT NULL,
          timestamp VARCHAR(255) NOT NULL
          );

     // To check if table is created run \dt and you should see something like this

     postgres=# \dt


      List of relations

       Schema |     Name     | Type  |  Owner
      --------+--------------+-------+----------
       public | transactions | table | postgres
        (1 row)


     ```
   - Import the CSV data:
     ```sql
     COPY data FROM '/tmp/your_data.csv' DELIMITER ',' CSV HEADER;
     ```

4. **Run a test query using plain Postgres:**
   
   ```sql
   EXPLAIN ANALYZE 
   SELECT
     age,
     is_active,
     COUNT(*) AS user_count
   FROM data
   GROUP BY age, is_active;
   ```
   Note the execution time.

5. **Run the same query using DuckDB inside Postgres:**
   
   ```sql
   SET duckdb.force_execution = true;
   
   EXPLAIN ANALYZE
   SELECT
     age, 
     is_active,
     COUNT(*) AS user_count  
   FROM data
   GROUP BY age, is_active;
   ```
   Compare DuckDB's execution time to Postgres'.

That's it! You're now equipped to pit Postgres against DuckDB and see how they perform on your own data and queries.

## The Future of DuckDB and Postgres

[...remaining content stays the same...]


## The Verdict on DuckDB Inside Postgres

So, is the DuckDB-Postgres tag team worth the hype? It's a solid "maybe" for now. While DuckDB didn't outpace Postgres in this test, I suspect there's more to the story. The performance boost likely depends heavily on the specifics of your data and queries.

Don't write off this dynamic duo just yet. As the DuckDB team continues to refine their Postgres integration, we may see some truly impressive speed-ups for certain analytics workloads.

In the meantime, it's exciting to see two data stalwarts joining forces in new and unexpected ways. Who knows what other surprising pairings the future of data engineering holds?