---
title: NTILE
description: Divides rows into a specified number of approximately equal groups
---

# NTILE

## Description
The `NTILE` window function divides an ordered result set into a specified number of approximately equal-sized groups (tiles or buckets) and assigns each row a bucket number from 1 to n. If the rows cannot be divided equally, larger buckets appear first. This function is commonly used for creating quartiles, deciles, percentiles, or any custom number of groups for statistical analysis and data segmentation.

## Syntax
```sql
NTILE(number_of_buckets) OVER (
    [PARTITION BY partition_expression, ...]
    [ORDER BY sort_expression [ASC|DESC], ...]
)
```

### Parameters
- **number_of_buckets** (required): Integer specifying the number of groups to create (must be positive)
- **PARTITION BY** (optional): Divides the result set into partitions. NTILE is applied separately to each partition
- **ORDER BY** (optional but recommended): Determines the order for assigning bucket numbers

### Return Value
- **Type**: BIGINT
- **Description**: Bucket number from 1 to number_of_buckets

## Example

### Sample Data
Consider the following `customer_orders` table:

| customer_id | total_spent | order_count | last_order_date |
|------------|-------------|-------------|-----------------|
| 1          | 5000        | 25          | 2024-03-15      |
| 2          | 1200        | 8           | 2024-03-10      |
| 3          | 3500        | 15          | 2024-03-14      |
| 4          | 800         | 5           | 2024-02-28      |
| 5          | 2000        | 12          | 2024-03-12      |
| 6          | 6500        | 30          | 2024-03-16      |
| 7          | 1500        | 10          | 2024-03-11      |
| 8          | 4200        | 20          | 2024-03-13      |
| 9          | 900         | 6           | 2024-03-05      |
| 10         | 2800        | 14          | 2024-03-09      |

### Query
```sql
SELECT 
    customer_id,
    total_spent,
    order_count,
    NTILE(4) OVER (ORDER BY total_spent DESC) AS spending_quartile,
    NTILE(5) OVER (ORDER BY total_spent DESC) AS spending_quintile,
    NTILE(10) OVER (ORDER BY total_spent DESC) AS spending_decile,
    NTILE(3) OVER (ORDER BY order_count DESC) AS activity_tertile
FROM customer_orders
ORDER BY total_spent DESC;
```

### Result
| customer_id | total_spent | order_count | spending_quartile | spending_quintile | spending_decile | activity_tertile |
|------------|-------------|-------------|------------------|------------------|-----------------|------------------|
| 6          | 6500        | 30          | 1                | 1                | 1               | 1                |
| 1          | 5000        | 25          | 1                | 1                | 2               | 1                |
| 8          | 4200        | 20          | 1                | 2                | 3               | 1                |
| 3          | 3500        | 15          | 2                | 2                | 4               | 2                |
| 10         | 2800        | 14          | 2                | 3                | 5               | 2                |
| 5          | 2000        | 12          | 2                | 3                | 6               | 2                |
| 7          | 1500        | 10          | 3                | 4                | 7               | 2                |
| 2          | 1200        | 8           | 3                | 4                | 8               | 3                |
| 9          | 900         | 6           | 4                | 5                | 9               | 3                |
| 4          | 800         | 5           | 4                | 5                | 10              | 3                |

## Common Use Cases

### 1. Creating Quartiles for Analysis
```sql
-- Divide data into quartiles (4 equal groups)
WITH quartile_analysis AS (
    SELECT 
        product_id,
        product_name,
        revenue,
        NTILE(4) OVER (ORDER BY revenue DESC) AS revenue_quartile
    FROM product_performance
)
SELECT 
    revenue_quartile,
    COUNT(*) AS product_count,
    MIN(revenue) AS min_revenue,
    AVG(revenue) AS avg_revenue,
    MAX(revenue) AS max_revenue,
    SUM(revenue) AS total_revenue,
    CASE revenue_quartile
        WHEN 1 THEN 'Top Performers (Q1)'
        WHEN 2 THEN 'Above Average (Q2)'
        WHEN 3 THEN 'Below Average (Q3)'
        WHEN 4 THEN 'Low Performers (Q4)'
    END AS quartile_label
FROM quartile_analysis
GROUP BY revenue_quartile
ORDER BY revenue_quartile;
```

### 2. Customer Segmentation by Deciles
```sql
-- Create customer value segments using deciles
WITH customer_deciles AS (
    SELECT 
        customer_id,
        customer_name,
        lifetime_value,
        NTILE(10) OVER (ORDER BY lifetime_value DESC) AS value_decile
    FROM customer_metrics
)
SELECT 
    customer_id,
    customer_name,
    lifetime_value,
    value_decile,
    CASE 
        WHEN value_decile = 1 THEN 'Top 10% - VIP'
        WHEN value_decile <= 2 THEN 'Top 20% - Premium'
        WHEN value_decile <= 3 THEN 'Top 30% - High Value'
        WHEN value_decile <= 5 THEN 'Middle Tier'
        WHEN value_decile <= 7 THEN 'Low-Mid Tier'
        ELSE 'Bottom 30% - At Risk'
    END AS customer_segment,
    CASE 
        WHEN value_decile <= 2 THEN 'Priority Support'
        WHEN value_decile <= 5 THEN 'Standard Support'
        ELSE 'Self-Service'
    END AS service_level
FROM customer_deciles
ORDER BY value_decile, lifetime_value DESC;
```

### 3. Performance Rating Distribution
```sql
-- Distribute employees into performance tiers
WITH performance_tiers AS (
    SELECT 
        employee_id,
        employee_name,
        department,
        performance_score,
        NTILE(5) OVER (ORDER BY performance_score DESC) AS company_tier,
        NTILE(5) OVER (PARTITION BY department ORDER BY performance_score DESC) AS dept_tier
    FROM employee_reviews
    WHERE review_year = 2024
)
SELECT 
    employee_id,
    employee_name,
    department,
    performance_score,
    company_tier,
    dept_tier,
    CASE company_tier
        WHEN 1 THEN 'Exceeds Expectations'
        WHEN 2 THEN 'Above Average'
        WHEN 3 THEN 'Meets Expectations'
        WHEN 4 THEN 'Below Average'
        WHEN 5 THEN 'Needs Improvement'
    END AS performance_rating
FROM performance_tiers
ORDER BY department, dept_tier, performance_score DESC;
```

### 4. A/B Test Group Assignment
```sql
-- Randomly assign users to test groups
WITH random_assignment AS (
    SELECT 
        user_id,
        email,
        signup_date,
        NTILE(3) OVER (ORDER BY RANDOM()) AS test_group
    FROM users
    WHERE status = 'active'
)
SELECT 
    user_id,
    email,
    CASE test_group
        WHEN 1 THEN 'Control Group'
        WHEN 2 THEN 'Test Variant A'
        WHEN 3 THEN 'Test Variant B'
    END AS group_assignment
FROM random_assignment
ORDER BY test_group, user_id;
```

### 5. Time-Based Cohort Analysis
```sql
-- Create monthly cohorts and analyze by quintiles
WITH cohort_data AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', signup_date) AS cohort_month,
        total_purchases,
        NTILE(5) OVER (
            PARTITION BY DATE_TRUNC('month', signup_date) 
            ORDER BY total_purchases DESC
        ) AS cohort_quintile
    FROM user_activity
)
SELECT 
    cohort_month,
    cohort_quintile,
    COUNT(*) AS user_count,
    AVG(total_purchases) AS avg_purchases,
    CASE cohort_quintile
        WHEN 1 THEN 'Top 20%'
        WHEN 2 THEN '20-40%'
        WHEN 3 THEN '40-60%'
        WHEN 4 THEN '60-80%'
        WHEN 5 THEN 'Bottom 20%'
    END AS quintile_label
FROM cohort_data
GROUP BY cohort_month, cohort_quintile
ORDER BY cohort_month, cohort_quintile;
```

## Handling Uneven Distribution

```sql
-- Demonstrate how NTILE handles rows that don't divide evenly
WITH sample_data AS (
    SELECT value FROM (VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9),(10)) AS t(value)
)
SELECT 
    value,
    NTILE(3) OVER (ORDER BY value) AS three_buckets,
    NTILE(4) OVER (ORDER BY value) AS four_buckets,
    NTILE(7) OVER (ORDER BY value) AS seven_buckets,
    -- Show which buckets get extra rows
    COUNT(*) OVER (PARTITION BY NTILE(3) OVER (ORDER BY value)) AS bucket_3_size,
    COUNT(*) OVER (PARTITION BY NTILE(4) OVER (ORDER BY value)) AS bucket_4_size
FROM sample_data
ORDER BY value;
-- Note: When 10 rows divided by 3, first bucket gets 4 rows, others get 3
```

## Creating Custom Percentiles

```sql
-- Use NTILE to approximate percentiles
WITH percentile_data AS (
    SELECT 
        value,
        NTILE(100) OVER (ORDER BY value) AS percentile
    FROM measurements
)
SELECT 
    MIN(CASE WHEN percentile >= 25 THEN value END) AS p25,
    MIN(CASE WHEN percentile >= 50 THEN value END) AS p50_median,
    MIN(CASE WHEN percentile >= 75 THEN value END) AS p75,
    MIN(CASE WHEN percentile >= 90 THEN value END) AS p90,
    MIN(CASE WHEN percentile >= 95 THEN value END) AS p95,
    MIN(CASE WHEN percentile >= 99 THEN value END) AS p99
FROM percentile_data;
```

## Bucket Size Calculation

```sql
-- Understanding bucket sizes with different row counts
WITH bucket_analysis AS (
    SELECT 
        COUNT(*) AS total_rows,
        n AS num_buckets,
        COUNT(*) / n AS base_size,
        COUNT(*) % n AS extra_rows
    FROM orders
    CROSS JOIN (VALUES (3), (4), (5), (10)) AS buckets(n)
    GROUP BY n
)
SELECT 
    total_rows,
    num_buckets,
    base_size,
    extra_rows,
    CASE 
        WHEN extra_rows > 0 
        THEN FORMAT('%s buckets with %s rows, %s buckets with %s rows',
                   extra_rows, base_size + 1,
                   num_buckets - extra_rows, base_size)
        ELSE FORMAT('All %s buckets with %s rows', num_buckets, base_size)
    END AS distribution
FROM bucket_analysis
ORDER BY num_buckets;
```

## Performance Optimization

```sql
-- Optimize NTILE with appropriate indexes
CREATE INDEX idx_customer_value ON customers(lifetime_value DESC);

-- Efficient segmentation query
WITH customer_segments AS (
    SELECT 
        customer_id,
        lifetime_value,
        last_purchase_date,
        NTILE(10) OVER (ORDER BY lifetime_value DESC) AS value_decile
    FROM customers
    WHERE status = 'active'
      AND last_purchase_date >= CURRENT_DATE - INTERVAL '1 year'
)
SELECT 
    value_decile,
    COUNT(*) AS customer_count,
    AVG(lifetime_value) AS avg_ltv
FROM customer_segments
GROUP BY value_decile
ORDER BY value_decile;
```

## Combining with Other Window Functions

```sql
-- Use NTILE with other ranking functions for detailed analysis
SELECT 
    student_name,
    test_score,
    NTILE(4) OVER (ORDER BY test_score DESC) AS quartile,
    NTILE(10) OVER (ORDER BY test_score DESC) AS decile,
    PERCENT_RANK() OVER (ORDER BY test_score) AS percent_rank,
    ROW_NUMBER() OVER (ORDER BY test_score DESC) AS rank,
    CASE 
        WHEN NTILE(4) OVER (ORDER BY test_score DESC) = 1 THEN 'A'
        WHEN NTILE(4) OVER (ORDER BY test_score DESC) = 2 THEN 'B'
        WHEN NTILE(4) OVER (ORDER BY test_score DESC) = 3 THEN 'C'
        ELSE 'D'
    END AS grade
FROM exam_results
ORDER BY test_score DESC;
```

## Related Functions
- [PERCENT_RANK](./percent_rank.md) - Relative rank as percentage
- [CUME_DIST](./cume_dist.md) - Cumulative distribution
- [ROW_NUMBER](./row_number.md) - Sequential row numbering
- [RANK](./rank.md) - Ranking with gaps
- [DENSE_RANK](./dense_rank.md) - Ranking without gaps