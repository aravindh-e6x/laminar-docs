---
title: ARRAY_AGG
description: Aggregates values into an array
---

# ARRAY_AGG

## Description
The `ARRAY_AGG` function aggregates values from multiple rows into a single array. It collects all non-NULL values from a column or expression and returns them as an array. This function is particularly useful for data denormalization, creating JSON structures, and grouping related values together. The order of elements can be controlled using ORDER BY within the function.

## Syntax
```sql
ARRAY_AGG([DISTINCT] expression [ORDER BY sort_expression])
```

### Parameters
- **DISTINCT** (optional): Includes only unique values in the array
- **expression**: Column or expression to aggregate into an array
- **ORDER BY** (optional): Orders elements within the array

### Return Value
- **Type**: ARRAY of the expression's type
- **Description**: An array containing all non-NULL values. Returns NULL if all values are NULL or no rows match.

## Example

### Sample Data
Consider the following `customer_orders` table:

| order_id | customer_id | product_name | category    | order_date | quantity |
|----------|------------|--------------|-------------|------------|----------|
| 1        | C001       | Laptop       | Electronics | 2024-03-01 | 1        |
| 2        | C001       | Mouse        | Electronics | 2024-03-01 | 2        |
| 3        | C002       | Notebook     | Stationery  | 2024-03-02 | 5        |
| 4        | C001       | Monitor      | Electronics | 2024-03-03 | 1        |
| 5        | C002       | Pen          | Stationery  | 2024-03-03 | 10       |
| 6        | C003       | Desk Chair   | Furniture   | 2024-03-04 | 1        |
| 7        | C002       | Laptop       | Electronics | 2024-03-05 | 1        |

### Query
```sql
SELECT 
    customer_id,
    ARRAY_AGG(product_name) AS all_products,
    ARRAY_AGG(DISTINCT category) AS unique_categories,
    ARRAY_AGG(product_name ORDER BY order_date) AS products_by_date,
    ARRAY_AGG(quantity) AS quantities
FROM customer_orders
GROUP BY customer_id
ORDER BY customer_id;
```

### Result
| customer_id | all_products                    | unique_categories         | products_by_date                | quantities |
|------------|----------------------------------|---------------------------|----------------------------------|-----------|
| C001       | [Laptop, Mouse, Monitor]        | [Electronics]             | [Laptop, Mouse, Monitor]        | [1, 2, 1] |
| C002       | [Notebook, Pen, Laptop]          | [Stationery, Electronics] | [Notebook, Pen, Laptop]          | [5, 10, 1]|
| C003       | [Desk Chair]                     | [Furniture]               | [Desk Chair]                     | [1]       |

## Common Use Cases

### 1. Creating JSON-Ready Structures
```sql
-- Aggregate data for JSON export
SELECT 
    customer_id,
    JSON_BUILD_OBJECT(
        'customer_id', customer_id,
        'order_count', COUNT(*),
        'products', ARRAY_AGG(
            JSON_BUILD_OBJECT(
                'name', product_name,
                'category', category,
                'quantity', quantity
            ) ORDER BY order_date
        ),
        'categories', ARRAY_AGG(DISTINCT category)
    ) AS customer_data
FROM customer_orders
GROUP BY customer_id;
```

### 2. Tag/Label Aggregation
```sql
-- Collect all tags associated with articles
SELECT 
    article_id,
    title,
    ARRAY_AGG(DISTINCT tag_name ORDER BY tag_name) AS tags,
    ARRAY_TO_STRING(ARRAY_AGG(DISTINCT tag_name ORDER BY tag_name), ', ') AS tags_string
FROM articles
JOIN article_tags USING (article_id)
GROUP BY article_id, title
HAVING 'technology' = ANY(ARRAY_AGG(tag_name));  -- Filter by tag presence
```

### 3. Timeline Creation
```sql
-- Create activity timeline for users
SELECT 
    user_id,
    ARRAY_AGG(
        activity_type || ' at ' || activity_timestamp::TEXT 
        ORDER BY activity_timestamp
    ) AS activity_timeline,
    ARRAY_AGG(DISTINCT activity_type) AS activity_types,
    ARRAY_AGG(activity_timestamp ORDER BY activity_timestamp) AS timestamps
FROM user_activities
WHERE activity_timestamp >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY user_id;
```

### 4. Denormalization for Performance
```sql
-- Denormalize related data into arrays
SELECT 
    p.product_id,
    p.product_name,
    ARRAY_AGG(DISTINCT c.color) AS available_colors,
    ARRAY_AGG(DISTINCT s.size) AS available_sizes,
    ARRAY_AGG(r.rating) AS all_ratings,
    AVG(r.rating) AS avg_rating
FROM products p
LEFT JOIN product_colors c ON p.product_id = c.product_id
LEFT JOIN product_sizes s ON p.product_id = s.product_id
LEFT JOIN reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.product_name;
```

### 5. Window Function with ARRAY_AGG
```sql
-- Collect values over a window
SELECT 
    date,
    sales_amount,
    ARRAY_AGG(sales_amount) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS last_7_days_sales,
    -- Calculate median from array
    (ARRAY_AGG(sales_amount) OVER (
        ORDER BY date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ))[ARRAY_LENGTH(
        ARRAY_AGG(sales_amount) OVER (
            ORDER BY date 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ), 1) / 2 + 1] AS rolling_median
FROM daily_sales
ORDER BY date;
```

## Array Operations

```sql
-- Working with aggregated arrays
WITH user_products AS (
    SELECT 
        customer_id,
        ARRAY_AGG(DISTINCT product_id) AS product_ids
    FROM orders
    GROUP BY customer_id
)
SELECT 
    a.customer_id AS customer_a,
    b.customer_id AS customer_b,
    a.product_ids AS products_a,
    b.product_ids AS products_b,
    -- Common products (intersection)
    ARRAY(
        SELECT UNNEST(a.product_ids) 
        INTERSECT 
        SELECT UNNEST(b.product_ids)
    ) AS common_products,
    -- Jaccard similarity
    CARDINALITY(ARRAY(
        SELECT UNNEST(a.product_ids) INTERSECT SELECT UNNEST(b.product_ids)
    ))::FLOAT / 
    CARDINALITY(ARRAY(
        SELECT UNNEST(a.product_ids) UNION SELECT UNNEST(b.product_ids)
    )) AS similarity_score
FROM user_products a
CROSS JOIN user_products b
WHERE a.customer_id < b.customer_id
    AND CARDINALITY(ARRAY(
        SELECT UNNEST(a.product_ids) INTERSECT SELECT UNNEST(b.product_ids)
    )) > 0;
```

## Filtering and Conditional Aggregation

```sql
-- Conditional array aggregation
SELECT 
    department,
    ARRAY_AGG(employee_name ORDER BY salary DESC) AS all_employees,
    ARRAY_AGG(employee_name ORDER BY salary DESC) 
        FILTER (WHERE salary > 50000) AS high_earners,
    ARRAY_AGG(CASE WHEN performance_rating >= 4 THEN employee_name END) 
        AS top_performers,
    ARRAY_AGG(DISTINCT skill) AS department_skills
FROM employees
LEFT JOIN employee_skills USING (employee_id)
GROUP BY department;
```

## NULL Handling

```sql
-- ARRAY_AGG behavior with NULLs
WITH sample_data AS (
    SELECT * FROM (VALUES 
        (1, 'A', 100),
        (1, 'B', NULL),
        (1, NULL, 300),
        (2, 'C', 200),
        (2, NULL, NULL)
    ) AS t(group_id, text_val, num_val)
)
SELECT 
    group_id,
    ARRAY_AGG(text_val) AS text_array,  -- NULLs excluded
    ARRAY_AGG(num_val) AS num_array,    -- NULLs excluded
    ARRAY_AGG(COALESCE(text_val, 'N/A')) AS text_with_default,
    ARRAY_LENGTH(ARRAY_AGG(text_val), 1) AS array_length
FROM sample_data
GROUP BY group_id;
```

## Performance Considerations

```sql
-- Optimize ARRAY_AGG for large datasets
-- Limit array size when possible
SELECT 
    category,
    -- Limit to top 10 products by sales
    ARRAY_AGG(product_name ORDER BY total_sales DESC LIMIT 10) AS top_products,
    -- Use sampling for very large groups
    ARRAY_AGG(product_name) FILTER (WHERE RANDOM() < 0.1) AS sample_products
FROM (
    SELECT 
        category,
        product_name,
        SUM(sales_amount) AS total_sales
    FROM sales
    GROUP BY category, product_name
) product_sales
GROUP BY category;

-- Create index for ordered aggregation
CREATE INDEX idx_orders_customer_date ON customer_orders(customer_id, order_date);
```

## Related Functions
- [STRING_AGG](./string_agg.md) - Concatenates strings with delimiter
- [JSON_AGG](./json_agg.md) - Aggregates values into JSON array
- [UNNEST](../array/unnest.md) - Expands array into rows
- [ARRAY_TO_STRING](../array/array_to_string.md) - Converts array to string
- [CARDINALITY](../array/cardinality.md) - Returns array length