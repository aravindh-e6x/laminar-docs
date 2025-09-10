---
title: Watermarks
description: Managing event time and handling late data in streaming queries
---


## Overview

Watermarks are timestamps that indicate the progress of event time in a streaming system. They serve as a declaration that no events with timestamps older than the watermark should be expected, allowing the system to trigger computations and emit results. Watermarks are crucial for handling out-of-order data and determining when time-based windows can be closed.

## Understanding Watermarks

### The Problem Watermarks Solve

In distributed streaming systems, events often arrive out of order due to:
- Network delays
- Processing delays  
- Clock skew between systems
- Retries and reprocessing

Without watermarks, the system cannot determine when it has received all events for a time window.

### How Watermarks Work

![Watermarks](/img/streaming/watermarks.svg)

## Defining Watermarks

### Basic Syntax

```sql
CREATE TABLE events (
    event_id BIGINT,
    user_id BIGINT,
    event_data VARCHAR,
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '30 seconds')
) WITH (
    connector = 'kafka',
    topic = 'events',
    format = 'json'
);
```

### Watermark Strategies

#### 1. Fixed Delay Watermark

Most common strategy - assumes maximum out-of-orderness:

```sql
-- Allow 1 minute for late data
WATERMARK FOR event_time AS (event_time - INTERVAL '1 minute')
```

#### 2. Bounded Out-of-Orderness

```sql
-- Events can be at most 5 minutes late
WATERMARK FOR timestamp_column AS (timestamp_column - INTERVAL '5 minutes')
```

#### 3. Percentage-Based Watermark

```sql
-- Watermark at 99th percentile of event time
-- (Conceptual - implementation varies by system)
WATERMARK FOR event_time AS PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY event_time)
```

## Examples

### E-commerce Order Processing

```sql
-- Orders stream with potential delays
CREATE TABLE orders (
    order_id VARCHAR,
    customer_id BIGINT,
    amount DECIMAL(10,2),
    order_status VARCHAR,
    order_time TIMESTAMP,
    WATERMARK FOR order_time AS (order_time - INTERVAL '2 minutes')
) WITH (
    connector = 'kafka',
    topic = 'orders',
    format = 'json'
);

-- Hourly sales with 2-minute late data tolerance
SELECT 
    DATE_TRUNC('hour', order_time) as hour,
    COUNT(*) as order_count,
    SUM(amount) as total_sales,
    COUNT(DISTINCT customer_id) as unique_customers
FROM orders
WHERE order_status = 'COMPLETED'
GROUP BY DATE_TRUNC('hour', order_time);
```

### Real-Time Monitoring with Strict Watermarks

```sql
-- Critical system metrics with minimal delay
CREATE TABLE system_metrics (
    server_id VARCHAR,
    metric_name VARCHAR,
    metric_value DOUBLE,
    severity VARCHAR,
    metric_time TIMESTAMP,
    WATERMARK FOR metric_time AS (metric_time - INTERVAL '5 seconds')
) WITH (
    connector = 'kafka',
    topic = 'metrics',
    format = 'json'
);

-- 1-minute alert windows with 5-second tolerance
SELECT 
    server_id,
    COUNT(*) FILTER (WHERE severity = 'CRITICAL') as critical_count,
    AVG(metric_value) as avg_value,
    TUMBLE(INTERVAL '1 minute') as window
FROM system_metrics
WHERE metric_name = 'CPU_USAGE'
GROUP BY server_id, window
HAVING COUNT(*) FILTER (WHERE severity = 'CRITICAL') > 3;
```

### IoT Sensor Data with Irregular Arrivals

```sql
-- Sensors may have connectivity issues
CREATE TABLE sensor_readings (
    sensor_id VARCHAR,
    location VARCHAR,
    temperature DOUBLE,
    humidity DOUBLE,
    battery_level INT,
    reading_time TIMESTAMP,
    WATERMARK FOR reading_time AS (reading_time - INTERVAL '10 minutes')
) WITH (
    connector = 'mqtt',
    topic = 'sensors/+/data'
);

-- Daily aggregations with 10-minute grace period
SELECT 
    location,
    DATE(reading_time) as reading_date,
    AVG(temperature) as avg_temperature,
    MIN(temperature) as min_temperature,
    MAX(temperature) as max_temperature,
    COUNT(*) as reading_count
FROM sensor_readings
GROUP BY location, DATE(reading_time);
```

## Watermark Propagation

### Through Single Streams

```sql
-- Watermark flows through transformations
WITH filtered_events AS (
    SELECT * FROM events 
    WHERE event_type = 'PURCHASE'
    -- Watermark automatically propagated
)
SELECT 
    user_id,
    COUNT(*) as purchase_count,
    TUMBLE(INTERVAL '1 hour') as window
FROM filtered_events
GROUP BY user_id, window;
```

### Through Joins

```sql
-- Orders with 2-minute watermark
CREATE TABLE orders (
    order_id VARCHAR,
    customer_id BIGINT,
    order_time TIMESTAMP,
    WATERMARK FOR order_time AS (order_time - INTERVAL '2 minutes')
) WITH (...);

-- Payments with 5-minute watermark  
CREATE TABLE payments (
    payment_id VARCHAR,
    order_id VARCHAR,
    payment_time TIMESTAMP,
    WATERMARK FOR payment_time AS (payment_time - INTERVAL '5 minutes')
) WITH (...);

-- Join uses minimum watermark (more conservative)
SELECT 
    o.order_id,
    o.order_time,
    p.payment_time,
    p.payment_time - o.order_time as payment_delay
FROM orders o
JOIN payments p ON o.order_id = p.order_id
WHERE p.payment_time BETWEEN o.order_time 
    AND o.order_time + INTERVAL '30 minutes';
-- Effective watermark: MIN(order_watermark, payment_watermark)
```

### Through Unions

```sql
-- Multiple data sources with different delays
CREATE TABLE mobile_events (
    user_id BIGINT,
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '1 minute')
) WITH (...);

CREATE TABLE web_events (
    user_id BIGINT,
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '30 seconds')
) WITH (...);

-- Union uses minimum watermark
SELECT * FROM mobile_events
UNION ALL
SELECT * FROM web_events;
-- Effective watermark: MIN(mobile_watermark, web_watermark)
```

## Late Data Handling

### Strategies for Late Events

#### 1. Drop Late Data (Default)

```sql
-- Events arriving after watermark are dropped
CREATE TABLE strict_events (
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '10 seconds')
) WITH (
    connector = 'kafka',
    late_data_strategy = 'DROP'  -- Hypothetical configuration
);
```

#### 2. Allow Late Data with Side Output

```sql
-- Capture late events for separate processing
CREATE TABLE events_with_late_handling (
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '1 minute')
) WITH (
    connector = 'kafka',
    late_data_strategy = 'SIDE_OUTPUT',
    late_data_topic = 'late_events'
);
```

#### 3. Update Results (Allowed Lateness)

```sql
-- Allow updates for late data within grace period
SELECT 
    user_id,
    COUNT(*) as event_count,
    TUMBLE(INTERVAL '5 minutes') as window
FROM events
GROUP BY user_id, window
-- WITH ALLOWED LATENESS = INTERVAL '10 minutes'  -- System-specific syntax
-- Results can be updated for 10 minutes after window closes
```

## Impact on Windows

The watermark determines when windows close and results are emitted. Different window types interact with watermarks in unique ways:

### Tumbling Windows

```sql
-- Window closes when watermark passes window end
SELECT 
    COUNT(*) as count,
    TUMBLE(INTERVAL '5 minutes') as window
FROM events
GROUP BY window;
-- 10:00-10:05 window closes when watermark > 10:05
```

### Session Windows

```sql
-- Session closes when watermark passes last event + gap
SELECT 
    user_id,
    COUNT(*) as events,
    SESSION(INTERVAL '30 minutes') as session
FROM user_events
GROUP BY user_id, session;
-- Session with last event at 10:30 closes when watermark > 11:00
```

## Watermark Patterns

### Multi-Stage Processing

```sql
-- Stage 1: Aggressive watermark for quick results
CREATE VIEW real_time_metrics AS
SELECT 
    metric_name,
    AVG(value) as avg_value,
    TUMBLE(INTERVAL '1 minute') as window
FROM (
    SELECT *, 
    event_time - INTERVAL '10 seconds' as quick_watermark
    FROM metrics
)
GROUP BY metric_name, window;

-- Stage 2: Conservative watermark for accuracy
CREATE VIEW accurate_metrics AS
SELECT 
    metric_name,
    AVG(value) as avg_value,
    TUMBLE(INTERVAL '1 minute') as window
FROM (
    SELECT *,
    event_time - INTERVAL '5 minutes' as accurate_watermark
    FROM metrics
)
GROUP BY metric_name, window;
```

### Adaptive Watermarks

```sql
-- Conceptual: Adjust watermark based on observed delays
WITH delay_stats AS (
    SELECT 
        PERCENTILE_CONT(0.99) WITHIN GROUP (
            ORDER BY processing_time - event_time
        ) as p99_delay
    FROM events
    WHERE processing_time > CURRENT_TIMESTAMP - INTERVAL '1 hour'
)
SELECT 
    event_time - (SELECT p99_delay FROM delay_stats) as adaptive_watermark,
    *
FROM events;
```

## Performance Considerations

### Memory Impact

```
Watermark Delay | Memory Usage | Completeness | Latency
----------------|--------------|--------------|----------
10 seconds      | Low          | 95%          | Very Low
1 minute        | Medium       | 99%          | Low
5 minutes       | High         | 99.9%        | Medium
30 minutes      | Very High    | 99.99%       | High
```

### State Management

```sql
-- Longer watermark delays = more state to maintain
-- State size ≈ (event_rate × watermark_delay × key_cardinality)

-- Example: 1000 events/sec, 5-minute watermark, 1000 keys
-- State size ≈ 1000 × 300 × 1000 = 300M events in memory
```

## Best Practices

### 1. Analyze Data Characteristics

```sql
-- Measure actual out-of-orderness
SELECT 
    PERCENTILE_CONT(0.99) WITHIN GROUP (
        ORDER BY received_time - event_time
    ) as p99_delay,
    PERCENTILE_CONT(0.999) WITHIN GROUP (
        ORDER BY received_time - event_time  
    ) as p999_delay,
    MAX(received_time - event_time) as max_delay
FROM events
WHERE received_time > CURRENT_TIMESTAMP - INTERVAL '1 day';
```

### 2. Balance Latency vs Completeness

```sql
-- Quick results with potential incompleteness
WATERMARK FOR event_time AS (event_time - INTERVAL '30 seconds')

-- Complete results with higher latency
WATERMARK FOR event_time AS (event_time - INTERVAL '5 minutes')
```

### 3. Monitor Watermark Progress

```sql
-- Track watermark advancement
SELECT 
    CURRENT_WATERMARK() as current_watermark,
    MAX(event_time) as max_event_time,
    CURRENT_TIMESTAMP as current_time,
    CURRENT_TIMESTAMP - CURRENT_WATERMARK() as watermark_lag
FROM events;
```

### 4. Handle Source-Specific Delays

```sql
-- Different watermarks for different sources
CREATE TABLE mobile_events (
    event_time TIMESTAMP,
    -- Mobile apps may have offline periods
    WATERMARK FOR event_time AS (event_time - INTERVAL '10 minutes')
) WITH (...);

CREATE TABLE server_events (
    event_time TIMESTAMP,
    -- Servers have reliable connectivity
    WATERMARK FOR event_time AS (event_time - INTERVAL '30 seconds')
) WITH (...);
```

## Troubleshooting

### Watermark Not Advancing

```sql
-- Check for stuck partitions
SELECT 
    partition_id,
    MIN(event_time) as oldest_event,
    MAX(event_time) as newest_event,
    COUNT(*) as event_count
FROM events
GROUP BY partition_id
ORDER BY oldest_event;
```

### Excessive Late Data

```sql
-- Monitor late event rate
WITH watermarked_events AS (
    SELECT 
        *,
        CASE 
            WHEN event_time < CURRENT_WATERMARK() THEN 'LATE'
            ELSE 'ON_TIME'
        END as timeliness
    FROM events
)
SELECT 
    timeliness,
    COUNT(*) as count,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () as percentage
FROM watermarked_events
GROUP BY timeliness;
```

## Related Topics
- [TUMBLE Windows](./tumble.md) - Fixed-size time windows
- [HOP Windows](./hop.md) - Sliding time windows
- [SESSION Windows](./session.md) - Gap-based activity windows
- [Event Time Processing](./event-time.md) - Working with event timestamps
- [Stream Processing Overview](./overview.md) - Streaming SQL concepts