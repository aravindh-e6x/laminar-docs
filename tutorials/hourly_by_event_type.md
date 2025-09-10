---
title: Hourly by Event Type - Multi-Dimensional Aggregation
---


## Overview
This example demonstrates grouping by both a data dimension (event_type) and a time window, creating multi-dimensional aggregations for time-series analysis.

## Key Concepts
- GROUP BY with multiple dimensions
- Combining data fields with window functions
- TUMBLE for fixed hourly buckets
- Time-series aggregation patterns

## Test Data
This example uses: `cars.json`

**Download**: [cars.json](/test-data/cars.json)

Ride-sharing events with different event types (pickup, dropoff).

## SQL Query

### Source Configuration
Stream source with event time field specification:
```sql
CREATE TABLE cars(
  timestamp TIMESTAMP,
  driver_id BIGINT,
  event_type TEXT,
  location TEXT
) WITH (
  connector = 'single_file',
  path = '$input_dir/cars.json',
  format = 'json',
  type = 'source',
  event_time_field = 'timestamp'
);
```

### Sink Configuration
Output table for multi-dimensional aggregated results:
```sql
CREATE TABLE group_by_aggregate (
  event_type TEXT,
  hour TIMESTAMP,
  count BIGINT
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```

### Processing Pipeline
Multi-dimensional aggregation grouping by event type and hourly windows:
```sql
INSERT INTO group_by_aggregate
SELECT event_type, window.start as hour, count
FROM (
  SELECT 
    event_type,
    TUMBLE(INTERVAL '1' HOUR) as window,
    COUNT(*) as count
  FROM cars
  GROUP BY 1,2
);
```

## Query Breakdown

### Multi-Dimensional Grouping
`GROUP BY 1,2` groups by:
1. `event_type`: Creates separate groups for 'pickup' and 'dropoff'
2. `TUMBLE(INTERVAL '1' HOUR)`: Creates hourly time buckets

This produces a matrix of results: event_type × hour.

### Window Function
`TUMBLE(INTERVAL '1' HOUR)`:
- Creates fixed 1-hour windows
- Non-overlapping buckets
- Aligned to hour boundaries

### Aggregation
`COUNT(*)`:
- Counts events per type per hour
- Simple but powerful for trend analysis

### Output Structure
Each record contains:
- `event_type`: The type of event
- `hour`: Start of the hour window
- `count`: Number of events in that hour

## Expected Output
Hourly counts broken down by event type:
```json
{"event_type": "pickup", "hour": "2023-09-18T14:00:00", "count": 28}
{"event_type": "dropoff", "hour": "2023-09-18T14:00:00", "count": 25}
{"event_type": "pickup", "hour": "2023-09-18T15:00:00", "count": 32}
{"event_type": "dropoff", "hour": "2023-09-18T15:00:00", "count": 30}
{"event_type": "pickup", "hour": "2023-09-18T16:00:00", "count": 35}
{"event_type": "dropoff", "hour": "2023-09-18T16:00:00", "count": 33}
```

Analysis insights:
- Compare pickup vs dropoff patterns
- Identify peak hours per event type
- Track supply/demand balance
- Detect anomalies in event patterns

Use cases:
- Business metrics dashboards
- Time-series analysis by category
- Comparative trend analysis
- Capacity planning by type and time