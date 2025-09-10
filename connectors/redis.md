---
sidebar_position: 7
title: Redis
---

The Redis connector enables Laminar to write data to Redis using various data structures (Strings, Lists, Hashes).

## Capabilities

- **Source**: No (Lookup tables supported for joins)
- **Sink**: Yes
- **Formats**: JSON, Raw
- **Data Structures**: String, List, Hash
- **Modes**: Standard Redis, Redis Cluster

## Configuration

### Connection Profile Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `connection` | object | Yes | - | Connection configuration (Standard or Clustered) |
| `username` | string | No | - | Username for authentication (supports env vars) |
| `password` | string | No | - | Password for authentication (supports env vars) |

#### Standard Connection
```json
{
  "connection": {
    "address": "redis://localhost:6379"
  }
}
```

#### Clustered Connection
```json
{
  "connection": {
    "addresses": [
      "redis://node1:6379",
      "redis://node2:6379",
      "redis://node3:6379"
    ]
  }
}
```

### Table Configuration

Redis supports three storage patterns:

#### String Table

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `keyPrefix` | string | Yes | - | Prefix for all keys |
| `keyColumn` | string | No | - | Column to append to prefix for key |
| `ttlSecs` | integer | No | - | TTL in seconds for expiration |

#### List Table

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `listPrefix` | string | Yes | - | Prefix for list keys |
| `listKeyColumn` | string | No | - | Column to append to prefix |
| `maxLength` | integer | No | - | Max list length (trimmed on write) |
| `operation` | enum | Yes | - | `Append` or `Prepend` |

#### Hash Table

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `hashKeyPrefix` | string | Yes | - | Prefix for hash keys |
| `hashKeyColumn` | string | No | - | Column for hash key |
| `hashFieldColumn` | string | Yes | - | Column for hash field name |

## API Usage

### Create Connection Profile

```bash
curl -X POST http://localhost:8000/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "redis_cache",
    "connector": "redis",
    "config": {
      "connection": {
        "address": "redis://cache.example.com:6379"
      },
      "username": "app_user",
      "password": "{{ REDIS_PASSWORD }}"
    }
  }'
```

### Create String Sink

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "user_sessions",
    "connector": "redis",
    "connectionProfileId": "prof_redis123",
    "config": {
      "connectorType": {
        "target": {
          "keyPrefix": "session:",
          "keyColumn": "session_id",
          "ttlSecs": 3600
        }
      }
    }
  }'
```

## SQL Usage

### String Storage

```sql
CREATE TABLE user_cache (
    user_id TEXT,
    user_data JSON,
    updated_at TIMESTAMP
) WITH (
    connector = 'redis',
    address = 'redis://localhost:6379',
    type = 'sink',
    'sink.type' = 'string',
    'sink.key_prefix' = 'user:',
    'sink.key_column' = 'user_id',
    'sink.ttl_secs' = '86400'
);
```

### List Storage (Event Stream)

```sql
CREATE TABLE event_stream (
    event_id BIGINT,
    event_type TEXT,
    payload JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'redis',
    address = 'redis://localhost:6379',
    type = 'sink',
    'sink.type' = 'list',
    'sink.list_prefix' = 'events:',
    'sink.list_key_column' = 'event_type',
    'sink.operation' = 'Append',
    'sink.max_length' = '10000'
);
```

### Hash Storage

```sql
CREATE TABLE product_inventory (
    product_id TEXT,
    warehouse_id TEXT,
    quantity INTEGER,
    last_updated TIMESTAMP
) WITH (
    connector = 'redis',
    address = 'redis://localhost:6379',
    type = 'sink',
    'sink.type' = 'hash',
    'sink.hash_key_prefix' = 'inventory:',
    'sink.hash_key_column' = 'warehouse_id',
    'sink.hash_field_column' = 'product_id'
);
```

## Examples

### Session Management

```sql
-- User sessions with TTL
CREATE TABLE sessions (
    session_id TEXT,
    user_id TEXT,
    user_email TEXT,
    permissions JSON,
    last_activity TIMESTAMP
) WITH (
    connector = 'redis',
    address = 'redis://cache:6379',
    password = '{{ REDIS_PASSWORD }}',
    type = 'sink',
    'sink.type' = 'string',
    'sink.key_prefix' = 'session:',
    'sink.key_column' = 'session_id',
    'sink.ttl_secs' = '1800'  -- 30 minute TTL
);

-- Update sessions from activity stream
INSERT INTO sessions
SELECT 
    session_id,
    user_id,
    user_email,
    JSON_OBJECT('roles', roles, 'scopes', scopes) as permissions,
    NOW() as last_activity
FROM user_activity
WHERE event_type = 'login';
```

### Real-time Leaderboard

```sql
-- Store leaderboard in Redis lists
CREATE TABLE leaderboard (
    game_id TEXT,
    player_id TEXT,
    score BIGINT,
    timestamp TIMESTAMP
) WITH (
    connector = 'redis',
    address = 'redis://localhost:6379',
    type = 'sink',
    'sink.type' = 'list',
    'sink.list_prefix' = 'leaderboard:',
    'sink.list_key_column' = 'game_id',
    'sink.operation' = 'Prepend',
    'sink.max_length' = '100'  -- Keep top 100
);

-- Update from game events
INSERT INTO leaderboard
SELECT 
    game_id,
    player_id,
    score,
    timestamp
FROM game_scores
WHERE score > 1000
ORDER BY score DESC;
```

### Inventory Tracking

```sql
-- Hash table for multi-warehouse inventory
CREATE TABLE inventory_updates (
    sku TEXT,
    warehouse TEXT,
    quantity INTEGER,
    updated_at TIMESTAMP
) WITH (
    connector = 'redis',
    address = 'redis://inventory:6379',
    type = 'sink',
    'sink.type' = 'hash',
    'sink.hash_key_prefix' = 'stock:',
    'sink.hash_key_column' = 'warehouse',
    'sink.hash_field_column' = 'sku'
);

-- Process inventory changes
INSERT INTO inventory_updates
SELECT 
    product_sku as sku,
    warehouse_code as warehouse,
    SUM(quantity_change) as quantity,
    MAX(event_time) as updated_at
FROM inventory_events
GROUP BY product_sku, warehouse_code;
```

## Data Structure Selection

### String Tables
- **Use for**: Simple key-value storage, caching, sessions
- **TTL Support**: Yes
- **Key Pattern**: `prefix + column_value`

### List Tables
- **Use for**: Event streams, queues, recent items
- **Operations**: Append (RPUSH) or Prepend (LPUSH)
- **Trimming**: Automatic with maxLength

### Hash Tables
- **Use for**: Complex objects, multi-field records
- **Key Pattern**: `prefix + key_column` → `{field_column: value}`
- **Efficiency**: Multiple fields under single key

## Best Practices

1. **Key Design**: Use consistent, hierarchical key naming (e.g., `type:id:field`)
2. **TTL Usage**: Set TTL for temporary data to prevent memory bloat
3. **List Trimming**: Use maxLength to cap list sizes
4. **Connection Pooling**: Redis connector manages connection pooling automatically
5. **Cluster Mode**: Use clustered connection for high availability
6. **Authentication**: Store passwords in environment variables

## Limitations

- Sink only (no source support for streaming)
- No support for Sorted Sets, Bitmaps, or HyperLogLog
- No transaction support
- No pub/sub capabilities (use NATS/Kafka for pub/sub)