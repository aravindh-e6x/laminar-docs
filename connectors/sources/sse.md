---
sidebar_position: 10
---

# SSE Source

The SSE (Server-Sent Events) source connector enables Laminar to consume real-time event streams from SSE endpoints, commonly used for live data feeds and notifications.

## Overview

Server-Sent Events provide a standard way to push real-time updates from servers to clients over HTTP. The SSE source maintains a persistent connection and processes events as they arrive.

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `endpoint` | string | Yes | SSE endpoint URL |
| `headers` | object | No | HTTP headers for the request |
| `reconnect_interval_ms` | integer | No | Reconnection interval (default: 3000) |
| `timeout_ms` | integer | No | Connection timeout (default: 60000) |

### Authentication

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `auth_type` | string | No | Authentication type (none, bearer, api_key) |
| `token` | string | No | Bearer token |
| `api_key` | string | No | API key |
| `api_key_header` | string | No | API key header name |

## SQL Example

```sql
CREATE CONNECTION sse_stream FROM sse_source
WITH (
    endpoint = 'https://stream.example.com/events',
    headers = '{"Accept": "text/event-stream"}',
    reconnect_interval_ms = 5000
);

CREATE TABLE live_events (
    event_id TEXT,
    event_type TEXT,
    timestamp TIMESTAMP,
    data JSON
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    format = 'json'
);
```

## Event Types

### Named Events

```sql
CREATE TABLE typed_events (
    event_name TEXT,
    event_data JSON,
    event_id TEXT,
    retry_ms INTEGER
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    capture_event_name = true,
    capture_event_id = true,
    capture_retry = true
);
```

### Filtered Events

```sql
CREATE TABLE filtered_events (
    data JSON
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    event_type_filter = '["message", "update", "notification"]'
);
```

## Data Formats

### JSON Events

```sql
CREATE TABLE json_events (
    payload JSON
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    format = 'json'
);
```

### Plain Text Events

```sql
CREATE TABLE text_events (
    message TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    format = 'text',
    add_timestamp = true
);
```

### Custom Parsing

```sql
CREATE TABLE parsed_events (
    id TEXT,
    value FLOAT,
    metadata JSON
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    format = 'custom',
    parser_regex = '^id:(\S+) value:(\d+\.?\d*) meta:(.*)$'
);
```

## Advanced Configuration

### Authenticated Streams

```sql
CREATE CONNECTION auth_sse FROM sse_source
WITH (
    endpoint = 'https://api.example.com/stream',
    auth_type = 'bearer',
    token = '${SSE_TOKEN}',
    headers = '{"X-Client-Id": "laminar-consumer"}'
);
```

### Last Event ID

```sql
CREATE CONNECTION resumable_sse FROM sse_source
WITH (
    endpoint = 'https://stream.example.com/events',
    last_event_id = '${LAST_PROCESSED_ID}',
    persist_event_id = true
);
```

### Keep-Alive Handling

```sql
CREATE CONNECTION keepalive_sse FROM sse_source
WITH (
    endpoint = 'https://stream.example.com/events',
    keepalive_timeout_ms = 45000,
    ignore_comments = false,
    comment_prefix = ':'
);
```

## Error Handling

### Reconnection Strategy

```sql
CREATE CONNECTION resilient_sse FROM sse_source
WITH (
    endpoint = 'https://stream.example.com/events',
    max_reconnect_attempts = 10,
    reconnect_backoff_ms = 1000,
    reconnect_multiplier = 2.0,
    max_reconnect_interval_ms = 60000
);
```

### Error Events

```sql
CREATE TABLE sse_with_errors (
    data JSON,
    error_message TEXT,
    error_timestamp TIMESTAMP
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    capture_errors = true,
    continue_on_error = true
);
```

## Stream Processing

### Event Buffering

```sql
CREATE TABLE buffered_events (
    data JSON
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    buffer_size = 1000,
    buffer_timeout_ms = 100
);
```

### Event Deduplication

```sql
CREATE TABLE unique_events (
    event_id TEXT PRIMARY KEY,
    data JSON
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    deduplicate_by = 'event_id',
    dedup_window_ms = 60000
);
```

## Monitoring

### Metrics

- `laminar_sse_events_received` - Total events received
- `laminar_sse_bytes_received` - Total bytes received
- `laminar_sse_connection_status` - Connection status (0=disconnected, 1=connected)
- `laminar_sse_reconnection_attempts` - Number of reconnection attempts
- `laminar_sse_event_lag_ms` - Event processing lag

### Health Checks

```sql
SELECT * FROM system.connector_status
WHERE connector_name = 'sse_source';
```

## Performance Tuning

### Parallel Processing

```sql
CREATE TABLE parallel_sse (
    data JSON
) WITH (
    connector = 'sse_source',
    connection = 'sse_stream',
    parallelism = 4,
    partition_by = 'event_type'
);
```

### Compression

```sql
CREATE CONNECTION compressed_sse FROM sse_source
WITH (
    endpoint = 'https://stream.example.com/events',
    headers = '{"Accept-Encoding": "gzip, deflate"}',
    decompress_response = true
);
```

## Best Practices

1. **Connection Management**: Handle reconnections gracefully
2. **Event ID Tracking**: Use Last-Event-ID for resumable streams
3. **Error Handling**: Implement proper error recovery
4. **Buffering**: Use appropriate buffer sizes for throughput
5. **Monitoring**: Track connection status and event lag

## Troubleshooting

### Connection Issues

```sql
-- Check connection logs
SELECT * FROM system.connector_logs
WHERE connector = 'sse_source'
AND level = 'ERROR'
ORDER BY timestamp DESC
LIMIT 10;
```

### Event Processing

```sql
-- Monitor event rate
SELECT 
    COUNT(*) as events_per_minute,
    AVG(processing_time_ms) as avg_processing_time
FROM system.connector_metrics
WHERE connector = 'sse_source'
AND timestamp > NOW() - INTERVAL '1 minute';
```

## Example Use Cases

### Live Stock Prices

```sql
CREATE TABLE stock_stream (
    symbol TEXT,
    price DECIMAL,
    volume BIGINT,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse_source',
    connection = 'stock_sse',
    endpoint = 'https://stream.stocks.com/prices',
    format = 'json'
);

-- Real-time price alerts
CREATE VIEW price_alerts AS
SELECT * FROM stock_stream
WHERE price > 1000 OR price < 10;
```

### Social Media Feed

```sql
CREATE TABLE social_stream (
    post_id TEXT,
    user_id TEXT,
    content TEXT,
    timestamp TIMESTAMP,
    metrics JSON
) WITH (
    connector = 'sse_source',
    connection = 'social_sse',
    endpoint = 'https://api.social.com/stream',
    event_type_filter = '["post", "comment", "like"]'
);
```

### IoT Sensor Stream

```sql
CREATE TABLE sensor_stream (
    device_id TEXT,
    reading_type TEXT,
    value FLOAT,
    unit TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse_source',
    connection = 'iot_sse',
    endpoint = 'https://iot.example.com/sensors/stream'
);
```