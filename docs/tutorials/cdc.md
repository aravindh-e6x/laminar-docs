---
sidebar_position: 3
title: Change Data Capture (CDC)
---

# Processing Debezium CDC Events

This tutorial demonstrates how to process Change Data Capture (CDC) events in Debezium format from Kafka using Laminar. Debezium is a popular open-source platform that captures database changes and streams them as events.

## What is Debezium?

Debezium captures row-level changes from databases and publishes them to Kafka in a standardized format. This approach provides:

- **Decoupled architecture** - Database changes flow through Kafka
- **Standardized format** - Consistent event structure across different databases
- **Real-time processing** - Stream changes as they happen
- **Event sourcing** - Build event-driven architectures

## Prerequisites

- Access to a Laminar workspace
- Your Laminar API credentials
- Kafka cluster with Debezium CDC events
- Understanding of:
  - Kafka topics and consumer groups
  - Debezium event structure
  - Basic CDC concepts

## Set Up Your Environment

Configure your Laminar API access:

```bash
# Set your workspace URL and API key
export LAMINAR_API_URL="https://your-workspace.laminar.cloud/api/v1"
export LAMINAR_API_KEY="your-api-key-here"
```

## Understanding Debezium Event Format

Debezium events have a standard structure:

```json
{
  "before": {...},        // Row state before change (null for inserts)
  "after": {...},         // Row state after change (null for deletes)
  "source": {
    "version": "1.9.0",
    "connector": "mysql",
    "name": "dbserver1",
    "ts_ms": 1640001234567,
    "db": "ecommerce",
    "table": "customers",
    "server_id": 1,
    "gtid": "...",
    "file": "mysql-bin.000003",
    "pos": 154
  },
  "op": "u",             // Operation: c=create, u=update, d=delete, r=read
  "ts_ms": 1640001234568  // Event timestamp
}
```

## Typical Debezium Topics

Debezium creates Kafka topics following a naming convention:

```
<server.name>.<database>.<table>
```

For example:
- `dbserver1.ecommerce.customers` - Customer table changes
- `dbserver1.ecommerce.orders` - Order table changes
- `dbserver1.inventory.products` - Product table changes

Each topic contains CDC events for a specific table, with all changes (inserts, updates, deletes) flowing through the same topic.

## Configure Kafka Connection for CDC

### Create Kafka Connection Profile

```bash
curl -X POST $LAMINAR_API_URL/connection_profiles \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "kafka-cdc-source",
    "connector": "kafka",
    "description": "Kafka cluster with Debezium CDC events",
    "config": {
      "bootstrap_servers": "kafka1.example.com:9092,kafka2.example.com:9092",
      "security_protocol": "SASL_SSL",
      "sasl_mechanism": "PLAIN",
      "sasl_username": "your-username",
      "sasl_password": "your-password"
    }
  }'
```

### Create Debezium Source Tables

#### Customers CDC Events

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "customers_cdc",
    "connector": "kafka",
    "connection_profile_id": "kafka-cdc-source",
    "table_type": "source",
    "schema": {
      "fields": [
        {"name": "id", "data_type": "INT", "nullable": false},
        {"name": "email", "data_type": "STRING", "nullable": false},
        {"name": "name", "data_type": "STRING", "nullable": false},
        {"name": "status", "data_type": "STRING", "nullable": true},
        {"name": "created_at", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "updated_at", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "_op", "data_type": "STRING", "nullable": false},
        {"name": "_ts", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "_deleted", "data_type": "BOOLEAN", "nullable": false}
      ]
    },
    "config": {
      "topic": "dbserver1.ecommerce.customers",
      "format": "debezium-json",
      "consumer_group": "laminar-cdc-processor",
      "offset_reset": "earliest",
      "debezium": {
        "schema_handling": "extract_after",
        "op_field": "_op",
        "timestamp_field": "_ts",
        "deleted_field": "_deleted"
      }
    },
    "event_time_field": "_ts",
    "watermark_delay": "5 seconds"
  }'
```

#### Orders CDC Events

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "orders_cdc",
    "connector": "kafka",
    "connection_profile_id": "kafka-cdc-source",
    "table_type": "source",
    "schema": {
      "fields": [
        {"name": "id", "data_type": "INT", "nullable": false},
        {"name": "customer_id", "data_type": "INT", "nullable": false},
        {"name": "order_status", "data_type": "STRING", "nullable": true},
        {"name": "total_amount", "data_type": "DECIMAL(10,2)", "nullable": true},
        {"name": "created_at", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "updated_at", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "_op", "data_type": "STRING", "nullable": false},
        {"name": "_ts", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "_deleted", "data_type": "BOOLEAN", "nullable": false}
      ]
    },
    "config": {
      "topic": "dbserver1.ecommerce.orders",
      "format": "debezium-json",
      "consumer_group": "laminar-cdc-processor",
      "offset_reset": "earliest",
      "debezium": {
        "schema_handling": "extract_after",
        "handle_deletes": true
      }
    }
  }'
```

## Processing Debezium Events

### Real-time Data Synchronization

Process Debezium events and sync to a data warehouse:

```sql
-- Sync customer changes from Debezium events
INSERT INTO customers_warehouse
SELECT 
    id,
    email,
    name,
    status,
    created_at,
    updated_at,
    CASE _op 
        WHEN 'c' THEN 'INSERT'
        WHEN 'u' THEN 'UPDATE'
        WHEN 'd' THEN 'DELETE'
        WHEN 'r' THEN 'SNAPSHOT'
    END as operation_type,
    _ts as sync_timestamp
FROM customers_cdc
WHERE NOT _deleted  -- Exclude deleted records
```

Deploy the pipeline:

```bash
curl -X POST $LAMINAR_API_URL/pipelines \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "debezium_sync_pipeline",
    "query": "INSERT INTO customers_warehouse SELECT id, email, name, status, created_at, updated_at, CASE _op WHEN '\''c'\'' THEN '\''INSERT'\'' WHEN '\''u'\'' THEN '\''UPDATE'\'' WHEN '\''d'\'' THEN '\''DELETE'\'' WHEN '\''r'\'' THEN '\''SNAPSHOT'\'' END as operation_type, _ts as sync_timestamp FROM customers_cdc WHERE NOT _deleted",
    "parallelism": 4,
    "checkpoint_interval": "30s"
  }'
```

### Event-Driven Processing

Trigger actions based on database changes:

```sql
-- Send notifications for high-value orders
INSERT INTO order_notifications
SELECT 
    o.id as order_id,
    o.customer_id,
    c.email,
    c.name as customer_name,
    o.total_amount,
    o.order_status,
    'High value order placed' as notification_type,
    o._ts as event_time
FROM orders_cdc o
JOIN customers_cdc c ON o.customer_id = c.id
WHERE o._op = 'c'  -- New orders only
  AND o.total_amount > 500
```

### Change Data Analytics

Analyze patterns in data changes:

```sql
-- Track customer status changes
INSERT INTO customer_status_changes
SELECT 
    id as customer_id,
    name,
    LAG(status) OVER (PARTITION BY id ORDER BY _ts) as previous_status,
    status as current_status,
    _ts as change_time,
    CASE 
        WHEN LAG(status) OVER (PARTITION BY id ORDER BY _ts) = 'active' 
         AND status = 'inactive' THEN 'churn'
        WHEN LAG(status) OVER (PARTITION BY id ORDER BY _ts) = 'inactive' 
         AND status = 'active' THEN 'reactivation'
        ELSE 'other'
    END as change_type
FROM customers_cdc
WHERE _op = 'u'  -- Updates only
```

### CDC with Deduplication

Handle duplicate changes:

```sql
-- Deduplicate CDC events by keeping latest change per record
INSERT INTO customers_latest
SELECT 
    id,
    email,
    name,
    status,
    created_at,
    updated_at
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY _ts DESC) as rn
    FROM customers_cdc
    WHERE _op != 'd'  -- Exclude deletes
) WHERE rn = 1
```

## Advanced CDC Patterns

### Snapshot and Incremental Sync

```sql
-- Initial snapshot + incremental changes
CREATE VIEW customers_current AS
SELECT * FROM (
    -- Latest state from CDC stream
    SELECT 
        id, email, name, status, created_at, updated_at,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY _ts DESC) as rn
    FROM customers_cdc
) WHERE rn = 1 AND _op != 'd';

-- Use in downstream processing
INSERT INTO customer_aggregates
SELECT 
    status,
    COUNT(*) as customer_count,
    MAX(updated_at) as last_update
FROM customers_current
GROUP BY status
```

### Handling Deletes

```sql
-- Track deleted records
INSERT INTO deleted_records_audit
SELECT 
    'customers' as table_name,
    id as record_id,
    email,
    name,
    _ts as deleted_at,
    CURRENT_USER() as deleted_by
FROM customers_cdc
WHERE _op = 'd';

-- Soft delete in target
UPDATE customers_replica
SET 
    is_deleted = true,
    deleted_at = _ts
FROM customers_cdc
WHERE customers_replica.id = customers_cdc.id
  AND customers_cdc._op = 'd';
```

### CDC to Multiple Targets

```sql
-- Route changes to different systems
-- Kafka for real-time consumers
INSERT INTO customers_kafka_sink
SELECT * FROM customers_cdc
WHERE _op IN ('c', 'u');

-- Elasticsearch for search
INSERT INTO customers_elastic_sink
SELECT 
    id,
    email,
    name,
    status,
    to_json(struct(created_at, updated_at)) as metadata
FROM customers_cdc
WHERE _op != 'd';

-- S3 for archival
INSERT INTO customers_s3_archive
SELECT 
    id,
    email,
    name,
    status,
    _op,
    _ts,
    DATE_FORMAT(_ts, 'yyyy/MM/dd') as partition_date
FROM customers_cdc;
```

## Monitoring CDC Pipelines

### Monitor Through Laminar Console

Laminar provides comprehensive CDC monitoring:

```bash
# Check pipeline health and CDC metrics
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/customer_sync_pipeline/cdc-status

# Get detailed CDC metrics
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/customer_sync_pipeline/jobs/{job_id}/metrics?filter=cdc

# Monitor replication lag
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/customer_sync_pipeline/replication-lag
```

Key metrics to monitor:
- **binlog_position**: Current position in MySQL binlog
- **records_behind_latest**: Number of records behind real-time
- **replication_lag_seconds**: Time delay from source database
- **snapshot_progress**: Progress of initial snapshot (if applicable)

### Verify Data Consistency

Laminar provides built-in data validation:

```bash
# Run consistency check
curl -X POST $LAMINAR_API_URL/pipelines/customer_sync_pipeline/validate \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "validation_type": "row_count",
    "source_table": "customers_cdc",
    "target_table": "customers_warehouse"
  }'

# Get validation report
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/customer_sync_pipeline/validation-report
```

## Performance Optimization

### Kafka Consumer Configuration

Optimize Kafka consumption for Debezium events:

```json
{
  "config": {
    "max.poll.records": "1000",
    "fetch.min.bytes": "1048576",
    "fetch.max.wait.ms": "500",
    "session.timeout.ms": "30000",
    "max.partition.fetch.bytes": "10485760"
  }
}
```

### Processing Parallelism

Match parallelism to Kafka partitions:

```json
{
  "pipeline": {
    "parallelism": 8,
    "checkpoint_interval": "30s",
    "max_idle_time": "60s"
  }
}
```

### Memory Management

```yaml
pipeline:
  resources:
    memory: 2Gi
  config:
    state_backend_memory: 512Mi
    network_buffer_memory: 256Mi
```

## Troubleshooting Debezium Processing

### Diagnostic Tools

Monitor and debug Debezium event processing:

```bash
# Test Kafka connection
curl -X POST $LAMINAR_API_URL/connection_profiles/kafka-cdc-source/test \
  -H "Authorization: Bearer $LAMINAR_API_KEY"

# Check consumer group status
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/debezium_sync_pipeline/consumer-status

# View processing logs
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/debezium_sync_pipeline/logs?filter=debezium&lines=100
```


## Testing CDC Pipeline

### Generate Test Changes

```sql
-- Insert new records
INSERT INTO customers (email, name, status) 
VALUES ('test@example.com', 'Test User', 'active');

-- Update existing records
UPDATE customers 
SET status = 'premium' 
WHERE email = 'john@example.com';

-- Delete records
DELETE FROM customers 
WHERE email = 'test@example.com';

-- Bulk updates
UPDATE orders 
SET order_status = 'shipped' 
WHERE order_status = 'processing';
```

### Verify Changes Are Captured

```bash
# Monitor real-time changes
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/customer_sync_pipeline/preview?limit=10

# Get change statistics
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/customer_sync_pipeline/cdc-stats
```

## Clean Up

```bash
# Stop pipeline
curl -X PATCH $LAMINAR_API_URL/pipelines/customer_sync_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -d '{"stop": "immediate"}'

# Delete pipeline
curl -X DELETE http://localhost:5115/api/v1/pipelines/customer_sync_pipeline

# Delete connection tables
curl -X DELETE http://localhost:5115/api/v1/connection_tables/customers_cdc
curl -X DELETE http://localhost:5115/api/v1/connection_tables/orders_cdc

# Delete connection profile
curl -X DELETE http://localhost:5115/api/v1/connection_profiles/mysql-cdc-source
```

## Next Steps

- Explore [Apache Iceberg Integration](./iceberg) - Build lakehouse architectures
- Learn about [Advanced SQL](../../sql/intro) - Complex transformations
- Review [MySQL CDC Connector](../../connectors/sources/mysql-cdc) - Detailed configuration
- Monitor with [Observability Tools](../../observability/intro) - Track CDC performance