---
title: PERCENT_RANK
description: Returns the relative rank of rows as a percentage from 0 to 1
---

# PERCENT_RANK

## Description
The `PERCENT_RANK` window function calculates the relative rank of each row within a partition as a percentage value between 0 and 1. It returns (rank - 1) / (total_rows - 1), where rank is determined by the ORDER BY clause. The first row always gets 0, and the last row gets 1 (except when there's only one row, which gets 0). This function is useful for calculating percentiles, creating quartiles, and understanding relative positioning within a dataset.

## Syntax
```sql
PERCENT_RANK() OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
)
```

### Parameters
- **PARTITION BY** (optional): Divides the result set into partitions. Percent rank is calculated separately for each partition
- **ORDER BY** (required): Determines the ranking order for percent rank calculation

### Return Value
- **Type**: DOUBLE PRECISION
- **Description**: A value between 0 and 1 representing the relative position (0 = first, 1 = last)

## Example

### Sample Data
Consider the following `exam_scores` table:

| student_id | subject | score | exam_date  |
|------------|---------|-------|------------|
| 1          | Math    | 95    | 2024-03-15 |
| 2          | Math    | 88    | 2024-03-15 |
| 3          | Math    | 92    | 2024-03-15 |
| 4          | Math    | 88    | 2024-03-15 |
| 5          | Math    | 76    | 2024-03-15 |
| 6          | Science | 91    | 2024-03-16 |
| 7          | Science | 85    | 2024-03-16 |
| 8          | Science | 94    | 2024-03-16 |
| 9          | Science | 85    | 2024-03-16 |
| 10         | Science | 79    | 2024-03-16 |

### Query
```sql
SELECT 
    student_id,
    subject,
    score,
    RANK() OVER (PARTITION BY subject ORDER BY score) AS rank,
    PERCENT_RANK() OVER (PARTITION BY subject ORDER BY score) AS percent_rank,
    ROUND(PERCENT_RANK() OVER (PARTITION BY subject ORDER BY score) * 100, 2) AS percentile,
    CUME_DIST() OVER (PARTITION BY subject ORDER BY score) AS cume_dist_comparison
FROM exam_scores
ORDER BY subject, score;
```

### Result
| student_id | subject | score | rank | percent_rank | percentile | cume_dist_comparison |
|------------|---------|-------|------|--------------|------------|---------------------|
| 5          | Math    | 76    | 1    | 0.00         | 0.00       | 0.20                |
| 2          | Math    | 88    | 2    | 0.25         | 25.00      | 0.60                |
| 4          | Math    | 88    | 2    | 0.25         | 25.00      | 0.60                |
| 3          | Math    | 92    | 4    | 0.75         | 75.00      | 0.80                |
| 1          | Math    | 95    | 5    | 1.00         | 100.00     | 1.00                |
| 10         | Science | 79    | 1    | 0.00         | 0.00       | 0.20                |
| 7          | Science | 85    | 2    | 0.25         | 25.00      | 0.60                |
| 9          | Science | 85    | 2    | 0.25         | 25.00      | 0.60                |
| 6          | Science | 91    | 4    | 0.75         | 75.00      | 0.80                |
| 8          | Science | 94    | 5    | 1.00         | 100.00     | 1.00                |

## Common Use Cases

### 1. Percentile Classification
```sql
-- Classify students into performance quartiles
WITH score_percentiles AS (
    SELECT 
        student_name,
        test_score,
        PERCENT_RANK() OVER (ORDER BY test_score) AS pct_rank
    FROM test_results
)
SELECT 
    student_name,
    test_score,
    ROUND(pct_rank * 100, 2) AS percentile,
    CASE 
        WHEN pct_rank <= 0.25 THEN 'Bottom Quartile (Q1)'
        WHEN pct_rank <= 0.50 THEN 'Second Quartile (Q2)'
        WHEN pct_rank <= 0.75 THEN 'Third Quartile (Q3)'
        ELSE 'Top Quartile (Q4)'
    END AS quartile,
    CASE 
        WHEN pct_rank >= 0.90 THEN 'Top 10%'
        WHEN pct_rank >= 0.75 THEN 'Top 25%'
        WHEN pct_rank >= 0.50 THEN 'Top 50%'
        ELSE 'Bottom 50%'
    END AS performance_tier
FROM score_percentiles
ORDER BY pct_rank DESC;
```

### 2. Salary Percentile Analysis
```sql
-- Analyze salary distribution and positioning
SELECT 
    employee_id,
    employee_name,
    department,
    salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 2) AS overall_percentile,
    ROUND(PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary) * 100, 2) AS dept_percentile,
    CASE 
        WHEN PERCENT_RANK() OVER (ORDER BY salary) >= 0.9 THEN 'High Earner'
        WHEN PERCENT_RANK() OVER (ORDER BY salary) <= 0.1 THEN 'Entry Level'
        ELSE 'Mid Range'
    END AS salary_category
FROM employees
WHERE employment_status = 'Active'
ORDER BY overall_percentile DESC;
```

### 3. Performance Benchmarking
```sql
-- Benchmark sales performance across regions
WITH sales_metrics AS (
    SELECT 
        salesperson,
        region,
        total_sales,
        PERCENT_RANK() OVER (ORDER BY total_sales) AS company_percentile,
        PERCENT_RANK() OVER (PARTITION BY region ORDER BY total_sales) AS regional_percentile
    FROM quarterly_sales
    WHERE quarter = 'Q4-2024'
)
SELECT 
    salesperson,
    region,
    total_sales,
    ROUND(company_percentile * 100, 1) AS company_pct,
    ROUND(regional_percentile * 100, 1) AS regional_pct,
    ROUND((regional_percentile - company_percentile) * 100, 1) AS regional_vs_company_diff
FROM sales_metrics
ORDER BY company_percentile DESC;
```

### 4. Distribution Analysis
```sql
-- Analyze value distribution using percent rank
WITH distribution AS (
    SELECT 
        value,
        category,
        PERCENT_RANK() OVER (PARTITION BY category ORDER BY value) AS pct_rank,
        COUNT(*) OVER (PARTITION BY category) AS category_count
    FROM measurements
)
SELECT 
    category,
    MIN(CASE WHEN pct_rank >= 0.00 THEN value END) AS min_value,
    MIN(CASE WHEN pct_rank >= 0.25 THEN value END) AS q1_value,
    MIN(CASE WHEN pct_rank >= 0.50 THEN value END) AS median_value,
    MIN(CASE WHEN pct_rank >= 0.75 THEN value END) AS q3_value,
    MAX(value) AS max_value,
    category_count AS sample_size
FROM distribution
GROUP BY category, category_count
ORDER BY category;
```

### 5. Grading Curves
```sql
-- Apply grading curve based on percent rank
WITH curved_grades AS (
    SELECT 
        student_id,
        raw_score,
        PERCENT_RANK() OVER (ORDER BY raw_score) AS pct_rank
    FROM exam_scores
)
SELECT 
    student_id,
    raw_score,
    ROUND(pct_rank * 100, 2) AS percentile,
    CASE 
        WHEN pct_rank >= 0.93 THEN 'A'
        WHEN pct_rank >= 0.85 THEN 'A-'
        WHEN pct_rank >= 0.77 THEN 'B+'
        WHEN pct_rank >= 0.70 THEN 'B'
        WHEN pct_rank >= 0.62 THEN 'B-'
        WHEN pct_rank >= 0.55 THEN 'C+'
        WHEN pct_rank >= 0.45 THEN 'C'
        WHEN pct_rank >= 0.35 THEN 'C-'
        WHEN pct_rank >= 0.25 THEN 'D'
        ELSE 'F'
    END AS curved_grade
FROM curved_grades
ORDER BY pct_rank DESC;
```

## Comparison with CUME_DIST

```sql
-- Understanding the difference between PERCENT_RANK and CUME_DIST
WITH sample AS (
    SELECT value FROM (VALUES (10), (20), (20), (30), (40)) AS t(value)
)
SELECT 
    value,
    ROW_NUMBER() OVER (ORDER BY value) AS row_num,
    RANK() OVER (ORDER BY value) AS rank,
    PERCENT_RANK() OVER (ORDER BY value) AS percent_rank,
    -- PERCENT_RANK = (rank - 1) / (count - 1)
    ROUND((RANK() OVER (ORDER BY value) - 1.0) / 
          (COUNT(*) OVER () - 1), 4) AS calculated_pct_rank,
    CUME_DIST() OVER (ORDER BY value) AS cume_dist,
    -- CUME_DIST = rank / count (includes ties)
    ROUND(RANK() OVER (ORDER BY value) * 1.0 / COUNT(*) OVER (), 4) AS calculated_cume_dist
FROM sample
ORDER BY value;
```

## Handling Edge Cases

```sql
-- Handle single row partitions (always returns 0)
WITH single_row_test AS (
    SELECT 
        group_id,
        value,
        COUNT(*) OVER (PARTITION BY group_id) AS group_size,
        PERCENT_RANK() OVER (PARTITION BY group_id ORDER BY value) AS pct_rank
    FROM (VALUES 
        ('A', 100),
        ('B', 200),
        ('B', 250),
        ('C', 300)
    ) AS t(group_id, value)
)
SELECT 
    group_id,
    value,
    group_size,
    pct_rank,
    CASE 
        WHEN group_size = 1 THEN 'Single row - always 0'
        ELSE 'Multiple rows - calculated'
    END AS explanation
FROM single_row_test
ORDER BY group_id, value;
```

## Creating Custom Percentile Bins

```sql
-- Create custom percentile bins for analysis
WITH percentile_data AS (
    SELECT 
        customer_id,
        lifetime_value,
        PERCENT_RANK() OVER (ORDER BY lifetime_value) AS pct_rank
    FROM customer_metrics
)
SELECT 
    CASE 
        WHEN pct_rank >= 0.95 THEN '95-100th percentile'
        WHEN pct_rank >= 0.90 THEN '90-95th percentile'
        WHEN pct_rank >= 0.75 THEN '75-90th percentile'
        WHEN pct_rank >= 0.50 THEN '50-75th percentile'
        WHEN pct_rank >= 0.25 THEN '25-50th percentile'
        ELSE '0-25th percentile'
    END AS percentile_bin,
    COUNT(*) AS customer_count,
    MIN(lifetime_value) AS min_value,
    AVG(lifetime_value) AS avg_value,
    MAX(lifetime_value) AS max_value
FROM percentile_data
GROUP BY 
    CASE 
        WHEN pct_rank >= 0.95 THEN '95-100th percentile'
        WHEN pct_rank >= 0.90 THEN '90-95th percentile'
        WHEN pct_rank >= 0.75 THEN '75-90th percentile'
        WHEN pct_rank >= 0.50 THEN '50-75th percentile'
        WHEN pct_rank >= 0.25 THEN '25-50th percentile'
        ELSE '0-25th percentile'
    END
ORDER BY MIN(lifetime_value) DESC;
```

## Performance Optimization

```sql
-- Optimize PERCENT_RANK with proper indexing
CREATE INDEX idx_scores ON exam_scores(subject, score);

-- Efficient percentile calculation for large datasets
WITH ranked_data AS (
    SELECT 
        student_id,
        score,
        subject,
        PERCENT_RANK() OVER (PARTITION BY subject ORDER BY score) AS pct_rank
    FROM exam_scores
    WHERE exam_date >= CURRENT_DATE - INTERVAL '1 year'
)
SELECT * FROM ranked_data
WHERE pct_rank >= 0.9  -- Top 10% only
ORDER BY subject, pct_rank DESC;
```

## Related Functions
- [CUME_DIST](./cume_dist.md) - Cumulative distribution (includes current row)
- [RANK](./rank.md) - Ranking with gaps
- [DENSE_RANK](./dense_rank.md) - Ranking without gaps
- [NTILE](./ntile.md) - Divide rows into equal buckets
- [ROW_NUMBER](./row_number.md) - Sequential row numbering