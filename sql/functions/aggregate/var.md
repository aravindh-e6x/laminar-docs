---
title: VAR
description: Calculates the variance of a set of values
---

# VAR

## Description
The `VAR` function calculates the sample variance of a set of numeric values, measuring how far the values are spread out from their average. Variance is the square of the standard deviation and is fundamental in statistical analysis, risk assessment, and data quality measurements. NULL values are ignored in the calculation.

## Syntax
```sql
VAR(expression)
VAR_SAMP(expression)  -- Same as VAR (sample variance)
VAR_POP(expression)   -- Population variance
VARIANCE(expression)  -- Alias for VAR_SAMP
```

### Parameters
- **expression** (numeric): A column or expression that evaluates to a numeric value

### Return Value
- **Type**: DOUBLE PRECISION
- **Description**: The variance of non-NULL values. Returns NULL if there are fewer than 2 values (for sample) or no values (for population).

## Example

### Sample Data
Consider the following `product_sales` table tracking daily sales variance:

| date       | product_id | region | units_sold | revenue | price_per_unit |
|------------|------------|--------|------------|---------|----------------|
| 2024-03-01 | P001       | North  | 150        | 4500    | 30.00          |
| 2024-03-01 | P001       | South  | 200        | 6000    | 30.00          |
| 2024-03-01 | P001       | East   | 175        | 5250    | 30.00          |
| 2024-03-02 | P001       | North  | 180        | 5400    | 30.00          |
| 2024-03-02 | P001       | South  | 160        | 4800    | 30.00          |
| 2024-03-02 | P001       | West   | 220        | 6600    | 30.00          |
| 2024-03-03 | P001       | North  | 195        | 5850    | 30.00          |
| 2024-03-03 | P001       | East   | 210        | 6300    | 30.00          |

### Query
```sql
SELECT 
    product_id,
    COUNT(*) AS sales_count,
    AVG(units_sold) AS avg_units,
    VAR_SAMP(units_sold) AS sample_variance,
    VAR_POP(units_sold) AS population_variance,
    STDDEV(units_sold) AS standard_deviation,
    -- Variance = StdDev^2
    POWER(STDDEV(units_sold), 2) AS variance_from_stddev
FROM product_sales
GROUP BY product_id;
```

### Result
| product_id | sales_count | avg_units | sample_variance | population_variance | standard_deviation | variance_from_stddev |
|------------|-------------|-----------|-----------------|--------------------|--------------------|---------------------|
| P001       | 8           | 184.375   | 593.125         | 518.984            | 24.356             | 593.125             |

## Common Use Cases

### 1. Portfolio Risk Analysis
```sql
-- Calculate variance in returns for risk assessment
SELECT 
    portfolio_id,
    COUNT(*) AS periods,
    AVG(daily_return) AS avg_return,
    VAR_SAMP(daily_return) AS return_variance,
    STDDEV(daily_return) AS return_volatility,
    -- Information ratio: return per unit of risk
    AVG(daily_return) / NULLIF(STDDEV(daily_return), 0) AS sharpe_ratio
FROM portfolio_returns
GROUP BY portfolio_id
ORDER BY return_variance;
```

### 2. Quality Control Analysis
```sql
-- Monitor production variance for quality control
WITH production_stats AS (
    SELECT 
        production_line,
        DATE_TRUNC('hour', timestamp) AS hour,
        AVG(measurement) AS hourly_mean,
        VAR_SAMP(measurement) AS hourly_variance
    FROM quality_measurements
    GROUP BY production_line, DATE_TRUNC('hour', timestamp)
)
SELECT 
    production_line,
    AVG(hourly_variance) AS avg_variance,
    MIN(hourly_variance) AS min_variance,
    MAX(hourly_variance) AS max_variance,
    CASE 
        WHEN MAX(hourly_variance) > 100 THEN 'High Variability - Investigate'
        WHEN MAX(hourly_variance) > 50 THEN 'Moderate Variability'
        ELSE 'Stable Process'
    END AS process_status
FROM production_stats
GROUP BY production_line;
```

### 3. A/B Testing Analysis
```sql
-- Compare variance between test groups
SELECT 
    test_group,
    COUNT(*) AS sample_size,
    AVG(conversion_rate) AS mean_conversion,
    VAR_SAMP(conversion_rate) AS variance,
    -- Standard error
    SQRT(VAR_SAMP(conversion_rate) / COUNT(*)) AS standard_error,
    -- 95% confidence interval
    AVG(conversion_rate) - 1.96 * SQRT(VAR_SAMP(conversion_rate) / COUNT(*)) AS ci_lower,
    AVG(conversion_rate) + 1.96 * SQRT(VAR_SAMP(conversion_rate) / COUNT(*)) AS ci_upper
FROM ab_test_results
GROUP BY test_group;
```

### 4. Time Series Decomposition
```sql
-- Analyze variance components over time
SELECT 
    DATE_TRUNC('month', sale_date) AS month,
    VAR_SAMP(daily_sales) AS total_variance,
    -- Between-day variance
    VAR_SAMP(AVG(daily_sales) OVER (PARTITION BY DATE_TRUNC('week', sale_date))) AS between_week_variance,
    -- Within-week variance
    AVG(VAR_SAMP(daily_sales) OVER (PARTITION BY DATE_TRUNC('week', sale_date))) AS within_week_variance
FROM (
    SELECT 
        sale_date,
        SUM(amount) AS daily_sales
    FROM sales
    GROUP BY sale_date
) daily_totals
GROUP BY DATE_TRUNC('month', sale_date);
```

### 5. Homogeneity of Variance Testing
```sql
-- Test if variance is consistent across groups (Levene's test preparation)
WITH group_stats AS (
    SELECT 
        group_id,
        value,
        AVG(value) OVER (PARTITION BY group_id) AS group_mean,
        MEDIAN(value) OVER (PARTITION BY group_id) AS group_median
    FROM measurements
)
SELECT 
    group_id,
    COUNT(*) AS n,
    VAR_SAMP(value) AS group_variance,
    -- Variance of absolute deviations from median (robust to outliers)
    VAR_SAMP(ABS(value - group_median)) AS mad_variance,
    -- F-statistic denominator
    AVG(VAR_SAMP(value)) OVER () AS pooled_variance
FROM group_stats
GROUP BY group_id;
```

## Sample vs Population Variance

```sql
-- Understanding the difference between sample and population variance
WITH example_data AS (
    SELECT value FROM (VALUES (2), (4), (6), (8), (10)) AS t(value)
)
SELECT 
    COUNT(*) AS n,
    AVG(value) AS mean,
    -- Sample variance: divides by (n-1) for unbiased estimate
    VAR_SAMP(value) AS sample_var,
    SUM(POWER(value - AVG(value) OVER(), 2)) / (COUNT(*) - 1) AS manual_sample_var,
    -- Population variance: divides by n
    VAR_POP(value) AS population_var,
    SUM(POWER(value - AVG(value) OVER(), 2)) / COUNT(*) AS manual_pop_var,
    -- Relationship: sample_var = population_var * n/(n-1)
    VAR_POP(value) * COUNT(*)::FLOAT / (COUNT(*) - 1) AS converted_to_sample
FROM example_data;
```

## Pooled Variance Calculation

```sql
-- Calculate pooled variance for multiple groups
WITH group_variances AS (
    SELECT 
        group_name,
        COUNT(*) AS n,
        VAR_SAMP(value) AS variance,
        (COUNT(*) - 1) AS degrees_of_freedom
    FROM grouped_data
    GROUP BY group_name
)
SELECT 
    SUM(degrees_of_freedom * variance) / SUM(degrees_of_freedom) AS pooled_variance,
    SQRT(SUM(degrees_of_freedom * variance) / SUM(degrees_of_freedom)) AS pooled_std_dev,
    SUM(degrees_of_freedom) AS total_df
FROM group_variances;
```

## NULL Handling

```sql
-- VAR behavior with NULL values
WITH nullable_data AS (
    SELECT * FROM (VALUES 
        ('A', 10),
        ('A', 20),
        ('A', NULL),
        ('A', 30),
        ('B', 15),
        ('B', NULL),
        ('B', 25)
    ) AS t(group_id, value)
)
SELECT 
    group_id,
    COUNT(*) AS total_rows,
    COUNT(value) AS non_null_count,
    VAR_SAMP(value) AS variance,
    -- Variance with NULL replacement
    VAR_SAMP(COALESCE(value, AVG(value) OVER (PARTITION BY group_id))) AS variance_null_as_mean
FROM nullable_data
GROUP BY group_id;
```

## Performance Optimization

```sql
-- Pre-calculate variance for large datasets
CREATE MATERIALIZED VIEW daily_variance_stats AS
SELECT 
    product_id,
    sale_date,
    COUNT(*) AS transaction_count,
    AVG(amount) AS mean_amount,
    VAR_SAMP(amount) AS variance_amount,
    MIN(amount) AS min_amount,
    MAX(amount) AS max_amount
FROM transactions
GROUP BY product_id, sale_date;

-- Use the materialized view for faster queries
SELECT 
    product_id,
    AVG(variance_amount) AS avg_daily_variance,
    VAR_SAMP(mean_amount) AS variance_of_daily_means
FROM daily_variance_stats
WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY product_id;
```

## Related Functions
- [STDDEV](./stddev.md) - Square root of variance
- [COVAR](./covar.md) - Covariance between two variables
- [CORR](./corr.md) - Correlation coefficient
- [AVG](./avg.md) - Mean value calculation
- [MAD](./mad.md) - Median absolute deviation
- [REGR_*](./regr.md) - Regression analysis functions