---
title: BOOL_AND
description: Returns true if all boolean values are true
---

# BOOL_AND

## Description
The `BOOL_AND` function returns true if all non-NULL boolean values in a group are true. It implements logical AND across multiple rows, making it useful for checking if all conditions are met across a dataset. NULL values are ignored. If all values are NULL or there are no rows, it returns NULL.

## Syntax
```sql
BOOL_AND(boolean_expression)
EVERY(boolean_expression)  -- SQL standard alias
```

### Parameters
- **boolean_expression**: A column or expression that evaluates to a boolean value

### Return Value
- **Type**: BOOLEAN
- **Description**: TRUE if all non-NULL values are TRUE, FALSE if any value is FALSE, NULL if all values are NULL

## Example

### Sample Data
Consider the following `system_health` table monitoring service statuses:

| service_id | service_name | timestamp           | is_healthy | response_time_ms | cpu_usage |
|------------|-------------|---------------------|------------|------------------|-----------|
| 1          | API Gateway | 2024-03-15 10:00:00 | true       | 45              | 35        |
| 2          | Database    | 2024-03-15 10:00:00 | true       | 12              | 60        |
| 3          | Cache       | 2024-03-15 10:00:00 | true       | 2               | 15        |
| 1          | API Gateway | 2024-03-15 10:01:00 | true       | 48              | 38        |
| 2          | Database    | 2024-03-15 10:01:00 | false      | 850             | 95        |
| 3          | Cache       | 2024-03-15 10:01:00 | true       | 3               | 18        |
| 1          | API Gateway | 2024-03-15 10:02:00 | true       | 52              | 40        |
| 2          | Database    | 2024-03-15 10:02:00 | true       | 15              | 65        |

### Query
```sql
SELECT 
    DATE_TRUNC('minute', timestamp) AS minute,
    BOOL_AND(is_healthy) AS all_services_healthy,
    BOOL_OR(NOT is_healthy) AS any_service_unhealthy,
    COUNT(*) AS check_count,
    COUNT(CASE WHEN is_healthy THEN 1 END) AS healthy_count
FROM system_health
GROUP BY DATE_TRUNC('minute', timestamp)
ORDER BY minute;
```

### Result
| minute              | all_services_healthy | any_service_unhealthy | check_count | healthy_count |
|--------------------|---------------------|----------------------|-------------|---------------|
| 2024-03-15 10:00:00| true                | false                | 3           | 3             |
| 2024-03-15 10:01:00| false               | true                 | 3           | 2             |
| 2024-03-15 10:02:00| true                | false                | 2           | 2             |

## Common Use Cases

### 1. System Availability Monitoring
```sql
-- Check if all critical services are running
SELECT 
    DATE_TRUNC('hour', check_time) AS hour,
    BOOL_AND(is_running) AS all_services_up,
    BOOL_AND(response_time < 1000) AS all_within_sla,
    BOOL_AND(error_rate < 0.01) AS acceptable_error_rates,
    CASE 
        WHEN BOOL_AND(is_running AND response_time < 1000 AND error_rate < 0.01) 
        THEN 'Healthy'
        ELSE 'Degraded'
    END AS system_status
FROM service_metrics
WHERE is_critical = true
GROUP BY DATE_TRUNC('hour', check_time)
ORDER BY hour DESC;
```

### 2. Data Quality Validation
```sql
-- Validate data completeness and quality
SELECT 
    table_name,
    BOOL_AND(record_count > 0) AS has_data,
    BOOL_AND(null_percentage < 5) AS low_null_rate,
    BOOL_AND(duplicate_percentage < 1) AS low_duplicate_rate,
    BOOL_AND(
        record_count > 0 AND 
        null_percentage < 5 AND 
        duplicate_percentage < 1
    ) AS passes_quality_check
FROM data_quality_metrics
WHERE check_date = CURRENT_DATE
GROUP BY table_name;
```

### 3. Compliance Checking
```sql
-- Verify all compliance requirements are met
SELECT 
    department,
    BOOL_AND(training_completed) AS all_trained,
    BOOL_AND(certification_valid) AS all_certified,
    BOOL_AND(background_check_passed) AS all_cleared,
    BOOL_AND(
        training_completed AND 
        certification_valid AND 
        background_check_passed
    ) AS fully_compliant,
    COUNT(*) AS employee_count
FROM employee_compliance
WHERE employment_status = 'Active'
GROUP BY department
HAVING NOT BOOL_AND(
    training_completed AND 
    certification_valid AND 
    background_check_passed
);  -- Show only non-compliant departments
```

### 4. Feature Flag Readiness
```sql
-- Check if feature can be enabled globally
SELECT 
    feature_name,
    BOOL_AND(canary_test_passed) AS all_canary_passed,
    BOOL_AND(performance_acceptable) AS performance_ok,
    BOOL_AND(no_critical_bugs) AS bug_free,
    BOOL_AND(rollback_tested) AS rollback_ready,
    MIN(test_coverage_percent) AS min_coverage,
    CASE 
        WHEN BOOL_AND(
            canary_test_passed AND 
            performance_acceptable AND 
            no_critical_bugs AND 
            rollback_tested
        ) AND MIN(test_coverage_percent) > 80
        THEN 'Ready for Production'
        ELSE 'Not Ready'
    END AS deployment_status
FROM feature_readiness
GROUP BY feature_name;
```

### 5. Transaction Validation
```sql
-- Validate all steps in multi-step transactions
WITH transaction_steps AS (
    SELECT 
        transaction_id,
        step_number,
        step_name,
        is_successful,
        execution_time
    FROM transaction_log
    WHERE transaction_date = CURRENT_DATE
)
SELECT 
    transaction_id,
    BOOL_AND(is_successful) AS all_steps_successful,
    COUNT(*) AS total_steps,
    SUM(execution_time) AS total_time,
    ARRAY_AGG(
        CASE WHEN NOT is_successful THEN step_name END
    ) FILTER (WHERE NOT is_successful) AS failed_steps
FROM transaction_steps
GROUP BY transaction_id
HAVING NOT BOOL_AND(is_successful);  -- Show only failed transactions
```

## Conditional Aggregation

```sql
-- Use BOOL_AND with conditions
SELECT 
    product_category,
    BOOL_AND(in_stock) AS all_in_stock,
    BOOL_AND(price > cost) AS all_profitable,
    BOOL_AND(quality_score >= 4) AS all_high_quality,
    -- Conditional BOOL_AND
    BOOL_AND(in_stock) FILTER (WHERE is_featured = true) AS featured_in_stock,
    BOOL_AND(review_score >= 4) FILTER (WHERE review_count > 10) AS well_reviewed
FROM products
GROUP BY product_category;
```

## Combining with Other Boolean Functions

```sql
-- Complex boolean logic across rows
SELECT 
    server_cluster,
    BOOL_AND(cpu_usage < 80) AS cpu_ok,
    BOOL_AND(memory_usage < 90) AS memory_ok,
    BOOL_AND(disk_usage < 85) AS disk_ok,
    BOOL_OR(needs_restart) AS any_need_restart,
    -- Combined check
    BOOL_AND(cpu_usage < 80 AND memory_usage < 90 AND disk_usage < 85) 
        AND NOT BOOL_OR(needs_restart) AS cluster_healthy
FROM server_metrics
WHERE metric_time >= CURRENT_TIMESTAMP - INTERVAL '5 minutes'
GROUP BY server_cluster;
```

## NULL Handling

```sql
-- BOOL_AND behavior with NULLs
WITH test_data AS (
    SELECT * FROM (VALUES 
        (1, true),
        (1, true),
        (1, NULL),
        (2, false),
        (2, true),
        (2, NULL),
        (3, NULL),
        (3, NULL)
    ) AS t(group_id, value)
)
SELECT 
    group_id,
    BOOL_AND(value) AS bool_and_result,  -- NULLs ignored
    BOOL_OR(value) AS bool_or_result,
    COUNT(*) AS total_rows,
    COUNT(value) AS non_null_count
FROM test_data
GROUP BY group_id;
-- Results:
-- group_id 1: true (all non-NULL are true)
-- group_id 2: false (contains false)
-- group_id 3: NULL (all NULL)
```

## Performance Optimization

```sql
-- Short-circuit evaluation for better performance
-- BOOL_AND returns false as soon as it encounters a false value

-- Efficient: check simple conditions first
SELECT 
    BOOL_AND(
        is_active AND  -- Simple boolean check first
        price > 0 AND  -- Simple comparison
        EXTRACT(YEAR FROM created_date) = 2024  -- More complex last
    ) AS all_valid
FROM large_table;

-- Create partial index for boolean columns
CREATE INDEX idx_active_products ON products(category) 
WHERE is_active = true AND in_stock = true;
```

## Related Functions
- [BOOL_OR](./bool_or.md) - Returns true if any value is true
- [EVERY](./every.md) - Alias for BOOL_AND
- [SOME](./some.md) - Alias for BOOL_OR
- [BIT_AND](./bit_and.md) - Bitwise AND aggregation
- [BIT_OR](./bit_or.md) - Bitwise OR aggregation