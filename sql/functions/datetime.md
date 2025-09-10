---
sidebar_position: 3
---

# Date/Time Functions

Functions for working with dates, times, timestamps, and intervals.

## Current Date/Time

### now() / current_timestamp
Returns the current timestamp.
```sql
SELECT now() -- Returns: '2024-01-15 14:30:45.123456'
SELECT current_timestamp -- Returns: '2024-01-15 14:30:45.123456'
```

### current_date
Returns the current date.
```sql
SELECT current_date -- Returns: '2024-01-15'
```

### current_time
Returns the current time.
```sql
SELECT current_time -- Returns: '14:30:45.123456'
```

### today()
Returns today's date at midnight.
```sql
SELECT today() -- Returns: '2024-01-15 00:00:00'
```

## Date/Time Construction

### make_date(year, month, day)
Creates a date from components.
```sql
SELECT make_date(2024, 1, 15) -- Returns: '2024-01-15'
```

### make_timestamp(year, month, day, hour, minute, second)
Creates a timestamp from components.
```sql
SELECT make_timestamp(2024, 1, 15, 14, 30, 45.5) 
-- Returns: '2024-01-15 14:30:45.500000'
```

### to_timestamp(unix_seconds)
Converts Unix timestamp to timestamp.
```sql
SELECT to_timestamp(1705329045) -- Returns: '2024-01-15 14:30:45'
```

### to_timestamp_millis(unix_millis)
Converts Unix milliseconds to timestamp.
```sql
SELECT to_timestamp_millis(1705329045123) -- Returns: '2024-01-15 14:30:45.123'
```

### to_timestamp_micros(unix_micros)
Converts Unix microseconds to timestamp.
```sql
SELECT to_timestamp_micros(1705329045123456) -- Returns: '2024-01-15 14:30:45.123456'
```

### to_timestamp_nanos(unix_nanos)
Converts Unix nanoseconds to timestamp.
```sql
SELECT to_timestamp_nanos(1705329045123456789) -- Returns: '2024-01-15 14:30:45.123456789'
```

### to_timestamp_seconds(unix_seconds)
Alias for to_timestamp.
```sql
SELECT to_timestamp_seconds(1705329045) -- Returns: '2024-01-15 14:30:45'
```

## Date/Time Extraction

### extract(field FROM source) / date_part(field, source)
Extracts a specific field from date/time.
```sql
SELECT extract(YEAR FROM '2024-01-15'::date) -- Returns: 2024
SELECT extract(MONTH FROM '2024-01-15'::date) -- Returns: 1
SELECT extract(DAY FROM '2024-01-15'::date) -- Returns: 15
SELECT extract(HOUR FROM '14:30:45'::time) -- Returns: 14
SELECT extract(MINUTE FROM '14:30:45'::time) -- Returns: 30
SELECT extract(SECOND FROM '14:30:45.123'::time) -- Returns: 45.123
SELECT extract(DOW FROM '2024-01-15'::date) -- Returns: 1 (Monday)
SELECT extract(WEEK FROM '2024-01-15'::date) -- Returns: 3
SELECT extract(QUARTER FROM '2024-01-15'::date) -- Returns: 1
SELECT extract(EPOCH FROM '2024-01-15 14:30:45'::timestamp) -- Returns: 1705329045
```

### year(date)
Extracts the year.
```sql
SELECT year('2024-01-15'::date) -- Returns: 2024
```

### month(date)
Extracts the month.
```sql
SELECT month('2024-01-15'::date) -- Returns: 1
```

### day(date) / dayofmonth(date)
Extracts the day of month.
```sql
SELECT day('2024-01-15'::date) -- Returns: 15
SELECT dayofmonth('2024-01-15'::date) -- Returns: 15
```

### hour(timestamp)
Extracts the hour.
```sql
SELECT hour('2024-01-15 14:30:45'::timestamp) -- Returns: 14
```

### minute(timestamp)
Extracts the minute.
```sql
SELECT minute('2024-01-15 14:30:45'::timestamp) -- Returns: 30
```

### second(timestamp)
Extracts the second.
```sql
SELECT second('2024-01-15 14:30:45.123'::timestamp) -- Returns: 45.123
```

### quarter(date)
Extracts the quarter (1-4).
```sql
SELECT quarter('2024-04-15'::date) -- Returns: 2
```

### week(date) / weekofyear(date)
Extracts the week of year.
```sql
SELECT week('2024-01-15'::date) -- Returns: 3
SELECT weekofyear('2024-01-15'::date) -- Returns: 3
```

### dayofweek(date) / dow(date)
Extracts the day of week (1=Monday, 7=Sunday).
```sql
SELECT dayofweek('2024-01-15'::date) -- Returns: 1
SELECT dow('2024-01-15'::date) -- Returns: 1
```

### dayofyear(date) / doy(date)
Extracts the day of year.
```sql
SELECT dayofyear('2024-01-15'::date) -- Returns: 15
SELECT doy('2024-01-15'::date) -- Returns: 15
```

## Date/Time Truncation

### date_trunc(field, source)
Truncates to specified precision.
```sql
SELECT date_trunc('year', '2024-03-15 14:30:45'::timestamp) 
-- Returns: '2024-01-01 00:00:00'

SELECT date_trunc('month', '2024-03-15 14:30:45'::timestamp) 
-- Returns: '2024-03-01 00:00:00'

SELECT date_trunc('day', '2024-03-15 14:30:45'::timestamp) 
-- Returns: '2024-03-15 00:00:00'

SELECT date_trunc('hour', '2024-03-15 14:30:45'::timestamp) 
-- Returns: '2024-03-15 14:00:00'

SELECT date_trunc('minute', '2024-03-15 14:30:45'::timestamp) 
-- Returns: '2024-03-15 14:30:00'
```

### datetrunc(field, source)
Alias for date_trunc.
```sql
SELECT datetrunc('day', '2024-03-15 14:30:45'::timestamp) 
-- Returns: '2024-03-15 00:00:00'
```

## Date/Time Binning

### date_bin(interval, source, origin)
Bins timestamps into regular intervals.
```sql
SELECT date_bin(INTERVAL '15 minutes', '2024-01-15 14:37:00'::timestamp, '2024-01-15 00:00:00'::timestamp)
-- Returns: '2024-01-15 14:30:00'
```

## Date/Time Arithmetic

### date_add / dateadd / timestampadd
Adds an interval to a date/time.
```sql
-- Adding days
SELECT '2024-01-15'::date + INTERVAL '7 days' -- Returns: '2024-01-22'

-- Adding months
SELECT '2024-01-15'::date + INTERVAL '2 months' -- Returns: '2024-03-15'

-- Adding hours
SELECT '2024-01-15 12:00:00'::timestamp + INTERVAL '3 hours' 
-- Returns: '2024-01-15 15:00:00'
```

### date_sub / datesub / timestampsub
Subtracts an interval from a date/time.
```sql
SELECT '2024-01-15'::date - INTERVAL '7 days' -- Returns: '2024-01-08'
```

### datediff(unit, start, end)
Returns the difference between two dates.
```sql
SELECT datediff('day', '2024-01-15'::date, '2024-01-22'::date) -- Returns: 7
SELECT datediff('month', '2024-01-15'::date, '2024-03-15'::date) -- Returns: 2
SELECT datediff('year', '2020-01-01'::date, '2024-01-01'::date) -- Returns: 4
```

### age(timestamp1, timestamp2)
Returns the interval between two timestamps.
```sql
SELECT age('2024-01-15'::timestamp, '2020-01-15'::timestamp) 
-- Returns: '4 years'
```

## Formatting Functions

### to_char(timestamp, format)
Formats timestamp as string.
```sql
SELECT to_char('2024-01-15 14:30:45'::timestamp, 'YYYY-MM-DD HH24:MI:SS')
-- Returns: '2024-01-15 14:30:45'

SELECT to_char('2024-01-15'::date, 'Month DD, YYYY')
-- Returns: 'January   15, 2024'

SELECT to_char('2024-01-15'::date, 'Day')
-- Returns: 'Monday   '
```

### to_date(string, format)
Parses string to date.
```sql
SELECT to_date('15-01-2024', 'DD-MM-YYYY') -- Returns: '2024-01-15'
SELECT to_date('2024/01/15', 'YYYY/MM/DD') -- Returns: '2024-01-15'
```

### to_timestamp(string, format)
Parses string to timestamp.
```sql
SELECT to_timestamp('2024-01-15 14:30:45', 'YYYY-MM-DD HH24:MI:SS')
-- Returns: '2024-01-15 14:30:45'
```

## Unix Timestamp Conversion

### from_unixtime(unix_seconds)
Converts Unix timestamp to timestamp.
```sql
SELECT from_unixtime(1705329045) -- Returns: '2024-01-15 14:30:45'
```

### unix_timestamp([timestamp])
Converts timestamp to Unix timestamp.
```sql
SELECT unix_timestamp('2024-01-15 14:30:45'::timestamp) -- Returns: 1705329045
SELECT unix_timestamp() -- Returns: current Unix timestamp
```

## Interval Functions

### justify_days(interval)
Adjusts interval days to months.
```sql
SELECT justify_days(INTERVAL '35 days') -- Returns: '1 mon 5 days'
```

### justify_hours(interval)
Adjusts interval hours to days.
```sql
SELECT justify_hours(INTERVAL '27 hours') -- Returns: '1 day 03:00:00'
```

### justify_interval(interval)
Adjusts interval using both justify_days and justify_hours.
```sql
SELECT justify_interval(INTERVAL '1 mon -1 hour') -- Returns: '29 days 23:00:00'
```

## Timezone Functions

### timezone(zone, timestamp)
Converts timestamp to specified timezone.
```sql
SELECT timezone('America/New_York', '2024-01-15 14:30:45'::timestamp)
```

### at time zone
Converts timestamp between timezones.
```sql
SELECT '2024-01-15 14:30:45'::timestamp AT TIME ZONE 'UTC' AT TIME ZONE 'America/New_York'
```

## Usage Examples

### Calculate age in years
```sql
SELECT 
    extract(YEAR FROM age(current_date, birth_date)) as age_years
FROM users;
```

### Group by month
```sql
SELECT 
    date_trunc('month', created_at) as month,
    COUNT(*) as count
FROM orders
GROUP BY date_trunc('month', created_at)
ORDER BY month;
```

### Find records from last 30 days
```sql
SELECT *
FROM events
WHERE created_at >= current_date - INTERVAL '30 days';
```

### Calculate business days between dates
```sql
SELECT 
    COUNT(*) FILTER (WHERE extract(DOW FROM date_series) NOT IN (0, 6)) as business_days
FROM generate_series(
    '2024-01-01'::date,
    '2024-01-31'::date,
    '1 day'::interval
) as date_series;
```

### Format timestamps for display
```sql
SELECT 
    to_char(created_at, 'Mon DD, YYYY at HH12:MI AM') as formatted_date
FROM events;
```