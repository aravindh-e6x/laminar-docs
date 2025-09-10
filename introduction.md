---
sidebar_position: 1
title: Introduction
---

Laminar is a high-performance, distributed stream ingestion/processing platform that combines the power of SQL with the
flexibility of modern stream processing. Laminar enables you to build, deploy, and manage
streaming applications that can process millions of events per second with low latency and finegrained atomic scalability.

## Core Capabilities

- **SQL-based stream processing** - Write streaming pipelines using familiar SQL syntax
- **Real-time analytics** - Process and analyze data as it arrives
- **Scalable architecture** - Horizontally scale to handle any data volume
- **Fault tolerance** - Automatic recovery with exactly-once processing guarantees
- **Rich connector ecosystem** - Connect to various data sources and sinks

## Key Concepts

### Pipelines

Pipelines are the core abstraction in Laminar. A pipeline defines:

- The data sources to read from
- The transformations to apply
- The destinations to write to

Pipelines are defined using SQL and can include:

- Filtering and projection
- Aggregations and windowing
- Joins between streams
- User-defined functions (UDFs)

### Connections

Connections define how Laminar interacts with external systems:

- **Connection Profiles** - Reusable configuration for connecting to systems
- **Connection Tables** - Schema definitions for sources and sinks

### Jobs

Jobs are running instances of pipelines. Each job:

- Executes the pipeline logic
- Maintains state and checkpoints
- Provides metrics and monitoring
- Can be stopped, started, and scaled

## Use Cases

Laminar is ideal for:

### Real-time Analytics

- Dashboard and monitoring systems
- Real-time reporting
- Metric aggregation
- Anomaly detection

### Data Integration

- Change Data Capture (CDC)
- ETL/ELT pipelines
- Data synchronization
- Event streaming

### Stream Processing

- Event-driven architectures
- Complex event processing
- Real-time ML feature computation
- IoT data processing

## Next Steps

- Understand the [Architecture](./architecture) of Laminar
- Learn [Pipeline Basics](./tutorials/pipeline-basics)
- Explore [Kafka Integration](./tutorials/kafka)
- Implement [Change Data Capture](./tutorials/cdc)