---
sidebar_position: 3
title: Kinesis
---

# AWS Kinesis Source Connector

The Kinesis source connector enables Laminar to consume data from Amazon Kinesis Data Streams with automatic scaling, resharding support, and exactly-once processing semantics.

## Overview

The Kinesis connector provides:
- **Real-time stream consumption** from Kinesis Data Streams
- **Automatic shard discovery** and rebalancing
- **Enhanced fan-out** support for lower latency
- **Exactly-once processing** with checkpointing
- **Cross-region support**
- **IAM and access key authentication**
- **Automatic retry and error handling**

## Configuration

### Basic Configuration

```sql
CREATE CONNECTION kinesis_source TO kinesis (
    stream_name = 'my-data-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `stream_name` | string | Yes | - | Kinesis stream name |
| `region` | string | Yes | - | AWS region |
| `type` | string | Yes | - | Must be 'source' |
| `format` | string | Yes | - | Data format (json, avro, raw) |
| `starting_position` | string | No | `LATEST` | Starting position (LATEST, TRIM_HORIZON, AT_TIMESTAMP) |
| `starting_timestamp` | string | No | - | Timestamp when using AT_TIMESTAMP |
| `consumer_name` | string | No | `laminar-consumer` | Enhanced fan-out consumer name |
| `max_records_per_fetch` | integer | No | 10000 | Max records per GetRecords call |
| `fetch_interval_ms` | integer | No | 1000 | Polling interval |

## Authentication

### IAM Role (Recommended)

```sql
-- Using IAM role attached to EC2/ECS/Lambda
CREATE CONNECTION kinesis_iam TO kinesis (
    stream_name = 'my-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json'
    -- No credentials needed, uses IAM role
);
```

### Access Keys

```sql
CREATE CONNECTION kinesis_keys TO kinesis (
    stream_name = 'my-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    access_key_id = '${secret:aws_access_key}',
    secret_access_key = '${secret:aws_secret_key}'
);
```

### Cross-Account Access

```sql
CREATE CONNECTION kinesis_cross_account TO kinesis (
    stream_name = 'my-stream',
    stream_arn = 'arn:aws:kinesis:us-east-1:123456789:stream/my-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    role_arn = 'arn:aws:iam::123456789:role/KinesisReadRole',
    external_id = 'unique-external-id'
);
```

## Starting Positions

### Latest Records

```sql
-- Start from most recent records
CREATE CONNECTION kinesis_latest TO kinesis (
    stream_name = 'events',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    starting_position = 'LATEST'
);
```

### From Beginning

```sql
-- Start from oldest available records
CREATE CONNECTION kinesis_beginning TO kinesis (
    stream_name = 'events',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    starting_position = 'TRIM_HORIZON'
);
```

### From Specific Timestamp

```sql
-- Start from specific point in time
CREATE CONNECTION kinesis_timestamp TO kinesis (
    stream_name = 'events',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    starting_position = 'AT_TIMESTAMP',
    starting_timestamp = '2024-01-01T00:00:00Z'
);
```

## Enhanced Fan-Out

### Standard Consumer

```sql
-- Standard GetRecords API (shared throughput)
CREATE CONNECTION kinesis_standard TO kinesis (
    stream_name = 'high-volume-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    use_enhanced_fanout = 'false'
);
```

### Enhanced Fan-Out Consumer

```sql
-- Dedicated throughput per consumer
CREATE CONNECTION kinesis_enhanced TO kinesis (
    stream_name = 'high-volume-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    use_enhanced_fanout = 'true',
    consumer_name = 'laminar-enhanced-consumer',
    consumer_arn = 'arn:aws:kinesis:us-east-1:123456789:stream/my-stream/consumer/my-consumer:1234567890'
);
```

## Data Formats

### JSON Format

```sql
CREATE CONNECTION kinesis_json TO kinesis (
    stream_name = 'json-events',
    region = 'us-east-1',
    type = 'source',
    format = 'json'
);

-- Usage
CREATE TABLE events (
    event_id TEXT,
    user_id BIGINT,
    event_type TEXT,
    timestamp TIMESTAMP,
    properties JSON
) WITH (
    connector = 'kinesis_json'
);
```

### Raw Binary Format

```sql
CREATE CONNECTION kinesis_raw TO kinesis (
    stream_name = 'binary-data',
    region = 'us-east-1',
    type = 'source',
    format = 'raw'
);

-- Access raw data and metadata
CREATE TABLE raw_records (
    data BYTEA,
    partition_key TEXT,
    sequence_number TEXT,
    approximate_arrival_timestamp TIMESTAMP,
    shard_id TEXT
) WITH (
    connector = 'kinesis_raw',
    include_metadata = 'true'
);
```

### Base64 Encoded Data

```sql
CREATE CONNECTION kinesis_base64 TO kinesis (
    stream_name = 'encoded-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'raw',
    base64_decode = 'true'
);
```

## Metadata Access

```sql
-- Table with Kinesis metadata
CREATE TABLE kinesis_events (
    -- Data fields
    id BIGINT,
    message TEXT,
    
    -- Kinesis metadata
    _kinesis_shard_id TEXT,
    _kinesis_sequence_number TEXT,
    _kinesis_partition_key TEXT,
    _kinesis_approximate_arrival TIMESTAMP,
    _kinesis_encryption_type TEXT
) WITH (
    connector = 'kinesis_source',
    include_metadata = 'true'
);

-- Query with metadata
SELECT 
    id,
    message,
    _kinesis_shard_id,
    _kinesis_sequence_number,
    LAG(_kinesis_sequence_number) OVER (
        PARTITION BY _kinesis_shard_id 
        ORDER BY _kinesis_sequence_number
    ) as prev_sequence
FROM kinesis_events;
```

## Shard Management

### Automatic Shard Discovery

```sql
-- Automatically discovers and reads from all shards
CREATE CONNECTION kinesis_auto_shard TO kinesis (
    stream_name = 'auto-scaling-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    shard_discovery_interval = '60s'
);
```

### Specific Shard Reading

```sql
-- Read from specific shards only
CREATE CONNECTION kinesis_specific_shards TO kinesis (
    stream_name = 'multi-shard-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    shard_ids = 'shardId-000000000000,shardId-000000000001'
);
```

## Performance Optimization

### Batch Processing

```sql
CREATE CONNECTION kinesis_batch TO kinesis (
    stream_name = 'high-throughput',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    max_records_per_fetch = 10000,
    fetch_interval_ms = 500,
    max_batch_size = 25000,
    batch_timeout_ms = 1000
);
```

### Parallel Processing

```sql
-- Configure parallel shard readers
CREATE CONNECTION kinesis_parallel TO kinesis (
    stream_name = 'parallel-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    parallelism = 10,  -- Number of parallel readers
    max_outstanding_checkpoints = 3
);
```

## Error Handling

### Retry Configuration

```sql
CREATE CONNECTION kinesis_retry TO kinesis (
    stream_name = 'critical-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    max_retries = 10,
    retry_backoff_ms = 1000,
    retry_max_backoff_ms = 60000,
    skip_corrupted_records = 'false'
);
```

### Dead Letter Stream

```sql
-- Main stream connection
CREATE CONNECTION kinesis_main TO kinesis (
    stream_name = 'main-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    on_error = 'continue',
    error_stream = 'error-stream'
);

-- Pipeline with error handling
CREATE PIPELINE process_with_errors AS
WITH validated AS (
    SELECT 
        *,
        CASE 
            WHEN is_valid_json(data) THEN 'valid'
            ELSE 'invalid'
        END as status
    FROM kinesis_events
)
SELECT * FROM validated WHERE status = 'valid';
```

## Monitoring

### Stream Metrics

```sql
-- Monitor consumption rate
SELECT 
    shard_id,
    COUNT(*) as records_processed,
    MIN(approximate_arrival_timestamp) as oldest_record,
    MAX(approximate_arrival_timestamp) as newest_record,
    EXTRACT(EPOCH FROM (MAX(approximate_arrival_timestamp) - MIN(approximate_arrival_timestamp))) as lag_seconds
FROM kinesis_events
WHERE approximate_arrival_timestamp > NOW() - INTERVAL '5 minutes'
GROUP BY shard_id;
```

### Checkpoint Status

```sql
-- View checkpoint progress
SELECT 
    stream_name,
    shard_id,
    last_sequence_number,
    checkpoint_timestamp,
    records_behind_latest
FROM system.kinesis_checkpoints
WHERE stream_name = 'my-stream';
```

## Multi-Region Setup

### Cross-Region Replication

```sql
-- Primary region consumer
CREATE CONNECTION kinesis_primary TO kinesis (
    stream_name = 'global-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json'
);

-- Backup region consumer
CREATE CONNECTION kinesis_backup TO kinesis (
    stream_name = 'global-stream-replica',
    region = 'eu-west-1',
    type = 'source',
    format = 'json'
);

-- Unified view
CREATE VIEW global_events AS
SELECT *, 'us-east-1' as source_region FROM kinesis_primary
UNION ALL
SELECT *, 'eu-west-1' as source_region FROM kinesis_backup;
```

## Cost Optimization

### On-Demand vs Provisioned

```sql
-- Configure for on-demand streams
CREATE CONNECTION kinesis_on_demand TO kinesis (
    stream_name = 'on-demand-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    -- Optimize for on-demand pricing
    max_records_per_fetch = 10000,
    fetch_interval_ms = 1000
);

-- Configure for provisioned streams
CREATE CONNECTION kinesis_provisioned TO kinesis (
    stream_name = 'provisioned-stream',
    region = 'us-east-1',
    type = 'source',
    format = 'json',
    -- Optimize for provisioned throughput
    max_records_per_fetch = 1000,
    fetch_interval_ms = 200
);
```

## Best Practices

1. **Use enhanced fan-out** for latency-sensitive applications
2. **Configure appropriate batch sizes** based on record size
3. **Monitor shard-level metrics** for balanced consumption
4. **Use IAM roles** instead of access keys
5. **Set up CloudWatch alarms** for lag monitoring
6. **Handle resharding events** gracefully
7. **Implement proper error handling** and retries
8. **Use checkpointing** for exactly-once processing
9. **Consider costs** when choosing polling intervals
10. **Test failover scenarios** for multi-region setups

## Troubleshooting

### Connection Issues

```sql
-- Test connection
SELECT test_connection('kinesis_source');

-- Check stream status
SELECT * FROM system.kinesis_streams
WHERE connection = 'kinesis_source';

-- Verify IAM permissions
SELECT * FROM system.kinesis_permissions
WHERE stream_arn = 'arn:aws:kinesis:us-east-1:123456789:stream/my-stream';
```

### Performance Issues

```bash
# Check shard iterator age
aws kinesis describe-stream-summary \
    --stream-name my-stream \
    --region us-east-1

# Monitor GetRecords latency
aws cloudwatch get-metric-statistics \
    --namespace AWS/Kinesis \
    --metric-name GetRecords.Latency \
    --dimensions Name=StreamName,Value=my-stream \
    --start-time 2024-01-01T00:00:00Z \
    --end-time 2024-01-01T01:00:00Z \
    --period 300 \
    --statistics Average
```