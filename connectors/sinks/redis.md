---
sidebar_position: 2
title: Redis
---

# Redis Sink Connector

The Redis sink connector enables Laminar to write data to Redis databases, supporting various data structures and operation modes for caching, pub/sub, and real-time data storage.

## Overview

The Redis sink provides:
- **Multiple data structures** (Strings, Hashes, Lists, Sets, Sorted Sets, Streams)
- **Pub/Sub messaging** support
- **Redis Cluster** and **Sentinel** support
- **Pipelining and batching** for performance
- **TTL management** for automatic expiration
- **Lua scripting** support
- **Transaction support** with MULTI/EXEC
- **Connection pooling** and retry logic

## Configuration

### Basic Configuration

```sql
CREATE CONNECTION redis_sink TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `host` | string | Yes | - | Redis server host |
| `port` | integer | No | 6379 | Redis server port |
| `type` | string | Yes | - | Must be 'sink' |
| `operation` | string | Yes | - | Redis operation (set, hset, lpush, etc.) |
| `password` | string | No | - | Redis password |
| `database` | integer | No | 0 | Redis database number |
| `key_field` | string | No | - | Field to use as Redis key |
| `ttl_seconds` | integer | No | - | Default TTL for keys |
| `batch_size` | integer | No | 1000 | Batch size for pipelining |

## Data Operations

### String Operations

```sql
-- Simple SET operation
CREATE CONNECTION redis_strings TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    key_field = 'id',
    ttl_seconds = 3600
);

-- Pipeline usage
CREATE PIPELINE cache_users AS
INSERT INTO redis_strings
SELECT 
    CONCAT('user:', user_id) as id,
    TO_JSON(user_data) as value
FROM users
WHERE updated_at > NOW() - INTERVAL '1 hour';
```

### Hash Operations

```sql
CREATE CONNECTION redis_hashes TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'hset',
    key_prefix = 'user:',
    key_field = 'user_id'
);

-- Store user profiles as hashes
CREATE PIPELINE user_profiles AS
INSERT INTO redis_hashes
SELECT 
    user_id,
    MAP {
        'name': name,
        'email': email,
        'last_login': CAST(last_login AS STRING),
        'status': status
    } as fields
FROM users;
```

### List Operations

```sql
-- LPUSH operation
CREATE CONNECTION redis_queue TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'lpush',
    key_name = 'task_queue'
);

-- Add tasks to queue
CREATE PIPELINE enqueue_tasks AS
INSERT INTO redis_queue
SELECT 
    JSON_OBJECT(
        'id': task_id,
        'type': task_type,
        'payload': payload,
        'created_at': NOW()
    ) as value
FROM tasks
WHERE status = 'pending';
```

### Set Operations

```sql
CREATE CONNECTION redis_sets TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'sadd',
    key_field = 'category'
);

-- Track unique visitors
CREATE PIPELINE unique_visitors AS
INSERT INTO redis_sets
SELECT 
    CONCAT('visitors:', DATE_FORMAT(timestamp, 'YYYY-MM-DD')) as category,
    user_id as member
FROM page_views;
```

### Sorted Set Operations

```sql
CREATE CONNECTION redis_sorted_sets TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'zadd',
    key_name = 'leaderboard'
);

-- Update leaderboard
CREATE PIPELINE update_scores AS
INSERT INTO redis_sorted_sets
SELECT 
    user_id as member,
    score as score
FROM game_scores
WHERE game_date = CURRENT_DATE();
```

### Stream Operations

```sql
CREATE CONNECTION redis_streams TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'xadd',
    key_name = 'events',
    stream_maxlen = 10000
);

-- Add events to stream
CREATE PIPELINE stream_events AS
INSERT INTO redis_streams
SELECT 
    '*' as id,  -- Auto-generate ID
    MAP {
        'event_type': event_type,
        'user_id': CAST(user_id AS STRING),
        'timestamp': CAST(timestamp AS STRING),
        'data': TO_JSON(event_data)
    } as fields
FROM events;
```

## Pub/Sub Operations

### Publishing Messages

```sql
CREATE CONNECTION redis_pubsub TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'publish',
    channel_field = 'channel'
);

-- Publish notifications
CREATE PIPELINE publish_notifications AS
INSERT INTO redis_pubsub
SELECT 
    CONCAT('notifications:', user_segment) as channel,
    JSON_OBJECT(
        'type': notification_type,
        'title': title,
        'message': message,
        'timestamp': NOW()
    ) as message
FROM notifications
WHERE active = true;
```

### Pattern-based Publishing

```sql
CREATE CONNECTION redis_pattern_pub TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'publish'
);

-- Route messages by pattern
CREATE PIPELINE route_messages AS
INSERT INTO redis_pattern_pub
SELECT 
    CASE 
        WHEN priority = 'high' THEN 'alerts:critical'
        WHEN priority = 'medium' THEN 'alerts:warning'
        ELSE 'alerts:info'
    END as channel,
    TO_JSON(alert_data) as message
FROM alerts;
```

## Advanced Features

### Transactions

```sql
CREATE CONNECTION redis_transactional TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'multi',
    transaction_size = 100
);

-- Atomic operations
CREATE PIPELINE atomic_updates AS
INSERT INTO redis_transactional
SELECT 
    ARRAY[
        JSON_OBJECT('op': 'set', 'key': CONCAT('balance:', account_id), 'value': new_balance),
        JSON_OBJECT('op': 'lpush', 'key': 'transactions', 'value': TO_JSON(transaction)),
        JSON_OBJECT('op': 'incr', 'key': 'transaction_count')
    ] as commands
FROM financial_transactions;
```

### Lua Scripting

```sql
CREATE CONNECTION redis_lua TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'eval',
    script = '
        local key = KEYS[1]
        local value = ARGV[1]
        local ttl = ARGV[2]
        redis.call("SET", key, value)
        if ttl then
            redis.call("EXPIRE", key, ttl)
        end
        return redis.call("GET", key)
    '
);

-- Execute Lua scripts
CREATE PIPELINE lua_processor AS
INSERT INTO redis_lua
SELECT 
    ARRAY[CONCAT('processed:', id)] as keys,
    ARRAY[TO_JSON(data), '3600'] as args
FROM raw_data;
```

### Pipelining

```sql
CREATE CONNECTION redis_pipeline TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'pipeline',
    batch_size = 1000,
    batch_timeout_ms = 100
);

-- Batch multiple operations
CREATE PIPELINE batch_processor AS
INSERT INTO redis_pipeline
SELECT 
    ARRAY_AGG(
        JSON_OBJECT(
            'operation': 'set',
            'key': CONCAT('item:', item_id),
            'value': TO_JSON(item_data),
            'ttl': 7200
        )
    ) as commands
FROM items
GROUP BY TUMBLE(timestamp, INTERVAL '1 second');
```

## Cluster and High Availability

### Redis Cluster

```sql
CREATE CONNECTION redis_cluster TO redis (
    cluster_nodes = 'node1:7000,node2:7000,node3:7000',
    type = 'sink',
    operation = 'set',
    cluster_mode = true,
    read_from_replicas = true
);
```

### Redis Sentinel

```sql
CREATE CONNECTION redis_sentinel TO redis (
    sentinel_hosts = 'sentinel1:26379,sentinel2:26379,sentinel3:26379',
    sentinel_master = 'mymaster',
    type = 'sink',
    operation = 'set',
    password = '${secret:redis_password}'
);
```

## Data Type Conversions

### JSON to Redis

```sql
CREATE CONNECTION redis_json TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'json.set',  -- RedisJSON module
    key_field = 'id'
);

-- Store complex JSON objects
CREATE PIPELINE json_storage AS
INSERT INTO redis_json
SELECT 
    CONCAT('doc:', document_id) as id,
    '$' as path,
    document_content as json_value
FROM documents;
```

### Geospatial Data

```sql
CREATE CONNECTION redis_geo TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'geoadd',
    key_name = 'locations'
);

-- Store location data
CREATE PIPELINE store_locations AS
INSERT INTO redis_geo
SELECT 
    longitude,
    latitude,
    location_id as member
FROM locations
WHERE active = true;
```

## Performance Optimization

### Connection Pooling

```sql
CREATE CONNECTION redis_pooled TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    connection_pool_size = 20,
    connection_timeout_ms = 5000,
    max_retries = 3
);
```

### Compression

```sql
CREATE CONNECTION redis_compressed TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    compression = 'gzip',
    compression_threshold = 1024  -- Compress values > 1KB
);
```

## TTL and Expiration

### Dynamic TTL

```sql
CREATE CONNECTION redis_dynamic_ttl TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    ttl_field = 'expiry_seconds'
);

-- Set TTL based on data
CREATE PIPELINE cache_with_ttl AS
INSERT INTO redis_dynamic_ttl
SELECT 
    cache_key,
    cache_value,
    CASE 
        WHEN cache_type = 'session' THEN 3600
        WHEN cache_type = 'static' THEN 86400
        ELSE 300
    END as expiry_seconds
FROM cache_entries;
```

### Sliding Window TTL

```sql
CREATE CONNECTION redis_sliding TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    ttl_seconds = 3600,
    refresh_ttl_on_update = true
);
```

## Error Handling

### Retry Logic

```sql
CREATE CONNECTION redis_retry TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    max_retries = 5,
    retry_backoff_ms = 1000,
    retry_max_backoff_ms = 30000,
    on_error = 'retry'  -- or 'skip', 'fail'
);
```

### Dead Letter Queue

```sql
-- Main Redis sink
CREATE CONNECTION redis_main TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set'
);

-- Fallback for failures
CREATE PIPELINE redis_with_dlq AS
WITH processed AS (
    SELECT 
        *,
        TRY_REDIS_SET(key, value) as result
    FROM data_stream
)
INSERT INTO redis_main
SELECT key, value FROM processed WHERE result = true
UNION ALL
INSERT INTO error_log
SELECT 
    key,
    value,
    'redis_write_failed' as error_type,
    NOW() as error_time
FROM processed WHERE result = false;
```

## Monitoring

### Connection Metrics

```sql
-- Monitor Redis sink performance
SELECT 
    connection_name,
    operations_sent,
    operations_succeeded,
    operations_failed,
    avg_latency_ms,
    p95_latency_ms,
    p99_latency_ms,
    bytes_sent,
    connection_errors
FROM system.redis_sink_metrics
WHERE connection_name LIKE 'redis_%';
```

### Operation Analysis

```sql
-- Analyze operation patterns
SELECT 
    DATE_TRUNC('minute', timestamp) as minute,
    operation_type,
    COUNT(*) as operation_count,
    AVG(latency_ms) as avg_latency,
    MAX(latency_ms) as max_latency,
    SUM(CASE WHEN success THEN 0 ELSE 1 END) as failures
FROM system.redis_operations
WHERE timestamp > NOW() - INTERVAL '1 hour'
GROUP BY DATE_TRUNC('minute', timestamp), operation_type
ORDER BY minute DESC;
```

## Security

### TLS/SSL Connection

```sql
CREATE CONNECTION redis_tls TO redis (
    host = 'secure-redis.example.com',
    port = 6380,
    type = 'sink',
    operation = 'set',
    tls_enabled = true,
    tls_ca_cert = '/certs/ca.pem',
    tls_cert = '/certs/client-cert.pem',
    tls_key = '/certs/client-key.pem'
);
```

### ACL Authentication

```sql
CREATE CONNECTION redis_acl TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    username = 'laminar_user',
    password = '${secret:redis_acl_password}',
    acl_enabled = true
);
```

## Best Practices

1. **Use pipelining** for bulk operations
2. **Configure appropriate TTLs** to manage memory
3. **Use connection pooling** for high throughput
4. **Monitor memory usage** in Redis
5. **Implement retry logic** for transient failures
6. **Use appropriate data structures** for your use case
7. **Enable persistence** for critical data
8. **Set up replication** for high availability
9. **Use Redis Cluster** for horizontal scaling
10. **Monitor slow operations** with SLOWLOG

## Troubleshooting

### Connection Test

```sql
-- Test Redis connection
SELECT test_connection('redis_sink');

-- Check connection status
SELECT * FROM system.redis_connections
WHERE connection = 'redis_sink';
```

### Debug Operations

```sql
-- Enable debug logging
CREATE CONNECTION redis_debug TO redis (
    host = 'localhost',
    port = 6379,
    type = 'sink',
    operation = 'set',
    debug_logging = true,
    log_operations = true
);

-- View debug logs
SELECT * FROM system.redis_logs
WHERE connection = 'redis_debug'
ORDER BY timestamp DESC
LIMIT 100;
```