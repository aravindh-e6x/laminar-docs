---
sidebar_position: 3
title: PostgreSQL
---

# PostgreSQL Sink Connector

The PostgreSQL sink connector enables Laminar to write data to PostgreSQL databases with support for various write modes, batch operations, and advanced PostgreSQL features.

## Overview

The PostgreSQL sink provides:
- **Multiple write modes** (INSERT, UPSERT, UPDATE, DELETE)
- **Batch operations** for high throughput
- **Transaction support** with ACID guarantees
- **COPY protocol** for bulk loading
- **Partitioned tables** support
- **Connection pooling** and retry logic
- **JSON/JSONB** data type support
- **Array and composite types**
- **Prepared statements** for performance

## Configuration

### Basic Configuration

```sql
CREATE CONNECTION postgres_sink TO postgresql (
    host = 'localhost',
    port = 5432,
    database = 'analytics',
    username = 'laminar',
    password = '${secret:pg_password}',
    table = 'events',
    type = 'sink'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `host` | string | Yes | - | PostgreSQL server host |
| `port` | integer | No | 5432 | PostgreSQL server port |
| `database` | string | Yes | - | Database name |
| `username` | string | Yes | - | Database username |
| `password` | string | Yes | - | Database password |
| `table` | string | Yes | - | Target table name |
| `schema` | string | No | public | Database schema |
| `write_mode` | string | No | insert | Write mode (insert, upsert, update, delete) |
| `batch_size` | integer | No | 1000 | Batch size for writes |

## Write Modes

### INSERT Mode

```sql
CREATE CONNECTION pg_insert TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'events',
    type = 'sink',
    write_mode = 'insert'
);

-- Insert new records
CREATE PIPELINE insert_events AS
INSERT INTO pg_insert
SELECT 
    event_id,
    user_id,
    event_type,
    timestamp,
    properties
FROM source_events;
```

### UPSERT Mode (INSERT ON CONFLICT)

```sql
CREATE CONNECTION pg_upsert TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'users',
    type = 'sink',
    write_mode = 'upsert',
    conflict_columns = 'user_id',
    update_columns = 'last_seen,activity_count'
);

-- Upsert user activity
CREATE PIPELINE upsert_users AS
INSERT INTO pg_upsert
SELECT 
    user_id,
    username,
    NOW() as last_seen,
    activity_count + 1 as activity_count
FROM user_activity;
```

### UPDATE Mode

```sql
CREATE CONNECTION pg_update TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'products',
    type = 'sink',
    write_mode = 'update',
    key_columns = 'product_id'
);

-- Update product prices
CREATE PIPELINE update_prices AS
INSERT INTO pg_update
SELECT 
    product_id,
    new_price as price,
    NOW() as updated_at
FROM price_changes;
```

### DELETE Mode

```sql
CREATE CONNECTION pg_delete TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'expired_sessions',
    type = 'sink',
    write_mode = 'delete',
    key_columns = 'session_id'
);

-- Delete expired sessions
CREATE PIPELINE cleanup_sessions AS
INSERT INTO pg_delete
SELECT session_id
FROM sessions
WHERE expired_at < NOW() - INTERVAL '24 hours';
```

## Bulk Operations

### COPY Protocol

```sql
CREATE CONNECTION pg_copy TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'large_dataset',
    type = 'sink',
    use_copy = true,
    copy_batch_size = 10000
);

-- Bulk load data
CREATE PIPELINE bulk_load AS
INSERT INTO pg_copy
SELECT * FROM source_data;
```

### Batch Inserts

```sql
CREATE CONNECTION pg_batch TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'metrics',
    type = 'sink',
    batch_size = 5000,
    batch_timeout_ms = 1000,
    use_prepared_statements = true
);
```

## Data Type Handling

### JSON/JSONB Columns

```sql
CREATE CONNECTION pg_json TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'json_data',
    type = 'sink'
);

-- Store JSON data
CREATE PIPELINE json_pipeline AS
INSERT INTO pg_json
SELECT 
    id,
    TO_JSON(event_data) as data,
    TO_JSONB(metadata) as metadata
FROM events;
```

### Array Types

```sql
CREATE CONNECTION pg_arrays TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'array_data',
    type = 'sink'
);

-- Store array data
CREATE PIPELINE array_pipeline AS
INSERT INTO pg_arrays
SELECT 
    user_id,
    ARRAY_AGG(tag) as tags,
    ARRAY_AGG(score) as scores
FROM user_data
GROUP BY user_id;
```

### Composite Types

```sql
CREATE CONNECTION pg_composite TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'composite_data',
    type = 'sink'
);

-- Store composite types
CREATE PIPELINE composite_pipeline AS
INSERT INTO pg_composite
SELECT 
    id,
    ROW(street, city, state, zip) as address,
    ROW(lat, lon) as coordinates
FROM locations;
```

## Partitioned Tables

### Time-based Partitioning

```sql
CREATE CONNECTION pg_partitioned TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'events',
    type = 'sink',
    partition_column = 'created_at',
    partition_type = 'range',
    create_partitions = true
);

-- Write to partitioned table
CREATE PIPELINE partitioned_writes AS
INSERT INTO pg_partitioned
SELECT 
    event_id,
    event_type,
    created_at,
    data
FROM events
WHERE created_at >= CURRENT_DATE;
```

### List Partitioning

```sql
CREATE CONNECTION pg_list_partition TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'regional_data',
    type = 'sink',
    partition_column = 'region',
    partition_type = 'list'
);
```

## Transaction Management

### Transactional Writes

```sql
CREATE CONNECTION pg_transactional TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'financial_transactions',
    type = 'sink',
    transaction_size = 100,
    isolation_level = 'READ_COMMITTED'
);

-- Ensure atomic writes
CREATE PIPELINE financial_pipeline AS
INSERT INTO pg_transactional
SELECT 
    transaction_id,
    account_id,
    amount,
    balance,
    timestamp
FROM transactions
WHERE validated = true;
```

### Two-Phase Commit

```sql
CREATE CONNECTION pg_2pc TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'critical_data',
    type = 'sink',
    use_2pc = true,
    prepared_transaction_id = 'laminar_${uuid}'
);
```

## Performance Optimization

### Connection Pooling

```sql
CREATE CONNECTION pg_pooled TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'high_volume',
    type = 'sink',
    connection_pool_size = 20,
    connection_timeout_ms = 5000,
    idle_timeout_ms = 600000
);
```

### Prepared Statements

```sql
CREATE CONNECTION pg_prepared TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'metrics',
    type = 'sink',
    use_prepared_statements = true,
    statement_cache_size = 100
);
```

### Parallel Writes

```sql
CREATE CONNECTION pg_parallel TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'large_table',
    type = 'sink',
    parallelism = 10,
    partition_by = 'hash(id)'
);
```

## Advanced Features

### Triggers and Rules

```sql
CREATE CONNECTION pg_triggers TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'audit_events',
    type = 'sink',
    disable_triggers = false,
    fire_triggers = 'ALWAYS'  -- ALWAYS, REPLICA, DISABLED
);
```

### Foreign Data Wrappers

```sql
CREATE CONNECTION pg_fdw TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'foreign_table',
    type = 'sink',
    is_foreign_table = true
);
```

### Logical Replication

```sql
CREATE CONNECTION pg_logical TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'replicated_table',
    type = 'sink',
    replication_slot = 'laminar_slot',
    publication = 'laminar_pub'
);
```

## Error Handling

### Retry Configuration

```sql
CREATE CONNECTION pg_retry TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'events',
    type = 'sink',
    max_retries = 5,
    retry_backoff_ms = 1000,
    retry_on_errors = 'connection_failure,deadlock'
);
```

### Dead Letter Queue

```sql
-- Main sink
CREATE CONNECTION pg_main TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'events',
    type = 'sink'
);

-- Error handler
CREATE PIPELINE pg_with_dlq AS
WITH validated AS (
    SELECT 
        *,
        validate_record(data) as is_valid
    FROM source_data
)
INSERT INTO pg_main
SELECT * FROM validated WHERE is_valid = true
UNION ALL
INSERT INTO error_log
SELECT 
    data,
    'validation_failed' as error_type,
    NOW() as error_time
FROM validated WHERE is_valid = false;
```

## Monitoring

### Write Metrics

```sql
-- Monitor PostgreSQL sink performance
SELECT 
    connection_name,
    rows_written,
    batches_sent,
    avg_batch_size,
    avg_write_latency_ms,
    failed_writes,
    retry_count,
    last_error
FROM system.postgresql_sink_metrics
WHERE connection_name LIKE 'pg_%';
```

### Connection Pool Stats

```sql
-- Monitor connection pool
SELECT 
    connection_name,
    pool_size,
    active_connections,
    idle_connections,
    waiting_requests,
    connection_wait_time_ms
FROM system.postgresql_pool_stats;
```

## Security

### SSL/TLS Connection

```sql
CREATE CONNECTION pg_ssl TO postgresql (
    host = 'secure.example.com',
    port = 5432,
    database = 'analytics',
    username = 'user',
    password = '${secret:password}',
    table = 'secure_data',
    type = 'sink',
    ssl_mode = 'require',  -- disable, allow, prefer, require, verify-ca, verify-full
    ssl_cert = '/certs/client-cert.pem',
    ssl_key = '/certs/client-key.pem',
    ssl_root_cert = '/certs/ca-cert.pem'
);
```

### Row-Level Security

```sql
CREATE CONNECTION pg_rls TO postgresql (
    host = 'localhost',
    database = 'analytics',
    username = 'app_user',
    password = '${secret:password}',
    table = 'user_data',
    type = 'sink',
    set_role = 'data_writer',
    session_variables = MAP {
        'app.current_tenant': 'tenant_123'
    }
);
```

## Best Practices

1. **Use COPY protocol** for bulk loads
2. **Enable connection pooling** for high concurrency
3. **Use prepared statements** for repeated queries
4. **Configure appropriate batch sizes**
5. **Use UPSERT** for idempotent writes
6. **Monitor connection pool usage**
7. **Set reasonable timeouts**
8. **Use SSL** in production
9. **Implement retry logic** for transient failures
10. **Partition large tables** for better performance

## Troubleshooting

### Connection Test

```sql
-- Test PostgreSQL connection
SELECT test_connection('postgres_sink');

-- Check table schema
SELECT * FROM system.postgresql_tables
WHERE connection = 'postgres_sink';
```

### Query Analysis

```sql
-- Analyze slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    max_time,
    rows
FROM system.postgresql_statements
WHERE connection = 'postgres_sink'
ORDER BY mean_time DESC
LIMIT 10;
```