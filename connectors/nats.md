---
sidebar_position: 4
title: NATS
---

The NATS connector enables Laminar to publish and subscribe to NATS messaging system, supporting both NATS Core and JetStream.

## Capabilities

- **Source**: NATS Core and JetStream subscriptions
- **Sink**: NATS Core publishing
- **Formats**: JSON, Raw bytes
- **Authentication**: None, Credentials, JWT with NKey
- **JetStream**: Advanced streaming with persistence and replay

## Configuration

### Connection Profile Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `servers` | string | Yes | - | Comma-separated list of NATS servers (e.g., `nats-1:4222,nats-2:4222`) |
| `authentication` | object | Yes | - | Authentication configuration (see Authentication section) |

### Authentication Options

#### No Authentication
```json
{
  "authentication": {}
}
```

#### Username/Password
```json
{
  "authentication": {
    "username": "user",
    "password": "{{ NATS_PASSWORD }}"
  }
}
```

#### JWT with NKey
```json
{
  "authentication": {
    "jwt": "{{ NATS_JWT }}",
    "nkeySeed": "{{ NATS_NKEY_SEED }}"
  }
}
```

### Table Configuration

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `connectorType` | object | Yes | - | Source or Sink configuration |

#### NATS Core Source

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `subject` | string | Yes | - | NATS subject to subscribe to |

#### NATS JetStream Source

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `stream` | string | Yes | - | JetStream stream name |
| `ackPolicy` | enum | No | `Explicit` | Acknowledgment policy: `Explicit`, `None`, `All` |
| `replayPolicy` | enum | No | `Instant` | Replay policy: `Original`, `Instant` |
| `ackWait` | integer | No | 300 | Seconds to wait for acknowledgment |
| `description` | string | No | - | Consumer description |
| `filterSubjects` | array | No | [] | Subject filters for consumer |
| `sampleFrequency` | integer | No | 0 | Sampling percentage (0-100) |
| `numReplicas` | integer | No | 1 | Number of consumer replicas |
| `inactiveThreshold` | integer | No | 600 | Seconds before inactive consumer cleanup |
| `rateLimit` | integer | No | -1 | Max messages/second (-1 for unlimited) |
| `maxAckPending` | integer | No | -1 | Max unacknowledged messages |
| `maxDeliver` | integer | No | -1 | Max delivery attempts |
| `maxWaiting` | integer | No | 1000000 | Max messages waiting for delivery |
| `maxBatch` | integer | No | 10000 | Max messages per batch |
| `maxBytes` | integer | No | 104857600 | Max bytes per batch |
| `maxExpires` | integer | No | 300000 | Max messages before expiry |

#### NATS Core Sink

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `subject` | string | Yes | - | NATS subject to publish to |

## API Usage

### Create Connection Profile

```bash
curl -X POST http://localhost:8000/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "nats_cluster",
    "connector": "nats",
    "config": {
      "servers": "nats1:4222,nats2:4222,nats3:4222",
      "authentication": {
        "username": "app_user",
        "password": "{{ NATS_PASSWORD }}"
      }
    }
  }'
```

### Create JetStream Source

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "events_stream",
    "connector": "nats",
    "connectionProfileId": "prof_nats123",
    "config": {
      "connectorType": {
        "sourceType": {
          "stream": "EVENTS",
          "ackPolicy": "Explicit",
          "replayPolicy": "Instant",
          "filterSubjects": ["events.>"],
          "maxBatch": 1000
        }
      }
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

### Create Core Sink

```bash
curl -X POST http://localhost:8000/v1/connection_tables \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "notifications",
    "connector": "nats",
    "connectionProfileId": "prof_nats123",
    "config": {
      "connectorType": {
        "sinkType": {
          "subject": "notifications.alerts"
        }
      }
    },
    "schema": {
      "format": {
        "json": {}
      }
    }
  }'
```

## SQL Usage

### NATS Core Source

```sql
CREATE TABLE events (
    id BIGINT,
    event_type TEXT,
    payload JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats',
    servers = 'nats://localhost:4222',
    'auth.type' = 'none',
    'source.type' = 'core',
    'source.subject' = 'events.>',
    type = 'source',
    format = 'json'
);
```

### JetStream Source

```sql
CREATE TABLE jetstream_events (
    id BIGINT,
    data TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats',
    servers = 'nats1:4222,nats2:4222',
    'auth.username' = 'reader',
    'auth.password' = '{{ NATS_PASSWORD }}',
    'source.type' = 'jetstream',
    'source.stream' = 'EVENTS',
    'source.ack_policy' = 'Explicit',
    'source.replay_policy' = 'Original',
    'source.filter_subjects' = 'events.important.*',
    type = 'source',
    format = 'json'
);
```

### NATS Core Sink

```sql
CREATE TABLE alerts (
    alert_id BIGINT,
    severity TEXT,
    message TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats',
    servers = 'nats://localhost:4222',
    'auth.type' = 'none',
    'sink.type' = 'core',
    'sink.subject' = 'alerts.critical',
    type = 'sink',
    format = 'json'
);
```

## Examples

### Event Processing with JetStream

```sql
-- Source: JetStream consumer
CREATE TABLE raw_events (
    event_id BIGINT,
    user_id TEXT,
    action TEXT,
    metadata JSON,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats',
    servers = 'nats://jetstream:4222',
    'source.type' = 'jetstream',
    'source.stream' = 'USER_EVENTS',
    'source.ack_policy' = 'Explicit',
    'source.max_batch' = '5000',
    'source.filter_subjects' = 'user.events.>',
    type = 'source',
    format = 'json'
);

-- Sink: Processed events to NATS Core
CREATE TABLE processed_events (
    event_id BIGINT,
    user_id TEXT,
    event_category TEXT,
    risk_score DOUBLE,
    processed_at TIMESTAMP
) WITH (
    connector = 'nats',
    servers = 'nats://jetstream:4222',
    'sink.type' = 'core',
    'sink.subject' = 'processed.events',
    type = 'sink',
    format = 'json'
);

-- Processing pipeline
INSERT INTO processed_events
SELECT 
    event_id,
    user_id,
    CASE 
        WHEN action LIKE 'login%' THEN 'authentication'
        WHEN action LIKE 'purchase%' THEN 'transaction'
        ELSE 'activity'
    END as event_category,
    CAST(JSON_VALUE(metadata, '$.risk_score') AS DOUBLE) as risk_score,
    NOW() as processed_at
FROM raw_events
WHERE user_id IS NOT NULL;
```

### Subject-Based Routing

```sql
-- Subscribe to wildcard subjects
CREATE TABLE all_metrics (
    subject TEXT,
    service TEXT,
    metric_name TEXT,
    value DOUBLE,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats',
    servers = 'nats://localhost:4222',
    'source.type' = 'core',
    'source.subject' = 'metrics.>',  -- Matches metrics.cpu, metrics.memory, etc.
    type = 'source',
    format = 'json'
);

-- Route to specific subjects based on content
CREATE TABLE critical_alerts (
    service TEXT,
    alert TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'nats',
    servers = 'nats://localhost:4222',
    'sink.type' = 'core',
    'sink.subject' = 'alerts.critical',
    type = 'sink',
    format = 'json'
);

INSERT INTO critical_alerts
SELECT 
    service,
    CONCAT(metric_name, ' is ', value) as alert,
    timestamp
FROM all_metrics
WHERE 
    (metric_name = 'cpu' AND value > 90) OR
    (metric_name = 'memory' AND value > 95);
```

## JetStream Features

### Acknowledgment Policies
- **Explicit**: Each message must be acknowledged individually
- **All**: Acknowledging a message acknowledges all previous messages
- **None**: No acknowledgment required (at-most-once delivery)

### Replay Policies
- **Instant**: Messages delivered as fast as possible
- **Original**: Messages delivered at original publish rate

### Consumer Configuration
JetStream consumers can be fine-tuned with:
- **Rate limiting**: Control message delivery rate
- **Batch size**: Optimize throughput vs latency
- **Replicas**: Ensure high availability
- **Filters**: Subscribe to specific subjects within a stream

## Best Practices

1. **JetStream vs Core**: Use JetStream for persistent, reliable streaming; Core for lightweight pub/sub
2. **Subject Design**: Use hierarchical subjects (e.g., `app.service.event`) for flexible routing
3. **Acknowledgments**: Use `Explicit` for critical data, `None` for metrics/telemetry
4. **Batching**: Tune `maxBatch` and `maxBytes` for optimal throughput
5. **Filtering**: Use `filterSubjects` to reduce network traffic
6. **Authentication**: Use JWT/NKey for production deployments