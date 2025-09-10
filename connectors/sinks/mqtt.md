---
sidebar_position: 6
---

# MQTT Sink

The MQTT sink connector enables Laminar to publish processed data to MQTT brokers for IoT devices and real-time messaging systems.

## Overview

The MQTT sink publishes messages to MQTT topics, supporting various QoS levels and message formatting options for IoT and messaging use cases.

## Configuration

### Connection Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `broker` | string | Yes | MQTT broker URL |
| `client_id` | string | No | Client identifier |
| `username` | string | No | Username for authentication |
| `password` | string | No | Password for authentication |
| `clean_session` | boolean | No | Start with clean session (default: true) |

### Publishing Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `topic` | string | No | Default topic (can be overridden per record) |
| `topic_field` | string | No | Field containing dynamic topic |
| `qos` | integer | No | Quality of Service (0, 1, or 2) |
| `retain` | boolean | No | Retain messages (default: false) |

## SQL Example

```sql
CREATE CONNECTION mqtt_sink_conn FROM mqtt_sink
WITH (
    broker = 'tcp://mqtt.broker.com:1883',
    username = 'publisher',
    password = '${MQTT_PASSWORD}',
    clean_session = true
);

INSERT INTO mqtt_output
SELECT 
    device_id,
    CONCAT('devices/', device_id, '/commands') as topic,
    command,
    timestamp
FROM device_commands
WITH (
    connector = 'mqtt_sink',
    connection = 'mqtt_sink_conn',
    topic_field = 'topic',
    qos = 1,
    format = 'json'
);
```

## Dynamic Topics

```sql
INSERT INTO mqtt_dynamic
SELECT 
    CASE 
        WHEN priority = 'high' THEN 'alerts/critical'
        WHEN priority = 'medium' THEN 'alerts/warning'
        ELSE 'alerts/info'
    END as topic,
    alert_data
FROM alerts
WITH (
    connector = 'mqtt_sink',
    connection = 'mqtt_sink_conn',
    topic_field = 'topic'
);
```

## Message Formats

### JSON Messages

```sql
INSERT INTO mqtt_json
SELECT * FROM sensor_data
WITH (
    connector = 'mqtt_sink',
    connection = 'mqtt_sink_conn',
    topic = 'sensors/data',
    format = 'json'
);
```

### Binary Messages

```sql
INSERT INTO mqtt_binary
SELECT 
    device_id,
    encode(payload, 'base64') as payload
FROM binary_data
WITH (
    connector = 'mqtt_sink',
    connection = 'mqtt_sink_conn',
    topic = 'devices/binary',
    format = 'raw_bytes'
);
```

## Best Practices

1. **QoS Selection**: Balance reliability and performance
2. **Topic Design**: Use hierarchical topics for organization
3. **Retention**: Use retained messages for state updates
4. **Batch Publishing**: Group messages for efficiency
5. **Connection Management**: Implement proper reconnection logic