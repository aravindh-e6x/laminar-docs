---
title: LAST_VALUE
description: Returns the last value in an ordered window frame
---

# LAST_VALUE

## Description
The `LAST_VALUE` window function returns the last value in an ordered set of values within a window frame. By default, the frame extends only to the current row, so you often need to explicitly specify the frame clause to include all rows in the partition. This function is useful for finding ending values, calculating ranges, and identifying final states in sequences.

## Syntax
```sql
LAST_VALUE(expression) OVER (
    [PARTITION BY partition_expression, ...]
    [ORDER BY sort_expression [ASC|DESC], ...]
    [frame_clause]
)
```

### Parameters
- **expression** (required): The column or expression to evaluate
- **PARTITION BY** (optional): Divides the result set into partitions
- **ORDER BY** (optional): Determines the order for finding the last value
- **frame_clause** (optional): Defines the window frame (default: RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

### Return Value
- **Type**: Same as the expression type
- **Description**: The last value within the window frame

## Example

### Sample Data
Consider the following `quarterly_results` table:

| quarter    | region | revenue | expenses | profit  |
|------------|--------|---------|----------|---------|
| 2023-Q1    | North  | 500000  | 400000   | 100000  |
| 2023-Q2    | North  | 550000  | 420000   | 130000  |
| 2023-Q3    | North  | 580000  | 430000   | 150000  |
| 2023-Q4    | North  | 620000  | 450000   | 170000  |
| 2024-Q1    | North  | 600000  | 440000   | 160000  |
| 2023-Q1    | South  | 450000  | 380000   | 70000   |
| 2023-Q2    | South  | 480000  | 390000   | 90000   |
| 2023-Q3    | South  | 520000  | 400000   | 120000  |
| 2023-Q4    | South  | 560000  | 410000   | 150000  |
| 2024-Q1    | South  | 540000  | 405000   | 135000  |

### Query
```sql
SELECT 
    quarter,
    region,
    revenue,
    profit,
    -- Default frame: only to current row
    LAST_VALUE(profit) OVER (
        PARTITION BY region ORDER BY quarter
    ) AS last_to_current,
    -- Explicit frame: entire partition
    LAST_VALUE(profit) OVER (
        PARTITION BY region ORDER BY quarter
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS final_profit,
    -- Difference from final value
    LAST_VALUE(profit) OVER (
        PARTITION BY region ORDER BY quarter
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) - profit AS diff_from_final,
    -- YoY comparison
    LAG(profit, 4) OVER (PARTITION BY region ORDER BY quarter) AS same_quarter_last_year
FROM quarterly_results
ORDER BY region, quarter;
```

### Result
| quarter | region | revenue | profit | last_to_current | final_profit | diff_from_final | same_quarter_last_year |
|---------|--------|---------|--------|-----------------|--------------|-----------------|------------------------|
| 2023-Q1 | North  | 500000  | 100000 | 100000          | 160000       | 60000           | NULL                   |
| 2023-Q2 | North  | 550000  | 130000 | 130000          | 160000       | 30000           | NULL                   |
| 2023-Q3 | North  | 580000  | 150000 | 150000          | 160000       | 10000           | NULL                   |
| 2023-Q4 | North  | 620000  | 170000 | 170000          | 160000       | -10000          | NULL                   |
| 2024-Q1 | North  | 600000  | 160000 | 160000          | 160000       | 0               | 100000                 |
| 2023-Q1 | South  | 450000  | 70000  | 70000           | 135000       | 65000           | NULL                   |
| 2023-Q2 | South  | 480000  | 90000  | 90000           | 135000       | 45000           | NULL                   |
| 2023-Q3 | South  | 520000  | 120000 | 120000          | 135000       | 15000           | NULL                   |
| 2023-Q4 | South  | 560000  | 150000 | 150000          | 135000       | -15000          | NULL                   |
| 2024-Q1 | South  | 540000  | 135000 | 135000          | 135000       | 0               | 70000                  |

## Common Use Cases

### 1. Finding Final States
```sql
-- Track final status of orders
WITH order_history AS (
    SELECT 
        order_id,
        status_date,
        order_status,
        LAST_VALUE(order_status) OVER (
            PARTITION BY order_id 
            ORDER BY status_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS final_status,
        LAST_VALUE(status_date) OVER (
            PARTITION BY order_id 
            ORDER BY status_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS final_status_date
    FROM order_status_changes
)
SELECT 
    order_id,
    MIN(status_date) AS first_status_date,
    MAX(status_date) AS last_status_date,
    MAX(final_status) AS final_status,
    COUNT(*) AS status_change_count
FROM order_history
GROUP BY order_id
ORDER BY order_id;
```

### 2. Calculating Ranges
```sql
-- Calculate daily trading ranges
SELECT 
    trading_date,
    symbol,
    opening_price,
    closing_price,
    high_price,
    low_price,
    FIRST_VALUE(opening_price) OVER (
        PARTITION BY symbol, trading_date 
        ORDER BY timestamp
    ) AS day_open,
    LAST_VALUE(closing_price) OVER (
        PARTITION BY symbol, trading_date 
        ORDER BY timestamp
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS day_close,
    high_price - low_price AS daily_range,
    closing_price - opening_price AS daily_change
FROM intraday_prices
ORDER BY symbol, trading_date;
```

### 3. Project Timeline Analysis
```sql
-- Analyze project timelines and delays
SELECT 
    project_id,
    milestone_name,
    planned_date,
    actual_date,
    LAST_VALUE(actual_date) OVER (
        PARTITION BY project_id 
        ORDER BY planned_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS project_completion_date,
    actual_date - planned_date AS delay_days,
    LAST_VALUE(actual_date) OVER (
        PARTITION BY project_id 
        ORDER BY planned_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) - FIRST_VALUE(planned_date) OVER (
        PARTITION BY project_id 
        ORDER BY planned_date
    ) AS total_project_duration
FROM project_milestones
WHERE actual_date IS NOT NULL
ORDER BY project_id, planned_date;
```

### 4. Customer Lifetime Value Endpoint
```sql
-- Track customer value progression to final state
WITH customer_ltv AS (
    SELECT 
        customer_id,
        transaction_date,
        transaction_amount,
        SUM(transaction_amount) OVER (
            PARTITION BY customer_id 
            ORDER BY transaction_date
        ) AS cumulative_value,
        LAST_VALUE(SUM(transaction_amount) OVER (
            PARTITION BY customer_id 
            ORDER BY transaction_date)) OVER (
            PARTITION BY customer_id 
            ORDER BY transaction_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS final_ltv
    FROM customer_transactions
)
SELECT 
    customer_id,
    transaction_date,
    transaction_amount,
    cumulative_value,
    final_ltv,
    ROUND(cumulative_value * 100.0 / final_ltv, 2) AS pct_of_final_ltv
FROM customer_ltv
ORDER BY customer_id, transaction_date;
```

### 5. Inventory Level Tracking
```sql
-- Track inventory levels through the day
SELECT 
    product_id,
    transaction_time,
    transaction_type,
    quantity,
    running_balance,
    FIRST_VALUE(running_balance) OVER (
        PARTITION BY product_id, DATE(transaction_time)
        ORDER BY transaction_time
    ) AS opening_balance,
    LAST_VALUE(running_balance) OVER (
        PARTITION BY product_id, DATE(transaction_time)
        ORDER BY transaction_time
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS closing_balance,
    MAX(running_balance) OVER (
        PARTITION BY product_id, DATE(transaction_time)
    ) AS daily_max,
    MIN(running_balance) OVER (
        PARTITION BY product_id, DATE(transaction_time)
    ) AS daily_min
FROM inventory_transactions
ORDER BY product_id, transaction_time;
```

## Frame Clause Importance

```sql
-- Demonstrating the importance of frame clause with LAST_VALUE
WITH sample_data AS (
    SELECT * FROM (VALUES 
        (1, 100),
        (2, 200),
        (3, 300),
        (4, 400),
        (5, 500)
    ) AS t(id, value)
)
SELECT 
    id,
    value,
    -- Default frame: to current row only
    LAST_VALUE(value) OVER (ORDER BY id) AS last_default,
    -- Explicit: to current row
    LAST_VALUE(value) OVER (
        ORDER BY id 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS last_to_current,
    -- Entire window
    LAST_VALUE(value) OVER (
        ORDER BY id 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_entire_window,
    -- Last 3 rows
    LAST_VALUE(value) OVER (
        ORDER BY id 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS last_3_rows
FROM sample_data
ORDER BY id;
```

## Combining FIRST_VALUE and LAST_VALUE

```sql
-- Complete range analysis
SELECT 
    group_id,
    date,
    value,
    FIRST_VALUE(value) OVER w AS first_val,
    LAST_VALUE(value) OVER w AS last_val,
    value - FIRST_VALUE(value) OVER w AS change_from_first,
    LAST_VALUE(value) OVER w - value AS change_to_last,
    LAST_VALUE(value) OVER w - FIRST_VALUE(value) OVER w AS total_change,
    CASE 
        WHEN value = FIRST_VALUE(value) OVER w THEN 'Start'
        WHEN value = LAST_VALUE(value) OVER w THEN 'End'
        ELSE 'Middle'
    END AS position
FROM time_series_data
WINDOW w AS (
    PARTITION BY group_id 
    ORDER BY date 
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY group_id, date;
```

## Performance Optimization

```sql
-- Optimize LAST_VALUE queries
CREATE INDEX idx_transactions_customer_date ON customer_transactions(customer_id, transaction_date);

-- Efficient frame specification
WITH optimized_query AS (
    SELECT 
        id,
        value,
        date,
        -- More efficient: specify only needed frame
        LAST_VALUE(value) OVER (
            ORDER BY date 
            ROWS BETWEEN CURRENT ROW AND 10 FOLLOWING
        ) AS next_10_last,
        -- Less efficient: entire partition
        LAST_VALUE(value) OVER (
            ORDER BY date 
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS partition_last
    FROM large_table
    WHERE date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT * FROM optimized_query
WHERE value < next_10_last;  -- Values that increase in next 10 rows
```

## Common Pitfall: Default Frame

```sql
-- IMPORTANT: LAST_VALUE default frame gotcha
-- The default frame for LAST_VALUE is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- This means it returns the current row's value, not the last in partition!

-- Wrong: Returns current row value
SELECT value, LAST_VALUE(value) OVER (ORDER BY id) AS last_val
FROM data;

-- Correct: Returns actual last value in partition
SELECT value, LAST_VALUE(value) OVER (
    ORDER BY id 
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
) AS last_val
FROM data;
```

## Related Functions
- [FIRST_VALUE](./first_value.md) - First value in window frame
- [NTH_VALUE](./nth_value.md) - Nth value in window frame
- [LAG](./lag.md) - Access previous row
- [LEAD](./lead.md) - Access following row
- [MAX](../aggregate/max.md) - Maximum value in group