---
sidebar_position: 2
---


This page provides a comprehensive overview of all available connectors in Laminar.

## Available Connectors

### Source Connectors

| Connector | Description |
|-----------|-------------|
| Kafka | Apache Kafka consumer for streaming data ingestion |
| Kinesis | AWS Kinesis consumer for cloud-native streaming |
| MQTT | MQTT broker subscriber for IoT data streams |
| NATS | NATS subscriber for lightweight messaging |
| RabbitMQ | RabbitMQ consumer for message queue integration |
| WebSocket | WebSocket client for real-time data streams |
| Webhook | HTTP webhook receiver for event-driven data |
| SSE | Server-Sent Events client for push notifications |
| Polling HTTP | Periodic HTTP polling for REST API integration |
| FileSystem | Read from files (JSON, CSV, Parquet) |
| Fluvio | Fluvio consumer for cloud-native streaming |
| Impulse | Internal test source for generating data |
| Nexmark | Benchmark source for auction system simulation |

### Sink Connectors

| Connector | Description |
|-----------|-------------|
| Kafka | Apache Kafka producer for streaming output |
| Kinesis | AWS Kinesis producer for cloud streaming |
| MQTT | MQTT publisher for IoT data distribution |
| NATS | NATS publisher for lightweight messaging |
| Redis | Redis writer for caching and state storage |
| Webhook | HTTP POST requests for event notifications |
| FileSystem | Write to files (JSON, Parquet) |
| Delta Lake | Delta Lake writer for data lake storage |
| Iceberg | Apache Iceberg writer for table format |
| Stdout | Standard output for debugging and development |
| Blackhole | Null sink for testing and benchmarking |

## Configuration Structure

Laminar uses a two-level configuration system for connectors:

### Connection Profiles (via API)
Reusable connection configurations created through the REST API:

```bash
# Create a connection profile for Kafka
curl -X POST http://localhost:8000/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "kafka_prod",
    "connector": "kafka",
    "config": {
      "bootstrapServers": "broker1:9092,broker2:9092",
      "authentication": {
        "protocol": "SASL_SSL",
        "mechanism": "SCRAM-SHA-256",
        "username": "user",
        "password": "secret"
      },
      "schemaRegistryEnum": {
        "endpoint": "http://registry:8081"
      }
    }
  }'
```

### Connection Tables (via API)
Table configurations that reference a connection profile:

```bash
# Create a connection table using the profile
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "orders_source",
    "connector": "kafka",
    "connectionProfileId": "kafka_prod_id",
    "config": {
      "topic": "orders",
      "type": "source",
      "offset": "earliest"
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

### Tables with Embedded Connectors (via SQL)
For simple cases, define tables with embedded connector configuration:

```sql
CREATE TABLE events (
    id BIGINT,
    timestamp TIMESTAMP,
    user_id TEXT,
    event_type TEXT
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    'source.offset' = 'earliest'
);
```

## Common Configuration Patterns

### Authentication Methods
| Method | Description | Used By |
|--------|-------------|---------|
| None | No authentication required | Local services, testing |
| SASL | Username/password authentication | Kafka, RabbitMQ |
| OAuth/JWT | Token-based authentication | HTTP, WebSocket |
| API Keys | Key/secret pairs | Cloud services |
| TLS/mTLS | Certificate-based auth | MQTT, NATS |
| AWS IAM | AWS role-based access | Kinesis, AWS MSK |

### Data Formats
| Format | Description | Supported Operations |
|--------|-------------|---------------------|
| JSON | JavaScript Object Notation | Read/Write, Schema inference |
| Avro | Binary format with schema | Read/Write, Schema Registry |
| Parquet | Columnar storage format | Read/Write, Efficient queries |
| Raw/Bytes | Unstructured binary data | Read/Write, Custom parsing |
| Debezium | CDC envelope format | Read/Write, Automatic unwrapping |

### Source Configuration Options
- **Offset Management**: `earliest`, `latest`, `group` (for message queues)
- **Consumer Groups**: Manage distributed consumption
- **Polling Intervals**: For HTTP/database sources
- **Watermark Strategies**: Control late event handling
- **Read Modes**: `read_committed`, `read_uncommitted` for transactional sources

### Sink Configuration Options
- **Commit Modes**: `at_least_once`, `exactly_once`
- **Batching**: Control batch size and flush intervals
- **Error Handling**: Retry policies, dead letter queues
- **Key Fields**: Specify partition/routing keys
- **Timestamp Fields**: Control event time in output

## Connection Management

### API Endpoints

#### Connection Profiles
- `POST /v1/connection_profiles` - Create a reusable connection profile
- `GET /v1/connection_profiles` - List all connection profiles
- `DELETE /v1/connection_profiles/:id` - Delete a connection profile
- `POST /v1/connection_profiles/test` - Test a connection profile
- `GET /v1/connection_profiles/:id/autocomplete` - Get autocomplete suggestions

#### Connection Tables
- `POST /v1/connection_tables` - Create a connection table
- `GET /v1/connection_tables` - List all connection tables
- `DELETE /v1/connection_tables/:id` - Delete a connection table
- `POST /v1/connection_tables/test` - Test a connection table
- `POST /v1/connection_tables/schemas/test` - Test schema inference

### Testing Connections

```bash
# Test a connection profile before creating it
curl -X POST http://localhost:8000/v1/connection_profiles/test \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "connector": "kafka",
    "config": {
      "bootstrapServers": "localhost:9092"
    }
  }'
```

## Complete Examples

### Kafka Setup via API

```bash
# Step 1: Create a connection profile for Kafka cluster
curl -X POST http://localhost:8000/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "production_kafka",
    "connector": "kafka",
    "config": {
      "bootstrapServers": "kafka1:9092,kafka2:9092,kafka3:9092",
      "authentication": {
        "protocol": "SASL_SSL",
        "mechanism": "SCRAM-SHA-256",
        "username": "app_user",
        "password": "secret"
      }
    }
  }'

# Response includes the profile ID
# {"id": "prof_abc123", "name": "production_kafka", ...}

# Step 2: Create a source connection table
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "orders_source",
    "connector": "kafka",
    "connectionProfileId": "prof_abc123",
    "config": {
      "topic": "orders",
      "type": "source",
      "offset": "earliest",
      "groupId": "order_processor"
    },
    "schema": {
      "format": {"json": {}},
      "fields": [
        {"name": "order_id", "type": {"primitive": "Int64"}},
        {"name": "customer_id", "type": {"primitive": "Utf8"}},
        {"name": "amount", "type": {"primitive": "Float64"}},
        {"name": "timestamp", "type": {"primitive": "Timestamp"}}
      ]
    }
  }'

# Step 3: Create a sink connection table
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "processed_orders_sink",
    "connector": "kafka",
    "connectionProfileId": "prof_abc123",
    "config": {
      "topic": "processed_orders",
      "type": "sink",
      "commitMode": "exactly_once"
    },
    "schema": {
      "format": {"json": {}}
    }
  }'
```

### Simple Table with Embedded Connector (SQL)
For development or simple cases, use SQL with embedded connector config:

```sql
CREATE TABLE events (
    id BIGINT,
    timestamp TIMESTAMP,
    user_id TEXT,
    event_type TEXT
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    'source.offset' = 'earliest'
);
```


## Metadata and Schema

### Automatic Schema Inference
Many connectors support automatic schema detection:
- JSON: Infers types from sample data
- Avro: Reads embedded schemas
- Parquet: Uses file metadata
- Schema Registry: Fetches schemas by ID

### Custom Schema Definition
```sql
CREATE TABLE events (
    id BIGINT,
    timestamp TIMESTAMP,
    user_id TEXT,
    event_type TEXT,
    properties JSON
) WITH (
    connector = 'kafka',
    topic = 'events',
    format = 'json'
);
```

### Metadata Field Extraction
Access connector metadata in queries:
- `_timestamp`: Message timestamp
- `_key`: Message key
- `_partition`: Partition number
- `_offset`: Message offset
- `_headers`: Message headers

