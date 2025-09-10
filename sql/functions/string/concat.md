---
title: CONCAT
description: Concatenates two or more strings into a single string
---

# CONCAT

## Description
The `CONCAT` function concatenates two or more string values into a single string. It accepts multiple arguments and joins them in the order provided. Unlike the concatenation operator (||), CONCAT handles NULL values by treating them as empty strings rather than propagating NULL through the entire result.

## Syntax
```sql
CONCAT(string1, string2 [, string3, ...])
```

### Parameters
- **string1** (string): First string to concatenate
- **string2** (string): Second string to concatenate
- **stringN** (string, optional): Additional strings to concatenate

### Return Value
- **Type**: STRING/VARCHAR
- **Description**: A single string containing all input strings joined together. NULL values are treated as empty strings.

## Example

### Sample Data
Consider the following `employees` table with name components:

| emp_id | first_name | middle_name | last_name | department |
|--------|------------|-------------|-----------|------------|
| 1      | John       | Michael     | Smith     | Sales      |
| 2      | Sarah      | NULL        | Johnson   | Marketing  |
| 3      | Robert     | James       | Williams  | IT         |
| 4      | Maria      | NULL        | Garcia    | HR         |
| 5      | David      | Lee         | Brown     | Finance    |

### Query
```sql
SELECT 
    emp_id,
    first_name,
    last_name,
    CONCAT(first_name, ' ', last_name) AS full_name,
    CONCAT(first_name, ' ', middle_name, ' ', last_name) AS full_name_with_middle,
    CONCAT(last_name, ', ', first_name) AS last_first,
    CONCAT(department, '-', emp_id) AS dept_code
FROM employees
ORDER BY emp_id;
```

### Result
| emp_id | first_name | last_name | full_name      | full_name_with_middle    | last_first       | dept_code    |
|--------|------------|-----------|----------------|-------------------------|------------------|--------------|
| 1      | John       | Smith     | John Smith     | John Michael Smith      | Smith, John      | Sales-1      |
| 2      | Sarah      | Johnson   | Sarah Johnson  | Sarah  Johnson          | Johnson, Sarah   | Marketing-2  |
| 3      | Robert     | Williams  | Robert Williams| Robert James Williams   | Williams, Robert | IT-3         |
| 4      | Maria      | Garcia    | Maria Garcia   | Maria  Garcia           | Garcia, Maria    | HR-4         |
| 5      | David      | Brown     | David Brown    | David Lee Brown         | Brown, David     | Finance-5    |

## Common Use Cases

### 1. Building Full Names
```sql
-- Create formatted full names with titles
SELECT 
    CONCAT(
        CASE 
            WHEN gender = 'M' THEN 'Mr. '
            WHEN gender = 'F' THEN 'Ms. '
            ELSE ''
        END,
        first_name, 
        ' ', 
        last_name
    ) AS formal_name
FROM customers;
```

### 2. Creating URLs or File Paths
```sql
-- Build complete URLs from components
SELECT 
    CONCAT(
        'https://',
        domain,
        '/',
        path,
        '/',
        filename,
        '.html'
    ) AS full_url
FROM web_pages;
```

### 3. Generating Codes or IDs
```sql
-- Create product SKUs
SELECT 
    CONCAT(
        category_code,
        '-',
        LPAD(product_id::text, 5, '0'),
        '-',
        year
    ) AS sku
FROM products;
```

### 4. Building SQL Queries Dynamically
```sql
-- Generate dynamic column lists
SELECT 
    table_name,
    CONCAT(
        'SELECT ',
        STRING_AGG(column_name, ', '),
        ' FROM ',
        table_name,
        ';'
    ) AS select_query
FROM information_schema.columns
GROUP BY table_name;
```

### 5. Formatting Addresses
```sql
-- Build complete mailing addresses
SELECT 
    CONCAT(
        street_number, ' ',
        street_name, ', ',
        city, ', ',
        state, ' ',
        zip_code
    ) AS full_address
FROM addresses;
```

## NULL Handling Comparison

```sql
-- Comparison of CONCAT vs || operator with NULLs
SELECT 
    first_name,
    middle_name,
    last_name,
    CONCAT(first_name, middle_name, last_name) AS concat_result,
    first_name || middle_name || last_name AS pipe_result
FROM employees
WHERE middle_name IS NULL;
```

Note: CONCAT treats NULL as empty string, while || propagates NULL.

## Related Functions
- [CONCAT_WS](./concat_ws.md) - Concatenate with separator
- [STRING_AGG](../aggregate/string_agg.md) - Aggregate concatenation
- [|| operator](./pipe_operator.md) - String concatenation operator
- [LPAD](./lpad.md) - Left pad string to specified length
- [RPAD](./rpad.md) - Right pad string to specified length