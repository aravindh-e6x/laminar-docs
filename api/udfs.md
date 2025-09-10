---
sidebar_position: 7
title: UDFs
---


User-Defined Functions (UDFs) allow you to extend Laminar's SQL capabilities with custom logic written in Rust. UDFs can be used in SQL queries for custom transformations, aggregations, and computations.

## Overview

The UDF API enables you to:
- Create and manage custom functions
- Write functions in Rust
- Validate function code before deployment
- Use functions in SQL queries
- Share functions across pipelines

## Endpoints

### List UDFs

Get all registered user-defined functions.

```http
GET /api/v1/udfs
```

#### Response

```json
{
  "data": [
    {
      "id": "udf_abc123",
      "name": "parse_user_agent",
      "description": "Parse user agent string into components",
      "definition": "pub fn parse_user_agent(ua: String) -> String {\n    // Parse logic here\n    ua.to_uppercase()\n}",
      "language": "rust",
      "return_type": "STRING",
      "parameters": [
        {
          "name": "ua",
          "type": "STRING",
          "nullable": false
        }
      ],
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "created_by": "user@example.com"
    },
    {
      "id": "udf_xyz789",
      "name": "calculate_distance",
      "description": "Calculate distance between two coordinates",
      "definition": "pub fn calculate_distance(lat1: f64, lon1: f64, lat2: f64, lon2: f64) -> f64 {\n    // Haversine formula\n    let r = 6371.0; // Earth's radius in km\n    let dlat = (lat2 - lat1).to_radians();\n    let dlon = (lon2 - lon1).to_radians();\n    let a = (dlat / 2.0).sin().powi(2) + lat1.to_radians().cos() * lat2.to_radians().cos() * (dlon / 2.0).sin().powi(2);\n    let c = 2.0 * a.sqrt().atan2((1.0 - a).sqrt());\n    r * c\n}",
      "language": "rust",
      "return_type": "DOUBLE",
      "parameters": [
        {
          "name": "lat1",
          "type": "DOUBLE",
          "nullable": false
        },
        {
          "name": "lon1",
          "type": "DOUBLE",
          "nullable": false
        },
        {
          "name": "lat2",
          "type": "DOUBLE",
          "nullable": false
        },
        {
          "name": "lon2",
          "type": "DOUBLE",
          "nullable": false
        }
      ],
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z",
      "created_by": "admin@example.com"
    }
  ]
}
```

### Create UDF

Create a new user-defined function.

```http
POST /api/v1/udfs
```

#### Request Body

```json
{
  "name": "extract_domain",
  "description": "Extract domain from URL",
  "definition": "pub fn extract_domain(url: String) -> Option<String> {\n    let parts: Vec<&str> = url.split(\"//\").collect();\n    if parts.len() > 1 {\n        let domain_part = parts[1].split('/').next()?;\n        let domain = domain_part.split(':').next()?;\n        Some(domain.to_string())\n    } else {\n        None\n    }\n}",
  "language": "rust",
  "return_type": "STRING",
  "parameters": [
    {
      "name": "url",
      "type": "STRING",
      "nullable": false
    }
  ],
  "nullable_return": true
}
```

#### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique function name |
| `description` | string | No | Function description |
| `definition` | string | Yes | Function implementation code |
| `language` | string | Yes | Programming language (currently only "rust") |
| `return_type` | string | Yes | SQL return type |
| `parameters` | array | Yes | Function parameters |
| `nullable_return` | boolean | No | Whether function can return NULL |

#### Response

```json
{
  "data": {
    "id": "udf_new123",
    "name": "extract_domain",
    "description": "Extract domain from URL",
    "definition": "pub fn extract_domain(url: String) -> Option<String> {...}",
    "language": "rust",
    "return_type": "STRING",
    "parameters": [
      {
        "name": "url",
        "type": "STRING",
        "nullable": false
      }
    ],
    "nullable_return": true,
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z",
    "created_by": "user@example.com"
  }
}
```

### Validate UDF

Validate UDF code before creating or updating.

```http
POST /api/v1/udfs/validate
```

#### Request Body

```json
{
  "name": "my_function",
  "definition": "pub fn my_function(x: i32, y: i32) -> i32 {\n    x + y\n}",
  "language": "rust",
  "return_type": "INT",
  "parameters": [
    {
      "name": "x",
      "type": "INT",
      "nullable": false
    },
    {
      "name": "y",
      "type": "INT",
      "nullable": false
    }
  ]
}
```

#### Response

Success:
```json
{
  "data": {
    "valid": true,
    "message": "Function validated successfully",
    "warnings": [],
    "test_results": [
      {
        "input": {"x": 5, "y": 3},
        "output": 8,
        "success": true
      }
    ]
  }
}
```

Error:
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Function validation failed",
    "details": {
      "line": 2,
      "column": 5,
      "error": "expected `;` or `}`, found `+`"
    }
  }
}
```

### Delete UDF

Delete an existing user-defined function.

```http
DELETE /api/v1/udfs/{id}
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | UDF ID |

#### Response

```json
{
  "data": {
    "message": "UDF deleted successfully"
  }
}
```

#### Error Responses

- `404 Not Found` - UDF not found
- `409 Conflict` - UDF is in use by pipelines

## UDF Types and Signatures

### Supported Data Types

| SQL Type | Rust Type | Description |
|----------|-----------|-------------|
| `BOOLEAN` | `bool` | Boolean value |
| `TINYINT` | `i8` | 8-bit integer |
| `SMALLINT` | `i16` | 16-bit integer |
| `INT` | `i32` | 32-bit integer |
| `BIGINT` | `i64` | 64-bit integer |
| `FLOAT` | `f32` | 32-bit float |
| `DOUBLE` | `f64` | 64-bit float |
| `STRING` | `String` | UTF-8 string |
| `BINARY` | `Vec<u8>` | Binary data |
| `DATE` | `chrono::NaiveDate` | Date |
| `TIME` | `chrono::NaiveTime` | Time |
| `TIMESTAMP` | `chrono::DateTime<Utc>` | Timestamp |
| `ARRAY<T>` | `Vec<T>` | Array |
| `MAP<K,V>` | `HashMap<K,V>` | Map |
| `STRUCT` | Custom struct | Structured data |

### Function Signatures

#### Scalar Functions

```rust
// Simple scalar function
pub fn upper_case(s: String) -> String {
    s.to_uppercase()
}

// Nullable return
pub fn safe_divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None
    } else {
        Some(a / b)
    }
}

// Multiple parameters
pub fn concat_with_separator(s1: String, s2: String, sep: String) -> String {
    format!("{}{}{}", s1, sep, s2)
}
```

#### Array Functions

```rust
// Array input
pub fn array_sum(arr: Vec<i32>) -> i32 {
    arr.iter().sum()
}

// Array output
pub fn split_string(s: String, delimiter: String) -> Vec<String> {
    s.split(&delimiter).map(|s| s.to_string()).collect()
}
```

#### Complex Types

```rust
use std::collections::HashMap;

// Map operations
pub fn map_get(map: HashMap<String, String>, key: String) -> Option<String> {
    map.get(&key).cloned()
}

// Struct operations (requires serde)
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
pub struct Point {
    x: f64,
    y: f64,
}

pub fn distance_from_origin(point: Point) -> f64 {
    (point.x.powi(2) + point.y.powi(2)).sqrt()
}
```

## UDF Examples

### String Manipulation

```json
{
  "name": "extract_email_domain",
  "description": "Extract domain from email address",
  "definition": "pub fn extract_email_domain(email: String) -> Option<String> {\n    let parts: Vec<&str> = email.split('@').collect();\n    if parts.len() == 2 {\n        Some(parts[1].to_string())\n    } else {\n        None\n    }\n}",
  "language": "rust",
  "return_type": "STRING",
  "parameters": [
    {
      "name": "email",
      "type": "STRING",
      "nullable": false
    }
  ],
  "nullable_return": true
}
```

### Mathematical Operations

```json
{
  "name": "percentile_rank",
  "description": "Calculate percentile rank",
  "definition": "pub fn percentile_rank(value: f64, values: Vec<f64>) -> f64 {\n    let mut sorted = values.clone();\n    sorted.sort_by(|a, b| a.partial_cmp(b).unwrap());\n    let count_below = sorted.iter().filter(|&&v| v < value).count() as f64;\n    let count_equal = sorted.iter().filter(|&&v| v == value).count() as f64;\n    let n = sorted.len() as f64;\n    ((count_below + 0.5 * count_equal) / n) * 100.0\n}",
  "language": "rust",
  "return_type": "DOUBLE",
  "parameters": [
    {
      "name": "value",
      "type": "DOUBLE",
      "nullable": false
    },
    {
      "name": "values",
      "type": "ARRAY<DOUBLE>",
      "nullable": false
    }
  ]
}
```

### JSON Processing

```json
{
  "name": "json_extract_field",
  "description": "Extract field from JSON string",
  "definition": "use serde_json::Value;\n\npub fn json_extract_field(json_str: String, field: String) -> Option<String> {\n    let v: Value = serde_json::from_str(&json_str).ok()?;\n    v.get(&field).and_then(|val| {\n        match val {\n            Value::String(s) => Some(s.clone()),\n            _ => Some(val.to_string())\n        }\n    })\n}",
  "language": "rust",
  "return_type": "STRING",
  "parameters": [
    {
      "name": "json_str",
      "type": "STRING",
      "nullable": false
    },
    {
      "name": "field",
      "type": "STRING",
      "nullable": false
    }
  ],
  "nullable_return": true
}
```

### Date/Time Operations

```json
{
  "name": "business_days_between",
  "description": "Calculate business days between two dates",
  "definition": "use chrono::{Datelike, Duration, NaiveDate, Weekday};\n\npub fn business_days_between(start: NaiveDate, end: NaiveDate) -> i32 {\n    let mut count = 0;\n    let mut current = start;\n    \n    while current <= end {\n        let weekday = current.weekday();\n        if weekday != Weekday::Sat && weekday != Weekday::Sun {\n            count += 1;\n        }\n        current = current + Duration::days(1);\n    }\n    \n    count\n}",
  "language": "rust",
  "return_type": "INT",
  "parameters": [
    {
      "name": "start",
      "type": "DATE",
      "nullable": false
    },
    {
      "name": "end",
      "type": "DATE",
      "nullable": false
    }
  ]
}
```

## Using UDFs in SQL

Once created, UDFs can be used in SQL queries:

```sql
-- String manipulation
SELECT 
    email,
    extract_email_domain(email) as domain
FROM users;

-- Mathematical operations
SELECT 
    student_id,
    score,
    percentile_rank(score, ARRAY_AGG(score) OVER ()) as percentile
FROM test_scores;

-- JSON processing
SELECT 
    event_id,
    json_extract_field(payload, 'user_id') as user_id,
    json_extract_field(payload, 'action') as action
FROM events;

-- Date operations
SELECT 
    order_date,
    ship_date,
    business_days_between(order_date, ship_date) as processing_days
FROM orders;
```

## Best Practices

1. **Name conventions**: Use descriptive, lowercase names with underscores
2. **Error handling**: Use `Option<T>` for nullable returns
3. **Performance**: Keep functions lightweight and avoid heavy computations
4. **Testing**: Validate functions thoroughly before deployment
5. **Documentation**: Add clear descriptions and examples
6. **Type safety**: Use appropriate Rust types for SQL types
7. **Dependencies**: Minimize external dependencies
8. **Idempotency**: Ensure functions are deterministic when possible

## Available Crates

The following Rust crates are available for use in UDFs:

- `chrono` - Date and time handling
- `serde` - Serialization/deserialization
- `serde_json` - JSON processing
- `regex` - Regular expressions
- `base64` - Base64 encoding/decoding
- `sha2` - SHA hashing
- `uuid` - UUID generation

## Error Handling

### Compilation Errors

```json
{
  "error": {
    "code": "COMPILATION_ERROR",
    "message": "Failed to compile UDF",
    "details": {
      "line": 5,
      "column": 10,
      "error": "cannot find value `x` in this scope"
    }
  }
}
```

### Runtime Errors

```json
{
  "error": {
    "code": "RUNTIME_ERROR",
    "message": "UDF execution failed",
    "details": {
      "function": "safe_divide",
      "error": "Division by zero"
    }
  }
}
```

### Type Mismatch

```json
{
  "error": {
    "code": "TYPE_ERROR",
    "message": "Type mismatch",
    "details": {
      "expected": "INT",
      "actual": "STRING",
      "parameter": "x"
    }
  }
}
```

## Limitations

- Maximum function size: 100KB
- Maximum execution time: 1 second
- Memory limit: 10MB per invocation
- No network calls allowed
- No file system access
- No global state modification