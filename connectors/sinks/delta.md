---
sidebar_position: 10
---

# Delta Lake Sink

The Delta Lake sink connector writes streaming data to Delta Lake tables, providing ACID transactions and time travel capabilities.

## Overview

Delta Lake sink enables writing to Delta tables with support for upserts, deletes, and schema evolution while maintaining data consistency.

## Configuration

### Table Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `table_path` | string | Yes | Delta table location |
| `table_name` | string | No | Table name for catalog |
| `write_mode` | string | No | Write mode (append, overwrite, upsert) |
| `partition_columns` | array | No | Partitioning columns |

### Delta Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `checkpoint_interval` | integer | No | Checkpoint frequency |
| `optimize_write` | boolean | No | Enable write optimization |
| `auto_compact` | boolean | No | Enable auto compaction |
| `schema_evolution` | boolean | No | Allow schema evolution |

## SQL Example

```sql
CREATE CONNECTION delta_conn FROM delta_sink
WITH (
    table_path = 's3://data-lake/sales',
    write_mode = 'append',
    partition_columns = '["year", "month"]',
    optimize_write = true
);

INSERT INTO delta_sales
SELECT 
    order_id,
    customer_id,
    amount,
    EXTRACT(YEAR FROM order_date) as year,
    EXTRACT(MONTH FROM order_date) as month
FROM orders
WITH (
    connector = 'delta_sink',
    connection = 'delta_conn'
);
```

## Write Modes

### Append Mode

```sql
INSERT INTO delta_append
SELECT * FROM new_records
WITH (
    connector = 'delta_sink',
    table_path = '/delta/events',
    write_mode = 'append'
);
```

### Upsert Mode

```sql
INSERT INTO delta_upsert
SELECT * FROM updates
WITH (
    connector = 'delta_sink',
    table_path = '/delta/customers',
    write_mode = 'upsert',
    merge_keys = '["customer_id"]',
    sequence_column = 'updated_at'
);
```

## Schema Evolution

```sql
INSERT INTO delta_evolving
SELECT * FROM evolving_data
WITH (
    connector = 'delta_sink',
    table_path = '/delta/flexible',
    schema_evolution = true,
    merge_schema = true
);
```

## Best Practices

1. **Partitioning**: Choose appropriate partition columns
2. **Compaction**: Schedule regular compaction
3. **Checkpointing**: Set checkpoint intervals based on volume
4. **Z-Ordering**: Optimize for query patterns
5. **Vacuum**: Clean up old files periodically