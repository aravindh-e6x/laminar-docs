---
title: Grouped Aggregates - Group By Statistics
---


## Overview
This example shows how to compute aggregate statistics for different groups of data using GROUP BY, partitioning the stream into buckets and computing statistics for each.

## Key Concepts
- GROUP BY clause for partitioned aggregations
- Modulo operator for creating groups
- Primary key in sink table for group identification
- Updating aggregates per group

## Test Data
**Download**: [impulse.json](/test-data/impulse.json)

The impulse dataset contains incrementing counter values (0, 1, 2, 3, ...) which are grouped using modulo 5.

## SQL Query

### 1. Source Configuration
```sql
--pk=counter_mod
CREATE TABLE impulse_source (
  timestamp TIMESTAMP,
  counter bigint unsigned not null,
  subtask_index bigint unsigned not null
) WITH (
  connector = 'single_file',
  path = '$input_dir/impulse.json',
  format = 'json',
  type = 'source'
);
```
Reads the incrementing counter dataset. The comment `--pk=counter_mod` indicates that the output will have a primary key on the modulo result.

### 2. Sink Configuration
```sql
CREATE TABLE aggregates (
  counter_mod BIGINT PRIMARY KEY,
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
Configures output with a PRIMARY KEY on `counter_mod` for group identification. The Debezium format tracks changes to each group's aggregates over time.

### 3. Processing Pipeline
```sql
INSERT INTO aggregates 
SELECT 
  counter % 5,
  min(counter),
  max(counter),
  sum(counter),
  count(*),
  avg(counter)
FROM impulse_source
GROUP BY 1
```
Partitions data into 5 groups using modulo operator (`counter % 5`), computing separate statistics for each group. Group 0 contains values 0,5,10,15..., Group 1 contains 1,6,11,16..., and so on.

## Query Breakdown

### Grouping Logic
`counter % 5` creates 5 groups (0-4) based on the remainder when dividing by 5:
- Group 0: counters 0, 5, 10, 15, ...
- Group 1: counters 1, 6, 11, 16, ...
- Group 2: counters 2, 7, 12, 17, ...
- Group 3: counters 3, 8, 13, 18, ...
- Group 4: counters 4, 9, 14, 19, ...

### Primary Key
The `counter_mod` field is marked as PRIMARY KEY, which:
- Identifies each group uniquely
- Enables proper update semantics in Debezium format
- Ensures each group's statistics are tracked separately

### Per-Group Statistics
For each of the 5 groups, the query maintains:
- Minimum value in that group
- Maximum value in that group
- Sum of all values in that group
- Count of records in that group
- Average value for that group

## Expected Output
The output will contain 5 rows (one per group), continuously updating as data flows:

After processing counters 0-9:
```json
{"counter_mod": 0, "min": 0, "max": 5, "sum": 5, "count": 2, "avg": 2.5}
{"counter_mod": 1, "min": 1, "max": 6, "sum": 7, "count": 2, "avg": 3.5}
{"counter_mod": 2, "min": 2, "max": 7, "sum": 9, "count": 2, "avg": 4.5}
{"counter_mod": 3, "min": 3, "max": 8, "sum": 11, "count": 2, "avg": 5.5}
{"counter_mod": 4, "min": 4, "max": 9, "sum": 13, "count": 2, "avg": 6.5}
```

Each group's statistics update independently as new matching values arrive.