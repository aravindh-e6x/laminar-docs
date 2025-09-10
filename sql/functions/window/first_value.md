---
title: FIRST_VALUE
description: Returns the first value in an ordered window frame
---

# FIRST_VALUE

## Description
The `FIRST_VALUE` window function returns the first value in an ordered set of values within a window frame. Unlike LAG which looks at a specific offset, FIRST_VALUE always returns the first value based on the ORDER BY clause within the current window frame. This function is useful for comparisons with baseline values, calculating ranges, and identifying starting points in sequences.

## Syntax
```sql
FIRST_VALUE(expression) OVER (
    [PARTITION BY partition_expression, ...]
    [ORDER BY sort_expression [ASC|DESC], ...]
    [frame_clause]
)
```

### Parameters
- **expression** (required): The column or expression to evaluate
- **PARTITION BY** (optional): Divides the result set into partitions
- **ORDER BY** (optional): Determines the order for finding the first value
- **frame_clause** (optional): Defines the window frame (default: RANGE UNBOUNDED PRECEDING)

### Return Value
- **Type**: Same as the expression type
- **Description**: The first value within the window frame

## Example

### Sample Data
Consider the following `daily_temperatures` table:

| date       | city        | temperature | humidity |
|------------|-------------|-------------|----------|
| 2024-03-01 | New York    | 45          | 65       |
| 2024-03-02 | New York    | 48          | 70       |
| 2024-03-03 | New York    | 52          | 68       |
| 2024-03-04 | New York    | 50          | 72       |
| 2024-03-05 | New York    | 55          | 60       |
| 2024-03-01 | Los Angeles | 68          | 55       |
| 2024-03-02 | Los Angeles | 70          | 58       |
| 2024-03-03 | Los Angeles | 72          | 52       |
| 2024-03-04 | Los Angeles | 71          | 54       |
| 2024-03-05 | Los Angeles | 69          | 56       |

### Query
```sql
SELECT 
    date,
    city,
    temperature,
    FIRST_VALUE(temperature) OVER (
        PARTITION BY city ORDER BY date
    ) AS first_temp,
    temperature - FIRST_VALUE(temperature) OVER (
        PARTITION BY city ORDER BY date
    ) AS change_from_start,
    FIRST_VALUE(temperature) OVER (
        PARTITION BY city ORDER BY temperature DESC
    ) AS max_temp_in_period,
    FIRST_VALUE(date) OVER (
        PARTITION BY city ORDER BY temperature DESC
    ) AS date_of_max_temp
FROM daily_temperatures
ORDER BY city, date;
```

### Result
| date       | city        | temperature | first_temp | change_from_start | max_temp_in_period | date_of_max_temp |
|------------|-------------|-------------|------------|-------------------|-------------------|------------------|
| 2024-03-01 | Los Angeles | 68          | 68         | 0                 | 72                | 2024-03-03       |
| 2024-03-02 | Los Angeles | 70          | 68         | 2                 | 72                | 2024-03-03       |
| 2024-03-03 | Los Angeles | 72          | 68         | 4                 | 72                | 2024-03-03       |
| 2024-03-04 | Los Angeles | 71          | 68         | 3                 | 72                | 2024-03-03       |
| 2024-03-05 | Los Angeles | 69          | 68         | 1                 | 72                | 2024-03-03       |
| 2024-03-01 | New York    | 45          | 45         | 0                 | 55                | 2024-03-05       |
| 2024-03-02 | New York    | 48          | 45         | 3                 | 55                | 2024-03-05       |
| 2024-03-03 | New York    | 52          | 45         | 7                 | 55                | 2024-03-05       |
| 2024-03-04 | New York    | 50          | 45         | 5                 | 55                | 2024-03-05       |
| 2024-03-05 | New York    | 55          | 45         | 10                | 55                | 2024-03-05       |

## Common Use Cases

### 1. Calculating Cumulative Growth
```sql
-- Track growth from initial value
SELECT 
    month,
    revenue,
    FIRST_VALUE(revenue) OVER (ORDER BY month) AS initial_revenue,
    revenue - FIRST_VALUE(revenue) OVER (ORDER BY month) AS absolute_growth,
    ROUND((revenue - FIRST_VALUE(revenue) OVER (ORDER BY month)) * 100.0 / 
          FIRST_VALUE(revenue) OVER (ORDER BY month), 2) AS growth_percentage,
    ROUND(revenue * 100.0 / FIRST_VALUE(revenue) OVER (ORDER BY month), 2) AS index_value
FROM monthly_metrics
ORDER BY month;
```

### 2. Session Start Analysis
```sql
-- Identify session start times and durations
WITH user_sessions AS (
    SELECT 
        session_id,
        user_id,
        event_timestamp,
        event_type,
        FIRST_VALUE(event_timestamp) OVER (
            PARTITION BY session_id ORDER BY event_timestamp
        ) AS session_start,
        FIRST_VALUE(event_type) OVER (
            PARTITION BY session_id ORDER BY event_timestamp
        ) AS first_action,
        LAST_VALUE(event_timestamp) OVER (
            PARTITION BY session_id ORDER BY event_timestamp
            RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS session_end
    FROM user_events
)
SELECT 
    session_id,
    user_id,
    session_start,
    session_end,
    first_action,
    EXTRACT(EPOCH FROM session_end - session_start) AS session_duration_seconds,
    COUNT(*) AS event_count
FROM user_sessions
GROUP BY session_id, user_id, session_start, session_end, first_action
ORDER BY session_start;
```

### 3. Price Range Analysis
```sql
-- Analyze daily price ranges
SELECT 
    trading_date,
    symbol,
    opening_price,
    closing_price,
    high_price,
    low_price,
    FIRST_VALUE(opening_price) OVER (
        PARTITION BY symbol 
        ORDER BY trading_date
        ROWS 29 PRECEDING
    ) AS month_opening,
    high_price - low_price AS daily_range,
    FIRST_VALUE(high_price) OVER (
        PARTITION BY symbol, trading_date 
        ORDER BY high_price DESC
    ) AS day_high,
    FIRST_VALUE(low_price) OVER (
        PARTITION BY symbol, trading_date 
        ORDER BY low_price ASC
    ) AS day_low
FROM stock_prices
WHERE trading_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY symbol, trading_date;
```

### 4. Employee Tenure Tracking
```sql
-- Track employee milestones from hire date
SELECT 
    employee_id,
    employee_name,
    department,
    hire_date,
    FIRST_VALUE(hire_date) OVER (
        PARTITION BY department ORDER BY hire_date
    ) AS first_hire_in_dept,
    hire_date - FIRST_VALUE(hire_date) OVER (
        PARTITION BY department ORDER BY hire_date
    ) AS days_after_first_hire,
    FIRST_VALUE(employee_name) OVER (
        PARTITION BY department ORDER BY hire_date
    ) AS most_senior_employee,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS highest_dept_salary
FROM employees
WHERE employment_status = 'Active'
ORDER BY department, hire_date;
```

### 5. Customer Journey Tracking
```sql
-- Analyze customer conversion paths
WITH customer_journey AS (
    SELECT 
        customer_id,
        touchpoint_date,
        touchpoint_type,
        conversion_value,
        FIRST_VALUE(touchpoint_type) OVER (
            PARTITION BY customer_id ORDER BY touchpoint_date
        ) AS first_touchpoint,
        FIRST_VALUE(touchpoint_date) OVER (
            PARTITION BY customer_id ORDER BY touchpoint_date
        ) AS first_interaction_date,
        touchpoint_date - FIRST_VALUE(touchpoint_date) OVER (
            PARTITION BY customer_id ORDER BY touchpoint_date
        ) AS days_since_first_touch
    FROM marketing_touchpoints
)
SELECT 
    first_touchpoint,
    COUNT(DISTINCT customer_id) AS customer_count,
    AVG(days_since_first_touch) AS avg_days_to_convert,
    SUM(conversion_value) AS total_conversion_value
FROM customer_journey
WHERE conversion_value > 0
GROUP BY first_touchpoint
ORDER BY total_conversion_value DESC;
```

## Frame Clause Examples

```sql
-- Different frame specifications with FIRST_VALUE
SELECT 
    date,
    value,
    -- Default: from start to current row
    FIRST_VALUE(value) OVER (ORDER BY date) AS first_from_start,
    
    -- Explicit unbounded preceding to current row
    FIRST_VALUE(value) OVER (
        ORDER BY date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS first_to_current,
    
    -- Last 7 days window
    FIRST_VALUE(value) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS first_last_7_rows,
    
    -- Entire partition
    FIRST_VALUE(value) OVER (
        ORDER BY date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_in_partition
FROM time_series
ORDER BY date;
```

## Combining with Other Window Functions

```sql
-- Use FIRST_VALUE with other window functions
SELECT 
    product_id,
    sale_date,
    quantity,
    price,
    FIRST_VALUE(price) OVER w AS initial_price,
    LAST_VALUE(price) OVER w AS final_price,
    MIN(price) OVER w AS min_price,
    MAX(price) OVER w AS max_price,
    price - FIRST_VALUE(price) OVER w AS price_change_from_start,
    CASE 
        WHEN price = FIRST_VALUE(price) OVER w THEN 'At Initial'
        WHEN price > FIRST_VALUE(price) OVER w THEN 'Above Initial'
        ELSE 'Below Initial'
    END AS price_position
FROM product_sales
WINDOW w AS (PARTITION BY product_id ORDER BY sale_date)
ORDER BY product_id, sale_date;
```

## Finding First Occurrence

```sql
-- Find first occurrence of specific conditions
WITH first_occurrences AS (
    SELECT 
        customer_id,
        order_date,
        order_status,
        order_amount,
        FIRST_VALUE(CASE WHEN order_status = 'Completed' THEN order_date END) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS first_completed_order,
        FIRST_VALUE(CASE WHEN order_amount > 100 THEN order_date END) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS first_large_order
    FROM orders
)
SELECT 
    customer_id,
    MIN(order_date) AS first_order,
    MIN(first_completed_order) AS first_completion,
    MIN(first_large_order) AS first_large_purchase,
    COUNT(*) AS total_orders
FROM first_occurrences
GROUP BY customer_id
ORDER BY customer_id;
```

## Performance Considerations

```sql
-- Optimize FIRST_VALUE queries
CREATE INDEX idx_temps_city_date ON daily_temperatures(city, date);
CREATE INDEX idx_sales_product_date ON product_sales(product_id, sale_date);

-- Efficient use with proper framing
SELECT 
    date,
    value,
    -- More efficient: smaller frame
    FIRST_VALUE(value) OVER (
        ORDER BY date 
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS first_30_days,
    -- Less efficient: entire partition
    FIRST_VALUE(value) OVER (
        ORDER BY date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_all_time
FROM large_dataset
WHERE date >= CURRENT_DATE - INTERVAL '90 days';
```

## Related Functions
- [LAST_VALUE](./last_value.md) - Last value in window frame
- [NTH_VALUE](./nth_value.md) - Nth value in window frame
- [LAG](./lag.md) - Access previous row
- [LEAD](./lead.md) - Access following row
- [MIN](../aggregate/min.md) - Minimum value in group