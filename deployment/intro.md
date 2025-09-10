---
sidebar_position: 1
---


This section covers everything you need to know about deploying Laminar in production environments.

## Deployment Options

Laminar supports multiple deployment models:

### Kubernetes
The recommended deployment method for production:
- Horizontal scaling
- High availability
- Resource management
- Service discovery

### Docker
For development and small deployments:
- Docker Compose for local development
- Single-node deployments
- Testing environments

### Cloud Providers
Native integration with major cloud platforms:
- AWS (EKS, EC2, S3)
- Google Cloud (GKE, GCS)
- Azure (AKS, Blob Storage)

## Architecture Overview

A typical Laminar deployment consists of:

- **Control Plane**: API server, scheduler, metadata store
- **Data Plane**: Worker nodes for pipeline execution
- **Storage Layer**: Checkpoints, state, and data storage
- **Monitoring Stack**: Metrics, logs, and traces

## System Requirements

### Minimum Requirements
- CPU: 4 cores
- Memory: 8 GB RAM
- Disk: 50 GB SSD
- Network: 1 Gbps

### Recommended Production Setup
- CPU: 16+ cores per node
- Memory: 32+ GB RAM per node
- Disk: 500 GB+ SSD with high IOPS
- Network: 10 Gbps

## Planning Your Deployment

### Sizing Considerations

1. **Data Volume**: Expected throughput (events/second)
2. **Processing Complexity**: Number of pipelines and transformations
3. **State Size**: Stateful operations and window sizes
4. **Latency Requirements**: End-to-end processing time

### High Availability

For production deployments, consider:

- Multiple control plane replicas
- Worker node redundancy
- Cross-AZ or multi-region deployment
- Automated failover and recovery

### Security

Essential security measures:

- TLS encryption for all communication
- Authentication and authorization
- Network segmentation
- Secrets management
- Audit logging

## Quick Start

### Local Development

```bash
docker-compose up -d

laminar status
```

### Kubernetes

```bash
helm repo add laminar https://charts.laminar.dev
helm install laminar laminar/laminar

kubectl get pods -n laminar
```

## Configuration Management

Laminar configuration can be managed through:

- Environment variables
- Configuration files (YAML/JSON)
- Kubernetes ConfigMaps/Secrets
- Cloud provider parameter stores

## Next Steps

- [Kubernetes Deployment](./kubernetes/setup) - Production deployment guide
- [Docker Setup](./docker/setup) - Local development setup
- [Configuration](./config/environment) - Detailed configuration options
- [Security](./config/security) - Security best practices