---
title: APPROX_DISTINCT
description: Returns an approximate count of distinct values using HyperLogLog algorithm
---

# APPROX_DISTINCT

## Description
The `APPROX_DISTINCT` function returns an approximate count of distinct values in a column using the HyperLogLog algorithm. This function is significantly faster and uses less memory than exact COUNT(DISTINCT) for large datasets, making it ideal for big data analytics where an exact count is not required. The typical error rate is around 2-3%.

## Syntax
```sql
APPROX_DISTINCT(expression)
```

### Parameters
- **expression**: A column or expression of any data type

### Return Value
- **Type**: BIGINT
- **Description**: An approximate count of distinct values. The error rate is typically ±2-3% with 97% confidence.

## Example

### Sample Data
Consider the following `web_analytics` table tracking website visits:

| visit_id | user_id | page_url           | session_id | country | timestamp           |
|----------|---------|-------------------|------------|---------|---------------------|
| 1        | U001    | /home             | S001       | USA     | 2024-03-01 10:00:00 |
| 2        | U002    | /products         | S002       | UK      | 2024-03-01 10:01:00 |
| 3        | U001    | /products         | S001       | USA     | 2024-03-01 10:02:00 |
| 4        | U003    | /home             | S003       | Canada  | 2024-03-01 10:03:00 |
| 5        | U002    | /checkout         | S002       | UK      | 2024-03-01 10:04:00 |
| ... millions more rows ... |

### Query
```sql
-- Compare approximate vs exact counts
SELECT 
    APPROX_DISTINCT(user_id) AS approx_unique_users,
    COUNT(DISTINCT user_id) AS exact_unique_users,
    ABS(APPROX_DISTINCT(user_id) - COUNT(DISTINCT user_id)) AS difference,
    ROUND(ABS(APPROX_DISTINCT(user_id) - COUNT(DISTINCT user_id)) * 100.0 / 
          COUNT(DISTINCT user_id), 2) AS error_percentage,
    APPROX_DISTINCT(session_id) AS approx_unique_sessions,
    APPROX_DISTINCT(page_url) AS approx_unique_pages
FROM web_analytics
WHERE timestamp >= CURRENT_DATE - INTERVAL '7 days';
```

### Result
| approx_unique_users | exact_unique_users | difference | error_percentage | approx_unique_sessions | approx_unique_pages |
|--------------------|-------------------|------------|------------------|----------------------|-------------------|
| 98547              | 100234            | 1687       | 1.68             | 245890               | 1523              |

## Common Use Cases

### 1. Real-Time Analytics Dashboard
```sql
-- Fast cardinality estimation for dashboard metrics
SELECT 
    DATE_TRUNC('hour', timestamp) AS hour,
    APPROX_DISTINCT(user_id) AS unique_visitors,
    APPROX_DISTINCT(session_id) AS unique_sessions,
    APPROX_DISTINCT(page_url) AS unique_pages_viewed,
    COUNT(*) AS total_page_views,
    COUNT(*) / NULLIF(APPROX_DISTINCT(session_id), 0) AS avg_pages_per_session
FROM web_analytics
WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY DATE_TRUNC('hour', timestamp)
ORDER BY hour DESC;
```

### 2. Data Quality Assessment
```sql
-- Quick cardinality checks for data profiling
SELECT 
    'customer_id' AS column_name,
    APPROX_DISTINCT(customer_id) AS approx_cardinality,
    COUNT(*) AS total_rows,
    APPROX_DISTINCT(customer_id) * 100.0 / COUNT(*) AS uniqueness_percentage
FROM transactions
UNION ALL
SELECT 
    'product_id',
    APPROX_DISTINCT(product_id),
    COUNT(*),
    APPROX_DISTINCT(product_id) * 100.0 / COUNT(*)
FROM transactions
UNION ALL
SELECT 
    'transaction_id',
    APPROX_DISTINCT(transaction_id),
    COUNT(*),
    APPROX_DISTINCT(transaction_id) * 100.0 / COUNT(*)
FROM transactions
ORDER BY uniqueness_percentage DESC;
```

### 3. Funnel Analysis
```sql
-- User progression through conversion funnel
WITH funnel_stages AS (
    SELECT 
        user_id,
        MAX(CASE WHEN page_url = '/home' THEN 1 ELSE 0 END) AS visited_home,
        MAX(CASE WHEN page_url = '/products' THEN 1 ELSE 0 END) AS viewed_products,
        MAX(CASE WHEN page_url = '/cart' THEN 1 ELSE 0 END) AS added_to_cart,
        MAX(CASE WHEN page_url = '/checkout' THEN 1 ELSE 0 END) AS reached_checkout,
        MAX(CASE WHEN page_url = '/confirmation' THEN 1 ELSE 0 END) AS completed_purchase
    FROM web_analytics
    WHERE timestamp >= CURRENT_DATE - INTERVAL '7 days'
    GROUP BY user_id
)
SELECT 
    'Home Page' AS stage,
    APPROX_DISTINCT(CASE WHEN visited_home = 1 THEN user_id END) AS unique_users
FROM funnel_stages
UNION ALL
SELECT 
    'Product View',
    APPROX_DISTINCT(CASE WHEN viewed_products = 1 THEN user_id END)
FROM funnel_stages
UNION ALL
SELECT 
    'Add to Cart',
    APPROX_DISTINCT(CASE WHEN added_to_cart = 1 THEN user_id END)
FROM funnel_stages
UNION ALL
SELECT 
    'Checkout',
    APPROX_DISTINCT(CASE WHEN reached_checkout = 1 THEN user_id END)
FROM funnel_stages
UNION ALL
SELECT 
    'Purchase',
    APPROX_DISTINCT(CASE WHEN completed_purchase = 1 THEN user_id END)
FROM funnel_stages;
```

### 4. Segmentation Analysis
```sql
-- Fast user segmentation with approximate counts
SELECT 
    country,
    device_type,
    APPROX_DISTINCT(user_id) AS unique_users,
    APPROX_DISTINCT(session_id) AS unique_sessions,
    COUNT(*) AS total_events,
    COUNT(*) / NULLIF(APPROX_DISTINCT(user_id), 0) AS avg_events_per_user
FROM user_events
WHERE event_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY country, device_type
HAVING APPROX_DISTINCT(user_id) > 100  -- Focus on significant segments
ORDER BY unique_users DESC
LIMIT 20;
```

### 5. Performance Comparison
```sql
-- Benchmark approximate vs exact for decision making
WITH timing_comparison AS (
    SELECT 
        -- Measure execution time difference
        CURRENT_TIMESTAMP AS start_time,
        COUNT(DISTINCT user_id) AS exact_count,
        CURRENT_TIMESTAMP AS exact_end_time,
        APPROX_DISTINCT(user_id) AS approx_count,
        CURRENT_TIMESTAMP AS approx_end_time
    FROM large_events_table
)
SELECT 
    exact_count,
    approx_count,
    ABS(exact_count - approx_count) AS difference,
    ROUND((exact_count - approx_count) * 100.0 / exact_count, 2) AS error_rate_pct,
    EXTRACT(EPOCH FROM (exact_end_time - start_time)) AS exact_time_seconds,
    EXTRACT(EPOCH FROM (approx_end_time - exact_end_time)) AS approx_time_seconds
FROM timing_comparison;
```

## Accuracy Considerations

```sql
-- Understanding accuracy at different scales
WITH accuracy_test AS (
    SELECT 
        cardinality_range,
        exact_distinct,
        approx_distinct,
        ABS(exact_distinct - approx_distinct) AS absolute_error,
        ROUND(ABS(exact_distinct - approx_distinct) * 100.0 / exact_distinct, 2) AS relative_error_pct
    FROM (
        SELECT 
            CASE 
                WHEN COUNT(DISTINCT user_id) < 1000 THEN '< 1K'
                WHEN COUNT(DISTINCT user_id) < 10000 THEN '1K-10K'
                WHEN COUNT(DISTINCT user_id) < 100000 THEN '10K-100K'
                WHEN COUNT(DISTINCT user_id) < 1000000 THEN '100K-1M'
                ELSE '> 1M'
            END AS cardinality_range,
            COUNT(DISTINCT user_id) AS exact_distinct,
            APPROX_DISTINCT(user_id) AS approx_distinct
        FROM events
        GROUP BY DATE_TRUNC('day', event_time)
    ) daily_stats
)
SELECT 
    cardinality_range,
    AVG(relative_error_pct) AS avg_error_pct,
    MAX(relative_error_pct) AS max_error_pct,
    MIN(relative_error_pct) AS min_error_pct
FROM accuracy_test
GROUP BY cardinality_range
ORDER BY cardinality_range;
```

## Combining with Other Aggregates

```sql
-- Use with other aggregate functions
SELECT 
    product_category,
    COUNT(*) AS total_sales,
    APPROX_DISTINCT(customer_id) AS unique_customers,
    APPROX_DISTINCT(product_id) AS unique_products,
    SUM(sale_amount) AS total_revenue,
    SUM(sale_amount) / NULLIF(APPROX_DISTINCT(customer_id), 0) AS revenue_per_customer,
    COUNT(*) / NULLIF(APPROX_DISTINCT(customer_id), 0) AS orders_per_customer
FROM sales
WHERE sale_date >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY product_category
ORDER BY total_revenue DESC;
```

## Memory Efficiency Example

```sql
-- Demonstrate memory efficiency for large-scale distinct counts
-- This would be memory-intensive with exact COUNT(DISTINCT)
SELECT 
    DATE_TRUNC('day', event_time) AS day,
    APPROX_DISTINCT(user_id) AS daily_users,
    APPROX_DISTINCT(ip_address) AS unique_ips,
    APPROX_DISTINCT(user_agent) AS unique_user_agents,
    APPROX_DISTINCT(CONCAT(user_id, '::', session_id)) AS unique_user_sessions
FROM billions_of_events
WHERE event_time >= CURRENT_DATE - INTERVAL '365 days'
GROUP BY DATE_TRUNC('day', event_time);
```

## Related Functions
- [COUNT](./count.md) - Exact count of rows
- [COUNT(DISTINCT)](./count.md) - Exact count of distinct values
- [APPROX_PERCENTILE](./approx_percentile.md) - Approximate percentile calculation
- [APPROX_MEDIAN](./approx_median.md) - Approximate median calculation
- [HLL_UNION](./hll_union.md) - Combine HyperLogLog sketches