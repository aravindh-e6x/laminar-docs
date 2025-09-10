---
sidebar_position: 8
title: RabbitMQ
---

The RabbitMQ connector enables Laminar to produce and consume messages from RabbitMQ Streams (not traditional queues).

## Capabilities

- **Source**: Yes
- **Sink**: Yes
- **Formats**: JSON, Avro, Parquet, Raw
- **Protocol**: RabbitMQ Streams API
- **Streaming**: Yes (uses RabbitMQ Streams, not traditional queues)

## Configuration

### Connection Profile Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `host` | string | No | localhost | RabbitMQ stream host |
| `username` | string | No | guest | Username for authentication |
| `password` | string | No | guest | Password for authentication |
| `virtualHost` | string | No | / | Virtual host to connect to |
| `port` | integer | No | 5552 | RabbitMQ stream port (not AMQP port) |
| `tlsConfig` | object | No | - | TLS configuration |
| `loadBalancerMode` | boolean | No | false | Enable load balancer mode |

#### TLS Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `enabled` | boolean | Yes | false | Enable TLS |
| `trustCertificates` | boolean | No | false | Trust certificates |
| `rootCertificatesPath` | string | No | - | Root certificates path |
| `clientCertificatesPath` | string | No | - | Client certificates path |
| `clientKeysPath` | string | No | - | Client keys path |

### Table Configuration

#### Source Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `stream` | string | Yes | - | RabbitMQ stream name |
| `type.offset` | enum | Yes | - | Starting offset: `first`, `last`, `next` |

#### Sink Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `stream` | string | Yes | - | RabbitMQ stream name |

## API Usage

### Create Connection Profile

```bash
curl -X POST http://localhost:8000/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "rabbitmq_streams",
    "connector": "rabbitmq",
    "config": {
      "host": "rabbitmq.example.com",
      "port": 5552,
      "username": "stream_user",
      "password": "{{ RABBITMQ_PASSWORD }}",
      "virtualHost": "/production"
    }
  }'
```

### Create Source Table

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "events_stream",
    "connector": "rabbitmq",
    "connectionProfileId": "prof_rabbitmq123",
    "config": {
      "stream": "events",
      "type": {
        "offset": "last"
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
    "connector": "rabbitmq",
    "connectionProfileId": "prof_rabbitmq123",
    "config": {
      "stream": "processed-events",
      "type": {}
    }
  }'
```

## SQL Usage

### Reading from RabbitMQ Stream

```sql
CREATE TABLE events (
    event_id BIGINT,
    event_type TEXT,
    payload JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'localhost',
    port = '5552',
    username = 'guest',
    password = 'guest',
    type = 'source',
    stream = 'events',
    'source.offset' = 'last',
    format = 'json'
);
```

### Writing to RabbitMQ Stream

```sql
CREATE TABLE alerts (
    alert_id BIGINT,
    severity TEXT,
    message TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'localhost',
    port = '5552',
    type = 'sink',
    stream = 'alerts',
    format = 'json'
);
```

## Examples

### Event Processing Pipeline

```sql
-- Source: Raw events from RabbitMQ
CREATE TABLE raw_events (
    event_id BIGINT,
    event_type TEXT,
    user_id TEXT,
    properties JSON,
    created_at TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'rabbitmq.internal',
    port = '5552',
    username = 'app',
    password = '{{ RABBITMQ_PASSWORD }}',
    type = 'source',
    stream = 'raw-events',
    'source.offset' = 'next',
    format = 'json'
);

-- Sink: Processed events back to RabbitMQ
CREATE TABLE processed_events (
    event_id BIGINT,
    event_type TEXT,
    user_segment TEXT,
    metrics JSON,
    processed_at TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'rabbitmq.internal',
    port = '5552',
    username = 'app',
    password = '{{ RABBITMQ_PASSWORD }}',
    type = 'sink',
    stream = 'processed-events',
    format = 'json'
);

-- Process events with enrichment
INSERT INTO processed_events
SELECT 
    event_id,
    event_type,
    CASE 
        WHEN JSON_VALUE(properties, '$.purchase_count') > 10 THEN 'high_value'
        WHEN JSON_VALUE(properties, '$.purchase_count') > 5 THEN 'medium_value'
        ELSE 'low_value'
    END as user_segment,
    JSON_OBJECT(
        'total_value', JSON_VALUE(properties, '$.total_value'),
        'item_count', JSON_VALUE(properties, '$.item_count')
    ) as metrics,
    NOW() as processed_at
FROM raw_events
WHERE event_type = 'purchase';
```

### IoT Telemetry with TLS

```sql
-- Secure connection for IoT data
CREATE TABLE iot_telemetry (
    device_id TEXT,
    sensor_type TEXT,
    value DOUBLE,
    unit TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'rabbitmq.iot.example.com',
    port = '5552',
    username = 'iot_reader',
    password = '{{ IOT_RABBITMQ_PASSWORD }}',
    'virtual_host' = '/iot',
    'tls.enabled' = 'true',
    'tls.trust_certificates' = 'true',
    type = 'source',
    stream = 'telemetry',
    'source.offset' = 'first',
    format = 'json'
);

-- Aggregate sensor readings
CREATE TABLE sensor_aggregates (
    device_id TEXT,
    sensor_type TEXT,
    avg_value DOUBLE,
    max_value DOUBLE,
    min_value DOUBLE,
    reading_count BIGINT,
    window_start TIMESTAMP,
    window_end TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'rabbitmq.iot.example.com',
    port = '5552',
    username = 'iot_writer',
    password = '{{ IOT_RABBITMQ_PASSWORD }}',
    'virtual_host' = '/iot',
    'tls.enabled' = 'true',
    type = 'sink',
    stream = 'aggregated-telemetry',
    format = 'json'
);

INSERT INTO sensor_aggregates
SELECT 
    device_id,
    sensor_type,
    AVG(value) as avg_value,
    MAX(value) as max_value,
    MIN(value) as min_value,
    COUNT(*) as reading_count,
    TUMBLE_START(timestamp, INTERVAL '5' MINUTE) as window_start,
    TUMBLE_END(timestamp, INTERVAL '5' MINUTE) as window_end
FROM iot_telemetry
GROUP BY 
    device_id,
    sensor_type,
    TUMBLE(timestamp, INTERVAL '5' MINUTE);
```

### Multi-Stream Join

```sql
-- Orders stream
CREATE TABLE orders (
    order_id BIGINT,
    customer_id TEXT,
    total_amount DECIMAL,
    order_time TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'localhost',
    type = 'source',
    stream = 'orders',
    'source.offset' = 'last',
    format = 'json'
);

-- Payments stream
CREATE TABLE payments (
    payment_id BIGINT,
    order_id BIGINT,
    amount DECIMAL,
    payment_time TIMESTAMP
) WITH (
    connector = 'rabbitmq',
    host = 'localhost',
    type = 'source',
    stream = 'payments',
    'source.offset' = 'last',
    format = 'json'
);

-- Matched transactions
CREATE TABLE matched_transactions (
    order_id BIGINT,
    customer_id TEXT,
    order_amount DECIMAL,
    payment_amount DECIMAL,
    payment_delay_minutes BIGINT
) WITH (
    connector = 'rabbitmq',
    host = 'localhost',
    type = 'sink',
    stream = 'matched-transactions',
    format = 'json'
);

-- Join orders with payments
INSERT INTO matched_transactions
SELECT 
    o.order_id,
    o.customer_id,
    o.total_amount as order_amount,
    p.amount as payment_amount,
    EXTRACT(EPOCH FROM (p.payment_time - o.order_time)) / 60 as payment_delay_minutes
FROM orders o
INNER JOIN payments p 
    ON o.order_id = p.order_id
    AND p.payment_time BETWEEN o.order_time AND o.order_time + INTERVAL '1' HOUR;
```

## Offset Behavior

### Source Offsets

- **`first`**: Start reading from the beginning of the stream
- **`last`**: Start reading from the end of the stream (only new messages)
- **`next`**: Resume from the last committed offset (for fault tolerance)

## Important Notes

1. **RabbitMQ Streams vs Queues**: This connector uses RabbitMQ Streams API (port 5552), not traditional AMQP queues (port 5672)
2. **Stream Creation**: Streams must be created in RabbitMQ before use
3. **Retention**: Streams have configurable retention policies in RabbitMQ
4. **Performance**: Streams are designed for high-throughput scenarios
5. **Ordering**: Messages maintain order within a stream
6. **Environment Variables**: Use `{{ VAR_NAME }}` syntax for sensitive values

## Limitations

- Only supports RabbitMQ Streams, not traditional queues/exchanges
- No support for RabbitMQ routing keys or exchanges
- Stream must exist before creating tables
- No automatic stream creation