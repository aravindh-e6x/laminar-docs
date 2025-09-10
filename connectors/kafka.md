---
sidebar_position: 1
title: Kafka
---

The Kafka connector enables Laminar to produce and consume messages from Apache Kafka clusters with support for various authentication methods, Schema Registry integration, and delivery semantics.

## Capabilities

- **Source**: Read messages from Kafka topics
- **Sink**: Write messages to Kafka topics
- **Formats**: JSON, Avro, Protobuf, Raw bytes, Debezium
- **Schema Registry**: Confluent Schema Registry support
- **Authentication**: None, SASL, AWS MSK IAM

## Configuration

### Connection Profile Configuration

Connection profiles store reusable Kafka cluster settings. 

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `bootstrapServers` | string | Yes | - | Comma-separated list of Kafka servers (e.g., `broker-1:9092,broker-2:9092`) |
| `authentication` | object | Yes | - | Authentication configuration (see Authentication section) |
| `schemaRegistryEnum` | object | No | None | Schema Registry configuration (see Schema Registry section) |
| `connectionProperties` | object | No | - | Additional rdkafka configuration options as key-value pairs |

### Table Configuration

Table configuration defines topic-specific settings for sources and sinks.

#### Common Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `topic` | string | Yes | - | The Kafka topic to use for this table |
| `type` | object | Yes | - | Table type configuration (Source or Sink) |
| `client_configs` | object | No | - | Additional Kafka client configs |
| `value_subject` | string | No | `{topic}-value` | Schema Registry subject name |

#### Source Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `offset` | enum | Yes | - | Starting offset: `latest`, `earliest`, or `group` |
| `read_mode` | enum | No | `read_uncommitted` | Read mode: `read_uncommitted` or `read_committed` |
| `group_id` | string | No | Auto-generated | Consumer group ID (overrides group_id_prefix) |
| `group_id_prefix` | string | No | - | Prefix for auto-generated group ID |

#### Sink Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `commit_mode` | enum | Yes | - | Commit mode: `at_least_once` or `exactly_once` |
| `key_field` | string | No | - | Field to use as message key |
| `timestamp_field` | string | No | Event time | Field to use as message timestamp |

## Authentication

### No Authentication

```json
{
  "authentication": {}
}
```

### SASL Authentication

```json
{
  "authentication": {
    "protocol": "SASL_SSL",
    "mechanism": "SCRAM-SHA-256",
    "username": "kafka_user",
    "password": "{{ KAFKA_PASSWORD }}"
  }
}
```

**SASL Fields:**
- `protocol`: SASL protocol (e.g., `SASL_PLAINTEXT`, `SASL_SSL`)
- `mechanism`: SASL mechanism (e.g., `SCRAM-SHA-256`, `SCRAM-SHA-512`, `PLAIN`)
- `username`: Username for authentication (supports env vars with `{{ VAR }}`)
- `password`: Password for authentication (supports env vars with `{{ VAR }}`)

### AWS MSK IAM Authentication

```json
{
  "authentication": {
    "region": "us-east-1"
  }
}
```

## Schema Registry

### No Schema Registry

```json
{
  "schemaRegistryEnum": {}
}
```

### Confluent Schema Registry

```json
{
  "schemaRegistryEnum": {
    "endpoint": "http://schema-registry:8081",
    "apiKey": "{{ SCHEMA_REGISTRY_KEY }}",
    "apiSecret": "{{ SCHEMA_REGISTRY_SECRET }}"
  }
}
```

**Fields:**
- `endpoint`: Schema Registry URL (required)
- `apiKey`: API key for authentication (optional, supports env vars)
- `apiSecret`: API secret for authentication (optional, supports env vars)

## API Usage

### Create Connection Profile

```bash
curl -X POST http://localhost:8000/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "production_kafka",
    "connector": "kafka",
    "config": {
      "bootstrapServers": "broker1:9092,broker2:9092,broker3:9092",
      "authentication": {
        "protocol": "SASL_SSL",
        "mechanism": "SCRAM-SHA-256",
        "username": "app_user",
        "password": "{{ KAFKA_PASSWORD }}"
      },
      "schemaRegistryEnum": {
        "endpoint": "http://schema-registry:8081",
        "apiKey": "{{ REGISTRY_KEY }}",
        "apiSecret": "{{ REGISTRY_SECRET }}"
      },
      "connectionProperties": {
        "socket.timeout.ms": "60000",
        "session.timeout.ms": "30000"
      }
    }
  }'
```

### Create Source Table

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "events_source",
    "connector": "kafka",
    "connectionProfileId": "prof_abc123",
    "config": {
      "topic": "events",
      "type": {
        "offset": "earliest",
        "read_mode": "read_committed",
        "group_id": "event_processor"
      },
      "client_configs": {
        "max.poll.records": "1000"
      }
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

### Create Sink Table

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "events_sink",
    "connector": "kafka",
    "connectionProfileId": "prof_abc123",
    "config": {
      "topic": "processed_events",
      "type": {
        "commit_mode": "exactly_once",
        "key_field": "user_id",
        "timestamp_field": "event_time"
      }
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

## SQL Usage

### Simple Source Table

```sql
CREATE TABLE events (
    id BIGINT,
    user_id TEXT,
    event_type TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    'source.offset' = 'earliest'
);
```

### Source with Authentication

```sql
CREATE TABLE secure_events (
    id BIGINT,
    data TEXT
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'broker1:9092,broker2:9092',
    topic = 'secure_events',
    type = 'source',
    format = 'json',
    'auth.type' = 'sasl',
    'auth.protocol' = 'SASL_SSL',
    'auth.mechanism' = 'SCRAM-SHA-256',
    'auth.username' = 'reader',
    'auth.password' = '{{ KAFKA_PASSWORD }}',
    'source.offset' = 'latest',
    'source.group_id' = 'secure_reader'
);
```

### Sink Table

```sql
CREATE TABLE output_events (
    id BIGINT,
    result TEXT,
    processed_at TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'processed',
    type = 'sink',
    format = 'json',
    'sink.commit_mode' = 'exactly_once',
    'sink.key_field' = 'id'
);
```

### With Schema Registry

```sql
CREATE TABLE avro_events (
    id BIGINT,
    data TEXT
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'avro_events',
    type = 'source',
    format = 'avro',
    'schema_registry.endpoint' = 'http://registry:8081',
    'schema_registry.api_key' = '{{ REGISTRY_KEY }}',
    'schema_registry.api_secret' = '{{ REGISTRY_SECRET }}',
    'source.offset' = 'earliest'
);
```

## Examples

### Event Stream Processing

```sql
-- Source table for raw events
CREATE TABLE raw_events (
    event_id BIGINT,
    user_id TEXT,
    event_type TEXT,
    payload JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'kafka:9092',
    topic = 'raw_events',
    type = 'source',
    format = 'json',
    'source.offset' = 'earliest',
    'source.group_id' = 'event_processor'
);

-- Sink table for processed events
CREATE TABLE processed_events (
    event_id BIGINT,
    user_id TEXT,
    event_category TEXT,
    score DOUBLE,
    processed_at TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'kafka:9092',
    topic = 'processed_events',
    type = 'sink',
    format = 'json',
    'sink.commit_mode' = 'exactly_once',
    'sink.key_field' = 'user_id'
);

-- Processing pipeline
INSERT INTO processed_events
SELECT 
    event_id,
    user_id,
    CASE 
        WHEN event_type LIKE 'purchase%' THEN 'transaction'
        WHEN event_type LIKE 'view%' THEN 'browse'
        ELSE 'other'
    END as event_category,
    CAST(JSON_VALUE(payload, '$.score') AS DOUBLE) as score,
    NOW() as processed_at
FROM raw_events
WHERE event_type IS NOT NULL;
```

### CDC with Debezium Format

```sql
-- Source table for CDC events
CREATE TABLE cdc_orders (
    order_id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    total DECIMAL,
    status TEXT,
    updated_at TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'mysql.shop.orders',
    type = 'source',
    format = 'debezium_json',
    'source.offset' = 'earliest'
);

-- Debezium format automatically handles:
-- - Insert/Update/Delete operations
-- - Before/After states
-- - Schema evolution
```

### Multi-Topic Consumer

```sql
-- Consume from multiple topics using regex
CREATE TABLE all_events (
    topic TEXT,
    partition INT,
    offset BIGINT,
    key TEXT,
    value TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'events-.*',  -- Regex pattern for multiple topics
    type = 'source',
    format = 'raw',
    'source.offset' = 'latest'
);
```

## Best Practices

1. **Consumer Groups**: Always specify a meaningful `group_id` for sources to enable offset management and parallel consumption
2. **Commit Modes**: Use `exactly_once` for critical data, `at_least_once` for better performance
3. **Schema Registry**: Use Schema Registry for schema evolution and compatibility checking
4. **Authentication**: Store credentials in environment variables using `{{ VAR }}` syntax
5. **Client Configs**: Tune `client_configs` for performance based on your workload
6. **Monitoring**: Track consumer lag and offset progression for sources

## Troubleshooting

### Common Issues

1. **Connection Refused**: Verify `bootstrap_servers` addresses and network connectivity
2. **Authentication Failed**: Check credentials and authentication mechanism
3. **Consumer Lag**: Increase parallelism or optimize processing logic
4. **Schema Registry Errors**: Verify endpoint URL and API credentials
5. **Offset Reset**: Use `group` offset with existing consumer group or `earliest`/`latest` for new groups