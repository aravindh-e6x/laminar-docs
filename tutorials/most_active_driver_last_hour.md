---
title: Most Active Driver - Top-N with Windows
---


## Overview
This example demonstrates how to find the most active driver in each time window using ROW_NUMBER() window function, a common pattern for leaderboards and top-N queries.

## Key Concepts
- HOP window for sliding time periods
- ROW_NUMBER() for ranking within partitions
- Watermarks for late event handling
- Top-N pattern with window functions

## Test Data
This example uses: `cars.json`

**Download**: [cars.json](/test-data/cars.json)

Ride-sharing events tracking driver pickups and dropoffs.

## SQL Query

### Source Configuration
Stream source with watermark configuration for handling late events:
```sql
CREATE TABLE cars (
  timestamp TIMESTAMP NOT NULL,
  driver_id BIGINT,
  event_type TEXT,
  location TEXT,
  watermark for timestamp AS (timestamp - interval '1 hour')
) WITH (
  connector = 'single_file',
  path = '$input_dir/cars.json',
  format = 'json',
  type = 'source'
);
```

### Sink Configuration
Output table for ranked driver activity results:
```sql
CREATE TABLE most_active_driver (
  start TIMESTAMP,
  end TIMESTAMP,
  driver_id BIGINT,
  count BIGINT,
  row_number BIGINT
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```

### Processing Pipeline
Top-N query using sliding windows and ROW_NUMBER for ranking:
```sql
INSERT INTO most_active_driver
SELECT window.start, window.end, driver_id, count, row_number 
FROM (
  SELECT *, 
    ROW_NUMBER() OVER (
      PARTITION BY window
      ORDER BY count DESC, driver_id desc
    ) as row_number
  FROM (
    SELECT 
      driver_id,
      hop(INTERVAL '1' minute, INTERVAL '1' hour) as window,
      count(*) as count
    FROM cars
    GROUP BY 1,2
  )
) WHERE row_number = 1
```

## Query Breakdown

### Watermark Definition
`watermark for timestamp AS (timestamp - interval '1 hour')`:
- Allows events up to 1 hour late
- Controls when windows can be finalized
- Balances completeness vs latency

### Sliding Window Aggregation
`hop(INTERVAL '1' minute, INTERVAL '1' hour)`:
- 1-hour windows sliding every minute
- Provides fresh updates every minute
- Each driver's activity counted per window

### Ranking Logic
`ROW_NUMBER() OVER (PARTITION BY window ORDER BY count DESC, driver_id desc)`:
- Ranks drivers within each window
- Highest count gets rank 1
- Tie-breaker: higher driver_id wins
- Deterministic ranking ensures consistent results

### Top-1 Filter
`WHERE row_number = 1`:
- Keeps only the top driver per window
- Could be modified to top-N by changing condition

## Expected Output
Most active driver for each sliding hour window:
```json
{"start": "2023-09-18T14:00:00", "end": "2023-09-18T15:00:00", "driver_id": 184, "count": 12, "row_number": 1}
{"start": "2023-09-18T14:01:00", "end": "2023-09-18T15:01:00", "driver_id": 184, "count": 12, "row_number": 1}
{"start": "2023-09-18T14:02:00", "end": "2023-09-18T15:02:00", "driver_id": 156, "count": 11, "row_number": 1}
```

Use cases:
- Real-time leaderboards
- Performance tracking
- Identifying top performers
- Anomaly detection (unusually active entities)