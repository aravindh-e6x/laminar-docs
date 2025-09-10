---
sidebar_position: 5
---

# Kinesis Sink

The Kinesis sink connector enables Laminar to publish processed data to Amazon Kinesis Data Streams for downstream consumption.

## Overview

Amazon Kinesis Data Streams sink allows you to write streaming data to Kinesis for real-time analytics, data lake ingestion, or triggering downstream processing.

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `stream_name` | string | Yes | Kinesis stream name |
| `region` | string | Yes | AWS region |
| `aws_access_key_id` | string | No | AWS access key ID |
| `aws_secret_access_key` | string | No | AWS secret access key |
| `role_arn` | string | No | IAM role ARN for assume role |
| `endpoint` | string | No | Custom endpoint (for LocalStack) |

### Producer Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `partition_key_field` | string | No | Field to use as partition key |
| `partition_strategy` | string | No | Partitioning strategy (hash, random, round_robin) |
| `aggregation` | boolean | No | Enable record aggregation (default: false) |
| `compression` | string | No | Compression type (none, gzip, snappy) |

## SQL Example

```sql
CREATE CONNECTION kinesis_sink_conn FROM kinesis_sink
WITH (
    stream_name = 'processed-events',
    region = 'us-east-1',
    aws_access_key_id = '${AWS_ACCESS_KEY}',
    aws_secret_access_key = '${AWS_SECRET_KEY}'
);

INSERT INTO kinesis_output
SELECT 
    event_id,
    processed_time,
    result,
    metadata
FROM processed_events
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    partition_key_field = 'event_id',
    format = 'json'
);
```

## Partitioning Strategies

### Hash Partitioning

```sql
INSERT INTO kinesis_partitioned
SELECT * FROM events
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    partition_strategy = 'hash',
    partition_key_field = 'user_id'
);
```

### Custom Partition Key

```sql
INSERT INTO kinesis_custom_key
SELECT 
    CONCAT(region, '-', user_id) as partition_key,
    *
FROM events
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    partition_key_field = 'partition_key'
);
```

## Data Formats

### JSON Format

```sql
INSERT INTO kinesis_json
SELECT * FROM events
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    format = 'json',
    json_compact = false
);
```

### Avro Format

```sql
INSERT INTO kinesis_avro
SELECT * FROM events
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    format = 'avro',
    schema_registry_url = 'http://schema-registry:8081'
);
```

## Performance Optimization

### Batch Configuration

```sql
INSERT INTO kinesis_batched
SELECT * FROM high_volume_events
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    batch_size = 500,
    batch_timeout_ms = 100,
    max_batch_bytes = 5242880
);
```

### Aggregation

```sql
INSERT INTO kinesis_aggregated
SELECT * FROM events
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    aggregation = true,
    aggregation_max_records = 100,
    aggregation_max_size = 1048576
);
```

## Monitoring

### Metrics

- `laminar_kinesis_records_sent` - Total records sent
- `laminar_kinesis_bytes_sent` - Total bytes sent
- `laminar_kinesis_put_record_latency` - PutRecords API latency
- `laminar_kinesis_throttles` - Throttling errors
- `laminar_kinesis_failures` - Failed record attempts

## Best Practices

1. **Partition Key Selection**: Choose keys that distribute data evenly
2. **Aggregation**: Enable for high-throughput scenarios
3. **Batch Size**: Balance between latency and throughput
4. **Error Handling**: Implement retry logic for transient failures
5. **Monitoring**: Track shard metrics and scaling needs

## Example Use Cases

### Real-time Analytics Pipeline

```sql
INSERT INTO kinesis_analytics
SELECT 
    user_id,
    event_type,
    timestamp,
    COUNT(*) as event_count,
    AVG(value) as avg_value
FROM raw_events
GROUP BY TUMBLE(timestamp, INTERVAL '1 minute'), user_id, event_type
WITH (
    connector = 'kinesis_sink',
    connection = 'kinesis_sink_conn',
    stream_name = 'analytics-stream'
);
```