---
sidebar_position: 7
---

# NATS Sink

The NATS sink connector enables Laminar to publish messages to NATS subjects for distributed messaging and microservices communication.

## Overview

NATS is a lightweight, high-performance messaging system. The NATS sink supports publishing to subjects with various message patterns.

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `servers` | array | Yes | NATS server URLs |
| `username` | string | No | Username for authentication |
| `password` | string | No | Password for authentication |
| `token` | string | No | Authentication token |
| `credentials_file` | string | No | Path to credentials file |

### Publishing Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `subject` | string | Yes | NATS subject |
| `subject_field` | string | No | Field for dynamic subject |
| `headers` | object | No | Message headers |

## SQL Example

```sql
CREATE CONNECTION nats_sink_conn FROM nats_sink
WITH (
    servers = '["nats://localhost:4222"]',
    username = 'publisher',
    password = '${NATS_PASSWORD}'
);

INSERT INTO nats_output
SELECT 
    order_id,
    'orders.processed' as subject,
    order_data
FROM processed_orders
WITH (
    connector = 'nats_sink',
    connection = 'nats_sink_conn',
    subject_field = 'subject',
    format = 'json'
);
```

## Message Patterns

### Publish-Subscribe

```sql
INSERT INTO nats_pubsub
SELECT * FROM events
WITH (
    connector = 'nats_sink',
    connection = 'nats_sink_conn',
    subject = 'events.stream'
);
```

### Request-Reply

```sql
INSERT INTO nats_request
SELECT 
    request_id,
    'service.request' as subject,
    request_data,
    'service.response' as reply_to
FROM requests
WITH (
    connector = 'nats_sink',
    connection = 'nats_sink_conn',
    subject_field = 'subject',
    reply_to_field = 'reply_to'
);
```

## Best Practices

1. **Subject Naming**: Use dot-notation for hierarchical organization
2. **Message Size**: Keep messages small for better performance
3. **Connection Pooling**: Use connection pools for high throughput
4. **Error Handling**: Implement proper retry logic
5. **Monitoring**: Track publish rates and failures