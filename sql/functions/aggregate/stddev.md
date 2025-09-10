---
title: STDDEV
description: Calculates the standard deviation of a set of values
---

# STDDEV

## Description
The `STDDEV` function calculates the sample standard deviation of a set of numeric values, measuring the amount of variation or dispersion in the dataset. It returns the square root of the sample variance. This function is essential for statistical analysis, quality control, risk assessment, and identifying outliers. NULL values are ignored in the calculation.

## Syntax
```sql
STDDEV(expression)
STDDEV_SAMP(expression)  -- Same as STDDEV
STDDEV_POP(expression)   -- Population standard deviation
```

### Parameters
- **expression** (numeric): A column or expression that evaluates to a numeric value

### Return Value
- **Type**: DOUBLE PRECISION
- **Description**: The standard deviation of non-NULL values. Returns NULL if there are fewer than 2 values (for sample) or no values (for population).

## Example

### Sample Data
Consider the following `performance_metrics` table tracking employee performance scores:

| employee_id | department | quarter | performance_score | sales_amount | customer_rating |
|------------|------------|---------|------------------|--------------|-----------------|
| 1          | Sales      | Q1      | 85               | 120000       | 4.5             |
| 2          | Sales      | Q1      | 92               | 150000       | 4.8             |
| 3          | Sales      | Q1      | 78               | 95000        | 4.2             |
| 4          | Marketing  | Q1      | 88               | NULL         | 4.6             |
| 5          | Marketing  | Q1      | 91               | NULL         | 4.7             |
| 6          | Engineering| Q1      | 95               | NULL         | 4.9             |
| 7          | Engineering| Q1      | 89               | NULL         | 4.4             |
| 8          | Engineering| Q1      | 93               | NULL         | 4.8             |

### Query
```sql
SELECT 
    COUNT(*) AS employee_count,
    AVG(performance_score) AS avg_score,
    STDDEV(performance_score) AS stddev_sample,
    STDDEV_POP(performance_score) AS stddev_population,
    VAR_SAMP(performance_score) AS variance_sample,
    MIN(performance_score) AS min_score,
    MAX(performance_score) AS max_score
FROM performance_metrics;
```

### Result
| employee_count | avg_score | stddev_sample | stddev_population | variance_sample | min_score | max_score |
|---------------|-----------|---------------|-------------------|-----------------|-----------|-----------|
| 8             | 88.875    | 5.843         | 5.463             | 34.125          | 78        | 95        |

## Common Use Cases

### 1. Identifying Outliers
```sql
-- Find scores outside 2 standard deviations
WITH stats AS (
    SELECT 
        AVG(performance_score) AS mean,
        STDDEV(performance_score) AS std_dev
    FROM performance_metrics
)
SELECT 
    employee_id,
    performance_score,
    s.mean,
    s.std_dev,
    ABS(performance_score - s.mean) / s.std_dev AS z_score,
    CASE 
        WHEN ABS(performance_score - s.mean) > 2 * s.std_dev THEN 'Outlier'
        ELSE 'Normal'
    END AS classification
FROM performance_metrics, stats s
ORDER BY z_score DESC;
```

### 2. Department Variability Analysis
```sql
-- Compare consistency across departments
SELECT 
    department,
    COUNT(*) AS employee_count,
    AVG(performance_score) AS avg_score,
    STDDEV(performance_score) AS score_stddev,
    STDDEV(customer_rating) AS rating_stddev,
    -- Coefficient of variation (relative variability)
    STDDEV(performance_score) * 100.0 / AVG(performance_score) AS cv_percentage
FROM performance_metrics
GROUP BY department
ORDER BY cv_percentage;
```

### 3. Time Series Volatility
```sql
-- Calculate rolling standard deviation
SELECT 
    quarter,
    employee_id,
    performance_score,
    AVG(performance_score) OVER w AS rolling_avg,
    STDDEV(performance_score) OVER w AS rolling_stddev,
    (performance_score - AVG(performance_score) OVER w) / 
        NULLIF(STDDEV(performance_score) OVER w, 0) AS normalized_score
FROM performance_metrics
WINDOW w AS (
    PARTITION BY employee_id 
    ORDER BY quarter 
    ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
)
ORDER BY employee_id, quarter;
```

### 4. Risk Assessment
```sql
-- Calculate risk metrics for sales teams
SELECT 
    department,
    AVG(sales_amount) AS avg_sales,
    STDDEV(sales_amount) AS sales_volatility,
    -- Sharpe ratio analog (return/risk)
    AVG(sales_amount) / NULLIF(STDDEV(sales_amount), 0) AS performance_risk_ratio,
    MIN(sales_amount) AS worst_performance,
    MAX(sales_amount) AS best_performance
FROM performance_metrics
WHERE sales_amount IS NOT NULL
GROUP BY department;
```

### 5. Quality Control Charts
```sql
-- Create control limits for process monitoring
WITH control_stats AS (
    SELECT 
        AVG(performance_score) AS center_line,
        AVG(performance_score) + 3 * STDDEV(performance_score) AS upper_control_limit,
        AVG(performance_score) - 3 * STDDEV(performance_score) AS lower_control_limit,
        AVG(performance_score) + STDDEV(performance_score) AS upper_1sigma,
        AVG(performance_score) - STDDEV(performance_score) AS lower_1sigma
    FROM performance_metrics
)
SELECT 
    p.employee_id,
    p.performance_score,
    c.center_line,
    c.upper_control_limit,
    c.lower_control_limit,
    CASE 
        WHEN p.performance_score > c.upper_control_limit THEN 'Above UCL'
        WHEN p.performance_score < c.lower_control_limit THEN 'Below LCL'
        WHEN p.performance_score > c.upper_1sigma THEN 'Above 1σ'
        WHEN p.performance_score < c.lower_1sigma THEN 'Below 1σ'
        ELSE 'Within 1σ'
    END AS control_status
FROM performance_metrics p, control_stats c;
```

## Sample vs Population Standard Deviation

```sql
-- Comparing sample and population standard deviation
WITH small_sample AS (
    SELECT value FROM (VALUES (10), (20), (30), (40), (50)) AS t(value)
)
SELECT 
    COUNT(*) AS n,
    AVG(value) AS mean,
    STDDEV_SAMP(value) AS sample_stddev,    -- Divides by (n-1)
    STDDEV_POP(value) AS population_stddev, -- Divides by n
    VAR_SAMP(value) AS sample_variance,
    VAR_POP(value) AS population_variance,
    -- Relationship: sample_stddev = population_stddev * sqrt(n/(n-1))
    STDDEV_POP(value) * SQRT(COUNT(*)::FLOAT / (COUNT(*) - 1)) AS calculated_sample_stddev
FROM small_sample;
```

## NULL Handling

```sql
-- STDDEV behavior with NULLs
WITH data_with_nulls AS (
    SELECT * FROM (VALUES 
        (1, 100),
        (2, 200),
        (3, NULL),
        (4, 150),
        (5, NULL),
        (6, 180)
    ) AS t(id, value)
)
SELECT 
    COUNT(*) AS total_rows,
    COUNT(value) AS non_null_values,
    AVG(value) AS mean,
    STDDEV(value) AS stddev,  -- Only considers non-NULL values
    -- Manual calculation to verify
    SQRT(SUM(POWER(value - AVG(value) OVER(), 2)) / (COUNT(value) - 1)) AS manual_stddev
FROM data_with_nulls;
```

## Performance Considerations

```sql
-- Optimize STDDEV calculations with indexing
CREATE INDEX idx_metrics_dept_score ON performance_metrics(department, performance_score);

-- Pre-aggregate for frequently accessed statistics
CREATE MATERIALIZED VIEW department_statistics AS
SELECT 
    department,
    quarter,
    COUNT(*) AS employee_count,
    AVG(performance_score) AS mean_score,
    STDDEV(performance_score) AS stddev_score,
    MIN(performance_score) AS min_score,
    MAX(performance_score) AS max_score,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY performance_score) AS median_score
FROM performance_metrics
GROUP BY department, quarter;
```

## Related Functions
- [VAR](./var.md) - Calculates variance (STDDEV squared)
- [AVG](./avg.md) - Calculates the mean value
- [COVAR](./covar.md) - Calculates covariance between two variables
- [CORR](./corr.md) - Calculates correlation coefficient
- [MAD](./mad.md) - Median absolute deviation
- [PERCENTILE_CONT](./percentile_cont.md) - Calculate percentiles