---
title: Creating UDFs
sidebar_position: 2
---

# Creating User-Defined Functions

This guide walks you through creating and deploying UDFs in Laminar.

## Basic Structure

Every UDF follows this structure:

```rust
#[udf]
fn function_name(param1: Type1, param2: Type2) -> ReturnType {
    // Your implementation
}
```

The `#[udf]` attribute marks the function as a UDF that Laminar will compile and make available in SQL.

## Step-by-Step Guide

### 1. Write Your UDF

Create a new UDF with the required logic:

```rust
#[udf]
fn normalize_phone(phone: String) -> String {
    // Remove all non-digits
    let digits: String = phone.chars()
        .filter(|c| c.is_ascii_digit())
        .collect();
    
    // Format as US phone number
    if digits.len() == 10 {
        format!("+1-{}-{}-{}", 
            &digits[0..3], 
            &digits[3..6], 
            &digits[6..10])
    } else {
        phone
    }
}
```

### 2. Add Dependencies (Optional)

If your UDF needs external crates, specify them in a comment block:

```rust
/*
[dependencies]
regex = "1.9"
chrono = "0.4"
serde_json = "1.0"
*/

#[udf]
fn parse_json_field(json: String, field: String) -> Option<String> {
    let value: serde_json::Value = serde_json::from_str(&json).ok()?;
    value.get(&field)?.as_str().map(|s| s.to_string())
}
```

### 3. Deploy via API

Use the Laminar API to create your UDF:

```bash
curl -X POST https://api.laminar.cloud/v1/udfs \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "normalize_phone",
    "prefix": "utils",
    "language": "rust",
    "definition": "/* UDF code here */",
    "description": "Normalizes phone numbers to US format"
  }'
```

### 4. Use in SQL

Once deployed, use your UDF in SQL queries:

```sql
SELECT 
    customer_id,
    utils.normalize_phone(phone_number) as formatted_phone
FROM customers
```

## Supported Data Types

### Input Types
- **Primitives**: `bool`, `i8`, `i16`, `i32`, `i64`, `f32`, `f64`
- **Strings**: `String`, `&str`
- **Binary**: `Vec<u8>`, `&[u8]`
- **Optional**: `Option<T>` for nullable inputs
- **Arrays**: `Vec<T>` for array inputs
- **Timestamps**: `SystemTime`, `NaiveDateTime`

### Return Types
- All input types
- `Result<T, E>` for error handling
- Tuples for multiple return values

## Error Handling

Use `Option` or `Result` for functions that might fail:

```rust
#[udf]
fn safe_divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 {
        None
    } else {
        Some(a / b)
    }
}

#[udf]
fn parse_int(s: String) -> Result<i64, String> {
    s.parse::<i64>()
        .map_err(|e| format!("Parse error: {}", e))
}
```

## Async UDFs

For external API calls or I/O operations:

```rust
#[udf]
async fn geocode_address(address: String) -> Option<String> {
    // Make HTTP request to geocoding service
    let client = reqwest::Client::new();
    let response = client
        .get("https://api.geocoding.service/geocode")
        .query(&[("address", address)])
        .send()
        .await
        .ok()?;
    
    let json: serde_json::Value = response.json().await.ok()?;
    json.get("coordinates")?.as_str().map(|s| s.to_string())
}
```

## Testing UDFs

Test your UDFs locally before deployment:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_normalize_phone() {
        assert_eq!(
            normalize_phone("(555) 123-4567".to_string()),
            "+1-555-123-4567"
        );
        assert_eq!(
            normalize_phone("5551234567".to_string()),
            "+1-555-123-4567"
        );
    }
}
```

## Validation

Laminar validates UDFs during creation:
- Syntax checking
- Type validation
- Dependency resolution
- Compilation testing

## Updating UDFs

To update an existing UDF:

```bash
curl -X PUT https://api.laminar.cloud/v1/udfs/{udf_id} \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "definition": "/* Updated UDF code */",
    "description": "Updated description"
  }'
```

## Best Practices

1. **Use Descriptive Names**: Choose clear function names that explain the purpose
2. **Add Documentation**: Include comments explaining complex logic
3. **Handle Errors Gracefully**: Use Option/Result types appropriately
4. **Test Thoroughly**: Include unit tests with your UDF
5. **Keep It Simple**: Break complex logic into multiple UDFs
6. **Version Control**: Track UDF definitions in your source control

## Next Steps

- [UDF Types](./udf-types) - Detailed type reference
- [Examples](./examples) - Real-world UDF examples
- [Async UDFs](./async-udfs) - External service integration