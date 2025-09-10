---
title: MIN
description: Returns the minimum value from a set of values
---

# MIN

## Description
The `MIN` function returns the smallest value from a set of values in a specified column or expression. It works with numeric, string, date/time, and other comparable data types. For strings, MIN returns the first value in alphabetical order. NULL values are ignored in the calculation. This function is commonly used for finding baseline values, earliest dates, or lower boundaries in your data.

## Syntax
```sql
MIN([DISTINCT] expression)
```

### Parameters
- **DISTINCT** (optional): Has no effect for MIN (included for SQL standard compatibility)
- **expression**: A column or expression that evaluates to a comparable value

### Return Value
- **Type**: Same as the input expression type
- **Description**: The minimum value from all non-NULL values. Returns NULL if all values are NULL or no rows match.

## Example

### Sample Data
Consider the following `employee_salaries` table tracking employee compensation:

| emp_id | employee_name | department   | salary  | bonus   | hire_date  | performance_score |
|--------|--------------|--------------|---------|---------|------------|-------------------|
| 1      | Alice Chen   | Engineering  | 120000  | 15000   | 2021-03-15 | 4.5               |
| 2      | Bob Smith    | Sales        | 85000   | 25000   | 2022-06-01 | 3.8               |
| 3      | Carol Davis  | Engineering  | 95000   | 10000   | 2023-01-10 | 4.2               |
| 4      | David Lee    | Marketing    | 75000   | NULL    | 2020-09-20 | 3.5               |
| 5      | Emma Wilson  | Sales        | 90000   | 20000   | 2021-11-05 | 4.0               |
| 6      | Frank Jones  | HR           | 70000   | 5000    | 2022-02-28 | NULL              |

### Query
```sql
SELECT 
    MIN(salary) AS min_salary,
    MIN(bonus) AS min_bonus,
    MIN(hire_date) AS earliest_hire,
    MIN(performance_score) AS lowest_score,
    MIN(employee_name) AS first_name_alphabetically,
    MIN(CASE WHEN department = 'Engineering' THEN salary END) AS min_engineering_salary
FROM employee_salaries;
```

### Result
| min_salary | min_bonus | earliest_hire | lowest_score | first_name_alphabetically | min_engineering_salary |
|------------|-----------|---------------|--------------|---------------------------|------------------------|
| 70000      | 5000      | 2020-09-20    | 3.5          | Alice Chen                | 95000                  |

## Common Use Cases

### 1. Finding Starting Points
```sql
-- Find earliest customer interactions
SELECT 
    customer_id,
    MIN(order_date) AS first_order_date,
    MIN(order_total) AS smallest_order,
    COUNT(*) AS total_orders,
    CURRENT_DATE - MIN(order_date) AS customer_lifetime_days
FROM orders
GROUP BY customer_id
ORDER BY first_order_date;
```

### 2. Range Analysis
```sql
-- Analyze data ranges by category
SELECT 
    product_category,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    MAX(price) - MIN(price) AS price_spread,
    MIN(stock_level) AS min_stock,
    MIN(last_restock_date) AS oldest_stock
FROM inventory
WHERE is_active = true
GROUP BY product_category
HAVING MIN(stock_level) < 10;
```

### 3. Time Series Analysis
```sql
-- Find daily minimums for monitoring
SELECT 
    DATE(timestamp) AS date,
    MIN(response_time) AS best_response_time,
    MIN(error_rate) AS lowest_error_rate,
    MIN(cpu_usage) AS min_cpu,
    AVG(response_time) AS avg_response_time
FROM performance_metrics
WHERE timestamp >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY DATE(timestamp)
ORDER BY date;
```

### 4. Window Function Usage
```sql
-- Calculate running minimum and identify new lows
SELECT 
    trading_date,
    stock_symbol,
    closing_price,
    MIN(closing_price) OVER (
        PARTITION BY stock_symbol 
        ORDER BY trading_date
    ) AS running_min,
    MIN(closing_price) OVER (
        PARTITION BY stock_symbol 
        ORDER BY trading_date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS min_last_30_days,
    closing_price = MIN(closing_price) OVER (
        PARTITION BY stock_symbol 
        ORDER BY trading_date
    ) AS is_new_low
FROM stock_prices
ORDER BY stock_symbol, trading_date;
```

### 5. Finding Baseline Values
```sql
-- Identify products below minimum threshold
SELECT 
    p.product_id,
    p.product_name,
    p.current_price,
    cat_min.min_category_price,
    p.current_price - cat_min.min_category_price AS price_above_min
FROM products p
JOIN (
    SELECT 
        category,
        MIN(current_price) AS min_category_price
    FROM products
    WHERE is_active = true
    GROUP BY category
) cat_min ON p.category = cat_min.category
WHERE p.current_price = cat_min.min_category_price;
```

## Data Type Comparisons

```sql
-- MIN with different data types
WITH diverse_data AS (
    SELECT 'Zebra' AS animal, 99.99 AS price, TIMESTAMP '2024-12-31 23:59:59' AS ts, TRUE AS flag
    UNION ALL SELECT 'Aardvark', 10.50, TIMESTAMP '2024-01-01 00:00:00', FALSE
    UNION ALL SELECT 'Lion', 49.99, TIMESTAMP '2024-06-15 12:30:00', TRUE
    UNION ALL SELECT NULL, 25.00, TIMESTAMP '2024-03-20 08:15:00', NULL
)
SELECT 
    MIN(animal) AS first_animal,        -- Returns 'Aardvark' (alphabetically first)
    MIN(price) AS lowest_price,         -- Returns 10.50
    MIN(ts) AS earliest_timestamp,      -- Returns '2024-01-01 00:00:00'
    MIN(flag) AS min_boolean            -- Returns FALSE (FALSE < TRUE in most systems)
FROM diverse_data;
```

## NULL Handling

```sql
-- Demonstrating MIN behavior with NULLs
WITH nullable_data AS (
    SELECT 'A' AS group_id, 100 AS value
    UNION ALL SELECT 'A', NULL
    UNION ALL SELECT 'A', 50
    UNION ALL SELECT 'B', NULL
    UNION ALL SELECT 'B', NULL
    UNION ALL SELECT 'C', 75
)
SELECT 
    group_id,
    MIN(value) AS min_value,                    -- NULLs are ignored
    MIN(COALESCE(value, 1000)) AS min_null_as_1000,  -- Treat NULL as 1000
    COUNT(value) AS non_null_count,
    COUNT(*) AS total_count
FROM nullable_data
GROUP BY group_id
ORDER BY group_id;
```

## Performance Considerations

```sql
-- Optimizing MIN queries with indexes

-- Single column MIN - benefits from index
CREATE INDEX idx_prices ON products(price);
SELECT MIN(price) FROM products;  -- Can use index for quick retrieval

-- MIN with WHERE clause - consider filtered index
CREATE INDEX idx_active_prices ON products(price) WHERE is_active = true;
SELECT MIN(price) FROM products WHERE is_active = true;

-- Grouped MIN - composite index helps
CREATE INDEX idx_category_price ON products(category, price);
SELECT category, MIN(price) 
FROM products 
GROUP BY category;
```

## Edge Cases

```sql
-- Special cases and boundary conditions
WITH edge_cases AS (
    SELECT 0 AS num, '' AS str, DATE '1900-01-01' AS dt
    UNION ALL SELECT -100, ' ', DATE '9999-12-31'  -- Note: space is greater than empty string
    UNION ALL SELECT NULL, NULL, NULL
)
SELECT 
    MIN(num) AS min_number,           -- Returns -100
    MIN(str) AS min_string,           -- Returns '' (empty string)
    MIN(dt) AS min_date,              -- Returns '1900-01-01'
    MIN(ABS(num)) AS min_absolute    -- Returns 0
FROM edge_cases;
```

## Related Functions
- [MAX](./max.md) - Returns the maximum value
- [AVG](./avg.md) - Calculates the average value
- [LEAST](../conditional/least.md) - Returns the smallest value from a list
- [FIRST_VALUE](./first_value.md) - Returns the first value in a group
- [PERCENTILE_CONT](./percentile_cont.md) - Returns a specific percentile value
- [MEDIAN](./median.md) - Returns the middle value