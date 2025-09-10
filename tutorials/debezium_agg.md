---
title: Debezium Aggregations - CDC with Updates
---


## Overview
This example shows how to perform aggregations on CDC (Change Data Capture) data in Debezium format, handling inserts, updates, and deletes properly.

## Key Concepts
- Debezium format for CDC data
- Primary keys in source and sink tables
- Aggregations over changing data
- Automatic envelope unwrapping

## Test Data
This example uses: `aggregate_updates.json`

**Download**: [aggregate_updates.json](/test-data/aggregate_updates.json)

E-commerce order data in Debezium format with insert, update, and delete operations.

## SQL Query

### Source Configuration
CDC source table with primary key for tracking change events:
```sql
--pk=id
CREATE TABLE debezium_source (
  id INT PRIMARY KEY,
  customer_name TEXT,
  product_name TEXT,
  quantity INTEGER,
  price FLOAT,
  order_date TIMESTAMP,
  status TEXT
) WITH (
  connector = 'single_file',
  path = '$input_dir/aggregate_updates.json',
  format = 'debezium_json',
  type = 'source'
);
```

### Sink Configuration
Output table with primary key and Debezium format for change tracking:
```sql
CREATE TABLE output (
  id TEXT PRIMARY KEY,
  c INT,
  d INT,
  q INT
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'debezium_json',
  type = 'sink'
);
```

### Processing Pipeline
Aggregation query handling CDC operations with product-level metrics:
```sql
INSERT INTO output
SELECT 
  concat('p_', product_name),
  count(*),
  count(distinct customer_name),
  sum(quantity + 5) + 10
FROM debezium_source
GROUP BY concat('p_', product_name);
```

## Query Breakdown

### Debezium Source
- `format = 'debezium_json'` automatically unwraps the Debezium envelope
- Handles before/after states, operation types (create/update/delete)
- Primary key (`id`) tracks record identity for updates

### Aggregation Logic
The query computes per-product metrics:
- `concat('p_', product_name)`: Creates product ID with prefix
- `count(*)`: Total orders per product
- `count(distinct customer_name)`: Unique customers per product
- `sum(quantity + 5) + 10`: Custom quantity calculation

### CDC Semantics
With Debezium format:
- **Inserts**: Add to aggregations
- **Updates**: Remove old values, add new values
- **Deletes**: Remove from aggregations
- Aggregations stay correct despite changes

### Output Format
Sink also uses `debezium_json` to:
- Track aggregate changes over time
- Provide before/after states for downstream systems
- Enable further CDC processing

## Expected Output
Product aggregations that update with CDC changes:
```json
{"before": null, "after": {"id": "p_laptop", "c": 5, "d": 4, "q": 35}}
{"before": {"id": "p_laptop", "c": 5, "d": 4, "q": 35}, "after": {"id": "p_laptop", "c": 6, "d": 5, "q": 42}}
{"before": {"id": "p_phone", "c": 3, "d": 3, "q": 28}, "after": {"id": "p_phone", "c": 2, "d": 2, "q": 21}}
```

Key behaviors:
- First occurrence shows `before: null` (new aggregate)
- Updates show both before and after states
- Deletes might decrease counts
- Perfect for maintaining materialized views from CDC sources