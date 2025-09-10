---
sidebar_position: 2
title: Metrics Reference
---

# Metrics Reference

Laminar exposes comprehensive metrics in Prometheus format, providing deep visibility into pipeline performance, resource utilization, and system health. All metrics are available at the `/metrics` endpoint on port 9090 by default.

## Metrics Endpoint

```
http://<laminar-host>:9090/metrics
```

## Metric Types

Laminar uses standard Prometheus metric types:

- **Counter**: Cumulative value that only increases
- **Gauge**: Value that can go up or down
- **Histogram**: Distribution of values with percentiles
- **Summary**: Similar to histogram with configurable quantiles

## Pipeline Metrics

### Data Flow Metrics

#### `laminar_messages_recv` (Counter)
Count of messages received by each subtask.

**Labels:**
- `node_id`: Node identifier
- `subtask_idx`: Subtask index within the operator
- `operator_name`: Name of the operator

**Example:**
```promql
# Messages per second by operator
rate(laminar_messages_recv[1m])

# Total messages received by pipeline
sum(laminar_messages_recv) by (job_id)
```

#### `laminar_messages_sent` (Counter)
Count of messages sent by each subtask.

**Labels:**
- `node_id`: Node identifier
- `subtask_idx`: Subtask index
- `operator_name`: Operator name

#### `laminar_bytes_recv` (Counter)
Total bytes received by each subtask.

**Labels:**
- `node_id`: Node identifier
- `subtask_idx`: Subtask index
- `operator_name`: Operator name

**Example:**
```promql
# Throughput in MB/s
rate(laminar_bytes_recv[1m]) / 1024 / 1024
```

#### `laminar_bytes_sent` (Counter)
Total bytes sent by each subtask.

**Labels:**
- `node_id`: Node identifier
- `subtask_idx`: Subtask index
- `operator_name`: Operator name

#### `laminar_batches_recv` (Counter)
Number of batches received by each subtask.

**Labels:**
- `node_id`: Node identifier
- `subtask_idx`: Subtask index
- `operator_name`: Operator name

#### `laminar_batches_sent` (Counter)
Number of batches sent by each subtask.

**Labels:**
- `node_id`: Node identifier
- `subtask_idx`: Subtask index
- `operator_name`: Operator name

### Error Metrics

#### `laminar_deserialization_errors` (Counter)
Count of deserialization errors encountered.

**Labels:**
- `node_id`: Node identifier
- `subtask_idx`: Subtask index
- `operator_name`: Operator name
- `error_type`: Type of deserialization error

**Example:**
```promql
# Error rate per operator
rate(laminar_deserialization_errors[5m])

# Alert on high error rate
laminar_deserialization_errors > 100
```

### Queue Metrics

#### `laminar_worker_tx_queue_size` (Gauge)
Current size of transmission queues between operators.

**Labels:**
- `node_id`: Source node
- `subtask_idx`: Source subtask
- `operator_name`: Source operator
- `next_node`: Destination node
- `next_node_idx`: Destination subtask

**Example:**
```promql
# Average queue size
avg(laminar_worker_tx_queue_size) by (operator_name)

# Queue backup detection
laminar_worker_tx_queue_size > 1000
```

#### `laminar_worker_tx_queue_rem` (Gauge)
Remaining capacity in transmission queues.

**Labels:**
- `node_id`: Source node
- `subtask_idx`: Source subtask
- `operator_name`: Source operator
- `next_node`: Destination node
- `next_node_idx`: Destination subtask

#### `laminar_worker_tx_bytes` (Gauge)
Number of bytes currently queued in transmission queues.

**Labels:**
- `node_id`: Source node
- `subtask_idx`: Source subtask
- `operator_name`: Source operator
- `next_node`: Destination node
- `next_node_idx`: Destination subtask

## System Metrics

### Node Metrics

#### `laminar_node_running_workers` (Gauge)
Number of workers currently managed by each node.

**Example:**
```promql
# Total workers in cluster
sum(laminar_node_running_workers)

# Workers per node
laminar_node_running_workers
```

### JVM Metrics (if applicable)

#### `jvm_memory_heap_used` (Gauge)
Heap memory currently used in bytes.

#### `jvm_memory_heap_max` (Gauge)
Maximum heap memory available in bytes.

#### `jvm_gc_collection_seconds` (Summary)
Time spent in garbage collection.

**Labels:**
- `gc`: Garbage collector name (e.g., "G1 Young", "G1 Old")

## Checkpoint Metrics

#### `laminar_checkpoint_duration_seconds` (Histogram)
Duration of checkpoint operations.

**Labels:**
- `job_id`: Job identifier
- `pipeline_id`: Pipeline identifier
- `checkpoint_id`: Checkpoint identifier

**Buckets:** 0.1, 0.5, 1, 5, 10, 30, 60, 120, 300

#### `laminar_checkpoint_size_bytes` (Histogram)
Size of checkpoint data.

**Labels:**
- `job_id`: Job identifier
- `pipeline_id`: Pipeline identifier

**Buckets:** 1KB, 10KB, 100KB, 1MB, 10MB, 100MB, 1GB

#### `laminar_checkpoint_failures` (Counter)
Number of failed checkpoint attempts.

**Labels:**
- `job_id`: Job identifier
- `pipeline_id`: Pipeline identifier
- `reason`: Failure reason

## Connector Metrics

### Kafka Connector

#### `laminar_kafka_consumer_lag` (Gauge)
Consumer lag in number of messages.

**Labels:**
- `topic`: Kafka topic
- `partition`: Partition number
- `consumer_group`: Consumer group ID

#### `laminar_kafka_messages_consumed` (Counter)
Total messages consumed from Kafka.

**Labels:**
- `topic`: Kafka topic
- `partition`: Partition number

#### `laminar_kafka_messages_produced` (Counter)
Total messages produced to Kafka.

**Labels:**
- `topic`: Kafka topic

### Database Connectors

#### `laminar_db_connections_active` (Gauge)
Number of active database connections.

**Labels:**
- `database`: Database name
- `host`: Database host

#### `laminar_db_connection_wait_time_seconds` (Histogram)
Time spent waiting for database connections.

**Labels:**
- `database`: Database name

## Latency Metrics

#### `laminar_processing_latency_seconds` (Histogram)
End-to-end processing latency.

**Labels:**
- `pipeline_id`: Pipeline identifier
- `source_operator`: Source operator
- `sink_operator`: Sink operator

**Buckets:** 0.001, 0.01, 0.1, 0.5, 1, 5, 10, 30, 60

#### `laminar_operator_latency_seconds` (Histogram)
Processing latency per operator.

**Labels:**
- `operator_name`: Operator name
- `operator_type`: Type of operator (source, transform, sink)

## Resource Metrics

#### `laminar_cpu_usage_percent` (Gauge)
CPU usage percentage per worker.

**Labels:**
- `worker_id`: Worker identifier
- `node_id`: Node identifier

#### `laminar_memory_usage_bytes` (Gauge)
Memory usage in bytes per worker.

**Labels:**
- `worker_id`: Worker identifier
- `node_id`: Node identifier

#### `laminar_disk_usage_bytes` (Gauge)
Disk usage for state and checkpoints.

**Labels:**
- `path`: Storage path
- `type`: Usage type (state, checkpoint, temp)

## Watermark Metrics

#### `laminar_watermark` (Gauge)
Current watermark value for each operator.

**Labels:**
- `operator_name`: Operator name
- `subtask_idx`: Subtask index

**Example:**
```promql
# Watermark lag (current time - watermark)
time() - laminar_watermark

# Watermark skew between subtasks
max(laminar_watermark) by (operator_name) - min(laminar_watermark) by (operator_name)
```

## Custom Business Metrics

Laminar allows defining custom metrics in SQL:

```sql
-- Define a custom counter
CREATE METRIC order_value_total AS
SELECT SUM(order_value) as value
FROM orders;

-- Define a custom gauge
CREATE METRIC active_sessions AS
SELECT COUNT(DISTINCT session_id) as value
FROM events
WHERE timestamp > NOW() - INTERVAL '5 minutes';
```

These appear as:
- `laminar_custom_order_value_total` (Counter)
- `laminar_custom_active_sessions` (Gauge)

## Prometheus Query Examples

### Dashboard Queries

#### Pipeline Health
```promql
# Overall pipeline throughput
sum(rate(laminar_messages_sent[5m])) by (pipeline_id)

# Error rate percentage
100 * sum(rate(laminar_deserialization_errors[5m])) / sum(rate(laminar_messages_recv[5m]))

# Average processing latency
histogram_quantile(0.95, laminar_processing_latency_seconds)
```

#### Resource Utilization
```promql
# CPU usage by pipeline
sum(laminar_cpu_usage_percent) by (pipeline_id)

# Memory usage trends
laminar_memory_usage_bytes / 1024 / 1024 / 1024  # GB

# Queue saturation
1 - (laminar_worker_tx_queue_rem / (laminar_worker_tx_queue_rem + laminar_worker_tx_queue_size))
```

#### Backpressure Detection
```promql
# High queue usage
laminar_worker_tx_queue_size / (laminar_worker_tx_queue_size + laminar_worker_tx_queue_rem) > 0.8

# Slow operators (high input rate, low output rate)
rate(laminar_messages_recv[1m]) - rate(laminar_messages_sent[1m]) > 100
```

### Alert Rules

```yaml
groups:
  - name: laminar_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(laminar_deserialization_errors[5m]) > 10
        for: 5m
        annotations:
          summary: "High error rate in pipeline"
          
      - alert: BackpressureDetected
        expr: laminar_worker_tx_queue_size > 5000
        for: 10m
        annotations:
          summary: "Queue backup detected"
          
      - alert: CheckpointFailures
        expr: rate(laminar_checkpoint_failures[15m]) > 0
        for: 5m
        annotations:
          summary: "Checkpoint failures detected"
          
      - alert: HighLatency
        expr: histogram_quantile(0.99, laminar_processing_latency_seconds) > 10
        for: 5m
        annotations:
          summary: "High processing latency"
```

## Grafana Dashboard

Import the Laminar dashboard using ID: `14789` or download from:
```
https://grafana.com/grafana/dashboards/14789
```

Key panels include:
- Pipeline throughput overview
- Error rates and types
- Resource utilization
- Latency percentiles
- Queue depths
- Checkpoint status
- Watermark progression

## Best Practices

1. **Label Cardinality**: Avoid high-cardinality labels that can cause metric explosion
2. **Retention**: Configure appropriate retention policies (typically 15-30 days)
3. **Scrape Interval**: Use 10-30 second intervals for most metrics
4. **Recording Rules**: Pre-compute expensive queries
5. **Alerting**: Set up alerts for critical metrics
6. **Dashboards**: Create role-specific dashboards (ops, dev, business)
7. **Aggregation**: Use appropriate aggregation functions (rate, increase, histogram_quantile)
8. **Federation**: Use Prometheus federation for large deployments

## Metric Export

Metrics can be exported to various backends:

```yaml
# Prometheus remote write
metrics:
  export:
    - type: prometheus_remote_write
      endpoint: https://prometheus.example.com/api/v1/write
      
# Datadog
    - type: datadog
      api_key: ${DATADOG_API_KEY}
      
# CloudWatch
    - type: cloudwatch
      region: us-east-1
      namespace: Laminar/Production
```

## Performance Impact

Metrics collection has minimal performance impact:
- CPU overhead: less than 1%
- Memory overhead: ~10MB per worker
- Network: ~1KB/s per worker

To reduce overhead:
1. Increase scrape intervals
2. Disable unused metrics
3. Use metric sampling
4. Aggregate before export