---
title: COALESCE
description: Returns the first non-NULL value from a list of expressions
---

# COALESCE

## Description
The `COALESCE` function returns the first non-NULL value from a list of expressions. It evaluates expressions from left to right and returns the first expression that is not NULL. If all expressions are NULL, it returns NULL. This function is essential for handling NULL values in SQL, providing default values, and combining data from multiple columns where some might be NULL.

## Syntax
```sql
COALESCE(expression1, expression2, ..., expressionN)
```

### Parameters
- **expression1, expression2, ..., expressionN** (any type): A list of expressions to evaluate. All expressions should be of compatible data types.

### Return Value
- **Type**: Same as the input expressions (must be compatible types)
- **Description**: The first non-NULL expression, or NULL if all expressions are NULL

## Example

### Sample Data
Consider the following `customer_contacts` table with multiple contact methods:

| customer_id | name          | mobile_phone   | work_phone     | home_phone     | email                | preferred_contact |
|-------------|---------------|----------------|----------------|----------------|----------------------|-------------------|
| 1           | John Smith    | +1-555-0101    | +1-555-0201    | NULL           | john@example.com     | mobile            |
| 2           | Jane Doe      | NULL           | +1-555-0202    | +1-555-0302    | jane@example.com     | email             |
| 3           | Bob Wilson    | +1-555-0103    | NULL           | NULL           | NULL                 | NULL              |
| 4           | Alice Brown   | NULL           | NULL           | +1-555-0304    | alice@example.com    | home              |
| 5           | Charlie Davis | NULL           | NULL           | NULL           | charlie@example.com  | email             |

### Query
```sql
SELECT 
    customer_id,
    name,
    -- Get first available phone number
    COALESCE(mobile_phone, work_phone, home_phone) AS primary_phone,
    -- Get first available contact method
    COALESCE(mobile_phone, work_phone, home_phone, email) AS any_contact,
    -- Provide default if no phone available
    COALESCE(mobile_phone, work_phone, home_phone, 'No phone') AS phone_or_default,
    -- Use preferred contact or fallback
    COALESCE(
        CASE preferred_contact
            WHEN 'mobile' THEN mobile_phone
            WHEN 'work' THEN work_phone
            WHEN 'home' THEN home_phone
            WHEN 'email' THEN email
        END,
        mobile_phone,
        work_phone,
        home_phone,
        email,
        'No contact info'
    ) AS best_contact
FROM customer_contacts
ORDER BY customer_id;
```

### Result
| customer_id | name          | primary_phone  | any_contact        | phone_or_default | best_contact        |
|-------------|---------------|----------------|--------------------|------------------|---------------------|
| 1           | John Smith    | +1-555-0101    | +1-555-0101        | +1-555-0101      | +1-555-0101         |
| 2           | Jane Doe      | +1-555-0202    | +1-555-0202        | +1-555-0202      | jane@example.com    |
| 3           | Bob Wilson    | +1-555-0103    | +1-555-0103        | +1-555-0103      | +1-555-0103         |
| 4           | Alice Brown   | +1-555-0304    | +1-555-0304        | +1-555-0304      | +1-555-0304         |
| 5           | Charlie Davis | NULL           | charlie@example.com | No phone         | charlie@example.com |

## Common Use Cases

### 1. Providing Default Values
```sql
-- Set default values for NULL columns
SELECT 
    product_id,
    product_name,
    COALESCE(description, 'No description available') AS description,
    COALESCE(price, 0.00) AS price,
    COALESCE(discount_percent, 0) AS discount,
    COALESCE(stock_quantity, 0) AS stock
FROM products;
```

### 2. Handling Optional Parameters
```sql
-- Use parameter or default value in WHERE clause
SELECT * 
FROM orders
WHERE order_date >= COALESCE(@start_date, '2024-01-01')
  AND order_date <= COALESCE(@end_date, CURRENT_DATE)
  AND status = COALESCE(@status, status);  -- If @status is NULL, matches all
```

### 3. Concatenating Names with Fallbacks
```sql
-- Build display name with fallbacks
SELECT 
    user_id,
    COALESCE(
        display_name,
        full_name,
        first_name || ' ' || last_name,
        username,
        email,
        'User #' || user_id
    ) AS display_name
FROM users;
```

### 4. Calculating with NULL Handling
```sql
-- Calculate total with optional components
SELECT 
    order_id,
    subtotal,
    COALESCE(discount_amount, 0) AS discount,
    COALESCE(tax_amount, subtotal * 0.08) AS tax,  -- Default 8% tax if NULL
    COALESCE(shipping_cost, 
        CASE 
            WHEN subtotal > 100 THEN 0  -- Free shipping over $100
            ELSE 10  -- Standard shipping
        END
    ) AS shipping,
    subtotal 
    - COALESCE(discount_amount, 0) 
    + COALESCE(tax_amount, subtotal * 0.08)
    + COALESCE(shipping_cost, 10) AS total
FROM orders;
```

### 5. Data Migration and Consolidation
```sql
-- Merge data from old and new systems
SELECT 
    COALESCE(new.customer_id, old.cust_id) AS customer_id,
    COALESCE(new.email, old.email_address, old.contact_email) AS email,
    COALESCE(new.created_date, old.signup_date, old.first_order_date) AS customer_since,
    COALESCE(new.status, 
        CASE old.active 
            WHEN 1 THEN 'active' 
            WHEN 0 THEN 'inactive' 
        END,
        'unknown'
    ) AS status
FROM new_customers new
FULL OUTER JOIN old_customers old 
    ON new.legacy_id = old.cust_id;
```

### 6. Handling Multiple Currency Columns
```sql
-- Get amount in preferred currency order
SELECT 
    transaction_id,
    COALESCE(
        amount_usd,
        amount_eur * 1.09,  -- Convert EUR to USD
        amount_gbp * 1.27,  -- Convert GBP to USD
        amount_jpy / 110    -- Convert JPY to USD
    ) AS amount_in_usd
FROM international_transactions;
```

## Comparison with Other NULL Functions

```sql
-- Comparing COALESCE with other NULL handling functions
WITH sample_data AS (
    SELECT 1 AS id, NULL AS val1, NULL AS val2, 'C' AS val3
    UNION ALL SELECT 2, 'A', NULL, 'C'
    UNION ALL SELECT 3, NULL, 'B', 'C'
    UNION ALL SELECT 4, 'A', 'B', 'C'
)
SELECT 
    id,
    val1,
    val2,
    val3,
    COALESCE(val1, val2, val3) AS coalesce_result,
    IFNULL(val1, val2) AS ifnull_result,  -- Only 2 arguments
    NVL(val1, 'default') AS nvl_result,    -- Oracle-style, 2 arguments
    CASE 
        WHEN val1 IS NOT NULL THEN val1
        WHEN val2 IS NOT NULL THEN val2
        ELSE val3
    END AS case_equivalent
FROM sample_data;
```

## Performance Tips

```sql
-- Efficient use of COALESCE in indexes and queries

-- Create a functional index on COALESCE expression
CREATE INDEX idx_primary_contact 
ON customers (COALESCE(mobile_phone, work_phone, home_phone));

-- Use COALESCE in JOIN conditions
SELECT 
    o.order_id,
    c.customer_name
FROM orders o
JOIN customers c ON c.customer_id = COALESCE(o.customer_id, o.guest_id);
```

## Edge Cases

```sql
-- COALESCE with different data types (requires compatible types)
SELECT 
    -- This works: all numeric types
    COALESCE(int_col, decimal_col, 0) AS numeric_coalesce,
    
    -- This works: all string types
    COALESCE(varchar_col, text_col, 'default') AS string_coalesce,
    
    -- This may fail: mixed types
    -- COALESCE(int_col, varchar_col) -- Error: incompatible types
    
    -- Solution: Cast to common type
    COALESCE(CAST(int_col AS VARCHAR), varchar_col) AS mixed_coalesce
FROM mixed_types_table;
```

## Related Functions
- [NULLIF](./nullif.md) - Returns NULL if two expressions are equal
- [IFNULL](./ifnull.md) - Two-argument version of COALESCE
- [NVL](./nvl.md) - Oracle-compatible NULL replacement
- [CASE](./case.md) - Conditional expressions with multiple conditions
- [IS NULL](./is_null.md) - Tests for NULL values