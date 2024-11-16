---
title: "AWS Lambda Event Source Mapping: The Magic Behind Kafka Offset Management"
date: 2024-11-16T15:06:41+02:00
draft: true
tags:
  - apache-spark
  - data-engineering
  - big-data
  - kafka
  - AWS
cover:
  image: "/posts/esm-offset-management/esm-offset-management.png"
  alt: "kafka-offset-management"
  caption: "kafka-offset-management"
---

![kafka-offset-management](/posts/esm-offset-management/esm-offset-management.png)


## Introduction

When building event-driven architectures with AWS Lambda and Apache Kafka, one of the most critical yet often misunderstood components is offset management especially for event source mapping when you use lambda functions. 

Many developers wonder: **Do I need to manage Kafka offsets manually?** or **What happens when my consumer group's offsets expire?** 

In this blog post, we'll demystify how AWS Lambda's Event Source Mapping handles Kafka offsets automatically and what you actually need to know as a developer.

## Understanding the Architecture


Before diving into the details, let's visualize how Lambda Event Source Mapping works with Kafka:

```mermaid
flowchart TD
    A[Kafka Topic] -->|Poll Messages| B[Event Source Mapping]
    B -->|Batch Messages| C[Lambda Function]
    B -->|Store Checkpoint| D[(Internal Checkpoint Store)]
    C -->|Success| E{Commit Strategy}
    E -->|Batch Success| F[Commit Offset to Kafka]
    E -->|Partial Failure| G[Retry Failed Records]
    G -->|Max Retries Exceeded| H[Move to DLQ]
    D -->|Recovery| B
    F -->|Next Batch| B
```


## The Magic of Automatic Offset Management


### 1. Initial Setup & Configuration

Let's start with a basic Event Source Mapping configuration:

```yaml
Resources:
  KafkaEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      FunctionName: !Ref MyLambdaFunction
      StartingPosition: TRIM_HORIZON  # or LATEST
      BatchSize: 100
      MaximumBatchingWindowInSeconds: 30
      ParallelizationFactor: 1
      MaximumRecordAgeInSeconds: 604800  # 7 days
      MaximumRetryAttempts: 2
      DestinationConfig:
        OnFailure:
          Destination: !Ref DLQArn
      KafkaConfiguration:
        AutoOffsetReset: "earliest"
```
{{<linebreaks>}}


So let's refresh our memory how lambda is commiting offsets :brain: From my previous blog on [event-source-mapping](https://blog.veskovujovic.me/posts/event-source-mapping-and-lambda-scaling/) we said 🗣️:

> Whenever lambda finishes with status code 200, the offset will be committed automatically for the kafka topic.


After refreshing our memory we can create a summary of the things that ESM does for us. 