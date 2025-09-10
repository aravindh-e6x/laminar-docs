---
title: Aggregates - Statistical Functions
---

## Overview
This example demonstrates basic aggregate functions (MIN, MAX, SUM, COUNT, AVG) applied to streaming data, showing how to compute statistics over an entire stream.

## Key Concepts
- MIN, MAX, SUM, COUNT, AVG aggregate functions
- Global aggregations (no GROUP BY)
- Debezium format for sink to handle updating results

## Test Data
**Download**: [impulse.json](/test-data/impulse.json)

The impulse dataset contains time-series data with:
- `timestamp`: Event timestamp
- `counter`: Incrementing counter value (0, 1, 2, ...)
- `subtask_index`: Partition identifier

## SQL Query

### 1. Source Configuration
```sql
CREATE TABLE impulse_source (
    timestamp TIMESTAMP,
    counter BIGINT UNSIGNED NOT NULL,
    subtask_index BIGINT UNSIGNED NOT NULL
) WITH (
    connector = 'single_file',
    path = '$input_dir/impulse.json',
    format = 'json',
    type = 'source'
);
```
 This sets up a source that reads time-series test data with incrementing counters. The UNSIGNED constraint ensures only non-negative values are accepted. Each record represents a discrete time point with a counter value.

### 2. Sink Configuration
```sql
CREATE TABLE aggregates (
    min BIGINT,
    max BIGINT,
    sum BIGINT,
    count BIGINT,
    avg DOUBLE
) WITH (
    connector = 'single_file',
    path = '$output_path',
    format = 'debezium_json',
    type = 'sink'
);
```
 The sink is configured with `debezium_json` format, which is crucial for aggregates. This format captures both the previous and new values for each update, allowing downstream systems to track how statistics evolve over time. The schema contains five columns for different statistical measures.

### 3. Processing Pipeline
```sql
INSERT INTO aggregates
SELECT
    MIN(counter),
    MAX(counter),
    SUM(counter),
    COUNT(*),
    AVG(counter)
FROM impulse_source;
```
 This pipeline continuously computes five statistical aggregates over the entire stream:
- `MIN(counter)`: Tracks the smallest value ever seen
- `MAX(counter)`: Tracks the largest value ever seen  
- `SUM(counter)`: Accumulates the total of all values
- `COUNT(*)`: Counts total records processed
- `AVG(counter)`: Calculates the running average

As each new record arrives, all five statistics are recalculated and emitted as an update.

## Expected Output
As the pipeline processes the incrementing counter values, the output will continuously update:
- MIN will remain 0 (first value)
- MAX will increase with each new record
- SUM will grow quadratically (sum of 0+1+2+...+n)
- COUNT will increment by 1 for each record
- AVG will approach n/2 for n records

Example progression:
- After 1 record: min=0, max=0, sum=0, count=1, avg=0
- After 5 records: min=0, max=4, sum=10, count=5, avg=2
- After 10 records: min=0, max=9, sum=45, count=10, avg=4.5