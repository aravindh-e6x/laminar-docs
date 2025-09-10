---
sidebar_position: 1
---

# SQL Reference Introduction

Laminar uses SQL as its primary language for defining data pipelines and transformations. This section provides a comprehensive reference for Laminar's SQL dialect.

## Overview

Laminar SQL extends standard SQL with streaming-specific constructs:

- **Continuous queries** that run indefinitely on streaming data
- **Window functions** for time-based and count-based aggregations
- **Watermarks** for handling late-arriving data
- **Joins** between streams and tables

## SQL Compatibility

Laminar SQL is based on Apache Calcite and supports:

- ANSI SQL standard syntax
- Common SQL functions and operators
- Complex data types (arrays, structs, maps)
- User-defined functions (UDFs)

## Key Concepts

### Streams vs Tables

In Laminar, data can be represented as:

- **Streams**: Unbounded sequences of events
- **Tables**: Bounded datasets or materialized views
- **Updating Tables**: Tables that change over time

### Time Semantics

Laminar supports two time semantics:

- **Event Time**: Time when the event occurred
- **Processing Time**: Time when the event is processed

### Windows

Windows divide infinite streams into finite chunks:

- **Tumbling Windows**: Fixed-size, non-overlapping
- **Sliding Windows**: Fixed-size, overlapping
- **Session Windows**: Variable-size, gap-based

## Basic Syntax

### Creating a Pipeline

```sql
CREATE PIPELINE my_pipeline AS
SELECT 
    user_id,
    COUNT(*) as event_count,
    window_start,
    window_end
FROM events
GROUP BY 
    user_id,
    TUMBLE(event_time, INTERVAL '1' HOUR);
```

### Creating a Table

```sql
CREATE TABLE users (
    user_id BIGINT,
    username VARCHAR,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id)
) WITH (
    connector = 'postgres',
    connection = 'my_postgres'
);
```

## Next Steps

- [SELECT Statements](./select) - Query syntax and operators
- [Window Functions](./windows) - Time-based aggregations
- [Joins](./joins) - Joining streams and tables
- [Data Types](./types/primitive) - Supported data types