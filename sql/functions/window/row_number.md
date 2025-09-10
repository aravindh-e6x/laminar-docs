---
title: ROW_NUMBER
description: Assigns a unique sequential integer to each row within a partition
---

# ROW_NUMBER

## Description
The `ROW_NUMBER` window function assigns a unique sequential integer to each row within a partition of a result set, starting at 1 for the first row in each partition. Unlike `RANK` and `DENSE_RANK`, `ROW_NUMBER` always generates unique numbers within a partition, even for rows with identical values in the ORDER BY columns. This makes it perfect for pagination, deduplication, and selecting top-N records per group.

## Syntax
```sql
ROW_NUMBER() OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
)
```

### Parameters
- **PARTITION BY** (optional): Divides the result set into partitions. Row numbering restarts at 1 for each partition.
- **ORDER BY** (required): Determines the sequence in which rows are numbered within each partition.

### Return Value
- **Type**: BIGINT
- **Description**: A unique sequential number starting from 1 within each partition

## Example

### Sample Data
Consider the following `employee_salaries` table showing employee compensation by department:

| emp_id | emp_name  | department | salary  | hire_date  |
|--------|-----------|------------|---------|------------|
| 101    | Alice     | Sales      | 75000   | 2021-03-15 |
| 102    | Bob       | Sales      | 82000   | 2020-07-22 |
| 103    | Charlie   | Sales      | 75000   | 2022-01-10 |
| 104    | Diana     | Marketing  | 68000   | 2021-09-01 |
| 105    | Eve       | Marketing  | 72000   | 2019-11-15 |
| 106    | Frank     | Marketing  | 72000   | 2020-05-20 |
| 107    | Grace     | IT         | 95000   | 2018-08-10 |
| 108    | Henry     | IT         | 88000   | 2021-02-28 |
| 109    | Iris      | IT         | 92000   | 2020-06-15 |

### Query
```sql
SELECT 
    emp_id,
    emp_name,
    department,
    salary,
    hire_date,
    -- Row number within entire result set
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS overall_rank,
    -- Row number within each department
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    -- Row number by hire date within department
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY hire_date) AS seniority_rank
FROM employee_salaries
ORDER BY department, salary DESC;
```

### Result
| emp_id | emp_name  | department | salary | hire_date  | overall_rank | dept_rank | seniority_rank |
|--------|-----------|------------|--------|------------|--------------|-----------|----------------|
| 107    | Grace     | IT         | 95000  | 2018-08-10 | 1            | 1         | 1              |
| 109    | Iris      | IT         | 92000  | 2020-06-15 | 2            | 2         | 2              |
| 108    | Henry     | IT         | 88000  | 2021-02-28 | 3            | 3         | 3              |
| 105    | Eve       | Marketing  | 72000  | 2019-11-15 | 5            | 1         | 1              |
| 106    | Frank     | Marketing  | 72000  | 2020-05-20 | 6            | 2         | 2              |
| 104    | Diana     | Marketing  | 68000  | 2021-09-01 | 7            | 3         | 3              |
| 102    | Bob       | Sales      | 82000  | 2020-07-22 | 4            | 1         | 2              |
| 101    | Alice     | Sales      | 75000  | 2021-03-15 | 8            | 2         | 1              |
| 103    | Charlie   | Sales      | 75000  | 2022-01-10 | 9            | 3         | 3              |

## Common Use Cases

### 1. Pagination
```sql
-- Get page 2 of results (items 11-20)
WITH numbered_results AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (ORDER BY created_date DESC) AS row_num
    FROM articles
    WHERE status = 'published'
)
SELECT * 
FROM numbered_results
WHERE row_num BETWEEN 11 AND 20;
```

### 2. Removing Duplicates
```sql
-- Keep only the most recent record for each customer
WITH numbered_records AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY updated_at DESC
        ) AS rn
    FROM customer_records
)
SELECT * 
FROM numbered_records
WHERE rn = 1;
```

### 3. Top-N Per Group
```sql
-- Get top 3 highest-paid employees per department
WITH ranked_employees AS (
    SELECT 
        emp_name,
        department,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department 
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees
)
SELECT * 
FROM ranked_employees
WHERE salary_rank <= 3
ORDER BY department, salary_rank;
```

### 4. Sequential Numbering After Gaps
```sql
-- Renumber invoice IDs sequentially after deletions
SELECT 
    invoice_id AS original_id,
    ROW_NUMBER() OVER (ORDER BY invoice_date, invoice_id) AS new_sequential_id,
    invoice_date,
    customer_id,
    amount
FROM invoices
ORDER BY new_sequential_id;
```

### 5. Finding First/Last Records
```sql
-- Find first and last order for each customer
WITH order_positions AS (
    SELECT 
        customer_id,
        order_id,
        order_date,
        order_total,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY order_date ASC
        ) AS order_sequence,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY order_date DESC
        ) AS reverse_sequence
    FROM orders
)
SELECT 
    customer_id,
    MAX(CASE WHEN order_sequence = 1 THEN order_id END) AS first_order_id,
    MAX(CASE WHEN order_sequence = 1 THEN order_date END) AS first_order_date,
    MAX(CASE WHEN reverse_sequence = 1 THEN order_id END) AS last_order_id,
    MAX(CASE WHEN reverse_sequence = 1 THEN order_date END) AS last_order_date
FROM order_positions
GROUP BY customer_id;
```

## Comparison with RANK and DENSE_RANK

```sql
-- Demonstrating differences between ranking functions
WITH scores AS (
    SELECT 'Alice' AS name, 95 AS score
    UNION ALL SELECT 'Bob', 92
    UNION ALL SELECT 'Charlie', 92
    UNION ALL SELECT 'Diana', 88
    UNION ALL SELECT 'Eve', 85
)
SELECT 
    name,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_number,
    RANK() OVER (ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM scores
ORDER BY score DESC;
```

Result shows:
- ROW_NUMBER: 1, 2, 3, 4, 5 (always unique)
- RANK: 1, 2, 2, 4, 5 (same rank for ties, gaps after ties)
- DENSE_RANK: 1, 2, 2, 3, 4 (same rank for ties, no gaps)

## Performance Considerations
- Always include ORDER BY in the OVER clause
- Use PARTITION BY to limit the scope of numbering
- Consider indexing columns used in PARTITION BY and ORDER BY
- ROW_NUMBER is non-deterministic for rows with identical ORDER BY values

## Related Functions
- [RANK](./rank.md) - Assigns rank with gaps for ties
- [DENSE_RANK](./dense_rank.md) - Assigns rank without gaps for ties
- [NTILE](./ntile.md) - Divides rows into buckets
- [LAG](./lag.md) - Accesses previous row value
- [LEAD](./lead.md) - Accesses next row value