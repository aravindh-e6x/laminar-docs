---
sidebar_position: 2
---

# Connectors Overview

This page provides a comprehensive overview of all available connectors in Laminar.

## Available Connectors

### Source Connectors

| Connector | Type | Description | Status |
|-----------|------|-------------|--------|
| Kafka | Streaming | Apache Kafka consumer | Production |
| MySQL CDC | CDC | MySQL Change Data Capture | Production |
| PostgreSQL CDC | CDC | PostgreSQL Change Data Capture | Production |
| Kinesis | Streaming | AWS Kinesis consumer | Production |
| WebSocket | Streaming | WebSocket client | Production |
| Webhook | HTTP | HTTP webhook receiver | Production |

### Sink Connectors

| Connector | Type | Description | Status |
|-----------|------|-------------|--------|
| Kafka | Streaming | Apache Kafka producer | Production |
| PostgreSQL | Database | PostgreSQL writer | Production |
| Redis | Cache | Redis writer | Production |  
| Delta Lake | Data Lake | Delta Lake writer | Production |
| Iceberg | Data Lake | Apache Iceberg writer | Beta |

## Connector Configuration

All connectors share common configuration patterns:

```sql
CREATE CONNECTION my_connection TO kafka (
    bootstrap_servers = 'localhost:9092',
    topic = 'my-topic',
    format = 'json'
);
```

## Connection Management

Laminar provides several ways to manage connections:

- **Connection Profiles**: Reusable connection configurations
- **Connection Tables**: Schema definitions for connections
- **Connection Testing**: Validate connections before use

## Best Practices

1. **Use connection profiles** for shared configurations
2. **Test connections** before deploying pipelines
3. **Monitor connector metrics** for performance
4. **Configure appropriate backpressure** for streaming sources
5. **Set proper error handling** policies