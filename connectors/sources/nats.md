---
sidebar_position: 6
title: NATS
---

# NATS Source Connector

The NATS source connector enables Laminar to consume messages from NATS messaging system, supporting Core NATS, JetStream, and various messaging patterns.

## Overview

The NATS connector provides:
- **Core NATS** and **JetStream** support
- **Subject-based routing** with wildcards
- **Queue groups** for load balancing
- **Durable subscriptions** for JetStream
- **Request-reply patterns**
- **Key-value and object stores**
- **Exactly-once delivery** with JetStream
- **TLS and authentication** support

## Configuration

### Basic Core NATS

```sql
CREATE CONNECTION nats_source TO nats (
    servers = 'nats://localhost:4222',
    subject = 'events.>',
    type = 'source',
    format = 'json'
);
```

### JetStream Configuration

```sql
CREATE CONNECTION nats_jetstream TO nats (
    servers = 'nats://localhost:4222',
    subject = 'orders.*',
    type = 'source',
    format = 'json',
    jetstream = true,
    stream_name = 'ORDERS',
    consumer_name = 'laminar-consumer'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `servers` | string | Yes | - | NATS server URLs |
| `subject` | string | Yes | - | Subject to subscribe to |
| `type` | string | Yes | - | Must be 'source' |
| `format` | string | Yes | - | Data format (json, protobuf, raw) |
| `jetstream` | boolean | No | false | Use JetStream |
| `stream_name` | string | No | - | JetStream stream name |
| `consumer_name` | string | No | - | JetStream consumer name |
| `queue_group` | string | No | - | Queue group name |
| `max_reconnects` | integer | No | 10 | Max reconnection attempts |

## Authentication

### No Authentication

```sql
CREATE CONNECTION nats_public TO nats (
    servers = 'nats://localhost:4222',
    subject = 'public.events',
    type = 'source',
    format = 'json'
);
```

### Token Authentication

```sql
CREATE CONNECTION nats_token TO nats (
    servers = 'nats://localhost:4222',
    subject = 'secure.events',
    type = 'source',
    format = 'json',
    token = '${secret:nats_token}'
);
```

### User/Password Authentication

```sql
CREATE CONNECTION nats_userpass TO nats (
    servers = 'nats://localhost:4222',
    subject = 'private.events',
    type = 'source',
    format = 'json',
    username = 'laminar',
    password = '${secret:nats_password}'
);
```

### NKey Authentication

```sql
CREATE CONNECTION nats_nkey TO nats (
    servers = 'nats://localhost:4222',
    subject = 'secure.data',
    type = 'source',
    format = 'json',
    nkey_seed = '${secret:nkey_seed}'
);
```

### JWT Authentication

```sql
CREATE CONNECTION nats_jwt TO nats (
    servers = 'nats://localhost:4222',
    subject = 'protected.stream',
    type = 'source',
    format = 'json',
    jwt = '${secret:nats_jwt}',
    nkey_seed = '${secret:nkey_seed}'
);
```

## Subject Patterns

### Exact Subject

```sql
CREATE CONNECTION nats_exact TO nats (
    servers = 'nats://localhost:4222',
    subject = 'orders.created',
    type = 'source',
    format = 'json'
);
```

### Wildcard Subscriptions

```sql
-- Single token wildcard (*)
CREATE CONNECTION nats_wildcard TO nats (
    servers = 'nats://localhost:4222',
    subject = 'events.*.created',  -- Matches events.order.created, events.user.created
    type = 'source',
    format = 'json'
);

-- Multi-token wildcard (>)
CREATE CONNECTION nats_multi_wildcard TO nats (
    servers = 'nats://localhost:4222',
    subject = 'logs.>',  -- Matches all subjects starting with logs.
    type = 'source',
    format = 'json'
);
```

## Queue Groups

### Load Balancing

```sql
CREATE CONNECTION nats_queue TO nats (
    servers = 'nats://localhost:4222',
    subject = 'tasks',
    type = 'source',
    format = 'json',
    queue_group = 'workers'
);

-- Multiple instances will load balance
CREATE PIPELINE worker_1 AS
SELECT * FROM nats_queue WHERE MOD(HASH(task_id), 2) = 0;

CREATE PIPELINE worker_2 AS
SELECT * FROM nats_queue WHERE MOD(HASH(task_id), 2) = 1;
```

## JetStream Features

### Stream Creation

```sql
CREATE CONNECTION nats_js_stream TO nats (
    servers = 'nats://localhost:4222',
    type = 'source',
    format = 'json',
    jetstream = true,
    stream_name = 'EVENTS',
    stream_subjects = 'events.>',
    stream_retention = 'limits',  -- limits, interest, workqueue
    stream_max_msgs = 1000000,
    stream_max_age = 86400,  -- 1 day
    stream_storage = 'file'  -- file or memory
);
```

### Durable Consumer

```sql
CREATE CONNECTION nats_durable TO nats (
    servers = 'nats://localhost:4222',
    subject = 'orders.>',
    type = 'source',
    format = 'json',
    jetstream = true,
    stream_name = 'ORDERS',
    consumer_name = 'laminar-durable',
    consumer_durable = true,
    consumer_ack_policy = 'explicit',
    consumer_max_deliver = 3
);
```

### Pull Consumer

```sql
CREATE CONNECTION nats_pull TO nats (
    servers = 'nats://localhost:4222',
    subject = 'events.>',
    type = 'source',
    format = 'json',
    jetstream = true,
    stream_name = 'EVENTS',
    consumer_name = 'laminar-pull',
    consumer_pull = true,
    pull_batch_size = 100,
    pull_timeout_ms = 1000
);
```

### Ordered Consumer

```sql
CREATE CONNECTION nats_ordered TO nats (
    servers = 'nats://localhost:4222',
    subject = 'ledger.>',
    type = 'source',
    format = 'json',
    jetstream = true,
    stream_name = 'LEDGER',
    consumer_ordered = true  -- Guaranteed message ordering
);
```

## Data Formats

### JSON Messages

```sql
CREATE CONNECTION nats_json TO nats (
    servers = 'nats://localhost:4222',
    subject = 'json.events',
    type = 'source',
    format = 'json'
);

-- Usage
CREATE TABLE nats_events (
    id TEXT,
    event_type TEXT,
    timestamp TIMESTAMP,
    data JSON
) WITH (
    connector = 'nats_json'
);
```

### Protobuf Messages

```sql
CREATE CONNECTION nats_protobuf TO nats (
    servers = 'nats://localhost:4222',
    subject = 'proto.events',
    type = 'source',
    format = 'protobuf',
    protobuf_message = 'com.example.Event',
    protobuf_descriptor = '/schemas/event.desc'
);
```

### Raw Binary

```sql
CREATE CONNECTION nats_raw TO nats (
    servers = 'nats://localhost:4222',
    subject = 'binary.data',
    type = 'source',
    format = 'raw'
);

-- Access raw data with headers
CREATE TABLE raw_messages (
    data BYTEA,
    subject TEXT,
    reply_to TEXT,
    headers JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats_raw',
    include_metadata = true
);
```

## Message Metadata

```sql
-- Table with NATS metadata
CREATE TABLE nats_with_metadata (
    -- Data fields
    payload JSON,
    
    -- NATS metadata
    _nats_subject TEXT,
    _nats_reply TEXT,
    _nats_headers JSON,
    _nats_sequence BIGINT,  -- JetStream only
    _nats_stream TEXT,       -- JetStream only
    _nats_consumer TEXT,     -- JetStream only
    _nats_delivered BIGINT   -- JetStream delivery count
) WITH (
    connector = 'nats_source',
    include_metadata = true
);

-- Query with metadata
SELECT 
    payload,
    _nats_subject,
    _nats_sequence,
    _nats_delivered
FROM nats_with_metadata
WHERE _nats_delivered > 1;  -- Redelivered messages
```

## Request-Reply Pattern

### Reply Handler

```sql
CREATE CONNECTION nats_reply TO nats (
    servers = 'nats://localhost:4222',
    subject = 'requests.>',
    type = 'source',
    format = 'json',
    enable_reply = true
);

-- Process requests and send replies
CREATE PIPELINE request_handler AS
WITH processed AS (
    SELECT 
        payload,
        _nats_reply as reply_to,
        CASE 
            WHEN JSON_EXTRACT(payload, '$.action') = 'get' THEN 
                get_data(JSON_EXTRACT(payload, '$.id'))
            ELSE 
                JSON_OBJECT('error', 'Unknown action')
        END as response
    FROM nats_reply
)
INSERT INTO nats_responder
SELECT reply_to as subject, response as payload
FROM processed;
```

## Key-Value Store

### KV Subscribe

```sql
CREATE CONNECTION nats_kv TO nats (
    servers = 'nats://localhost:4222',
    type = 'source',
    format = 'json',
    jetstream = true,
    kv_bucket = 'CONFIG',
    kv_keys = 'settings.>',  -- Watch specific keys
    kv_watch = true
);

-- Track configuration changes
CREATE TABLE config_changes (
    key TEXT,
    value JSON,
    revision BIGINT,
    operation TEXT,  -- PUT, DELETE
    timestamp TIMESTAMP
) WITH (
    connector = 'nats_kv'
);
```

## Object Store

### Object Watch

```sql
CREATE CONNECTION nats_object TO nats (
    servers = 'nats://localhost:4222',
    type = 'source',
    format = 'raw',
    jetstream = true,
    object_bucket = 'FILES',
    object_watch = true
);

-- Track file uploads
CREATE TABLE file_events (
    name TEXT,
    size BIGINT,
    chunks INT,
    digest TEXT,
    metadata JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats_object'
);
```

## Clustering and HA

### Multiple Servers

```sql
CREATE CONNECTION nats_cluster TO nats (
    servers = 'nats://server1:4222,nats://server2:4222,nats://server3:4222',
    subject = 'events.>',
    type = 'source',
    format = 'json',
    randomize_servers = true
);
```

### Leafnode Connection

```sql
CREATE CONNECTION nats_leaf TO nats (
    servers = 'nats://leafnode.example.com:7422',
    subject = 'remote.>',
    type = 'source',
    format = 'json',
    tls_required = true,
    tls_ca_file = '/certs/ca.pem'
);
```

## Performance Optimization

### Batching

```sql
CREATE CONNECTION nats_batch TO nats (
    servers = 'nats://localhost:4222',
    subject = 'high-volume.>',
    type = 'source',
    format = 'json',
    batch_size = 1000,
    batch_timeout_ms = 100,
    pending_limits = 65536
);
```

### Flow Control

```sql
CREATE CONNECTION nats_flow TO nats (
    servers = 'nats://localhost:4222',
    subject = 'stream.>',
    type = 'source',
    format = 'json',
    jetstream = true,
    stream_name = 'STREAM',
    consumer_rate_limit = 1000,  -- Messages per second
    consumer_max_ack_pending = 100
);
```

## Error Handling

### Retry Configuration

```sql
CREATE CONNECTION nats_retry TO nats (
    servers = 'nats://localhost:4222',
    subject = 'critical.>',
    type = 'source',
    format = 'json',
    max_reconnects = -1,  -- Infinite
    reconnect_wait_ms = 2000,
    reconnect_jitter_ms = 100,
    ping_interval_s = 120,
    ping_max_outstanding = 2
);
```

### Dead Letter Queue

```sql
-- JetStream with DLQ
CREATE CONNECTION nats_dlq TO nats (
    servers = 'nats://localhost:4222',
    subject = 'orders.>',
    type = 'source',
    format = 'json',
    jetstream = true,
    stream_name = 'ORDERS',
    consumer_max_deliver = 3,
    consumer_backoff = '1s,5s,10s'
);

-- Handle failed messages
CREATE PIPELINE dlq_handler AS
SELECT * FROM nats_dlq
WHERE _nats_delivered >= 3;  -- Max delivery attempts reached
```

## Monitoring

### Connection Metrics

```sql
-- Monitor NATS connections
SELECT 
    connection_name,
    server_url,
    state,
    subscriptions_count,
    messages_received,
    bytes_received,
    messages_dropped,
    reconnects,
    last_error
FROM system.nats_connections
WHERE connection_name LIKE 'nats_%';
```

### JetStream Metrics

```sql
-- Monitor JetStream consumers
SELECT 
    stream_name,
    consumer_name,
    pending_messages,
    ack_pending,
    redelivered,
    waiting_pulls,
    last_delivered,
    lag
FROM system.nats_jetstream_consumers
WHERE stream_name = 'EVENTS';
```

## Security

### TLS Configuration

```sql
CREATE CONNECTION nats_tls TO nats (
    servers = 'nats://secure.example.com:4222',
    subject = 'secure.>',
    type = 'source',
    format = 'json',
    tls_required = true,
    tls_ca_file = '/certs/ca.pem',
    tls_cert_file = '/certs/client-cert.pem',
    tls_key_file = '/certs/client-key.pem',
    tls_verify = true
);
```

### Permissions

```sql
CREATE CONNECTION nats_perms TO nats (
    servers = 'nats://localhost:4222',
    subject = 'restricted.>',
    type = 'source',
    format = 'json',
    user = 'limited_user',
    password = '${secret:user_password}',
    permissions = MAP {
        'publish': ['responses.>'],
        'subscribe': ['requests.>', 'events.>']
    }
);
```

## Best Practices

1. **Use JetStream** for guaranteed delivery
2. **Configure appropriate ACK policies** for your use case
3. **Use queue groups** for horizontal scaling
4. **Set reasonable reconnect limits**
5. **Monitor consumer lag** in JetStream
6. **Use durable consumers** for critical data
7. **Configure flow control** for high-volume streams
8. **Implement proper error handling**
9. **Use TLS** in production
10. **Set up clustering** for high availability

## Troubleshooting

### Connection Test

```sql
-- Test NATS connection
SELECT test_connection('nats_source');

-- Check server info
SELECT * FROM system.nats_server_info
WHERE connection = 'nats_source';
```

### Debug Logging

```sql
CREATE CONNECTION nats_debug TO nats (
    servers = 'nats://localhost:4222',
    subject = 'debug.>',
    type = 'source',
    format = 'json',
    debug = true,
    trace = true
);

-- View debug logs
SELECT * FROM system.nats_logs
WHERE connection = 'nats_debug'
ORDER BY timestamp DESC
LIMIT 100;
```