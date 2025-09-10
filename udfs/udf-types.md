---
title: UDF Types
sidebar_position: 3
---

# UDF Type Reference

This page provides a comprehensive reference for all UDF types and their usage patterns.

## Scalar UDFs

Scalar UDFs process individual values and return a single result. They're the most common type of UDF.

### Basic Scalar UDF

```rust
#[udf]
fn uppercase(s: String) -> String {
    s.to_uppercase()
}
```

### With Multiple Parameters

```rust
#[udf]
fn calculate_discount(price: f64, discount_percent: f64) -> f64 {
    price * (1.0 - discount_percent / 100.0)
}
```

### With Optional Values

```rust
#[udf]
fn extract_year(date_str: Option<String>) -> Option<i32> {
    date_str?
        .split('-')
        .next()
        .and_then(|year| year.parse().ok())
}
```

## Async UDFs

Async UDFs enable external service calls with built-in timeout and concurrency control.

### Configuration

Async UDFs support these runtime parameters:
- **Timeout**: Maximum execution time (default: 1 second)
- **Concurrency**: Maximum parallel executions (default: 100)
- **Ordered**: Preserve input order (default: false)

### Basic Async UDF

```rust
#[udf]
async fn validate_email(email: String) -> bool {
    // Call email validation service
    let client = reqwest::Client::new();
    let response = client
        .get("https://api.email-validator.net/validate")
        .query(&[("email", email)])
        .timeout(Duration::from_millis(500))
        .send()
        .await;
    
    match response {
        Ok(resp) => resp.status().is_success(),
        Err(_) => false,
    }
}
```

### With Caching

```rust
use std::sync::Arc;
use std::collections::HashMap;
use tokio::sync::RwLock;

static CACHE: Lazy<Arc<RwLock<HashMap<String, String>>>> = 
    Lazy::new(|| Arc::new(RwLock::new(HashMap::new())));

#[udf]
async fn cached_lookup(key: String) -> Option<String> {
    // Check cache first
    {
        let cache = CACHE.read().await;
        if let Some(value) = cache.get(&key) {
            return Some(value.clone());
        }
    }
    
    // Fetch from external service
    let value = fetch_from_service(&key).await?;
    
    // Update cache
    {
        let mut cache = CACHE.write().await;
        cache.insert(key.clone(), value.clone());
    }
    
    Some(value)
}
```

## Aggregate UDFs

Aggregate UDFs process groups of values and return a single result.

### Basic Aggregate

```rust
#[udf]
fn median(mut values: Vec<f64>) -> Option<f64> {
    if values.is_empty() {
        return None;
    }
    
    values.sort_by(|a, b| a.partial_cmp(b).unwrap());
    let mid = values.len() / 2;
    
    if values.len() % 2 == 0 {
        Some((values[mid - 1] + values[mid]) / 2.0)
    } else {
        Some(values[mid])
    }
}
```

### Custom Statistics

```rust
#[udf]
fn percentile(values: Vec<f64>, p: f64) -> Option<f64> {
    if values.is_empty() || p < 0.0 || p > 100.0 {
        return None;
    }
    
    let mut sorted = values.clone();
    sorted.sort_by(|a, b| a.partial_cmp(b).unwrap());
    
    let index = (p / 100.0 * (sorted.len() - 1) as f64).round() as usize;
    Some(sorted[index])
}
```

## Complex Types

### Working with JSON

```rust
/*
[dependencies]
serde_json = "1.0"
*/

#[udf]
fn json_extract_nested(json: String, path: String) -> Option<String> {
    let value: serde_json::Value = serde_json::from_str(&json).ok()?;
    
    let mut current = &value;
    for key in path.split('.') {
        current = current.get(key)?;
    }
    
    match current {
        serde_json::Value::String(s) => Some(s.clone()),
        other => Some(other.to_string()),
    }
}
```

### Working with Arrays

```rust
#[udf]
fn array_intersection(a: Vec<i64>, b: Vec<i64>) -> Vec<i64> {
    let set_a: HashSet<_> = a.into_iter().collect();
    let set_b: HashSet<_> = b.into_iter().collect();
    
    set_a.intersection(&set_b)
        .cloned()
        .collect()
}
```

### Working with Timestamps

```rust
/*
[dependencies]
chrono = "0.4"
*/

use chrono::{DateTime, Utc, Duration};

#[udf]
fn add_business_days(date: String, days: i64) -> Option<String> {
    let dt = DateTime::parse_from_rfc3339(&date).ok()?;
    let mut current = dt.with_timezone(&Utc);
    let mut days_added = 0;
    
    while days_added < days {
        current = current + Duration::days(1);
        // Skip weekends
        if current.weekday().num_days_from_monday() < 5 {
            days_added += 1;
        }
    }
    
    Some(current.to_rfc3339())
}
```

## Type Mappings

### SQL to Rust Type Mappings

| SQL Type | Rust Type | Notes |
|----------|-----------|-------|
| BOOLEAN | `bool` | |
| TINYINT | `i8` | |
| SMALLINT | `i16` | |
| INT | `i32` | |
| BIGINT | `i64` | |
| FLOAT | `f32` | |
| DOUBLE | `f64` | |
| VARCHAR | `String` | |
| CHAR | `String` | |
| BINARY | `Vec<u8>` | |
| TIMESTAMP | `SystemTime` | Or use chrono types |
| DATE | `NaiveDate` | From chrono |
| ARRAY&lt;T&gt; | `Vec<T>` | |
| NULL | `Option<T>` | For nullable columns |

### Return Type Rules

1. **Non-nullable returns**: Use plain types (`String`, `i64`, etc.)
2. **Nullable returns**: Use `Option<T>`
3. **Error handling**: Use `Result<T, E>` where `E: ToString`
4. **Multiple values**: Use tuples `(T1, T2, ...)`

## Performance Considerations

### Memory Management

```rust
#[udf]
fn process_large_array(data: Vec<String>) -> String {
    // Use iterators to avoid unnecessary allocations
    data.iter()
        .filter(|s| !s.is_empty())
        .map(|s| s.to_uppercase())
        .collect::<Vec<_>>()
        .join(",")
}
```

### Avoiding Copies

```rust
#[udf]
fn efficient_string_processing(s: String) -> String {
    // Reuse the input string's allocation when possible
    if s.len() < 100 {
        s.to_uppercase()
    } else {
        // Only process what we need
        s.chars()
            .take(100)
            .collect::<String>()
            .to_uppercase()
    }
}
```

## Error Handling Patterns

### Using Option

```rust
#[udf]
fn safe_parse(s: String) -> Option<i64> {
    s.parse().ok()
}
```

### Using Result

```rust
#[udf]
fn validated_parse(s: String) -> Result<i64, String> {
    s.parse()
        .map_err(|e| format!("Invalid number '{}': {}", s, e))
}
```

### Fallback Values

```rust
#[udf]
fn parse_with_default(s: String, default: i64) -> i64 {
    s.parse().unwrap_or(default)
}
```

## Next Steps

- [Examples](./examples) - Real-world UDF implementations
- [Async UDFs](./async-udfs) - Deep dive into async UDFs
- [Best Practices](./best-practices) - Performance and design tips