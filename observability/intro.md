---
sidebar_position: 1
---

# Observability Introduction

Comprehensive observability is crucial for operating Laminar in production. This section covers metrics, logging, tracing, and alerting.

## Overview

Laminar provides built-in observability features based on the three pillars:

- **Metrics**: Quantitative measurements of system behavior
- **Logging**: Detailed event records for debugging
- **Tracing**: Request flow through the distributed system

## Observability Stack

### Metrics
- Prometheus-compatible metrics endpoint
- Pre-configured Grafana dashboards
- Custom metrics API

### Logging
- Structured JSON logging
- Log aggregation support
- Configurable log levels

### Tracing
- OpenTelemetry integration
- Distributed trace correlation
- Performance profiling

### Alerting
- Prometheus AlertManager integration
- Configurable alert rules
- Multiple notification channels

## Key Metrics

### Pipeline Metrics
- **Throughput**: Events processed per second
- **Latency**: End-to-end processing time
- **Backpressure**: Queue depths and wait times
- **Error Rate**: Failed events and retries

### System Metrics
- **Resource Usage**: CPU, memory, disk, network
- **JVM Metrics**: Heap usage, GC activity
- **Connection Pools**: Active connections, wait times

### Business Metrics
- **Data Quality**: Schema violations, null values
- **SLA Compliance**: Latency percentiles
- **Cost Metrics**: Resource consumption

## Getting Started

### Enable Metrics Collection

```yaml
# config.yaml
observability:
  metrics:
    enabled: true
    port: 9090
    interval: 10s
```

### Configure Logging

```yaml
logging:
  level: INFO
  format: json
  outputs:
    - console
    - file: /var/log/laminar/laminar.log
```

### Setup Tracing

```yaml
tracing:
  enabled: true
  sampler: adaptive
  sample_rate: 0.1
  exporter:
    type: otlp
    endpoint: http://jaeger:4317
```

## Monitoring Dashboard

Laminar includes pre-built dashboards for:

- Pipeline performance overview
- Resource utilization
- Error analysis
- Latency breakdown
- Checkpoint progress

## Best Practices

### Metrics
1. Use consistent naming conventions
2. Add meaningful labels
3. Avoid high-cardinality metrics
4. Set up recording rules for complex queries

### Logging
1. Use structured logging
2. Include correlation IDs
3. Log at appropriate levels
4. Implement log rotation

### Tracing
1. Sample appropriately for production
2. Add custom spans for business logic
3. Include relevant context in spans
4. Set up trace retention policies

### Alerting
1. Alert on symptoms, not causes
2. Include runbook links
3. Test alert rules regularly
4. Implement escalation policies

## Integration Examples

### Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'laminar'
    static_configs:
      - targets: ['laminar-api:9090']
    metrics_path: '/metrics'
```

### Grafana

```json
{
  "datasources": [
    {
      "name": "Laminar Metrics",
      "type": "prometheus",
      "url": "http://prometheus:9090"
    }
  ]
}
```

### Jaeger

```yaml
# docker-compose.yml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "4317:4317"
```

## Troubleshooting Guide

Common observability issues:

1. **Missing Metrics**
   - Check metrics endpoint accessibility
   - Verify Prometheus scrape configuration

2. **Log Volume Too High**
   - Adjust log levels
   - Implement sampling

3. **Trace Gaps**
   - Check trace propagation
   - Verify sampling configuration

## Next Steps

- [Metrics Setup](./metrics/prometheus) - Prometheus configuration
- [Logging Configuration](./logging/setup) - Detailed logging setup
- [Distributed Tracing](./tracing/opentelemetry) - OpenTelemetry guide
- [Alert Rules](./alerting/rules) - Creating effective alerts