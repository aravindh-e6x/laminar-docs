---
sidebar_position: 5
title: WebSocket
---

The WebSocket connector enables Laminar to connect to WebSocket servers for real-time bidirectional communication.

## Capabilities

- **Source**: Yes (receive messages from WebSocket server)
- **Sink**: No
- **Formats**: JSON, Raw text
- **Protocols**: WS, WSS (secure)
- **Authentication**: Via headers

## Configuration

### Table Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `endpoint` | string | Yes | - | WebSocket endpoint URL (e.g., `wss://example.com:8080/ws`) |
| `headers` | string | No | - | Comma-separated list of headers (supports env vars) |
| `subscription_messages` | array | No | - | Messages to send after connection (max 2048 chars each) |

Note: WebSocket connector does not require a connection profile.

## API Usage

### Create WebSocket Source

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "market_data",
    "connector": "websocket",
    "config": {
      "endpoint": "wss://stream.example.com/v1/market",
      "headers": "Authorization: Bearer {{ WS_TOKEN }}",
      "subscription_messages": [
        "{\"type\":\"subscribe\",\"channels\":[\"trades\",\"orderbook\"]}",
        "{\"type\":\"heartbeat\",\"interval\":30}"
      ]
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

## SQL Usage

### Basic WebSocket Source

```sql
CREATE TABLE websocket_stream (
    id BIGINT,
    message TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'websocket',
    endpoint = 'ws://localhost:8080/stream',
    type = 'source',
    format = 'json'
);
```

### With Authentication and Subscription

```sql
CREATE TABLE market_data (
    symbol TEXT,
    price DOUBLE,
    volume BIGINT,
    timestamp TIMESTAMP
) WITH (
    connector = 'websocket',
    endpoint = 'wss://api.exchange.com/ws',
    headers = 'Authorization: Bearer {{ API_TOKEN }},X-Client-ID: laminar',
    subscription_messages = '["{\"op\":\"subscribe\",\"args\":[\"ticker:BTC\",\"ticker:ETH\"]}"]',
    type = 'source',
    format = 'json'
);
```

## Examples

### Financial Market Data

```sql
-- Connect to market data feed
CREATE TABLE trades (
    trade_id BIGINT,
    symbol TEXT,
    price DECIMAL,
    quantity DECIMAL,
    side TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'websocket',
    endpoint = 'wss://data.exchange.com/v2/stream',
    headers = 'X-API-Key: {{ EXCHANGE_API_KEY }}',
    subscription_messages = '[
        "{\"action\":\"auth\",\"key\":\"{{ EXCHANGE_API_KEY }}\"}",
        "{\"action\":\"subscribe\",\"trades\":[\"*\"]}"
    ]',
    type = 'source',
    format = 'json'
);

-- Calculate volume-weighted average price (VWAP)
CREATE VIEW vwap AS
SELECT 
    symbol,
    SUM(price * quantity) / SUM(quantity) as vwap,
    SUM(quantity) as total_volume,
    COUNT(*) as trade_count,
    TUMBLE(INTERVAL '1 MINUTE') as window
FROM trades
GROUP BY symbol, window;
```

### Real-time Chat Application

```sql
-- Connect to chat WebSocket
CREATE TABLE chat_messages (
    room_id TEXT,
    user_id TEXT,
    message TEXT,
    message_type TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'websocket',
    endpoint = 'wss://chat.app.com/ws',
    headers = 'Authorization: Bearer {{ CHAT_TOKEN }}',
    subscription_messages = '[
        "{\"type\":\"join\",\"rooms\":[\"general\",\"tech\",\"random\"]}"
    ]',
    type = 'source',
    format = 'json'
);

-- Track active users per room
CREATE VIEW active_users AS
SELECT 
    room_id,
    COUNT(DISTINCT user_id) as active_users,
    HOP(INTERVAL '30 SECONDS', INTERVAL '5 MINUTES') as window
FROM chat_messages
GROUP BY room_id, window;
```

### IoT Sensor Stream

```sql
-- Connect to IoT WebSocket endpoint
CREATE TABLE sensor_readings (
    device_id TEXT,
    sensor_type TEXT,
    value DOUBLE,
    unit TEXT,
    location JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'websocket',
    endpoint = 'wss://iot.platform.com/stream',
    headers = 'X-Device-Token: {{ IOT_TOKEN }},X-Protocol-Version: 2.0',
    subscription_messages = '[
        "{\"command\":\"subscribe\",\"filter\":{\"type\":\"temperature\"}}"
    ]',
    type = 'source',
    format = 'json'
);

-- Detect anomalies
CREATE VIEW anomalies AS
SELECT 
    device_id,
    sensor_type,
    value,
    AVG(value) OVER (
        PARTITION BY device_id 
        ORDER BY timestamp 
        RANGE INTERVAL '1 HOUR' PRECEDING
    ) as avg_value,
    timestamp
FROM sensor_readings
WHERE ABS(value - avg_value) > 3 * STDDEV(value) OVER (
    PARTITION BY device_id 
    ORDER BY timestamp 
    RANGE INTERVAL '1 HOUR' PRECEDING
);
```

### Blockchain Events

```sql
-- Connect to blockchain event stream
CREATE TABLE blockchain_events (
    block_number BIGINT,
    transaction_hash TEXT,
    event_type TEXT,
    contract_address TEXT,
    data JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'websocket',
    endpoint = 'wss://eth.node.com/ws',
    subscription_messages = '[
        "{\"jsonrpc\":\"2.0\",\"method\":\"eth_subscribe\",\"params\":[\"logs\",{\"topics\":[]}],\"id\":1}"
    ]',
    type = 'source',
    format = 'json'
);

-- Track contract activity
CREATE VIEW contract_activity AS
SELECT 
    contract_address,
    event_type,
    COUNT(*) as event_count,
    TUMBLE(INTERVAL '1 HOUR') as window
FROM blockchain_events
GROUP BY contract_address, event_type, window;
```

## Subscription Messages

The `subscription_messages` field allows sending initialization messages after connection:

1. **Authentication**: Send auth tokens or API keys
2. **Channel Subscription**: Subscribe to specific data channels
3. **Configuration**: Set connection parameters
4. **Heartbeat**: Configure keep-alive messages

Example subscription flow:
```json
[
  "{\"type\":\"auth\",\"token\":\"secret\"}",
  "{\"type\":\"subscribe\",\"channels\":[\"data\",\"events\"]}",
  "{\"type\":\"config\",\"heartbeat\":true,\"compression\":\"gzip\"}"
]
```

## Headers Format

Headers should be comma-separated key:value pairs:
```
Authorization: Bearer token123,X-Client-Version: 1.0,Accept: application/json
```

## Best Practices

1. **Use WSS**: Always use secure WebSocket (wss://) in production
2. **Authentication**: Store credentials in environment variables
3. **Reconnection**: WebSocket source automatically reconnects on disconnection
4. **Message Size**: Keep subscription messages under 2048 characters
5. **Heartbeat**: Configure heartbeat/ping messages to keep connection alive
6. **Error Handling**: Monitor for connection drops and invalid messages

## Limitations

- Source only (cannot send messages back to WebSocket server)
- No support for binary WebSocket frames (text only)
- Cannot send messages after initial subscription
- Single WebSocket connection per table