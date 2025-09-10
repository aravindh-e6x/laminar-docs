---
title: TUMBLE Window
description: Fixed-size, non-overlapping time windows for streaming aggregations
---

# TUMBLE Window Function

## Description

The `TUMBLE` function creates fixed-size, non-overlapping (tumbling) time windows over streaming data. Each event belongs to exactly one window, making it ideal for creating time-based buckets for aggregations like hourly reports, daily summaries, or fixed-interval metrics.

## Syntax

```sql
TUMBLE(window_size)
```

### Parameters
- **window_size** (INTERVAL): The size of each tumbling window (e.g., `INTERVAL '1 hour'`, `INTERVAL '5 minutes'`)

### Return Value
- **Type**: Window descriptor with `start` and `end` timestamps
- **Description**: A window object that can be used in GROUP BY clauses and accessed for window boundaries

## How Tumbling Windows Work

Tumbling windows divide the time axis into fixed-size, consecutive, non-overlapping intervals:

![Tumbling Windows](/img/streaming/tumble-window.svg)

Each event is assigned to exactly one window based on its timestamp.

## Example

### Sample Streaming Data
Consider a continuous stream of transaction events:

```sql
CREATE TABLE transactions (
    transaction_id BIGINT,
    user_id BIGINT,
    amount DECIMAL(10,2),
    category VARCHAR,
    transaction_time TIMESTAMP,
    WATERMARK FOR transaction_time AS (transaction_time - INTERVAL '30 seconds')
) WITH (
    connector = 'kafka',
    topic = 'transactions',
    format = 'json'
);
```

### Basic Tumbling Window Query

```sql
-- 5-minute tumbling windows for transaction summaries
SELECT 
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount,
    MIN(amount) as min_amount,
    MAX(amount) as max_amount,
    TUMBLE(INTERVAL '5 minutes') as window
FROM transactions
GROUP BY window;
```

### Result Stream (Continuous Output)
| transaction_count | total_amount | avg_amount | min_amount | max_amount | window.start        | window.end          |
|------------------|-------------|------------|------------|------------|-------------------|-------------------|
| 145              | 28450.00    | 196.21     | 5.99       | 1250.00    | 2024-03-15 10:00:00 | 2024-03-15 10:05:00 |
| 162              | 31200.50    | 192.60     | 8.50       | 2100.00    | 2024-03-15 10:05:00 | 2024-03-15 10:10:00 |
| 138              | 26800.75    | 194.21     | 12.00      | 1850.00    | 2024-03-15 10:10:00 | 2024-03-15 10:15:00 |

## Common Use Cases

### 1. Hourly Metrics Dashboard

```sql
-- Hourly business metrics
SELECT 
    category,
    COUNT(*) as orders_per_hour,
    SUM(amount) as hourly_revenue,
    COUNT(DISTINCT user_id) as unique_customers,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) as median_order_value,
    TUMBLE(INTERVAL '1 hour') as hour_window
FROM transactions
GROUP BY category, hour_window;
```

### 2. Real-Time Error Monitoring

```sql
-- 1-minute error rate monitoring
CREATE TABLE application_logs (
    log_level VARCHAR,
    service_name VARCHAR,
    message TEXT,
    timestamp TIMESTAMP,
    WATERMARK FOR timestamp AS (timestamp - INTERVAL '5 seconds')
) WITH (
    connector = 'kafka',
    topic = 'app_logs'
);

SELECT 
    service_name,
    COUNT(*) as total_logs,
    COUNT(*) FILTER (WHERE log_level = 'ERROR') as error_count,
    COUNT(*) FILTER (WHERE log_level = 'WARN') as warning_count,
    ROUND(COUNT(*) FILTER (WHERE log_level = 'ERROR') * 100.0 / COUNT(*), 2) as error_rate,
    TUMBLE(INTERVAL '1 minute') as window
FROM application_logs
GROUP BY service_name, window
HAVING COUNT(*) FILTER (WHERE log_level = 'ERROR') > 10;  -- Alert threshold
```

### 3. IoT Sensor Aggregations

```sql
-- 10-second sensor reading aggregations
CREATE TABLE sensor_readings (
    sensor_id VARCHAR,
    location VARCHAR,
    temperature DOUBLE,
    humidity DOUBLE,
    reading_time TIMESTAMP,
    WATERMARK FOR reading_time AS (reading_time - INTERVAL '2 seconds')
) WITH (
    connector = 'mqtt',
    topic = 'sensors/+/data'
);

SELECT 
    location,
    COUNT(DISTINCT sensor_id) as active_sensors,
    AVG(temperature) as avg_temperature,
    STDDEV(temperature) as temp_std_dev,
    MIN(temperature) as min_temp,
    MAX(temperature) as max_temp,
    AVG(humidity) as avg_humidity,
    TUMBLE(INTERVAL '10 seconds') as window
FROM sensor_readings
GROUP BY location, window;
```

### 4. Traffic Analysis

```sql
-- 15-minute website traffic analysis
SELECT 
    page_category,
    COUNT(*) as page_views,
    COUNT(DISTINCT session_id) as unique_sessions,
    COUNT(DISTINCT user_id) as unique_users,
    AVG(time_on_page) as avg_time_on_page,
    TUMBLE(INTERVAL '15 minutes') as window
FROM page_views
GROUP BY page_category, window
ORDER BY window.start, page_views DESC;
```

### 5. Financial Market Data

```sql
-- 1-minute OHLC (candlestick) data
SELECT 
    symbol,
    FIRST_VALUE(price) as open,
    MAX(price) as high,
    MIN(price) as low,
    LAST_VALUE(price) as close,
    SUM(volume) as volume,
    COUNT(*) as trade_count,
    TUMBLE(INTERVAL '1 minute') as window
FROM trades
GROUP BY symbol, window;
```

## Window Boundaries and Alignment

Tumbling windows are aligned to epoch time (1970-01-01 00:00:00 UTC):

```sql
-- 1-hour windows align to clock hours
-- Window boundaries: 00:00, 01:00, 02:00, etc.
SELECT 
    TUMBLE(INTERVAL '1 hour') as window,
    COUNT(*) as event_count
FROM events
GROUP BY window;

-- 5-minute windows align to 5-minute marks
-- Window boundaries: 00:00, 00:05, 00:10, etc.
SELECT 
    TUMBLE(INTERVAL '5 minutes') as window,
    COUNT(*) as event_count
FROM events
GROUP BY window;
```

## Accessing Window Properties

```sql
-- Access window start and end times
SELECT 
    window.start as window_start,
    window.end as window_end,
    window.end - window.start as window_duration,
    COUNT(*) as event_count,
    TUMBLE(INTERVAL '30 minutes') as window
FROM events
GROUP BY window;
```

## Combining with Other Clauses

### With WHERE Filters

```sql
-- Tumbling windows with filtering
SELECT 
    category,
    SUM(amount) as total_sales,
    TUMBLE(INTERVAL '1 hour') as window
FROM transactions
WHERE amount > 100
GROUP BY category, window;
```

### With HAVING Conditions

```sql
-- Only output windows meeting criteria
SELECT 
    store_id,
    COUNT(*) as transaction_count,
    SUM(amount) as total_revenue,
    TUMBLE(INTERVAL '30 minutes') as window
FROM sales
GROUP BY store_id, window
HAVING SUM(amount) > 10000;  -- High-volume windows only
```

### With JOIN Operations

```sql
-- Join aggregated windows
WITH hourly_sales AS (
    SELECT 
        product_id,
        SUM(quantity) as units_sold,
        TUMBLE(INTERVAL '1 hour') as window
    FROM sales
    GROUP BY product_id, window
),
hourly_inventory AS (
    SELECT 
        product_id,
        AVG(stock_level) as avg_stock,
        TUMBLE(INTERVAL '1 hour') as window
    FROM inventory_updates
    GROUP BY product_id, window
)
SELECT 
    s.product_id,
    s.units_sold,
    i.avg_stock,
    s.window.start as hour
FROM hourly_sales s
JOIN hourly_inventory i
    ON s.product_id = i.product_id
    AND s.window = i.window
WHERE s.units_sold > i.avg_stock * 0.5;  -- High demand alert
```

## Late Data Handling

Events arriving after the watermark are handled based on the configured strategy:

```sql
-- Watermark allows 1 minute of late data
CREATE TABLE events (
    event_id BIGINT,
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '1 minute')
) WITH (
    connector = 'kafka',
    topic = 'events'
);

-- Windows close 1 minute after window.end
SELECT 
    COUNT(*) as event_count,
    TUMBLE(INTERVAL '5 minutes') as window
FROM events
GROUP BY window;
```

## Performance Considerations

1. **Window Size**: Smaller windows require more frequent state updates
2. **State Management**: Each window maintains aggregate state until watermark passes
3. **Output Frequency**: Results are emitted when watermark crosses window boundary
4. **Memory Usage**: Proportional to number of keys × number of open windows

## Best Practices

1. **Choose Appropriate Window Size**: Balance between granularity and performance
2. **Set Reasonable Watermarks**: Allow for typical late data without excessive delay
3. **Use Indexes on Time Columns**: Improve windowing performance
4. **Monitor State Size**: Track memory usage for high-cardinality groupings
5. **Consider Downstream Systems**: Ensure consumers can handle output rate

## Comparison with Other Windows

| Window Type | Overlapping | Size | Use Case |
|------------|-------------|------|----------|
| **TUMBLE** | No | Fixed | Period-based reporting |
| **HOP** | Yes | Fixed | Moving averages |
| **SESSION** | No | Variable | User activity analysis |

## Related Functions
- [HOP](./hop.md) - Sliding windows with overlap
- [SESSION](./session.md) - Session-based windows
- [Watermarks](./watermarks.md) - Handling event time and late data
- [Windowed Aggregations](./windowed-aggregations.md) - Aggregation patterns