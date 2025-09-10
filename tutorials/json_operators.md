---
title: JSON Operators - Working with JSON Data
---


## Overview
This example demonstrates how to work with unstructured JSON data using JSON operators to extract specific fields dynamically.

## Key Concepts
- Unstructured JSON ingestion
- JSON arrow operator (->)
- Dynamic field extraction
- Handling missing fields

## Test Data
This example uses: `cars.json`

**Download**: [cars.json](/test-data/cars.json)

Ride-sharing events ingested as raw JSON objects.

## SQL Query

### Source Configuration
Unstructured JSON source storing entire records as JSON values:
```sql
CREATE TABLE cars (
  value JSON
) WITH (
  connector = 'single_file',
  path = '$input_dir/cars.json',
  format = 'json',
  type = 'source',
  'json.unstructured' = 'true'
);
```

### Sink Configuration
Generic output table with schema inferred from query:
```sql
CREATE TABLE sink WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```

### Processing Pipeline
JSON field extraction using arrow operators with null handling:
```sql
INSERT INTO sink
SELECT 
  'test' as a,
  value->'driver_id' as b,
  value->'event_type' as c,
  value->'not_a_field' as d
FROM cars;
```

## Query Breakdown

### Unstructured JSON Source
`'json.unstructured' = 'true'`:
- Treats entire JSON as single field
- No predefined schema required
- Flexible for varying JSON structures
- Entire record stored in `value` column

### JSON Arrow Operator
`value->'field_name'`:
- Extracts field from JSON object
- Returns JSON value (not string)
- Preserves original data type
- Returns NULL for missing fields

### Field Extraction Examples
- `value->'driver_id'`: Extracts driver ID number
- `value->'event_type'`: Extracts event type string
- `value->'not_a_field'`: Returns NULL (field doesn't exist)

### Static Field
`'test' as a`:
- Adds constant value to output
- Useful for tagging or versioning

## Expected Output
Extracted JSON fields with mixed types:
```json
{"a": "test", "b": 155, "c": "pickup", "d": null}
{"a": "test", "b": 163, "c": "pickup", "d": null}
{"a": "test", "b": 184, "c": "dropoff", "d": null}
```

Key behaviors:
- Original JSON types preserved (numbers stay numbers)
- Missing fields become NULL
- Can extract nested fields with chained operators
- Useful for schema evolution and flexible data processing

Advanced usage:
- `value->'nested'->'field'` for nested extraction
- `value->>'field'` for text extraction (converts to string)
- `value->'array'->0` for array element access