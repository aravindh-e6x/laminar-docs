---
sidebar_position: 2
title: Kafka Integration
---

# Kafka Integration Tutorial

Apache Kafka is one of the most popular streaming platforms, and Laminar provides first-class support for Kafka as both a source and sink. This tutorial will guide you through building pipelines with Kafka.

## Prerequisites

- Access to a Laminar workspace
- Your Laminar API credentials
- Access to a Kafka cluster with:
  - Broker addresses
  - Authentication credentials (if required)
  - Topic permissions
- Basic understanding of Kafka concepts (topics, partitions, consumer groups)

## Set Up Your Environment

Configure your Laminar API access:

```bash
# Set your workspace URL and API key
export LAMINAR_API_URL="https://your-workspace.laminar.cloud/api/v1"
export LAMINAR_API_KEY="your-api-key-here"
```

## Creating Kafka Connection Profile

### Basic Kafka Connection

For a Kafka cluster without authentication:

```bash
curl -X POST $LAMINAR_API_URL/connection_profiles \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "kafka-cluster",
    "connector": "kafka",
    "description": "My Kafka cluster",
    "config": {
      "bootstrap_servers": "broker1.example.com:9092,broker2.example.com:9092"
    }
  }'
```

### Secured Kafka Connection (SASL/SSL)

```bash
curl -X POST $LAMINAR_API_URL/connection_profiles \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "kafka-production",
    "connector": "kafka",
    "description": "Production Kafka cluster with SASL/SSL",
    "config": {
      "bootstrap_servers": "broker1.example.com:9093,broker2.example.com:9093",
      "security_protocol": "SASL_SSL",
      "sasl_mechanism": "PLAIN",
      "sasl_username": "user",
      "sasl_password": "password",
      "ssl_ca_location": "/path/to/ca-cert",
      "ssl_certificate_location": "/path/to/client-cert",
      "ssl_key_location": "/path/to/client-key"
    }
  }'
```

## Preparing Kafka Topics

Ensure your Kafka cluster has the required topics created. You can create them using your Kafka management tools or the Kafka CLI:

```bash
# Example: Create topics using Kafka CLI (adjust broker address)
kafka-topics --create \
  --topic user-events \
  --bootstrap-server your-broker:9092 \
  --partitions 3 \
  --replication-factor 2

kafka-topics --create \
  --topic user-aggregates \
  --bootstrap-server your-broker:9092 \
  --partitions 3 \
  --replication-factor 2
```

## Kafka Source Configuration

### JSON Format Source

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user_events",
    "connector": "kafka",
    "connection_profile_id": "kafka-cluster",
    "table_type": "source",
    "schema": {
      "fields": [
        {"name": "user_id", "data_type": "BIGINT", "nullable": false},
        {"name": "event_type", "data_type": "STRING", "nullable": false},
        {"name": "product_id", "data_type": "STRING", "nullable": true},
        {"name": "quantity", "data_type": "INT", "nullable": true},
        {"name": "price", "data_type": "DOUBLE", "nullable": true},
        {"name": "event_time", "data_type": "TIMESTAMP", "nullable": false}
      ]
    },
    "config": {
      "topic": "user-events",
      "format": "json",
      "consumer_group": "laminar-consumer",
      "offset_reset": "earliest",
      "properties": {
        "max.poll.records": "500",
        "session.timeout.ms": "30000"
      }
    },
    "event_time_field": "event_time",
    "watermark_delay": "10 seconds"
  }'
```

### Avro Format Source

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user_events_avro",
    "connector": "kafka",
    "connection_profile_id": "kafka-cluster",
    "table_type": "source",
    "schema": {
      "fields": [
        {"name": "user_id", "data_type": "BIGINT", "nullable": false},
        {"name": "event_type", "data_type": "STRING", "nullable": false},
        {"name": "event_time", "data_type": "TIMESTAMP", "nullable": false}
      ]
    },
    "config": {
      "topic": "user-events-avro",
      "format": "avro",
      "format_options": {
        "schema_registry_url": "https://schema-registry.example.com",
        "schema_subject": "user-events-value"
      },
      "consumer_group": "laminar-avro-consumer"
    }
  }'
```

## Kafka Sink Configuration

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user_aggregates_sink",
    "connector": "kafka",
    "connection_profile_id": "kafka-cluster",
    "table_type": "sink",
    "schema": {
      "fields": [
        {"name": "user_id", "data_type": "BIGINT", "nullable": false},
        {"name": "total_orders", "data_type": "BIGINT", "nullable": false},
        {"name": "total_revenue", "data_type": "DOUBLE", "nullable": false},
        {"name": "last_order_time", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "window_start", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "window_end", "data_type": "TIMESTAMP", "nullable": false}
      ]
    },
    "config": {
      "topic": "user-aggregates",
      "format": "json",
      "compression_type": "gzip",
      "batch_size": 1000,
      "linger_ms": 100,
      "properties": {
        "acks": "all",
        "retries": "3",
        "max.in.flight.requests.per.connection": "5"
      }
    }
  }'
```

## Building Kafka Pipelines

### Real-time Aggregation Pipeline

Create a pipeline that aggregates user events in real-time:

```sql
INSERT INTO user_aggregates_sink
SELECT 
    user_id,
    COUNT(*) as total_orders,
    SUM(price * quantity) as total_revenue,
    MAX(event_time) as last_order_time,
    TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start,
    TUMBLE_END(event_time, INTERVAL '5' MINUTE) as window_end
FROM user_events
WHERE event_type = 'purchase'
GROUP BY 
    user_id,
    TUMBLE(event_time, INTERVAL '5' MINUTE)
```

Deploy the pipeline:

```bash
curl -X POST $LAMINAR_API_URL/pipelines \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "kafka_aggregation_pipeline",
    "query": "INSERT INTO user_aggregates_sink SELECT user_id, COUNT(*) as total_orders, SUM(price * quantity) as total_revenue, MAX(event_time) as last_order_time, TUMBLE_START(event_time, INTERVAL '\''5'\'' MINUTE) as window_start, TUMBLE_END(event_time, INTERVAL '\''5'\'' MINUTE) as window_end FROM user_events WHERE event_type = '\''purchase'\'' GROUP BY user_id, TUMBLE(event_time, INTERVAL '\''5'\'' MINUTE)",
    "parallelism": 3
  }'
```

### Stream-to-Stream Join

Join two Kafka streams:

```sql
-- First, create another source table for product data
CREATE TABLE products (
    product_id STRING,
    product_name STRING,
    category STRING,
    base_price DOUBLE
) WITH (
    'connector' = 'kafka',
    'topic' = 'products',
    'format' = 'json'
);

-- Join user events with product data
INSERT INTO enriched_events
SELECT 
    e.user_id,
    e.event_type,
    e.product_id,
    p.product_name,
    p.category,
    e.quantity,
    e.price as sale_price,
    p.base_price,
    (e.price - p.base_price) * e.quantity as profit,
    e.event_time
FROM user_events e
INNER JOIN products p
ON e.product_id = p.product_id
```

### Deduplication Pipeline

Remove duplicate events based on a unique key:

```sql
INSERT INTO deduplicated_events
SELECT 
    user_id,
    event_type,
    product_id,
    quantity,
    price,
    event_time
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, product_id 
            ORDER BY event_time DESC
        ) as rn
    FROM user_events
) WHERE rn = 1
```

## Producing Test Data to Kafka

### Using Laminar's Test Data Generator

Laminar provides a built-in test data generator for Kafka sources:

```bash
curl -X POST $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline/test-data \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "user_events",
    "count": 100,
    "interval_ms": 100
  }'
```

### Using Python Producer

Alternatively, produce data directly to your Kafka cluster:

```python
from kafka import KafkaProducer
import json
from datetime import datetime
import random
import time

# Create producer
producer = KafkaProducer(
    bootstrap_servers=['broker1.example.com:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Generate and send events
for i in range(100):
    event = {
        'user_id': random.randint(1, 10),
        'event_type': random.choice(['view', 'purchase', 'cart_add']),
        'product_id': f'P{random.randint(1, 50):03d}',
        'quantity': random.randint(1, 5),
        'price': round(random.uniform(10, 200), 2),
        'event_time': datetime.utcnow().isoformat() + 'Z'
    }
    
    producer.send('user-events', value=event)
    print(f"Sent: {event}")
    time.sleep(0.1)

producer.flush()
producer.close()
```

## Consuming Results from Kafka

### Monitor Through Laminar Console

You can monitor the output directly in the Laminar Console:

```bash
# Get pipeline output preview
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline/preview?limit=10
```

### Using Python Consumer

Consume messages directly from your Kafka cluster:

```python
from kafka import KafkaConsumer
import json

# Create consumer
consumer = KafkaConsumer(
    'user-aggregates',
    bootstrap_servers=['broker1.example.com:9092'],
    auto_offset_reset='earliest',
    enable_auto_commit=True,
    group_id='result-reader',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Read messages
for message in consumer:
    print(f"Partition: {message.partition}, Offset: {message.offset}")
    print(f"Value: {message.value}")
    print("---")
```

## Advanced Kafka Features

### Exactly-Once Semantics

Configure exactly-once processing:

```bash
curl -X POST $LAMINAR_API_URL/pipelines \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "exactly_once_pipeline",
    "query": "INSERT INTO output SELECT * FROM input",
    "config": {
      "execution.checkpointing.mode": "EXACTLY_ONCE",
      "execution.checkpointing.interval": "10s",
      "kafka.transaction.timeout.ms": "900000"
    }
  }'
```

### Kafka Headers

Access Kafka headers in your pipeline:

```sql
SELECT 
    user_id,
    event_type,
    GET_HEADER('correlation_id') as correlation_id,
    GET_HEADER('source_system') as source_system,
    event_time
FROM user_events
```

### Dynamic Topic Routing

Route messages to different topics based on content:

```sql
-- Create multiple sink tables for different topics
CREATE TABLE high_value_orders WITH (
    'connector' = 'kafka',
    'topic' = 'high-value-orders'
);

CREATE TABLE regular_orders WITH (
    'connector' = 'kafka',
    'topic' = 'regular-orders'
);

-- Route based on order value
INSERT INTO high_value_orders
SELECT * FROM user_events
WHERE event_type = 'purchase' AND price * quantity > 1000;

INSERT INTO regular_orders
SELECT * FROM user_events
WHERE event_type = 'purchase' AND price * quantity <= 1000;
```

## Monitoring Kafka Pipelines

### Monitor Pipeline Metrics

Laminar provides built-in monitoring for Kafka pipelines:

```bash
# Get pipeline metrics
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline/jobs/{job_id}/metrics

# Check Kafka-specific metrics including consumer lag
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline/jobs/{job_id}/metrics?filter=kafka

# Get consumer group information
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline/consumer-groups
```

## Performance Tuning

### Source Optimization

```json
{
  "config": {
    "properties": {
      "fetch.min.bytes": "1048576",
      "fetch.max.wait.ms": "500",
      "max.partition.fetch.bytes": "10485760",
      "max.poll.records": "1000"
    }
  }
}
```

### Sink Optimization

```json
{
  "config": {
    "batch_size": 10000,
    "linger_ms": 1000,
    "buffer_memory": "33554432",
    "compression_type": "snappy",
    "properties": {
      "max.in.flight.requests.per.connection": "5",
      "retries": "2147483647",
      "retry.backoff.ms": "100"
    }
  }
}
```

### Parallelism Configuration

Match pipeline parallelism to your Kafka topic partitions for optimal performance:

```bash
# Get topic information from Laminar
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/connection_tables/user_events/topic-info

# Set pipeline parallelism to match partition count
curl -X PATCH $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"parallelism": 3}'
```

## Monitoring and Debugging

### Pipeline Health Check

```bash
# Check pipeline status
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline/health

# View pipeline logs
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline/logs?lines=100

# Get connection diagnostics
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/connection_profiles/kafka-cluster/test
```

## Clean Up

```bash
# Stop pipeline
curl -X PATCH $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"stop": "immediate"}'

# Delete pipeline
curl -X DELETE $LAMINAR_API_URL/pipelines/kafka_aggregation_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY"

# Delete connection tables
curl -X DELETE $LAMINAR_API_URL/connection_tables/user_events \
  -H "Authorization: Bearer $LAMINAR_API_KEY"
  
curl -X DELETE $LAMINAR_API_URL/connection_tables/user_aggregates_sink \
  -H "Authorization: Bearer $LAMINAR_API_KEY"

# Delete connection profile
curl -X DELETE $LAMINAR_API_URL/connection_profiles/kafka-cluster \
  -H "Authorization: Bearer $LAMINAR_API_KEY"
```

## Next Steps

- Explore [Change Data Capture](./cdc) - Implement CDC pipelines
- Learn about [Apache Iceberg](./iceberg) - Build data lakehouse pipelines
- Review [Kafka Connector Reference](../../connectors/sources/kafka) - Detailed configuration options
- Monitor with [Observability Tools](../../observability/intro) - Track pipeline performance