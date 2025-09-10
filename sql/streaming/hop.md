---
title: HOP Window
description: Sliding windows with configurable overlap for streaming aggregations
---

# HOP Window Function

## Description

The `HOP` function creates fixed-size, overlapping (hopping/sliding) windows over streaming data. Unlike tumbling windows, hop windows can overlap, allowing events to belong to multiple windows. This is ideal for computing moving averages, trend detection, and smooth time-series analytics.

## Syntax

```sql
HOP(hop_size, window_size)
```

### Parameters
- **hop_size** (INTERVAL): The distance between window starts (slide interval)
- **window_size** (INTERVAL): The total size of each window

### Return Value
- **Type**: Window descriptor with `start` and `end` timestamps
- **Description**: A window object that can be used in GROUP BY clauses and accessed for window boundaries

## How Hopping Windows Work

Hopping windows create overlapping time intervals where each window slides forward by the hop size:

![Hopping Windows](/img/streaming/hop-window.svg)

Each event can belong to multiple windows (window_size / hop_size windows).

## Example

### Sample Streaming Data

```sql
CREATE TABLE sensor_metrics (
    sensor_id VARCHAR,
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    reading_time TIMESTAMP,
    WATERMARK FOR reading_time AS (reading_time - INTERVAL '10 seconds')
) WITH (
    connector = 'kafka',
    topic = 'iot_sensors',
    format = 'json'
);
```

### Basic Hopping Window Query

```sql
-- 10-minute windows sliding every 2 minutes
SELECT 
    AVG(temperature) as avg_temp,
    MIN(temperature) as min_temp,
    MAX(temperature) as max_temp,
    STDDEV(temperature) as temp_stddev,
    COUNT(*) as reading_count,
    HOP(INTERVAL '2 minutes', INTERVAL '10 minutes') as window
FROM sensor_metrics
GROUP BY window;
```

### Result Stream (Continuous Output)
| avg_temp | min_temp | max_temp | temp_stddev | reading_count | window.start        | window.end          |
|----------|----------|----------|-------------|---------------|-------------------|-------------------|
| 22.5     | 19.2     | 25.8     | 1.87        | 480          | 2024-03-15 10:00:00 | 2024-03-15 10:10:00 |
| 22.8     | 19.5     | 26.2     | 1.92        | 485          | 2024-03-15 10:02:00 | 2024-03-15 10:12:00 |
| 23.1     | 20.1     | 26.5     | 1.85        | 492          | 2024-03-15 10:04:00 | 2024-03-15 10:14:00 |
| 23.3     | 20.4     | 26.8     | 1.79        | 488          | 2024-03-15 10:06:00 | 2024-03-15 10:16:00 |

## Common Use Cases

### 1. Moving Averages for Trading

```sql
-- 5-minute moving average, updated every minute
CREATE TABLE trades (
    symbol VARCHAR,
    price DECIMAL(10,2),
    volume BIGINT,
    trade_time TIMESTAMP,
    WATERMARK FOR trade_time AS (trade_time - INTERVAL '5 seconds')
) WITH (
    connector = 'kafka',
    topic = 'market_data'
);

SELECT 
    symbol,
    AVG(price) as moving_avg_price,
    SUM(volume) as total_volume,
    MIN(price) as period_low,
    MAX(price) as period_high,
    COUNT(*) as trade_count,
    HOP(INTERVAL '1 minute', INTERVAL '5 minutes') as window
FROM trades
GROUP BY symbol, window
ORDER BY window.start DESC, symbol;
```

### 2. Trend Detection

```sql
-- Detect temperature trends with overlapping windows
WITH temperature_windows AS (
    SELECT 
        sensor_id,
        AVG(temperature) as avg_temp,
        HOP(INTERVAL '5 minutes', INTERVAL '30 minutes') as window
    FROM sensor_metrics
    GROUP BY sensor_id, window
),
temperature_trends AS (
    SELECT 
        sensor_id,
        avg_temp,
        LAG(avg_temp) OVER (PARTITION BY sensor_id ORDER BY window.start) as prev_avg,
        window
    FROM temperature_windows
)
SELECT 
    sensor_id,
    avg_temp,
    prev_avg,
    CASE 
        WHEN avg_temp > prev_avg + 2 THEN 'RISING_FAST'
        WHEN avg_temp > prev_avg THEN 'RISING'
        WHEN avg_temp < prev_avg - 2 THEN 'FALLING_FAST'
        WHEN avg_temp < prev_avg THEN 'FALLING'
        ELSE 'STABLE'
    END as trend,
    window.start as window_start
FROM temperature_trends
WHERE prev_avg IS NOT NULL;
```

### 3. Smooth Real-Time Analytics

```sql
-- Website traffic with smooth transitions
CREATE TABLE page_views (
    user_id BIGINT,
    page_url VARCHAR,
    referrer VARCHAR,
    duration_seconds INT,
    view_time TIMESTAMP,
    WATERMARK FOR view_time AS (view_time - INTERVAL '30 seconds')
) WITH (
    connector = 'kafka',
    topic = 'clickstream'
);

-- 15-minute windows, updated every 3 minutes
SELECT 
    COUNT(DISTINCT user_id) as unique_users,
    COUNT(*) as page_views,
    AVG(duration_seconds) as avg_duration,
    COUNT(*) / COUNT(DISTINCT user_id) as pages_per_user,
    HOP(INTERVAL '3 minutes', INTERVAL '15 minutes') as window
FROM page_views
GROUP BY window;
```

### 4. Anomaly Detection with Baseline

```sql
-- Network traffic anomaly detection
CREATE TABLE network_traffic (
    source_ip VARCHAR,
    destination_ip VARCHAR,
    bytes_transferred BIGINT,
    packet_count INT,
    timestamp TIMESTAMP,
    WATERMARK FOR timestamp AS (timestamp - INTERVAL '10 seconds')
) WITH (
    connector = 'kafka',
    topic = 'network_logs'
);

-- Baseline: 1-hour windows, updated every 10 minutes
WITH baseline AS (
    SELECT 
        source_ip,
        AVG(bytes_transferred) as avg_bytes,
        STDDEV(bytes_transferred) as stddev_bytes,
        HOP(INTERVAL '10 minutes', INTERVAL '1 hour') as window
    FROM network_traffic
    GROUP BY source_ip, window
),
-- Current: 10-minute windows, updated every 2 minutes  
current AS (
    SELECT 
        source_ip,
        AVG(bytes_transferred) as current_avg_bytes,
        MAX(bytes_transferred) as max_bytes,
        HOP(INTERVAL '2 minutes', INTERVAL '10 minutes') as window
    FROM network_traffic
    GROUP BY source_ip, window
)
SELECT 
    c.source_ip,
    c.current_avg_bytes,
    b.avg_bytes as baseline_avg,
    b.stddev_bytes,
    (c.current_avg_bytes - b.avg_bytes) / b.stddev_bytes as z_score,
    CASE 
        WHEN ABS((c.current_avg_bytes - b.avg_bytes) / b.stddev_bytes) > 3 
        THEN 'ANOMALY'
        ELSE 'NORMAL'
    END as status,
    c.window.start as detection_time
FROM current c
JOIN baseline b 
    ON c.source_ip = b.source_ip
    AND c.window.start >= b.window.start
    AND c.window.end <= b.window.end
WHERE b.stddev_bytes > 0;
```

### 5. User Engagement Metrics

```sql
-- Overlapping session windows for engagement analysis
CREATE TABLE user_events (
    user_id BIGINT,
    event_type VARCHAR,
    engagement_score INT,
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '1 minute')
) WITH (
    connector = 'kafka',
    topic = 'user_activity'
);

-- 30-minute engagement windows, updated every 5 minutes
SELECT 
    user_id,
    COUNT(*) as event_count,
    COUNT(DISTINCT event_type) as unique_events,
    SUM(engagement_score) as total_engagement,
    AVG(engagement_score) as avg_engagement,
    CASE 
        WHEN SUM(engagement_score) > 100 THEN 'HIGH'
        WHEN SUM(engagement_score) > 50 THEN 'MEDIUM'
        ELSE 'LOW'
    END as engagement_level,
    HOP(INTERVAL '5 minutes', INTERVAL '30 minutes') as window
FROM user_events
GROUP BY user_id, window
HAVING COUNT(*) > 5;  -- Active users only
```

## Overlap Factor and Performance

The overlap factor determines how many windows each event belongs to:

```sql
Overlap Factor = window_size / hop_size

Examples:
- HOP(1 min, 5 min) → Overlap Factor = 5 (each event in 5 windows)
- HOP(5 min, 10 min) → Overlap Factor = 2 (each event in 2 windows)
- HOP(10 min, 10 min) → Overlap Factor = 1 (equivalent to TUMBLE)
```

### Performance Implications

```sql
-- High overlap (more computation, smoother results)
SELECT 
    AVG(value) as smooth_avg,
    HOP(INTERVAL '1 second', INTERVAL '1 minute') as window  -- Factor: 60
FROM metrics
GROUP BY window;

-- Low overlap (less computation, more discrete)
SELECT 
    AVG(value) as discrete_avg,
    HOP(INTERVAL '30 seconds', INTERVAL '1 minute') as window  -- Factor: 2
FROM metrics
GROUP BY window;
```

## Advanced Patterns

### Multiple Hop Windows

```sql
-- Different granularities for comprehensive monitoring
WITH short_term AS (
    SELECT 
        AVG(cpu_usage) as avg_cpu_1min,
        HOP(INTERVAL '10 seconds', INTERVAL '1 minute') as window
    FROM system_metrics
    GROUP BY window
),
medium_term AS (
    SELECT 
        AVG(cpu_usage) as avg_cpu_5min,
        HOP(INTERVAL '1 minute', INTERVAL '5 minutes') as window
    FROM system_metrics
    GROUP BY window
),
long_term AS (
    SELECT 
        AVG(cpu_usage) as avg_cpu_15min,
        HOP(INTERVAL '5 minutes', INTERVAL '15 minutes') as window
    FROM system_metrics
    GROUP BY window
)
SELECT 
    s.avg_cpu_1min,
    m.avg_cpu_5min,
    l.avg_cpu_15min,
    s.window.start as timestamp
FROM short_term s
JOIN medium_term m ON s.window.start = m.window.start
JOIN long_term l ON s.window.start = l.window.start;
```

### Weighted Moving Averages

```sql
-- More weight to recent data using multiple hop windows
WITH weighted_windows AS (
    SELECT 
        symbol,
        price,
        trade_time,
        CASE 
            WHEN trade_time > window.end - INTERVAL '1 minute' THEN 3
            WHEN trade_time > window.end - INTERVAL '2 minutes' THEN 2
            ELSE 1
        END as weight,
        HOP(INTERVAL '1 minute', INTERVAL '3 minutes') as window
    FROM trades
)
SELECT 
    symbol,
    SUM(price * weight) / SUM(weight) as weighted_avg_price,
    window
FROM weighted_windows
GROUP BY symbol, window;
```

## Window Alignment

Hop windows align based on epoch time, with the hop size determining the start times:

```sql
-- 5-minute hop: windows start at :00, :05, :10, :15, etc.
HOP(INTERVAL '5 minutes', INTERVAL '15 minutes')
-- Windows: [00:00-00:15], [00:05-00:20], [00:10-00:25], ...

-- 2-minute hop: windows start at :00, :02, :04, :06, etc.
HOP(INTERVAL '2 minutes', INTERVAL '10 minutes')
-- Windows: [00:00-00:10], [00:02-00:12], [00:04-00:14], ...
```

## Best Practices

1. **Choose Appropriate Hop Size**: 
   - Smaller hop → smoother transitions, more computation
   - Larger hop → more discrete updates, less computation

2. **Consider Overlap Factor**:
   - Factor 2-3: Good balance for most use cases
   - Factor > 5: Only for critical smooth monitoring
   - Factor 1: Use TUMBLE instead

3. **Memory Management**:
   - Each event stored in multiple windows
   - Memory usage ≈ overlap_factor × data_size

4. **Late Data Handling**:
   - Set watermarks considering all overlapping windows
   - Late data affects multiple window results

5. **Output Management**:
   - Higher overlap means more frequent outputs
   - Consider downstream system capacity

## Comparison with Other Windows

| Aspect | TUMBLE | HOP | SESSION |
|--------|--------|-----|---------|
| Overlap | No | Yes | No |
| Size | Fixed | Fixed | Variable |
| Events per Window | Each in 1 | Each in multiple | Varies |
| Use Case | Discrete buckets | Smooth analytics | Activity-based |
| Computation | Low | Medium-High | Medium |
| Output Frequency | Per window_size | Per hop_size | Per session end |

## Related Functions
- [TUMBLE](./tumble.md) - Non-overlapping fixed windows
- [SESSION](./session.md) - Gap-based activity windows
- [Watermarks](./watermarks.md) - Handling event time and late data
- [Windowed Aggregations](./windowed-aggregations.md) - Aggregation patterns