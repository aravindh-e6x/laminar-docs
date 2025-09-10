---
sidebar_position: 1
title: Kafka
---

# Kafka Source Connector

The Kafka source connector enables Laminar to consume messages from Apache Kafka topics in real-time. It supports various authentication methods, message formats, and delivery semantics.

## Overview

The Kafka connector provides:
- **Exactly-once semantics** with checkpointing
- **Multiple authentication methods** (SASL, AWS MSK IAM, SSL)
- **Schema Registry integration** (Confluent, AWS Glue)
- **Multiple data formats** (JSON, Avro, Protobuf, Raw)
- **Consumer group management**
- **Automatic offset management**
- **Parallel consumption** from multiple partitions

## Configuration

### Connection Configuration

```sql
CREATE CONNECTION kafka_source TO kafka (
    bootstrap_servers = 'broker1:9092,broker2:9092,broker3:9092',
    topic = 'events',
    group_id = 'laminar-consumer-group',
    type = 'source',
    format = 'json'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `bootstrap_servers` | string | Yes | - | Comma-separated list of Kafka brokers |
| `topic` | string | Yes | - | Kafka topic to consume from |
| `group_id` | string | No | `laminar-{uuid}` | Consumer group ID |
| `type` | string | Yes | - | Must be 'source' for consuming |
| `format` | string | Yes | - | Data format (json, avro, protobuf, raw) |
| `offset` | string | No | `earliest` | Starting offset (earliest, latest, timestamp) |
| `timestamp` | string | No | - | Start timestamp when offset='timestamp' |
| `batch_size` | integer | No | 1000 | Max records per batch |
| `batch_timeout_ms` | integer | No | 1000 | Max wait time for batch |

## Authentication

### No Authentication

```sql
CREATE CONNECTION kafka_public TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'public-events',
    type = 'source',
    format = 'json'
);
```

### SASL/PLAIN

```sql
CREATE CONNECTION kafka_sasl TO kafka (
    bootstrap_servers = 'broker:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    'auth.type' = 'sasl',
    'auth.mechanism' = 'PLAIN',
    'auth.protocol' = 'SASL_SSL',
    'auth.username' = 'user',
    'auth.password' = '${secret:kafka_password}'
);
```

### SASL/SCRAM

```sql
CREATE CONNECTION kafka_scram TO kafka (
    bootstrap_servers = 'broker:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    'auth.type' = 'sasl',
    'auth.mechanism' = 'SCRAM-SHA-256',
    'auth.protocol' = 'SASL_SSL',
    'auth.username' = 'user',
    'auth.password' = '${secret:kafka_password}'
);
```

### AWS MSK IAM

```sql
CREATE CONNECTION kafka_msk TO kafka (
    bootstrap_servers = 'b-1.msk-cluster.xxx.kafka.us-east-1.amazonaws.com:9098',
    topic = 'events',
    type = 'source',
    format = 'json',
    'auth.type' = 'aws_msk_iam',
    'auth.region' = 'us-east-1'
);
```

### SSL/TLS

```sql
CREATE CONNECTION kafka_ssl TO kafka (
    bootstrap_servers = 'broker:9093',
    topic = 'events',
    type = 'source',
    format = 'json',
    'security.protocol' = 'SSL',
    'ssl.ca.location' = '/certs/ca-cert.pem',
    'ssl.certificate.location' = '/certs/client-cert.pem',
    'ssl.key.location' = '/certs/client-key.pem',
    'ssl.key.password' = '${secret:key_password}'
);
```

## Data Formats

### JSON Format

```sql
CREATE CONNECTION kafka_json TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'json-events',
    type = 'source',
    format = 'json',
    'json.unstructured' = 'false'
);

-- Usage
CREATE TABLE events (
    id BIGINT,
    name TEXT,
    timestamp TIMESTAMP,
    metadata JSON
) WITH (
    connector = 'kafka_json'
);
```

### Avro Format

```sql
CREATE CONNECTION kafka_avro TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'avro-events',
    type = 'source',
    format = 'avro',
    'schema_registry.endpoint' = 'http://schema-registry:8081',
    'schema_registry.api_key' = 'api_key',
    'schema_registry.api_secret' = '${secret:sr_secret}'
);
```

### Confluent Schema Registry

```sql
CREATE CONNECTION kafka_confluent TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'confluent_avro',
    'schema_registry.endpoint' = 'http://confluent-sr:8081',
    'value_subject' = 'events-value',
    'schema_registry.api_key' = '${secret:confluent_key}',
    'schema_registry.api_secret' = '${secret:confluent_secret}'
);
```

### Raw Format

```sql
CREATE CONNECTION kafka_raw TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'raw-data',
    type = 'source',
    format = 'raw'
);

-- Usage with raw bytes
CREATE TABLE raw_events (
    key BYTEA,
    value BYTEA,
    timestamp TIMESTAMP,
    partition INT,
    offset BIGINT
) WITH (
    connector = 'kafka_raw',
    include_metadata = 'true'
);
```

## Metadata Fields

Access Kafka metadata in your queries:

```sql
CREATE TABLE events_with_metadata (
    -- Data fields
    id BIGINT,
    message TEXT,
    
    -- Kafka metadata fields
    _kafka_topic TEXT,
    _kafka_partition INT,
    _kafka_offset BIGINT,
    _kafka_timestamp TIMESTAMP,
    _kafka_key BYTEA,
    _kafka_headers JSON
) WITH (
    connector = 'kafka_source',
    include_metadata = 'true'
);

-- Query with metadata
SELECT 
    id,
    message,
    _kafka_partition,
    _kafka_offset,
    _kafka_timestamp
FROM events_with_metadata
WHERE _kafka_partition = 0;
```

## Consumer Groups

### Automatic Group Management

```sql
-- Laminar automatically manages consumer groups
CREATE CONNECTION kafka_auto_group TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json'
    -- group_id is auto-generated
);
```

### Manual Group Assignment

```sql
-- Specify custom consumer group
CREATE CONNECTION kafka_custom_group TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    group_id = 'my-application-group'
);
```

### Multiple Consumers

```sql
-- Multiple pipelines can share a consumer group for load balancing
CREATE CONNECTION kafka_shared TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'high-volume-events',
    type = 'source',
    format = 'json',
    group_id = 'shared-processing-group'
);

-- Pipeline 1
CREATE PIPELINE processor_1 AS
SELECT * FROM events WHERE MOD(hash(id), 2) = 0;

-- Pipeline 2 (shares the same group)
CREATE PIPELINE processor_2 AS
SELECT * FROM events WHERE MOD(hash(id), 2) = 1;
```

## Offset Management

### Start from Beginning

```sql
CREATE CONNECTION kafka_from_start TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    offset = 'earliest'
);
```

### Start from End

```sql
CREATE CONNECTION kafka_from_end TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    offset = 'latest'
);
```

### Start from Timestamp

```sql
CREATE CONNECTION kafka_from_timestamp TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    offset = 'timestamp',
    timestamp = '2024-01-01T00:00:00Z'
);
```

### Commit Strategies

```sql
CREATE CONNECTION kafka_commit TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    'enable.auto.commit' = 'false',
    'commit.interval.ms' = '5000',
    'auto.offset.reset' = 'earliest'
);
```

## Advanced Configuration

### Custom Kafka Properties

```sql
CREATE CONNECTION kafka_advanced TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    
    -- Performance tuning
    'fetch.min.bytes' = '1024',
    'fetch.max.wait.ms' = '500',
    'max.partition.fetch.bytes' = '1048576',
    
    -- Connection settings
    'session.timeout.ms' = '30000',
    'heartbeat.interval.ms' = '3000',
    'connections.max.idle.ms' = '540000',
    
    -- Consumer settings
    'max.poll.records' = '500',
    'max.poll.interval.ms' = '300000'
);
```

### Multiple Topics

```sql
-- Subscribe to multiple topics with pattern
CREATE CONNECTION kafka_pattern TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic_pattern = 'events-.*',  -- Regex pattern
    type = 'source',
    format = 'json'
);

-- Subscribe to topic list
CREATE CONNECTION kafka_list TO kafka (
    bootstrap_servers = 'localhost:9092',
    topics = 'events,orders,users',  -- Comma-separated list
    type = 'source',
    format = 'json'
);
```

## Error Handling

### Dead Letter Queue

```sql
CREATE CONNECTION kafka_with_dlq TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    bad_data = 'dlq',  -- Send bad records to DLQ
    dlq_topic = 'events-dlq'
);
```

### Skip Bad Records

```sql
CREATE CONNECTION kafka_skip_bad TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events',
    type = 'source',
    format = 'json',
    bad_data = 'skip',  -- Skip malformed records
    max_errors_per_batch = '10'
);
```

## Monitoring

### Consumer Lag Metrics

```sql
-- Monitor consumer lag
SELECT 
    _kafka_partition,
    MAX(_kafka_offset) as latest_offset,
    COUNT(*) as messages_processed,
    MAX(_kafka_timestamp) as latest_message_time
FROM events_with_metadata
GROUP BY _kafka_partition;
```

### Connection Health

```sql
-- Check connection status
SELECT * FROM system.connections 
WHERE name = 'kafka_source';

-- View consumer group details
SELECT * FROM system.kafka_consumer_groups
WHERE group_id = 'my-application-group';
```

## Best Practices

1. **Use appropriate consumer groups** for parallel processing
2. **Configure proper batch sizes** based on message size and volume
3. **Set reasonable timeouts** for network and processing
4. **Monitor consumer lag** to detect processing delays
5. **Use schema registry** for schema evolution
6. **Configure proper authentication** for production
7. **Set up dead letter queues** for error handling
8. **Use checkpointing** for exactly-once processing
9. **Tune fetch parameters** for optimal throughput
10. **Monitor partition distribution** for balanced consumption

## Testing the Kafka Connector

### Quick Test with Docker Compose

We provide a complete testing environment with Docker Compose that includes Arroyo, Kafka, and automatic data generation.

#### 1. Start the Test Environment

```bash
cd ../../docker-compose/kafka
docker-compose up -d
```

This starts:
- Arroyo cluster (API on port 5115)
- PostgreSQL for Arroyo metadata
- Kafka broker with Zookeeper
- Automatic test data generator
- Kafka UI for visualization (port 8080)

#### 2. Run Tests with Postman

Import the Postman collection and run tests:

```bash
# Using Newman CLI
newman run ../../postman-collections/Kafka-Connector-Tests.postman_collection.json \
  --folder "Kafka Tests" \
  --environment ../../postman-collections/environments/local.postman_environment.json

# Or use Postman UI - import the collection and environment
```

#### 3. Test Scenarios Available

- **Kafka Source Test**: Kafka → Preview (verify reading from Kafka)
- **Kafka Sink Test**: Impulse → Kafka (verify writing to Kafka)
- **End-to-End Test**: Kafka → Transform → Kafka (complete pipeline)

#### 4. Verify Results

```bash
# Check data in Kafka topics
docker-compose exec kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic test-output \
  --from-beginning

# View in Kafka UI
open http://localhost:8080
```

#### 5. Clean Up

```bash
docker-compose down -v
```

For detailed testing instructions, see [Kafka Testing Guide](../../docker-compose/kafka/README.md).

## Troubleshooting

### Connection Issues

```sql
-- Test connection
SELECT test_connection('kafka_source');

-- Check broker connectivity
SELECT * FROM system.kafka_brokers
WHERE connection = 'kafka_source';
```

### Consumer Group Issues

```bash
# Reset consumer group offset
laminar kafka reset-offset \
  --connection kafka_source \
  --group my-group \
  --to-earliest

# List consumer groups
laminar kafka list-groups \
  --connection kafka_source
```

### Performance Issues

```sql
-- Analyze consumption rate
WITH consumption_stats AS (
    SELECT 
        DATE_TRUNC('minute', _kafka_timestamp) as minute,
        COUNT(*) as messages,
        AVG(EXTRACT(EPOCH FROM (NOW() - _kafka_timestamp))) as lag_seconds
    FROM events_with_metadata
    WHERE _kafka_timestamp > NOW() - INTERVAL '1 hour'
    GROUP BY DATE_TRUNC('minute', _kafka_timestamp)
)
SELECT * FROM consumption_stats
ORDER BY minute DESC;
```