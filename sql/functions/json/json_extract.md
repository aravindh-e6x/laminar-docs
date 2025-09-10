---
title: JSON_EXTRACT
description: Extracts a value from a JSON document using a JSON path expression
---

# JSON_EXTRACT

## Description
The `JSON_EXTRACT` function extracts a value from a JSON document using a JSON path expression. It allows you to navigate through complex JSON structures and retrieve specific values, whether they are scalars, objects, or arrays. The function returns the extracted value as a JSON string, preserving the JSON formatting. This is essential for working with semi-structured data stored in JSON columns.

## Syntax
```sql
JSON_EXTRACT(json_document, json_path)
```

### Parameters
- **json_document** (string/json): The JSON document to extract from
- **json_path** (string): A JSON path expression starting with '$' that specifies what to extract

### Return Value
- **Type**: JSON/STRING
- **Description**: The extracted JSON value as a string. Returns NULL if the path doesn't exist or if either argument is NULL.

## Example

### Sample Data
Consider the following `user_profiles` table with JSON data:

| user_id | username | profile_data                                                                                      |
|---------|----------|---------------------------------------------------------------------------------------------------|
| 1       | alice    | `{"name": "Alice Smith", "age": 28, "email": "alice@example.com", "settings": {"theme": "dark", "notifications": true}, "hobbies": ["reading", "hiking"]}` |
| 2       | bob      | `{"name": "Bob Jones", "age": 35, "email": "bob@example.com", "settings": {"theme": "light", "notifications": false}, "hobbies": ["gaming", "cooking", "music"]}` |
| 3       | charlie  | `{"name": "Charlie Brown", "age": 42, "email": "charlie@example.com", "address": {"city": "New York", "zip": "10001"}, "hobbies": ["photography"]}` |
| 4       | diana    | `{"name": "Diana Prince", "age": 31, "phone": "+1234567890", "settings": {"theme": "auto", "language": "es"}, "scores": [85, 92, 88]}` |
| 5       | eve      | `{"name": "Eve Anderson", "contact": {"email": "eve@example.com", "phone": "+9876543210"}, "preferences": {"newsletter": true}}` |

### Query
```sql
SELECT 
    user_id,
    username,
    JSON_EXTRACT(profile_data, '$.name') AS full_name,
    JSON_EXTRACT(profile_data, '$.age') AS age,
    JSON_EXTRACT(profile_data, '$.email') AS email,
    JSON_EXTRACT(profile_data, '$.settings.theme') AS theme,
    JSON_EXTRACT(profile_data, '$.hobbies[0]') AS first_hobby,
    JSON_EXTRACT(profile_data, '$.address.city') AS city
FROM user_profiles
ORDER BY user_id;
```

### Result
| user_id | username | full_name        | age  | email                  | theme  | first_hobby  | city       |
|---------|----------|------------------|------|------------------------|--------|--------------|------------|
| 1       | alice    | "Alice Smith"    | 28   | "alice@example.com"    | "dark" | "reading"    | NULL       |
| 2       | bob      | "Bob Jones"      | 35   | "bob@example.com"      | "light"| "gaming"     | NULL       |
| 3       | charlie  | "Charlie Brown"  | 42   | "charlie@example.com"  | NULL   | "photography"| "New York" |
| 4       | diana    | "Diana Prince"   | 31   | NULL                   | "auto" | NULL         | NULL       |
| 5       | eve      | "Eve Anderson"   | NULL | NULL                   | NULL   | NULL         | NULL       |

## Common Use Cases

### 1. Extracting Nested Values
```sql
-- Extract deeply nested configuration values
SELECT 
    app_id,
    JSON_EXTRACT(config, '$.database.connection.host') AS db_host,
    JSON_EXTRACT(config, '$.database.connection.port') AS db_port,
    JSON_EXTRACT(config, '$.database.pool.max_connections') AS max_conn
FROM application_configs
WHERE JSON_EXTRACT(config, '$.environment') = '"production"';
```

### 2. Working with JSON Arrays
```sql
-- Extract array elements and array properties
SELECT 
    product_id,
    JSON_EXTRACT(details, '$.variants[0].sku') AS first_sku,
    JSON_EXTRACT(details, '$.variants[1].price') AS second_price,
    JSON_EXTRACT(details, '$.variants[*].color') AS all_colors,
    JSON_LENGTH(JSON_EXTRACT(details, '$.variants')) AS variant_count
FROM products
WHERE JSON_EXTRACT(details, '$.category') = '"electronics"';
```

### 3. Filtering by JSON Content
```sql
-- Find users with specific settings
SELECT 
    user_id,
    username,
    JSON_EXTRACT(preferences, '$.notifications.email') AS email_notif,
    JSON_EXTRACT(preferences, '$.notifications.sms') AS sms_notif
FROM users
WHERE JSON_EXTRACT(preferences, '$.notifications.email') = 'true'
   OR JSON_EXTRACT(preferences, '$.notifications.push') = 'true';
```

### 4. Building Reports from JSON Data
```sql
-- Extract metrics from JSON logs
SELECT 
    DATE(timestamp) AS date,
    JSON_EXTRACT(event_data, '$.action') AS action,
    COUNT(*) AS event_count,
    AVG(CAST(JSON_UNQUOTE(JSON_EXTRACT(event_data, '$.duration')) AS DECIMAL)) AS avg_duration,
    MAX(CAST(JSON_UNQUOTE(JSON_EXTRACT(event_data, '$.value')) AS DECIMAL)) AS max_value
FROM event_logs
WHERE JSON_EXTRACT(event_data, '$.category') = '"user_interaction"'
GROUP BY DATE(timestamp), JSON_EXTRACT(event_data, '$.action');
```

### 5. Handling Different JSON Structures
```sql
-- Extract email from different JSON structures
SELECT 
    user_id,
    COALESCE(
        JSON_UNQUOTE(JSON_EXTRACT(profile, '$.email')),
        JSON_UNQUOTE(JSON_EXTRACT(profile, '$.contact.email')),
        JSON_UNQUOTE(JSON_EXTRACT(profile, '$.emails[0]'))
    ) AS user_email
FROM users_mixed_schema;
```

### 6. Complex Path Expressions
```sql
-- Using wildcards and filters in paths
SELECT 
    order_id,
    -- Extract all item names
    JSON_EXTRACT(order_data, '$.items[*].name') AS all_items,
    -- Extract specific item by index
    JSON_EXTRACT(order_data, '$.items[2].price') AS third_item_price,
    -- Extract with conditions (if supported)
    JSON_EXTRACT(order_data, '$.items[?(@.quantity > 1)].name') AS bulk_items  -- Filter syntax
FROM orders;
```

## JSON Path Syntax

| Path Expression | Description |
|----------------|-------------|
| `$` | Root element |
| `$.key` | Direct child with key "key" |
| `$.key1.key2` | Nested child |
| `$[0]` | First array element |
| `$[*]` | All array elements |
| `$.key[*].subkey` | Subkey from all array elements |
| `$..key` | Recursive search for "key" |

## Type Conversion

```sql
-- Converting extracted JSON values to SQL types
SELECT 
    user_id,
    -- Remove quotes from string values
    JSON_UNQUOTE(JSON_EXTRACT(data, '$.name')) AS name_string,
    -- Cast to integer
    CAST(JSON_EXTRACT(data, '$.age') AS INTEGER) AS age_int,
    -- Cast to decimal
    CAST(JSON_EXTRACT(data, '$.balance') AS DECIMAL(10,2)) AS balance_decimal,
    -- Cast to boolean
    CASE JSON_EXTRACT(data, '$.active') 
        WHEN 'true' THEN TRUE 
        WHEN 'false' THEN FALSE 
        ELSE NULL 
    END AS is_active
FROM user_data;
```

## NULL Handling

```sql
-- Handling NULL values in JSON extraction
WITH test_data AS (
    SELECT 1 AS id, '{"a": 1, "b": null}' AS json_col
    UNION ALL SELECT 2, '{"a": 2}'
    UNION ALL SELECT 3, NULL
    UNION ALL SELECT 4, '{"c": 3}'
)
SELECT 
    id,
    json_col,
    JSON_EXTRACT(json_col, '$.a') AS extract_a,
    JSON_EXTRACT(json_col, '$.b') AS extract_b,
    JSON_EXTRACT(json_col, '$.missing') AS extract_missing,
    COALESCE(JSON_EXTRACT(json_col, '$.b'), '{"default": true}') AS with_default
FROM test_data;
```

## Performance Considerations
- Create functional indexes on frequently extracted paths
- Consider storing frequently accessed values in regular columns
- Use generated columns for commonly extracted values
- Validate JSON structure before insertion to ensure consistent paths

## Related Functions
- [JSON_VALUE](./json_value.md) - Extracts scalar value from JSON
- [JSON_QUERY](./json_query.md) - Extracts JSON object or array
- [JSON_SET](./json_set.md) - Modifies JSON document
- [JSON_CONTAINS](./json_contains.md) - Checks if JSON contains value
- [JSON_KEYS](./json_keys.md) - Returns JSON object keys