---
title: Updating Inner Join - Stream-to-Stream Joins
---


## Overview
This example shows how to join a stream with a filtered view of itself, demonstrating updating join semantics where results change as new matching records arrive.

## Key Concepts
- CREATE VIEW for reusable subqueries
- Stream-to-stream joins
- Join on computed conditions
- Updating join results with Debezium format

## Test Data
This example uses: `impulse.json`

**Download**: [impulse.json](/test-data/impulse.json)

Sequential counter data joined with its filtered subset.

## SQL Query

### Source Configuration
Stream source with sequential counter data for self-join operations:
```sql
--pk=left_count
CREATE TABLE impulse (
  timestamp TIMESTAMP,
  counter bigint unsigned not null,
  subtask_index bigint unsigned not null
) WITH (
  connector = 'single_file',
  path = '$input_dir/impulse.json',
  format = 'json',
  type = 'source',
  event_time_field = 'timestamp'
);

CREATE VIEW impulse_odd AS (
  SELECT * FROM impulse
  WHERE counter % 2 == 1
);
```

### Sink Configuration
Output table with Debezium format for tracking join state changes:
```sql
CREATE TABLE output (
  left_count bigint,
  right_count bigint
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'debezium_json',
  type = 'sink'
);
```

### Processing Pipeline
Stream-to-stream join between source and filtered view:
```sql
INSERT INTO output
SELECT A.counter, B.counter
FROM impulse A
JOIN impulse_odd B ON A.counter = B.counter;
```

## Query Breakdown

### View Definition
`CREATE VIEW impulse_odd`:
- Filters for odd counter values (1, 3, 5, ...)
- Creates reusable filtered stream
- Simplifies join query syntax

### Join Logic
`JOIN ... ON A.counter = B.counter`:
- Joins all records with odd records
- Only odd values from stream A match
- Even values from A have no match in B

### Updating Semantics
As new records arrive:
1. Each odd counter from impulse stream
2. Matches with same counter in impulse_odd view
3. Produces output record
4. Even counters produce no output (no match)

### Debezium Output
The sink captures join evolution:
- New matches appear as inserts
- Join state updates tracked
- Useful for maintaining derived tables

## Expected Output
Only odd counters produce output:
```json
{"before": null, "after": {"left_count": 1, "right_count": 1}}
{"before": null, "after": {"left_count": 3, "right_count": 3}}
{"before": null, "after": {"left_count": 5, "right_count": 5}}
{"before": null, "after": {"left_count": 7, "right_count": 7}}
```

Key patterns:
- Self-joins for finding patterns within stream
- Filtering before joining for efficiency
- Only matching records produce output
- Useful for correlation analysis and pattern matching