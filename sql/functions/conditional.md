---
sidebar_position: 8
---

# Conditional Functions

Functions for conditional logic and control flow in SQL queries.

## CASE Expressions

### CASE WHEN
Evaluates conditions and returns values.
```sql
-- Simple CASE
SELECT 
    CASE status
        WHEN 'active' THEN 'Active User'
        WHEN 'inactive' THEN 'Inactive User'
        WHEN 'suspended' THEN 'Suspended'
        ELSE 'Unknown'
    END as status_label
FROM users;

-- Searched CASE
SELECT 
    CASE 
        WHEN age < 18 THEN 'Minor'
        WHEN age BETWEEN 18 AND 65 THEN 'Adult'
        WHEN age > 65 THEN 'Senior'
        ELSE 'Unknown'
    END as age_group
FROM users;
```

## Null Handling

### coalesce(value1, value2, ...)
Returns first non-null value.
```sql
SELECT coalesce(NULL, NULL, 'default', 'other'); 
-- Returns: 'default'

SELECT coalesce(phone, mobile, email, 'No contact') as contact
FROM users;
```

### nullif(value1, value2)
Returns NULL if values are equal, otherwise value1.
```sql
SELECT nullif(5, 5); -- Returns: NULL
SELECT nullif(5, 3); -- Returns: 5

-- Avoid division by zero
SELECT amount / nullif(quantity, 0) as unit_price
FROM orders;
```

### ifnull(value, default) / nvl(value, default)
Returns default if value is NULL.
```sql
SELECT ifnull(NULL, 'default'); -- Returns: 'default'
SELECT ifnull('value', 'default'); -- Returns: 'value'

SELECT nvl(discount, 0) as discount_amount
FROM orders;
```

### nvl2(value, if_not_null, if_null)
Returns if_not_null if value is not NULL, otherwise if_null.
```sql
SELECT nvl2(bonus, salary + bonus, salary) as total_pay
FROM employees;
```

### isnan(value)
Checks if value is NaN.
```sql
SELECT isnan(0.0 / 0.0); -- Returns: true
SELECT isnan(1.0); -- Returns: false
```

### isfinite(value)
Checks if value is finite (not infinite or NaN).
```sql
SELECT isfinite(1.0); -- Returns: true
SELECT isfinite(1.0 / 0.0); -- Returns: false
```

### isinf(value)
Checks if value is infinite.
```sql
SELECT isinf(1.0 / 0.0); -- Returns: true
SELECT isinf(1.0); -- Returns: false
```

### iszero(value)
Checks if value is zero.
```sql
SELECT iszero(0); -- Returns: true
SELECT iszero(0.0); -- Returns: true
SELECT iszero(1); -- Returns: false
```

### nanvl(value, default)
Returns default if value is NaN.
```sql
SELECT nanvl(0.0 / 0.0, 0); -- Returns: 0
SELECT nanvl(1.0, 0); -- Returns: 1.0
```

## Comparison Functions

### greatest(value1, value2, ...)
Returns the greatest value.
```sql
SELECT greatest(1, 5, 3, 9, 2); -- Returns: 9
SELECT greatest('apple', 'banana', 'cherry'); -- Returns: 'cherry'

SELECT greatest(date1, date2, date3) as latest_date
FROM events;
```

### least(value1, value2, ...)
Returns the smallest value.
```sql
SELECT least(1, 5, 3, 9, 2); -- Returns: 1
SELECT least('apple', 'banana', 'cherry'); -- Returns: 'apple'

SELECT least(price1, price2, price3) as best_price
FROM price_comparison;
```

## Logical Functions

### and / AND operator
Logical AND.
```sql
SELECT true AND true; -- Returns: true
SELECT true AND false; -- Returns: false
SELECT true AND NULL; -- Returns: NULL
```

### or / OR operator
Logical OR.
```sql
SELECT true OR false; -- Returns: true
SELECT false OR false; -- Returns: false
SELECT false OR NULL; -- Returns: NULL
```

### not / NOT operator
Logical NOT.
```sql
SELECT NOT true; -- Returns: false
SELECT NOT false; -- Returns: true
SELECT NOT NULL; -- Returns: NULL
```

### xor(bool1, bool2)
Logical XOR (exclusive OR).
```sql
SELECT xor(true, true); -- Returns: false
SELECT xor(true, false); -- Returns: true
SELECT xor(false, false); -- Returns: false
```

## Type Checking

### typeof(value)
Returns the data type of value.
```sql
SELECT typeof(123); -- Returns: 'INTEGER'
SELECT typeof(3.14); -- Returns: 'DOUBLE'
SELECT typeof('hello'); -- Returns: 'VARCHAR'
SELECT typeof(current_date); -- Returns: 'DATE'
```

## Decode Function

### decode(value, search1, result1 [, search2, result2, ...] [, default])
Compares value to search values and returns corresponding result.
```sql
SELECT decode(
    status,
    'A', 'Active',
    'I', 'Inactive',
    'S', 'Suspended',
    'Unknown'
) as status_name
FROM users;

-- Equivalent to:
SELECT 
    CASE status
        WHEN 'A' THEN 'Active'
        WHEN 'I' THEN 'Inactive'
        WHEN 'S' THEN 'Suspended'
        ELSE 'Unknown'
    END as status_name
FROM users;
```

## IF Function

### if(condition, true_value, false_value)
Returns true_value if condition is true, otherwise false_value.
```sql
SELECT if(age >= 18, 'Adult', 'Minor') as age_category
FROM users;

SELECT if(quantity > 0, 'In Stock', 'Out of Stock') as availability
FROM products;
```

## Usage Examples

### Handle NULL values in calculations
```sql
SELECT 
    product_name,
    price * (1 - coalesce(discount, 0)) as final_price
FROM products;
```

### Conditional aggregation
```sql
SELECT 
    COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
    COUNT(CASE WHEN status = 'pending' THEN 1 END) as pending,
    COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed
FROM orders;
```

### Dynamic sorting
```sql
SELECT * FROM products
ORDER BY 
    CASE 
        WHEN @sort_by = 'price' THEN price
        WHEN @sort_by = 'name' THEN name
        ELSE created_at
    END;
```

### Categorization
```sql
SELECT 
    customer_id,
    CASE 
        WHEN total_purchases > 10000 THEN 'Platinum'
        WHEN total_purchases > 5000 THEN 'Gold'
        WHEN total_purchases > 1000 THEN 'Silver'
        ELSE 'Bronze'
    END as customer_tier
FROM (
    SELECT 
        customer_id,
        SUM(amount) as total_purchases
    FROM orders
    GROUP BY customer_id
);
```

### Safe division
```sql
SELECT 
    revenue,
    costs,
    CASE 
        WHEN costs = 0 THEN NULL
        ELSE revenue / costs
    END as roi
FROM financial_data;
```

### Complex conditions
```sql
SELECT 
    user_id,
    CASE 
        WHEN last_login IS NULL THEN 'Never logged in'
        WHEN last_login < current_date - INTERVAL '30 days' THEN 'Inactive'
        WHEN last_login < current_date - INTERVAL '7 days' THEN 'Active'
        ELSE 'Very Active'
    END as activity_status
FROM users;
```