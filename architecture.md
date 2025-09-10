---
sidebar_position: 2
title: Architecture
---

# Architecture

Laminar is built on a distributed, scalable architecture designed for high-throughput, low-latency stream processing. This guide provides an in-depth look at the system architecture and its components.

### System Overview

![Laminar Distributed System Architecture](img/distributed-system-architecture.svg)

### Core Components

#### Control Plane

The control plane manages the lifecycle of pipelines and jobs, handling all administrative operations.

**API Service**

* Exposes REST API endpoints
* Handles authentication and authorization
* Validates pipeline configurations
* Manages connection profiles and tables

**Pipeline Manager**

* Stores pipeline definitions
* Validates SQL queries
* Compiles pipelines into execution plans
* Manages pipeline versions

**Job Controller**

* Schedules and deploys jobs
* Monitors job health
* Handles job lifecycle (start, stop, restart)
* Manages resource allocation

**Metadata Store**

* Persists pipeline configurations
* Stores connection information
* Maintains job history
* Tracks system state

**Scheduler**

* Assigns tasks to workers
* Balances load across nodes
* Handles worker failures
* Manages task parallelism

**Checkpoint Coordinator**

* Triggers periodic checkpoints
* Coordinates distributed snapshots
* Manages checkpoint storage
* Handles recovery coordination

#### Data Plane

The data plane executes the actual stream processing logic.

**Worker Nodes**

Worker nodes run the distributed computation:

* **Task Execution** - Run pipeline operators
* **State Management** - Maintain operator state
* **Data Processing** - Execute transformations
* **Network Communication** - Exchange data between nodes

Each worker node contains:

* **Task Managers** - Execute individual tasks
* **State Backend** - Local state storage
* **Network Stack** - Inter-node communication
* **Metrics Collector** - Performance monitoring

#### Storage Layer

**PostgreSQL (Metadata)**

Stores all system metadata:

* Pipeline definitions
* Connection configurations
* Job metadata
* User information
* System configuration

**Object Storage (State)**

Stores distributed state:

* Checkpoints
* Savepoints
* Large state objects
* Historical data

Supported backends:

* Amazon S3
* Google Cloud Storage
* Azure Blob Storage
* MinIO
* Local filesystem

**Prometheus (Metrics)**

Collects and stores metrics:

* System metrics
* Job metrics
* Operator metrics
* Custom metrics

### Data Flow

#### Pipeline Execution Flow

![Pipeline Execution Flow](img/pipeline-execution-flow.svg)

#### Fault Tolerance

Laminar provides exactly-once processing guarantees through:

**Checkpointing**

* Periodic consistent snapshots
* Asynchronous checkpoint barriers
* Incremental checkpointing
* Checkpoint alignment

**State Management**

* Distributed state backends
* State versioning
* State migration
* State recovery

**Failure Recovery**

* Automatic failure detection
* Checkpoint-based recovery
* Task-level restarts
* Pipeline-level recovery

### Scalability

#### Horizontal Scaling

Laminar scales horizontally by:

* Adding more worker nodes
* Increasing task parallelism
* Partitioning data streams
* Distributing state

#### Vertical Scaling

Individual components can be scaled vertically:

* Increase CPU cores
* Add more memory
* Enhance network bandwidth
* Expand storage capacity

#### Auto-scaling

Laminar supports auto-scaling based on:

* CPU utilization
* Memory usage
* Queue depth
* Throughput metrics
* Custom metrics

### Next Steps

* Learn about [Configuration](docs/concepts/configuration/) options
* Explore [Deployment](deployment/overview/) strategies
* Understand [Observability](observability/intro/) features
* Review [Performance Tuning](docs/concepts/configuration/#performance-tuning) guidelines
