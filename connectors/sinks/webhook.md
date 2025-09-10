---
sidebar_position: 8
---

# Webhook Sink

The Webhook sink connector enables Laminar to send HTTP/HTTPS requests to external endpoints, integrating with webhooks and REST APIs.

## Overview

The Webhook sink sends processed data as HTTP requests to configured endpoints, supporting various HTTP methods, authentication types, and retry strategies.

## Configuration

### Endpoint Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `endpoint` | string | Yes | Target URL |
| `method` | string | No | HTTP method (default: POST) |
| `headers` | object | No | HTTP headers |
| `timeout_ms` | integer | No | Request timeout (default: 30000) |

### Authentication

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `auth_type` | string | No | Authentication type |
| `username` | string | No | Basic auth username |
| `password` | string | No | Basic auth password |
| `token` | string | No | Bearer token |

## SQL Example

```sql
CREATE CONNECTION webhook_conn FROM webhook_sink
WITH (
    endpoint = 'https://api.example.com/webhook',
    method = 'POST',
    headers = '{"Content-Type": "application/json"}',
    auth_type = 'bearer',
    token = '${WEBHOOK_TOKEN}'
);

INSERT INTO webhook_notifications
SELECT 
    event_id,
    event_type,
    timestamp,
    payload
FROM important_events
WITH (
    connector = 'webhook_sink',
    connection = 'webhook_conn',
    format = 'json'
);
```

## Dynamic Endpoints

```sql
INSERT INTO dynamic_webhooks
SELECT 
    CONCAT('https://api.example.com/', service, '/webhook') as endpoint,
    data
FROM service_events
WITH (
    connector = 'webhook_sink',
    endpoint_field = 'endpoint',
    method = 'POST'
);
```

## Request Formats

### JSON Payload

```sql
INSERT INTO json_webhook
SELECT * FROM events
WITH (
    connector = 'webhook_sink',
    connection = 'webhook_conn',
    format = 'json',
    json_compact = false
);
```

### Form Data

```sql
INSERT INTO form_webhook
SELECT 
    'user_update' as action,
    user_id,
    email,
    updated_at
FROM user_changes
WITH (
    connector = 'webhook_sink',
    connection = 'webhook_conn',
    format = 'form',
    content_type = 'application/x-www-form-urlencoded'
);
```

## Retry Configuration

```sql
CREATE CONNECTION resilient_webhook FROM webhook_sink
WITH (
    endpoint = 'https://api.example.com/webhook',
    max_retries = 5,
    retry_backoff_ms = 1000,
    retry_multiplier = 2.0,
    retry_on_status = '[429, 500, 502, 503, 504]'
);
```

## Best Practices

1. **Retry Strategy**: Configure appropriate retry policies
2. **Timeout Settings**: Set reasonable timeouts
3. **Batch Requests**: Group multiple records when possible
4. **Authentication**: Use secure authentication methods
5. **Error Handling**: Implement dead letter queues for failures