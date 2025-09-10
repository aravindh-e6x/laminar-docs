---
sidebar_position: 2
---

# MySQL CDC Connector

The MySQL CDC (Change Data Capture) connector enables real-time streaming of changes from MySQL databases into Laminar pipelines.

## Overview

The MySQL CDC connector uses the MySQL binary log (binlog) to capture row-level changes in your MySQL database. It supports:

- Initial snapshot of existing data
- Real-time streaming of inserts, updates, and deletes
- Exactly-once semantics with checkpointing
- Schema evolution handling

## Prerequisites

### MySQL Configuration

Your MySQL server must have binary logging enabled:

```sql
-- Check if binlog is enabled
SHOW VARIABLES LIKE 'log_bin';

-- Required MySQL configuration (my.cnf)
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
expire_logs_days = 7
```

### Required Permissions

The MySQL user needs the following permissions:

```sql
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT 
ON *.* TO 'laminar_user'@'%';
```

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `hostname` | string | Yes | MySQL server hostname |
| `port` | integer | No | MySQL server port (default: 3306) |
| `username` | string | Yes | MySQL username |
| `password` | string | Yes | MySQL password |
| `database` | string | Yes | Database to monitor |
| `tables` | array | No | Specific tables to monitor (default: all) |
| `server_id` | integer | No | Unique server ID (default: auto-generated) |

### Snapshot Configuration

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `snapshot.mode` | string | `initial` | Snapshot mode: `initial`, `never`, `schema_only` |
| `snapshot.isolation` | string | `REPEATABLE_READ` | Transaction isolation level |
| `snapshot.fetch_size` | integer | 10000 | Rows per fetch |
| `snapshot.max_threads` | integer | 4 | Parallel snapshot threads |

## Usage Example

### Creating a MySQL CDC Connection

```sql
CREATE CONNECTION mysql_source TO mysql_cdc (
    hostname = 'mysql.example.com',
    port = 3306,
    username = 'laminar_user',
    password = '${secret:mysql_password}',
    database = 'production',
    tables = ['orders', 'customers'],
    snapshot.mode = 'initial',
    snapshot.fetch_size = 20000
);
```

### Using in a Pipeline

```sql
CREATE PIPELINE orders_processing AS
SELECT 
    op,
    after.order_id as order_id,
    after.customer_id as customer_id,
    after.total_amount as amount,
    after.created_at as created_at
FROM mysql_source
WHERE table_name = 'orders'
  AND op IN ('c', 'u'); -- Only inserts and updates
```

## Output Schema

The MySQL CDC connector outputs events in Debezium format:

```json
{
  "before": { /* Row state before change */ },
  "after": { /* Row state after change */ },
  "op": "c", // Operation: c=create, u=update, d=delete, r=read
  "source": {
    "database": "production",
    "table": "orders",
    "timestamp": 1234567890,
    "binlog_file": "mysql-bin.000003",
    "binlog_position": 154
  }
}
```

## Performance Tuning

### Snapshot Performance

For large tables, optimize snapshot performance:

```sql
-- Use cursor-based fetching for very large tables
snapshot.use_cursor = true,
snapshot.cursor_fetch_size = 50000,

-- Enable parallel snapshot
snapshot.parallel_workers = 8,

-- Use streaming for medium tables
snapshot.use_streaming = true,
snapshot.stream_buffer_size = 100000
```

### Binlog Reading

Optimize binlog reading:

```sql
-- Buffer size for binlog events
binlog.buffer_size = 65536,

-- Keep-alive for long-running connections
connection.keepalive_enabled = true,
connection.keepalive_interval = 30
```

## Monitoring

Key metrics to monitor:

- **Lag**: Time difference between event creation and processing
- **Snapshot Progress**: Rows processed during initial snapshot
- **Binlog Position**: Current position in the binary log
- **Error Rate**: Failed events or connection issues

## Limitations

- Does not capture DDL changes (schema changes)
- Requires ROW format binlog
- Cannot capture changes before binlog was enabled
- Table must have a primary key for updates/deletes

## Troubleshooting

### Common Issues

1. **Connection timeout during snapshot**
   - Increase `snapshot.lock_timeout`
   - Use smaller `snapshot.fetch_size`

2. **High memory usage**
   - Reduce `snapshot.fetch_size`
   - Lower `snapshot.max_threads`

3. **Binlog retention**
   - Ensure `expire_logs_days` is sufficient
   - Monitor disk space for binlog files

4. **Permission denied errors**
   - Verify all required permissions
   - Check firewall rules