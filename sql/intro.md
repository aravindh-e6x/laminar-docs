---
sidebar_position: 1
title: SQL Overview
---

# SQL Reference

Laminar uses SQL as its primary language for defining data pipelines and transformations. Under the hood, Laminar uses [Apache DataFusion](https://arrow.apache.org/datafusion/) as its SQL engine, extending it with streaming-specific semantics.

## DataFusion Foundation

Laminar is built on Apache DataFusion, which provides:

- **ANSI SQL compliance** with extensive SQL function support
- **Columnar execution** using Apache Arrow format
- **Query optimization** with cost-based optimizer
- **Extensibility** through user-defined functions

For detailed information about SQL functions, operators, and syntax, refer to the [DataFusion SQL Reference](https://arrow.apache.org/datafusion/user-guide/sql/index.html).

## Streaming Extensions

While DataFusion handles the core SQL processing, Laminar wraps it with streaming-specific semantics:

### Streaming Concepts

- **Continuous queries** that run indefinitely on unbounded data streams
- **Window functions** for time-based and count-based aggregations
- **Watermarks** for handling late-arriving data and controlling result emission
- **Stream-table joins** with temporal alignment

For detailed information about these streaming features, see:

- [Streaming Overview](./streaming/overview) - Core streaming concepts
- [Tumble Windows](./streaming/tumble) - Fixed-size, non-overlapping windows
- [Hop Windows](./streaming/hop) - Fixed-size, overlapping windows  
- [Session Windows](./streaming/session) - Variable-size, activity-based windows
- [Watermarks](./streaming/watermarks) - Late data handling

## Next Steps

- Explore [Streaming SQL](./streaming/overview) for streaming-specific features
- Learn about [Window Functions](./streaming/tumble) for time-based aggregations
- Check [DataFusion Documentation](https://arrow.apache.org/datafusion/) for function reference
- See [Tutorials](/docs/tutorials/introduction) for practical examples