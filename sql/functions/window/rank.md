---
title: RANK
description: Returns the rank of rows with gaps for tied values
---

# RANK

## Description
The `RANK` window function assigns a rank to each row within a partition based on the values in the ORDER BY clause. When multiple rows have the same values (ties), they receive the same rank, and the next rank value is incremented by the number of tied rows, creating gaps in the ranking sequence. This function is commonly used for competitive rankings, percentile calculations, and identifying top performers.

## Syntax
```sql
RANK() OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
)
```

### Parameters
- **PARTITION BY** (optional): Divides the result set into partitions. Ranking restarts at 1 for each partition
- **ORDER BY** (required): Determines the ranking order

### Return Value
- **Type**: BIGINT
- **Description**: Rank value starting from 1, with gaps after ties

## Example

### Sample Data
Consider the following `sales_performance` table:

| salesperson_id | region | quarter | sales_amount | units_sold |
|---------------|--------|---------|--------------|------------|
| 1             | North  | Q1      | 150000       | 120        |
| 2             | North  | Q1      | 150000       | 115        |
| 3             | North  | Q1      | 125000       | 100        |
| 4             | South  | Q1      | 180000       | 140        |
| 5             | South  | Q1      | 165000       | 130        |
| 6             | South  | Q1      | 165000       | 135        |
| 7             | East   | Q1      | 140000       | 110        |
| 8             | East   | Q1      | 140000       | 110        |

### Query
```sql
SELECT 
    salesperson_id,
    region,
    sales_amount,
    units_sold,
    RANK() OVER (ORDER BY sales_amount DESC) AS overall_rank,
    RANK() OVER (PARTITION BY region ORDER BY sales_amount DESC) AS regional_rank,
    DENSE_RANK() OVER (ORDER BY sales_amount DESC) AS dense_rank_comparison,
    ROW_NUMBER() OVER (ORDER BY sales_amount DESC) AS row_number_comparison
FROM sales_performance
ORDER BY overall_rank, salesperson_id;
```

### Result
| salesperson_id | region | sales_amount | units_sold | overall_rank | regional_rank | dense_rank_comparison | row_number_comparison |
|---------------|--------|--------------|------------|--------------|---------------|--------------------|---------------------|
| 4             | South  | 180000       | 140        | 1            | 1             | 1                  | 1                   |
| 5             | South  | 165000       | 130        | 2            | 2             | 2                  | 2                   |
| 6             | South  | 165000       | 135        | 2            | 2             | 2                  | 3                   |
| 1             | North  | 150000       | 120        | 4            | 1             | 3                  | 4                   |
| 2             | North  | 150000       | 115        | 4            | 1             | 3                  | 5                   |
| 7             | East   | 140000       | 110        | 6            | 1             | 4                  | 6                   |
| 8             | East   | 140000       | 110        | 6            | 1             | 4                  | 7                   |
| 3             | North  | 125000       | 100        | 8            | 3             | 5                  | 8                   |

## Common Use Cases

### 1. Competition Rankings
```sql
-- Rank students by test scores with proper tie handling
SELECT 
    student_name,
    test_score,
    RANK() OVER (ORDER BY test_score DESC) AS rank,
    CASE 
        WHEN RANK() OVER (ORDER BY test_score DESC) = 1 THEN 'Gold Medal'
        WHEN RANK() OVER (ORDER BY test_score DESC) = 2 THEN 'Silver Medal'
        WHEN RANK() OVER (ORDER BY test_score DESC) = 3 THEN 'Bronze Medal'
        ELSE 'Participant'
    END AS award
FROM test_results
ORDER BY rank;
```

### 2. Percentile Calculation
```sql
-- Calculate percentile ranks
WITH ranked_scores AS (
    SELECT 
        student_id,
        score,
        RANK() OVER (ORDER BY score) AS rank_asc,
        COUNT(*) OVER () AS total_count
    FROM exam_scores
)
SELECT 
    student_id,
    score,
    rank_asc,
    ROUND((rank_asc - 1) * 100.0 / (total_count - 1), 2) AS percentile
FROM ranked_scores
ORDER BY score DESC;
```

### 3. Top N with Ties
```sql
-- Get all employees who rank in top 3 by sales (including ties)
WITH ranked_sales AS (
    SELECT 
        employee_name,
        total_sales,
        RANK() OVER (ORDER BY total_sales DESC) AS sales_rank
    FROM employee_sales
)
SELECT 
    employee_name,
    total_sales,
    sales_rank
FROM ranked_sales
WHERE sales_rank <= 3
ORDER BY sales_rank, employee_name;
```

### 4. Year-over-Year Ranking Changes
```sql
-- Track ranking changes between years
WITH yearly_ranks AS (
    SELECT 
        company_name,
        year,
        revenue,
        RANK() OVER (PARTITION BY year ORDER BY revenue DESC) AS yearly_rank
    FROM company_revenues
    WHERE year IN (2023, 2024)
)
SELECT 
    c1.company_name,
    c1.revenue AS revenue_2023,
    c1.yearly_rank AS rank_2023,
    c2.revenue AS revenue_2024,
    c2.yearly_rank AS rank_2024,
    c1.yearly_rank - c2.yearly_rank AS rank_improvement
FROM yearly_ranks c1
JOIN yearly_ranks c2 ON c1.company_name = c2.company_name
WHERE c1.year = 2023 AND c2.year = 2024
ORDER BY c2.yearly_rank;
```

### 5. Grouped Rankings
```sql
-- Rank products within categories and overall
SELECT 
    product_name,
    category,
    rating,
    review_count,
    RANK() OVER (ORDER BY rating DESC, review_count DESC) AS overall_rank,
    RANK() OVER (PARTITION BY category ORDER BY rating DESC, review_count DESC) AS category_rank
FROM products
WHERE status = 'active'
ORDER BY category, category_rank;
```

## Handling Ties with Multiple Columns

```sql
-- Complex tie-breaking with multiple columns
SELECT 
    athlete_name,
    event,
    primary_score,
    secondary_score,
    -- Rank by primary score only (allows ties)
    RANK() OVER (
        PARTITION BY event 
        ORDER BY primary_score DESC
    ) AS primary_rank,
    -- Rank by both scores (fewer ties)
    RANK() OVER (
        PARTITION BY event 
        ORDER BY primary_score DESC, secondary_score DESC
    ) AS combined_rank
FROM competition_results
ORDER BY event, combined_rank;
```

## Identifying Rank Gaps

```sql
-- Find gaps in ranking (useful for understanding data distribution)
WITH rankings AS (
    SELECT 
        value,
        RANK() OVER (ORDER BY value DESC) AS rank,
        LAG(RANK()) OVER (ORDER BY value DESC) AS prev_rank
    FROM data_points
)
SELECT 
    value,
    rank,
    prev_rank,
    rank - COALESCE(prev_rank, 0) AS gap_size,
    CASE 
        WHEN rank - COALESCE(prev_rank, 0) > 1 THEN 'Gap exists'
        ELSE 'Consecutive'
    END AS gap_status
FROM rankings
ORDER BY rank;
```

## Performance Optimization

```sql
-- Efficient ranking with proper indexing
-- Create composite index for partition and order columns
CREATE INDEX idx_sales_region_amount ON sales_data(region, sales_amount DESC);

-- Query will benefit from the index
SELECT 
    salesperson,
    region,
    sales_amount,
    RANK() OVER (PARTITION BY region ORDER BY sales_amount DESC) AS regional_rank
FROM sales_data
WHERE region IN ('North', 'South')
  AND sales_amount > 0;
```

## Combining with Aggregate Functions

```sql
-- Rank departments by average salary
WITH dept_averages AS (
    SELECT 
        department,
        AVG(salary) AS avg_salary,
        COUNT(*) AS employee_count,
        SUM(salary) AS total_payroll
    FROM employees
    GROUP BY department
)
SELECT 
    department,
    avg_salary,
    employee_count,
    total_payroll,
    RANK() OVER (ORDER BY avg_salary DESC) AS salary_rank,
    RANK() OVER (ORDER BY employee_count DESC) AS size_rank
FROM dept_averages
ORDER BY salary_rank;
```

## Common Pitfalls and Solutions

```sql
-- Pitfall: Unexpected gaps in sequential operations
-- If you need continuous numbering, use DENSE_RANK or ROW_NUMBER

-- Pitfall: NULL handling in ORDER BY
SELECT 
    product_name,
    price,
    RANK() OVER (ORDER BY price DESC) AS rank_nulls_last,  -- NULLs ranked last
    RANK() OVER (ORDER BY price DESC NULLS FIRST) AS rank_nulls_first
FROM products;

-- Pitfall: Performance with large partitions
-- Consider pre-aggregating or filtering data before ranking
WITH filtered_data AS (
    SELECT * FROM large_table
    WHERE date >= CURRENT_DATE - INTERVAL '30 days'
)
SELECT *, RANK() OVER (PARTITION BY category ORDER BY value DESC)
FROM filtered_data;
```

## Related Functions
- [ROW_NUMBER](./row_number.md) - Sequential numbering without ties
- [DENSE_RANK](./dense_rank.md) - Ranking without gaps
- [PERCENT_RANK](./percent_rank.md) - Relative rank as percentage
- [CUME_DIST](./cume_dist.md) - Cumulative distribution
- [NTILE](./ntile.md) - Distribute rows into buckets