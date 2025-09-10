---
title: Introduction to Examples
---


This section contains a comprehensive collection of SQL streaming examples that demonstrate various capabilities of the Laminar streaming engine. These examples are derived from our test suite and showcase real patterns you can use in your streaming pipelines.

## About the Examples

All examples in this section use the `single_file` connector for both sources and sinks. This approach provides:

- **Reproducibility**: Every example can be run with the same test data
- **Simplicity**: No external dependencies or setup required
- **Learning Focus**: Concentrate on SQL patterns without infrastructure complexity

## Test Data Files

The examples use the following test data files, which you can download for local testing:

### impulse.json
A time-series dataset with incrementing counters, useful for demonstrating window functions and aggregations.
- Fields: `timestamp`, `counter`, `subtask_index`
- Pattern: Regular time intervals with incrementing counter values

### cars.json
Ride-sharing event data with pickup/dropoff events, perfect for location-based analytics and driver activity tracking.
- Fields: `timestamp`, `driver_id`, `event_type`, `location`
- Events: pickup and dropoff events from various drivers and locations

### aggregate_updates.json
E-commerce order data in Debezium CDC format, demonstrating change data capture patterns.
- Fields: `id`, `customer_name`, `product_name`, `quantity`, `price`, `order_date`, `status`
- Format: Debezium envelope with before/after states

### nexmark_bids.json
Auction bid data from the Nexmark benchmark suite.
- Fields: `timestamp`, `auction_id`, `bidder_id`, `price`
- Use case: Online auction analytics

### session_window.json
Session-based user activity data for demonstrating session windowing.
- Fields: Similar to impulse.json but designed for session analysis

### sorted_cars.json
Pre-sorted version of cars.json for order-dependent operations.
- Fields: Same as cars.json but with guaranteed timestamp ordering

## SQL Query Structure

Each example follows this pattern:

```sql
-- Create source table
CREATE TABLE source_table (
    -- field definitions
) WITH (
    connector = 'single_file',
    path = '$input_dir/data.json',
    format = 'json',
    type = 'source'
);

-- Create sink table
CREATE TABLE sink_table (
    -- field definitions
) WITH (
    connector = 'single_file',
    path = '$output_path',
    format = 'json',
    type = 'sink'
);

-- Processing query
INSERT INTO sink_table
SELECT -- transformations
FROM source_table;
```

## Key Concepts Covered

The examples demonstrate:

- **Window Functions**: TUMBLE, HOP, SESSION windows for time-based aggregations
- **Aggregations**: COUNT, SUM, AVG, MIN, MAX, and custom aggregations
- **Joins**: Inner, outer, left, right joins with various conditions
- **CDC Processing**: Debezium format handling and changelog processing
- **UDFs**: User-defined functions in Rust and Python
- **Advanced Patterns**: Watermarks, late event handling, updating aggregates

## Getting Started

1. Download the test data files from the `test-data` directory
2. Choose an example that matches your use case
3. Copy the SQL and modify the file paths to point to your test data
4. Run the query to see the results

Each example page includes:
- Overview of what the query demonstrates
- Key concepts being illustrated
- Complete SQL code
- Detailed breakdown of the query logic
- Expected output format