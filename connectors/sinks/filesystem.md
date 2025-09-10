---
sidebar_position: 9
---

# Filesystem Sink

The Filesystem sink connector writes processed data to files in local or network-mounted filesystems, supporting various formats and partitioning strategies.

## Overview

The Filesystem sink creates files containing processed data, with support for different file formats, compression, and directory partitioning schemes.

## Configuration

### Directory Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `path` | string | Yes | Base directory path |
| `filename_pattern` | string | No | File naming pattern |
| `partition_by` | array | No | Partitioning columns |
| `file_format` | string | Yes | Output format (json, csv, parquet, avro) |

### File Management

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `max_file_size` | string | No | Maximum file size (e.g., "100MB") |
| `max_records_per_file` | integer | No | Max records per file |
| `compression` | string | No | Compression type |
| `rollover_interval_ms` | integer | No | File rollover interval |

## SQL Example

```sql
CREATE CONNECTION fs_sink_conn FROM filesystem_sink
WITH (
    path = '/data/output',
    file_format = 'parquet',
    compression = 'snappy'
);

INSERT INTO filesystem_output
SELECT 
    date,
    region,
    product,
    sales_amount
FROM sales_data
WITH (
    connector = 'filesystem_sink',
    connection = 'fs_sink_conn',
    partition_by = '["date", "region"]',
    filename_pattern = 'sales_${date}_${region}.parquet'
);
```

## File Formats

### JSON Files

```sql
INSERT INTO json_files
SELECT * FROM events
WITH (
    connector = 'filesystem_sink',
    path = '/data/json',
    file_format = 'json',
    json_lines = true
);
```

### CSV Files

```sql
INSERT INTO csv_files
SELECT * FROM tabular_data
WITH (
    connector = 'filesystem_sink',
    path = '/data/csv',
    file_format = 'csv',
    csv_header = true,
    csv_delimiter = ','
);
```

### Parquet Files

```sql
INSERT INTO parquet_files
SELECT * FROM analytics_data
WITH (
    connector = 'filesystem_sink',
    path = '/data/parquet',
    file_format = 'parquet',
    compression = 'snappy',
    parquet_row_group_size = 100000
);
```

## Partitioning Strategies

### Time-based Partitioning

```sql
INSERT INTO time_partitioned
SELECT * FROM events
WITH (
    connector = 'filesystem_sink',
    path = '/data/events',
    partition_by = '["year", "month", "day"]',
    partition_extractor = 'timestamp'
);
```

### Custom Partitioning

```sql
INSERT INTO custom_partitioned
SELECT 
    EXTRACT(YEAR FROM timestamp) as year,
    EXTRACT(MONTH FROM timestamp) as month,
    region,
    *
FROM events
WITH (
    connector = 'filesystem_sink',
    path = '/data/partitioned',
    partition_by = '["year", "month", "region"]'
);
```

## Best Practices

1. **File Size**: Balance between file size and number of files
2. **Compression**: Use appropriate compression for your use case
3. **Partitioning**: Partition by commonly queried columns
4. **File Formats**: Choose format based on downstream requirements
5. **Cleanup**: Implement file retention policies