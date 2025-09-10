---
title: Union - Combining Multiple Streams
---


## Overview
This example shows how to combine data from multiple sources using UNION ALL, merging streams while preserving source identification.

## Key Concepts
- UNION ALL for stream combination
- Source tagging for lineage tracking
- Multiple table sources
- Stream merging patterns

## Test Data
**Download**: [impulse.json](/test-data/impulse.json)

Same data file read as two separate sources to demonstrate union.

## SQL Query

### 1. Source Configuration
```sql
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
First source table reading from the impulse dataset.

```sql
CREATE TABLE second_impulse_source (
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
Second source table reading the same file independently, simulating multiple data streams.

### 2. Sink Configuration
```sql
CREATE TABLE union_output (
  counter bigint,
  source text
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```
Output table with a `source` field to track which stream each record came from.

### 3. Processing Pipeline
```sql
INSERT INTO union_output
SELECT counter, 'first' as source FROM impulse_source
UNION ALL 
SELECT counter, 'second' as source FROM second_impulse_source
```
Combines both streams using UNION ALL, which preserves duplicates. Each stream is tagged with a source identifier ('first' or 'second') for lineage tracking. Records from both sources are interleaved in the output based on processing timing.
- Preserves all records from both sources
- Maintains duplicates (same counter appears twice)
- Faster than UNION (which would deduplicate)

### Source Tagging
Adding source identifier:
- `'first' as source` tags records from first table
- `'second' as source` tags records from second table
- Essential for tracking data lineage
- Helps with debugging and auditing

### Schema Alignment
Both SELECT statements must have:
- Same number of columns
- Compatible data types
- Columns aligned by position

## Expected Output
Each counter value appears twice with different source tags:
```json
{"counter": 0, "source": "first"}
{"counter": 0, "source": "second"}
{"counter": 1, "source": "first"}
{"counter": 1, "source": "second"}
{"counter": 2, "source": "first"}
{"counter": 2, "source": "second"}
```

Stream interleaving:
- Order depends on processing timing
- Records may interleave between sources
- No guaranteed ordering without ORDER BY

Use cases:
- Combining data from multiple regions
- Merging different data formats
- Failover and redundancy patterns
- A/B testing with parallel streams
- Multi-tenant data processing