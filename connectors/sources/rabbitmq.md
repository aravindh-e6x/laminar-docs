---
sidebar_position: 8
---

# RabbitMQ Source

The RabbitMQ source connector enables Laminar to consume messages from RabbitMQ queues, supporting various exchange types and routing patterns.

## Overview

RabbitMQ is a robust message broker that implements AMQP (Advanced Message Queuing Protocol). The RabbitMQ source can consume from queues with automatic acknowledgment and error handling.

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `host` | string | Yes | RabbitMQ host |
| `port` | integer | No | Port number (default: 5672) |
| `username` | string | No | Username (default: guest) |
| `password` | string | No | Password (default: guest) |
| `vhost` | string | No | Virtual host (default: /) |
| `connection_name` | string | No | Connection name for monitoring |

### Queue Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `queue` | string | Yes | Queue name to consume from |
| `exchange` | string | No | Exchange to bind to |
| `routing_key` | string | No | Routing key for binding |
| `prefetch_count` | integer | No | Prefetch count (default: 100) |
| `auto_ack` | boolean | No | Auto-acknowledge messages (default: false) |

## SQL Example

```sql
CREATE CONNECTION rabbitmq_connection FROM rabbitmq_source
WITH (
    host = 'rabbitmq.example.com',
    port = 5672,
    username = 'app_user',
    password = '${RABBITMQ_PASSWORD}',
    vhost = '/production'
);

CREATE TABLE orders (
    order_id TEXT,
    customer_id TEXT,
    timestamp TIMESTAMP,
    amount DECIMAL,
    items JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'orders_queue',
    exchange = 'orders_exchange',
    routing_key = 'new.order',
    format = 'json'
);
```

## Exchange Types

### Direct Exchange

```sql
CREATE TABLE direct_messages (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'direct_queue',
    exchange = 'direct_exchange',
    exchange_type = 'direct',
    routing_key = 'specific.route'
);
```

### Topic Exchange

```sql
CREATE TABLE topic_messages (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'topic_queue',
    exchange = 'topic_exchange',
    exchange_type = 'topic',
    routing_key = 'logs.*.error'
);
```

### Fanout Exchange

```sql
CREATE TABLE fanout_messages (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'fanout_queue',
    exchange = 'fanout_exchange',
    exchange_type = 'fanout'
);
```

### Headers Exchange

```sql
CREATE TABLE headers_messages (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'headers_queue',
    exchange = 'headers_exchange',
    exchange_type = 'headers',
    headers_match = 'all',
    headers = '{"type": "alert", "severity": "high"}'
);
```

## Message Formats

### JSON Messages

```sql
CREATE TABLE json_messages (
    data JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'json_queue',
    format = 'json'
);
```

### Avro Messages

```sql
CREATE TABLE avro_messages (
    -- Schema fields
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'avro_queue',
    format = 'avro',
    schema_registry_url = 'http://schema-registry:8081'
);
```

### Protobuf Messages

```sql
CREATE TABLE protobuf_messages (
    -- Schema fields
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'protobuf_queue',
    format = 'protobuf',
    protobuf_descriptor_file = '/schemas/messages.desc',
    protobuf_message_type = 'com.example.Message'
);
```

## Advanced Configuration

### TLS/SSL Connection

```sql
CREATE CONNECTION secure_rabbitmq FROM rabbitmq_source
WITH (
    host = 'rabbitmq.example.com',
    port = 5671,
    username = 'app_user',
    password = '${RABBITMQ_PASSWORD}',
    tls_enabled = true,
    tls_ca_cert = '/certs/ca.pem',
    tls_client_cert = '/certs/client.pem',
    tls_client_key = '/certs/client.key'
);
```

### Queue Declaration

```sql
CREATE TABLE auto_queue (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'auto_created_queue',
    queue_declare = true,
    queue_durable = true,
    queue_exclusive = false,
    queue_auto_delete = false,
    queue_arguments = '{"x-message-ttl": 3600000}'
);
```

### Dead Letter Queue

```sql
CREATE TABLE with_dlq (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'main_queue',
    dlq_exchange = 'dlq_exchange',
    dlq_routing_key = 'failed.messages',
    max_retries = 3
);
```

## Performance Tuning

### Prefetch Configuration

```sql
CREATE TABLE optimized_consumer (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'high_volume_queue',
    prefetch_count = 500,
    prefetch_size = 0,
    prefetch_global = false
);
```

### Parallel Consumers

```sql
CREATE TABLE parallel_processing (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'parallel_queue',
    consumer_count = 10,
    prefetch_count = 100
);
```

## Monitoring

### Metrics

- `laminar_rabbitmq_messages_consumed` - Total messages consumed
- `laminar_rabbitmq_messages_acked` - Messages acknowledged
- `laminar_rabbitmq_messages_rejected` - Messages rejected
- `laminar_rabbitmq_queue_depth` - Current queue depth
- `laminar_rabbitmq_consumer_count` - Active consumer count

### Health Checks

```sql
SELECT * FROM system.connector_status
WHERE connector_name = 'rabbitmq_source';
```

## Error Handling

### Retry Configuration

```sql
CREATE TABLE with_retry (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'retry_queue',
    retry_initial_interval = 1000,
    retry_max_interval = 60000,
    retry_multiplier = 2.0,
    max_retries = 5
);
```

### Error Queue

```sql
CREATE TABLE with_error_handling (
    payload JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'main_queue',
    error_queue = 'error_queue',
    error_exchange = 'error_exchange',
    include_error_details = true
);
```

## Best Practices

1. **Queue Design**: Use appropriate exchange types for routing patterns
2. **Prefetch Tuning**: Balance between throughput and memory usage
3. **Acknowledgment Strategy**: Use manual acks for critical data
4. **Connection Pooling**: Use connection pools for high concurrency
5. **Queue TTL**: Set appropriate message TTL to prevent queue buildup

## Troubleshooting

### Connection Issues

```sql
-- Check connection logs
SELECT * FROM system.connector_logs
WHERE connector = 'rabbitmq_source'
AND level IN ('ERROR', 'WARN')
ORDER BY timestamp DESC
LIMIT 20;
```

### Message Processing

```sql
-- Monitor consumption rate
SELECT 
    COUNT(*) as messages_per_minute,
    AVG(processing_time_ms) as avg_processing_time
FROM system.connector_metrics
WHERE connector = 'rabbitmq_source'
AND timestamp > NOW() - INTERVAL '1 minute';
```

## Example Use Cases

### Order Processing Pipeline

```sql
CREATE TABLE order_events (
    order_id TEXT,
    event_type TEXT,
    timestamp TIMESTAMP,
    data JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'order_events',
    exchange = 'orders',
    routing_key = 'order.*'
);

-- Process different order events
CREATE VIEW order_confirmations AS
SELECT * FROM order_events 
WHERE event_type = 'confirmed';

CREATE VIEW order_shipments AS
SELECT * FROM order_events 
WHERE event_type = 'shipped';
```

### Log Aggregation

```sql
CREATE TABLE application_logs (
    app_name TEXT,
    level TEXT,
    timestamp TIMESTAMP,
    message TEXT,
    context JSON
) WITH (
    connector = 'rabbitmq_source',
    connection = 'rabbitmq_connection',
    queue = 'logs',
    exchange = 'logs',
    routing_key = '#'
);
```