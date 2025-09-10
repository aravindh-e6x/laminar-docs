---
sidebar_position: 4
title: Apache Iceberg
---

# Apache Iceberg Integration Tutorial

Apache Iceberg is an open table format for huge analytic datasets. This tutorial shows how to build streaming data lakehouse pipelines with Laminar and Iceberg.

## What is Apache Iceberg?

Apache Iceberg provides:
- **ACID transactions** on data lakes
- **Time travel** and versioning
- **Schema evolution** without rewriting data
- **Partition evolution** for better performance
- **Hidden partitioning** for easier queries
- **Streaming and batch** unified processing

## Prerequisites

- Access to a Laminar workspace
- Your Laminar API credentials
- Access to:
  - Object storage (S3, GCS, or Azure)
  - Iceberg catalog (AWS Glue, Hive Metastore, or REST)
- Basic understanding of data lakehouse concepts

## Set Up Your Environment

Configure your Laminar API access:

```bash
# Set your workspace URL and API key
export LAMINAR_API_URL="https://your-workspace.laminar.cloud/api/v1"
export LAMINAR_API_KEY="your-api-key-here"

# Set your cloud storage credentials (example for AWS)
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_REGION=us-east-1
```

## Create Iceberg Connection Profile

### S3 + AWS Glue Catalog

For AWS S3 with Glue Catalog:

```bash
curl -X POST $LAMINAR_API_URL/connection_profiles \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iceberg-s3-glue",
    "connector": "iceberg",
    "description": "Iceberg with S3 and Glue Catalog",
    "config": {
      "catalog_type": "glue",
      "warehouse": "s3://my-data-lake/warehouse",
      "region": "us-east-1",
      "database": "analytics",
      "io_impl": "org.apache.iceberg.aws.s3.S3FileIO",
      "s3_endpoint": "https://s3.amazonaws.com",
      "s3_access_key_id": "${AWS_ACCESS_KEY_ID}",
      "s3_secret_access_key": "${AWS_SECRET_ACCESS_KEY}"
    }
  }'
```

### REST Catalog with S3

For REST catalog with S3-compatible storage:

```bash
curl -X POST $LAMINAR_API_URL/connection_profiles \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iceberg-rest",
    "connector": "iceberg",
    "description": "Iceberg with REST Catalog",
    "config": {
      "catalog_type": "rest",
      "uri": "https://iceberg-catalog.example.com",
      "warehouse": "s3://my-data-lake/warehouse/",
      "io_impl": "org.apache.iceberg.aws.s3.S3FileIO",
      "s3_endpoint": "https://s3.amazonaws.com",
      "s3_access_key_id": "${AWS_ACCESS_KEY_ID}",
      "s3_secret_access_key": "${AWS_SECRET_ACCESS_KEY}",
      "s3_path_style_access": false
    }
  }'
```

## Create Iceberg Tables

### Events Table (Sink)

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "events_iceberg",
    "connector": "iceberg",
    "connection_profile_id": "iceberg-s3-glue",
    "table_type": "sink",
    "schema": {
      "fields": [
        {"name": "event_id", "data_type": "STRING", "nullable": false},
        {"name": "user_id", "data_type": "BIGINT", "nullable": false},
        {"name": "event_type", "data_type": "STRING", "nullable": false},
        {"name": "product_id", "data_type": "STRING", "nullable": true},
        {"name": "quantity", "data_type": "INT", "nullable": true},
        {"name": "price", "data_type": "DOUBLE", "nullable": true},
        {"name": "event_time", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "processing_time", "data_type": "TIMESTAMP", "nullable": false}
      ]
    },
    "config": {
      "catalog": "iceberg",
      "database": "analytics",
      "table": "events",
      "write_mode": "append",
      "partition_spec": [
        {
          "field": "event_time",
          "transform": "day"
        },
        {
          "field": "event_type",
          "transform": "identity"
        }
      ],
      "table_properties": {
        "write.format.default": "parquet",
        "write.parquet.compression": "snappy",
        "commit.retry.num-retries": "3",
        "write.metadata.delete-after-commit.enabled": "true",
        "write.metadata.previous-versions-max": "10"
      }
    }
  }'
```

### Aggregates Table (Sink)

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user_aggregates_iceberg",
    "connector": "iceberg",
    "connection_profile_id": "iceberg-s3-glue",
    "table_type": "sink",
    "schema": {
      "fields": [
        {"name": "user_id", "data_type": "BIGINT", "nullable": false},
        {"name": "date", "data_type": "DATE", "nullable": false},
        {"name": "total_events", "data_type": "BIGINT", "nullable": false},
        {"name": "total_purchases", "data_type": "BIGINT", "nullable": false},
        {"name": "total_revenue", "data_type": "DOUBLE", "nullable": false},
        {"name": "unique_products", "data_type": "BIGINT", "nullable": false},
        {"name": "last_event_time", "data_type": "TIMESTAMP", "nullable": false}
      ]
    },
    "config": {
      "catalog": "iceberg",
      "database": "analytics",
      "table": "user_daily_aggregates",
      "write_mode": "upsert",
      "primary_keys": ["user_id", "date"],
      "partition_spec": [
        {
          "field": "date",
          "transform": "identity"
        }
      ]
    }
  }'
```

## Streaming to Iceberg Pipelines

### Real-time Event Streaming

Stream events from Kafka to Iceberg:

```sql
-- Stream events to Iceberg with exactly-once semantics
INSERT INTO events_iceberg
SELECT 
    event_id,
    user_id,
    event_type,
    product_id,
    quantity,
    price,
    event_time,
    CURRENT_TIMESTAMP as processing_time
FROM kafka_events_source
WHERE event_type IN ('view', 'purchase', 'cart_add', 'cart_remove')
```

Deploy the pipeline:

```bash
curl -X POST $LAMINAR_API_URL/pipelines \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "kafka_to_iceberg_pipeline",
    "query": "INSERT INTO events_iceberg SELECT event_id, user_id, event_type, product_id, quantity, price, event_time, CURRENT_TIMESTAMP as processing_time FROM kafka_events_source WHERE event_type IN ('\''view'\'', '\''purchase'\'', '\''cart_add'\'', '\''cart_remove'\'')",
    "parallelism": 4,
    "checkpoint_interval_micros": 60000000
  }'
```

### Incremental Aggregation Pipeline

Build incremental aggregates:

```sql
-- Daily user aggregates with incremental updates
INSERT INTO user_aggregates_iceberg
SELECT 
    user_id,
    DATE(event_time) as date,
    COUNT(*) as total_events,
    COUNT(CASE WHEN event_type = 'purchase' THEN 1 END) as total_purchases,
    SUM(CASE WHEN event_type = 'purchase' THEN price * quantity ELSE 0 END) as total_revenue,
    COUNT(DISTINCT product_id) as unique_products,
    MAX(event_time) as last_event_time
FROM kafka_events_source
GROUP BY 
    user_id,
    DATE(event_time)
```

### CDC to Iceberg Pipeline

Sync database changes to Iceberg:

```sql
-- Sync customer data from MySQL CDC to Iceberg
INSERT INTO customers_iceberg
SELECT 
    id,
    email,
    name,
    status,
    created_at,
    updated_at,
    _ts as cdc_timestamp,
    _op as cdc_operation,
    DATE(_ts) as partition_date
FROM mysql_customers_cdc
WHERE _op IN ('c', 'u')  -- Only inserts and updates
```

## Advanced Iceberg Features

### Time Travel Queries

Query historical data:

```sql
-- Query data as of specific timestamp
SELECT * FROM events_iceberg 
FOR SYSTEM_TIME AS OF TIMESTAMP '2024-01-01 00:00:00'
WHERE user_id = 12345;

-- Query data from specific snapshot
SELECT * FROM events_iceberg 
FOR VERSION AS OF 1234567890
WHERE event_type = 'purchase';
```

### Merge Into (Upserts)

Perform upserts with MERGE:

```sql
-- Merge new data into existing table
MERGE INTO user_profiles_iceberg AS target
USING user_updates_stream AS source
ON target.user_id = source.user_id
WHEN MATCHED AND source.updated_at > target.updated_at THEN
    UPDATE SET 
        email = source.email,
        name = source.name,
        status = source.status,
        updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT (user_id, email, name, status, created_at, updated_at)
    VALUES (source.user_id, source.email, source.name, 
            source.status, source.created_at, source.updated_at);
```

### Compaction Pipeline

Optimize file layout:

```sql
-- Compact small files periodically
CALL system.rewrite_data_files(
    table => 'analytics.events_iceberg',
    strategy => 'binpack',
    options => map(
        'target-file-size-bytes', '134217728',  -- 128MB
        'min-files-per-group', '5'
    )
);
```

### Schema Evolution

Add new columns dynamically:

```sql
-- Add new column to Iceberg table
ALTER TABLE events_iceberg 
ADD COLUMNS (
    session_id STRING COMMENT 'User session identifier',
    device_type STRING COMMENT 'Device type: mobile, desktop, tablet'
);

-- Pipeline automatically handles new schema
INSERT INTO events_iceberg
SELECT 
    event_id,
    user_id,
    event_type,
    product_id,
    quantity,
    price,
    event_time,
    CURRENT_TIMESTAMP as processing_time,
    session_id,  -- New field
    device_type  -- New field
FROM enhanced_events_source;
```

## Iceberg Table Maintenance

### Expire Snapshots

```sql
-- Remove old snapshots
CALL system.expire_snapshots(
    table => 'analytics.events_iceberg',
    older_than => TIMESTAMP '2024-01-01 00:00:00',
    retain_last => 10
);
```

### Remove Orphan Files

```sql
-- Clean up orphan files
CALL system.remove_orphan_files(
    table => 'analytics.events_iceberg',
    older_than => TIMESTAMP '2024-01-01 00:00:00',
    dry_run => false
);
```

### Rewrite Manifests

```sql
-- Optimize manifest files
CALL system.rewrite_manifests(
    table => 'analytics.events_iceberg',
    use_caching => true
);
```

## Monitoring Iceberg Pipelines

### Table Statistics

```sql
-- Get table statistics
SELECT 
    snapshot_id,
    committed_at,
    summary['total-records'] as total_records,
    summary['total-files'] as total_files,
    summary['total-data-files-size'] as data_size_bytes
FROM analytics.events_iceberg.snapshots
ORDER BY committed_at DESC
LIMIT 10;
```

### File Statistics

```sql
-- Analyze file distribution
SELECT 
    file_path,
    file_format,
    record_count,
    file_size_in_bytes,
    column_sizes,
    value_counts,
    null_value_counts
FROM analytics.events_iceberg.files
WHERE snapshot_id = (SELECT MAX(snapshot_id) FROM analytics.events_iceberg.snapshots);
```

### Pipeline Metrics

Monitor your Iceberg pipelines through Laminar:

```bash
# Get Iceberg-specific metrics
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/kafka_to_iceberg_pipeline/jobs/{job_id}/metrics?filter=iceberg

# Monitor table statistics
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/connection_tables/events_iceberg/stats

# Key metrics to monitor:
# - records_written
# - bytes_written  
# - files_created
# - snapshots_committed
# - compaction_time
```

## Performance Optimization

### Write Configuration

```json
{
  "config": {
    "write_batch_size": 10000,
    "write_parallelism": 4,
    "file_size_bytes": 134217728,
    "compression": "snappy",
    "write_mode": "append",
    "commit_interval": "60s"
  }
}
```

### Partitioning Strategy

```json
{
  "partition_spec": [
    {
      "field": "event_time",
      "transform": "hour"  // hour, day, month, year
    },
    {
      "field": "user_id",
      "transform": "bucket[100]"  // Hash buckets
    },
    {
      "field": "region",
      "transform": "identity"  // Direct partitioning
    }
  ]
}
```

### Read Optimization

```sql
-- Use partition pruning
SELECT * FROM events_iceberg
WHERE event_time >= CURRENT_DATE - INTERVAL '7' DAY
  AND event_type = 'purchase';

-- Use projection pushdown
SELECT user_id, event_type, price
FROM events_iceberg
WHERE date = '2024-01-01';

-- Use predicate pushdown
SELECT * FROM events_iceberg
WHERE user_id IN (SELECT user_id FROM premium_users);
```

## Integration with Analytics Tools

### Query Through Laminar

Laminar provides direct SQL querying of Iceberg tables:

```bash
# Query Iceberg tables through Laminar
curl -X POST $LAMINAR_API_URL/sql/query \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT date, COUNT(DISTINCT user_id) as dau, SUM(total_revenue) as revenue FROM user_aggregates_iceberg WHERE date >= CURRENT_DATE - 30 GROUP BY date",
    "catalog": "iceberg"
  }'
```

### External Tool Integration

Connect analytics tools using Laminar's Iceberg endpoint:

```python
# Python example using Laminar's Iceberg connector
from laminar import IcebergClient

client = IcebergClient(
    api_url=os.environ['LAMINAR_API_URL'],
    api_key=os.environ['LAMINAR_API_KEY']
)

# Query Iceberg table
df = spark.sql("""
    SELECT 
        date,
        COUNT(DISTINCT user_id) as daily_active_users,
        SUM(total_revenue) as daily_revenue
    FROM iceberg.analytics.user_daily_aggregates
    WHERE date >= current_date - 30
    GROUP BY date
    ORDER BY date
""")

df.show()
```

You can also connect BI tools directly to your Iceberg tables through Laminar's JDBC endpoint:

```sql
-- Connection string for BI tools
jdbc:laminar://your-workspace.laminar.cloud/iceberg?catalog=analytics

-- Query from Trino
SELECT 
    date_trunc('hour', event_time) as hour,
    COUNT(*) as events,
    COUNT(DISTINCT user_id) as unique_users
FROM iceberg.analytics.events
WHERE event_time >= current_timestamp - interval '24' hour
GROUP BY 1
ORDER BY 1;
```

## Monitoring and Troubleshooting

### Iceberg Table Management

Laminar provides built-in Iceberg table management:

```bash
# List all Iceberg tables
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/iceberg/tables

# Get table metadata
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/iceberg/tables/events_iceberg/metadata

# View table history
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/iceberg/tables/events_iceberg/history

# Run table maintenance
curl -X POST $LAMINAR_API_URL/iceberg/tables/events_iceberg/maintenance \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "compact",
    "options": {
      "target_file_size_mb": 128
    }
  }'
```

## Clean Up

```bash
# Stop pipeline
curl -X PATCH $LAMINAR_API_URL/pipelines/kafka_to_iceberg_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"stop": "immediate"}'

# Delete pipeline
curl -X DELETE $LAMINAR_API_URL/pipelines/kafka_to_iceberg_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY"

# Delete connection tables
curl -X DELETE $LAMINAR_API_URL/connection_tables/events_iceberg \
  -H "Authorization: Bearer $LAMINAR_API_KEY"
  
curl -X DELETE $LAMINAR_API_URL/connection_tables/user_aggregates_iceberg \
  -H "Authorization: Bearer $LAMINAR_API_KEY"

# Delete connection profile
curl -X DELETE $LAMINAR_API_URL/connection_profiles/iceberg-s3-glue \
  -H "Authorization: Bearer $LAMINAR_API_KEY"
```

## Next Steps

- Review [Iceberg Sink Configuration](../../connectors/sinks/iceberg) - Detailed configuration options
- Explore [Advanced SQL Features](../../sql/intro) - Window functions and complex queries
- Monitor with [Observability Tools](../../observability/intro) - Track Iceberg pipeline performance
- Learn about [Kafka Integration](./kafka) - Stream data into Iceberg tables