---
title: DATE_TRUNC
description: Truncates a timestamp to a specified precision level
---

# DATE_TRUNC

## Description
The `DATE_TRUNC` function truncates a timestamp or date to a specified precision, setting all smaller units to their minimum values (usually zero or one). This is extremely useful for grouping timestamps by time periods like hour, day, month, or year. Unlike rounding, truncation always rounds down to the beginning of the specified period.

## Syntax
```sql
DATE_TRUNC(precision, timestamp_expression)
```

### Parameters
- **precision** (string): The unit to truncate to. Valid values include:
  - `'microseconds'`, `'milliseconds'`, `'second'`, `'minute'`, `'hour'`
  - `'day'`, `'week'`, `'month'`, `'quarter'`, `'year'`
  - `'decade'`, `'century'`, `'millennium'`
- **timestamp_expression** (timestamp): The timestamp or date to truncate

### Return Value
- **Type**: TIMESTAMP
- **Description**: The input timestamp truncated to the specified precision, with all smaller units set to their minimum values

## Example

### Sample Data
Consider the following `user_activity` table tracking user actions:

| activity_id | user_id | action      | timestamp                   |
|-------------|---------|-------------|-----------------------------|
| 1           | 101     | login       | 2024-03-15 09:23:45.123456 |
| 2           | 102     | purchase    | 2024-03-15 14:37:22.789012 |
| 3           | 103     | view_page   | 2024-03-16 11:15:33.456789 |
| 4           | 101     | logout      | 2024-03-16 16:45:10.234567 |
| 5           | 104     | signup      | 2024-03-17 08:52:18.901234 |

### Query
```sql
SELECT 
    activity_id,
    user_id,
    action,
    timestamp AS original_timestamp,
    DATE_TRUNC('hour', timestamp) AS truncated_hour,
    DATE_TRUNC('day', timestamp) AS truncated_day,
    DATE_TRUNC('month', timestamp) AS truncated_month,
    DATE_TRUNC('quarter', timestamp) AS truncated_quarter
FROM user_activity
ORDER BY activity_id;
```

### Result
| activity_id | user_id | action    | original_timestamp          | truncated_hour         | truncated_day          | truncated_month        | truncated_quarter     |
|-------------|---------|-----------|----------------------------|------------------------|------------------------|------------------------|------------------------|
| 1           | 101     | login     | 2024-03-15 09:23:45.123456| 2024-03-15 09:00:00   | 2024-03-15 00:00:00   | 2024-03-01 00:00:00   | 2024-01-01 00:00:00   |
| 2           | 102     | purchase  | 2024-03-15 14:37:22.789012| 2024-03-15 14:00:00   | 2024-03-15 00:00:00   | 2024-03-01 00:00:00   | 2024-01-01 00:00:00   |
| 3           | 103     | view_page | 2024-03-16 11:15:33.456789| 2024-03-16 11:00:00   | 2024-03-16 00:00:00   | 2024-03-01 00:00:00   | 2024-01-01 00:00:00   |
| 4           | 101     | logout    | 2024-03-16 16:45:10.234567| 2024-03-16 16:00:00   | 2024-03-16 00:00:00   | 2024-03-01 00:00:00   | 2024-01-01 00:00:00   |
| 5           | 104     | signup    | 2024-03-17 08:52:18.901234| 2024-03-17 08:00:00   | 2024-03-17 00:00:00   | 2024-03-01 00:00:00   | 2024-01-01 00:00:00   |

## Common Use Cases

### 1. Time-Based Aggregations
```sql
-- Count events per hour
SELECT 
    DATE_TRUNC('hour', created_at) AS hour,
    COUNT(*) AS event_count,
    AVG(response_time) AS avg_response_time
FROM events
GROUP BY DATE_TRUNC('hour', created_at)
ORDER BY hour;
```

### 2. Daily/Monthly Reports
```sql
-- Monthly sales summary
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(*) AS total_orders,
    SUM(order_total) AS revenue
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

### 3. Week-over-Week Comparisons
```sql
-- Compare this week vs last week
WITH weekly_data AS (
    SELECT 
        DATE_TRUNC('week', transaction_date) AS week,
        SUM(amount) AS weekly_total
    FROM transactions
    WHERE transaction_date >= CURRENT_DATE - INTERVAL '14 days'
    GROUP BY DATE_TRUNC('week', transaction_date)
)
SELECT 
    week,
    weekly_total,
    LAG(weekly_total) OVER (ORDER BY week) AS previous_week,
    weekly_total - LAG(weekly_total) OVER (ORDER BY week) AS week_over_week_change
FROM weekly_data;
```

### 4. Session Analysis
```sql
-- Group user activities into hourly sessions
SELECT 
    user_id,
    DATE_TRUNC('hour', activity_time) AS session_hour,
    COUNT(*) AS actions_in_hour,
    MIN(activity_time) AS session_start,
    MAX(activity_time) AS session_end
FROM user_activities
GROUP BY user_id, DATE_TRUNC('hour', activity_time);
```

### 5. Time Bucket Analysis
```sql
-- Analyze patterns by different time granularities
SELECT 
    DATE_TRUNC('day', created_at) AS day,
    DATE_TRUNC('hour', created_at) AS hour,
    COUNT(*) AS count,
    AVG(value) AS avg_value
FROM metrics
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY 
    DATE_TRUNC('day', created_at),
    DATE_TRUNC('hour', created_at)
ORDER BY hour;
```

## Precision Examples

```sql
SELECT 
    '2024-03-15 14:37:22.789012'::timestamp AS original,
    DATE_TRUNC('second', '2024-03-15 14:37:22.789012'::timestamp) AS second,
    -- Result: 2024-03-15 14:37:22
    DATE_TRUNC('minute', '2024-03-15 14:37:22.789012'::timestamp) AS minute,
    -- Result: 2024-03-15 14:37:00
    DATE_TRUNC('hour', '2024-03-15 14:37:22.789012'::timestamp) AS hour,
    -- Result: 2024-03-15 14:00:00
    DATE_TRUNC('day', '2024-03-15 14:37:22.789012'::timestamp) AS day,
    -- Result: 2024-03-15 00:00:00
    DATE_TRUNC('week', '2024-03-15 14:37:22.789012'::timestamp) AS week,
    -- Result: 2024-03-11 00:00:00 (Monday of that week)
    DATE_TRUNC('month', '2024-03-15 14:37:22.789012'::timestamp) AS month,
    -- Result: 2024-03-01 00:00:00
    DATE_TRUNC('quarter', '2024-03-15 14:37:22.789012'::timestamp) AS quarter,
    -- Result: 2024-01-01 00:00:00
    DATE_TRUNC('year', '2024-03-15 14:37:22.789012'::timestamp) AS year;
    -- Result: 2024-01-01 00:00:00
```

## Related Functions
- [DATE_BIN](./date_bin.md) - Bins timestamps into regular intervals
- [EXTRACT](./extract.md) - Extracts date/time components
- [DATE_PART](./date_part.md) - Similar to EXTRACT
- [TO_CHAR](./to_char.md) - Formats timestamps as strings
- [CURRENT_DATE](./current_date.md) - Returns current date