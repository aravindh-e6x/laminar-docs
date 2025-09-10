---
sidebar_position: 4
title: WebSocket
---

# WebSocket Source Connector

The WebSocket source connector enables Laminar to receive real-time streaming data through WebSocket connections, supporting both client and server modes with automatic reconnection and message buffering.

## Overview

The WebSocket connector provides:
- **Bidirectional communication** over persistent connections
- **Client and server modes** for flexible deployment
- **Automatic reconnection** with exponential backoff
- **Message buffering** and flow control
- **TLS/WSS support** for secure connections
- **Custom headers** and authentication
- **Multiple data formats** (JSON, Text, Binary)
- **Heartbeat/ping-pong** support

## Configuration

### WebSocket Client Mode

```sql
CREATE CONNECTION websocket_client TO websocket (
    endpoint = 'wss://stream.example.com/events',
    mode = 'client',
    format = 'json'
);
```

### WebSocket Server Mode

```sql
CREATE CONNECTION websocket_server TO websocket (
    port = 8080,
    path = '/events',
    mode = 'server',
    format = 'json'
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `endpoint` | string | Yes (client) | - | WebSocket endpoint URL |
| `port` | integer | Yes (server) | - | Server port to listen on |
| `path` | string | No | `/` | Server endpoint path |
| `mode` | string | Yes | - | Connection mode (client/server) |
| `format` | string | Yes | - | Data format (json, text, binary) |
| `headers` | map | No | - | Custom HTTP headers |
| `reconnect` | boolean | No | true | Auto-reconnect on disconnect |
| `reconnect_interval_ms` | integer | No | 5000 | Initial reconnect delay |
| `max_reconnect_attempts` | integer | No | -1 | Max reconnection attempts (-1 = infinite) |
| `heartbeat_interval_s` | integer | No | 30 | Heartbeat interval in seconds |
| `buffer_size` | integer | No | 1000 | Message buffer size |

## Authentication

### Basic Authentication

```sql
CREATE CONNECTION ws_basic_auth TO websocket (
    endpoint = 'wss://api.example.com/stream',
    mode = 'client',
    format = 'json',
    headers = MAP {
        'Authorization': 'Basic ${secret:ws_credentials}'
    }
);
```

### Bearer Token

```sql
CREATE CONNECTION ws_bearer TO websocket (
    endpoint = 'wss://api.example.com/stream',
    mode = 'client',
    format = 'json',
    headers = MAP {
        'Authorization': 'Bearer ${secret:api_token}'
    }
);
```

### API Key Authentication

```sql
CREATE CONNECTION ws_api_key TO websocket (
    endpoint = 'wss://api.example.com/stream',
    mode = 'client',
    format = 'json',
    headers = MAP {
        'X-API-Key': '${secret:api_key}',
        'X-Client-ID': 'laminar-stream-processor'
    }
);
```

## Data Formats

### JSON Messages

```sql
CREATE CONNECTION ws_json TO websocket (
    endpoint = 'wss://stream.example.com/json',
    mode = 'client',
    format = 'json'
);

-- Usage
CREATE TABLE ws_events (
    event_id TEXT,
    event_type TEXT,
    timestamp TIMESTAMP,
    data JSON
) WITH (
    connector = 'ws_json'
);

-- Query JSON data
SELECT 
    event_id,
    event_type,
    JSON_EXTRACT(data, '$.user_id') as user_id,
    JSON_EXTRACT(data, '$.amount') as amount
FROM ws_events
WHERE event_type = 'transaction';
```

### Text Messages

```sql
CREATE CONNECTION ws_text TO websocket (
    endpoint = 'wss://chat.example.com/messages',
    mode = 'client',
    format = 'text'
);

-- Parse text messages
CREATE TABLE chat_messages (
    message TEXT,
    received_at TIMESTAMP DEFAULT NOW()
) WITH (
    connector = 'ws_text'
);
```

### Binary Messages

```sql
CREATE CONNECTION ws_binary TO websocket (
    endpoint = 'wss://data.example.com/binary',
    mode = 'client',
    format = 'binary'
);

-- Handle binary data
CREATE TABLE binary_stream (
    data BYTEA,
    size INT,
    received_at TIMESTAMP DEFAULT NOW()
) WITH (
    connector = 'ws_binary'
);
```

## Client Mode Examples

### Stock Market Data

```sql
CREATE CONNECTION stock_stream TO websocket (
    endpoint = 'wss://stream.finance.com/stocks',
    mode = 'client',
    format = 'json',
    headers = MAP {
        'X-API-Key': '${secret:finance_api_key}'
    },
    reconnect = true,
    heartbeat_interval_s = 10
);

-- Real-time stock processing
CREATE PIPELINE stock_processor AS
SELECT 
    symbol,
    price,
    volume,
    AVG(price) OVER (
        PARTITION BY symbol 
        ORDER BY timestamp 
        ROWS BETWEEN 99 PRECEDING AND CURRENT ROW
    ) as moving_avg_100,
    timestamp
FROM stock_stream
WHERE symbol IN ('AAPL', 'GOOGL', 'MSFT');
```

### IoT Sensor Data

```sql
CREATE CONNECTION iot_stream TO websocket (
    endpoint = 'wss://iot.example.com/sensors',
    mode = 'client',
    format = 'json',
    reconnect = true,
    max_reconnect_attempts = 100,
    buffer_size = 5000
);

-- Process sensor readings
CREATE PIPELINE sensor_analytics AS
SELECT 
    sensor_id,
    location,
    AVG(temperature) as avg_temp,
    MAX(temperature) as max_temp,
    MIN(temperature) as min_temp,
    COUNT(*) as reading_count,
    TUMBLE(timestamp, INTERVAL '5 minutes') as window
FROM iot_stream
GROUP BY 
    sensor_id,
    location,
    TUMBLE(timestamp, INTERVAL '5 minutes');
```

## Server Mode Examples

### Event Collection Server

```sql
CREATE CONNECTION event_collector TO websocket (
    port = 8080,
    path = '/collect',
    mode = 'server',
    format = 'json',
    max_connections = 1000,
    buffer_size = 10000
);

-- Process collected events
CREATE TABLE collected_events (
    client_id TEXT,
    event_type TEXT,
    payload JSON,
    received_at TIMESTAMP DEFAULT NOW()
) WITH (
    connector = 'event_collector',
    include_metadata = true
);
```

### Multi-Client Chat Server

```sql
CREATE CONNECTION chat_server TO websocket (
    port = 8081,
    path = '/chat',
    mode = 'server',
    format = 'json',
    broadcast = true,  -- Broadcast to all connected clients
    max_connections = 500
);

-- Store and forward messages
CREATE PIPELINE chat_processor AS
INSERT INTO chat_outgoing
SELECT 
    JSON_OBJECT(
        'user': user_name,
        'message': message,
        'timestamp': NOW(),
        'room': room_id
    ) as outbound_message
FROM chat_server
WHERE message_type = 'chat';
```

## Connection Management

### Reconnection Strategy

```sql
CREATE CONNECTION ws_resilient TO websocket (
    endpoint = 'wss://critical.example.com/stream',
    mode = 'client',
    format = 'json',
    reconnect = true,
    reconnect_interval_ms = 1000,  -- Start with 1 second
    reconnect_backoff_multiplier = 2,  -- Double each time
    max_reconnect_interval_ms = 60000,  -- Max 1 minute
    max_reconnect_attempts = -1  -- Infinite retries
);
```

### Connection Pool

```sql
-- Multiple connections for load balancing
CREATE CONNECTION ws_pool_1 TO websocket (
    endpoint = 'wss://server1.example.com/stream',
    mode = 'client',
    format = 'json'
);

CREATE CONNECTION ws_pool_2 TO websocket (
    endpoint = 'wss://server2.example.com/stream',
    mode = 'client',
    format = 'json'
);

-- Unified view
CREATE VIEW balanced_stream AS
SELECT *, 'server1' as source FROM ws_pool_1
UNION ALL
SELECT *, 'server2' as source FROM ws_pool_2;
```

## Message Handling

### Message Filtering

```sql
-- Connection with subscription
CREATE CONNECTION ws_filtered TO websocket (
    endpoint = 'wss://stream.example.com/data',
    mode = 'client',
    format = 'json',
    subscribe_message = JSON_OBJECT(
        'action': 'subscribe',
        'channels': ['trades', 'orders'],
        'symbols': ['BTC', 'ETH']
    )
);
```

### Message Acknowledgment

```sql
CREATE CONNECTION ws_with_ack TO websocket (
    endpoint = 'wss://reliable.example.com/stream',
    mode = 'client',
    format = 'json',
    auto_ack = false,  -- Manual acknowledgment
    ack_timeout_ms = 5000
);

-- Process with acknowledgment
CREATE PIPELINE reliable_processor AS
WITH processed AS (
    SELECT 
        *,
        process_message(data) as result
    FROM ws_with_ack
)
SELECT 
    ACKNOWLEDGE_MESSAGE(message_id) as ack
FROM processed
WHERE result.success = true;
```

## Error Handling

### Connection Error Recovery

```sql
CREATE CONNECTION ws_error_handling TO websocket (
    endpoint = 'wss://api.example.com/stream',
    mode = 'client',
    format = 'json',
    on_error = 'continue',  -- continue, fail, retry
    max_errors_per_minute = 10,
    error_buffer_size = 100
);

-- Monitor errors
CREATE TABLE ws_errors (
    error_type TEXT,
    error_message TEXT,
    failed_message TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
) WITH (
    connector = 'ws_error_handling',
    stream = 'errors'
);
```

### Dead Letter Queue

```sql
-- Main connection
CREATE CONNECTION ws_main TO websocket (
    endpoint = 'wss://stream.example.com/data',
    mode = 'client',
    format = 'json',
    dlq_enabled = true
);

-- Process valid messages
CREATE PIPELINE valid_messages AS
SELECT * FROM ws_main
WHERE IS_VALID_JSON(message);

-- Handle invalid messages
CREATE PIPELINE invalid_messages AS
INSERT INTO error_log
SELECT 
    message as raw_message,
    'invalid_json' as error_type,
    NOW() as error_time
FROM ws_main
WHERE NOT IS_VALID_JSON(message);
```

## Security

### TLS/SSL Configuration

```sql
CREATE CONNECTION ws_secure TO websocket (
    endpoint = 'wss://secure.example.com/stream',
    mode = 'client',
    format = 'json',
    tls_verify = true,
    tls_ca_cert = '/certs/ca.pem',
    tls_client_cert = '/certs/client.pem',
    tls_client_key = '/certs/client.key',
    tls_hostname_verification = true
);
```

### Mutual TLS

```sql
CREATE CONNECTION ws_mtls TO websocket (
    endpoint = 'wss://mtls.example.com/stream',
    mode = 'client',
    format = 'json',
    tls_verify = true,
    tls_ca_cert = '${secret:ca_cert}',
    tls_client_cert = '${secret:client_cert}',
    tls_client_key = '${secret:client_key}'
);
```

## Performance Optimization

### Batching

```sql
CREATE CONNECTION ws_batched TO websocket (
    endpoint = 'wss://bulk.example.com/stream',
    mode = 'client',
    format = 'json',
    batch_size = 1000,
    batch_timeout_ms = 100,
    compression = 'gzip'
);
```

### Message Compression

```sql
CREATE CONNECTION ws_compressed TO websocket (
    endpoint = 'wss://compressed.example.com/stream',
    mode = 'client',
    format = 'json',
    compression = 'zstd',
    compression_level = 3
);
```

## Monitoring

### Connection Metrics

```sql
-- Monitor WebSocket connections
SELECT 
    connection_name,
    connection_state,
    connected_since,
    messages_received,
    messages_sent,
    bytes_received,
    bytes_sent,
    reconnect_count,
    last_error
FROM system.websocket_connections
WHERE connection_name LIKE 'ws_%';
```

### Message Flow

```sql
-- Analyze message flow
SELECT 
    DATE_TRUNC('minute', received_at) as minute,
    COUNT(*) as message_count,
    AVG(message_size) as avg_size,
    MAX(message_size) as max_size,
    SUM(message_size) as total_bytes
FROM ws_events
WHERE received_at > NOW() - INTERVAL '1 hour'
GROUP BY DATE_TRUNC('minute', received_at)
ORDER BY minute DESC;
```

## Best Practices

1. **Use connection pooling** for high availability
2. **Configure appropriate reconnection** strategies
3. **Implement heartbeat** for connection health
4. **Set reasonable buffer sizes** based on message volume
5. **Use compression** for large messages
6. **Monitor connection state** and metrics
7. **Handle errors gracefully** with DLQs
8. **Use TLS/WSS** for production
9. **Implement backpressure** for flow control
10. **Test failover scenarios** regularly

## Troubleshooting

### Connection Issues

```sql
-- Test WebSocket connection
SELECT test_connection('websocket_client');

-- Check connection logs
SELECT * FROM system.websocket_logs
WHERE connection = 'websocket_client'
ORDER BY timestamp DESC
LIMIT 100;
```

### Performance Analysis

```sql
-- Analyze latency
WITH latency_stats AS (
    SELECT 
        EXTRACT(EPOCH FROM (processed_at - received_at)) * 1000 as latency_ms,
        DATE_TRUNC('minute', received_at) as minute
    FROM ws_events
    WHERE received_at > NOW() - INTERVAL '1 hour'
)
SELECT 
    minute,
    COUNT(*) as messages,
    AVG(latency_ms) as avg_latency,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY latency_ms) as p50,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) as p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) as p99
FROM latency_stats
GROUP BY minute
ORDER BY minute DESC;
```