---
sidebar_position: 7
---

# MQTT Source

The MQTT source connector allows Laminar to consume messages from MQTT brokers, enabling IoT data ingestion and real-time telemetry processing.

## Overview

MQTT (Message Queuing Telemetry Transport) is a lightweight messaging protocol designed for IoT devices and low-bandwidth environments. The MQTT source subscribes to one or more topics and ingests messages as events.

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `broker` | string | Yes | MQTT broker URL (e.g., `tcp://localhost:1883`) |
| `client_id` | string | No | Unique client identifier (auto-generated if not provided) |
| `username` | string | No | Username for authentication |
| `password` | string | No | Password for authentication |
| `clean_session` | boolean | No | Start with clean session (default: true) |
| `keep_alive` | integer | No | Keep-alive interval in seconds (default: 60) |

### Subscription Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `topics` | array | Yes | List of topics to subscribe to |
| `qos` | integer | No | Quality of Service level (0, 1, or 2, default: 1) |

## SQL Example

```sql
CREATE CONNECTION mqtt_connection FROM mqtt_source
WITH (
    broker = 'tcp://mqtt.broker.com:1883',
    username = 'iot_reader',
    password = '${MQTT_PASSWORD}',
    clean_session = true
);

CREATE TABLE iot_telemetry (
    device_id TEXT,
    timestamp TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT,
    location TEXT
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["sensors/+/telemetry", "devices/+/status"]',
    qos = 1,
    format = 'json'
);
```

## Message Formats

### JSON Format

```sql
CREATE TABLE mqtt_json (
    payload JSON
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["data/json"]',
    format = 'json'
);
```

### Raw Bytes

```sql
CREATE TABLE mqtt_raw (
    topic TEXT,
    payload BYTEA,
    timestamp TIMESTAMP
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["data/binary"]',
    format = 'raw_bytes',
    'raw_bytes.include_topic' = 'true'
);
```

### Avro Format

```sql
CREATE TABLE mqtt_avro (
    schema_registry_url = 'http://schema-registry:8081',
    value_schema_id = 100
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["data/avro"]',
    format = 'avro'
);
```

## Topic Patterns

MQTT supports wildcards for flexible topic subscription:

- `+` - Single-level wildcard (matches one level)
- `#` - Multi-level wildcard (matches all remaining levels)

### Examples

```sql
-- Subscribe to all sensor telemetry
topics = '["sensors/+/telemetry"]'

-- Subscribe to all topics under devices
topics = '["devices/#"]'

-- Multiple patterns
topics = '["sensors/+/temp", "devices/+/status", "alerts/#"]'
```

## Advanced Configuration

### TLS/SSL

```sql
CREATE CONNECTION secure_mqtt FROM mqtt_source
WITH (
    broker = 'ssl://mqtt.broker.com:8883',
    tls_ca_cert = '/path/to/ca.crt',
    tls_client_cert = '/path/to/client.crt',
    tls_client_key = '/path/to/client.key',
    tls_verify = true
);
```

### Last Will and Testament

```sql
CREATE CONNECTION mqtt_with_lwt FROM mqtt_source
WITH (
    broker = 'tcp://localhost:1883',
    lwt_topic = 'clients/disconnected',
    lwt_payload = '{"client": "laminar-source", "status": "offline"}',
    lwt_qos = 1,
    lwt_retain = true
);
```

### Persistent Sessions

```sql
CREATE CONNECTION persistent_mqtt FROM mqtt_source
WITH (
    broker = 'tcp://localhost:1883',
    client_id = 'laminar-persistent-001',
    clean_session = false,
    session_expiry_interval = 3600
);
```

## Performance Tuning

### Batch Processing

```sql
CREATE TABLE mqtt_batched (
    ...
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["high-volume/#"]',
    batch_size = 1000,
    batch_timeout_ms = 100
);
```

### Parallel Processing

```sql
CREATE TABLE mqtt_parallel (
    ...
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["data/#"]',
    parallelism = 4
);
```

## Monitoring

### Metrics

- `laminar_mqtt_messages_received` - Total messages received
- `laminar_mqtt_bytes_received` - Total bytes received
- `laminar_mqtt_connection_status` - Connection status (0=disconnected, 1=connected)
- `laminar_mqtt_subscription_count` - Number of active subscriptions
- `laminar_mqtt_reconnect_attempts` - Number of reconnection attempts

### Health Checks

```sql
SELECT * FROM system.connector_status
WHERE connector_name = 'mqtt_source';
```

## Error Handling

### Reconnection Strategy

```sql
CREATE CONNECTION mqtt_resilient FROM mqtt_source
WITH (
    broker = 'tcp://localhost:1883',
    reconnect_interval = 5,
    max_reconnect_attempts = 10,
    reconnect_backoff_multiplier = 2.0
);
```

### Dead Letter Queue

```sql
CREATE TABLE mqtt_with_dlq (
    ...
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["data/#"]',
    dead_letter_topic = 'errors/mqtt',
    max_retries = 3
);
```

## Best Practices

1. **Client ID Management**: Use unique, descriptive client IDs for better monitoring
2. **QoS Selection**: 
   - QoS 0 for high-volume, loss-tolerant data
   - QoS 1 for important messages
   - QoS 2 for critical, exactly-once delivery
3. **Topic Design**: Use hierarchical topics for better organization
4. **Connection Pooling**: Use multiple connections for high-throughput scenarios
5. **Clean Sessions**: Use persistent sessions for critical data streams

## Troubleshooting

### Connection Issues

```sql
-- Check connection status
SELECT * FROM system.connector_logs
WHERE connector = 'mqtt_source'
AND level = 'ERROR'
ORDER BY timestamp DESC
LIMIT 10;
```

### Message Processing

```sql
-- Monitor message rates
SELECT 
    COUNT(*) as messages_per_minute,
    MIN(timestamp) as window_start,
    MAX(timestamp) as window_end
FROM iot_telemetry
WHERE timestamp > NOW() - INTERVAL '1 minute';
```

## Example Use Cases

### IoT Sensor Monitoring

```sql
CREATE TABLE sensor_data (
    sensor_id TEXT,
    reading_time TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT,
    pressure FLOAT
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["sensors/+/readings"]',
    format = 'json'
);

-- Alert on anomalies
CREATE VIEW temperature_alerts AS
SELECT 
    sensor_id,
    temperature,
    reading_time
FROM sensor_data
WHERE temperature > 40.0 OR temperature < -10.0;
```

### Fleet Tracking

```sql
CREATE TABLE vehicle_telemetry (
    vehicle_id TEXT,
    timestamp TIMESTAMP,
    latitude FLOAT,
    longitude FLOAT,
    speed FLOAT,
    fuel_level FLOAT
) WITH (
    connector = 'mqtt_source',
    connection = 'mqtt_connection',
    topics = '["vehicles/+/telemetry"]',
    format = 'json'
);
```