---
sidebar_position: 11
---

# Iceberg Sink

The Iceberg sink connector writes streaming data to Apache Iceberg tables, providing efficient data lake management with ACID guarantees.

## Overview

Apache Iceberg sink enables writing to Iceberg tables with support for schema evolution, partitioning evolution, and efficient metadata management.

## Configuration

### Table Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `catalog` | string | Yes | Iceberg catalog name |
| `database` | string | Yes | Database/namespace |
| `table` | string | Yes | Table name |
| `catalog_type` | string | Yes | Catalog type (hive, hadoop, glue) |

### Write Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `write_mode` | string | No | Write mode (append, overwrite, upsert) |
| `target_file_size_mb` | integer | No | Target file size |
| `commit_interval_ms` | integer | No | Commit interval |
| `snapshot_properties` | object | No | Custom snapshot properties |

## SQL Example

```sql
CREATE CONNECTION iceberg_conn FROM iceberg_sink
WITH (
    catalog = 'prod_catalog',
    catalog_type = 'glue',
    database = 'analytics',
    table = 'events'
);

INSERT INTO iceberg_events
SELECT 
    event_id,
    event_type,
    timestamp,
    user_id,
    properties
FROM raw_events
WITH (
    connector = 'iceberg_sink',
    connection = 'iceberg_conn',
    write_mode = 'append'
);
```

## Catalog Types

### AWS Glue Catalog

```sql
CREATE CONNECTION glue_iceberg FROM iceberg_sink
WITH (
    catalog_type = 'glue',
    catalog = 'aws_catalog',
    database = 'data_lake',
    table = 'transactions',
    aws_region = 'us-east-1'
);
```

### Hive Metastore

```sql
CREATE CONNECTION hive_iceberg FROM iceberg_sink
WITH (
    catalog_type = 'hive',
    catalog = 'hive_catalog',
    database = 'warehouse',
    table = 'facts',
    hive_metastore_uri = 'thrift://metastore:9083'
);
```

## Write Operations

### Upsert Operations

```sql
INSERT INTO iceberg_upsert
SELECT * FROM changes
WITH (
    connector = 'iceberg_sink',
    connection = 'iceberg_conn',
    write_mode = 'upsert',
    equality_fields = '["id"]',
    sequence_field = 'updated_at'
);
```

### Partitioned Writes

```sql
INSERT INTO iceberg_partitioned
SELECT 
    *,
    DATE(timestamp) as date_partition
FROM events
WITH (
    connector = 'iceberg_sink',
    connection = 'iceberg_conn',
    partition_spec = 'date_partition:day'
);
```

## Best Practices

1. **File Size**: Configure appropriate target file sizes
2. **Commit Frequency**: Balance between latency and efficiency
3. **Compaction**: Schedule regular table maintenance
4. **Metadata**: Use snapshot properties for tracking
5. **Evolution**: Plan for schema and partition evolution