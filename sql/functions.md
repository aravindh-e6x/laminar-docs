---
sidebar_position: 2
---


Laminar provides a comprehensive set of SQL functions for data manipulation and analysis. This section documents all available functions organized by category.

## Function Categories

### Scalar Functions
Functions that operate on individual values and return a single result per row.

- **[Math Functions](./functions/math)** - Arithmetic and mathematical operations
- **[String Functions](./functions/string)** - Text manipulation and pattern matching
- **[Date/Time Functions](./functions/datetime)** - Temporal data operations
- **[Conditional Functions](./functions/conditional)** - Logical operations and conditionals
- **[Array Functions](./functions/array)** - Array manipulation
- **[Struct Functions](./functions/struct)** - Struct/record operations
- **[JSON Functions](./functions/json)** - JSON parsing and manipulation
- **[Binary Functions](./functions/binary)** - Binary and bitwise operations
- **[Hash Functions](./functions/hash)** - Cryptographic and hash functions
- **[Regular Expression Functions](./functions/regex)** - Pattern matching with regex

### Aggregate Functions
Functions that operate on sets of values and return a single result.

- **[Statistical Aggregates](./functions/aggregate)** - SUM, AVG, COUNT, etc.
- **[Approximate Aggregates](./functions/aggregate#approximate)** - HyperLogLog, percentiles
- **[Statistical Functions](./functions/aggregate#statistical)** - Variance, standard deviation

### Window Functions
Functions that perform calculations across a set of rows while preserving individual row results.

- **[Ranking Functions](./functions/window#ranking)** - ROW_NUMBER, RANK, DENSE_RANK
- **[Analytic Functions](./functions/window#analytic)** - LAG, LEAD, FIRST_VALUE, LAST_VALUE
- **[Aggregate Window Functions](./functions/window#aggregate)** - Running totals, moving averages

## Function Syntax

### Basic Function Call
```sql
SELECT function_name(argument1, argument2, ...)
FROM table_name;
```

### Function with Named Parameters
```sql
SELECT substring(column_name FROM 1 FOR 5)
FROM table_name;
```

### Nested Functions
```sql
SELECT upper(trim(column_name))
FROM table_name;
```

## Type Casting

DataFusion supports explicit type casting using either SQL standard syntax or PostgreSQL-style casting:

```sql
-- SQL standard
CAST(expression AS datatype)

-- PostgreSQL style
expression::datatype
```

## NULL Handling

Most functions return NULL when any argument is NULL, unless explicitly documented otherwise. Use COALESCE or IS NULL checks to handle NULL values:

```sql
-- Replace NULL with default value
SELECT COALESCE(column_name, 'default_value')

-- Check for NULL
SELECT CASE WHEN column_name IS NULL THEN 'N/A' ELSE column_name END
```

## Performance Considerations

1. **Function Pushdown**: Laminar optimizes function execution by pushing filters down to storage when possible
2. **Vectorized Execution**: Functions operate on columnar data for improved performance
3. **Lazy Evaluation**: Functions are only evaluated when results are needed
4. **Type Inference**: Laminar automatically infers types, reducing casting overhead

## Compatibility

Laminar's SQL functions are largely compatible with PostgreSQL, with some additions from other databases:
- Core functions follow PostgreSQL semantics
- Additional functions from Apache Spark SQL
- Custom functions specific to streaming operations