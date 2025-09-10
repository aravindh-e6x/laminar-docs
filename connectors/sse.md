---
sidebar_position: 6
title: SSE (Server-Sent Events)
---

The SSE connector enables Laminar to consume Server-Sent Events from HTTP endpoints for real-time data streaming.

## Capabilities

- **Source**: Yes (receive events from SSE endpoints)
- **Sink**: No
- **Formats**: JSON, Raw text
- **Protocols**: HTTP, HTTPS
- **Authentication**: Via headers

## Configuration

### Table Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | - | SSE endpoint URL (e.g., `https://example.com:8080/sse`) |
| `headers` | string | No | - | Comma-separated list of headers (supports env vars) |
| `events` | string | No | - | Comma-separated list of event types to listen for |

Note: SSE connector does not require a connection profile.

## API Usage

### Create SSE Source

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "live_updates",
    "connector": "sse",
    "config": {
      "endpoint": "https://api.example.com/v1/events",
      "headers": "Authorization: Bearer {{ SSE_TOKEN }}",
      "events": "update,insert,delete"
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

## SQL Usage

### Basic SSE Source

```sql
CREATE TABLE sse_events (
    id BIGINT,
    event_type TEXT,
    data JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse',
    endpoint = 'https://stream.example.com/events',
    type = 'source',
    format = 'json'
);
```

### With Authentication and Event Filtering

```sql
CREATE TABLE filtered_events (
    event_id TEXT,
    category TEXT,
    payload JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse',
    endpoint = 'https://api.service.com/sse',
    headers = 'Authorization: Bearer {{ API_TOKEN }},Accept: text/event-stream',
    events = 'user_action,system_event,notification',
    type = 'source',
    format = 'json'
);
```

## Examples

### Real-time Analytics Dashboard

```sql
-- Connect to analytics SSE stream
CREATE TABLE analytics_events (
    metric_name TEXT,
    value DOUBLE,
    dimensions JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse',
    endpoint = 'https://analytics.app.com/realtime',
    headers = 'X-API-Key: {{ ANALYTICS_KEY }}',
    events = 'metric_update',
    type = 'source',
    format = 'json'
);

-- Calculate rolling metrics
CREATE VIEW rolling_metrics AS
SELECT 
    metric_name,
    AVG(value) as avg_value,
    MAX(value) as max_value,
    MIN(value) as min_value,
    COUNT(*) as data_points,
    HOP(INTERVAL '10 SECONDS', INTERVAL '1 MINUTE') as window
FROM analytics_events
GROUP BY metric_name, window;
```

### Stock Price Streaming

```sql
-- Connect to stock price SSE feed
CREATE TABLE stock_prices (
    symbol TEXT,
    price DECIMAL,
    volume BIGINT,
    bid DECIMAL,
    ask DECIMAL,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse',
    endpoint = 'https://market-data.broker.com/sse/prices',
    headers = 'Authorization: Bearer {{ BROKER_TOKEN }}',
    events = 'price_update,trade',
    type = 'source',
    format = 'json'
);

-- Detect price movements
CREATE VIEW price_alerts AS
SELECT 
    symbol,
    price as current_price,
    LAG(price, 1) OVER (PARTITION BY symbol ORDER BY timestamp) as prev_price,
    (price - LAG(price, 1) OVER (PARTITION BY symbol ORDER BY timestamp)) / 
        LAG(price, 1) OVER (PARTITION BY symbol ORDER BY timestamp) * 100 as pct_change,
    timestamp
FROM stock_prices
WHERE ABS(pct_change) > 5.0;
```

### GitHub Events Stream

```sql
-- Connect to GitHub events SSE
CREATE TABLE github_events (
    event_type TEXT,
    repo_name TEXT,
    actor TEXT,
    action TEXT,
    payload JSON,
    created_at TIMESTAMP
) WITH (
    connector = 'sse',
    endpoint = 'https://api.github.com/events',
    headers = 'Authorization: token {{ GITHUB_TOKEN }},Accept: text/event-stream',
    type = 'source',
    format = 'json'
);

-- Track repository activity
CREATE VIEW repo_activity AS
SELECT 
    repo_name,
    event_type,
    COUNT(*) as event_count,
    COUNT(DISTINCT actor) as unique_contributors,
    TUMBLE(INTERVAL '1 HOUR') as window
FROM github_events
GROUP BY repo_name, event_type, window;
```

### News Feed Aggregation

```sql
-- Connect to news SSE feed
CREATE TABLE news_stream (
    article_id TEXT,
    title TEXT,
    category TEXT,
    source TEXT,
    summary TEXT,
    url TEXT,
    published_at TIMESTAMP
) WITH (
    connector = 'sse',
    endpoint = 'https://news-api.com/v2/stream',
    headers = 'X-API-Key: {{ NEWS_API_KEY }}',
    events = 'breaking_news,update',
    type = 'source',
    format = 'json'
);

-- Trending topics analysis
CREATE VIEW trending_topics AS
SELECT 
    category,
    COUNT(*) as article_count,
    ARRAY_AGG(DISTINCT source) as sources,
    TUMBLE(INTERVAL '15 MINUTES') as window
FROM news_stream
GROUP BY category, window
ORDER BY window DESC, article_count DESC;
```

### System Monitoring

```sql
-- Connect to monitoring SSE endpoint
CREATE TABLE system_metrics (
    host TEXT,
    metric TEXT,
    value DOUBLE,
    tags JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'sse',
    endpoint = 'https://monitoring.internal/sse/metrics',
    headers = 'X-Auth-Token: {{ MONITORING_TOKEN }}',
    events = 'cpu,memory,disk,network',
    type = 'source',
    format = 'json'
);

-- Alert on anomalies
CREATE VIEW system_alerts AS
SELECT 
    host,
    metric,
    value,
    CASE 
        WHEN metric = 'cpu' AND value > 90 THEN 'CRITICAL'
        WHEN metric = 'memory' AND value > 85 THEN 'WARNING'
        WHEN metric = 'disk' AND value > 95 THEN 'CRITICAL'
        ELSE 'OK'
    END as severity,
    timestamp
FROM system_metrics
WHERE value > CASE metric
    WHEN 'cpu' THEN 80
    WHEN 'memory' THEN 75
    WHEN 'disk' THEN 90
    ELSE 100
END;
```

## Event Types

The `events` field allows filtering specific SSE event types:
- If not specified, all events are consumed
- Multiple event types can be specified as comma-separated values
- Event types are case-sensitive

Example SSE event stream:
```
event: user_action
data: {"user_id": "123", "action": "click"}

event: system_event
data: {"level": "info", "message": "Service started"}

event: notification
data: {"recipient": "user@example.com", "type": "alert"}
```

## Headers Format

Headers should be comma-separated key:value pairs:
```
Authorization: Bearer token123,Accept: text/event-stream,X-Client-ID: laminar
```

## Best Practices

1. **Use HTTPS**: Always use secure connections in production
2. **Authentication**: Store tokens in environment variables
3. **Event Filtering**: Specify event types to reduce unnecessary processing
4. **Reconnection**: SSE source automatically reconnects on connection loss
5. **Headers**: Include `Accept: text/event-stream` for proper content negotiation
6. **Timeout Handling**: SSE maintains persistent connections with automatic keep-alive

## Limitations

- Source only (cannot send events back to server)
- Unidirectional communication (server to client only)
- Text-based protocol (no binary data)
- No custom retry logic configuration