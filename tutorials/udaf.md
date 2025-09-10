---
title: User-Defined Aggregate Functions (UDAF)
---


## Overview
This example demonstrates custom aggregate functions (UDAFs) that extend SQL with domain-specific aggregation logic like median calculation and custom computations.

## Key Concepts
- User-defined aggregate functions
- Custom aggregation logic
- TUMBLE windows with UDAFs
- Multiple UDAFs in single query

## Test Data
This example uses: `impulse.json`

**Download**: [impulse.json](/test-data/impulse.json)

Sequential counter data for demonstrating custom aggregations.

## SQL Query

### Source Configuration
Stream source with sequential data for custom aggregate function testing:
```sql
CREATE TABLE impulse_source (
  timestamp TIMESTAMP,
  counter bigint unsigned not null,
  subtask_index bigint unsigned not null
) WITH (
  connector = 'single_file',
  path = '$input_dir/impulse.json',
  format = 'json',
  type = 'source'
);
```

### Sink Configuration
Output table for custom aggregate function results:
```sql
CREATE TABLE udaf (
  median bigint,
  none_value double,
  max_product bigint
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```

### Processing Pipeline
Windowed query applying multiple user-defined aggregate functions:
```sql
INSERT INTO udaf
SELECT median, none_value, max_product 
FROM (
  SELECT
    tumble(interval '30' day) as window,
    my_median(counter) as median,
    none_udf(counter) as none_value,
    max_product(counter, subtask_index) as max_product
  FROM impulse_source
  GROUP BY 1
)
```

## Query Breakdown

### Window Definition
`tumble(interval '30' day)`:
- Large 30-day windows to aggregate all test data
- Non-overlapping windows
- Ensures all UDAFs process complete dataset

### Custom Aggregate Functions

**my_median(counter)**:
- Calculates the median value of the counter field
- Maintains sorted list internally
- Returns middle value (or average of two middle values)

**none_udf(counter)**:
- Example UDAF that might return NULL or special value
- Demonstrates handling of nullable results
- Could implement custom business logic

**max_product(counter, subtask_index)**:
- Takes two parameters
- Might compute maximum of (counter × subtask_index)
- Shows multi-parameter UDAF usage

### UDAF Implementation
UDAFs typically require:
1. **Accumulator**: State maintained during aggregation
2. **Accumulate**: Add new value to state
3. **Merge**: Combine two accumulators (for distributed processing)
4. **Finalize**: Extract final result from accumulator

## Expected Output
Custom aggregation results:
```json
{"median": 50, "none_value": null, "max_product": 999}
```

Results depend on UDAF implementations:
- Median: Middle value of all counters
- None value: Could be NULL or computed value
- Max product: Maximum of counter × subtask_index

Use cases:
- Statistical functions (percentiles, standard deviation)
- Business-specific metrics
- Complex aggregations not available in standard SQL
- Domain-specific calculations