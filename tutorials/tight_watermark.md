---
title: Tight Watermark - Zero Late Event Tolerance
---


## Overview
This example demonstrates a tight watermark configuration that doesn't allow late events, ensuring windows close immediately when their time expires.

## Key Concepts
- Watermark without delay
- Immediate window closure
- CREATE VIEW for query composition
- Window end time extraction

## Test Data
This example uses: `cars.json`

**Download**: [cars.json](/test-data/cars.json)

Time-ordered ride-sharing events.

## SQL Query

### Source Configuration
Stream source with tight watermark configuration (zero tolerance for late events):
```sql
CREATE TABLE cars(
  timestamp TIMESTAMP,
  driver_id BIGINT,
  event_type TEXT,
  location TEXT,
  WATERMARK for timestamp 
) WITH (
  connector = 'single_file',
  path = '$input_dir/cars.json',
  format = 'json',
  type = 'source'
);
```

### Sink Configuration
Output table for aggregated count results with timestamps:
```sql
CREATE TABLE group_by_aggregate (
  timestamp TIMESTAMP,
  count BIGINT
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```

### Processing Pipeline
View-based aggregation using tumbling windows with immediate closure:
```sql
CREATE VIEW group_by_view AS (
  SELECT window.end as timestamp, count
  FROM (
    SELECT 
      TUMBLE(INTERVAL '1' hour) as window,
      COUNT(*) as count
    FROM cars
    GROUP BY 1
  )
);

INSERT INTO group_by_aggregate
SELECT timestamp, count FROM group_by_view
```

## Query Breakdown

### Tight Watermark
`WATERMARK for timestamp`:
- No delay specified (implicit: timestamp - 0)
- Windows close exactly at boundary
- No tolerance for late events
- Maximum timeliness, potential data loss

### View Definition
`CREATE VIEW group_by_view`:
- Encapsulates window aggregation logic
- Simplifies final INSERT statement
- Reusable query component
- Improves readability

### Window End Time
`window.end as timestamp`:
- Uses window end time as output timestamp
- Marks when aggregation completed
- Useful for downstream processing
- Clear temporal boundaries

### Tumbling Window
`TUMBLE(INTERVAL '1' hour)`:
- Fixed 1-hour windows
- Non-overlapping buckets
- Completes immediately at hour boundary

## Expected Output
Hourly counts emitted exactly at window boundaries:
```json
{"timestamp": "2023-09-18T15:00:00", "count": 53}
{"timestamp": "2023-09-18T16:00:00", "count": 48}
{"timestamp": "2023-09-18T17:00:00", "count": 61}
```

Characteristics:
- Results appear exactly at hour marks
- No late event corrections
- Lowest latency possible
- May miss events arriving after window closes

Trade-offs:
- **Pros**: Minimal latency, predictable timing, simple logic
- **Cons**: Data loss for late events, no correction capability

Use cases:
- Real-time dashboards requiring immediate updates
- Systems where timeliness > completeness
- Ordered data streams with no late arrivals
- Low-latency alerting systems