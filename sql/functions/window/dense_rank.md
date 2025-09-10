---
title: DENSE_RANK
description: Returns the rank of rows without gaps for tied values
---

# DENSE_RANK

## Description
The `DENSE_RANK` window function assigns a rank to each row within a partition based on the ORDER BY clause values. Unlike `RANK`, when multiple rows have the same values (ties), they receive the same rank, but the next distinct value receives the immediately following rank number without gaps. This makes it ideal for creating continuous ranking sequences and categorizing data into consecutive groups.

## Syntax
```sql
DENSE_RANK() OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
)
```

### Parameters
- **PARTITION BY** (optional): Divides the result set into partitions. Ranking restarts at 1 for each partition
- **ORDER BY** (required): Determines the ranking order

### Return Value
- **Type**: BIGINT
- **Description**: Rank value starting from 1, without gaps between consecutive ranks

## Example

### Sample Data
Consider the following `product_ratings` table:

| product_id | category    | product_name      | rating | review_count |
|------------|------------|-------------------|--------|--------------|
| 1          | Electronics | Laptop Pro        | 4.8    | 1250         |
| 2          | Electronics | Wireless Mouse    | 4.5    | 890          |
| 3          | Electronics | USB Hub           | 4.5    | 650          |
| 4          | Books      | SQL Mastery       | 4.8    | 320          |
| 5          | Books      | Data Science 101  | 4.6    | 280          |
| 6          | Books      | Python Guide      | 4.6    | 450          |
| 7          | Clothing   | Winter Jacket     | 4.7    | 180          |
| 8          | Clothing   | Running Shoes     | 4.5    | 220          |

### Query
```sql
SELECT 
    product_id,
    category,
    product_name,
    rating,
    DENSE_RANK() OVER (ORDER BY rating DESC) AS overall_dense_rank,
    RANK() OVER (ORDER BY rating DESC) AS overall_rank,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY rating DESC) AS category_dense_rank,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY rating DESC, review_count DESC) AS category_row_num
FROM product_ratings
ORDER BY category, rating DESC;
```

### Result
| product_id | category    | product_name      | rating | overall_dense_rank | overall_rank | category_dense_rank | category_row_num |
|------------|------------|-------------------|--------|-------------------|--------------|-------------------|-----------------|
| 5          | Books      | Data Science 101  | 4.6    | 3                 | 5            | 2                 | 2               |
| 6          | Books      | Python Guide      | 4.6    | 3                 | 5            | 2                 | 1               |
| 4          | Books      | SQL Mastery       | 4.8    | 1                 | 1            | 1                 | 1               |
| 7          | Clothing   | Winter Jacket     | 4.7    | 2                 | 3            | 1                 | 1               |
| 8          | Clothing   | Running Shoes     | 4.5    | 4                 | 7            | 2                 | 2               |
| 1          | Electronics | Laptop Pro        | 4.8    | 1                 | 1            | 1                 | 1               |
| 2          | Electronics | Wireless Mouse    | 4.5    | 4                 | 7            | 2                 | 1               |
| 3          | Electronics | USB Hub           | 4.5    | 4                 | 7            | 2                 | 2               |

## Common Use Cases

### 1. Creating Rating Tiers
```sql
-- Categorize products into quality tiers based on ratings
WITH ranked_products AS (
    SELECT 
        product_name,
        rating,
        DENSE_RANK() OVER (ORDER BY rating DESC) AS quality_rank
    FROM products
    WHERE rating IS NOT NULL
)
SELECT 
    product_name,
    rating,
    quality_rank,
    CASE 
        WHEN quality_rank = 1 THEN 'Premium'
        WHEN quality_rank = 2 THEN 'High Quality'
        WHEN quality_rank = 3 THEN 'Standard'
        WHEN quality_rank = 4 THEN 'Budget'
        ELSE 'Economy'
    END AS quality_tier
FROM ranked_products
ORDER BY quality_rank, product_name;
```

### 2. Salary Grade Assignment
```sql
-- Assign salary grades without gaps
WITH salary_ranks AS (
    SELECT 
        employee_id,
        employee_name,
        salary,
        department,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_grade
    FROM employees
)
SELECT 
    employee_id,
    employee_name,
    department,
    salary,
    salary_grade,
    CHAR(64 + salary_grade) AS grade_letter  -- A, B, C, etc.
FROM salary_ranks
WHERE salary_grade <= 10  -- Top 10 salary grades
ORDER BY salary_grade, employee_name;
```

### 3. Finding Nth Highest Value
```sql
-- Find the 3rd highest salary in each department
WITH dense_ranked AS (
    SELECT 
        department,
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT DISTINCT
    department,
    salary AS third_highest_salary
FROM dense_ranked
WHERE salary_rank = 3
ORDER BY department;
```

### 4. Consecutive Group Identification
```sql
-- Identify consecutive days with same status
WITH status_changes AS (
    SELECT 
        date,
        status,
        DENSE_RANK() OVER (ORDER BY date) -
        DENSE_RANK() OVER (PARTITION BY status ORDER BY date) AS group_id
    FROM daily_status
)
SELECT 
    MIN(date) AS start_date,
    MAX(date) AS end_date,
    status,
    COUNT(*) AS consecutive_days
FROM status_changes
GROUP BY status, group_id
HAVING COUNT(*) >= 3  -- At least 3 consecutive days
ORDER BY start_date;
```

### 5. Tournament Standings
```sql
-- Create tournament leaderboard with proper tie handling
SELECT 
    player_name,
    total_points,
    wins,
    DENSE_RANK() OVER (ORDER BY total_points DESC, wins DESC) AS position,
    total_points - FIRST_VALUE(total_points) OVER (
        ORDER BY total_points DESC, wins DESC
    ) AS points_behind_leader
FROM tournament_scores
ORDER BY position, player_name;
```

## Gap Analysis: RANK vs DENSE_RANK

```sql
-- Demonstrate the difference in gap handling
WITH scores AS (
    SELECT score, student FROM (VALUES 
        (95, 'Alice'),
        (92, 'Bob'),
        (92, 'Charlie'),
        (92, 'David'),
        (88, 'Eve'),
        (85, 'Frank')
    ) AS t(score, student)
)
SELECT 
    student,
    score,
    RANK() OVER (ORDER BY score DESC) AS rank_with_gaps,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank_no_gaps,
    RANK() OVER (ORDER BY score DESC) - 
        DENSE_RANK() OVER (ORDER BY score DESC) AS gap_difference
FROM scores
ORDER BY score DESC;
```

## Creating Percentile Groups

```sql
-- Use DENSE_RANK to create custom percentile groups
WITH score_ranks AS (
    SELECT 
        student_id,
        test_score,
        DENSE_RANK() OVER (ORDER BY test_score DESC) AS score_rank,
        COUNT(DISTINCT test_score) OVER () AS distinct_scores
    FROM test_results
)
SELECT 
    student_id,
    test_score,
    score_rank,
    CASE 
        WHEN score_rank <= distinct_scores * 0.1 THEN 'Top 10%'
        WHEN score_rank <= distinct_scores * 0.25 THEN 'Top 25%'
        WHEN score_rank <= distinct_scores * 0.5 THEN 'Top 50%'
        ELSE 'Bottom 50%'
    END AS performance_group
FROM score_ranks
ORDER BY score_rank;
```

## Handling NULL Values

```sql
-- Different NULL handling strategies with DENSE_RANK
SELECT 
    product_name,
    price,
    -- NULLs last (default in most databases)
    DENSE_RANK() OVER (ORDER BY price DESC) AS rank_nulls_last,
    -- NULLs first
    DENSE_RANK() OVER (ORDER BY price DESC NULLS FIRST) AS rank_nulls_first,
    -- Exclude NULLs with CASE
    CASE 
        WHEN price IS NOT NULL 
        THEN DENSE_RANK() OVER (ORDER BY price DESC)
    END AS rank_exclude_nulls
FROM products
ORDER BY price DESC NULLS LAST;
```

## Performance Considerations

```sql
-- Optimize DENSE_RANK queries with appropriate indexes
CREATE INDEX idx_category_rating ON products(category, rating DESC);

-- Efficient query using the index
SELECT 
    product_id,
    product_name,
    category,
    rating,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY rating DESC) AS category_rank
FROM products
WHERE category IN ('Electronics', 'Books')
  AND rating >= 4.0;

-- Consider materialized views for frequently accessed rankings
CREATE MATERIALIZED VIEW product_rankings AS
SELECT 
    product_id,
    category,
    rating,
    DENSE_RANK() OVER (ORDER BY rating DESC) AS overall_rank,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY rating DESC) AS category_rank
FROM products
WHERE status = 'active';
```

## Advanced Usage: Multi-Level Ranking

```sql
-- Create hierarchical rankings
WITH multi_rank AS (
    SELECT 
        region,
        country,
        city,
        sales_amount,
        DENSE_RANK() OVER (ORDER BY sales_amount DESC) AS global_rank,
        DENSE_RANK() OVER (PARTITION BY region ORDER BY sales_amount DESC) AS regional_rank,
        DENSE_RANK() OVER (PARTITION BY region, country ORDER BY sales_amount DESC) AS country_rank
    FROM sales_by_location
)
SELECT 
    region,
    country,
    city,
    sales_amount,
    global_rank,
    regional_rank,
    country_rank,
    CONCAT('G', global_rank, '-R', regional_rank, '-C', country_rank) AS rank_code
FROM multi_rank
ORDER BY global_rank, regional_rank, country_rank;
```

## Related Functions
- [RANK](./rank.md) - Ranking with gaps for ties
- [ROW_NUMBER](./row_number.md) - Sequential numbering without ties
- [PERCENT_RANK](./percent_rank.md) - Relative rank as percentage
- [CUME_DIST](./cume_dist.md) - Cumulative distribution
- [NTILE](./ntile.md) - Distribute rows into equal buckets