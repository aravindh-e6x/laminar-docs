---
title: AVG
description: Calculates the arithmetic mean of a set of numeric values
---

# AVG

## Description
The `AVG` function calculates the arithmetic mean (average) of a set of numeric values. It sums all non-NULL values in the specified column or expression and divides by the count of non-NULL values. This function is commonly used in statistical analysis, reporting, and data aggregation. NULL values are ignored in the calculation.

## Syntax
```sql
AVG([DISTINCT] expression)
```

### Parameters
- **DISTINCT** (optional): When specified, calculates the average of unique values only
- **expression** (numeric): A column or expression that evaluates to a numeric value

### Return Value
- **Type**: DECIMAL/FLOAT (typically with higher precision than input)
- **Description**: The arithmetic mean of all non-NULL values. Returns NULL if all values are NULL or no rows match.

## Example

### Sample Data
Consider the following `product_reviews` table with customer ratings:

| review_id | product_id | customer_id | rating | review_date |
|-----------|------------|-------------|--------|-------------|
| 1         | 101        | 501         | 4.5    | 2024-03-01  |
| 2         | 101        | 502         | 5.0    | 2024-03-02  |
| 3         | 101        | 503         | 3.5    | 2024-03-03  |
| 4         | 102        | 504         | 4.0    | 2024-03-04  |
| 5         | 102        | 505         | NULL   | 2024-03-05  |
| 6         | 102        | 506         | 4.5    | 2024-03-06  |
| 7         | 103        | 507         | 2.0    | 2024-03-07  |
| 8         | 103        | 508         | 3.0    | 2024-03-08  |

### Query
```sql
-- Calculate various averages
SELECT 
    'Overall' AS category,
    COUNT(*) AS total_reviews,
    COUNT(rating) AS reviews_with_rating,
    AVG(rating) AS average_rating,
    ROUND(AVG(rating), 2) AS avg_rating_rounded
FROM product_reviews

UNION ALL

-- Average by product
SELECT 
    'Product ' || product_id AS category,
    COUNT(*) AS total_reviews,
    COUNT(rating) AS reviews_with_rating,
    AVG(rating) AS average_rating,
    ROUND(AVG(rating), 2) AS avg_rating_rounded
FROM product_reviews
GROUP BY product_id
ORDER BY category;
```

### Result
| category     | total_reviews | reviews_with_rating | average_rating | avg_rating_rounded |
|--------------|---------------|---------------------|----------------|-------------------|
| Overall      | 8             | 7                   | 3.7857         | 3.79              |
| Product 101  | 3             | 3                   | 4.3333         | 4.33              |
| Product 102  | 3             | 2                   | 4.2500         | 4.25              |
| Product 103  | 2             | 2                   | 2.5000         | 2.50              |

## Common Use Cases

### 1. Statistical Analysis
```sql
-- Calculate average with standard deviation
SELECT 
    category,
    AVG(price) AS avg_price,
    STDDEV(price) AS price_stddev,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING AVG(price) > 50;
```

### 2. Moving Averages
```sql
-- Calculate 7-day moving average
SELECT 
    date,
    daily_sales,
    AVG(daily_sales) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7day
FROM sales_data
ORDER BY date;
```

### 3. Weighted Average
```sql
-- Calculate weighted average price
SELECT 
    product_category,
    SUM(price * quantity) / SUM(quantity) AS weighted_avg_price,
    AVG(price) AS simple_avg_price
FROM inventory
GROUP BY product_category;
```

### 4. Comparing to Average
```sql
-- Find products priced above category average
SELECT 
    p.product_name,
    p.category,
    p.price,
    cat_avg.avg_price,
    p.price - cat_avg.avg_price AS diff_from_avg,
    ROUND((p.price / cat_avg.avg_price - 1) * 100, 2) AS percent_diff
FROM products p
JOIN (
    SELECT category, AVG(price) AS avg_price
    FROM products
    GROUP BY category
) cat_avg ON p.category = cat_avg.category
WHERE p.price > cat_avg.avg_price;
```

### 5. Time-Based Averages
```sql
-- Average sales by day of week
SELECT 
    EXTRACT(DOW FROM order_date) AS day_of_week,
    CASE EXTRACT(DOW FROM order_date)
        WHEN 0 THEN 'Sunday'
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
    END AS day_name,
    AVG(order_total) AS avg_order_value,
    COUNT(*) AS order_count
FROM orders
GROUP BY EXTRACT(DOW FROM order_date)
ORDER BY day_of_week;
```

## NULL Handling

```sql
-- Demonstrating NULL handling in AVG
WITH sample_data AS (
    SELECT 1 AS id, 10 AS value
    UNION ALL SELECT 2, 20
    UNION ALL SELECT 3, NULL
    UNION ALL SELECT 4, 30
    UNION ALL SELECT 5, NULL
)
SELECT 
    COUNT(*) AS total_rows,
    COUNT(value) AS non_null_values,
    SUM(value) AS sum_values,
    AVG(value) AS average_value,
    -- AVG ignores NULLs: (10+20+30)/3 = 20
    SUM(value)::FLOAT / COUNT(*) AS avg_including_nulls_as_zero
    -- This would treat NULLs as 0: (10+20+0+30+0)/5 = 12
FROM sample_data;
```

## DISTINCT Usage

```sql
-- Average of distinct values
WITH duplicate_scores AS (
    SELECT 85 AS score
    UNION ALL SELECT 85
    UNION ALL SELECT 90
    UNION ALL SELECT 90
    UNION ALL SELECT 95
)
SELECT 
    AVG(score) AS avg_all,          -- (85+85+90+90+95)/5 = 89
    AVG(DISTINCT score) AS avg_distinct  -- (85+90+95)/3 = 90
FROM duplicate_scores;
```

## Related Functions
- [SUM](./sum.md) - Calculates the sum of values
- [COUNT](./count.md) - Counts rows or non-null values
- [MIN](./min.md) - Returns the minimum value
- [MAX](./max.md) - Returns the maximum value
- [MEDIAN](./median.md) - Calculates the median value
- [STDDEV](./stddev.md) - Calculates standard deviation