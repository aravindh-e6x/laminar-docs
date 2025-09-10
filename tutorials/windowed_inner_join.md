---
title: Windowed Inner Join - Joining Stream Windows
---


## Overview
This example shows how to join two windowed aggregations using an INNER JOIN, combining results from different event types within the same time windows.

## Key Concepts
- TUMBLE window function for fixed-size windows
- INNER JOIN on window boundaries
- Separate aggregations joined together
- Filtering before aggregation

## Test Data
**Download**: [cars.json](/test-data/cars.json)

Ride-sharing events with pickup and dropoff event types.

## SQL Query

### 1. Source Configuration
```sql
CREATE TABLE cars (
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
 Creates a source table for ride-sharing events. The `event_time_field` setting tells the system to use the `timestamp` field for time-based operations like windowing, ensuring events are processed based on when they occurred rather than when they arrive.

### 2. Sink Configuration
```sql
CREATE TABLE hourly_aggregates (
  hour TIMESTAMP,
  drivers BIGINT,
  pickups BIGINT
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```
 Defines the output table for hourly statistics. Each row will contain the hour timestamp and counts for both dropoff and pickup drivers. The JSON format makes the output easy to consume by downstream systems.

### 3. Processing Pipeline
```sql
INSERT INTO hourly_aggregates
SELECT window.start as hour, dropoff_drivers, pickup_drivers 
FROM (
  SELECT dropoffs.window as window, dropoff_drivers, pickup_drivers
  FROM (
    SELECT 
      TUMBLE(INTERVAL '1' hour) as window,
      COUNT(distinct driver_id) as dropoff_drivers 
    FROM cars 
    WHERE event_type = 'dropoff'
    GROUP BY 1
  ) dropoffs
  INNER JOIN (
    SELECT 
      TUMBLE(INTERVAL '1' hour) as window,
      COUNT(distinct driver_id) as pickup_drivers 
    FROM cars 
    WHERE event_type = 'pickup'
    GROUP BY 1
  ) pickups
  ON dropoffs.window = pickups.window
)
```
 This pipeline performs a complex join operation:

1. **Left branch (dropoffs)**: Filters for dropoff events, groups them into 1-hour windows, and counts unique drivers
2. **Right branch (pickups)**: Does the same for pickup events in parallel
3. **Join operation**: Combines results where the time windows match exactly
4. **Output**: Extracts the window start time and both driver counts

The INNER JOIN ensures output only when both pickups and dropoffs occur in the same hour, providing balanced metrics for supply/demand analysis.

## Expected Output
Hourly statistics showing driver activity:
```json
{"hour": "2023-09-18T14:00:00", "drivers": 45, "pickups": 52}
{"hour": "2023-09-18T15:00:00", "drivers": 38, "pickups": 41}
{"hour": "2023-09-18T16:00:00", "drivers": 50, "pickups": 48}
```

Note:
- Output only for hours with both pickups AND dropoffs
- Missing hours indicate no activity in one or both categories
- Useful for balanced metrics and comparative analysis