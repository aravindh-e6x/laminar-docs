---
sidebar_position: 9
title: Kinesis
---

The Kinesis connector enables Laminar to produce and consume messages from Amazon Kinesis Data Streams.

## Capabilities

- **Source**: Yes
- **Sink**: Yes
- **Formats**: JSON, Avro, Parquet, Raw
- **Authentication**: AWS IAM (uses AWS SDK credential chain)
- **Batching**: Automatic batching for sink operations

## Configuration

### Authentication

Kinesis uses the standard AWS SDK credential chain:
1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. AWS credentials file (`~/.aws/credentials`)
3. IAM role (when running on EC2/ECS/Lambda)
4. Instance metadata service

### Table Configuration

#### Source Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `streamName` | string | Yes | - | Kinesis stream name |
| `awsRegion` | string | No | - | AWS region (overrides default) |
| `type.offset` | enum | Yes | - | Starting position: `latest`, `earliest` |

#### Sink Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `streamName` | string | Yes | - | Kinesis stream name |
| `awsRegion` | string | No | - | AWS region (overrides default) |
| `type.recordsPerBatch` | integer | No | 500 | Records per batch (max 500) |
| `type.batchMaxBufferSize` | integer | No | 4000000 | Max batch size in bytes (max 4MB) |
| `type.batchFlushIntervalMillis` | integer | No | 1000 | Flush interval in milliseconds |

## API Usage

### Create Source Table

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "events_stream",
    "connector": "kinesis",
    "config": {
      "streamName": "production-events",
      "awsRegion": "us-east-1",
      "type": {
        "offset": "latest"
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
    "name": "processed_events",
    "connector": "kinesis",
    "config": {
      "streamName": "processed-events",
      "awsRegion": "us-west-2",
      "type": {
        "recordsPerBatch": 250,
        "batchFlushIntervalMillis": 500
      }
    }
  }'
```

## SQL Usage

### Reading from Kinesis

```sql
CREATE TABLE events (
    event_id TEXT,
    event_type TEXT,
    user_id TEXT,
    properties JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'production-events',
    aws_region = 'us-east-1',
    type = 'source',
    'source.offset' = 'latest',
    format = 'json'
);
```

### Writing to Kinesis

```sql
CREATE TABLE processed_events (
    event_id TEXT,
    event_type TEXT,
    metrics JSON,
    processed_at TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'processed-events',
    aws_region = 'us-east-1',
    type = 'sink',
    'sink.records_per_batch' = '250',
    'sink.batch_flush_interval_millis' = '500',
    format = 'json'
);
```

## Examples

### Real-time Analytics Pipeline

```sql
-- Source: Raw clickstream data
CREATE TABLE clickstream (
    session_id TEXT,
    user_id TEXT,
    page_url TEXT,
    referrer TEXT,
    user_agent TEXT,
    ip_address TEXT,
    event_time TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'clickstream-raw',
    aws_region = 'us-east-1',
    type = 'source',
    'source.offset' = 'latest',
    format = 'json'
);

-- Sink: Aggregated metrics
CREATE TABLE page_metrics (
    page_url TEXT,
    unique_users BIGINT,
    total_views BIGINT,
    avg_session_duration_sec DOUBLE,
    window_start TIMESTAMP,
    window_end TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'page-metrics',
    aws_region = 'us-east-1',
    type = 'sink',
    'sink.records_per_batch' = '500',
    format = 'json'
);

-- Calculate 5-minute metrics
INSERT INTO page_metrics
SELECT 
    page_url,
    COUNT(DISTINCT user_id) as unique_users,
    COUNT(*) as total_views,
    AVG(session_duration) as avg_session_duration_sec,
    TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start,
    TUMBLE_END(event_time, INTERVAL '5' MINUTE) as window_end
FROM (
    SELECT 
        page_url,
        user_id,
        session_id,
        EXTRACT(EPOCH FROM (MAX(event_time) - MIN(event_time))) as session_duration,
        event_time
    FROM clickstream
    GROUP BY page_url, user_id, session_id, event_time
)
GROUP BY 
    page_url,
    TUMBLE(event_time, INTERVAL '5' MINUTE);
```

### Cross-Region Data Replication

```sql
-- Source from us-east-1
CREATE TABLE orders_east (
    order_id BIGINT,
    customer_id TEXT,
    items JSON,
    total_amount DECIMAL,
    order_time TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'orders',
    aws_region = 'us-east-1',
    type = 'source',
    'source.offset' = 'earliest',
    format = 'json'
);

-- Replicate to us-west-2
CREATE TABLE orders_west (
    order_id BIGINT,
    customer_id TEXT,
    items JSON,
    total_amount DECIMAL,
    order_time TIMESTAMP,
    replicated_at TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'orders-replica',
    aws_region = 'us-west-2',
    type = 'sink',
    format = 'json'
);

-- Replicate with added metadata
INSERT INTO orders_west
SELECT 
    order_id,
    customer_id,
    items,
    total_amount,
    order_time,
    NOW() as replicated_at
FROM orders_east;
```

### IoT Data Processing

```sql
-- IoT sensor data stream
CREATE TABLE sensor_data (
    device_id TEXT,
    location JSON,
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    timestamp TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'iot-sensors',
    aws_region = 'eu-west-1',
    type = 'source',
    'source.offset' = 'latest',
    format = 'json'
);

-- Anomaly alerts stream
CREATE TABLE anomaly_alerts (
    device_id TEXT,
    alert_type TEXT,
    severity TEXT,
    details JSON,
    detected_at TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'iot-alerts',
    aws_region = 'eu-west-1',
    type = 'sink',
    'sink.records_per_batch' = '100',
    'sink.batch_flush_interval_millis' = '100',
    format = 'json'
);

-- Detect anomalies
INSERT INTO anomaly_alerts
SELECT 
    device_id,
    'temperature_anomaly' as alert_type,
    CASE 
        WHEN temperature > 40 THEN 'critical'
        WHEN temperature > 35 THEN 'warning'
        ELSE 'info'
    END as severity,
    JSON_OBJECT(
        'temperature', temperature,
        'location', location,
        'threshold', 35
    ) as details,
    timestamp as detected_at
FROM sensor_data
WHERE temperature > 30;
```

### Transaction Monitoring

```sql
-- Financial transactions
CREATE TABLE transactions (
    transaction_id TEXT,
    account_id TEXT,
    merchant_id TEXT,
    amount DECIMAL,
    currency TEXT,
    transaction_time TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'financial-transactions',
    type = 'source',
    'source.offset' = 'latest',
    format = 'json'
);

-- Fraud detection results
CREATE TABLE fraud_checks (
    transaction_id TEXT,
    risk_score DOUBLE,
    risk_factors JSON,
    is_suspicious BOOLEAN,
    checked_at TIMESTAMP
) WITH (
    connector = 'kinesis',
    stream_name = 'fraud-detection',
    type = 'sink',
    'sink.records_per_batch' = '200',
    format = 'json'
);

-- Detect suspicious patterns
INSERT INTO fraud_checks
SELECT 
    transaction_id,
    CASE 
        WHEN amount > 10000 THEN 0.8
        WHEN hour_tx_count > 10 THEN 0.7
        WHEN amount > avg_amount * 3 THEN 0.6
        ELSE 0.1
    END as risk_score,
    JSON_OBJECT(
        'large_amount', amount > 10000,
        'high_frequency', hour_tx_count > 10,
        'amount_anomaly', amount > avg_amount * 3
    ) as risk_factors,
    amount > 10000 OR hour_tx_count > 10 as is_suspicious,
    NOW() as checked_at
FROM (
    SELECT 
        t.transaction_id,
        t.amount,
        COUNT(*) OVER (
            PARTITION BY t.account_id 
            ORDER BY t.transaction_time 
            RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
        ) as hour_tx_count,
        AVG(amount) OVER (
            PARTITION BY t.account_id
            ORDER BY t.transaction_time
            ROWS BETWEEN 100 PRECEDING AND CURRENT ROW
        ) as avg_amount,
        t.transaction_time
    FROM transactions t
);
```

## Batching Behavior

### Sink Batching

The Kinesis sink automatically batches records for efficiency:
- Records are batched until `recordsPerBatch` is reached (max 500)
- Or until `batchMaxBufferSize` bytes is reached (max 4MB)
- Or until `batchFlushIntervalMillis` milliseconds have passed
- Whichever condition is met first triggers a flush

## Best Practices

1. **Shard Management**: Ensure adequate shard count for expected throughput
2. **Partition Keys**: Kinesis automatically handles partition key distribution
3. **Error Handling**: Failed records are automatically retried with exponential backoff
4. **Region Selection**: Place streams close to data sources/consumers
5. **Monitoring**: Use CloudWatch metrics to monitor stream health
6. **Cost Optimization**: Use appropriate retention period and shard count

## Limitations

- Maximum record size: 1MB
- Maximum batch size: 500 records or 4MB
- No support for Kinesis Analytics or Kinesis Firehose
- Partition key is automatically generated (not configurable)
- No support for enhanced fan-out consumers