---
sidebar_position: 3
title: Pipeline Clusters
---

# Pipeline Clusters

Pipeline clusters provide a lightweight, self-contained way to run individual Laminar pipelines. This deployment mode is ideal for development, testing, CI/CD pipelines, and serverless platforms.

## Overview

A pipeline cluster is a minimal Laminar deployment that:
- Runs a single pipeline in isolation
- Requires minimal resources
- Starts quickly with low overhead
- Supports state management and checkpointing
- Can be deployed on serverless platforms

## Use Cases

### Development & Testing
- Rapid prototyping of streaming pipelines
- Local development without full cluster setup
- Integration testing in CI/CD pipelines
- Debugging and performance testing

### Serverless Deployments
- **AWS Fargate**: Containerized pipeline execution
- **Google Cloud Run**: Fully managed serverless pipelines
- **Azure Container Instances**: On-demand pipeline runs
- **Fly.io**: Edge-deployed stream processing

### Batch Processing
- Scheduled pipeline runs
- Event-triggered processing
- Finite stream processing
- Data migration and ETL jobs

## Running Pipeline Clusters

### Using the CLI

The `laminar run` command creates and manages pipeline clusters:

```bash
# Run a pipeline from a SQL file
laminar run --sql-file pipeline.sql --name my-pipeline

# Run with custom parallelism
laminar run --sql-file pipeline.sql --parallelism 4

# Run with state directory
laminar run --sql-file pipeline.sql --state-dir /data/checkpoints

# Run from stdin
echo "INSERT INTO output SELECT * FROM input" | laminar run

# Run from environment variable
export LAMINAR_SQL="INSERT INTO output SELECT * FROM input"
laminar run --sql-env LAMINAR_SQL
```

### Command Options

| Option | Description | Default |
|--------|-------------|---------|
| `--name` | Pipeline name | auto-generated |
| `--sql-file` | Path to SQL file | - |
| `--sql` | Inline SQL query | - |
| `--sql-env` | Environment variable containing SQL | - |
| `--parallelism` | Number of parallel subtasks | 1 |
| `--state-dir` | Directory for checkpoints | `/tmp/laminar` |
| `--checkpoint-interval` | Checkpoint frequency (ms) | 10000 |
| `--force` | Force start without confirmation | false |
| `--config` | Path to configuration file | - |

## Configuration

### Basic Configuration

Create a `pipeline.yaml` configuration file:

```yaml
name: my-pipeline
parallelism: 4
checkpoint:
  interval: 30s
  path: s3://my-bucket/checkpoints
sql: |
  CREATE PIPELINE orders_processing AS
  INSERT INTO processed_orders
  SELECT 
    order_id,
    customer_id,
    total_amount,
    process_timestamp()
  FROM raw_orders
  WHERE status = 'pending'
```

Run with configuration:
```bash
laminar run --config pipeline.yaml
```

### Environment Variables

Configure pipeline clusters using environment variables:

```bash
# Core settings
export LAMINAR_PIPELINE_NAME=my-pipeline
export LAMINAR_PARALLELISM=4

# Checkpoint configuration
export LAMINAR_CHECKPOINT_URL=s3://my-bucket/checkpoints
export LAMINAR_CHECKPOINT_INTERVAL=30000

# Database configuration
export LAMINAR_DATABASE_TYPE=sqlite
export LAMINAR_DATABASE_PATH=/data/laminar.db

# Storage configuration
export LAMINAR_ARTIFACT_URL=s3://my-bucket/artifacts
export LAMINAR_S3_ENDPOINT=https://s3.amazonaws.com
export AWS_ACCESS_KEY_ID=your-key
export AWS_SECRET_ACCESS_KEY=your-secret
```

## State Management

### Checkpointing

Pipeline clusters support full state management:

```yaml
checkpoint:
  enabled: true
  interval: 30s
  path: s3://checkpoints/my-pipeline
  min_pause_between: 5m
  timeout: 10m
  retention:
    max_checkpoints: 10
    max_age: 7d
```

### Recovering from Checkpoints

Resume a pipeline from the last checkpoint:

```bash
# List available checkpoints
laminar checkpoint list --pipeline my-pipeline

# Restore from specific checkpoint
laminar run --restore-from checkpoint-123 --sql-file pipeline.sql

# Restore from latest checkpoint
laminar run --restore-latest --sql-file pipeline.sql
```

## Docker Deployment

### Using Official Image

```dockerfile
FROM ghcr.io/laminar/laminar:latest

COPY pipeline.sql /app/pipeline.sql
COPY config.yaml /app/config.yaml

CMD ["laminar", "run", "--config", "/app/config.yaml"]
```

### Docker Compose

```yaml
version: '3.8'
services:
  pipeline:
    image: ghcr.io/laminar/laminar:latest
    environment:
      - LAMINAR_PIPELINE_NAME=my-pipeline
      - LAMINAR_CHECKPOINT_URL=s3://checkpoints
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
    volumes:
      - ./pipeline.sql:/app/pipeline.sql
    command: laminar run --sql-file /app/pipeline.sql
```

## Serverless Deployments

### AWS Fargate

Deploy pipeline clusters on AWS Fargate:

```json
{
  "family": "laminar-pipeline",
  "taskRoleArn": "arn:aws:iam::account:role/laminar-task-role",
  "executionRoleArn": "arn:aws:iam::account:role/laminar-execution-role",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "pipeline",
      "image": "ghcr.io/laminar/laminar:latest",
      "environment": [
        {"name": "LAMINAR_PIPELINE_NAME", "value": "orders-processing"},
        {"name": "LAMINAR_CHECKPOINT_URL", "value": "s3://checkpoints"}
      ],
      "command": ["laminar", "run", "--sql", "INSERT INTO processed SELECT * FROM orders"],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/laminar",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "pipeline"
        }
      }
    }
  ]
}
```

### Google Cloud Run

Deploy on Cloud Run:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: laminar-pipeline
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/execution-environment: gen2
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containerConcurrency: 1
      timeoutSeconds: 3600
      containers:
      - image: ghcr.io/laminar/laminar:latest
        resources:
          limits:
            cpu: "2"
            memory: 4Gi
        env:
        - name: LAMINAR_PIPELINE_NAME
          value: orders-processing
        - name: LAMINAR_CHECKPOINT_URL
          value: gs://checkpoints
        command:
        - laminar
        - run
        - --sql-file
        - /app/pipeline.sql
```

### Kubernetes Jobs

Run as Kubernetes Job:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pipeline-job
spec:
  template:
    spec:
      containers:
      - name: pipeline
        image: ghcr.io/laminar/laminar:latest
        command: ["laminar", "run"]
        args: ["--sql-file", "/config/pipeline.sql"]
        env:
        - name: LAMINAR_CHECKPOINT_URL
          value: s3://checkpoints
        volumeMounts:
        - name: config
          mountPath: /config
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
      volumes:
      - name: config
        configMap:
          name: pipeline-config
      restartPolicy: OnFailure
```

## Multi-Statement Pipelines

Pipeline clusters can run multiple SQL statements:

```sql
-- Create temporary view
CREATE TEMPORARY VIEW enriched_orders AS
SELECT 
  o.*,
  c.customer_name,
  c.customer_tier
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Process high-value orders
INSERT INTO high_value_orders
SELECT * FROM enriched_orders
WHERE total_amount > 1000;

-- Process regular orders
INSERT INTO regular_orders
SELECT * FROM enriched_orders
WHERE total_amount <= 1000;

-- Update statistics
INSERT INTO order_stats
SELECT 
  DATE_TRUNC('hour', order_time) as hour,
  COUNT(*) as order_count,
  SUM(total_amount) as total_revenue
FROM enriched_orders
GROUP BY DATE_TRUNC('hour', order_time);
```

## Monitoring

### Metrics

Pipeline clusters expose metrics on port 9090:

```bash
# Enable metrics
laminar run --metrics-port 9090 --sql-file pipeline.sql

# Prometheus scrape configuration
scrape_configs:
  - job_name: 'laminar-pipeline'
    static_configs:
      - targets: ['localhost:9090']
```

### Logging

Configure logging levels and outputs:

```bash
# Set log level
export RUST_LOG=info,laminar=debug

# JSON logging
export LAMINAR_LOG_FORMAT=json

# Log to file
laminar run --log-file /var/log/pipeline.log --sql-file pipeline.sql
```

### Health Checks

Pipeline clusters provide health endpoints:

```bash
# Liveness probe
curl http://localhost:8080/health/live

# Readiness probe
curl http://localhost:8080/health/ready

# Metrics
curl http://localhost:9090/metrics
```

## Best Practices

### Resource Allocation
- Set appropriate CPU and memory limits
- Use parallelism based on data volume
- Monitor resource usage and adjust

### Checkpoint Strategy
- Use object storage for production checkpoints
- Set checkpoint intervals based on recovery requirements
- Clean up old checkpoints regularly

### Error Handling
- Configure restart policies
- Set up alerting for failures
- Use dead letter queues for failed records

### Security
- Use IAM roles instead of credentials
- Encrypt checkpoints and data
- Run in private networks when possible

## Troubleshooting

### Common Issues

**Pipeline Won't Start**
```bash
# Check configuration
laminar run --validate --sql-file pipeline.sql

# Increase log verbosity
export RUST_LOG=debug
laminar run --sql-file pipeline.sql
```

**Checkpoint Failures**
```bash
# Test storage access
laminar storage test --url s3://my-bucket/test

# Check permissions
aws s3 ls s3://my-bucket/checkpoints/
```

**Performance Issues**
```bash
# Increase parallelism
laminar run --parallelism 8 --sql-file pipeline.sql

# Profile pipeline
laminar run --profile --sql-file pipeline.sql
```

## Limitations

Pipeline clusters have some limitations compared to full deployments:

- Single pipeline per cluster
- Limited to one job manager
- No pipeline management UI
- Basic scheduling capabilities
- No multi-tenancy support

For production workloads requiring these features, consider [Kubernetes deployment](./kubernetes) or [full cluster deployment](./overview).