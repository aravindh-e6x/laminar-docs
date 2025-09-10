---
sidebar_position: 3
title: Core Concepts
---


Understanding Laminar's core concepts is essential for building effective streaming data pipelines. This guide explains how the different components work together to create a complete data processing system.

## Overview

Laminar follows a logical flow for setting up and running streaming pipelines:

```
Connectors → Connection Profiles → Connection Tables → Pipelines → Jobs
```

Each component builds upon the previous one, creating a flexible and reusable architecture for data processing.

## Connectors

**Connectors** are the bridge between Laminar and external systems. They define how Laminar can read from (source) or write to (sink) various data systems.

### What is a Connector?

A connector is a pre-built integration that:
- Handles the protocol and communication with external systems
- Manages data serialization/deserialization
- Provides configuration schemas for connections
- Supports specific data formats (JSON, Avro, Parquet, etc.)

### Available Connectors

Laminar includes connectors for:
- **Message Queues**: Kafka, RabbitMQ, NATS
- **Cloud Streams**: AWS Kinesis
- **Databases**: PostgreSQL, MySQL (via CDC)
- **Object Storage**: S3, GCS (via filesystem connector)
- **APIs**: Webhook, WebSocket, HTTP Polling
- **IoT**: MQTT
- **Caching**: Redis

### Example

```sql
-- Kafka connector can be used as both source and sink
-- It knows how to connect to Kafka brokers and handle Kafka-specific features
SELECT * FROM events WITH (
  connector = 'kafka',  -- Using the Kafka connector
  bootstrap_servers = 'localhost:9092',
  topic = 'user-events',
  format = 'json'
);
```

## Connection Profiles

**Connection Profiles** store reusable connection configurations for external systems. They centralize credentials and connection parameters, making them easy to manage and share across multiple tables.

### Purpose

Connection profiles allow you to:
- Store credentials securely in one place
- Reuse connection settings across multiple pipelines
- Update connection details without modifying pipelines
- Test connections before using them

### Structure

A connection profile contains:
- **Name**: Unique identifier for the profile
- **Connector Type**: Which connector this profile is for
- **Configuration**: Connection-specific settings (hosts, ports, credentials)

### Example

```json
{
  "name": "production-kafka",
  "connector": "kafka",
  "config": {
    "bootstrapServers": "kafka1.prod:9092,kafka2.prod:9092",
    "authentication": {
      "type": "SASL",
      "username": "app-user",
      "password": "{{ KAFKA_PASSWORD }}"
    }
  }
}
```

Once created, this profile can be referenced by multiple connection tables without repeating the configuration.

## Connection Tables

**Connection Tables** define the schema and specific configuration for data sources and sinks. They combine a connector (via a connection profile) with table-specific settings.

### What is a Connection Table?

A connection table:
- Defines the schema (columns and data types) for your data
- Specifies table-specific configuration (topics, queries, etc.)
- References a connection profile for connection details
- Can be either a source (input) or sink (output)

### Components

1. **Schema Definition**: Column names, data types, and nullability
2. **Table Configuration**: Specific settings like Kafka topics or database tables
3. **Format Settings**: How data is serialized (JSON, Avro, etc.)
4. **Time Settings**: Event time fields and watermark configuration

### Example

```json
{
  "name": "user_events",
  "connector": "kafka",
  "connectionProfileId": "prof_abc123",  // References the connection profile
  "config": {
    "topic": "user-events",
    "format": "json",
    "type": "source",
    "offset": "latest"
  },
  "schema": {
    "fields": [
      {"name": "user_id", "type": "BIGINT"},
      {"name": "event_type", "type": "STRING"},
      {"name": "timestamp", "type": "TIMESTAMP"}
    ]
  }
}
```

## Pipelines

**Pipelines** contain the SQL logic that transforms data from sources to sinks. They define the continuous computation that processes streaming data.

### What is a Pipeline?

A pipeline:
- Contains SQL queries that define data transformations
- References connection tables for inputs and outputs
- Runs continuously on streaming data
- Can include complex operations like joins, aggregations, and windowing

### Pipeline Components

1. **SQL Query**: The transformation logic
2. **Parallelism**: Number of parallel workers
3. **Checkpointing**: State management configuration
4. **UDFs**: Optional user-defined functions

### Example

```sql
-- Pipeline that aggregates user events
CREATE PIPELINE user_analytics AS

INSERT INTO user_metrics
SELECT 
  user_id,
  event_type,
  COUNT(*) as event_count,
  TUMBLE_START(timestamp, INTERVAL '1' HOUR) as window_start
FROM user_events  -- Source connection table
GROUP BY 
  user_id,
  event_type,
  TUMBLE(timestamp, INTERVAL '1' HOUR);
```

### Pipeline Lifecycle

1. **Created**: Pipeline is defined but not running
2. **Starting**: Resources are being allocated
3. **Running**: Actively processing data
4. **Stopping**: Gracefully shutting down
5. **Stopped**: No longer processing data
6. **Failed**: Encountered an error

## Jobs

**Jobs** are executions of pipelines. Each time you start a pipeline, it creates a job that represents that specific run.

### What is a Job?

A job:
- Is an instance of a running pipeline
- Has its own state and checkpoints
- Tracks metrics and progress
- Can be monitored and managed independently
- Maintains exactly-once processing guarantees

### Job Features

1. **Checkpointing**: Periodic state snapshots for fault tolerance
2. **Metrics**: Performance and progress monitoring
3. **Error Tracking**: Logs and error messages
4. **Output Management**: Viewing sample outputs
5. **Restart Capability**: Can resume from checkpoints

### Job States

- **Running**: Actively processing data
- **Completed**: Finished successfully
- **Failed**: Encountered an unrecoverable error
- **Stopping**: In the process of shutting down
- **Stopped**: Manually stopped by user

## Complete Flow Example

Let's walk through a complete example of setting up a data pipeline:

### Step 1: Choose a Connector
```bash
# List available connectors
GET /api/v1/connectors

# Kafka connector is available for our use case
```

### Step 2: Create Connection Profile
```bash
# Create a reusable connection to Kafka
POST /api/v1/connection_profiles
{
  "name": "prod-kafka",
  "connector": "kafka",
  "config": {
    "bootstrapServers": "kafka.prod:9092"
  }
}
```

### Step 3: Create Connection Tables
```bash
# Source table for reading events
POST /api/v1/connection_tables
{
  "name": "raw_events",
  "connectionProfileId": "prof_123",
  "config": {
    "topic": "events",
    "type": "source"
  }
}

# Sink table for writing results
POST /api/v1/connection_tables
{
  "name": "processed_events",
  "connectionProfileId": "prof_123",
  "config": {
    "topic": "processed",
    "type": "sink"
  }
}
```

### Step 4: Create Pipeline
```sql
CREATE PIPELINE event_processor AS

INSERT INTO processed_events
SELECT 
  event_id,
  user_id,
  event_type,
  -- Transform the data
  UPPER(event_type) as normalized_type,
  JSON_VALUE(properties, '$.amount') as amount,
  timestamp
FROM raw_events
WHERE event_type IS NOT NULL;
```

### Step 5: Start Pipeline (Creates a Job)
```bash
POST /api/v1/pipelines/pipe_456/start

# This creates job_789 which begins processing data
```

### Step 6: Monitor the Job
```bash
# Check job status
GET /api/v1/jobs/job_789

# View metrics
GET /api/v1/jobs/job_789/metrics

# Check for errors
GET /api/v1/jobs/job_789/errors
```
