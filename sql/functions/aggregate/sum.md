---
title: SUM
description: Calculates the total sum of numeric values
---

# SUM

## Description
The `SUM` function calculates the total sum of all non-NULL numeric values in a specified column or expression. It's one of the most frequently used aggregate functions for calculating totals, running totals, and cumulative values. NULL values are ignored in the calculation. This function is essential for financial calculations, inventory management, and statistical analysis.

## Syntax
```sql
SUM([DISTINCT] expression)
```

### Parameters
- **DISTINCT** (optional): When specified, sums only unique values
- **expression** (numeric): A column or expression that evaluates to a numeric value

### Return Value
- **Type**: Same numeric type as input (INTEGER, BIGINT, DECIMAL, FLOAT)
- **Description**: The sum of all non-NULL values. Returns NULL if all values are NULL or no rows match.

## Example

### Sample Data
Consider the following `sales_transactions` table tracking daily sales:

| transaction_id | store_id | product_category | quantity | unit_price | total_amount | sale_date  |
|---------------|----------|------------------|----------|------------|--------------|------------|
| 1             | 101      | Electronics      | 2        | 299.99     | 599.98       | 2024-03-01 |
| 2             | 101      | Clothing         | 5        | 49.99      | 249.95       | 2024-03-01 |
| 3             | 102      | Electronics      | 1        | 899.99     | 899.99       | 2024-03-02 |
| 4             | 102      | Food             | 10       | 4.99       | 49.90        | 2024-03-02 |
| 5             | 101      | Clothing         | NULL     | 79.99      | NULL         | 2024-03-03 |
| 6             | 103      | Electronics      | 3        | 199.99     | 599.97       | 2024-03-03 |

### Query
```sql
SELECT 
    SUM(total_amount) AS total_revenue,
    SUM(quantity) AS total_items_sold,
    SUM(DISTINCT unit_price) AS sum_unique_prices,
    COUNT(*) AS transaction_count,
    SUM(total_amount) / COUNT(*) AS avg_transaction_value,
    SUM(CASE WHEN product_category = 'Electronics' THEN total_amount END) AS electronics_revenue
FROM sales_transactions;
```

### Result
| total_revenue | total_items_sold | sum_unique_prices | transaction_count | avg_transaction_value | electronics_revenue |
|--------------|------------------|-------------------|-------------------|--------------------|-------------------|
| 2399.79      | 21               | 1534.94           | 6                 | 399.97             | 2099.94           |

## Common Use Cases

### 1. Running Totals
```sql
-- Calculate cumulative sales by date
SELECT 
    sale_date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY sale_date) AS running_total,
    SUM(total_amount) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_sales
FROM sales_transactions
ORDER BY sale_date;
```

### 2. Group Totals with Percentages
```sql
-- Calculate category totals and percentages
WITH category_totals AS (
    SELECT 
        product_category,
        SUM(total_amount) AS category_total
    FROM sales_transactions
    WHERE total_amount IS NOT NULL
    GROUP BY product_category
)
SELECT 
    product_category,
    category_total,
    SUM(category_total) OVER () AS grand_total,
    ROUND(category_total * 100.0 / SUM(category_total) OVER (), 2) AS percentage
FROM category_totals
ORDER BY category_total DESC;
```

### 3. Period-over-Period Comparison
```sql
-- Compare daily sales with previous day
SELECT 
    sale_date,
    SUM(total_amount) AS daily_total,
    LAG(SUM(total_amount)) OVER (ORDER BY sale_date) AS previous_day,
    SUM(total_amount) - LAG(SUM(total_amount)) OVER (ORDER BY sale_date) AS day_over_day_change
FROM sales_transactions
GROUP BY sale_date
ORDER BY sale_date;
```

### 4. Conditional Sums
```sql
-- Multiple conditional sums in one query
SELECT 
    store_id,
    SUM(total_amount) AS total_sales,
    SUM(CASE WHEN product_category = 'Electronics' THEN total_amount ELSE 0 END) AS electronics_sales,
    SUM(CASE WHEN product_category = 'Clothing' THEN total_amount ELSE 0 END) AS clothing_sales,
    SUM(CASE WHEN sale_date = CURRENT_DATE THEN total_amount ELSE 0 END) AS today_sales
FROM sales_transactions
GROUP BY store_id;
```

### 5. Nested Aggregations
```sql
-- Sum of averages across groups
WITH daily_averages AS (
    SELECT 
        sale_date,
        store_id,
        AVG(total_amount) AS avg_transaction
    FROM sales_transactions
    GROUP BY sale_date, store_id
)
SELECT 
    store_id,
    SUM(avg_transaction) AS sum_of_daily_averages,
    AVG(avg_transaction) AS avg_of_daily_averages
FROM daily_averages
GROUP BY store_id;
```

## NULL Handling

```sql
-- Demonstrating SUM behavior with NULLs
WITH sample_data AS (
    SELECT 100 AS amount
    UNION ALL SELECT 200
    UNION ALL SELECT NULL
    UNION ALL SELECT 150
    UNION ALL SELECT NULL
)
SELECT 
    SUM(amount) AS sum_ignoring_nulls,        -- Returns 450
    SUM(COALESCE(amount, 0)) AS sum_null_as_zero,  -- Returns 450
    COUNT(*) AS total_rows,                   -- Returns 5
    COUNT(amount) AS non_null_count,          -- Returns 3
    SUM(amount) / COUNT(*) AS avg_with_nulls, -- Returns 90 (450/5)
    SUM(amount) / COUNT(amount) AS true_avg   -- Returns 150 (450/3)
FROM sample_data;
```

## DISTINCT Usage

```sql
-- SUM with DISTINCT values
WITH duplicate_values AS (
    SELECT 100 AS value
    UNION ALL SELECT 100
    UNION ALL SELECT 200
    UNION ALL SELECT 200
    UNION ALL SELECT 300
)
SELECT 
    SUM(value) AS sum_all,           -- Returns 900 (100+100+200+200+300)
    SUM(DISTINCT value) AS sum_distinct,  -- Returns 600 (100+200+300)
    COUNT(*) AS count_all,           -- Returns 5
    COUNT(DISTINCT value) AS count_distinct  -- Returns 3
FROM duplicate_values;
```

## Performance Optimization

```sql
-- Indexing for SUM queries
-- Create index on columns used in WHERE and GROUP BY
CREATE INDEX idx_sales_date_store ON sales_transactions(sale_date, store_id, total_amount);

-- This query benefits from the index
SELECT 
    sale_date,
    store_id,
    SUM(total_amount) AS daily_store_total
FROM sales_transactions
WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY sale_date, store_id;

-- Materialized view for frequently calculated sums
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
    sale_date,
    SUM(total_amount) AS daily_total,
    SUM(quantity) AS daily_quantity,
    COUNT(*) AS transaction_count
FROM sales_transactions
GROUP BY sale_date;
```

## Overflow Handling

```sql
-- Handling potential overflow with large sums
-- Cast to larger numeric type if needed
SELECT 
    CAST(SUM(large_value) AS BIGINT) AS safe_sum,
    SUM(CAST(large_value AS DECIMAL(20,2))) AS precise_sum
FROM very_large_numbers;
```

## Related Functions
- [AVG](./avg.md) - Calculates the average value
- [COUNT](./count.md) - Counts rows or non-null values
- [MIN](./min.md) - Returns the minimum value
- [MAX](./max.md) - Returns the maximum value
- [STDDEV](./stddev.md) - Calculates standard deviation
- [VAR](./var.md) - Calculates variance