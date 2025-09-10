---
title: COUNT
description: Returns the number of rows or non-null values in a result set
---

# COUNT

## Description
The `COUNT` function returns the number of rows that match a specified condition. It has three main forms: `COUNT(*)` counts all rows including those with NULL values, `COUNT(expression)` counts non-NULL values in a specific column, and `COUNT(DISTINCT expression)` counts unique non-NULL values. This is one of the most fundamental aggregate functions used in SQL for data analysis and reporting.

## Syntax
```sql
COUNT(*)
COUNT([DISTINCT] expression)
```

### Parameters
- **\*** : Counts all rows, including rows with NULL values
- **DISTINCT** (optional): When specified, counts only unique non-NULL values
- **expression**: A column or expression to count non-NULL values

### Return Value
- **Type**: BIGINT
- **Description**: The number of rows or non-NULL values. Returns 0 if no rows match, never returns NULL.

## Example

### Sample Data
Consider the following `orders` table tracking customer orders:

| order_id | customer_id | product_id | quantity | price | order_status | order_date |
|----------|------------|------------|----------|-------|--------------|------------|
| 1        | 101        | 501        | 2        | 29.99 | completed    | 2024-03-01 |
| 2        | 102        | 502        | 1        | 49.99 | completed    | 2024-03-02 |
| 3        | 101        | 501        | 3        | 29.99 | pending      | 2024-03-03 |
| 4        | 103        | NULL       | 1        | 19.99 | cancelled    | 2024-03-04 |
| 5        | 102        | 503        | NULL     | 39.99 | completed    | 2024-03-05 |
| 6        | 104        | 502        | 2        | 49.99 | completed    | 2024-03-06 |

### Query
```sql
SELECT 
    COUNT(*) AS total_rows,
    COUNT(product_id) AS orders_with_product,
    COUNT(quantity) AS orders_with_quantity,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(DISTINCT product_id) AS unique_products,
    COUNT(DISTINCT order_status) AS unique_statuses
FROM orders;
```

### Result
| total_rows | orders_with_product | orders_with_quantity | unique_customers | unique_products | unique_statuses |
|------------|-------------------|---------------------|-----------------|-----------------|-----------------|
| 6          | 5                 | 5                   | 4               | 3               | 3               |

## Common Use Cases

### 1. Counting Rows with Conditions
```sql
-- Count orders by status
SELECT 
    order_status,
    COUNT(*) AS order_count,
    COUNT(product_id) AS valid_product_orders,
    ROUND(COUNT(product_id) * 100.0 / COUNT(*), 2) AS valid_product_pct
FROM orders
GROUP BY order_status
ORDER BY order_count DESC;
```

### 2. Distinct Value Counting
```sql
-- Count unique customers and their order patterns
SELECT 
    DATE_TRUNC('month', order_date) AS month,
    COUNT(*) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(*) * 1.0 / COUNT(DISTINCT customer_id) AS avg_orders_per_customer
FROM orders
WHERE order_status = 'completed'
GROUP BY DATE_TRUNC('month', order_date);
```

### 3. Conditional Counting
```sql
-- Count orders based on different conditions
SELECT 
    customer_id,
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN order_status = 'completed' THEN 1 END) AS completed_orders,
    COUNT(CASE WHEN order_status = 'cancelled' THEN 1 END) AS cancelled_orders,
    COUNT(CASE WHEN price > 40 THEN 1 END) AS high_value_orders
FROM orders
GROUP BY customer_id;
```

### 4. Window Function Usage
```sql
-- Running count of orders per customer
SELECT 
    order_id,
    customer_id,
    order_date,
    COUNT(*) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_order_count,
    COUNT(*) OVER (PARTITION BY customer_id) AS total_customer_orders
FROM orders
ORDER BY customer_id, order_date;
```

### 5. Existence Checking
```sql
-- Find customers with multiple orders
SELECT 
    customer_id,
    COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 1;
```

## NULL Handling

```sql
-- Demonstrating COUNT behavior with NULLs
WITH sample_data AS (
    SELECT 1 AS id, 'A' AS category, 100 AS value
    UNION ALL SELECT 2, 'B', 200
    UNION ALL SELECT 3, NULL, 300
    UNION ALL SELECT 4, 'A', NULL
    UNION ALL SELECT 5, 'B', NULL
)
SELECT 
    COUNT(*) AS count_all_rows,           -- Returns 5 (counts all rows)
    COUNT(category) AS count_category,     -- Returns 4 (excludes NULL categories)
    COUNT(value) AS count_value,          -- Returns 3 (excludes NULL values)
    COUNT(DISTINCT category) AS unique_categories,  -- Returns 2 (A and B, excludes NULL)
    COUNT(1) AS count_constant            -- Returns 5 (same as COUNT(*))
FROM sample_data;
```

## Performance Considerations

```sql
-- COUNT(*) vs COUNT(column) performance
-- COUNT(*) is typically faster as it doesn't check for NULL values

-- Fast: counts all rows
SELECT COUNT(*) FROM large_table;

-- Slower: must check each value for NULL
SELECT COUNT(nullable_column) FROM large_table;

-- Use COUNT(*) when you need total row count
-- Use COUNT(column) only when NULL checking is required
```

## Related Functions
- [SUM](./sum.md) - Calculates the sum of values
- [AVG](./avg.md) - Calculates the average of values
- [MIN](./min.md) - Returns the minimum value
- [MAX](./max.md) - Returns the maximum value
- [COUNT DISTINCT](./count_distinct.md) - Counts unique values
- [APPROX_DISTINCT](./approx_distinct.md) - Approximate count of distinct values