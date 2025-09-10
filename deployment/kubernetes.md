---
sidebar_position: 4
title: Kubernetes
---


Kubernetes is the recommended deployment method for running Laminar in production. This guide covers deploying Laminar on Kubernetes using Helm, including configuration for various cloud providers and production best practices.

## Prerequisites

### Required Components
- Kubernetes cluster (version ≥ 1.25)
- Helm 3.x installed
- kubectl configured to access your cluster
- Storage backend (S3, GCS, or Azure Blob Storage)
- PostgreSQL database (managed or self-hosted)

### Optional Components
- Prometheus for metrics collection
- Grafana for visualization
- External secrets manager
- Ingress controller for external access

## Quick Start

### 1. Add Helm Repository

```bash
helm repo add laminar https://charts.laminar.dev
helm repo update

helm search repo laminar
```

### 2. Basic Installation

```bash
helm install laminar laminar/laminar

kubectl create namespace laminar
helm install laminar laminar/laminar -n laminar

helm install my-streaming laminar/laminar
```

### 3. Verify Installation

```bash
kubectl get pods -n laminar

kubectl get svc -n laminar

helm status laminar -n laminar
```

## Configuration

### Values File

Create a `values.yaml` file for your configuration:

```yaml
global:
  # Cluster name for identification
  clusterName: production
  
  # Storage configuration
  storage:
    checkpoint:
      url: s3://laminar-checkpoints/prod
    artifact:
      url: s3://laminar-artifacts/prod
  
  # Database configuration
  database:
    type: postgres
    host: postgres.example.com
    port: 5432
    database: laminar
    username: laminar
    # Use existing secret for password
    existingSecret: laminar-db-secret
    secretKey: password

apiServer:
  replicas: 3
  resources:
    requests:
      cpu: 2
      memory: 4Gi
    limits:
      cpu: 4
      memory: 8Gi

worker:
  replicas: 5
  resources:
    requests:
      cpu: 4
      memory: 16Gi
    limits:
      cpu: 8
      memory: 32Gi

webui:
  enabled: true
  ingress:
    enabled: true
    className: nginx
    host: laminar.example.com
    tls:
      enabled: true
      secretName: laminar-tls

prometheus:
  enabled: true
  serviceMonitor:
    enabled: true
```

Install with custom values:
```bash
helm install laminar laminar/laminar -f values.yaml -n laminar
```

## Cloud Provider Configuration

### Amazon EKS

Configuration for AWS EKS deployment:

```yaml
global:
  cloud:
    provider: aws
    region: us-east-1
  
  storage:
    checkpoint:
      url: s3://laminar-checkpoints-prod
    artifact:
      url: s3://laminar-artifacts-prod
    
    # S3 configuration
    s3:
      region: us-east-1
      # Use IRSA for authentication
      useIRSA: true

serviceAccount:
  create: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/laminar-role

postgresql:
  enabled: false
  
database:
  type: postgres
  host: laminar-db.abc123.us-east-1.rds.amazonaws.com
  port: 5432
  database: laminar
  username: laminar
  existingSecret: rds-secret

apiServer:
  nodeSelector:
    node-group: control-plane
  tolerations:
    - key: control-plane
      operator: Equal
      value: "true"
      effect: NoSchedule

worker:
  nodeSelector:
    node-group: workers
  # Use spot instances for workers
  tolerations:
    - key: spot
      operator: Equal
      value: "true"
      effect: NoSchedule

ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789:certificate/abc
```

Deploy on EKS:
```bash
kubectl create namespace laminar

eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=laminar \
  --name=laminar \
  --attach-policy-arn=arn:aws:iam::123456789:policy/LaminarPolicy \
  --approve

helm install laminar laminar/laminar -f eks-values.yaml -n laminar
```

### Google GKE

Configuration for Google GKE deployment:

```yaml
global:
  cloud:
    provider: gcp
    project: my-project
    region: us-central1
  
  storage:
    checkpoint:
      url: gs://laminar-checkpoints-prod
    artifact:
      url: gs://laminar-artifacts-prod
    
    # GCS configuration
    gcs:
      project: my-project
      # Use Workload Identity
      useWorkloadIdentity: true

serviceAccount:
  create: true
  annotations:
    iam.gke.io/gcp-service-account: laminar@my-project.iam.gserviceaccount.com

postgresql:
  enabled: false
  
database:
  type: postgres
  host: 10.20.30.40  # Private IP of Cloud SQL
  port: 5432
  database: laminar
  username: laminar
  # Use Cloud SQL Proxy
  cloudSqlProxy:
    enabled: true
    instanceConnectionName: my-project:us-central1:laminar-db

resources:
  apiServer:
    requests:
      cpu: 2
      memory: 4Gi
    limits:
      cpu: 2
      memory: 4Gi
  
  worker:
    requests:
      cpu: 4
      memory: 16Gi
    limits:
      cpu: 4
      memory: 16Gi

ingress:
  enabled: true
  className: gce
  annotations:
    ingress.gcp.kubernetes.io/pre-shared-cert: laminar-cert
    cloud.google.com/backend-config: '{"default": "laminar-backend-config"}'
```

Deploy on GKE:
```bash
kubectl create namespace laminar

kubectl annotate serviceaccount laminar \
  -n laminar \
  iam.gke.io/gcp-service-account=laminar@my-project.iam.gserviceaccount.com

helm install laminar laminar/laminar -f gke-values.yaml -n laminar
```

### Azure AKS

Configuration for Azure AKS deployment:

```yaml
global:
  cloud:
    provider: azure
    region: eastus
    resourceGroup: laminar-rg
  
  storage:
    checkpoint:
      url: abs://checkpoints
    artifact:
      url: abs://artifacts
    
    # Azure Blob Storage configuration
    azure:
      storageAccount: laminarstorageacct
      # Use Managed Identity
      useManagedIdentity: true

podIdentity:
  enabled: true
  identityName: laminar-identity
  identityClientId: 12345678-1234-1234-1234-123456789abc

postgresql:
  enabled: false
  
database:
  type: postgres
  host: laminar-db.postgres.database.azure.com
  port: 5432
  database: laminar
  username: laminar@laminar-db
  sslMode: require
  existingSecret: postgres-secret

persistence:
  storageClass: managed-csi-premium
  size: 100Gi

ingress:
  enabled: true
  className: azure/application-gateway
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/use-private-ip: "false"
```

## Advanced Configuration

### High Availability

Configure Laminar for high availability:

```yaml
highAvailability:
  enabled: true

apiServer:
  replicas: 3
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app.kubernetes.io/component
            operator: In
            values:
            - api-server
        topologyKey: kubernetes.io/hostname

jobManager:
  replicas: 3
  enableLeaderElection: true

database:
  type: postgres
  hosts:
    - postgres-1.example.com
    - postgres-2.example.com
    - postgres-3.example.com
  port: 5432
  database: laminar
  # Connection pooling
  pool:
    maxConnections: 100
    minConnections: 10
```

### Auto-scaling

Configure horizontal pod autoscaling:

```yaml
autoscaling:
  enabled: true
  
  # API Server autoscaling
  apiServer:
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
  
  # Worker autoscaling
  worker:
    minReplicas: 3
    maxReplicas: 50
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metrics
    - type: Pods
      pods:
        metric:
          name: pending_tasks
        target:
          type: AverageValue
          averageValue: "30"
```

### Security

Security-hardened configuration:

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true

networkPolicy:
  enabled: true
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: laminar
  egress:
    - to:
      - namespaceSelector:
          matchLabels:
            name: laminar
    # Allow DNS
    - to:
      - namespaceSelector: {}
        podSelector:
          matchLabels:
            k8s-app: kube-dns
      ports:
      - port: 53
        protocol: UDP

tls:
  enabled: true
  # Internal TLS
  internal:
    enabled: true
    certManager: true
    issuer: laminar-ca-issuer
  # External TLS
  external:
    enabled: true
    certManager: true
    issuer: letsencrypt-prod

secrets:
  provider: kubernetes  # or vault, aws-secrets-manager, etc.
  vault:
    enabled: false
    address: https://vault.example.com
    role: laminar
    path: secret/data/laminar
```

## Monitoring & Observability

### Prometheus Integration

```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
    labels:
      prometheus: kube-prometheus
    interval: 30s
    scrapeTimeout: 10s

grafana:
  enabled: true
  dashboards:
    enabled: true
    label: grafana_dashboard
    folder: laminar
```

### Logging

```yaml
logging:
  enabled: true
  level: INFO
  format: json
  
  # Fluentd/Fluent Bit integration
  fluentd:
    enabled: true
    config: |
      <source>
        @type tail
        path /var/log/laminar/*.log
        pos_file /var/log/fluentd-laminar.pos
        tag laminar.*
        <parse>
          @type json
        </parse>
      </source>
      
      <match laminar.**>
        @type elasticsearch
        host elasticsearch.logging.svc.cluster.local
        port 9200
        logstash_format true
        logstash_prefix laminar
      </match>
```

## Upgrade & Rollback

### Upgrading Laminar

```bash
helm repo update

helm search repo laminar --versions

helm upgrade laminar laminar/laminar \
  -f values.yaml \
  -n laminar \
  --dry-run

helm upgrade laminar laminar/laminar \
  -f values.yaml \
  -n laminar \
  --wait \
  --timeout 10m

kubectl rollout status deployment/laminar-api-server -n laminar
```

### Rolling Back

```bash
helm history laminar -n laminar

helm rollback laminar -n laminar

helm rollback laminar 3 -n laminar
```

## Troubleshooting

### Common Issues

**Pods Not Starting**
```bash
kubectl get pods -n laminar
kubectl describe pod <pod-name> -n laminar

kubectl logs <pod-name> -n laminar
kubectl logs <pod-name> -n laminar --previous
```

**Storage Access Issues**
```bash
kubectl get sa laminar -n laminar -o yaml

kubectl auth can-i get pods --as=system:serviceaccount:laminar:laminar

kubectl run test-storage --rm -it \
  --image=laminar/laminar:latest \
  --serviceaccount=laminar \
  -- laminar storage test s3://my-bucket
```

**Database Connection Issues**
```bash
kubectl run test-db --rm -it \
  --image=postgres:14 \
  -- psql -h postgres.example.com -U laminar -d laminar

kubectl get secret laminar-db-secret -n laminar -o yaml
```

### Debug Mode

Enable debug logging:
```yaml
debug:
  enabled: true
  verbosity: 5
  
env:
  - name: RUST_LOG
    value: debug,laminar=trace
  - name: RUST_BACKTRACE
    value: "full"
```

## Best Practices

1. **Use separate namespaces** for different environments
2. **Enable resource quotas** to prevent resource exhaustion
3. **Configure pod disruption budgets** for high availability
4. **Use network policies** to restrict traffic
5. **Enable audit logging** for security compliance
6. **Regular backups** of stateful components
7. **Monitor resource usage** and adjust limits accordingly
8. **Use GitOps** for configuration management
9. **Implement progressive rollouts** for updates
10. **Document your configuration** and deployment procedures