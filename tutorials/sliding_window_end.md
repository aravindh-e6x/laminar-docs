---
title: Sliding Window (HOP) - Overlapping Windows
---


## Overview
This example demonstrates HOP windows (sliding windows) that create overlapping time windows, computing aggregates that update more frequently than the window size.

## Key Concepts
- HOP window function for sliding/overlapping windows
- Window slide interval vs window size
- Event time processing with event_time_field
- Window start and end timestamps in output

## Test Data
**Download**: [impulse.json](/test-data/impulse.json)

Time-series data with regular timestamps at 200ms intervals.

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
  event_time_field = 'timestamp',
  type = 'source'
);
```
Configures the source with `event_time_field` to use the timestamp column for window calculations, ensuring proper time-based processing.

### 2. Sink Configuration
```sql
CREATE TABLE impulse_sink (
  count bigint,
  min bigint,
  max bigint,
  start timestamp,
  end timestamp
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```
Output table includes window boundaries (start/end) along with the aggregated statistics for each window.

### 3. Processing Pipeline
```sql
INSERT INTO impulse_sink
SELECT count, min, max, window.start, window.end 
FROM (
  SELECT
    hop(interval '2 second', interval '10 second') as window,
    count(*) as count,
    min(counter) as min,
    max(counter) as max
  FROM impulse_source
  GROUP BY 1
);
```
Creates sliding windows using HOP function with 2-second slide and 10-second duration. This means:
- Each window spans 10 seconds
- New window starts every 2 seconds
- Windows overlap by 8 seconds (80% overlap)
- Each event appears in multiple windows
```

## Query Breakdown

### HOP Window Function
`hop(interval '2 second', interval '10 second')` creates:
- Windows of 10 seconds duration
- New window every 2 seconds (slide interval)
- 80% overlap between consecutive windows

### Window Overlap Pattern
With a 2-second slide and 10-second duration:
- Window 1: [0s - 10s]
- Window 2: [2s - 12s] (overlaps 8s with Window 1)
- Window 3: [4s - 14s] (overlaps 8s with Window 2)
- And so on...

### Event Time Processing
`event_time_field = 'timestamp'` ensures windows are based on event timestamps, not processing time.

### Window Metadata
The output includes `window.start` and `window.end` timestamps, showing the exact time boundaries of each window.

## Expected Output
Each output record represents one 10-second window with statistics:
```json
{"count": 50, "min": 0, "max": 49, "start": "2023-10-09T17:13:20", "end": "2023-10-09T17:13:30"}
{"count": 50, "min": 10, "max": 59, "start": "2023-10-09T17:13:22", "end": "2023-10-09T17:13:32"}
{"count": 50, "min": 20, "max": 69, "start": "2023-10-09T17:13:24", "end": "2023-10-09T17:13:34"}
```

Key observations:
- Each window contains 50 records (10 seconds ÷ 200ms interval)
- Windows overlap, so records appear in multiple windows
- New results emit every 2 seconds with updated statistics
- Useful for smooth, frequently-updating metrics and moving averages