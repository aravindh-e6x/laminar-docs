---
title: CORR
description: Calculates the correlation coefficient between two sets of values
---

# CORR

## Description
The `CORR` function calculates the Pearson correlation coefficient between two sets of numeric values, measuring the linear relationship between them. The result ranges from -1 to 1, where 1 indicates perfect positive correlation, -1 indicates perfect negative correlation, and 0 indicates no linear correlation. This function is essential for statistical analysis, feature selection, and understanding relationships between variables.

## Syntax
```sql
CORR(expression1, expression2)
```

### Parameters
- **expression1** (numeric): First column or expression
- **expression2** (numeric): Second column or expression

### Return Value
- **Type**: DOUBLE PRECISION
- **Description**: Correlation coefficient between -1 and 1. Returns NULL if there are fewer than 2 pairs of non-NULL values.

## Example

### Sample Data
Consider the following `marketing_metrics` table analyzing campaign performance:

| campaign_id | ad_spend | impressions | clicks | conversions | revenue |
|------------|----------|-------------|--------|-------------|---------|
| 1          | 1000     | 50000       | 1500   | 75          | 3750    |
| 2          | 2000     | 95000       | 3200   | 180         | 9000    |
| 3          | 1500     | 72000       | 2100   | 120         | 6000    |
| 4          | 3000     | 140000      | 4800   | 290         | 14500   |
| 5          | 500      | 28000       | 600    | 25          | 1250    |
| 6          | 2500     | 118000      | 3900   | 220         | 11000   |
| 7          | 1800     | 85000       | 2600   | 145         | 7250    |
| 8          | 4000     | 185000      | 6200   | 380         | 19000   |

### Query
```sql
SELECT 
    CORR(ad_spend, revenue) AS spend_revenue_corr,
    CORR(impressions, clicks) AS impressions_clicks_corr,
    CORR(clicks, conversions) AS clicks_conversions_corr,
    CORR(ad_spend, impressions) AS spend_impressions_corr,
    CORR(conversions, revenue) AS conversions_revenue_corr
FROM marketing_metrics;
```

### Result
| spend_revenue_corr | impressions_clicks_corr | clicks_conversions_corr | spend_impressions_corr | conversions_revenue_corr |
|-------------------|------------------------|------------------------|----------------------|-------------------------|
| 0.9987            | 0.9994                 | 0.9996                 | 0.9991               | 1.0000                  |

## Common Use Cases

### 1. Feature Correlation Matrix
```sql
-- Create correlation matrix for feature selection
WITH correlations AS (
    SELECT 
        'ad_spend' AS feature1, 'impressions' AS feature2, CORR(ad_spend, impressions) AS correlation
    FROM marketing_metrics
    UNION ALL
    SELECT 'ad_spend', 'clicks', CORR(ad_spend, clicks) FROM marketing_metrics
    UNION ALL
    SELECT 'ad_spend', 'conversions', CORR(ad_spend, conversions) FROM marketing_metrics
    UNION ALL
    SELECT 'impressions', 'clicks', CORR(impressions, clicks) FROM marketing_metrics
    UNION ALL
    SELECT 'impressions', 'conversions', CORR(impressions, conversions) FROM marketing_metrics
    UNION ALL
    SELECT 'clicks', 'conversions', CORR(clicks, conversions) FROM marketing_metrics
)
SELECT 
    feature1,
    feature2,
    ROUND(correlation, 4) AS correlation,
    CASE 
        WHEN ABS(correlation) > 0.9 THEN 'Very Strong'
        WHEN ABS(correlation) > 0.7 THEN 'Strong'
        WHEN ABS(correlation) > 0.5 THEN 'Moderate'
        WHEN ABS(correlation) > 0.3 THEN 'Weak'
        ELSE 'Very Weak'
    END AS strength
FROM correlations
ORDER BY ABS(correlation) DESC;
```

### 2. Time-Lagged Correlations
```sql
-- Find optimal lag for predictive relationships
WITH lagged_data AS (
    SELECT 
        date,
        temperature,
        LAG(temperature, 1) OVER (ORDER BY date) AS temp_lag1,
        LAG(temperature, 7) OVER (ORDER BY date) AS temp_lag7,
        ice_cream_sales
    FROM daily_sales
)
SELECT 
    CORR(temperature, ice_cream_sales) AS same_day_corr,
    CORR(temp_lag1, ice_cream_sales) AS lag1_day_corr,
    CORR(temp_lag7, ice_cream_sales) AS lag7_day_corr
FROM lagged_data;
```

### 3. Cross-Category Correlations
```sql
-- Analyze relationships between different product categories
SELECT 
    p1.category AS category1,
    p2.category AS category2,
    CORR(p1.daily_sales, p2.daily_sales) AS sales_correlation
FROM 
    (SELECT date, category, SUM(sales) AS daily_sales 
     FROM sales GROUP BY date, category) p1
JOIN 
    (SELECT date, category, SUM(sales) AS daily_sales 
     FROM sales GROUP BY date, category) p2
ON p1.date = p2.date AND p1.category < p2.category
GROUP BY p1.category, p2.category
HAVING COUNT(*) > 30  -- Ensure sufficient data points
ORDER BY ABS(CORR(p1.daily_sales, p2.daily_sales)) DESC;
```

### 4. Moving Correlation Windows
```sql
-- Calculate rolling correlation over time
SELECT 
    date,
    CORR(price, volume) OVER (
        ORDER BY date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS rolling_30day_correlation,
    CORR(price, volume) OVER (
        ORDER BY date 
        ROWS BETWEEN 89 PRECEDING AND CURRENT ROW
    ) AS rolling_90day_correlation
FROM stock_data
ORDER BY date;
```

### 5. Partial Correlation Analysis
```sql
-- Calculate correlation while controlling for a third variable
WITH residuals AS (
    SELECT 
        -- Residuals after regressing on control variable
        sales - (REGR_INTERCEPT(sales, marketing_spend) OVER () + 
                REGR_SLOPE(sales, marketing_spend) OVER () * marketing_spend) AS sales_residual,
        price - (REGR_INTERCEPT(price, marketing_spend) OVER () + 
                REGR_SLOPE(price, marketing_spend) OVER () * marketing_spend) AS price_residual,
        marketing_spend
    FROM product_metrics
)
SELECT 
    CORR(sales, price) AS raw_correlation,
    (SELECT CORR(sales_residual, price_residual) FROM residuals) AS partial_correlation
FROM product_metrics;
```

## Statistical Significance Testing

```sql
-- Calculate correlation with significance test
WITH correlation_stats AS (
    SELECT 
        CORR(x, y) AS r,
        COUNT(*) AS n
    FROM data_points
    WHERE x IS NOT NULL AND y IS NOT NULL
)
SELECT 
    r AS correlation,
    n AS sample_size,
    -- t-statistic for correlation
    r * SQRT((n - 2) / (1 - r * r)) AS t_statistic,
    -- Approximate p-value interpretation
    CASE 
        WHEN ABS(r * SQRT((n - 2) / (1 - r * r))) > 3.291 THEN 'p < 0.001'
        WHEN ABS(r * SQRT((n - 2) / (1 - r * r))) > 2.576 THEN 'p < 0.01'
        WHEN ABS(r * SQRT((n - 2) / (1 - r * r))) > 1.96 THEN 'p < 0.05'
        ELSE 'p >= 0.05'
    END AS significance
FROM correlation_stats;
```

## Correlation vs Causation Warning

```sql
-- Example showing spurious correlation
SELECT 
    year,
    CORR(ice_cream_sales, swimming_pool_drownings) 
        OVER (ORDER BY year ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS correlation,
    -- Both are actually correlated with temperature (confounding variable)
    CORR(temperature, ice_cream_sales) 
        OVER (ORDER BY year ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS temp_ice_cream_corr,
    CORR(temperature, swimming_pool_drownings) 
        OVER (ORDER BY year ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS temp_drowning_corr
FROM seasonal_data
ORDER BY year;
```

## NULL Handling

```sql
-- CORR behavior with NULL values
WITH sample_data AS (
    SELECT * FROM (VALUES 
        (1, 10, 100),
        (2, 20, 200),
        (3, NULL, 300),
        (4, 40, NULL),
        (5, 50, 500)
    ) AS t(id, x, y)
)
SELECT 
    COUNT(*) AS total_rows,
    COUNT(x) AS x_non_null,
    COUNT(y) AS y_non_null,
    COUNT(CASE WHEN x IS NOT NULL AND y IS NOT NULL THEN 1 END) AS complete_pairs,
    CORR(x, y) AS correlation  -- Only uses complete pairs (rows 1, 2, 5)
FROM sample_data;
```

## Performance Optimization

```sql
-- Pre-calculate correlations for large datasets
CREATE MATERIALIZED VIEW feature_correlations AS
SELECT 
    f1.feature_name AS feature1,
    f2.feature_name AS feature2,
    CORR(f1.value, f2.value) AS correlation,
    COUNT(*) AS pair_count
FROM feature_values f1
JOIN feature_values f2 ON f1.record_id = f2.record_id
WHERE f1.feature_name <= f2.feature_name
GROUP BY f1.feature_name, f2.feature_name
HAVING COUNT(*) > 100;  -- Minimum sample size

-- Use materialized view for quick lookups
SELECT * FROM feature_correlations
WHERE ABS(correlation) > 0.8  -- Find highly correlated features
AND feature1 != feature2;
```

## Related Functions
- [COVAR_SAMP](./covar.md) - Sample covariance
- [REGR_SLOPE](./regr.md) - Linear regression slope
- [REGR_INTERCEPT](./regr.md) - Linear regression intercept
- [STDDEV](./stddev.md) - Standard deviation
- [VAR](./var.md) - Variance