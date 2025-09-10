---
sidebar_position: 7
---

# JSON Functions

Functions for parsing, extracting, and manipulating JSON data.

## JSON Extraction

### json_get(json, path) / get_json_object(json, path)
Extracts value from JSON using JSONPath.
```sql
SELECT json_get('{"name": "John", "age": 30}', '$.name'); 
-- Returns: "John"

SELECT json_get('{"user": {"name": "John"}}', '$.user.name'); 
-- Returns: "John"

SELECT json_get('[1, 2, 3]', '$[1]'); 
-- Returns: 2
```

### json_get_str(json, path) / json_extract_string(json, path)
Extracts value as string.
```sql
SELECT json_get_str('{"name": "John"}', '$.name'); 
-- Returns: 'John' (without quotes)
```

### json_get_int(json, path) / json_extract_int(json, path)
Extracts value as integer.
```sql
SELECT json_get_int('{"age": 30}', '$.age'); 
-- Returns: 30
```

### json_get_float(json, path) / json_extract_float(json, path)
Extracts value as float.
```sql
SELECT json_get_float('{"price": 19.99}', '$.price'); 
-- Returns: 19.99
```

### json_get_bool(json, path) / json_extract_bool(json, path)
Extracts value as boolean.
```sql
SELECT json_get_bool('{"active": true}', '$.active'); 
-- Returns: true
```

## JSON Path Operations

### json_contains(json, path [, value])
Checks if JSON contains path or value at path.
```sql
SELECT json_contains('{"a": 1, "b": 2}', '$.a'); 
-- Returns: true

SELECT json_contains('{"items": [1, 2, 3]}', '$.items', 2); 
-- Returns: true
```

### json_keys(json [, path])
Returns array of keys.
```sql
SELECT json_keys('{"a": 1, "b": 2, "c": 3}'); 
-- Returns: ['a', 'b', 'c']

SELECT json_keys('{"user": {"name": "John", "age": 30}}', '$.user'); 
-- Returns: ['name', 'age']
```

### json_length(json [, path])
Returns length of JSON array or object.
```sql
SELECT json_length('[1, 2, 3]'); 
-- Returns: 3

SELECT json_length('{"a": 1, "b": 2}'); 
-- Returns: 2

SELECT json_length('{"items": [1, 2, 3]}', '$.items'); 
-- Returns: 3
```

### json_paths(json)
Returns all paths in JSON.
```sql
SELECT json_paths('{"a": {"b": 1}, "c": 2}'); 
-- Returns: ['$.a.b', '$.c']
```

## JSON Array Functions

### json_array_get(json_array, index)
Gets element from JSON array.
```sql
SELECT json_array_get('[1, 2, 3]', 1); 
-- Returns: 2
```

### json_array_contains(json_array, value)
Checks if JSON array contains value.
```sql
SELECT json_array_contains('[1, 2, 3]', 2); 
-- Returns: true
```

### json_array_length(json_array)
Returns length of JSON array.
```sql
SELECT json_array_length('[1, 2, 3]'); 
-- Returns: 3
```

## JSON Construction

### to_json(value)
Converts value to JSON.
```sql
SELECT to_json(123); 
-- Returns: '123'

SELECT to_json('hello'); 
-- Returns: '"hello"'

SELECT to_json(ARRAY[1, 2, 3]); 
-- Returns: '[1,2,3]'

SELECT to_json(STRUCT(1 as a, 'b' as b)); 
-- Returns: '{"a":1,"b":"b"}'
```

### json_object(key1, value1, key2, value2, ...)
Creates JSON object from key-value pairs.
```sql
SELECT json_object('name', 'John', 'age', 30); 
-- Returns: '{"name":"John","age":30}'
```

### json_array(value1, value2, ...)
Creates JSON array from values.
```sql
SELECT json_array(1, 2, 3); 
-- Returns: '[1,2,3]'

SELECT json_array('a', 'b', 'c'); 
-- Returns: '["a","b","c"]'
```

### json_build_object(key1, value1, key2, value2, ...)
Builds JSON object.
```sql
SELECT json_build_object('name', 'John', 'age', 30); 
-- Returns: '{"name":"John","age":30}'
```

### json_build_array(value1, value2, ...)
Builds JSON array.
```sql
SELECT json_build_array(1, 2, 3); 
-- Returns: '[1,2,3]'
```

## JSON Modification

### json_set(json, path, value)
Sets value at path in JSON.
```sql
SELECT json_set('{"a": 1}', '$.b', 2); 
-- Returns: '{"a":1,"b":2}'

SELECT json_set('{"a": {"b": 1}}', '$.a.c', 3); 
-- Returns: '{"a":{"b":1,"c":3}}'
```

### json_insert(json, path, value)
Inserts value at path if it doesn't exist.
```sql
SELECT json_insert('{"a": 1}', '$.b', 2); 
-- Returns: '{"a":1,"b":2}'

SELECT json_insert('{"a": 1}', '$.a', 2); 
-- Returns: '{"a":1}' (no change, path exists)
```

### json_replace(json, path, value)
Replaces value at path if it exists.
```sql
SELECT json_replace('{"a": 1}', '$.a', 2); 
-- Returns: '{"a":2}'

SELECT json_replace('{"a": 1}', '$.b', 2); 
-- Returns: '{"a":1}' (no change, path doesn't exist)
```

### json_remove(json, path)
Removes value at path.
```sql
SELECT json_remove('{"a": 1, "b": 2}', '$.b'); 
-- Returns: '{"a":1}'

SELECT json_remove('[1, 2, 3]', '$[1]'); 
-- Returns: '[1,3]'
```

### json_merge(json1, json2)
Merges two JSON objects.
```sql
SELECT json_merge('{"a": 1}', '{"b": 2}'); 
-- Returns: '{"a":1,"b":2}'

SELECT json_merge('{"a": 1}', '{"a": 2, "b": 3}'); 
-- Returns: '{"a":2,"b":3}'
```

### json_merge_patch(target, patch)
Applies JSON merge patch (RFC 7396).
```sql
SELECT json_merge_patch('{"a": 1, "b": 2}', '{"b": 3, "c": 4}'); 
-- Returns: '{"a":1,"b":3,"c":4}'

SELECT json_merge_patch('{"a": {"b": 1}}', '{"a": {"c": 2}}'); 
-- Returns: '{"a":{"b":1,"c":2}}'
```

## JSON Validation

### is_json(string)
Checks if string is valid JSON.
```sql
SELECT is_json('{"a": 1}'); 
-- Returns: true

SELECT is_json('{invalid}'); 
-- Returns: false
```

### json_valid(string)
Alias for is_json.
```sql
SELECT json_valid('[]'); 
-- Returns: true
```

### json_type(json [, path])
Returns JSON type at path.
```sql
SELECT json_type('{"a": 1}'); 
-- Returns: 'object'

SELECT json_type('[1, 2, 3]'); 
-- Returns: 'array'

SELECT json_type('123'); 
-- Returns: 'number'

SELECT json_type('"hello"'); 
-- Returns: 'string'

SELECT json_type('true'); 
-- Returns: 'boolean'

SELECT json_type('null'); 
-- Returns: 'null'
```

## JSON Formatting

### json_pretty(json)
Formats JSON with indentation.
```sql
SELECT json_pretty('{"a":1,"b":{"c":2}}');
-- Returns:
-- {
--   "a": 1,
--   "b": {
--     "c": 2
--   }
-- }
```

### json_strip_nulls(json)
Removes null values from JSON.
```sql
SELECT json_strip_nulls('{"a": 1, "b": null, "c": {"d": null, "e": 2}}');
-- Returns: '{"a":1,"c":{"e":2}}'
```

## JSON Conversion

### json_to_array(json)
Converts JSON array to SQL array.
```sql
SELECT json_to_array('[1, 2, 3]'); 
-- Returns: [1, 2, 3] (SQL array)
```

### json_to_struct(json)
Converts JSON object to SQL struct.
```sql
SELECT json_to_struct('{"a": 1, "b": "hello"}'); 
-- Returns: {a: 1, b: 'hello'} (SQL struct)
```

## JSONPath Expressions

Laminar supports JSONPath expressions:

- `$` - Root object
- `.key` - Object member
- `[n]` - Array element
- `[*]` - All array elements
- `..` - Recursive descent
- `?()` - Filter expression

Examples:
```sql
-- Extract nested value
SELECT json_get('{"user": {"name": "John"}}', '$.user.name');

-- Extract from array
SELECT json_get('[{"id": 1}, {"id": 2}]', '$[0].id');

-- Extract all array elements
SELECT json_get('[{"name": "A"}, {"name": "B"}]', '$[*].name');

-- Recursive search
SELECT json_get('{"a": {"b": 1}, "c": {"b": 2}}', '$..b');
```

## Usage Examples

### Parse API response
```sql
SELECT 
    json_get_str(response, '$.data.user.name') as user_name,
    json_get_int(response, '$.data.user.age') as age,
    json_get_bool(response, '$.data.user.active') as is_active
FROM api_responses;
```

### Extract nested JSON arrays
```sql
SELECT 
    id,
    json_get(data, '$.items[*].price') as prices,
    json_array_length(json_get(data, '$.items')) as item_count
FROM orders;
```

### Build JSON from columns
```sql
SELECT 
    json_object(
        'id', user_id,
        'name', user_name,
        'metadata', json_object(
            'created', created_at,
            'updated', updated_at
        )
    ) as user_json
FROM users;
```

### Filter by JSON content
```sql
SELECT * FROM events
WHERE json_get_str(payload, '$.event_type') = 'purchase'
  AND json_get_float(payload, '$.amount') > 100;
```