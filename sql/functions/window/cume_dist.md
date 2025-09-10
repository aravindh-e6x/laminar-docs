---
title: CUME_DIST
description: Returns the cumulative distribution of values within a partition
---

# CUME_DIST

## Description
The `CUME_DIST` (cumulative distribution) window function calculates the relative position of a value within a partition, returning the percentage of rows that have values less than or equal to the current row's value. The result ranges from 0 to 1, where 1 indicates the maximum value. Unlike PERCENT_RANK, CUME_DIST includes the current row in its calculation, making it useful for understanding what proportion of the data falls at or below a certain threshold.

## Syntax
```sql
CUME_DIST() OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
)
```

### Parameters
- **PARTITION BY** (optional): Divides the result set into partitions. Cumulative distribution is calculated separately for each partition
- **ORDER BY** (required): Determines the order for cumulative distribution calculation

### Return Value
- **Type**: DOUBLE PRECISION
- **Description**: A value between 0 (exclusive) and 1 (inclusive) representing the cumulative distribution

## Example

### Sample Data
Consider the following `response_times` table:

| request_id | endpoint      | response_ms | timestamp           |
|------------|--------------|-------------|---------------------|
| 1          | /api/users   | 45          | 2024-03-15 10:00:00 |
| 2          | /api/users   | 52          | 2024-03-15 10:00:01 |
| 3          | /api/users   | 48          | 2024-03-15 10:00:02 |
| 4          | /api/users   | 52          | 2024-03-15 10:00:03 |
| 5          | /api/users   | 65          | 2024-03-15 10:00:04 |
| 6          | /api/orders  | 120         | 2024-03-15 10:00:00 |
| 7          | /api/orders  | 95          | 2024-03-15 10:00:01 |
| 8          | /api/orders  | 110         | 2024-03-15 10:00:02 |
| 9          | /api/orders  | 95          | 2024-03-15 10:00:03 |
| 10         | /api/orders  | 150         | 2024-03-15 10:00:04 |

### Query
```sql
SELECT 
    request_id,
    endpoint,
    response_ms,
    CUME_DIST() OVER (PARTITION BY endpoint ORDER BY response_ms) AS cume_dist,
    ROUND(CUME_DIST() OVER (PARTITION BY endpoint ORDER BY response_ms) * 100, 2) AS percentile,
    PERCENT_RANK() OVER (PARTITION BY endpoint ORDER BY response_ms) AS pct_rank_comparison,
    COUNT(*) OVER (PARTITION BY endpoint ORDER BY response_ms RANGE UNBOUNDED PRECEDING) AS cumulative_count
FROM response_times
ORDER BY endpoint, response_ms;
```

### Result
| request_id | endpoint     | response_ms | cume_dist | percentile | pct_rank_comparison | cumulative_count |
|------------|-------------|-------------|-----------|------------|-------------------|------------------|
| 1          | /api/users  | 45          | 0.20      | 20.00      | 0.00              | 1                |
| 3          | /api/users  | 48          | 0.40      | 40.00      | 0.25              | 2                |
| 2          | /api/users  | 52          | 0.80      | 80.00      | 0.50              | 4                |
| 4          | /api/users  | 52          | 0.80      | 80.00      | 0.50              | 4                |
| 5          | /api/users  | 65          | 1.00      | 100.00     | 1.00              | 5                |
| 7          | /api/orders | 95          | 0.40      | 40.00      | 0.00              | 2                |
| 9          | /api/orders | 95          | 0.40      | 40.00      | 0.00              | 2                |
| 8          | /api/orders | 110         | 0.60      | 60.00      | 0.50              | 3                |
| 6          | /api/orders | 120         | 0.80      | 80.00      | 0.75              | 4                |
| 10         | /api/orders | 150         | 1.00      | 100.00     | 1.00              | 5                |

## Common Use Cases

### 1. Performance SLA Monitoring
```sql
-- Calculate what percentage of requests meet SLA thresholds
WITH response_analysis AS (
    SELECT 
        endpoint,
        response_ms,
        CUME_DIST() OVER (PARTITION BY endpoint ORDER BY response_ms) AS cume_dist
    FROM response_times
    WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '1 hour'
)
SELECT 
    endpoint,
    MAX(CASE WHEN response_ms <= 100 THEN cume_dist ELSE 0 END) * 100 AS pct_under_100ms,
    MAX(CASE WHEN response_ms <= 200 THEN cume_dist ELSE 0 END) * 100 AS pct_under_200ms,
    MAX(CASE WHEN response_ms <= 500 THEN cume_dist ELSE 0 END) * 100 AS pct_under_500ms,
    MIN(CASE WHEN cume_dist >= 0.95 THEN response_ms END) AS p95_response_time,
    MIN(CASE WHEN cume_dist >= 0.99 THEN response_ms END) AS p99_response_time
FROM response_analysis
GROUP BY endpoint;
```

### 2. Statistical Percentile Calculation
```sql
-- Find specific percentiles using CUME_DIST
WITH score_distribution AS (
    SELECT 
        score,
        CUME_DIST() OVER (ORDER BY score) AS cume_dist,
        LAG(CUME_DIST()) OVER (ORDER BY score) AS prev_cume_dist
    FROM test_scores
)
SELECT 
    MIN(CASE WHEN cume_dist >= 0.25 AND (prev_cume_dist < 0.25 OR prev_cume_dist IS NULL) THEN score END) AS p25,
    MIN(CASE WHEN cume_dist >= 0.50 AND (prev_cume_dist < 0.50 OR prev_cume_dist IS NULL) THEN score END) AS p50_median,
    MIN(CASE WHEN cume_dist >= 0.75 AND (prev_cume_dist < 0.75 OR prev_cume_dist IS NULL) THEN score END) AS p75,
    MIN(CASE WHEN cume_dist >= 0.90 AND (prev_cume_dist < 0.90 OR prev_cume_dist IS NULL) THEN score END) AS p90,
    MIN(CASE WHEN cume_dist >= 0.95 AND (prev_cume_dist < 0.95 OR prev_cume_dist IS NULL) THEN score END) AS p95,
    MIN(CASE WHEN cume_dist >= 0.99 AND (prev_cume_dist < 0.99 OR prev_cume_dist IS NULL) THEN score END) AS p99
FROM score_distribution;
```

### 3. Income Distribution Analysis
```sql
-- Analyze income inequality using cumulative distribution
WITH income_dist AS (
    SELECT 
        household_id,
        annual_income,
        CUME_DIST() OVER (ORDER BY annual_income) AS income_percentile,
        SUM(annual_income) OVER (ORDER BY annual_income) / SUM(annual_income) OVER () AS cumulative_income_share
    FROM household_income
)
SELECT 
    CASE 
        WHEN income_percentile <= 0.20 THEN 'Bottom 20%'
        WHEN income_percentile <= 0.40 THEN '20-40%'
        WHEN income_percentile <= 0.60 THEN '40-60%'
        WHEN income_percentile <= 0.80 THEN '60-80%'
        ELSE 'Top 20%'
    END AS income_quintile,
    COUNT(*) AS household_count,
    MIN(annual_income) AS min_income,
    MAX(annual_income) AS max_income,
    AVG(annual_income) AS avg_income,
    MAX(cumulative_income_share) * 100 AS cumulative_income_pct
FROM income_dist
GROUP BY 
    CASE 
        WHEN income_percentile <= 0.20 THEN 'Bottom 20%'
        WHEN income_percentile <= 0.40 THEN '20-40%'
        WHEN income_percentile <= 0.60 THEN '40-60%'
        WHEN income_percentile <= 0.80 THEN '60-80%'
        ELSE 'Top 20%'
    END
ORDER BY MIN(annual_income);
```

### 4. Quality Control Thresholds
```sql
-- Identify products falling below quality thresholds
WITH quality_metrics AS (
    SELECT 
        product_id,
        defect_rate,
        CUME_DIST() OVER (ORDER BY defect_rate) AS quality_percentile
    FROM product_quality_tests
    WHERE test_date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT 
    product_id,
    defect_rate,
    ROUND(quality_percentile * 100, 2) AS percentile,
    CASE 
        WHEN quality_percentile <= 0.10 THEN 'Excellent (Top 10%)'
        WHEN quality_percentile <= 0.25 THEN 'Good'
        WHEN quality_percentile <= 0.75 THEN 'Acceptable'
        WHEN quality_percentile <= 0.90 THEN 'Needs Improvement'
        ELSE 'Critical (Bottom 10%)'
    END AS quality_rating,
    CASE 
        WHEN quality_percentile > 0.90 THEN 'Immediate Action Required'
        WHEN quality_percentile > 0.75 THEN 'Monitor Closely'
        ELSE 'Within Standards'
    END AS action_required
FROM quality_metrics
ORDER BY defect_rate DESC;
```

### 5. Customer Lifetime Value Segmentation
```sql
-- Segment customers based on LTV distribution
WITH ltv_distribution AS (
    SELECT 
        customer_id,
        lifetime_value,
        CUME_DIST() OVER (ORDER BY lifetime_value) AS ltv_percentile,
        NTILE(10) OVER (ORDER BY lifetime_value) AS ltv_decile
    FROM customer_metrics
)
SELECT 
    customer_id,
    lifetime_value,
    ROUND(ltv_percentile * 100, 2) AS percentile,
    ltv_decile,
    CASE 
        WHEN ltv_percentile >= 0.95 THEN 'VIP'
        WHEN ltv_percentile >= 0.80 THEN 'High Value'
        WHEN ltv_percentile >= 0.50 THEN 'Mid Value'
        WHEN ltv_percentile >= 0.20 THEN 'Low Value'
        ELSE 'At Risk'
    END AS customer_segment
FROM ltv_distribution
WHERE ltv_percentile >= 0.90  -- Focus on top 10%
ORDER BY lifetime_value DESC;
```

## Formula Comparison

```sql
-- Understanding CUME_DIST calculation
WITH sample_data AS (
    SELECT value FROM (VALUES (10), (20), (20), (30), (40)) AS t(value)
)
SELECT 
    value,
    ROW_NUMBER() OVER (ORDER BY value) AS row_num,
    COUNT(*) OVER () AS total_rows,
    -- CUME_DIST = (number of rows <= current value) / total rows
    CUME_DIST() OVER (ORDER BY value) AS cume_dist,
    COUNT(*) OVER (ORDER BY value RANGE UNBOUNDED PRECEDING) * 1.0 / 
        COUNT(*) OVER () AS calculated_cume_dist,
    -- Compare with PERCENT_RANK
    PERCENT_RANK() OVER (ORDER BY value) AS percent_rank
FROM sample_data
ORDER BY value;
```

## Handling Ties

```sql
-- CUME_DIST behavior with tied values
WITH tied_scores AS (
    SELECT 
        player,
        score,
        RANK() OVER (ORDER BY score DESC) AS rank,
        CUME_DIST() OVER (ORDER BY score DESC) AS cume_dist_desc,
        CUME_DIST() OVER (ORDER BY score ASC) AS cume_dist_asc
    FROM (VALUES 
        ('Alice', 100),
        ('Bob', 95),
        ('Charlie', 95),
        ('David', 95),
        ('Eve', 90)
    ) AS t(player, score)
)
SELECT 
    player,
    score,
    rank,
    ROUND(cume_dist_desc * 100, 2) AS top_percentile,
    ROUND((1 - cume_dist_asc) * 100, 2) AS bottom_percentile,
    CASE 
        WHEN cume_dist_desc <= 0.10 THEN 'Top 10%'
        WHEN cume_dist_desc <= 0.25 THEN 'Top 25%'
        ELSE 'Other'
    END AS classification
FROM tied_scores
ORDER BY score DESC, player;
```

## Creating Empirical CDF

```sql
-- Create empirical cumulative distribution function
WITH cdf_data AS (
    SELECT DISTINCT
        value,
        CUME_DIST() OVER (ORDER BY value) AS cdf,
        COUNT(*) OVER (ORDER BY value RANGE UNBOUNDED PRECEDING) AS cumulative_count,
        COUNT(*) OVER () AS total_count
    FROM measurements
)
SELECT 
    value,
    cdf,
    cumulative_count || '/' || total_count AS fraction,
    ROUND(cdf * 100, 2) AS percentage,
    STRING_AGG('█', '') OVER (ORDER BY value ROWS UNBOUNDED PRECEDING) AS histogram
FROM cdf_data
ORDER BY value;
```

## Performance Considerations

```sql
-- Optimize CUME_DIST queries
CREATE INDEX idx_response_times ON response_times(endpoint, response_ms);

-- Efficient percentile monitoring
WITH RECURSIVE percentiles AS (
    SELECT 0.50 AS target_percentile
    UNION ALL
    SELECT 0.90
    UNION ALL
    SELECT 0.95
    UNION ALL
    SELECT 0.99
)
SELECT 
    p.target_percentile,
    MIN(r.response_ms) AS response_time_threshold
FROM response_times r
CROSS JOIN percentiles p
WHERE CUME_DIST() OVER (ORDER BY r.response_ms) >= p.target_percentile
GROUP BY p.target_percentile
ORDER BY p.target_percentile;
```

## Related Functions
- [PERCENT_RANK](./percent_rank.md) - Relative rank as percentage (excludes current row)
- [NTILE](./ntile.md) - Divide rows into equal buckets
- [RANK](./rank.md) - Ranking with gaps
- [DENSE_RANK](./dense_rank.md) - Ranking without gaps
- [ROW_NUMBER](./row_number.md) - Sequential row numbering