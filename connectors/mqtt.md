---
sidebar_position: 2
title: MQTT
---

The MQTT connector enables Laminar to publish and subscribe to MQTT brokers for IoT and real-time messaging applications.

## Capabilities

- **Source**: Subscribe to MQTT topics
- **Sink**: Publish to MQTT topics  
- **Formats**: JSON, Raw bytes
- **QoS Levels**: AtMostOnce, AtLeastOnce, ExactlyOnce
- **TLS Support**: Including client certificates

## Configuration

### Connection Profile Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `url` | string | Yes | - | Broker URL (e.g., `tcp://localhost:1883`, `mqtts://broker:8883`) |
| `username` | string | No | - | Username for authentication (supports env vars) |
| `password` | string | No | - | Password for authentication (supports env vars) |
| `clientPrefix` | string | No | `arroyo-mqtt` | Prefix for client ID generation |
| `maxPacketSize` | integer | No | 10240 | Maximum MQTT packet size in bytes (0-4294967295) |
| `tls` | object | No | - | TLS configuration (see TLS section) |

### TLS Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `ca` | string | No | - | Path to CA certificate file |
| `cert` | string | No | - | Path to client certificate file |
| `key` | string | No | - | Path to client key file |

### Table Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `topic` | string | Yes | - | MQTT topic to publish/subscribe |
| `qos` | enum | No | AtMostOnce | Quality of Service level |
| `type` | object | Yes | - | Table type (Source or Sink) |

#### Source Configuration

No additional fields for source.

#### Sink Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `retain` | boolean | Yes | - | Whether to retain messages |

## API Usage

### Create Connection Profile

```bash
curl -X POST http://localhost:8000/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "iot_broker",
    "connector": "mqtt",
    "config": {
      "url": "mqtts://mqtt.broker.com:8883",
      "username": "device_client",
      "password": "{{ MQTT_PASSWORD }}",
      "clientPrefix": "laminar",
      "tls": {
        "ca": "/certs/ca.pem",
        "cert": "/certs/client.pem",
        "key": "/certs/client.key"
      }
    }
  }'
```

### Create Source Table

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "sensor_data",
    "connector": "mqtt",
    "connectionProfileId": "prof_mqtt123",
    "config": {
      "topic": "sensors/+/temperature",
      "type": {},
      "qos": "AtLeastOnce"
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

### Create Sink Table

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "alert_publisher",
    "connector": "mqtt",
    "connectionProfileId": "prof_mqtt123",
    "config": {
      "topic": "alerts/critical",
      "type": {
        "retain": true
      },
      "qos": "ExactlyOnce"
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

## SQL Usage

### Simple Source

```sql
CREATE TABLE sensor_readings (
    sensor_id TEXT,
    temperature DOUBLE,
    humidity DOUBLE,
    timestamp TIMESTAMP
) WITH (
    connector = 'mqtt',
    url = 'tcp://localhost:1883',
    topic = 'sensors/+/data',
    type = 'source',
    format = 'json',
    qos = 'AtLeastOnce'
);
```

### Authenticated Source

```sql
CREATE TABLE secure_sensors (
    device_id TEXT,
    value DOUBLE,
    timestamp TIMESTAMP
) WITH (
    connector = 'mqtt',
    url = 'mqtts://broker.iot.com:8883',
    username = 'reader',
    password = '{{ MQTT_PASSWORD }}',
    topic = 'devices/+/telemetry',
    type = 'source',
    format = 'json',
    'tls.ca' = '/certs/ca.pem'
);
```

### Sink with Retain

```sql
CREATE TABLE status_updates (
    device_id TEXT,
    status TEXT,
    last_seen TIMESTAMP
) WITH (
    connector = 'mqtt',
    url = 'tcp://localhost:1883',
    topic = 'devices/status',
    type = 'sink',
    format = 'json',
    qos = 'AtLeastOnce',
    'sink.retain' = 'true'
);
```

## Examples

### IoT Sensor Processing

```sql
-- Source: Raw sensor data
CREATE TABLE raw_sensors (
    sensor_id TEXT,
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    timestamp TIMESTAMP
) WITH (
    connector = 'mqtt',
    url = 'tcp://mqtt.local:1883',
    topic = 'sensors/+/raw',
    type = 'source',
    format = 'json',
    qos = 'AtLeastOnce'
);

-- Sink: Processed alerts
CREATE TABLE alerts (
    sensor_id TEXT,
    alert_type TEXT,
    value DOUBLE,
    threshold DOUBLE,
    timestamp TIMESTAMP
) WITH (
    connector = 'mqtt',
    url = 'tcp://mqtt.local:1883',
    topic = 'alerts/sensors',
    type = 'sink',
    format = 'json',
    qos = 'ExactlyOnce',
    'sink.retain' = 'false'
);

-- Processing: Detect anomalies
INSERT INTO alerts
SELECT 
    sensor_id,
    'HIGH_TEMPERATURE' as alert_type,
    temperature as value,
    40.0 as threshold,
    timestamp
FROM raw_sensors
WHERE temperature > 40.0;
```

### Multi-Level Topic Subscription

```sql
-- Subscribe to hierarchical topics
CREATE TABLE all_devices (
    topic TEXT,
    device_type TEXT,
    device_id TEXT,
    payload JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'mqtt',
    url = 'tcp://broker:1883',
    topic = 'devices/+/+/status',  -- e.g., devices/sensor/123/status
    type = 'source',
    format = 'json'
);

-- Extract topic levels
CREATE VIEW device_status AS
SELECT 
    SPLIT_PART(topic, '/', 2) as device_type,
    SPLIT_PART(topic, '/', 3) as device_id,
    JSON_VALUE(payload, '$.online') as online,
    JSON_VALUE(payload, '$.battery') as battery,
    timestamp
FROM all_devices;
```

## QoS Levels

- **AtMostOnce (0)**: Fire-and-forget, no acknowledgment
- **AtLeastOnce (1)**: Acknowledged delivery, possible duplicates
- **ExactlyOnce (2)**: Guaranteed single delivery

## Best Practices

1. **Topic Design**: Use hierarchical topics for better organization (e.g., `location/device/metric`)
2. **QoS Selection**: Balance reliability vs performance based on data criticality
3. **Retained Messages**: Use retain for status/configuration that new subscribers need
4. **Client ID**: Set meaningful `clientPrefix` for debugging
5. **TLS**: Always use TLS for production deployments
6. **Wildcards**: Use `+` for single-level and `#` for multi-level subscriptions