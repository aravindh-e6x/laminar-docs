---
sidebar_position: 1
title: Kafka
---

# Kafka Sink Connector

The Kafka sink connector enables Laminar to produce messages to Apache Kafka topics with exactly-once delivery guarantees and flexible serialization options.

## Overview

The Kafka sink provides:
- **Exactly-once semantics** with transactional writes
- **Multiple serialization formats** (JSON, Avro, Protobuf)
- **Schema Registry integration**
- **Partitioning strategies**
- **Compression support**
- **Batch optimization**
- **Error handling and retries**

## Configuration

### Basic Configuration

```sql
CREATE CONNECTION kafka_sink TO kafka (
    bootstrap_servers = 'broker1:9092,broker2:9092',
    topic = 'output-events',
    type = 'sink',
    format = 'json'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `bootstrap_servers` | string | Yes | - | Comma-separated list of Kafka brokers |
| `topic` | string | Yes | - | Target Kafka topic |
| `type` | string | Yes | - | Must be 'sink' for producing |
| `format` | string | Yes | - | Data format (json, avro, protobuf, raw) |
| `key_field` | string | No | - | Field to use as message key |
| `partition_field` | string | No | - | Field for custom partitioning |
| `compression` | string | No | `snappy` | Compression (none, gzip, snappy, lz4, zstd) |
| `batch_size` | integer | No | 16384 | Max messages per batch |
| `linger_ms` | integer | No | 100 | Max wait time before sending batch |

## Usage Examples

### Simple JSON Sink

```sql
-- Create sink connection
CREATE CONNECTION kafka_json_sink TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'processed-events',
    type = 'sink',
    format = 'json'
);

-- Use in pipeline
CREATE PIPELINE process_events AS
INSERT INTO kafka_json_sink
SELECT 
    id,
    user_id,
    event_type,
    timestamp,
    TO_JSON(metadata) as metadata
FROM source_events
WHERE event_type = 'purchase';
```

### Sink with Message Key

```sql
-- Configure with message key for partitioning
CREATE CONNECTION kafka_keyed_sink TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'user-events',
    type = 'sink',
    format = 'json',
    key_field = 'user_id'  -- Use user_id as message key
);

-- Pipeline ensures ordering per user
CREATE PIPELINE user_aggregates AS
INSERT INTO kafka_keyed_sink
SELECT 
    user_id,
    COUNT(*) as event_count,
    SUM(amount) as total_amount,
    MAX(timestamp) as last_activity
FROM events
GROUP BY user_id, TUMBLE(timestamp, INTERVAL '1 hour');
```

## Serialization Formats

### JSON Format

```sql
CREATE CONNECTION kafka_json_output TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'json-output',
    type = 'sink',
    format = 'json',
    'json.pretty' = 'false',
    'json.include_nulls' = 'true'
);

-- Pipeline with JSON formatting
INSERT INTO kafka_json_output
SELECT 
    id,
    name,
    CASE 
        WHEN score > 90 THEN 'A'
        WHEN score > 80 THEN 'B'
        ELSE 'C'
    END as grade,
    TO_JSON(details) as details_json
FROM students;
```

### Avro Format with Schema Registry

```sql
CREATE CONNECTION kafka_avro_sink TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'avro-output',
    type = 'sink',
    format = 'avro',
    'schema_registry.endpoint' = 'http://schema-registry:8081',
    'schema_registry.api_key' = '${secret:sr_key}',
    'schema_registry.api_secret' = '${secret:sr_secret}',
    'value_subject' = 'avro-output-value',
    'auto.register.schemas' = 'true'
);
```

### Protobuf Format

```sql
CREATE CONNECTION kafka_protobuf_sink TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'protobuf-output',
    type = 'sink',
    format = 'protobuf',
    'protobuf.message' = 'com.example.Event',
    'schema_registry.endpoint' = 'http://schema-registry:8081'
);
```

### Raw Binary Format

```sql
CREATE CONNECTION kafka_raw_sink TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'binary-data',
    type = 'sink',
    format = 'raw'
);

-- Send raw bytes
INSERT INTO kafka_raw_sink
SELECT 
    encode(data, 'base64') as value,
    encode(key, 'utf8') as key
FROM binary_stream;
```

## Partitioning Strategies

### Hash Partitioning

```sql
CREATE CONNECTION kafka_hash_partition TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'partitioned-events',
    type = 'sink',
    format = 'json',
    key_field = 'customer_id',  -- Hash based on customer_id
    'partitioner' = 'murmur2_random'  -- Consistent hashing
);
```

### Custom Partitioning

```sql
CREATE CONNECTION kafka_custom_partition TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'geo-events',
    type = 'sink',
    format = 'json',
    partition_field = 'region_partition'  -- Explicit partition field
);

-- Pipeline with calculated partition
INSERT INTO kafka_custom_partition
SELECT 
    *,
    CASE region
        WHEN 'US' THEN 0
        WHEN 'EU' THEN 1
        WHEN 'ASIA' THEN 2
        ELSE 3
    END as region_partition
FROM events;
```

### Round-Robin Partitioning

```sql
CREATE CONNECTION kafka_round_robin TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'balanced-events',
    type = 'sink',
    format = 'json'
    -- No key_field means round-robin distribution
);
```

## Transactional Writes

### Exactly-Once Semantics

```sql
CREATE CONNECTION kafka_transactional TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'critical-events',
    type = 'sink',
    format = 'json',
    'transactional.id' = 'laminar-producer-1',
    'enable.idempotence' = 'true',
    'acks' = 'all',
    'retries' = '10',
    'max.in.flight.requests.per.connection' = '5'
);
```

### Two-Phase Commit

```sql
-- Sink with two-phase commit for consistency
CREATE CONNECTION kafka_2pc TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'financial-transactions',
    type = 'sink',
    format = 'json',
    'transactional.id' = 'financial-processor',
    'transaction.timeout.ms' = '60000',
    'enable.idempotence' = 'true'
);

-- Coordinated pipeline
CREATE PIPELINE financial_processing AS
WITH validated_transactions AS (
    SELECT * FROM transactions
    WHERE amount > 0 AND status = 'valid'
)
INSERT INTO kafka_2pc
SELECT * FROM validated_transactions;
```

## Performance Optimization

### Batching Configuration

```sql
CREATE CONNECTION kafka_optimized TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'high-volume',
    type = 'sink',
    format = 'json',
    
    -- Batching settings
    'batch.size' = '32768',  -- 32KB batches
    'linger.ms' = '100',     -- Wait up to 100ms
    'buffer.memory' = '67108864',  -- 64MB buffer
    
    -- Compression
    'compression.type' = 'snappy',
    
    -- Network settings
    'send.buffer.bytes' = '131072',
    'max.request.size' = '1048576'
);
```

### Async Writes

```sql
CREATE CONNECTION kafka_async TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'async-events',
    type = 'sink',
    format = 'json',
    'acks' = '1',  -- Leader acknowledgment only
    'max.in.flight.requests.per.connection' = '10',
    'retries' = '3'
);
```

## Error Handling

### Retry Configuration

```sql
CREATE CONNECTION kafka_with_retry TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'reliable-events',
    type = 'sink',
    format = 'json',
    'retries' = '10',
    'retry.backoff.ms' = '100',
    'request.timeout.ms' = '30000',
    'delivery.timeout.ms' = '120000'
);
```

### Dead Letter Queue

```sql
-- Main sink
CREATE CONNECTION kafka_main_sink TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'processed-events',
    type = 'sink',
    format = 'json'
);

-- DLQ sink
CREATE CONNECTION kafka_dlq_sink TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'failed-events-dlq',
    type = 'sink',
    format = 'json'
);

-- Pipeline with error handling
CREATE PIPELINE with_error_handling AS
WITH processed AS (
    SELECT 
        *,
        CASE 
            WHEN is_valid(data) THEN 'main'
            ELSE 'dlq'
        END as target
    FROM source_events
)
INSERT INTO kafka_main_sink
SELECT * FROM processed WHERE target = 'main'
UNION ALL
INSERT INTO kafka_dlq_sink
SELECT * FROM processed WHERE target = 'dlq';
```

## Headers and Metadata

### Adding Message Headers

```sql
-- Connection with header support
CREATE CONNECTION kafka_with_headers TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'events-with-headers',
    type = 'sink',
    format = 'json'
);

-- Pipeline adding headers
CREATE PIPELINE add_headers AS
INSERT INTO kafka_with_headers
SELECT 
    *,
    MAP {
        'source': 'laminar',
        'version': '1.0',
        'timestamp': CAST(NOW() AS STRING),
        'correlation_id': UUID()
    } as _kafka_headers
FROM events;
```

## Multi-Topic Sink

### Dynamic Topic Routing

```sql
-- Base connection
CREATE CONNECTION kafka_multi TO kafka (
    bootstrap_servers = 'localhost:9092',
    type = 'sink',
    format = 'json',
    topic_field = '_target_topic'  -- Dynamic topic from field
);

-- Pipeline with topic routing
CREATE PIPELINE route_by_type AS
INSERT INTO kafka_multi
SELECT 
    *,
    CASE event_type
        WHEN 'order' THEN 'orders-topic'
        WHEN 'payment' THEN 'payments-topic'
        WHEN 'shipment' THEN 'shipments-topic'
        ELSE 'misc-topic'
    END as _target_topic
FROM events;
```

## Monitoring

### Producer Metrics

```sql
-- Monitor producer performance
SELECT 
    connection_name,
    messages_sent,
    bytes_sent,
    send_rate,
    error_rate,
    avg_batch_size,
    avg_request_latency_ms
FROM system.kafka_producer_metrics
WHERE connection_name = 'kafka_sink';
```

### Queue Status

```sql
-- Check producer buffer status
SELECT 
    connection_name,
    buffer_available_bytes,
    buffer_total_bytes,
    waiting_threads,
    metadata_age_ms
FROM system.kafka_producer_status
WHERE connection_name = 'kafka_sink';
```

## Best Practices

1. **Use transactional writes** for critical data
2. **Configure appropriate batch sizes** based on throughput
3. **Set proper compression** to reduce network usage
4. **Use message keys** for ordering guarantees
5. **Monitor producer metrics** for performance issues
6. **Configure retries** for temporary failures
7. **Use async writes** for high throughput
8. **Set appropriate timeouts** based on SLAs
9. **Implement error handling** with DLQs
10. **Test failover scenarios** regularly

## Troubleshooting

### Connection Test

```sql
-- Test sink connection
SELECT test_connection('kafka_sink');

-- Verify topic exists
SELECT * FROM system.kafka_topics
WHERE connection = 'kafka_sink';
```

### Performance Analysis

```sql
-- Analyze write performance
WITH producer_stats AS (
    SELECT 
        DATE_TRUNC('minute', timestamp) as minute,
        COUNT(*) as messages,
        SUM(bytes) as total_bytes,
        AVG(latency_ms) as avg_latency,
        MAX(latency_ms) as max_latency
    FROM system.kafka_producer_logs
    WHERE connection = 'kafka_sink'
        AND timestamp > NOW() - INTERVAL '1 hour'
    GROUP BY DATE_TRUNC('minute', timestamp)
)
SELECT * FROM producer_stats
ORDER BY minute DESC;
```