---
sidebar_position: 3
title: Configuration
---

Laminar provides extensive configuration options to customize the platform for your specific requirements. This guide covers all configuration aspects from basic setup to advanced tuning.

## Configuration Overview

Laminar can be configured through:
- **Environment variables** - Runtime configuration
- **Configuration files** - YAML/TOML configuration
- **Command-line arguments** - Override defaults
- **API calls** - Dynamic configuration
- **Web UI** - Visual configuration

## Configuration Files

### Main Configuration File

The main configuration file (`laminar.yaml`) controls core settings:

```yaml
# laminar.yaml
api:
  host: 0.0.0.0
  port: 8000
  tls:
    enabled: false
  cors:
    enabled: true
    origins: ["*"]
  auth:
    enabled: false
    type: jwt
    
pipeline:
  defaults:
    parallelism: 1
    checkpoint_interval: 60s
    max_retry_attempts: 3
  limits:
    max_parallelism: 16
    max_memory_per_task: 4Gi
  optimization:
    enable_cost_based_optimizer: true
  state:
    backend: rocksdb
    
jobs:
  scheduler:
    type: default
    max_concurrent_jobs: 100
  resources:
    default_memory: 1Gi
    default_cpu: 1
  failure:
    restart_strategy: exponential_backoff
    max_restart_attempts: 5
  checkpoint:
    interval: 60s
    timeout: 10m
    
worker:
  task_manager:
    num_task_slots: 4
    memory_per_slot: 1Gi
  network:
    buffer_size: 32768
  state:
    backend: rocksdb
    cache_size: 256Mi
    
state_storage:
  type: s3
  s3:
    bucket: laminar-state
    region: us-east-1
    endpoint: https://s3.amazonaws.com
    
metadata:
  type: postgresql
  connection:
    host: localhost
    port: 5432
    database: laminar_metadata
    username: laminar
    password: ${DB_PASSWORD}
  pool:
    min_connections: 10
    max_connections: 100
    
security:
  authentication:
    enabled: true
    providers:
      - type: jwt
        secret: ${JWT_SECRET}
  authorization:
    enabled: true
    type: rbac
  encryption:
    tls:
      enabled: true
      min_version: TLS1.2
      
monitoring:
  metrics:
    enabled: true
    reporters:
      - type: prometheus
        port: 9090
  logging:
    level: info
    outputs:
      - type: console
        format: json
  tracing:
    enabled: true
    backend: jaeger
```

### Environment Variables

All configuration values can be overridden using environment variables:

```bash
# API Configuration
export LAMINAR_API_HOST=0.0.0.0
export LAMINAR_API_PORT=5115
export LAMINAR_API_AUTH_ENABLED=true
export LAMINAR_API_AUTH_TYPE=jwt

# Pipeline Configuration
export LAMINAR_PIPELINE_DEFAULTS_PARALLELISM=4
export LAMINAR_PIPELINE_DEFAULTS_CHECKPOINT_INTERVAL=60s
export LAMINAR_PIPELINE_STATE_BACKEND=rocksdb

# Jobs Configuration
export LAMINAR_JOBS_SCHEDULER_TYPE=default
export LAMINAR_JOBS_RESOURCES_DEFAULT_MEMORY=2Gi
export LAMINAR_JOBS_CHECKPOINT_INTERVAL=60s

# Worker Configuration
export LAMINAR_WORKER_TASK_MANAGER_NUM_TASK_SLOTS=4
export LAMINAR_WORKER_STATE_BACKEND=rocksdb

# State Storage Configuration
export LAMINAR_STATE_STORAGE_TYPE=s3
export LAMINAR_STATE_STORAGE_S3_BUCKET=laminar-state
export LAMINAR_STATE_STORAGE_S3_REGION=us-west-2

# Metadata Database Configuration
export LAMINAR_METADATA_TYPE=postgresql
export LAMINAR_METADATA_CONNECTION_HOST=postgres.example.com
export LAMINAR_METADATA_CONNECTION_PORT=5432
export LAMINAR_METADATA_CONNECTION_DATABASE=laminar_metadata
export LAMINAR_METADATA_CONNECTION_USERNAME=admin
export LAMINAR_METADATA_CONNECTION_PASSWORD=secret

# Security Configuration
export LAMINAR_SECURITY_AUTHENTICATION_ENABLED=true
export LAMINAR_SECURITY_ENCRYPTION_TLS_ENABLED=true

# Monitoring Configuration
export LAMINAR_MONITORING_LOGGING_LEVEL=debug
export LAMINAR_MONITORING_METRICS_ENABLED=true
```

## Component Configuration

### API Server

Configure the API server behavior:

```yaml
api:
  # Network settings
  host: 0.0.0.0
  port: 5115
  
  # TLS/SSL
  tls:
    enabled: false
    cert_file: /path/to/cert.pem
    key_file: /path/to/key.pem
    
  # Rate limiting
  rate_limit:
    enabled: true
    requests_per_minute: 1000
    burst: 100
    
  # CORS
  cors:
    enabled: true
    origins: ["*"]
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH"]
    headers: ["Content-Type", "Authorization"]
    
  # Authentication
  auth:
    enabled: false
    type: jwt
    secret: ${JWT_SECRET}
    expiry: 24h
```

### Pipeline Manager

Configure pipeline execution settings:

```yaml
pipeline:
  # Default settings for new pipelines
  defaults:
    parallelism: 1
    checkpoint_interval: 60s
    max_retry_attempts: 3
    
  # Resource limits
  limits:
    max_parallelism: 16
    max_memory_per_task: 4Gi
    max_cpu_per_task: 2
    
  # Optimization
  optimization:
    enable_cost_based_optimizer: true
    enable_predicate_pushdown: true
    enable_projection_pushdown: true
    
  # State management
  state:
    backend: rocksdb
    incremental_checkpoints: true
    compression: snappy
```

### Job Controller

Configure job execution and management:

```yaml
jobs:
  # Scheduling
  scheduler:
    type: default
    max_concurrent_jobs: 100
    scheduling_interval: 1s
    
  # Resource management
  resources:
    default_memory: 1Gi
    default_cpu: 1
    memory_overcommit_ratio: 1.2
    
  # Failure handling
  failure:
    restart_strategy: exponential_backoff
    max_restart_attempts: 5
    restart_delay: 10s
    failure_rate_interval: 5m
    max_failures_per_interval: 3
    
  # Checkpointing
  checkpoint:
    interval: 60s
    timeout: 10m
    min_pause_between_checkpoints: 30s
    max_concurrent_checkpoints: 1
    cleanup_policy: retain_last_n
    cleanup_retain_count: 3
```

### Worker Configuration

Configure worker nodes:

```yaml
worker:
  # Task execution
  task_manager:
    num_task_slots: 4
    memory_per_slot: 1Gi
    cpu_per_slot: 1
    
  # Network
  network:
    buffer_size: 32768
    num_buffers: 2048
    backlog: 128
    
  # State backend
  state:
    backend: rocksdb
    cache_size: 256Mi
    write_buffer_size: 64Mi
    block_cache_size: 128Mi
    
  # Metrics
  metrics:
    enabled: true
    interval: 10s
    reporters: [prometheus, jmx]
```

## Storage Configuration

### State Storage

Configure state backend storage:

```yaml
state_storage:
  # Backend type
  type: s3  # Options: s3, gcs, azure, filesystem
  
  # S3 Configuration
  s3:
    bucket: laminar-state
    region: us-east-1
    endpoint: https://s3.amazonaws.com
    access_key_id: ${AWS_ACCESS_KEY_ID}
    secret_access_key: ${AWS_SECRET_ACCESS_KEY}
    path_prefix: /state
    
  # Filesystem Configuration
  filesystem:
    path: /var/laminar/state
    
  # Performance
  performance:
    multipart_upload_threshold: 100Mi
    multipart_upload_concurrency: 10
    connection_timeout: 30s
    request_timeout: 60s
```

### Metadata Storage

Configure metadata database:

```yaml
metadata:
  # Database type
  type: postgresql  # Options: postgresql, mysql
  
  # Connection settings
  connection:
    host: localhost
    port: 5432
    database: laminar_metadata
    username: laminar
    password: ${DB_PASSWORD}
    
  # Connection pool
  pool:
    min_connections: 10
    max_connections: 100
    connection_timeout: 30s
    idle_timeout: 10m
    max_lifetime: 30m
    
  # Performance
  performance:
    statement_cache_size: 100
    query_timeout: 30s
```

## Security Configuration

### Authentication

```yaml
security:
  authentication:
    enabled: true
    providers:
      - type: jwt
        secret: ${JWT_SECRET}
        algorithm: HS256
        expiry: 24h
      - type: oauth
        provider: google
        client_id: ${OAUTH_CLIENT_ID}
        client_secret: ${OAUTH_CLIENT_SECRET}
      - type: ldap
        url: ldap://ldap.example.com
        base_dn: dc=example,dc=com
        user_dn_template: uid={0},ou=users
```

### Authorization

```yaml
security:
  authorization:
    enabled: true
    type: rbac
    default_role: viewer
    roles:
      - name: admin
        permissions: ["*"]
      - name: developer
        permissions: ["pipeline:*", "job:*", "connection:read"]
      - name: viewer
        permissions: ["*:read"]
```

### Encryption

```yaml
security:
  encryption:
    # Data in transit
    tls:
      enabled: true
      min_version: TLS1.2
      cipher_suites:
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        
    # Data at rest
    at_rest:
      enabled: true
      algorithm: AES256
      key_management: kms
      kms_key_id: ${KMS_KEY_ID}
```


## Monitoring Configuration

### Metrics

```yaml
monitoring:
  metrics:
    enabled: true
    reporters:
      - type: prometheus
        port: 9090
        endpoint: /metrics
      - type: jmx
        port: 9999
      - type: statsd
        host: localhost
        port: 8125
        
    # Metric intervals
    collection_interval: 10s
    reporting_interval: 60s
    
    # Metric filters
    include_patterns: ["laminar.*"]
    exclude_patterns: ["*.debug.*"]
```

### Logging

```yaml
monitoring:
  logging:
    # Log levels: trace, debug, info, warn, error
    level: info
    
    # Output configuration
    outputs:
      - type: console
        format: json
      - type: file
        path: /var/log/laminar/laminar.log
        rotation:
          max_size: 100Mi
          max_age: 7d
          max_backups: 10
      - type: syslog
        host: localhost
        port: 514
        protocol: udp
        
    # Log filtering
    loggers:
      "org.apache": warn
      "io.netty": warn
      "laminar.pipeline": debug
```

### Tracing

```yaml
monitoring:
  tracing:
    enabled: true
    backend: jaeger
    
    # Jaeger configuration
    jaeger:
      endpoint: http://localhost:14268/api/traces
      sampler:
        type: probabilistic
        param: 0.1
      
    # Trace settings
    max_trace_duration: 5m
    max_spans_per_trace: 1000
```



## Next Steps

- Build your [First Pipeline](../tutorials/first-pipeline)
- Explore [Deployment Options](../../deployment/overview)
- Configure [Observability](../../observability/intro)
- Set up [Security](./configuration#security-configuration)