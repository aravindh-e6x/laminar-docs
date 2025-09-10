---
sidebar_position: 1
---

# Introduction to Connectors

Laminar provides a comprehensive set of connectors for integrating with various data sources and sinks. This section covers all available connectors and their configurations.

## What are Connectors?

Connectors are the primary mechanism for getting data into and out of Laminar. They handle:

- **Data ingestion** from external systems (sources)
- **Data output** to external systems (sinks)
- **Format conversion** between different data formats
- **Schema management** and validation

## Types of Connectors

### Sources
Sources bring data into Laminar from external systems:
- Streaming sources (Kafka, Kinesis, WebSocket)
- Database CDC sources (MySQL, PostgreSQL)
- HTTP sources (Webhook, Polling)

### Sinks
Sinks send processed data to external systems:
- Message queues (Kafka, Redis)
- Databases (PostgreSQL)
- Data lakes (Delta Lake, Iceberg)

### Formats
Laminar supports various data formats:
- JSON
- Avro
- Parquet
- Protocol Buffers

## Getting Started

To use a connector, you need to:

1. Choose the appropriate connector type
2. Configure the connection properties
3. Define the data schema
4. Create a pipeline using the connector

Each connector has specific configuration requirements documented in its respective section.