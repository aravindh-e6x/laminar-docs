---
title: NTH_VALUE
description: Returns the value at a specified position within a window frame
---

# NTH_VALUE

## Description
The `NTH_VALUE` window function returns the value at the nth position within an ordered window frame. Unlike FIRST_VALUE or LAST_VALUE which return extremes, NTH_VALUE allows you to access any specific position, making it useful for finding medians, quartiles, or any specific ranked value within a partition. If the nth position doesn't exist (n is greater than the number of rows), the function returns NULL.

## Syntax
```sql
NTH_VALUE(expression, n) OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
    [frame_clause]
)
```

### Parameters
- **expression** (required): The column or expression to evaluate
- **n** (required): The position within the window frame (must be a positive integer)
- **PARTITION BY** (optional): Divides the result set into partitions
- **ORDER BY** (required): Determines the order for position calculation
- **frame_clause** (optional): Defines the window frame

### Return Value
- **Type**: Same as the expression type
- **Description**: The value at the nth position, or NULL if n exceeds the frame size

## Example

### Sample Data
Consider the following `race_results` table:

| race_id | runner_name | finish_time | position |
|---------|------------|-------------|----------|
| 1       | Alice      | 00:25:30    | 1        |
| 1       | Bob        | 00:26:15    | 2        |
| 1       | Charlie    | 00:26:45    | 3        |
| 1       | Diana      | 00:27:20    | 4        |
| 1       | Eve        | 00:27:55    | 5        |
| 1       | Frank      | 00:28:30    | 6        |
| 2       | George     | 00:24:45    | 1        |
| 2       | Helen      | 00:25:20    | 2        |
| 2       | Ivan       | 00:25:55    | 3        |
| 2       | Julia      | 00:26:30    | 4        |
| 2       | Kevin      | 00:27:05    | 5        |

### Query
```sql
SELECT 
    race_id,
    runner_name,
    finish_time,
    position,
    NTH_VALUE(runner_name, 1) OVER w AS first_place,
    NTH_VALUE(runner_name, 2) OVER w AS second_place,
    NTH_VALUE(runner_name, 3) OVER w AS third_place,
    NTH_VALUE(finish_time, 3) OVER w AS bronze_time,
    -- Median position (3rd of 6 for race 1, 3rd of 5 for race 2)
    NTH_VALUE(finish_time, 3) OVER w AS median_time
FROM race_results
WINDOW w AS (
    PARTITION BY race_id 
    ORDER BY finish_time
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY race_id, position;
```

### Result
| race_id | runner_name | finish_time | position | first_place | second_place | third_place | bronze_time | median_time |
|---------|------------|-------------|----------|-------------|--------------|-------------|-------------|-------------|
| 1       | Alice      | 00:25:30    | 1        | Alice       | Bob          | Charlie     | 00:26:45    | 00:26:45    |
| 1       | Bob        | 00:26:15    | 2        | Alice       | Bob          | Charlie     | 00:26:45    | 00:26:45    |
| 1       | Charlie    | 00:26:45    | 3        | Alice       | Bob          | Charlie     | 00:26:45    | 00:26:45    |
| 1       | Diana      | 00:27:20    | 4        | Alice       | Bob          | Charlie     | 00:26:45    | 00:26:45    |
| 1       | Eve        | 00:27:55    | 5        | Alice       | Bob          | Charlie     | 00:26:45    | 00:26:45    |
| 1       | Frank      | 00:28:30    | 6        | Alice       | Bob          | Charlie     | 00:26:45    | 00:26:45    |
| 2       | George     | 00:24:45    | 1        | George      | Helen        | Ivan        | 00:25:55    | 00:25:55    |
| 2       | Helen      | 00:25:20    | 2        | George      | Helen        | Ivan        | 00:25:55    | 00:25:55    |
| 2       | Ivan       | 00:25:55    | 3        | George      | Helen        | Ivan        | 00:25:55    | 00:25:55    |
| 2       | Julia      | 00:26:30    | 4        | George      | Helen        | Ivan        | 00:25:55    | 00:25:55    |
| 2       | Kevin      | 00:27:05    | 5        | George      | Helen        | Ivan        | 00:25:55    | 00:25:55    |

## Common Use Cases

### 1. Finding Median Values
```sql
-- Calculate median using NTH_VALUE
WITH ranked_salaries AS (
    SELECT 
        department,
        employee_name,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS row_num,
        COUNT(*) OVER (PARTITION BY department) AS dept_count
    FROM employees
)
SELECT 
    department,
    employee_name,
    salary,
    row_num,
    dept_count,
    NTH_VALUE(salary, (dept_count + 1) / 2) OVER (
        PARTITION BY department 
        ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS median_salary_lower,
    NTH_VALUE(salary, (dept_count + 2) / 2) OVER (
        PARTITION BY department 
        ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS median_salary_upper,
    (NTH_VALUE(salary, (dept_count + 1) / 2) OVER (
        PARTITION BY department 
        ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) + NTH_VALUE(salary, (dept_count + 2) / 2) OVER (
        PARTITION BY department 
        ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    )) / 2.0 AS median_salary
FROM ranked_salaries
ORDER BY department, salary;
```

### 2. Quartile Analysis
```sql
-- Calculate quartiles using NTH_VALUE
WITH quartile_data AS (
    SELECT 
        product_category,
        product_name,
        price,
        ROW_NUMBER() OVER (PARTITION BY product_category ORDER BY price) AS row_num,
        COUNT(*) OVER (PARTITION BY product_category) AS category_count
    FROM products
)
SELECT DISTINCT
    product_category,
    category_count AS products_in_category,
    NTH_VALUE(price, 1) OVER w AS min_price,
    NTH_VALUE(price, CAST(category_count * 0.25 AS INT)) OVER w AS q1_price,
    NTH_VALUE(price, CAST(category_count * 0.50 AS INT)) OVER w AS median_price,
    NTH_VALUE(price, CAST(category_count * 0.75 AS INT)) OVER w AS q3_price,
    NTH_VALUE(price, category_count) OVER w AS max_price
FROM quartile_data
WINDOW w AS (
    PARTITION BY product_category 
    ORDER BY price
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY product_category;
```

### 3. Podium Finish Analysis
```sql
-- Track podium finishes in competitions
SELECT 
    competition_date,
    event_type,
    NTH_VALUE(athlete_name, 1) OVER w AS gold_winner,
    NTH_VALUE(score, 1) OVER w AS gold_score,
    NTH_VALUE(athlete_name, 2) OVER w AS silver_winner,
    NTH_VALUE(score, 2) OVER w AS silver_score,
    NTH_VALUE(athlete_name, 3) OVER w AS bronze_winner,
    NTH_VALUE(score, 3) OVER w AS bronze_score,
    COUNT(*) OVER (PARTITION BY competition_date, event_type) AS total_participants
FROM competition_results
WINDOW w AS (
    PARTITION BY competition_date, event_type 
    ORDER BY score DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
GROUP BY competition_date, event_type
ORDER BY competition_date, event_type;
```

### 4. Sales Target Analysis
```sql
-- Find specific ranked performers
WITH sales_rankings AS (
    SELECT 
        region,
        salesperson,
        total_sales,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_sales DESC) AS sales_rank
    FROM quarterly_sales
)
SELECT 
    region,
    COUNT(*) AS team_size,
    NTH_VALUE(salesperson, 1) OVER w AS top_performer,
    NTH_VALUE(total_sales, 1) OVER w AS top_sales,
    NTH_VALUE(salesperson, 5) OVER w AS fifth_performer,
    NTH_VALUE(total_sales, 5) OVER w AS fifth_sales,
    NTH_VALUE(salesperson, 10) OVER w AS tenth_performer,
    NTH_VALUE(total_sales, 10) OVER w AS tenth_sales
FROM sales_rankings
WINDOW w AS (
    PARTITION BY region 
    ORDER BY total_sales DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
GROUP BY region
ORDER BY region;
```

### 5. Time Series Sampling
```sql
-- Sample every nth value from time series
WITH numbered_series AS (
    SELECT 
        timestamp,
        sensor_id,
        temperature,
        ROW_NUMBER() OVER (PARTITION BY sensor_id ORDER BY timestamp) AS row_num
    FROM sensor_readings
    WHERE timestamp >= CURRENT_TIMESTAMP - INTERVAL '24 hours'
)
SELECT 
    sensor_id,
    -- Sample every 10th reading
    NTH_VALUE(timestamp, row_num) OVER w AS sample_time,
    NTH_VALUE(temperature, row_num) OVER w AS sample_temp
FROM numbered_series
WHERE MOD(row_num, 10) = 1  -- Every 10th row
WINDOW w AS (
    PARTITION BY sensor_id 
    ORDER BY timestamp
    ROWS BETWEEN CURRENT ROW AND CURRENT ROW
)
ORDER BY sensor_id, sample_time;
```

## Handling Missing Positions

```sql
-- Demonstrate NULL handling when n exceeds frame size
WITH small_dataset AS (
    SELECT * FROM (VALUES 
        ('A', 1, 100),
        ('A', 2, 200),
        ('A', 3, 300),
        ('B', 1, 150),
        ('B', 2, 250)
    ) AS t(group_id, position, value)
)
SELECT 
    group_id,
    position,
    value,
    NTH_VALUE(value, 1) OVER w AS first_val,
    NTH_VALUE(value, 2) OVER w AS second_val,
    NTH_VALUE(value, 3) OVER w AS third_val,
    NTH_VALUE(value, 4) OVER w AS fourth_val,  -- NULL for group B
    NTH_VALUE(value, 5) OVER w AS fifth_val    -- NULL for both groups
FROM small_dataset
WINDOW w AS (
    PARTITION BY group_id 
    ORDER BY position
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY group_id, position;
```

## Dynamic Position Selection

```sql
-- Use dynamic position based on calculation
WITH dynamic_nth AS (
    SELECT 
        category,
        item_name,
        score,
        -- Calculate position to retrieve (e.g., top 20%)
        CEIL(COUNT(*) OVER (PARTITION BY category) * 0.2) AS target_position,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY score DESC) AS rank
    FROM items
)
SELECT 
    category,
    item_name,
    score,
    rank,
    target_position,
    NTH_VALUE(score, CAST(target_position AS INT)) OVER (
        PARTITION BY category 
        ORDER BY score DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS top_20_pct_threshold
FROM dynamic_nth
ORDER BY category, rank;
```

## Frame Clause Considerations

```sql
-- NTH_VALUE with different frame specifications
SELECT 
    id,
    value,
    -- Default frame
    NTH_VALUE(value, 2) OVER (ORDER BY id) AS nth_default,
    -- Entire partition
    NTH_VALUE(value, 2) OVER (
        ORDER BY id 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS nth_entire,
    -- Limited frame
    NTH_VALUE(value, 2) OVER (
        ORDER BY id 
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS nth_limited
FROM data
ORDER BY id;
```

## Performance Optimization

```sql
-- Optimize NTH_VALUE queries
CREATE INDEX idx_results_race_time ON race_results(race_id, finish_time);

-- Efficient use for large datasets
WITH optimized AS (
    SELECT 
        partition_col,
        value,
        NTH_VALUE(value, 10) OVER (
            PARTITION BY partition_col 
            ORDER BY value
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) AS tenth_value
    FROM large_table
    WHERE filter_condition = true  -- Reduce data first
)
SELECT * FROM optimized
WHERE value <= tenth_value;  -- Top 10 values per partition
```

## Related Functions
- [FIRST_VALUE](./first_value.md) - First value in window frame
- [LAST_VALUE](./last_value.md) - Last value in window frame
- [LAG](./lag.md) - Access previous row at specific offset
- [LEAD](./lead.md) - Access following row at specific offset
- [ROW_NUMBER](./row_number.md) - Assign row numbers