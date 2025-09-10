---
title: SESSION Window
description: Dynamic gap-based windows for user activity and session analysis
---

# SESSION Window Function

## Description

The `SESSION` function creates dynamic, variable-size windows based on gaps of inactivity in the event stream. Unlike fixed-size windows (TUMBLE, HOP), session windows grow with activity and close after a specified period of inactivity. This is ideal for analyzing user sessions, transaction sequences, and any activity-based patterns.

## Syntax

```sql
SESSION(gap_duration)
```

### Parameters
- **gap_duration** (INTERVAL): The maximum time gap between events before a new session starts

### Return Value
- **Type**: Window descriptor with `start` and `end` timestamps
- **Description**: A dynamic window that expands with activity and closes after the gap duration

## How Session Windows Work

Session windows group events that are temporally close, creating a new window when the gap exceeds the threshold:

![Session Windows](/img/streaming/session-window.svg)

Each session window:
- Starts with the first event
- Expands as new events arrive within the gap duration
- Closes when no event arrives for the gap duration

## Example

### Sample Streaming Data

```sql
CREATE TABLE user_activities (
    user_id BIGINT,
    session_id VARCHAR,
    page_url VARCHAR,
    action_type VARCHAR,
    duration_seconds INT,
    activity_time TIMESTAMP,
    WATERMARK FOR activity_time AS (activity_time - INTERVAL '1 minute')
) WITH (
    connector = 'kafka',
    topic = 'user_clickstream',
    format = 'json'
);
```

### Basic Session Window Query

```sql
-- User sessions with 30-minute inactivity timeout
SELECT 
    user_id,
    COUNT(*) as page_views,
    COUNT(DISTINCT page_url) as unique_pages,
    SUM(duration_seconds) as total_duration,
    MIN(activity_time) as session_start,
    MAX(activity_time) as session_end,
    SESSION(INTERVAL '30 minutes') as session_window
FROM user_activities
GROUP BY user_id, session_window;
```

### Result Stream (Continuous Output)
| user_id | page_views | unique_pages | total_duration | session_start       | session_end         |
|---------|------------|--------------|----------------|-------------------|-------------------|
| 1001    | 15         | 8            | 425            | 2024-03-15 10:00:00 | 2024-03-15 10:28:45 |
| 1002    | 7          | 5            | 180            | 2024-03-15 10:05:30 | 2024-03-15 10:15:20 |
| 1001    | 23         | 12           | 680            | 2024-03-15 11:10:00 | 2024-03-15 11:45:30 |
| 1003    | 42         | 18           | 1250           | 2024-03-15 10:00:00 | 2024-03-15 10:58:00 |

## Common Use Cases

### 1. User Session Analytics

```sql
-- Comprehensive user session analysis
WITH user_sessions AS (
    SELECT 
        user_id,
        COUNT(*) as page_views,
        COUNT(DISTINCT page_url) as unique_pages,
        SUM(duration_seconds) as total_duration_seconds,
        MIN(activity_time) as session_start,
        MAX(activity_time) as session_end,
        SESSION(INTERVAL '30 minutes') as session
    FROM user_activities
    GROUP BY user_id, session
)
SELECT 
    user_id,
    page_views,
    unique_pages,
    total_duration_seconds,
    EXTRACT(EPOCH FROM (session_end - session_start)) as session_length_seconds,
    CASE 
        WHEN page_views = 1 THEN 'BOUNCE'
        WHEN page_views <= 3 THEN 'SHORT'
        WHEN page_views <= 10 THEN 'MEDIUM'
        ELSE 'LONG'
    END as session_type,
    CASE 
        WHEN total_duration_seconds < 30 THEN 'VERY_LOW'
        WHEN total_duration_seconds < 300 THEN 'LOW'
        WHEN total_duration_seconds < 900 THEN 'MEDIUM'
        ELSE 'HIGH'
    END as engagement_level,
    session_start,
    session_end
FROM user_sessions;
```

### 2. Transaction Sequence Analysis

```sql
-- Group related transactions into sessions
CREATE TABLE transactions (
    user_id BIGINT,
    transaction_id VARCHAR,
    amount DECIMAL(10,2),
    merchant VARCHAR,
    category VARCHAR,
    transaction_time TIMESTAMP,
    WATERMARK FOR transaction_time AS (transaction_time - INTERVAL '30 seconds')
) WITH (
    connector = 'kafka',
    topic = 'transactions'
);

-- Shopping sessions with 15-minute gaps
SELECT 
    user_id,
    COUNT(*) as transaction_count,
    SUM(amount) as session_total,
    AVG(amount) as avg_transaction,
    COUNT(DISTINCT merchant) as merchants_visited,
    COUNT(DISTINCT category) as categories_purchased,
    ARRAY_AGG(merchant ORDER BY transaction_time) as merchant_sequence,
    SESSION(INTERVAL '15 minutes') as shopping_session
FROM transactions
GROUP BY user_id, shopping_session
HAVING COUNT(*) >= 2;  -- Multi-transaction sessions only
```

### 3. IoT Device Activity Monitoring

```sql
-- Device online/offline sessions
CREATE TABLE device_heartbeats (
    device_id VARCHAR,
    status VARCHAR,
    cpu_usage DOUBLE,
    memory_usage DOUBLE,
    temperature DOUBLE,
    heartbeat_time TIMESTAMP,
    WATERMARK FOR heartbeat_time AS (heartbeat_time - INTERVAL '10 seconds')
) WITH (
    connector = 'mqtt',
    topic = 'devices/+/heartbeat'
);

-- Device sessions with 5-minute timeout
SELECT 
    device_id,
    COUNT(*) as heartbeat_count,
    AVG(cpu_usage) as avg_cpu,
    AVG(memory_usage) as avg_memory,
    MAX(temperature) as max_temperature,
    MIN(heartbeat_time) as online_since,
    MAX(heartbeat_time) as last_seen,
    SESSION(INTERVAL '5 minutes') as device_session
FROM device_heartbeats
WHERE status = 'ACTIVE'
GROUP BY device_id, device_session;
```

### 4. Customer Support Chat Sessions

```sql
-- Support conversation sessions
CREATE TABLE support_messages (
    ticket_id VARCHAR,
    user_id BIGINT,
    agent_id VARCHAR,
    message_type VARCHAR,  -- 'USER' or 'AGENT'
    message TEXT,
    sentiment_score DOUBLE,
    message_time TIMESTAMP,
    WATERMARK FOR message_time AS (message_time - INTERVAL '20 seconds')
) WITH (
    connector = 'kafka',
    topic = 'support_chat'
);

-- Chat sessions with 10-minute inactivity
WITH chat_sessions AS (
    SELECT 
        ticket_id,
        user_id,
        COUNT(*) as total_messages,
        COUNT(*) FILTER (WHERE message_type = 'USER') as user_messages,
        COUNT(*) FILTER (WHERE message_type = 'AGENT') as agent_messages,
        AVG(sentiment_score) as avg_sentiment,
        MIN(message_time) as conversation_start,
        MAX(message_time) as conversation_end,
        SESSION(INTERVAL '10 minutes') as chat_session
    FROM support_messages
    GROUP BY ticket_id, user_id, chat_session
)
SELECT 
    ticket_id,
    user_id,
    total_messages,
    user_messages,
    agent_messages,
    ROUND(agent_messages::DECIMAL / NULLIF(user_messages, 0), 2) as response_ratio,
    avg_sentiment,
    EXTRACT(EPOCH FROM (conversation_end - conversation_start)) / 60 as duration_minutes,
    CASE 
        WHEN avg_sentiment < -0.5 THEN 'VERY_NEGATIVE'
        WHEN avg_sentiment < 0 THEN 'NEGATIVE'
        WHEN avg_sentiment < 0.5 THEN 'NEUTRAL'
        ELSE 'POSITIVE'
    END as session_sentiment,
    conversation_start,
    conversation_end
FROM chat_sessions;
```

### 5. Gaming Session Analysis

```sql
-- Player gaming sessions
CREATE TABLE game_events (
    player_id BIGINT,
    event_type VARCHAR,
    score_change INT,
    level INT,
    coins_earned INT,
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '30 seconds')
) WITH (
    connector = 'kafka',
    topic = 'game_events'
);

-- Gaming sessions with 20-minute breaks
SELECT 
    player_id,
    COUNT(*) as total_events,
    COUNT(DISTINCT level) as levels_played,
    SUM(score_change) as session_score,
    SUM(coins_earned) as coins_earned,
    COUNT(*) FILTER (WHERE event_type = 'ACHIEVEMENT') as achievements,
    MIN(event_time) as session_start,
    MAX(event_time) as session_end,
    SESSION(INTERVAL '20 minutes') as gaming_session
FROM game_events
GROUP BY player_id, gaming_session;
```

## Advanced Patterns

### Session Metrics with User Journey

```sql
-- Track user journey through conversion funnel
WITH session_events AS (
    SELECT 
        user_id,
        page_url,
        CASE 
            WHEN page_url LIKE '%/product/%' THEN 'BROWSE'
            WHEN page_url LIKE '%/cart%' THEN 'CART'
            WHEN page_url LIKE '%/checkout%' THEN 'CHECKOUT'
            WHEN page_url LIKE '%/confirm%' THEN 'PURCHASE'
            ELSE 'OTHER'
        END as funnel_stage,
        activity_time,
        SESSION(INTERVAL '30 minutes') as session
    FROM user_activities
)
SELECT 
    user_id,
    COUNT(DISTINCT funnel_stage) as stages_reached,
    MAX(CASE WHEN funnel_stage = 'BROWSE' THEN 1 ELSE 0 END) as visited_product,
    MAX(CASE WHEN funnel_stage = 'CART' THEN 1 ELSE 0 END) as added_to_cart,
    MAX(CASE WHEN funnel_stage = 'CHECKOUT' THEN 1 ELSE 0 END) as started_checkout,
    MAX(CASE WHEN funnel_stage = 'PURCHASE' THEN 1 ELSE 0 END) as completed_purchase,
    MIN(activity_time) as session_start,
    MAX(activity_time) as session_end,
    session
FROM session_events
GROUP BY user_id, session;
```

### Multi-Device Session Tracking

```sql
-- Track user sessions across devices
CREATE TABLE cross_device_events (
    user_id BIGINT,
    device_id VARCHAR,
    device_type VARCHAR,
    event_type VARCHAR,
    event_time TIMESTAMP,
    WATERMARK FOR event_time AS (event_time - INTERVAL '1 minute')
) WITH (
    connector = 'kafka',
    topic = 'cross_device_tracking'
);

-- Unified sessions across devices (1-hour gap)
SELECT 
    user_id,
    COUNT(DISTINCT device_id) as devices_used,
    ARRAY_AGG(DISTINCT device_type) as device_types,
    COUNT(*) as total_events,
    MIN(event_time) as session_start,
    MAX(event_time) as session_end,
    SESSION(INTERVAL '1 hour') as unified_session
FROM cross_device_events
GROUP BY user_id, unified_session
HAVING COUNT(DISTINCT device_id) > 1;  -- Multi-device sessions
```

### Session-Based Anomaly Detection

```sql
-- Detect unusual session patterns
WITH normal_sessions AS (
    -- Historical baseline
    SELECT 
        user_id,
        AVG(COUNT(*)) OVER (PARTITION BY user_id) as avg_events,
        STDDEV(COUNT(*)) OVER (PARTITION BY user_id) as stddev_events,
        SESSION(INTERVAL '30 minutes') as session
    FROM user_activities
    WHERE activity_time < CURRENT_TIMESTAMP - INTERVAL '1 day'
    GROUP BY user_id, session
),
current_sessions AS (
    -- Current sessions
    SELECT 
        user_id,
        COUNT(*) as event_count,
        COUNT(DISTINCT page_url) as unique_pages,
        SESSION(INTERVAL '30 minutes') as session
    FROM user_activities
    WHERE activity_time >= CURRENT_TIMESTAMP - INTERVAL '1 day'
    GROUP BY user_id, session
)
SELECT 
    c.user_id,
    c.event_count,
    n.avg_events,
    n.stddev_events,
    (c.event_count - n.avg_events) / NULLIF(n.stddev_events, 0) as z_score,
    CASE 
        WHEN ABS((c.event_count - n.avg_events) / NULLIF(n.stddev_events, 0)) > 3 
        THEN 'ANOMALOUS'
        ELSE 'NORMAL'
    END as session_classification
FROM current_sessions c
LEFT JOIN normal_sessions n ON c.user_id = n.user_id;
```

## Session Window Characteristics

### Dynamic Window Boundaries

```sql
-- Sessions expand with activity
Events at: 10:00, 10:05, 10:08, 10:20, 10:22 (gap = 15 min)
Session 1: [10:00 - 10:23] (includes first 3 events + gap)
Session 2: [10:20 - 10:37] (includes last 2 events + gap)
```

### Handling Concurrent Sessions

```sql
-- Different session windows for different keys
SELECT 
    user_id,
    device_id,
    COUNT(*) as events,
    SESSION(INTERVAL '15 minutes') as session
FROM multi_device_events
GROUP BY user_id, device_id, session;
-- Each (user_id, device_id) pair has independent sessions
```

## Performance Considerations

1. **State Management**: 
   - Open sessions keep state until gap timeout
   - Memory usage proportional to active sessions

2. **Watermark Impact**:
   - Sessions close based on watermark + gap duration
   - Late events may create new sessions

3. **Gap Duration Selection**:
   - Short gaps: More sessions, quicker results
   - Long gaps: Fewer sessions, more memory usage

4. **Key Cardinality**:
   - High cardinality (many users) = more concurrent sessions
   - Consider state backend capacity

## Best Practices

1. **Choose Appropriate Gap Duration**:
   - Analyze typical user behavior patterns
   - Consider business requirements for session definition

2. **Handle Session Boundaries**:
   - Account for sessions spanning query boundaries
   - Use watermarks appropriately

3. **Optimize for Memory**:
   - Set reasonable gap durations
   - Consider session TTL for long-running queries

4. **Monitor Session Metrics**:
   - Track average session duration
   - Monitor number of concurrent sessions
   - Watch for unusually long sessions

5. **Late Data Strategy**:
   - Define clear policies for late-arriving events
   - Consider impact on session boundaries

## Comparison with Fixed Windows

| Feature | SESSION | TUMBLE | HOP |
|---------|---------|--------|-----|
| Window Size | Variable | Fixed | Fixed |
| Based On | Activity gaps | Time intervals | Sliding intervals |
| Use Case | User sessions | Periodic reports | Moving averages |
| Boundaries | Dynamic | Predetermined | Predetermined |
| Memory Usage | Variable | Predictable | Predictable |
| Event Assignment | Based on gaps | Based on time | Multiple windows |

## Related Functions
- [TUMBLE](./tumble.md) - Fixed-size non-overlapping windows
- [HOP](./hop.md) - Fixed-size overlapping windows
- [Watermarks](./watermarks.md) - Handling event time
- [Windowed Aggregations](./windowed-aggregations.md) - Aggregation patterns