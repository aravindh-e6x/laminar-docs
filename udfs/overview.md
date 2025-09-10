---
title: UDF Overview
sidebar_position: 1
---

# User-Defined Functions (UDFs)

User-Defined Functions (UDFs) allow you to extend Laminar's SQL capabilities with custom Rust functions. UDFs are compiled to native code and run efficiently within your streaming pipelines.

## Why Use UDFs?

UDFs enable you to:
- Implement custom business logic not available in standard SQL
- Parse complex data formats
- Call external services asynchronously
- Create reusable functions across pipelines
- Optimize performance-critical operations

## Types of UDFs

### Scalar UDFs
Process individual values and return a single result:
```rust
#[udf]
fn parse_user_agent(ua: String) -> String {
    // Parse and extract browser name
}
```

### Async UDFs
Make external API calls with configurable timeouts:
```rust
#[udf]
async fn enrich_with_api(id: i64) -> Option<String> {
    // Call external service
}
```

### Aggregate UDFs
Create custom aggregation functions:
```rust
#[udf]
fn custom_percentile(values: Vec<f64>) -> f64 {
    // Calculate custom percentile
}
```

## Quick Example

Here's a simple UDF that extracts a domain from an email:

```rust
#[udf]
fn extract_domain(email: String) -> Option<String> {
    email.split('@').nth(1).map(|s| s.to_string())
}
```

Use it in SQL:
```sql
SELECT 
    email,
    extract_domain(email) as domain
FROM users
```

## Key Features

- **Native Performance**: UDFs compile to optimized machine code
- **Type Safety**: Strong typing with Rust's type system
- **Dependency Support**: Include external crates via TOML configuration
- **Hot Reloading**: Update UDFs without restarting pipelines
- **Error Handling**: Graceful error handling with Option/Result types
- **Testing Support**: Test UDFs locally before deployment

## Getting Started

1. [Create your first UDF](./creating-udfs)
2. [Explore UDF examples](./examples)
3. [Learn best practices](./best-practices)

## Language Support

Currently, Laminar supports UDFs written in:
- **Rust** - Full support with all features
- **Python** (experimental) - Limited support for simple functions

## Next Steps

- [Creating UDFs](./creating-udfs) - Step-by-step guide
- [UDF Types](./udf-types) - Detailed type documentation
- [Examples](./examples) - Real-world UDF examples