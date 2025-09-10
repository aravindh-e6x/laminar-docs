---
sidebar_position: 4
title: Elasticsearch
---

# Elasticsearch Sink Connector

The Elasticsearch sink connector enables Laminar to index data into Elasticsearch clusters for full-text search, analytics, and visualization with Kibana.

## Overview

The Elasticsearch sink provides:
- **Bulk indexing** for high throughput
- **Index templates** and **mappings** management
- **Document routing** and **sharding**
- **Upsert and update** operations
- **Index lifecycle management** (ILM)
- **Security features** (SSL, authentication)
- **Multiple cluster support**
- **Automatic retries** and error handling

## Configuration

### Basic Configuration

```sql
CREATE CONNECTION elastic_sink TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'events',
    type = 'sink'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `nodes` | string | Yes | - | Elasticsearch nodes (comma-separated) |
| `index` | string | Yes | - | Index name or pattern |
| `type` | string | Yes | - | Must be 'sink' |
| `doc_type` | string | No | _doc | Document type |
| `id_field` | string | No | - | Field to use as document ID |
| `bulk_size` | integer | No | 1000 | Bulk request size |
| `bulk_timeout_ms` | integer | No | 1000 | Bulk flush timeout |
| `refresh` | string | No | false | Refresh policy (true, false, wait_for) |

## Authentication

### No Authentication

```sql
CREATE CONNECTION elastic_public TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'public_data',
    type = 'sink'
);
```

### Basic Authentication

```sql
CREATE CONNECTION elastic_basic TO elasticsearch (
    nodes = 'https://elastic.example.com:9200',
    index = 'secure_data',
    type = 'sink',
    username = 'elastic',
    password = '${secret:elastic_password}'
);
```

### API Key Authentication

```sql
CREATE CONNECTION elastic_apikey TO elasticsearch (
    nodes = 'https://elastic.example.com:9200',
    index = 'events',
    type = 'sink',
    api_key_id = 'key_id',
    api_key = '${secret:elastic_api_key}'
);
```

### Cloud ID (Elastic Cloud)

```sql
CREATE CONNECTION elastic_cloud TO elasticsearch (
    cloud_id = 'deployment:dXMtZWFzdC0xLmF3cy5mb3VuZC5pbyQ...',
    index = 'cloud_data',
    type = 'sink',
    username = 'elastic',
    password = '${secret:cloud_password}'
);
```

## Index Operations

### Simple Indexing

```sql
CREATE CONNECTION elastic_index TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'products',
    type = 'sink',
    id_field = 'product_id'
);

-- Index products
CREATE PIPELINE index_products AS
INSERT INTO elastic_index
SELECT 
    product_id,
    name,
    description,
    price,
    category,
    tags,
    NOW() as indexed_at
FROM products;
```

### Dynamic Index Names

```sql
CREATE CONNECTION elastic_dynamic TO elasticsearch (
    nodes = 'http://localhost:9200',
    index_pattern = 'logs-{date}',
    type = 'sink',
    date_format = 'YYYY.MM.DD'
);

-- Index with daily indices
CREATE PIPELINE daily_logs AS
INSERT INTO elastic_dynamic
SELECT 
    log_level,
    message,
    service,
    timestamp,
    DATE_FORMAT(timestamp, 'YYYY.MM.DD') as date
FROM application_logs;
```

### Index Templates

```sql
CREATE CONNECTION elastic_template TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'metrics-*',
    type = 'sink',
    template_name = 'metrics_template',
    template_pattern = 'metrics-*',
    template_settings = '{
        "number_of_shards": 3,
        "number_of_replicas": 1,
        "refresh_interval": "5s"
    }',
    template_mappings = '{
        "properties": {
            "timestamp": {"type": "date"},
            "value": {"type": "double"},
            "tags": {"type": "keyword"}
        }
    }'
);
```

## Document Operations

### Upsert Documents

```sql
CREATE CONNECTION elastic_upsert TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'users',
    type = 'sink',
    id_field = 'user_id',
    operation = 'upsert'
);

-- Upsert user profiles
CREATE PIPELINE upsert_users AS
INSERT INTO elastic_upsert
SELECT 
    user_id,
    username,
    email,
    last_active,
    preferences,
    NOW() as updated_at
FROM user_updates;
```

### Update Documents

```sql
CREATE CONNECTION elastic_update TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'inventory',
    type = 'sink',
    id_field = 'sku',
    operation = 'update',
    update_script = 'ctx._source.quantity = params.quantity; ctx._source.updated = params.now'
);

-- Update inventory
CREATE PIPELINE update_inventory AS
INSERT INTO elastic_update
SELECT 
    sku,
    MAP {
        'quantity': new_quantity,
        'now': NOW()
    } as params
FROM inventory_changes;
```

### Delete Documents

```sql
CREATE CONNECTION elastic_delete TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'expired_sessions',
    type = 'sink',
    id_field = 'session_id',
    operation = 'delete'
);

-- Delete expired sessions
CREATE PIPELINE cleanup_sessions AS
INSERT INTO elastic_delete
SELECT session_id
FROM sessions
WHERE expired_at < NOW() - INTERVAL '24 hours';
```

## Bulk Operations

### Bulk Indexing

```sql
CREATE CONNECTION elastic_bulk TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'events',
    type = 'sink',
    bulk_size = 5000,
    bulk_timeout_ms = 1000,
    bulk_concurrent_requests = 2
);

-- High-volume indexing
CREATE PIPELINE bulk_events AS
INSERT INTO elastic_bulk
SELECT * FROM high_volume_stream;
```

### Mixed Bulk Operations

```sql
CREATE CONNECTION elastic_mixed TO elasticsearch (
    nodes = 'http://localhost:9200',
    type = 'sink',
    bulk_actions_field = '_action'
);

-- Mixed operations in one bulk
CREATE PIPELINE mixed_operations AS
INSERT INTO elastic_mixed
SELECT 
    CASE 
        WHEN operation = 'create' THEN 
            JSON_OBJECT('index': JSON_OBJECT('_index': 'products', '_id': id))
        WHEN operation = 'update' THEN 
            JSON_OBJECT('update': JSON_OBJECT('_index': 'products', '_id': id))
        WHEN operation = 'delete' THEN 
            JSON_OBJECT('delete': JSON_OBJECT('_index': 'products', '_id': id))
    END as _action,
    data
FROM operations_stream;
```

## Data Enrichment

### Ingest Pipelines

```sql
CREATE CONNECTION elastic_pipeline TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'enriched_data',
    type = 'sink',
    pipeline = 'enrich_pipeline'
);

-- Use ingest pipeline for enrichment
CREATE PIPELINE enrich_data AS
INSERT INTO elastic_pipeline
SELECT 
    id,
    raw_data,
    source_ip
FROM raw_events;
```

### GeoIP Enrichment

```sql
CREATE CONNECTION elastic_geoip TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'geo_events',
    type = 'sink',
    pipeline = 'geoip_pipeline',
    pipeline_definition = '{
        "processors": [
            {
                "geoip": {
                    "field": "client_ip",
                    "target_field": "geo"
                }
            }
        ]
    }'
);
```

## Index Management

### Index Lifecycle Management

```sql
CREATE CONNECTION elastic_ilm TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'logs',
    type = 'sink',
    ilm_policy = 'logs_policy',
    ilm_rollover_alias = 'logs-write',
    ilm_pattern = 'logs-{now/d}-000001'
);

-- Configure ILM policy
CREATE PIPELINE ilm_logs AS
INSERT INTO elastic_ilm
SELECT * FROM log_stream;
```

### Data Streams

```sql
CREATE CONNECTION elastic_datastream TO elasticsearch (
    nodes = 'http://localhost:9200',
    data_stream = 'metrics-laminar',
    type = 'sink',
    timestamp_field = '@timestamp'
);

-- Write to data stream
CREATE PIPELINE datastream_metrics AS
INSERT INTO elastic_datastream
SELECT 
    metric_name,
    value,
    tags,
    timestamp as '@timestamp'
FROM metrics;
```

## Search and Analytics

### Percolator Queries

```sql
CREATE CONNECTION elastic_percolator TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'percolator_queries',
    type = 'sink'
);

-- Store percolator queries
CREATE PIPELINE store_queries AS
INSERT INTO elastic_percolator
SELECT 
    query_id,
    JSON_OBJECT(
        'query': JSON_OBJECT(
            'match': JSON_OBJECT(
                'message': alert_pattern
            )
        )
    ) as query,
    alert_name,
    severity
FROM alert_definitions;
```

### Aggregation Pipelines

```sql
CREATE CONNECTION elastic_aggs TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'analytics',
    type = 'sink'
);

-- Pre-aggregate data
CREATE PIPELINE aggregate_metrics AS
INSERT INTO elastic_aggs
SELECT 
    DATE_TRUNC('hour', timestamp) as hour,
    service,
    COUNT(*) as request_count,
    AVG(response_time) as avg_response_time,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time) as p95_response_time
FROM requests
GROUP BY DATE_TRUNC('hour', timestamp), service;
```

## Performance Optimization

### Connection Pooling

```sql
CREATE CONNECTION elastic_pool TO elasticsearch (
    nodes = 'http://node1:9200,http://node2:9200,http://node3:9200',
    index = 'high_volume',
    type = 'sink',
    connection_pool_size = 10,
    connection_timeout_ms = 5000,
    socket_timeout_ms = 60000,
    max_retry_timeout_ms = 30000
);
```

### Compression

```sql
CREATE CONNECTION elastic_compressed TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'compressed_data',
    type = 'sink',
    compression = 'gzip',
    bulk_compression = true
);
```

### Routing

```sql
CREATE CONNECTION elastic_routing TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'sharded_data',
    type = 'sink',
    routing_field = 'tenant_id',
    number_of_shards = 10
);

-- Route documents by tenant
CREATE PIPELINE routed_indexing AS
INSERT INTO elastic_routing
SELECT 
    document_id,
    tenant_id,
    data
FROM multi_tenant_data;
```

## Security

### SSL/TLS Configuration

```sql
CREATE CONNECTION elastic_ssl TO elasticsearch (
    nodes = 'https://secure.elastic.com:9200',
    index = 'secure_data',
    type = 'sink',
    ssl_verification = true,
    ssl_ca_cert = '/certs/ca.pem',
    ssl_client_cert = '/certs/client-cert.pem',
    ssl_client_key = '/certs/client-key.pem'
);
```

### Field-Level Security

```sql
CREATE CONNECTION elastic_fls TO elasticsearch (
    nodes = 'https://localhost:9200',
    index = 'sensitive_data',
    type = 'sink',
    username = 'limited_user',
    password = '${secret:password}',
    excluded_fields = 'ssn,credit_card'
);
```

## Error Handling

### Retry Configuration

```sql
CREATE CONNECTION elastic_retry TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'events',
    type = 'sink',
    max_retries = 5,
    retry_on_conflict = 3,
    retry_backoff_ms = 100,
    retry_on_status = '429,503'
);
```

### Dead Letter Queue

```sql
-- Main Elasticsearch sink
CREATE CONNECTION elastic_main TO elasticsearch (
    nodes = 'http://localhost:9200',
    index = 'production',
    type = 'sink'
);

-- Handle failures
CREATE PIPELINE elastic_with_dlq AS
WITH validated AS (
    SELECT 
        *,
        validate_document(data) as is_valid
    FROM source_stream
)
INSERT INTO elastic_main
SELECT * FROM validated WHERE is_valid = true
UNION ALL
INSERT INTO dlq_table
SELECT 
    data,
    'validation_failed' as reason,
    NOW() as failed_at
FROM validated WHERE is_valid = false;
```

## Monitoring

### Sink Metrics

```sql
-- Monitor Elasticsearch sink performance
SELECT 
    connection_name,
    documents_indexed,
    documents_failed,
    bulk_requests_sent,
    avg_bulk_size,
    avg_indexing_latency_ms,
    last_error
FROM system.elasticsearch_sink_metrics
WHERE connection_name LIKE 'elastic_%';
```

### Bulk Operation Stats

```sql
-- Analyze bulk operations
SELECT 
    DATE_TRUNC('minute', timestamp) as minute,
    COUNT(*) as bulk_requests,
    SUM(doc_count) as total_documents,
    AVG(took_ms) as avg_bulk_time,
    MAX(took_ms) as max_bulk_time
FROM system.elasticsearch_bulk_stats
WHERE timestamp > NOW() - INTERVAL '1 hour'
GROUP BY DATE_TRUNC('minute', timestamp)
ORDER BY minute DESC;
```

## Best Practices

1. **Use bulk operations** for high throughput
2. **Configure appropriate refresh intervals**
3. **Use index templates** for consistent mappings
4. **Implement ILM** for data retention
5. **Set proper number of shards** based on data volume
6. **Use routing** for better query performance
7. **Enable compression** for network efficiency
8. **Monitor cluster health** regularly
9. **Use data streams** for time-series data
10. **Configure circuit breakers** appropriately

## Troubleshooting

### Connection Test

```sql
-- Test Elasticsearch connection
SELECT test_connection('elastic_sink');

-- Check cluster health
SELECT * FROM system.elasticsearch_cluster_health
WHERE connection = 'elastic_sink';
```

### Index Analysis

```sql
-- Analyze index statistics
SELECT 
    index_name,
    doc_count,
    store_size_bytes,
    segment_count,
    search_query_total,
    search_query_time_ms
FROM system.elasticsearch_indices
WHERE connection = 'elastic_sink'
ORDER BY doc_count DESC;
```