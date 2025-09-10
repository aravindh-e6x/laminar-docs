---
title: COVAR
description: Calculates the covariance between two sets of values
---

# COVAR

## Description
The `COVAR` function calculates the covariance between two sets of numeric values, measuring how two variables change together. A positive covariance indicates that the variables tend to move in the same direction, while negative covariance indicates they move in opposite directions. Unlike correlation, covariance is not normalized and depends on the scale of the variables.

## Syntax
```sql
COVAR_SAMP(expression1, expression2)  -- Sample covariance
COVAR_POP(expression1, expression2)   -- Population covariance
```

### Parameters
- **expression1** (numeric): First column or expression
- **expression2** (numeric): Second column or expression

### Return Value
- **Type**: DOUBLE PRECISION
- **Description**: The covariance between the two expressions. Returns NULL if there are fewer than 2 pairs of non-NULL values (for sample) or no pairs (for population).

## Example

### Sample Data
Consider the following `store_performance` table tracking store metrics:

| store_id | month    | foot_traffic | sales_amount | advertising_spend |
|----------|----------|--------------|--------------|------------------|
| 1        | 2024-01  | 5000         | 125000       | 5000             |
| 1        | 2024-02  | 5500         | 138000       | 6000             |
| 1        | 2024-03  | 6200         | 155000       | 7500             |
| 2        | 2024-01  | 3000         | 75000        | 3000             |
| 2        | 2024-02  | 3200         | 80000        | 3500             |
| 2        | 2024-03  | 3800         | 95000        | 4500             |
| 3        | 2024-01  | 7000         | 175000       | 8000             |
| 3        | 2024-02  | 7500         | 188000       | 9000             |

### Query
```sql
SELECT 
    COVAR_SAMP(foot_traffic, sales_amount) AS traffic_sales_covar,
    COVAR_POP(foot_traffic, sales_amount) AS traffic_sales_covar_pop,
    COVAR_SAMP(advertising_spend, sales_amount) AS ad_sales_covar,
    -- Relationship to correlation
    COVAR_SAMP(foot_traffic, sales_amount) / 
        (STDDEV(foot_traffic) * STDDEV(sales_amount)) AS calculated_correlation,
    CORR(foot_traffic, sales_amount) AS direct_correlation
FROM store_performance;
```

### Result
| traffic_sales_covar | traffic_sales_covar_pop | ad_sales_covar | calculated_correlation | direct_correlation |
|--------------------|------------------------|----------------|------------------------|-------------------|
| 625000000          | 555555556              | 31250000       | 1.0000                 | 1.0000            |

## Common Use Cases

### 1. Portfolio Risk Analysis
```sql
-- Calculate covariance matrix for portfolio optimization
WITH returns AS (
    SELECT 
        date,
        stock_a_return,
        stock_b_return,
        stock_c_return
    FROM daily_returns
)
SELECT 
    'A-B' AS pair,
    COVAR_SAMP(stock_a_return, stock_b_return) AS covariance,
    CORR(stock_a_return, stock_b_return) AS correlation
FROM returns
UNION ALL
SELECT 
    'A-C',
    COVAR_SAMP(stock_a_return, stock_c_return),
    CORR(stock_a_return, stock_c_return)
FROM returns
UNION ALL
SELECT 
    'B-C',
    COVAR_SAMP(stock_b_return, stock_c_return),
    CORR(stock_b_return, stock_c_return)
FROM returns;
```

### 2. Market Beta Calculation
```sql
-- Calculate stock beta using covariance
WITH market_data AS (
    SELECT 
        date,
        stock_return,
        market_return
    FROM stock_market_returns
    WHERE date >= CURRENT_DATE - INTERVAL '1 year'
)
SELECT 
    stock_symbol,
    COVAR_SAMP(stock_return, market_return) AS covariance,
    VAR_SAMP(market_return) AS market_variance,
    -- Beta = Covariance(stock, market) / Variance(market)
    COVAR_SAMP(stock_return, market_return) / NULLIF(VAR_SAMP(market_return), 0) AS beta,
    CORR(stock_return, market_return) AS correlation,
    AVG(stock_return) AS avg_return,
    STDDEV(stock_return) AS volatility
FROM market_data
GROUP BY stock_symbol;
```

### 3. Sensitivity Analysis
```sql
-- Analyze how variables co-vary with target metric
SELECT 
    'Temperature' AS variable,
    COVAR_SAMP(temperature, energy_consumption) AS covariance,
    STDDEV(temperature) AS variable_stddev,
    STDDEV(energy_consumption) AS target_stddev,
    COVAR_SAMP(temperature, energy_consumption) / 
        (STDDEV(temperature) * STDDEV(energy_consumption)) AS correlation
FROM environmental_data
UNION ALL
SELECT 
    'Humidity',
    COVAR_SAMP(humidity, energy_consumption),
    STDDEV(humidity),
    STDDEV(energy_consumption),
    CORR(humidity, energy_consumption)
FROM environmental_data
UNION ALL
SELECT 
    'Wind Speed',
    COVAR_SAMP(wind_speed, energy_consumption),
    STDDEV(wind_speed),
    STDDEV(energy_consumption),
    CORR(wind_speed, energy_consumption)
FROM environmental_data
ORDER BY ABS(covariance) DESC;
```

### 4. Time-Varying Covariance
```sql
-- Calculate rolling covariance windows
SELECT 
    date,
    COVAR_SAMP(price_a, price_b) OVER (
        ORDER BY date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS covar_30day,
    COVAR_SAMP(price_a, price_b) OVER (
        ORDER BY date 
        ROWS BETWEEN 89 PRECEDING AND CURRENT ROW
    ) AS covar_90day,
    -- Normalized by product of standard deviations for comparison
    CORR(price_a, price_b) OVER (
        ORDER BY date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS corr_30day
FROM price_data
ORDER BY date;
```

### 5. Principal Component Analysis Preparation
```sql
-- Create covariance matrix for PCA
WITH standardized AS (
    SELECT 
        (feature1 - AVG(feature1) OVER()) / STDDEV(feature1) OVER() AS std_feature1,
        (feature2 - AVG(feature2) OVER()) / STDDEV(feature2) OVER() AS std_feature2,
        (feature3 - AVG(feature3) OVER()) / STDDEV(feature3) OVER() AS std_feature3
    FROM feature_data
)
SELECT 
    'feature1' AS row_feature,
    'feature1' AS col_feature,
    COVAR_SAMP(std_feature1, std_feature1) AS covariance
FROM standardized
UNION ALL
SELECT 'feature1', 'feature2', COVAR_SAMP(std_feature1, std_feature2) FROM standardized
UNION ALL
SELECT 'feature1', 'feature3', COVAR_SAMP(std_feature1, std_feature3) FROM standardized
UNION ALL
SELECT 'feature2', 'feature2', COVAR_SAMP(std_feature2, std_feature2) FROM standardized
UNION ALL
SELECT 'feature2', 'feature3', COVAR_SAMP(std_feature2, std_feature3) FROM standardized
UNION ALL
SELECT 'feature3', 'feature3', COVAR_SAMP(std_feature3, std_feature3) FROM standardized;
```

## Sample vs Population Covariance

```sql
-- Compare sample and population covariance
WITH sample_data AS (
    SELECT x, y FROM (VALUES 
        (1, 2),
        (2, 4),
        (3, 5),
        (4, 8),
        (5, 10)
    ) AS t(x, y)
)
SELECT 
    COUNT(*) AS n,
    COVAR_SAMP(x, y) AS sample_covariance,
    COVAR_POP(x, y) AS population_covariance,
    -- Manual calculation for verification
    SUM((x - AVG(x) OVER()) * (y - AVG(y) OVER())) / (COUNT(*) - 1) AS manual_sample_covar,
    SUM((x - AVG(x) OVER()) * (y - AVG(y) OVER())) / COUNT(*) AS manual_pop_covar,
    -- Relationship: sample_covar = pop_covar * n/(n-1)
    COVAR_POP(x, y) * COUNT(*)::FLOAT / (COUNT(*) - 1) AS converted_to_sample
FROM sample_data;
```

## Covariance Decomposition

```sql
-- Decompose total covariance into components
WITH grouped_data AS (
    SELECT 
        group_id,
        x,
        y,
        AVG(x) OVER (PARTITION BY group_id) AS group_mean_x,
        AVG(y) OVER (PARTITION BY group_id) AS group_mean_y,
        AVG(x) OVER () AS grand_mean_x,
        AVG(y) OVER () AS grand_mean_y
    FROM measurements
)
SELECT 
    -- Total covariance
    COVAR_SAMP(x, y) AS total_covariance,
    -- Within-group covariance
    AVG(within_group_covar) AS avg_within_covariance,
    -- Between-group covariance
    COVAR_SAMP(group_mean_x, group_mean_y) AS between_group_covariance
FROM (
    SELECT 
        group_id,
        COVAR_SAMP(x, y) AS within_group_covar,
        AVG(group_mean_x) AS group_mean_x,
        AVG(group_mean_y) AS group_mean_y
    FROM grouped_data
    GROUP BY group_id
) group_stats;
```

## NULL Handling

```sql
-- Covariance with NULL values
WITH nullable_data AS (
    SELECT * FROM (VALUES 
        (1, 10, 100),
        (2, 20, NULL),
        (3, NULL, 300),
        (4, 40, 400),
        (5, 50, 500)
    ) AS t(id, x, y)
)
SELECT 
    COUNT(*) AS total_rows,
    COUNT(CASE WHEN x IS NOT NULL AND y IS NOT NULL THEN 1 END) AS complete_pairs,
    COVAR_SAMP(x, y) AS covariance,  -- Only uses complete pairs
    VAR_SAMP(x) AS variance_x,
    VAR_SAMP(y) AS variance_y,
    CORR(x, y) AS correlation
FROM nullable_data;
```

## Performance Considerations

```sql
-- Optimize covariance calculations
CREATE INDEX idx_paired_values ON measurements(entity_id, metric1, metric2);

-- Pre-aggregate for faster analysis
CREATE MATERIALIZED VIEW covariance_summary AS
SELECT 
    DATE_TRUNC('day', timestamp) AS day,
    metric_type_1,
    metric_type_2,
    COUNT(*) AS pair_count,
    COVAR_SAMP(value_1, value_2) AS daily_covariance,
    CORR(value_1, value_2) AS daily_correlation,
    AVG(value_1) AS avg_value_1,
    AVG(value_2) AS avg_value_2
FROM metric_pairs
GROUP BY DATE_TRUNC('day', timestamp), metric_type_1, metric_type_2
HAVING COUNT(*) > 10;
```

## Related Functions
- [CORR](./corr.md) - Correlation coefficient (normalized covariance)
- [VAR](./var.md) - Variance (covariance of a variable with itself)
- [STDDEV](./stddev.md) - Standard deviation
- [REGR_*](./regr.md) - Regression analysis functions
- [AVG](./avg.md) - Mean calculation