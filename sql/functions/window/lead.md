---
title: LEAD
description: Accesses data from a following row in the result set
---

# LEAD

## Description
The `LEAD` window function provides access to a row at a specified physical offset after the current row within a partition. It allows you to look ahead in your result set and compare values between the current row and future rows without using self-joins. This function is particularly useful for predictive analysis, identifying upcoming events, and calculating forward-looking metrics.

## Syntax
```sql
LEAD(expression [, offset [, default]]) OVER (
    [PARTITION BY partition_expression, ...]
    ORDER BY sort_expression [ASC|DESC], ...
)
```

### Parameters
- **expression** (required): The column or expression to retrieve from the following row
- **offset** (optional): Number of rows forward from the current row (default: 1)
- **default** (optional): Value to return when the offset goes beyond the partition boundary (default: NULL)
- **PARTITION BY** (optional): Divides the result set into partitions
- **ORDER BY** (required): Determines the order for LEAD calculation

### Return Value
- **Type**: Same as the expression type
- **Description**: Value from the following row, or default/NULL if no following row exists

## Example

### Sample Data
Consider the following `project_milestones` table:

| project_id | milestone_date | milestone_name        | status     | budget_spent |
|------------|---------------|-----------------------|------------|--------------|
| 1          | 2024-01-15    | Project Kickoff       | Completed  | 10000        |
| 1          | 2024-02-28    | Design Phase          | Completed  | 35000        |
| 1          | 2024-04-15    | Development Phase     | In Progress| 78000        |
| 1          | 2024-06-30    | Testing Phase         | Planned    | 0            |
| 1          | 2024-08-15    | Deployment            | Planned    | 0            |
| 2          | 2024-02-01    | Requirements Gathering| Completed  | 8000         |
| 2          | 2024-03-15    | Architecture Design   | Completed  | 22000        |
| 2          | 2024-05-01    | Implementation        | In Progress| 45000        |
| 2          | 2024-06-15    | User Acceptance       | Planned    | 0            |

### Query
```sql
SELECT 
    project_id,
    milestone_date,
    milestone_name,
    status,
    LEAD(milestone_name) OVER (PARTITION BY project_id ORDER BY milestone_date) AS next_milestone,
    LEAD(milestone_date) OVER (PARTITION BY project_id ORDER BY milestone_date) AS next_milestone_date,
    LEAD(milestone_date) OVER (PARTITION BY project_id ORDER BY milestone_date) - milestone_date AS days_to_next,
    budget_spent,
    LEAD(budget_spent, 1, 0) OVER (PARTITION BY project_id ORDER BY milestone_date) AS next_budget
FROM project_milestones
ORDER BY project_id, milestone_date;
```

### Result
| project_id | milestone_date | milestone_name         | status      | next_milestone     | next_milestone_date | days_to_next | budget_spent | next_budget |
|------------|---------------|------------------------|-------------|-------------------|-------------------|--------------|--------------|-------------|
| 1          | 2024-01-15    | Project Kickoff        | Completed   | Design Phase      | 2024-02-28        | 44           | 10000        | 35000       |
| 1          | 2024-02-28    | Design Phase           | Completed   | Development Phase | 2024-04-15        | 47           | 35000        | 78000       |
| 1          | 2024-04-15    | Development Phase      | In Progress | Testing Phase     | 2024-06-30        | 76           | 78000        | 0           |
| 1          | 2024-06-30    | Testing Phase          | Planned     | Deployment        | 2024-08-15        | 46           | 0            | 0           |
| 1          | 2024-08-15    | Deployment             | Planned     | NULL              | NULL              | NULL         | 0            | 0           |
| 2          | 2024-02-01    | Requirements Gathering | Completed   | Architecture Design| 2024-03-15        | 43           | 8000         | 22000       |
| 2          | 2024-03-15    | Architecture Design    | Completed   | Implementation    | 2024-05-01        | 47           | 22000        | 45000       |
| 2          | 2024-05-01    | Implementation         | In Progress | User Acceptance   | 2024-06-15        | 45           | 45000        | 0           |
| 2          | 2024-06-15    | User Acceptance        | Planned     | NULL              | NULL              | NULL         | 0            | 0           |

## Common Use Cases

### 1. Event Sequence Analysis
```sql
-- Analyze user journey through application events
WITH user_journey AS (
    SELECT 
        user_id,
        event_timestamp,
        event_name,
        LEAD(event_name) OVER (PARTITION BY user_id ORDER BY event_timestamp) AS next_event,
        LEAD(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp) AS next_event_time,
        EXTRACT(EPOCH FROM 
            LEAD(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp) - event_timestamp
        ) AS seconds_to_next
    FROM user_events
)
SELECT 
    event_name,
    next_event,
    COUNT(*) AS transition_count,
    AVG(seconds_to_next) AS avg_time_to_next,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY seconds_to_next) AS median_time_to_next
FROM user_journey
WHERE next_event IS NOT NULL
GROUP BY event_name, next_event
ORDER BY transition_count DESC;
```

### 2. Predictive Maintenance Scheduling
```sql
-- Identify upcoming maintenance needs
SELECT 
    equipment_id,
    maintenance_date,
    maintenance_type,
    LEAD(maintenance_date) OVER (PARTITION BY equipment_id ORDER BY maintenance_date) AS next_maintenance,
    LEAD(maintenance_date) OVER (PARTITION BY equipment_id ORDER BY maintenance_date) - CURRENT_DATE AS days_until_next,
    CASE 
        WHEN LEAD(maintenance_date) OVER (PARTITION BY equipment_id ORDER BY maintenance_date) - CURRENT_DATE <= 7 
        THEN 'Urgent - Within 1 Week'
        WHEN LEAD(maintenance_date) OVER (PARTITION BY equipment_id ORDER BY maintenance_date) - CURRENT_DATE <= 30 
        THEN 'Soon - Within 1 Month'
        WHEN LEAD(maintenance_date) OVER (PARTITION BY equipment_id ORDER BY maintenance_date) - CURRENT_DATE <= 90 
        THEN 'Scheduled - Within 3 Months'
        ELSE 'Future'
    END AS priority
FROM maintenance_schedule
WHERE maintenance_date >= CURRENT_DATE
ORDER BY days_until_next NULLS LAST;
```

### 3. Price Change Detection
```sql
-- Detect and analyze price changes
WITH price_changes AS (
    SELECT 
        product_id,
        effective_date,
        price,
        LEAD(price) OVER (PARTITION BY product_id ORDER BY effective_date) AS next_price,
        LEAD(effective_date) OVER (PARTITION BY product_id ORDER BY effective_date) AS next_price_date,
        LEAD(price) OVER (PARTITION BY product_id ORDER BY effective_date) - price AS price_change,
        ROUND((LEAD(price) OVER (PARTITION BY product_id ORDER BY effective_date) - price) / price * 100, 2) AS price_change_pct
    FROM product_pricing
)
SELECT 
    product_id,
    effective_date,
    price,
    next_price,
    next_price_date,
    price_change,
    price_change_pct,
    CASE 
        WHEN price_change_pct > 10 THEN 'Major Increase'
        WHEN price_change_pct > 0 THEN 'Minor Increase'
        WHEN price_change_pct < -10 THEN 'Major Decrease'
        WHEN price_change_pct < 0 THEN 'Minor Decrease'
        ELSE 'No Change'
    END AS change_category
FROM price_changes
WHERE next_price IS NOT NULL
ORDER BY ABS(price_change_pct) DESC;
```

### 4. Customer Churn Prediction
```sql
-- Identify customers likely to churn based on activity patterns
WITH activity_gaps AS (
    SELECT 
        customer_id,
        activity_date,
        LEAD(activity_date) OVER (PARTITION BY customer_id ORDER BY activity_date) AS next_activity,
        LEAD(activity_date) OVER (PARTITION BY customer_id ORDER BY activity_date) - activity_date AS days_until_next,
        AVG(LEAD(activity_date) OVER (PARTITION BY customer_id ORDER BY activity_date) - activity_date) 
            OVER (PARTITION BY customer_id) AS avg_days_between
    FROM customer_activities
)
SELECT 
    customer_id,
    MAX(activity_date) AS last_activity,
    CURRENT_DATE - MAX(activity_date) AS days_since_last,
    AVG(days_until_next) AS avg_days_between_activities,
    CASE 
        WHEN CURRENT_DATE - MAX(activity_date) > AVG(days_until_next) * 3 THEN 'High Risk'
        WHEN CURRENT_DATE - MAX(activity_date) > AVG(days_until_next) * 2 THEN 'Medium Risk'
        WHEN CURRENT_DATE - MAX(activity_date) > AVG(days_until_next) * 1.5 THEN 'Low Risk'
        ELSE 'Active'
    END AS churn_risk
FROM activity_gaps
GROUP BY customer_id
HAVING MAX(activity_date) < CURRENT_DATE - INTERVAL '7 days'
ORDER BY days_since_last DESC;
```

### 5. Shift Scheduling Analysis
```sql
-- Analyze shift patterns and coverage gaps
SELECT 
    employee_id,
    shift_date,
    shift_start,
    shift_end,
    LEAD(shift_date) OVER (PARTITION BY employee_id ORDER BY shift_date) AS next_shift_date,
    LEAD(shift_start) OVER (PARTITION BY employee_id ORDER BY shift_date) AS next_shift_start,
    -- Calculate rest period between shifts
    EXTRACT(HOUR FROM 
        LEAD(shift_start) OVER (PARTITION BY employee_id ORDER BY shift_date) - shift_end
    ) AS rest_hours,
    CASE 
        WHEN LEAD(shift_date) OVER (PARTITION BY employee_id ORDER BY shift_date) = shift_date 
        THEN 'Double Shift'
        WHEN LEAD(shift_date) OVER (PARTITION BY employee_id ORDER BY shift_date) = shift_date + 1 
        THEN 'Consecutive Days'
        ELSE 'Regular'
    END AS shift_pattern
FROM employee_shifts
ORDER BY employee_id, shift_date;
```

## Combining LAG and LEAD

```sql
-- Compare previous, current, and next values
SELECT 
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
    LEAD(revenue) OVER (ORDER BY month) AS next_revenue,
    -- Identify local maxima/minima
    CASE 
        WHEN revenue > LAG(revenue) OVER (ORDER BY month) 
         AND revenue > LEAD(revenue) OVER (ORDER BY month) THEN 'Peak'
        WHEN revenue < LAG(revenue) OVER (ORDER BY month) 
         AND revenue < LEAD(revenue) OVER (ORDER BY month) THEN 'Valley'
        WHEN revenue > LAG(revenue) OVER (ORDER BY month) 
         AND revenue < LEAD(revenue) OVER (ORDER BY month) THEN 'Rising'
        WHEN revenue < LAG(revenue) OVER (ORDER BY month) 
         AND revenue > LEAD(revenue) OVER (ORDER BY month) THEN 'Falling'
        ELSE 'Stable'
    END AS trend_position
FROM monthly_revenue
ORDER BY month;
```

## Multiple LEAD Offsets

```sql
-- Look ahead multiple periods
SELECT 
    quarter,
    sales,
    LEAD(sales, 1) OVER (ORDER BY quarter) AS next_q1,
    LEAD(sales, 2) OVER (ORDER BY quarter) AS next_q2,
    LEAD(sales, 3) OVER (ORDER BY quarter) AS next_q3,
    LEAD(sales, 4) OVER (ORDER BY quarter) AS next_year,
    -- Calculate forward-looking growth rates
    ROUND((LEAD(sales, 4) OVER (ORDER BY quarter) - sales) / sales * 100, 2) AS projected_annual_growth
FROM quarterly_sales
ORDER BY quarter;
```

## Gap Fill with LEAD

```sql
-- Fill gaps in time series data
WITH filled_data AS (
    SELECT 
        date,
        value,
        LEAD(date) OVER (ORDER BY date) AS next_date,
        LEAD(value) OVER (ORDER BY date) AS next_value,
        -- Linear interpolation for missing dates
        value + (LEAD(value) OVER (ORDER BY date) - value) / 
                NULLIF(LEAD(date) OVER (ORDER BY date) - date, 0) AS interpolated_daily_value
    FROM sparse_time_series
)
SELECT 
    date,
    value,
    next_date,
    next_date - date - 1 AS gap_days,
    interpolated_daily_value
FROM filled_data
WHERE next_date - date > 1  -- Find gaps
ORDER BY date;
```

## Performance Optimization

```sql
-- Create indexes for LEAD queries
CREATE INDEX idx_events_user_time ON user_events(user_id, event_timestamp);
CREATE INDEX idx_milestones_project_date ON project_milestones(project_id, milestone_date);

-- Efficient LEAD usage with filtering
WITH future_analysis AS (
    SELECT 
        id,
        value,
        date,
        LEAD(value) OVER (ORDER BY date) AS next_value
    FROM large_dataset
    WHERE date >= CURRENT_DATE - INTERVAL '90 days'
      AND date <= CURRENT_DATE + INTERVAL '90 days'
)
SELECT * FROM future_analysis
WHERE next_value > value * 1.2;  -- Find 20% increases
```

## Related Functions
- [LAG](./lag.md) - Access previous rows
- [FIRST_VALUE](./first_value.md) - First value in window
- [LAST_VALUE](./last_value.md) - Last value in window
- [NTH_VALUE](./nth_value.md) - Nth value in window
- [ROW_NUMBER](./row_number.md) - Sequential row numbering