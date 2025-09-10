---
sidebar_position: 9
---

# Polling HTTP Source

The Polling HTTP source connector periodically fetches data from HTTP/HTTPS endpoints, ideal for integrating with REST APIs and web services.

## Overview

The Polling HTTP source makes periodic HTTP requests to configured endpoints and ingests the responses as events. It supports various authentication methods, response formats, and pagination strategies.

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `endpoint` | string | Yes | HTTP/HTTPS endpoint URL |
| `method` | string | No | HTTP method (default: GET) |
| `interval_ms` | integer | Yes | Polling interval in milliseconds |
| `headers` | object | No | HTTP headers |
| `timeout_ms` | integer | No | Request timeout (default: 30000) |

### Authentication

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `auth_type` | string | No | Authentication type (none, basic, bearer, api_key) |
| `username` | string | No | Username for basic auth |
| `password` | string | No | Password for basic auth |
| `token` | string | No | Bearer token |
| `api_key` | string | No | API key |
| `api_key_header` | string | No | API key header name |

## SQL Example

```sql
CREATE CONNECTION http_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 60000,
    auth_type = 'bearer',
    token = '${API_TOKEN}',
    headers = '{"Accept": "application/json"}'
);

CREATE TABLE api_data (
    id TEXT,
    timestamp TIMESTAMP,
    value FLOAT,
    metadata JSON
) WITH (
    connector = 'polling_http_source',
    connection = 'http_api',
    format = 'json',
    json_path = '$.data[*]'
);
```

## Authentication Methods

### Basic Authentication

```sql
CREATE CONNECTION basic_auth_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 30000,
    auth_type = 'basic',
    username = 'user',
    password = '${API_PASSWORD}'
);
```

### Bearer Token

```sql
CREATE CONNECTION bearer_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 30000,
    auth_type = 'bearer',
    token = '${BEARER_TOKEN}'
);
```

### API Key

```sql
CREATE CONNECTION api_key_auth FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 30000,
    auth_type = 'api_key',
    api_key = '${API_KEY}',
    api_key_header = 'X-API-Key'
);
```

### OAuth 2.0

```sql
CREATE CONNECTION oauth_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 30000,
    auth_type = 'oauth2',
    client_id = '${CLIENT_ID}',
    client_secret = '${CLIENT_SECRET}',
    token_url = 'https://auth.example.com/token',
    scope = 'read:data'
);
```

## Response Formats

### JSON Response

```sql
CREATE TABLE json_api (
    data JSON
) WITH (
    connector = 'polling_http_source',
    connection = 'http_api',
    format = 'json'
);
```

### CSV Response

```sql
CREATE TABLE csv_api (
    id INTEGER,
    name TEXT,
    value FLOAT
) WITH (
    connector = 'polling_http_source',
    connection = 'http_api',
    format = 'csv',
    csv_header = true,
    csv_delimiter = ','
);
```

### XML Response

```sql
CREATE TABLE xml_api (
    data JSON
) WITH (
    connector = 'polling_http_source',
    connection = 'http_api',
    format = 'xml',
    xml_root_path = '/response/items/item'
);
```

## Pagination

### Offset-based Pagination

```sql
CREATE CONNECTION paginated_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 60000,
    pagination_type = 'offset',
    page_size = 100,
    offset_param = 'offset',
    limit_param = 'limit'
);
```

### Page-based Pagination

```sql
CREATE CONNECTION page_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 60000,
    pagination_type = 'page',
    page_size = 50,
    page_param = 'page',
    size_param = 'per_page'
);
```

### Cursor-based Pagination

```sql
CREATE CONNECTION cursor_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 60000,
    pagination_type = 'cursor',
    cursor_param = 'after',
    cursor_path = '$.meta.next_cursor'
);
```

### Link Header Pagination

```sql
CREATE CONNECTION link_header_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 60000,
    pagination_type = 'link_header',
    link_rel = 'next'
);
```

## Advanced Configuration

### Request Body

```sql
CREATE CONNECTION post_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/query',
    method = 'POST',
    interval_ms = 30000,
    body = '{"query": "SELECT * FROM data WHERE timestamp > ${last_poll_time}"}',
    headers = '{"Content-Type": "application/json"}'
);
```

### Dynamic Parameters

```sql
CREATE CONNECTION dynamic_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 60000,
    query_params = '{"from": "${last_poll_time}", "to": "${current_time}"}',
    time_format = 'ISO8601'
);
```

### Rate Limiting

```sql
CREATE CONNECTION rate_limited_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 1000,
    rate_limit_requests = 100,
    rate_limit_window_ms = 60000,
    retry_on_rate_limit = true,
    rate_limit_backoff_ms = 5000
);
```

## State Management

### Incremental Polling

```sql
CREATE TABLE incremental_data (
    id TEXT,
    updated_at TIMESTAMP,
    data JSON
) WITH (
    connector = 'polling_http_source',
    connection = 'http_api',
    state_type = 'timestamp',
    state_field = 'updated_at',
    state_param = 'since',
    initial_state = '2024-01-01T00:00:00Z'
);
```

### Checkpoint-based Polling

```sql
CREATE TABLE checkpoint_data (
    id TEXT,
    sequence BIGINT,
    data JSON
) WITH (
    connector = 'polling_http_source',
    connection = 'http_api',
    state_type = 'checkpoint',
    state_field = 'sequence',
    state_param = 'after_sequence'
);
```

## Error Handling

### Retry Configuration

```sql
CREATE CONNECTION resilient_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 60000,
    max_retries = 3,
    retry_backoff_ms = 1000,
    retry_multiplier = 2.0,
    retry_on_status = '[429, 500, 502, 503, 504]'
);
```

### Circuit Breaker

```sql
CREATE CONNECTION circuit_breaker_api FROM polling_http_source
WITH (
    endpoint = 'https://api.example.com/data',
    interval_ms = 30000,
    circuit_breaker_enabled = true,
    circuit_breaker_threshold = 5,
    circuit_breaker_timeout_ms = 60000
);
```

## Monitoring

### Metrics

- `laminar_http_requests_total` - Total HTTP requests made
- `laminar_http_request_duration_seconds` - Request duration histogram
- `laminar_http_response_status` - Response status codes
- `laminar_http_bytes_received` - Total bytes received
- `laminar_http_pagination_pages` - Pages fetched per poll

### Health Checks

```sql
SELECT * FROM system.connector_status
WHERE connector_name = 'polling_http_source';
```

## Best Practices

1. **Polling Interval**: Balance between data freshness and API rate limits
2. **Pagination**: Use appropriate strategy based on API design
3. **State Management**: Implement incremental polling to avoid duplicate data
4. **Error Handling**: Configure retries and circuit breakers for resilience
5. **Authentication**: Use secure credential management

## Troubleshooting

### Request Debugging

```sql
-- View recent requests
SELECT * FROM system.connector_logs
WHERE connector = 'polling_http_source'
AND message LIKE '%HTTP%'
ORDER BY timestamp DESC
LIMIT 10;
```

### Response Analysis

```sql
-- Check response patterns
SELECT 
    status_code,
    COUNT(*) as count,
    AVG(response_time_ms) as avg_time
FROM system.http_metrics
WHERE connector = 'polling_http_source'
GROUP BY status_code;
```

## Example Use Cases

### Weather Data Integration

```sql
CREATE TABLE weather_data (
    location TEXT,
    timestamp TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT,
    conditions TEXT
) WITH (
    connector = 'polling_http_source',
    connection = 'weather_api',
    endpoint = 'https://api.weather.com/current',
    interval_ms = 300000,
    query_params = '{"units": "metric", "locations": "NYC,LA,CHI"}'
);
```

### Stock Price Monitoring

```sql
CREATE TABLE stock_prices (
    symbol TEXT,
    timestamp TIMESTAMP,
    price DECIMAL,
    volume BIGINT
) WITH (
    connector = 'polling_http_source',
    connection = 'stock_api',
    endpoint = 'https://api.stocks.com/quotes',
    interval_ms = 60000,
    query_params = '{"symbols": "AAPL,GOOGL,MSFT"}'
);
```

### GitHub Repository Stats

```sql
CREATE TABLE github_stats (
    repo TEXT,
    stars INTEGER,
    forks INTEGER,
    issues INTEGER,
    updated_at TIMESTAMP
) WITH (
    connector = 'polling_http_source',
    connection = 'github_api',
    endpoint = 'https://api.github.com/repos/laminar/laminar',
    interval_ms = 3600000,
    headers = '{"Accept": "application/vnd.github.v3+json"}'
);
```