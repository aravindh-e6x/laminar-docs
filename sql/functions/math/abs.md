---
title: ABS
description: Returns the absolute value of a numeric expression
---

# ABS

## Description
The `ABS` function returns the absolute (positive) value of a numeric expression. It removes the sign from negative numbers while leaving positive numbers and zero unchanged. This function is commonly used when you need to work with magnitudes regardless of direction, such as calculating distances, differences, or deviations.

## Syntax
```sql
ABS(numeric_expression)
```

### Parameters
- **numeric_expression** (numeric): Any numeric value or expression that evaluates to a number

### Return Value
- **Type**: Same as input type (INTEGER, DECIMAL, FLOAT, etc.)
- **Description**: The absolute value of the input. Always returns a non-negative number.

## Example

### Sample Data
Consider the following `temperature_changes` table tracking daily temperature variations:

| day_id | city         | morning_temp | evening_temp | temp_change |
|--------|--------------|-------------|--------------|-------------|
| 1      | New York     | 15.5        | 22.3         | 6.8         |
| 2      | Los Angeles  | 18.2        | 16.7         | -1.5        |
| 3      | Chicago      | 8.9         | 5.4          | -3.5        |
| 4      | Houston      | 25.1        | 28.6         | 3.5         |
| 5      | Phoenix      | 31.2        | 27.8         | -3.4        |

### Query
```sql
SELECT 
    day_id,
    city,
    temp_change,
    ABS(temp_change) AS absolute_change
FROM temperature_changes
ORDER BY day_id;
```

### Result
| day_id | city         | temp_change | absolute_change |
|--------|--------------|-------------|-----------------|
| 1      | New York     | 6.8         | 6.8             |
| 2      | Los Angeles  | -1.5        | 1.5             |
| 3      | Chicago      | -3.5        | 3.5             |
| 4      | Houston      | 3.5         | 3.5             |
| 5      | Phoenix      | -3.4        | 3.4             |

## Common Use Cases

### 1. Calculating Distance or Difference
```sql
-- Find products with price variance from target price
SELECT 
    product_name,
    current_price,
    target_price,
    ABS(current_price - target_price) AS price_variance
FROM products
WHERE ABS(current_price - target_price) > 10;
```

### 2. Finding Outliers
```sql
-- Find transactions that deviate significantly from average
SELECT 
    transaction_id,
    amount,
    avg_amount,
    ABS(amount - avg_amount) AS deviation
FROM transactions
CROSS JOIN (SELECT AVG(amount) as avg_amount FROM transactions) avg_t
WHERE ABS(amount - avg_amount) > 1000;
```

### 3. Calculating Absolute Percentage Change
```sql
-- Calculate absolute percentage change in sales
SELECT 
    month,
    this_year_sales,
    last_year_sales,
    ABS((this_year_sales - last_year_sales) / last_year_sales * 100) AS abs_percent_change
FROM monthly_sales;
```

## Related Functions
- [SIGN](./sign.md) - Returns the sign of a number (-1, 0, or 1)
- [ROUND](./round.md) - Rounds a number to specified decimal places
- [CEIL](./ceil.md) - Returns the smallest integer greater than or equal to a number
- [FLOOR](./floor.md) - Returns the largest integer less than or equal to a number