---
title: MAX
description: Returns the maximum value from a set of values
---

# MAX

## Description
The `MAX` function returns the largest value from a set of values in a specified column or expression. It works with numeric, string, date/time, and other comparable data types. For strings, MAX returns the last value in alphabetical order. NULL values are ignored in the calculation. This function is essential for finding peak values, latest dates, or boundary conditions in your data.

## Syntax
```sql
MAX([DISTINCT] expression)
```

### Parameters
- **DISTINCT** (optional): Has no effect for MAX (included for SQL standard compatibility)
- **expression**: A column or expression that evaluates to a comparable value

### Return Value
- **Type**: Same as the input expression type
- **Description**: The maximum value from all non-NULL values. Returns NULL if all values are NULL or no rows match.

## Example

### Sample Data
Consider the following `product_inventory` table tracking product stock levels:

| product_id | product_name    | category     | stock_quantity | price  | last_restock_date |
|------------|----------------|--------------|----------------|--------|-------------------|
| 1          | Laptop Pro     | Electronics  | 45             | 1299.99| 2024-03-15       |
| 2          | Office Chair   | Furniture    | 120            | 299.99 | 2024-03-10       |
| 3          | Desk Lamp      | Furniture    | 89             | 49.99  | 2024-03-18       |
| 4          | Wireless Mouse | Electronics  | 200            | 29.99  | 2024-03-20       |
| 5          | Monitor 4K     | Electronics  | NULL           | 599.99 | 2024-03-12       |
| 6          | Notebook       | Stationery   | 500            | 4.99   | 2024-03-19       |

### Query
```sql
SELECT 
    MAX(stock_quantity) AS max_stock,
    MAX(price) AS highest_price,
    MAX(last_restock_date) AS most_recent_restock,
    MAX(product_name) AS last_product_alphabetically,
    COUNT(*) AS total_products,
    MAX(CASE WHEN category = 'Electronics' THEN price END) AS max_electronics_price
FROM product_inventory;
```

### Result
| max_stock | highest_price | most_recent_restock | last_product_alphabetically | total_products | max_electronics_price |
|-----------|--------------|--------------------|-----------------------------|----------------|--------------------|
| 500       | 1299.99      | 2024-03-20         | Wireless Mouse              | 6              | 1299.99            |

## Common Use Cases

### 1. Finding Latest Records
```sql
-- Find the most recent order for each customer
SELECT 
    customer_id,
    MAX(order_date) AS last_order_date,
    MAX(order_id) AS latest_order_id,
    COUNT(*) AS total_orders
FROM orders
GROUP BY customer_id
HAVING MAX(order_date) >= CURRENT_DATE - INTERVAL '30 days';
```

### 2. Price Analysis
```sql
-- Analyze price ranges by category
SELECT 
    category,
    MAX(price) AS max_price,
    MIN(price) AS min_price,
    MAX(price) - MIN(price) AS price_range,
    AVG(price) AS avg_price,
    COUNT(*) AS product_count
FROM products
WHERE is_active = true
GROUP BY category
ORDER BY max_price DESC;
```

### 3. Time-Based Maximum
```sql
-- Find peak values per time period
SELECT 
    DATE_TRUNC('hour', timestamp) AS hour,
    MAX(cpu_usage) AS peak_cpu,
    MAX(memory_usage) AS peak_memory,
    MAX(network_throughput) AS peak_network,
    AVG(cpu_usage) AS avg_cpu
FROM system_metrics
WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '24 hours'
GROUP BY DATE_TRUNC('hour', timestamp)
ORDER BY hour;
```

### 4. Window Function Usage
```sql
-- Calculate running maximum and compare to current value
SELECT 
    date,
    sales_amount,
    MAX(sales_amount) OVER (ORDER BY date) AS running_max,
    MAX(sales_amount) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS max_last_7_days,
    sales_amount = MAX(sales_amount) OVER (ORDER BY date) AS is_new_record
FROM daily_sales
ORDER BY date;
```

### 5. Correlated Subqueries
```sql
-- Find products priced at category maximum
SELECT 
    p1.product_id,
    p1.product_name,
    p1.category,
    p1.price
FROM products p1
WHERE p1.price = (
    SELECT MAX(p2.price)
    FROM products p2
    WHERE p2.category = p1.category
);
```

## Data Type Examples

```sql
-- MAX with different data types
WITH mixed_data AS (
    SELECT 'Apple' AS text_col, 100 AS num_col, DATE '2024-01-15' AS date_col, TIME '14:30:00' AS time_col
    UNION ALL SELECT 'Zebra', 50, DATE '2024-03-20', TIME '09:15:00'
    UNION ALL SELECT 'Banana', 200, DATE '2024-02-10', TIME '18:45:00'
    UNION ALL SELECT NULL, 150, DATE '2024-03-25', TIME '12:00:00'
)
SELECT 
    MAX(text_col) AS max_text,      -- Returns 'Zebra' (last alphabetically)
    MAX(num_col) AS max_number,     -- Returns 200
    MAX(date_col) AS latest_date,   -- Returns '2024-03-25'
    MAX(time_col) AS latest_time    -- Returns '18:45:00'
FROM mixed_data;
```

## NULL Handling

```sql
-- Demonstrating MAX behavior with NULLs
WITH sample_data AS (
    SELECT 1 AS id, 100 AS value
    UNION ALL SELECT 2, 200
    UNION ALL SELECT 3, NULL
    UNION ALL SELECT 4, 150
    UNION ALL SELECT 5, NULL
)
SELECT 
    MAX(value) AS max_value,           -- Returns 200 (NULLs ignored)
    MAX(COALESCE(value, 0)) AS max_with_null_as_zero,  -- Treats NULL as 0
    COUNT(*) AS total_rows,
    COUNT(value) AS non_null_values
FROM sample_data;
```

## Performance Optimization

```sql
-- Using indexes for MAX optimization
-- Create index on frequently queried MAX columns
CREATE INDEX idx_orders_date ON orders(order_date);

-- This query can use the index for fast MAX retrieval
SELECT MAX(order_date) FROM orders;

-- For grouped MAX, consider composite indexes
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- This grouped query benefits from the composite index
SELECT customer_id, MAX(order_date) 
FROM orders 
GROUP BY customer_id;
```

## Related Functions
- [MIN](./min.md) - Returns the minimum value
- [AVG](./avg.md) - Calculates the average value
- [SUM](./sum.md) - Calculates the sum of values
- [GREATEST](../conditional/greatest.md) - Returns the greatest value from a list
- [FIRST_VALUE](./first_value.md) - Returns the first value in a group
- [LAST_VALUE](./last_value.md) - Returns the last value in a group