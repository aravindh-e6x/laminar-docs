---
title: Session Window - Activity-Based Windows
---


## Overview
This example demonstrates SESSION windows that group events based on activity gaps, creating dynamic window boundaries when activity pauses for a specified duration.

## Key Concepts
- SESSION window function for gap-based windowing
- Dynamic window sizes based on activity patterns
- User session tracking
- Gap duration configuration

## Test Data
**Download**: [impulse.json](/test-data/impulse.json)

Regular time-series data used to simulate user sessions with activity gaps.

## SQL Query

### 1. Source Configuration
```sql
CREATE TABLE impulse_source (
  timestamp TIMESTAMP,
  counter bigint not null,
  subtask_index bigint not null
) WITH (
  connector = 'single_file',
  path = '$input_dir/impulse.json',
  format = 'json',
  type = 'source',
  event_time_field = 'timestamp'
);
```
Configures the source with event-time processing to properly detect gaps between events for session windowing.

### 2. Sink Configuration
```sql
CREATE TABLE session_window_output (
  start timestamp,
  end timestamp,
  user_id bigint,
  rows bigint
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```
Output includes session boundaries (start/end), user identifier, and count of events per session.

### 3. Processing Pipeline
```sql
INSERT INTO session_window_output
SELECT window.start, window.end, user_id, rows 
FROM (
  SELECT
    SESSION(interval '20 seconds') as window,
    CASE WHEN counter % 10 = 0 THEN 0 ELSE counter END as user_id,
    count(*) as rows
  FROM impulse_source 
  GROUP BY window, user_id
)
```
Creates session windows with 20-second gap detection. The CASE statement maps every 10th counter to user_id 0, simulating multiple users. Sessions close after 20 seconds of inactivity and new events after the gap start new sessions.
```

## Query Breakdown

### Session Window Function
`SESSION(interval '20 seconds')` creates windows that:
- Start when the first event arrives
- Extend with each new event
- Close when no events arrive for 20 seconds
- Each session has variable duration

### User ID Logic
`CASE WHEN counter % 10 = 0 THEN 0 ELSE counter END`:
- Maps every 10th counter (0, 10, 20...) to user_id 0
- Creates artificial user sessions
- Simulates multiple users with different activity patterns

### Session Boundaries
Sessions are created per user_id:
- A new session starts with the first event for a user
- Session extends as long as events keep arriving within 20 seconds
- Session closes after 20 seconds of inactivity
- Next event after gap starts a new session

## Expected Output
Output shows variable-length sessions based on activity:
```json
{"start": "2023-10-09T17:13:20", "end": "2023-10-09T17:13:40", "user_id": 1, "rows": 1}
{"start": "2023-10-09T17:13:20", "end": "2023-10-09T17:13:42", "user_id": 0, "rows": 11}
{"start": "2023-10-09T17:13:20.400", "end": "2023-10-09T17:13:40.400", "user_id": 2, "rows": 1}
```

Key patterns:
- Session duration varies based on activity
- Users with more events have longer sessions
- Gap detection creates natural session boundaries
- Useful for user behavior analysis, clickstream analytics, and activity tracking