---
sidebar_position: 2
title: Deployment Overview
---


Laminar is designed for distributed, production-grade deployments that can scale horizontally to handle millions of events per second. This guide covers the various deployment options and key considerations for running Laminar in production.

## Deployment Options

Laminar supports multiple deployment modes to fit different use cases and infrastructure preferences:

### 1. Kubernetes (Recommended for Production)
The preferred deployment method for production environments, offering:
- Automatic scaling and resource management
- Built-in high availability
- Container orchestration
- Easy integration with cloud providers (AWS EKS, Google GKE, Azure AKS)

[Learn more about Kubernetes deployment →](./kubernetes)

### 2. Pipeline Clusters
Lightweight, self-contained clusters for running individual pipelines:
- Ideal for development and testing
- Serverless platform support (AWS Fargate, Google Cloud Run, Fly.io)
- Minimal resource overhead
- Single pipeline execution

[Learn more about pipeline clusters →](./pipeline-clusters)

### 3. Virtual Machines & Bare Metal
Direct deployment on VMs or physical servers:
- Full control over infrastructure
- Custom performance optimizations
- On-premises deployment support
- Requires manual orchestration

[Learn more about VM deployment →](./vm)

### 4. Docker Compose (Development Only)
Quick local setup for development:
- Single command startup
- Pre-configured services
- Not suitable for production
- Local testing and development

## Architecture Components

A complete Laminar deployment consists of several key components:

### Control Plane
- **API Server**: Handles user requests and pipeline management
- **Scheduler**: Manages job scheduling and resource allocation
- **Web UI**: Provides visual interface for monitoring and management

### Data Plane
- **Workers**: Execute pipeline tasks and stream processing
- **Task Managers**: Coordinate task execution across workers
- **State Backend**: Manages stateful computations

### Supporting Services
- **Metadata Database**: Stores pipeline definitions and system metadata
- **Checkpoint Storage**: Persists pipeline state for fault tolerance
- **Artifact Storage**: Stores compiled pipelines and UDF libraries

## System Requirements

### Database

Laminar requires a metadata database for storing pipeline definitions, job history, and system configuration:

#### PostgreSQL (Recommended for Production)
```yaml
database:
  type: postgres
  host: postgres.example.com
  port: 5432
  database: laminar
  username: laminar_user
  password: ${DB_PASSWORD}
```

#### SQLite (Development Only)
```yaml
database:
  type: sqlite
  path: /data/laminar.db
```

### Storage Backend

Centralized storage is required for checkpoints and artifacts. All nodes in the cluster must have access to the same storage backend.

#### Supported Storage Systems

**Object Storage (Recommended)**:
- **Amazon S3**: `s3://bucket-name/path`
- **Google Cloud Storage**: `gs://bucket-name/path`
- **Azure Blob Storage**: `abs://container-name/path`
- **MinIO**: S3-compatible object storage

**Filesystem Storage (Development)**:
- **Local filesystem**: `file:///path/to/storage`
- **Network filesystem**: NFS, GlusterFS, etc.

#### Storage Configuration

Configure storage locations using environment variables:

```bash
LAMINAR_CHECKPOINT_URL=s3://laminar-checkpoints/prod

LAMINAR_ARTIFACT_URL=s3://laminar-artifacts/prod

LAMINAR_S3_ENDPOINT=https://minio.example.com
LAMINAR_S3_REGION=us-east-1
```

### Resource Requirements

#### Minimum Requirements (Development)
- **CPU**: 2 cores
- **Memory**: 4 GB RAM
- **Storage**: 20 GB
- **Network**: 100 Mbps

#### Recommended Production Requirements

**Control Plane (per node)**:
- **CPU**: 4-8 cores
- **Memory**: 8-16 GB RAM
- **Storage**: 50 GB SSD
- **Network**: 1 Gbps

**Worker Nodes (per node)**:
- **CPU**: 8-16 cores
- **Memory**: 32-64 GB RAM
- **Storage**: 100 GB SSD
- **Network**: 10 Gbps

### Network Requirements

Laminar requires the following network connectivity:

| Component | Port | Protocol | Description |
|-----------|------|----------|-------------|
| Web UI | 8000 | HTTP/HTTPS | Web interface |
| API Server | 8001 | HTTP/HTTPS | REST API |
| gRPC API | 9000 | TCP | Internal RPC |
| Metrics | 9090 | HTTP | Prometheus metrics |
| Worker RPC | 6669 | TCP | Worker communication |

## High Availability

For production deployments, consider these high availability configurations:

### Multi-Region Deployment
```yaml
regions:
  primary:
    region: us-east-1
    zones: [us-east-1a, us-east-1b, us-east-1c]
  secondary:
    region: us-west-2
    zones: [us-west-2a, us-west-2b]
```

### Component Redundancy
- **Control Plane**: Run 3+ instances for quorum
- **Workers**: Scale based on workload (N+2 recommended)
- **Database**: Use managed services with automatic failover
- **Storage**: Use replicated object storage

## Security Considerations

### Authentication & Authorization
- **OAuth2/OIDC**: Integration with enterprise identity providers
- **API Keys**: For programmatic access
- **RBAC**: Role-based access control for fine-grained permissions

### Network Security
- **TLS/SSL**: Encrypt all communication
- **Network Policies**: Restrict inter-component communication
- **Private Networks**: Deploy in VPC/private networks
- **Firewall Rules**: Whitelist only necessary ports

### Data Security
- **Encryption at Rest**: Encrypt checkpoint and artifact storage
- **Encryption in Transit**: Use TLS for all network communication
- **Secrets Management**: Use Kubernetes secrets or cloud KMS

## Monitoring & Observability

### Metrics
Laminar exposes Prometheus-compatible metrics:
```yaml
metrics:
  enabled: true
  port: 9090
  path: /metrics
```

### Logging
Configure structured logging:
```yaml
logging:
  level: INFO
  format: json
  outputs:
    - stdout
    - file: /var/log/laminar/laminar.log
```

### Tracing
Enable distributed tracing:
```yaml
tracing:
  enabled: true
  provider: jaeger
  endpoint: http://jaeger:14268/api/traces
```

## Deployment Checklist

Before deploying to production, ensure:

- [ ] **Database**: PostgreSQL configured with backups
- [ ] **Storage**: Object storage configured and accessible
- [ ] **Networking**: All required ports open and secured
- [ ] **TLS/SSL**: Certificates configured for HTTPS
- [ ] **Monitoring**: Metrics, logging, and alerting configured
- [ ] **Backup**: Automated backup strategy in place
- [ ] **Disaster Recovery**: Recovery procedures documented
- [ ] **Security**: Authentication, authorization, and encryption enabled
- [ ] **Scaling**: Auto-scaling policies configured
- [ ] **Documentation**: Runbooks and operational procedures ready

## Next Steps

Choose your deployment method:

- [Deploy on Kubernetes](./kubernetes) - Recommended for production
- [Set up Pipeline Clusters](./pipeline-clusters) - For lightweight deployments
- [Deploy on VMs](./vm) - For custom infrastructure
- [Configure Laminar](./configuration) - Detailed configuration guide