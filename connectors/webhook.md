---
sidebar_position: 3
title: Webhook
---

The Webhook connector enables Laminar to send HTTP POST requests to external endpoints, useful for notifications and integrations.

## Capabilities

- **Source**: No
- **Sink**: Yes
- **Formats**: JSON, Raw
- **Methods**: POST
- **Authentication**: Via headers

## Configuration

### Table Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | - | The endpoint URL to receive webhooks (supports env vars) |
| `headers` | string | No | - | Comma-separated list of headers (supports env vars) |

Note: Webhook connector does not require a connection profile.

## API Usage

### Create Sink Table

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "alert_webhook",
    "connector": "webhook",
    "config": {
      "endpoint": "https://api.example.com/webhooks/alerts",
      "headers": "Authorization: Bearer {{ WEBHOOK_TOKEN }},X-Source: Laminar"
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

## SQL Usage

### Simple Webhook Sink

```sql
CREATE TABLE webhook_notifications (
    event_id BIGINT,
    event_type TEXT,
    message TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'webhook',
    endpoint = 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL',
    format = 'json',
    type = 'sink'
);
```

### With Authentication Headers

```sql
CREATE TABLE api_events (
    id BIGINT,
    data JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'webhook',
    endpoint = 'https://api.service.com/events',
    headers = 'Authorization: Bearer {{ API_TOKEN }},Content-Type: application/json,X-Client-ID: laminar',
    format = 'json',
    type = 'sink'
);
```

## Examples

### Slack Notifications

```sql
-- Create Slack webhook sink
CREATE TABLE slack_alerts (
    text TEXT,
    channel TEXT,
    username TEXT,
    icon_emoji TEXT
) WITH (
    connector = 'webhook',
    endpoint = '{{ SLACK_WEBHOOK_URL }}',
    format = 'json',
    type = 'sink'
);

-- Send alerts from processing
INSERT INTO slack_alerts
SELECT 
    CONCAT('🚨 Alert: ', error_message) as text,
    '#alerts' as channel,
    'Laminar Bot' as username,
    ':warning:' as icon_emoji
FROM error_events
WHERE severity = 'CRITICAL';
```

### External API Integration

```sql
-- Sink for external API
CREATE TABLE customer_api (
    customer_id TEXT,
    action TEXT,
    payload JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'webhook',
    endpoint = 'https://crm.example.com/api/v2/events',
    headers = 'Authorization: Bearer {{ CRM_API_KEY }},X-Source: Laminar,X-Version: 1.0',
    format = 'json',
    type = 'sink'
);

-- Send customer events
INSERT INTO customer_api
SELECT 
    user_id as customer_id,
    'purchase' as action,
    JSON_OBJECT(
        'amount', total_amount,
        'items', item_count,
        'category', product_category
    ) as payload,
    NOW() as timestamp
FROM completed_orders;
```

### Batch Notifications

```sql
-- Aggregate and send batch notifications
CREATE TABLE daily_summary_webhook (
    date TEXT,
    total_events BIGINT,
    unique_users BIGINT,
    top_events ARRAY<TEXT>,
    metrics JSON
) WITH (
    connector = 'webhook',
    endpoint = 'https://analytics.internal/daily-summary',
    headers = 'X-Report-Type: daily,X-Team: data-platform',
    format = 'json',
    type = 'sink'
);

-- Send daily summaries
INSERT INTO daily_summary_webhook
SELECT 
    CAST(window.start AS TEXT) as date,
    COUNT(*) as total_events,
    COUNT(DISTINCT user_id) as unique_users,
    ARRAY_AGG(DISTINCT event_type) as top_events,
    JSON_OBJECT(
        'avg_session_duration', AVG(session_duration),
        'total_revenue', SUM(revenue)
    ) as metrics
FROM (
    SELECT *, TUMBLE(INTERVAL '1 DAY') as window
    FROM events
    GROUP BY window
);
```

## Headers Format

Headers should be provided as comma-separated key:value pairs:
```
Header1: Value1,Header2: Value2,Header3: Value3
```

Example:
```
Authorization: Bearer token123,Content-Type: application/json,X-Request-ID: abc-123
```

## Best Practices

1. **Endpoint Validation**: Test webhook endpoints before deployment
2. **Authentication**: Store tokens in environment variables
3. **Retry Logic**: Webhook sink automatically retries on failure
4. **Payload Size**: Keep payloads under typical API limits (1-10MB)
5. **Headers**: Include identifying headers for debugging
6. **Rate Limiting**: Be aware of target API rate limits

## Limitations

- Only supports POST requests
- No support for custom HTTP methods
- Response is not captured (fire-and-forget)
- No built-in request signing (use headers for auth)