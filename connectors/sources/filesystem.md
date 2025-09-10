---
sidebar_position: 11
---

# Filesystem Source

The Filesystem source connector monitors directories for new files and ingests them as they appear, supporting various file formats and processing patterns.

## Overview

The Filesystem source watches specified directories for new files, processes them according to configured patterns, and tracks processed files to avoid duplicates. It supports local filesystems and network-mounted storage.

## Configuration

### Directory Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `path` | string | Yes | Directory path to monitor |
| `pattern` | string | No | File pattern (glob or regex) |
| `recursive` | boolean | No | Scan subdirectories (default: false) |
| `poll_interval_ms` | integer | No | Scan interval (default: 1000) |

### File Processing

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `format` | string | Yes | File format (json, csv, parquet, avro, text) |
| `compression` | string | No | Compression type (none, gzip, snappy, lz4) |
| `delete_after_read` | boolean | No | Delete files after processing (default: false) |
| `move_after_read` | string | No | Move files to directory after processing |

## SQL Example

```sql
CREATE CONNECTION fs_source FROM filesystem_source
WITH (
    path = '/data/incoming',
    pattern = '*.json',
    recursive = true,
    poll_interval_ms = 5000
);

CREATE TABLE file_data (
    filename TEXT,
    timestamp TIMESTAMP,
    content JSON
) WITH (
    connector = 'filesystem_source',
    connection = 'fs_source',
    format = 'json',
    include_filename = true,
    delete_after_read = true
);
```

## File Formats

### JSON Files

```sql
CREATE TABLE json_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/json',
    pattern = '*.json',
    format = 'json'
);
```

### CSV Files

```sql
CREATE TABLE csv_files (
    id INTEGER,
    name TEXT,
    value FLOAT,
    date DATE
) WITH (
    connector = 'filesystem_source',
    path = '/data/csv',
    pattern = '*.csv',
    format = 'csv',
    csv_header = true,
    csv_delimiter = ',',
    csv_quote = '"'
);
```

### Parquet Files

```sql
CREATE TABLE parquet_files (
    -- Schema automatically inferred from Parquet
) WITH (
    connector = 'filesystem_source',
    path = '/data/parquet',
    pattern = '*.parquet',
    format = 'parquet'
);
```

### Avro Files

```sql
CREATE TABLE avro_files (
    -- Schema from Avro file
) WITH (
    connector = 'filesystem_source',
    path = '/data/avro',
    pattern = '*.avro',
    format = 'avro'
);
```

### Text Files

```sql
CREATE TABLE log_files (
    line TEXT,
    line_number BIGINT,
    filename TEXT
) WITH (
    connector = 'filesystem_source',
    path = '/var/log/app',
    pattern = '*.log',
    format = 'text',
    include_line_number = true,
    include_filename = true
);
```

## Pattern Matching

### Glob Patterns

```sql
CREATE TABLE glob_match (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data',
    pattern = '**/*.{json,jsonl}',
    pattern_type = 'glob'
);
```

### Regex Patterns

```sql
CREATE TABLE regex_match (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data',
    pattern = '^data_\d{8}\.json$',
    pattern_type = 'regex'
);
```

### Multiple Patterns

```sql
CREATE TABLE multi_pattern (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data',
    patterns = '["*.json", "*.jsonl", "data_*.txt"]'
);
```

## Advanced Configuration

### Compressed Files

```sql
CREATE TABLE compressed_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/compressed',
    pattern = '*.json.gz',
    compression = 'gzip'
);
```

### File Metadata

```sql
CREATE TABLE files_with_metadata (
    content JSON,
    filename TEXT,
    filepath TEXT,
    file_size BIGINT,
    modified_time TIMESTAMP,
    created_time TIMESTAMP
) WITH (
    connector = 'filesystem_source',
    path = '/data',
    include_metadata = true
);
```

### Incremental Processing

```sql
CREATE TABLE incremental_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/incremental',
    pattern = '*.json',
    start_position = 'beginning',
    track_progress = true,
    state_file = '/var/lib/laminar/fs_state.json'
);
```

## File Management

### Archive After Processing

```sql
CREATE TABLE archive_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/incoming',
    pattern = '*.json',
    move_after_read = '/data/archive',
    archive_timestamp = true
);
```

### Error Handling

```sql
CREATE TABLE files_with_errors (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/incoming',
    pattern = '*.json',
    error_path = '/data/errors',
    max_retries = 3,
    continue_on_error = true
);
```

### File Locking

```sql
CREATE TABLE locked_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/shared',
    pattern = '*.json',
    use_file_lock = true,
    lock_timeout_ms = 5000
);
```

## Performance Tuning

### Batch Processing

```sql
CREATE TABLE batch_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/batch',
    pattern = '*.json',
    batch_size = 100,
    batch_timeout_ms = 1000
);
```

### Parallel Processing

```sql
CREATE TABLE parallel_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/parallel',
    pattern = '*.json',
    parallelism = 8,
    max_files_per_scan = 1000
);
```

## Monitoring

### Metrics

- `laminar_fs_files_discovered` - Total files discovered
- `laminar_fs_files_processed` - Files successfully processed
- `laminar_fs_files_failed` - Files that failed processing
- `laminar_fs_bytes_read` - Total bytes read
- `laminar_fs_scan_duration_ms` - Directory scan duration

### Health Checks

```sql
SELECT * FROM system.connector_status
WHERE connector_name = 'filesystem_source';
```

## Best Practices

1. **File Patterns**: Use specific patterns to avoid processing unwanted files
2. **Archive Strategy**: Move or delete processed files to prevent reprocessing
3. **Error Handling**: Configure error paths for failed files
4. **State Management**: Enable progress tracking for large directories
5. **File Locking**: Use locks in shared filesystem environments

## Troubleshooting

### File Discovery Issues

```sql
-- Check discovered files
SELECT * FROM system.connector_logs
WHERE connector = 'filesystem_source'
AND message LIKE '%discovered%'
ORDER BY timestamp DESC;
```

### Processing Errors

```sql
-- View file processing errors
SELECT 
    filename,
    error_message,
    retry_count,
    timestamp
FROM system.file_errors
WHERE connector = 'filesystem_source'
ORDER BY timestamp DESC;
```

## Example Use Cases

### Log File Processing

```sql
CREATE TABLE application_logs (
    timestamp TIMESTAMP,
    level TEXT,
    message TEXT,
    context JSON
) WITH (
    connector = 'filesystem_source',
    path = '/var/log/application',
    pattern = '*.log',
    format = 'json_lines',
    delete_after_read = false,
    move_after_read = '/var/log/processed'
);
```

### Data Lake Ingestion

```sql
CREATE TABLE data_lake_files (
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/mnt/data-lake/landing',
    pattern = '**/*.parquet',
    format = 'parquet',
    recursive = true,
    move_after_read = '/mnt/data-lake/processed'
);
```

### CSV Batch Processing

```sql
CREATE TABLE csv_batches (
    batch_id TEXT,
    record_id INTEGER,
    data JSON
) WITH (
    connector = 'filesystem_source',
    path = '/data/csv-batches',
    pattern = 'batch_*.csv',
    format = 'csv',
    csv_header = true,
    batch_id_from_filename = true
);
```