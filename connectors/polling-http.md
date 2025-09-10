---
sidebar_position: 10
title: Polling HTTP
---

The Polling HTTP connector enables Laminar to periodically fetch data from HTTP endpoints and process changes.

## Capabilities

- **Source**: Yes
- **Sink**: No
- **Formats**: JSON, Raw
- **Methods**: GET, POST, PUT, PATCH
- **Change Detection**: Can emit only on changes or all responses

## Configuration

### Table Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | - | HTTP endpoint URL |
| `headers` | string | No | - | Comma-separated headers |
| `method` | enum | No | GET | HTTP method: GET, POST, PUT, PATCH |
| `body` | string | No | - | Request body for POST/PUT/PATCH |
| `pollIntervalMs` | integer | No | 1000 | Polling interval in milliseconds |
| `emitBehavior` | enum | No | all | Emit behavior: `all` or `changed` |

## API Usage

### Create Polling Source

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "api_monitor",
    "connector": "polling_http",
    "config": {
      "endpoint": "https://api.example.com/status",
      "headers": "Authorization: Bearer {{ API_TOKEN }}",
      "method": "GET",
      "pollIntervalMs": 5000,
      "emitBehavior": "changed"
    }
  }'
```

## SQL Usage

### Basic Polling

```sql
CREATE TABLE api_status (
    status TEXT,
    services JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'polling_http',
    endpoint = 'https://api.example.com/health',
    poll_interval_ms = '10000',
    emit_behavior = 'changed',
    format = 'json'
);
```

### Authenticated Polling

```sql
CREATE TABLE metrics_data (
    metric_name TEXT,
    value DOUBLE,
    labels JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'polling_http',
    endpoint = 'https://metrics.internal/api/v1/current',
    headers = 'Authorization: Bearer {{ METRICS_TOKEN }},Accept: application/json',
    method = 'GET',
    poll_interval_ms = '5000',
    format = 'json'
);
```

## Examples

### Service Health Monitoring

```sql
-- Poll multiple service health endpoints
CREATE TABLE service_health (
    service_name TEXT,
    status TEXT,
    response_time_ms INTEGER,
    error_count INTEGER,
    last_check TIMESTAMP
) WITH (
    connector = 'polling_http',
    endpoint = 'https://status.example.com/api/services',
    headers = 'X-API-Key: {{ STATUS_API_KEY }}',
    poll_interval_ms = '30000',
    emit_behavior = 'all',
    format = 'json'
);

-- Alert on unhealthy services
CREATE TABLE health_alerts (
    service_name TEXT,
    status TEXT,
    alert_message TEXT,
    alert_time TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'health-alerts',
    type = 'sink',
    format = 'json'
);

INSERT INTO health_alerts
SELECT 
    service_name,
    status,
    CONCAT('Service ', service_name, ' is ', status, ' with ', error_count, ' errors') as alert_message,
    NOW() as alert_time
FROM service_health
WHERE status != 'healthy' OR error_count > 0;
```

### Price Feed Monitoring

```sql
-- Poll cryptocurrency prices
CREATE TABLE crypto_prices (
    symbol TEXT,
    price DECIMAL,
    volume DECIMAL,
    market_cap DECIMAL,
    last_updated TIMESTAMP
) WITH (
    connector = 'polling_http',
    endpoint = 'https://api.crypto.com/v1/prices',
    poll_interval_ms = '1000',
    emit_behavior = 'changed',
    format = 'json'
);

-- Track price changes
CREATE TABLE price_changes (
    symbol TEXT,
    old_price DECIMAL,
    new_price DECIMAL,
    change_percent DOUBLE,
    change_time TIMESTAMP
) WITH (
    connector = 'webhook',
    url = 'https://alerts.example.com/price-changes',
    type = 'sink',
    format = 'json'
);

INSERT INTO price_changes
SELECT 
    c.symbol,
    LAG(c.price) OVER (PARTITION BY c.symbol ORDER BY c.last_updated) as old_price,
    c.price as new_price,
    ((c.price - LAG(c.price) OVER (PARTITION BY c.symbol ORDER BY c.last_updated)) / 
     LAG(c.price) OVER (PARTITION BY c.symbol ORDER BY c.last_updated) * 100) as change_percent,
    c.last_updated as change_time
FROM crypto_prices c
WHERE LAG(c.price) OVER (PARTITION BY c.symbol ORDER BY c.last_updated) IS NOT NULL
  AND ABS((c.price - LAG(c.price) OVER (PARTITION BY c.symbol ORDER BY c.last_updated)) / 
          LAG(c.price) OVER (PARTITION BY c.symbol ORDER BY c.last_updated)) > 0.05;
```

### Weather Data Collection

```sql
-- Poll weather API
CREATE TABLE weather_data (
    location TEXT,
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    wind_speed DOUBLE,
    conditions TEXT,
    observation_time TIMESTAMP
) WITH (
    connector = 'polling_http',
    endpoint = 'https://api.weather.com/v1/current',
    headers = 'X-RapidAPI-Key: {{ WEATHER_API_KEY }}',
    method = 'POST',
    body = '{"locations": ["NYC", "LAX", "ORD"]}',
    poll_interval_ms = '300000',  -- 5 minutes
    emit_behavior = 'all',
    format = 'json'
);

-- Aggregate hourly averages
CREATE TABLE weather_hourly (
    location TEXT,
    avg_temperature DOUBLE,
    avg_humidity DOUBLE,
    max_wind_speed DOUBLE,
    hour_start TIMESTAMP
) WITH (
    connector = 'kafka',
    bootstrap_servers = 'localhost:9092',
    topic = 'weather-hourly',
    type = 'sink',
    format = 'json'
);

INSERT INTO weather_hourly
SELECT 
    location,
    AVG(temperature) as avg_temperature,
    AVG(humidity) as avg_humidity,
    MAX(wind_speed) as max_wind_speed,
    TUMBLE_START(observation_time, INTERVAL '1' HOUR) as hour_start
FROM weather_data
GROUP BY 
    location,
    TUMBLE(observation_time, INTERVAL '1' HOUR);
```

### Inventory Sync

```sql
-- Poll inventory API for changes
CREATE TABLE inventory_updates (
    sku TEXT,
    warehouse_id TEXT,
    quantity INTEGER,
    reserved INTEGER,
    available INTEGER,
    last_modified TIMESTAMP
) WITH (
    connector = 'polling_http',
    endpoint = 'https://erp.company.com/api/inventory/changes',
    headers = 'Authorization: Basic {{ ERP_CREDENTIALS }}',
    method = 'GET',
    poll_interval_ms = '60000',
    emit_behavior = 'changed',
    format = 'json'
);

-- Process low stock alerts
CREATE TABLE low_stock_alerts (
    sku TEXT,
    warehouse_id TEXT,
    available INTEGER,
    reorder_point INTEGER,
    alert_time TIMESTAMP
) WITH (
    connector = 'redis',
    address = 'redis://localhost:6379',
    type = 'sink',
    'sink.type' = 'list',
    'sink.list_prefix' = 'alerts:low-stock:',
    'sink.operation' = 'Append',
    format = 'json'
);

INSERT INTO low_stock_alerts
SELECT 
    sku,
    warehouse_id,
    available,
    50 as reorder_point,  -- Default reorder point
    NOW() as alert_time
FROM inventory_updates
WHERE available < 50;
```

## Emit Behavior

### `all`
- Emits a record for every successful poll
- Useful for time-series data or regular snapshots
- Can result in duplicate data if endpoint returns same response

### `changed`
- Only emits when response differs from previous poll
- Reduces data volume for slowly changing endpoints
- Compares entire response body for changes
- First poll always emits

## Best Practices

1. **Polling Interval**: Balance between data freshness and API rate limits
2. **Authentication**: Use environment variables for API keys and tokens
3. **Error Handling**: Failed polls are retried with exponential backoff
4. **Change Detection**: Use `changed` for slowly changing data to reduce volume
5. **Request Body**: Use for POST/PUT endpoints that require parameters
6. **Headers**: Include all required headers including Content-Type

## Limitations

- Source only (no sink support)
- No support for OAuth flows (use pre-generated tokens)
- No pagination support for large responses
- Response size limited to available memory
- No support for binary responses
- No custom retry policies