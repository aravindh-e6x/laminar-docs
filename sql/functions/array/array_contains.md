---
title: ARRAY_CONTAINS
description: Checks if an array contains a specific element
---

# ARRAY_CONTAINS

## Description
The `ARRAY_CONTAINS` function checks whether an array contains a specific element. It returns TRUE if the element is found in the array, FALSE otherwise. This function is essential for filtering data based on array membership, such as finding records with specific tags, categories, or attributes stored in array columns. The function performs an exact match comparison and is case-sensitive for string elements.

## Syntax
```sql
ARRAY_CONTAINS(array_expression, search_element)
```

### Parameters
- **array_expression** (array): The array to search in
- **search_element** (any): The element to search for. Must be compatible with the array element type.

### Return Value
- **Type**: BOOLEAN
- **Description**: Returns TRUE if the element exists in the array, FALSE if not found, NULL if either argument is NULL

## Example

### Sample Data
Consider the following `products` table with tags stored as arrays:

| product_id | product_name | categories                      | tags                               | sizes        | prices        |
|------------|--------------|----------------------------------|-------------------------------------|--------------|---------------|
| 1          | T-Shirt      | ['Clothing', 'Men']             | ['cotton', 'casual', 'summer']    | ['S','M','L']| [19.99, 24.99]|
| 2          | Laptop       | ['Electronics', 'Computers']    | ['portable', 'work', 'gaming']    | NULL         | [899.99]      |
| 3          | Coffee Mug   | ['Kitchen', 'Drinkware']        | ['ceramic', 'microwave-safe']     | ['250ml']    | [12.99, 14.99]|
| 4          | Backpack     | ['Accessories', 'Travel']       | ['waterproof', 'laptop', 'travel']| ['One Size'] | [59.99]       |
| 5          | Notebook     | ['Stationery', 'Office']        | ['ruled', 'a4', 'spiral']         | ['A4', 'A5'] | [4.99, 6.99]  |

### Query
```sql
SELECT 
    product_id,
    product_name,
    categories,
    tags,
    ARRAY_CONTAINS(categories, 'Electronics') AS is_electronic,
    ARRAY_CONTAINS(tags, 'laptop') AS has_laptop_tag,
    ARRAY_CONTAINS(tags, 'cotton') AS is_cotton,
    ARRAY_CONTAINS(sizes, 'M') AS has_medium_size,
    ARRAY_CONTAINS(prices, 19.99) AS has_price_1999
FROM products
ORDER BY product_id;
```

### Result
| product_id | product_name | categories                    | tags                               | is_electronic | has_laptop_tag | is_cotton | has_medium_size | has_price_1999 |
|------------|--------------|-------------------------------|-------------------------------------|---------------|----------------|-----------|-----------------|----------------|
| 1          | T-Shirt      | ['Clothing', 'Men']          | ['cotton', 'casual', 'summer']    | false         | false          | true      | true            | true           |
| 2          | Laptop       | ['Electronics', 'Computers']  | ['portable', 'work', 'gaming']    | true          | false          | false     | NULL            | false          |
| 3          | Coffee Mug   | ['Kitchen', 'Drinkware']      | ['ceramic', 'microwave-safe']     | false         | false          | false     | false           | false          |
| 4          | Backpack     | ['Accessories', 'Travel']     | ['waterproof', 'laptop', 'travel']| false         | true           | false     | false           | false          |
| 5          | Notebook     | ['Stationery', 'Office']      | ['ruled', 'a4', 'spiral']         | false         | false          | false     | false           | false          |

## Common Use Cases

### 1. Filtering by Tags
```sql
-- Find all products with specific tags
SELECT 
    product_name,
    tags,
    price
FROM products
WHERE ARRAY_CONTAINS(tags, 'waterproof')
   OR ARRAY_CONTAINS(tags, 'water-resistant');
```

### 2. Multi-Category Search
```sql
-- Find products in multiple categories
SELECT 
    product_id,
    product_name,
    categories
FROM products
WHERE ARRAY_CONTAINS(categories, 'Electronics')
   OR ARRAY_CONTAINS(categories, 'Computers')
   OR ARRAY_CONTAINS(categories, 'Gadgets');
```

### 3. User Permissions Check
```sql
-- Check if user has required permission
SELECT 
    user_id,
    username,
    roles,
    CASE 
        WHEN ARRAY_CONTAINS(roles, 'admin') THEN 'Full Access'
        WHEN ARRAY_CONTAINS(roles, 'editor') THEN 'Edit Access'
        WHEN ARRAY_CONTAINS(roles, 'viewer') THEN 'Read Only'
        ELSE 'No Access'
    END AS access_level
FROM users;
```

### 4. Finding Common Interests
```sql
-- Find users with specific interests
WITH user_interests AS (
    SELECT 
        user_id,
        username,
        interests
    FROM user_profiles
)
SELECT 
    u1.username AS user1,
    u2.username AS user2,
    u1.interests AS user1_interests,
    u2.interests AS user2_interests
FROM user_interests u1
JOIN user_interests u2 ON u1.user_id < u2.user_id
WHERE ARRAY_CONTAINS(u1.interests, 'hiking')
  AND ARRAY_CONTAINS(u2.interests, 'hiking');
```

### 5. Inventory Availability
```sql
-- Check product availability in specific sizes
SELECT 
    product_name,
    available_sizes,
    CASE 
        WHEN ARRAY_CONTAINS(available_sizes, 'M') 
         AND ARRAY_CONTAINS(available_sizes, 'L') 
        THEN 'Both M and L available'
        WHEN ARRAY_CONTAINS(available_sizes, 'M') 
        THEN 'Only M available'
        WHEN ARRAY_CONTAINS(available_sizes, 'L') 
        THEN 'Only L available'
        ELSE 'Neither M nor L available'
    END AS size_availability
FROM inventory
WHERE category = 'Clothing';
```

### 6. Feature Flags
```sql
-- Check if features are enabled for users
SELECT 
    user_id,
    email,
    enabled_features,
    ARRAY_CONTAINS(enabled_features, 'dark_mode') AS has_dark_mode,
    ARRAY_CONTAINS(enabled_features, 'beta_features') AS is_beta_tester,
    ARRAY_CONTAINS(enabled_features, 'premium') AS is_premium
FROM user_settings
WHERE ARRAY_CONTAINS(enabled_features, 'notifications');
```

## Combining with Other Array Functions

```sql
-- Complex array operations
SELECT 
    product_id,
    product_name,
    tags,
    ARRAY_LENGTH(tags) AS tag_count,
    ARRAY_CONTAINS(tags, 'sale') AS on_sale,
    CASE 
        WHEN ARRAY_CONTAINS(tags, 'new') AND ARRAY_CONTAINS(tags, 'featured') 
        THEN 'New & Featured'
        WHEN ARRAY_CONTAINS(tags, 'new') 
        THEN 'New Arrival'
        WHEN ARRAY_CONTAINS(tags, 'featured') 
        THEN 'Featured'
        ELSE 'Regular'
    END AS product_status
FROM products
WHERE ARRAY_LENGTH(tags) > 0;
```

## NULL Handling

```sql
-- Demonstrating NULL behavior
WITH test_data AS (
    SELECT 1 AS id, ARRAY[1, 2, 3] AS arr
    UNION ALL SELECT 2, ARRAY[4, 5, NULL]
    UNION ALL SELECT 3, NULL
    UNION ALL SELECT 4, ARRAY[]
)
SELECT 
    id,
    arr,
    ARRAY_CONTAINS(arr, 2) AS contains_2,
    ARRAY_CONTAINS(arr, NULL) AS contains_null,
    ARRAY_CONTAINS(NULL, 2) AS null_array_check,
    ARRAY_CONTAINS(arr, 999) AS contains_999
FROM test_data;
```

## Performance Tips
- Create indexes on array columns for better performance
- Consider using GIN indexes for PostgreSQL-based systems
- For large arrays, consider normalizing data into separate tables
- Use ARRAY_CONTAINS early in WHERE clauses for better optimization

## Related Functions
- [ARRAY_POSITION](./array_position.md) - Returns position of element in array
- [ARRAY_HAS_ANY](./array_has_any.md) - Checks if arrays have common elements
- [ARRAY_HAS_ALL](./array_has_all.md) - Checks if array contains all specified elements
- [ARRAY_LENGTH](./array_length.md) - Returns the length of an array
- [ARRAY_REMOVE](./array_remove.md) - Removes elements from array