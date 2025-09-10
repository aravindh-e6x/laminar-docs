---
sidebar_position: 5
---

# Window Functions

Window functions perform calculations across a set of rows that are related to the current row, without grouping the result set.

## Window Function Syntax

```sql
function_name() OVER (
    [PARTITION BY partition_expression]
    [ORDER BY sort_expression]
    [frame_clause]
)
```

## Ranking Functions

### row_number()
Assigns sequential integers to rows within a partition.
```sql
SELECT 
    name,
    department,
    salary,
    row_number() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;
```

### rank()
Assigns rank with gaps for ties.
```sql
SELECT 
    name,
    score,
    rank() OVER (ORDER BY score DESC) as rank
FROM contestants;
-- Scores: 95, 92, 92, 88 → Ranks: 1, 2, 2, 4
```

### dense_rank()
Assigns rank without gaps for ties.
```sql
SELECT 
    name,
    score,
    dense_rank() OVER (ORDER BY score DESC) as dense_rank
FROM contestants;
-- Scores: 95, 92, 92, 88 → Ranks: 1, 2, 2, 3
```

### percent_rank()
Calculates relative rank (0 to 1).
```sql
SELECT 
    name,
    score,
    percent_rank() OVER (ORDER BY score) as percentile
FROM contestants;
-- Returns values between 0 and 1
```

### cume_dist()
Calculates cumulative distribution.
```sql
SELECT 
    name,
    score,
    cume_dist() OVER (ORDER BY score) as cumulative_dist
FROM contestants;
```

### ntile(n)
Divides rows into n buckets.
```sql
SELECT 
    name,
    salary,
    ntile(4) OVER (ORDER BY salary) as quartile
FROM employees;
-- Divides employees into 4 equal groups
```

## Value Functions

### lag(expression [, offset [, default]])
Accesses value from previous row.
```sql
SELECT 
    date,
    sales,
    lag(sales, 1) OVER (ORDER BY date) as previous_sales,
    sales - lag(sales, 1, 0) OVER (ORDER BY date) as growth
FROM daily_sales;
```

### lead(expression [, offset [, default]])
Accesses value from following row.
```sql
SELECT 
    date,
    sales,
    lead(sales, 1) OVER (ORDER BY date) as next_sales
FROM daily_sales;
```

### first_value(expression)
Returns first value in frame.
```sql
SELECT 
    date,
    price,
    first_value(price) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as starting_price
FROM stock_prices;
```

### last_value(expression)
Returns last value in frame.
```sql
SELECT 
    date,
    price,
    last_value(price) OVER (
        ORDER BY date
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
    ) as ending_price
FROM stock_prices;
```

### nth_value(expression, n)
Returns nth value in frame.
```sql
SELECT 
    date,
    price,
    nth_value(price, 2) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as second_price
FROM stock_prices;
```

## Aggregate Window Functions

All standard aggregate functions can be used as window functions:

### Running Totals
```sql
SELECT 
    date,
    amount,
    sum(amount) OVER (ORDER BY date) as running_total
FROM transactions;
```

### Moving Averages
```sql
SELECT 
    date,
    price,
    avg(price) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7day
FROM stock_prices;
```

### Running Count
```sql
SELECT 
    date,
    event,
    count(*) OVER (
        PARTITION BY event 
        ORDER BY date
    ) as event_count
FROM events;
```

### Running Min/Max
```sql
SELECT 
    date,
    temperature,
    min(temperature) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as min_to_date,
    max(temperature) OVER (
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as max_to_date
FROM weather;
```

## Frame Specification

### Row-based Frames
```sql
ROWS BETWEEN start AND end
```

Options:
- `UNBOUNDED PRECEDING` - Start of partition
- `n PRECEDING` - n rows before current
- `CURRENT ROW` - Current row
- `n FOLLOWING` - n rows after current
- `UNBOUNDED FOLLOWING` - End of partition

Examples:
```sql
-- Last 3 rows including current
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

-- Next 5 rows excluding current
ROWS BETWEEN 1 FOLLOWING AND 5 FOLLOWING

-- All rows in partition
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

### Range-based Frames
```sql
RANGE BETWEEN start AND end
```

Works with ORDER BY values rather than physical rows:
```sql
SELECT 
    date,
    sales,
    sum(sales) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) as sales_7day
FROM daily_sales;
```

### Groups-based Frames
```sql
GROUPS BETWEEN start AND end
```

Works with groups of peer rows (same ORDER BY value):
```sql
SELECT 
    score,
    name,
    avg(score) OVER (
        ORDER BY score
        GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) as avg_adjacent_groups
FROM results;
```

## Common Patterns

### Year-over-Year Comparison
```sql
SELECT 
    date,
    sales,
    lag(sales, 365) OVER (ORDER BY date) as sales_last_year,
    100.0 * (sales - lag(sales, 365) OVER (ORDER BY date)) / 
        lag(sales, 365) OVER (ORDER BY date) as yoy_growth_pct
FROM daily_sales;
```

### Ranking Within Groups
```sql
SELECT 
    department,
    employee,
    salary,
    rank() OVER (PARTITION BY department ORDER BY salary DESC) as dept_rank,
    rank() OVER (ORDER BY salary DESC) as company_rank
FROM employees;
```

### Finding Gaps in Sequences
```sql
SELECT 
    id,
    id - row_number() OVER (ORDER BY id) as gap_group
FROM sequences
ORDER BY id;
```

### Calculating Percentiles
```sql
SELECT 
    name,
    score,
    percent_rank() OVER (ORDER BY score) * 100 as percentile
FROM test_scores;
```

### Deduplication
```sql
WITH ranked AS (
    SELECT 
        *,
        row_number() OVER (
            PARTITION BY customer_id 
            ORDER BY created_at DESC
        ) as rn
    FROM orders
)
SELECT * FROM ranked WHERE rn = 1;
```

### Session Analysis
```sql
SELECT 
    user_id,
    timestamp,
    CASE 
        WHEN timestamp - lag(timestamp) OVER (
            PARTITION BY user_id ORDER BY timestamp
        ) > INTERVAL '30 minutes'
        THEN 1 
        ELSE 0 
    END as new_session
FROM events;
```

## Performance Considerations

1. **Partition Pruning**: Use PARTITION BY to limit window scope
2. **Frame Optimization**: Smaller frames perform better
3. **Index Usage**: ORDER BY columns should be indexed
4. **Memory Usage**: Large windows consume more memory

## Limitations

- Cannot nest window functions
- Cannot use window functions in WHERE clause
- Cannot use window functions in GROUP BY
- DISTINCT is not supported in window functions