---
title: Streaming SQL Overview
description: Introduction to streaming SQL concepts and capabilities in Laminar
---


## Introduction

Laminar extends standard SQL with powerful streaming capabilities, enabling real-time data processing and analytics. Unlike traditional batch SQL that operates on static datasets, streaming SQL processes continuous data flows, providing immediate insights as data arrives.

## Key Concepts

### Streams vs Tables

In streaming SQL, data exists in two fundamental forms:

![Stream-Table Duality](/img/streaming/stream-table-duality.svg)

- **Streams**: Unbounded sequences of events that continuously flow through the system
- **Tables**: Materialized views of streams at specific points in time

### Time Semantics

Streaming SQL operates with two distinct notions of time:

#### Event Time
The time when an event actually occurred in the real world, embedded within the event data itself. This is the most accurate representation for analytics.

```sql
CREATE TABLE events (
    user_id BIGINT,
    action VARCHAR,
    timestamp TIMESTAMP
) WITH (
    connector = 'kafka',
    event_time_field = 'timestamp'  -- Specify event time column
);
```

#### Processing Time
The time when an event is processed by the system. While simpler to implement, it can lead to non-deterministic results.

### Watermarks

Watermarks are timestamps that indicate the completeness of data up to a certain point in event time. They help the system determine when to emit results and handle late-arriving data.

```sql
CREATE TABLE sensor_data (
    sensor_id INT,
    reading DOUBLE,
    timestamp TIMESTAMP,
    WATERMARK FOR timestamp AS (timestamp - INTERVAL '5 seconds')
) WITH (
    connector = 'kafka',
    topic = 'sensors'
);
```

## Streaming SQL Capabilities

### 1. Time-Based Windows

Laminar supports three types of time windows for grouping streaming data:

![Window Types Comparison](/img/streaming/window-comparison.svg)

- **Tumbling Windows**: Fixed-size, non-overlapping windows
- **Hopping Windows**: Fixed-size, overlapping windows  
- **Session Windows**: Dynamic windows based on activity gaps

```sql
-- Tumbling window: 1-minute aggregations
SELECT 
    COUNT(*) as event_count,
    TUMBLE(INTERVAL '1 minute') as window
FROM events
GROUP BY window;

-- Hopping window: 5-minute windows, sliding every 1 minute
SELECT 
    AVG(value) as avg_value,
    HOP(INTERVAL '1 minute', INTERVAL '5 minutes') as window
FROM metrics
GROUP BY window;

-- Session window: New session after 30 seconds of inactivity
SELECT 
    user_id,
    COUNT(*) as actions,
    SESSION(INTERVAL '30 seconds') as session
FROM user_events
GROUP BY user_id, session;
```

### 2. Continuous Queries

Streaming queries run continuously, producing results as new data arrives:

```sql
-- Real-time moving average
SELECT 
    symbol,
    AVG(price) OVER (
        PARTITION BY symbol 
        ORDER BY trade_time 
        RANGE INTERVAL '5 minutes' PRECEDING
    ) as moving_avg,
    price as current_price
FROM trades;
```

### 3. Stream-Stream Joins

Join multiple data streams within time windows:

```sql
-- Join orders with payments within 10 minutes
SELECT 
    o.order_id,
    o.amount,
    p.payment_method
FROM orders o
JOIN payments p
    ON o.order_id = p.order_id
    AND o.order_time BETWEEN p.payment_time - INTERVAL '10 minutes' 
                        AND p.payment_time + INTERVAL '10 minutes';
```

### 4. Change Data Capture (CDC)

Process database change events in real-time:

```sql
CREATE TABLE users_cdc (
    user_id BIGINT,
    name VARCHAR,
    email VARCHAR,
    updated_at TIMESTAMP
) WITH (
    connector = 'kafka',
    format = 'debezium_json',
    topic = 'mysql.users'
);

-- Track user updates
SELECT 
    user_id,
    name,
    updated_at,
    LAG(name) OVER (PARTITION BY user_id ORDER BY updated_at) as previous_name
FROM users_cdc;
```

## Stream Processing Patterns

### Pattern Detection
Identify sequences of events that match specific patterns:

```sql
-- Detect potential fraud: multiple failed logins followed by success
SELECT 
    user_id,
    COUNT(*) FILTER (WHERE status = 'failed') as failed_attempts,
    MAX(timestamp) as last_attempt
FROM login_events
GROUP BY user_id, SESSION(INTERVAL '5 minutes')
HAVING COUNT(*) FILTER (WHERE status = 'failed') >= 3;
```

### Real-Time Aggregations
Compute metrics continuously as data flows:

```sql
-- Real-time sales dashboard
SELECT 
    product_category,
    COUNT(*) as transaction_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_transaction,
    TUMBLE(INTERVAL '1 hour') as hour
FROM sales
GROUP BY product_category, hour;
```

### Deduplication
Remove duplicate events within time windows:

```sql
-- Deduplicate events within 1-minute windows
WITH deduped AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY event_id 
            ORDER BY received_at
        ) as rn
    FROM raw_events
)
SELECT * FROM deduped WHERE rn = 1;
```

## State Management

Streaming queries maintain state to compute results over time. Laminar provides:

- **Automatic State Management**: State is managed transparently
- **Checkpointing**: Periodic state snapshots for fault tolerance
- **TTL Configuration**: Automatic cleanup of old state

```sql
-- Configure state TTL for updating streams
SET updating_ttl = INTERVAL '1 hour';
```

## Delivery Guarantees

Laminar supports different delivery semantics:

- **At-least-once**: Messages may be processed multiple times
- **Exactly-once**: Each message is processed exactly once (with transactional sources/sinks)

## Best Practices

1. **Use Event Time**: Prefer event time over processing time for deterministic results
2. **Set Appropriate Watermarks**: Balance between latency and completeness
3. **Manage State Size**: Use windowing and TTLs to bound state growth
4. **Optimize Joins**: Use time-bounded joins to limit state retention
5. **Monitor Memory Usage**: Track state size and memory consumption

## Next Steps

- Learn about [Window Functions](./window-functions.md) for time-based aggregations
- Understand [Watermarks](./watermarks.md) for handling out-of-order data
- Explore [Streaming Joins](./streaming-joins.md) for correlating multiple streams
- Read about [CDC Processing](./cdc-processing.md) for database synchronization

## Example: Real-Time Analytics Pipeline

![Streaming Pipeline](/img/streaming/streaming-pipeline.svg)

```sql
-- Source: Raw clickstream data
CREATE TABLE clickstream (
    user_id BIGINT,
    session_id VARCHAR,
    page_url VARCHAR,
    timestamp TIMESTAMP,
    WATERMARK FOR timestamp AS (timestamp - INTERVAL '10 seconds')
) WITH (
    connector = 'kafka',
    topic = 'clickstream',
    format = 'json'
);

-- Processing: Session analysis
CREATE VIEW user_sessions AS
SELECT 
    user_id,
    session_id,
    COUNT(*) as page_views,
    COUNT(DISTINCT page_url) as unique_pages,
    MIN(timestamp) as session_start,
    MAX(timestamp) as session_end,
    SESSION(INTERVAL '30 minutes') as session_window
FROM clickstream
GROUP BY user_id, session_id, session_window;

-- Sink: Materialized results
CREATE TABLE session_metrics (
    user_id BIGINT,
    session_duration_seconds BIGINT,
    page_views INT,
    unique_pages INT
) WITH (
    connector = 'postgres',
    table = 'session_metrics'
);

INSERT INTO session_metrics
SELECT 
    user_id,
    EXTRACT(EPOCH FROM (session_end - session_start)) as session_duration_seconds,
    page_views,
    unique_pages
FROM user_sessions;
```

This pipeline continuously processes clickstream data, analyzes user sessions in real-time, and stores aggregated metrics for dashboarding and further analysis.