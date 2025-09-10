---
title: LAG
description: Accesses data from a previous row in the result set
---

# LAG

## Description
The `LAG` window function provides access to a row at a specified physical offset before the current row within a partition. It allows you to compare values between the current row and previous rows without using self-joins. This function is essential for calculating differences, detecting changes, and analyzing trends in time-series data.

## Syntax
```sql
LAG(expression [, offset [, default]]) OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
)
```

### Parameters
- **expression** (required): The column or expression to retrieve from the previous row
- **offset** (optional): Number of rows back from the current row (default: 1)
- **default** (optional): Value to return when the offset goes beyond the partition boundary (default: NULL)
- **PARTITION BY** (optional): Divides the result set into partitions
- **ORDER BY** (required): Determines the order for LAG calculation

### Return Value
- **Type**: Same as the expression type
- **Description**: Value from the previous row, or default/NULL if no previous row exists

## Example

### Sample Data
Consider the following `stock_prices` table:

| date       | symbol | closing_price | volume    |
|------------|--------|--------------|-----------|
| 2024-03-11 | AAPL   | 172.50       | 45000000  |
| 2024-03-12 | AAPL   | 173.25       | 48000000  |
| 2024-03-13 | AAPL   | 171.80       | 52000000  |
| 2024-03-14 | AAPL   | 174.10       | 47000000  |
| 2024-03-15 | AAPL   | 175.50       | 55000000  |
| 2024-03-11 | GOOGL  | 138.20       | 22000000  |
| 2024-03-12 | GOOGL  | 139.50       | 24000000  |
| 2024-03-13 | GOOGL  | 138.90       | 21000000  |
| 2024-03-14 | GOOGL  | 140.25       | 26000000  |
| 2024-03-15 | GOOGL  | 141.00       | 28000000  |

### Query
```sql
SELECT 
    date,
    symbol,
    closing_price,
    LAG(closing_price) OVER (PARTITION BY symbol ORDER BY date) AS prev_price,
    closing_price - LAG(closing_price) OVER (PARTITION BY symbol ORDER BY date) AS daily_change,
    ROUND((closing_price - LAG(closing_price) OVER (PARTITION BY symbol ORDER BY date)) / 
          LAG(closing_price) OVER (PARTITION BY symbol ORDER BY date) * 100, 2) AS pct_change,
    LAG(closing_price, 2) OVER (PARTITION BY symbol ORDER BY date) AS price_2_days_ago,
    LAG(volume, 1, 0) OVER (PARTITION BY symbol ORDER BY date) AS prev_volume
FROM stock_prices
ORDER BY symbol, date;
```

### Result
| date       | symbol | closing_price | prev_price | daily_change | pct_change | price_2_days_ago | prev_volume |
|------------|--------|--------------|------------|--------------|------------|------------------|-------------|
| 2024-03-11 | AAPL   | 172.50       | NULL       | NULL         | NULL       | NULL             | 0           |
| 2024-03-12 | AAPL   | 173.25       | 172.50     | 0.75         | 0.43       | NULL             | 45000000    |
| 2024-03-13 | AAPL   | 171.80       | 173.25     | -1.45        | -0.84      | 172.50           | 48000000    |
| 2024-03-14 | AAPL   | 174.10       | 171.80     | 2.30         | 1.34       | 173.25           | 52000000    |
| 2024-03-15 | AAPL   | 175.50       | 174.10     | 1.40         | 0.80       | 171.80           | 47000000    |
| 2024-03-11 | GOOGL  | 138.20       | NULL       | NULL         | NULL       | NULL             | 0           |
| 2024-03-12 | GOOGL  | 139.50       | 138.20     | 1.30         | 0.94       | NULL             | 22000000    |
| 2024-03-13 | GOOGL  | 138.90       | 139.50     | -0.60        | -0.43      | 138.20           | 24000000    |
| 2024-03-14 | GOOGL  | 140.25       | 138.90     | 1.35         | 0.97       | 139.50           | 21000000    |
| 2024-03-15 | GOOGL  | 141.00       | 140.25     | 0.75         | 0.53       | 138.90           | 26000000    |

## Common Use Cases

### 1. Year-over-Year Comparison
```sql
-- Compare current month with same month last year
WITH monthly_sales AS (
    SELECT 
        DATE_TRUNC('month', sale_date) AS month,
        SUM(amount) AS total_sales
    FROM sales
    GROUP BY DATE_TRUNC('month', sale_date)
)
SELECT 
    month,
    total_sales,
    LAG(total_sales, 12) OVER (ORDER BY month) AS same_month_last_year,
    total_sales - LAG(total_sales, 12) OVER (ORDER BY month) AS yoy_change,
    ROUND((total_sales - LAG(total_sales, 12) OVER (ORDER BY month)) / 
          NULLIF(LAG(total_sales, 12) OVER (ORDER BY month), 0) * 100, 2) AS yoy_growth_pct
FROM monthly_sales
ORDER BY month DESC;
```

### 2. Session Analysis
```sql
-- Identify new sessions based on time gaps
WITH user_events AS (
    SELECT 
        user_id,
        event_timestamp,
        LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp) AS prev_event_time,
        EXTRACT(EPOCH FROM event_timestamp - 
                LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp)) AS seconds_since_last
    FROM events
)
SELECT 
    user_id,
    event_timestamp,
    prev_event_time,
    seconds_since_last,
    CASE 
        WHEN prev_event_time IS NULL THEN 'New Session'
        WHEN seconds_since_last > 1800 THEN 'New Session'  -- 30 minute timeout
        ELSE 'Same Session'
    END AS session_status
FROM user_events
ORDER BY user_id, event_timestamp;
```

### 3. Streak Detection
```sql
-- Find winning/losing streaks
WITH game_results AS (
    SELECT 
        player_id,
        game_date,
        result,
        LAG(result) OVER (PARTITION BY player_id ORDER BY game_date) AS prev_result,
        CASE 
            WHEN result = LAG(result) OVER (PARTITION BY player_id ORDER BY game_date) 
            THEN 0 
            ELSE 1 
        END AS streak_change
    FROM games
)
SELECT 
    player_id,
    game_date,
    result,
    SUM(streak_change) OVER (PARTITION BY player_id ORDER BY game_date) AS streak_id,
    ROW_NUMBER() OVER (PARTITION BY player_id, 
                       SUM(streak_change) OVER (PARTITION BY player_id ORDER BY game_date) 
                       ORDER BY game_date) AS streak_length
FROM game_results
ORDER BY player_id, game_date;
```

### 4. Inventory Movement Tracking
```sql
-- Track inventory changes and calculate running balance
SELECT 
    product_id,
    transaction_date,
    transaction_type,
    quantity,
    LAG(running_balance, 1, 0) OVER (PARTITION BY product_id ORDER BY transaction_date) AS prev_balance,
    CASE transaction_type
        WHEN 'IN' THEN LAG(running_balance, 1, 0) OVER (PARTITION BY product_id ORDER BY transaction_date) + quantity
        WHEN 'OUT' THEN LAG(running_balance, 1, 0) OVER (PARTITION BY product_id ORDER BY transaction_date) - quantity
    END AS new_balance,
    quantity - LAG(quantity, 1, 0) OVER (PARTITION BY product_id ORDER BY transaction_date) AS quantity_change
FROM inventory_transactions
ORDER BY product_id, transaction_date;
```

### 5. Customer Behavior Changes
```sql
-- Detect changes in customer purchase patterns
WITH purchase_patterns AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', purchase_date) AS month,
        COUNT(*) AS purchase_count,
        SUM(amount) AS total_spent,
        AVG(amount) AS avg_order_value
    FROM purchases
    GROUP BY customer_id, DATE_TRUNC('month', purchase_date)
)
SELECT 
    customer_id,
    month,
    purchase_count,
    LAG(purchase_count) OVER (PARTITION BY customer_id ORDER BY month) AS prev_month_count,
    total_spent,
    LAG(total_spent) OVER (PARTITION BY customer_id ORDER BY month) AS prev_month_spent,
    CASE 
        WHEN LAG(purchase_count) OVER (PARTITION BY customer_id ORDER BY month) = 0 
             AND purchase_count > 0 THEN 'Reactivated'
        WHEN purchase_count = 0 
             AND LAG(purchase_count) OVER (PARTITION BY customer_id ORDER BY month) > 0 THEN 'Churned'
        WHEN purchase_count > LAG(purchase_count) OVER (PARTITION BY customer_id ORDER BY month) * 1.5 THEN 'Increasing'
        WHEN purchase_count < LAG(purchase_count) OVER (PARTITION BY customer_id ORDER BY month) * 0.5 THEN 'Decreasing'
        ELSE 'Stable'
    END AS trend
FROM purchase_patterns
ORDER BY customer_id, month;
```

## Moving Averages with LAG

```sql
-- Calculate various moving averages
SELECT 
    date,
    value,
    -- Simple 3-day moving average using LAG
    (value + 
     LAG(value, 1) OVER (ORDER BY date) + 
     LAG(value, 2) OVER (ORDER BY date)) / 3.0 AS ma_3_day,
    -- 5-day moving average
    (value + 
     LAG(value, 1) OVER (ORDER BY date) + 
     LAG(value, 2) OVER (ORDER BY date) +
     LAG(value, 3) OVER (ORDER BY date) +
     LAG(value, 4) OVER (ORDER BY date)) / 5.0 AS ma_5_day
FROM time_series_data
ORDER BY date;
```

## Gap Detection

```sql
-- Find gaps in sequential data
WITH numbered_data AS (
    SELECT 
        id,
        sequence_number,
        LAG(sequence_number) OVER (ORDER BY sequence_number) AS prev_sequence
    FROM sequences
)
SELECT 
    prev_sequence + 1 AS gap_start,
    sequence_number - 1 AS gap_end,
    sequence_number - prev_sequence - 1 AS gap_size
FROM numbered_data
WHERE sequence_number - prev_sequence > 1
ORDER BY gap_start;
```

## Multiple LAG Comparisons

```sql
-- Compare with multiple previous periods
SELECT 
    month,
    sales,
    LAG(sales, 1) OVER (ORDER BY month) AS prev_1_month,
    LAG(sales, 3) OVER (ORDER BY month) AS prev_3_months,
    LAG(sales, 6) OVER (ORDER BY month) AS prev_6_months,
    LAG(sales, 12) OVER (ORDER BY month) AS prev_12_months,
    sales - LAG(sales, 1) OVER (ORDER BY month) AS mom_change,
    sales - LAG(sales, 3) OVER (ORDER BY month) AS qoq_change,
    sales - LAG(sales, 12) OVER (ORDER BY month) AS yoy_change
FROM monthly_metrics
ORDER BY month DESC;
```

## Performance Considerations

```sql
-- Optimize LAG queries with appropriate indexes
CREATE INDEX idx_stock_symbol_date ON stock_prices(symbol, date);
CREATE INDEX idx_events_user_timestamp ON events(user_id, event_timestamp);

-- Efficient LAG usage in large datasets
WITH ranked_data AS (
    SELECT 
        date,
        value,
        LAG(value) OVER (ORDER BY date) AS prev_value
    FROM large_table
    WHERE date >= CURRENT_DATE - INTERVAL '30 days'  -- Limit data first
)
SELECT * FROM ranked_data
WHERE value > prev_value * 1.1;  -- 10% increase detection
```

## Related Functions
- [LEAD](./lead.md) - Access following rows
- [FIRST_VALUE](./first_value.md) - First value in window
- [LAST_VALUE](./last_value.md) - Last value in window
- [NTH_VALUE](./nth_value.md) - Nth value in window
- [ROW_NUMBER](./row_number.md) - Sequential row numbering