---
title: ROUND
description: Rounds a numeric value to a specified number of decimal places
---

# ROUND

## Description
The `ROUND` function rounds a numeric value to a specified number of decimal places. When the decimal places parameter is positive, it rounds to that many decimal places. When negative, it rounds to the left of the decimal point. When omitted, it rounds to the nearest integer. The function uses standard rounding rules: values of 5 or greater round up, values less than 5 round down.

## Syntax
```sql
ROUND(numeric_expression [, decimal_places])
```

### Parameters
- **numeric_expression** (numeric): The number to be rounded
- **decimal_places** (integer, optional): Number of decimal places to round to. Default is 0.
  - Positive values: Round to N decimal places
  - Negative values: Round to N places left of decimal point
  - Zero or omitted: Round to nearest integer

### Return Value
- **Type**: Same as input type (maintains precision type)
- **Description**: The rounded value according to specified precision

## Example

### Sample Data
Consider the following `product_metrics` table with various measurements:

| product_id | product_name | weight_kg | price    | rating  | quantity |
|------------|-------------|-----------|----------|---------|----------|
| 1          | Laptop      | 2.4567    | 1299.99  | 4.678   | 145      |
| 2          | Mouse       | 0.1234    | 29.456   | 4.234   | 523      |
| 3          | Keyboard    | 0.8901    | 79.999   | 3.567   | 287      |
| 4          | Monitor     | 5.6789    | 399.444  | 4.891   | 92       |
| 5          | Headphones  | 0.3456    | 89.555   | 4.123   | 1456     |

### Query
```sql
SELECT 
    product_id,
    product_name,
    price,
    ROUND(price) AS price_rounded,
    ROUND(price, 1) AS price_1_decimal,
    rating,
    ROUND(rating, 2) AS rating_2_decimal,
    quantity,
    ROUND(quantity, -1) AS quantity_tens,
    ROUND(quantity, -2) AS quantity_hundreds
FROM product_metrics
ORDER BY product_id;
```

### Result
| product_id | product_name | price    | price_rounded | price_1_decimal | rating | rating_2_decimal | quantity | quantity_tens | quantity_hundreds |
|------------|-------------|----------|---------------|-----------------|--------|------------------|----------|---------------|-------------------|
| 1          | Laptop      | 1299.99  | 1300          | 1300.0          | 4.678  | 4.68             | 145      | 150           | 100               |
| 2          | Mouse       | 29.456   | 29            | 29.5            | 4.234  | 4.23             | 523      | 520           | 500               |
| 3          | Keyboard    | 79.999   | 80            | 80.0            | 3.567  | 3.57             | 287      | 290           | 300               |
| 4          | Monitor     | 399.444  | 399           | 399.4           | 4.891  | 4.89             | 92       | 90            | 100               |
| 5          | Headphones  | 89.555   | 90            | 89.6            | 4.123  | 4.12             | 1456     | 1460          | 1500              |

## Common Use Cases

### 1. Financial Calculations
```sql
-- Round monetary values to 2 decimal places
SELECT 
    order_id,
    subtotal,
    tax_rate,
    ROUND(subtotal * tax_rate, 2) AS tax_amount,
    ROUND(subtotal * (1 + tax_rate), 2) AS total_amount
FROM orders;
```

### 2. Statistical Reporting
```sql
-- Round percentages for display
SELECT 
    category,
    sales,
    total_sales,
    ROUND(sales / total_sales * 100, 1) AS percentage_of_total
FROM category_sales
CROSS JOIN (SELECT SUM(sales) as total_sales FROM category_sales) t;
```

### 3. Inventory Management
```sql
-- Round to nearest packaging unit (e.g., cases of 12)
SELECT 
    product_name,
    units_needed,
    ROUND(units_needed / 12.0) * 12 AS cases_to_order
FROM inventory_requirements;
```

### 4. Data Aggregation
```sql
-- Round averages for cleaner reporting
SELECT 
    department,
    ROUND(AVG(salary), -3) AS avg_salary_thousands,
    ROUND(AVG(years_experience), 1) AS avg_experience
FROM employees
GROUP BY department;
```

## Edge Cases and Notes
- Rounding 0.5 rounds up to 1 (banker's rounding may vary by system)
- Negative numbers: -2.5 rounds to -3 (away from zero)
- Very large numbers may lose precision due to floating-point limitations
- NULL input returns NULL

## Related Functions
- [TRUNC](./trunc.md) - Truncates a number without rounding
- [CEIL](./ceil.md) - Rounds up to the nearest integer
- [FLOOR](./floor.md) - Rounds down to the nearest integer
- [ABS](./abs.md) - Returns the absolute value