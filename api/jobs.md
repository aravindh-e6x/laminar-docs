---
sidebar_position: 6
title: Jobs
---


The Jobs API provides endpoints for monitoring and managing pipeline executions. A job represents a running instance of a pipeline and tracks its execution state, metrics, and output.

## Job Object

A job object contains:

```json
{
  "id": "job_9876543210",
  "pipeline_id": "pipe_1234567890",
  "running_desired": true,
  "state": "Running",
  "run_id": 3,
  "start_time": 1704067200000,
  "finish_time": null,
  "tasks": 4,
  "failure_message": null,
  "created_at": 1704067200000,
  "metrics": {
    "records_in": 1000000,
    "records_out": 950000,
    "bytes_in": 104857600,
    "bytes_out": 99614720,
    "errors": 0,
    "warnings": 5
  }
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique job identifier |
| `pipeline_id` | string | Associated pipeline ID |
| `running_desired` | boolean | Whether the job should be running |
| `state` | string | Current job state |
| `run_id` | integer | Run number for this pipeline |
| `start_time` | integer | Start timestamp (milliseconds) |
| `finish_time` | integer | Finish timestamp (milliseconds) |
| `tasks` | integer | Number of tasks in the job |
| `failure_message` | string | Error message if failed |
| `created_at` | integer | Creation timestamp |
| `metrics` | object | Job performance metrics |

## Endpoints

### List All Jobs

Get all jobs across all pipelines.

```http
GET /api/v1/jobs
```

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `pipeline_id` | string | Filter by pipeline |
| `state` | string | Filter by state |
| `running` | boolean | Filter running/stopped jobs |
| `limit` | integer | Maximum results (default: 20) |
| `offset` | integer | Skip results |
| `sort` | string | Sort field and order |

#### Response

```json
{
  "data": [
    {
      "id": "job_9876543210",
      "pipeline_id": "pipe_1234567890",
      "state": "Running",
      "run_id": 3,
      "start_time": 1704067200000,
      "tasks": 4
    }
  ],
  "has_more": false
}
```

#### Example

```bash
curl "http://localhost:5115/api/v1/jobs?running=true"

curl "http://localhost:5115/api/v1/jobs?state=Failed"
```

### Get Job Details

Retrieve detailed information about a specific job.

```http
GET /api/v1/jobs/{job_id}
```

#### Path Parameters

| Parameter | Description |
|-----------|-------------|
| `job_id` | The job ID |

#### Response

Returns the complete job object with metrics and configuration.

#### Example

```bash
curl http://localhost:5115/api/v1/jobs/job_9876543210
```

### Get Job Metrics

Get detailed metrics for a job.

```http
GET /api/v1/jobs/{job_id}/metrics
```

#### Response

```json
{
  "records_in": 1000000,
  "records_out": 950000,
  "bytes_in": 104857600,
  "bytes_out": 99614720,
  "errors": 0,
  "warnings": 5,
  "latency_p50": 45,
  "latency_p95": 120,
  "latency_p99": 250,
  "throughput": 10000,
  "backpressure": 0.15,
  "operators": [
    {
      "operator_id": "source_1",
      "operator_name": "KafkaSource",
      "records_in": 1000000,
      "records_out": 1000000,
      "errors": 0
    }
  ]
}
```

### Get Job Checkpoints

List all checkpoints for a job.

```http
GET /api/v1/jobs/{job_id}/checkpoints
```

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Maximum results |
| `offset` | integer | Skip results |
| `completed` | boolean | Filter completed checkpoints |

#### Response

```json
{
  "data": [
    {
      "id": "chk_123456",
      "job_id": "job_9876543210",
      "epoch": 10,
      "start_time": 1704067200000,
      "finish_time": 1704067205000,
      "state": "completed",
      "size_bytes": 1048576,
      "operator_states": [
        {
          "operator_id": "source_1",
          "state_size": 524288
        }
      ]
    }
  ]
}
```

### Get Checkpoint Details

Get detailed information about a specific checkpoint.

```http
GET /api/v1/jobs/{job_id}/checkpoints/{checkpoint_id}
```

#### Response

```json
{
  "id": "chk_123456",
  "job_id": "job_9876543210",
  "epoch": 10,
  "start_time": 1704067200000,
  "finish_time": 1704067205000,
  "state": "completed",
  "size_bytes": 1048576,
  "backend": "filesystem",
  "location": "s3://checkpoints/job_9876543210/epoch_10",
  "operator_states": [
    {
      "operator_id": "source_1",
      "operator_name": "KafkaSource",
      "state_size": 524288,
      "subtask_states": [
        {
          "subtask_index": 0,
          "state_size": 131072
        }
      ]
    }
  ]
}
```

### Get Job Output

Retrieve output data from a job (for preview/debug pipelines).

```http
GET /api/v1/jobs/{job_id}/output
```

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `operator_id` | string | Filter by operator |
| `limit` | integer | Maximum records |
| `offset` | integer | Skip records |
| `format` | string | Output format: `json`, `csv` |

#### Response

```json
{
  "data": [
    {
      "timestamp": 1704067200000,
      "record": {
        "id": 1,
        "message": "Test message",
        "value": 99.99
      }
    }
  ],
  "operator_id": "sink_1",
  "total_records": 1000
}
```

### Get Job Errors

Retrieve error messages from a job.

```http
GET /api/v1/jobs/{job_id}/errors
```

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `level` | string | Filter by level: `error`, `warning` |
| `operator_id` | string | Filter by operator |
| `limit` | integer | Maximum results |

#### Response

```json
{
  "data": [
    {
      "id": "err_123",
      "timestamp": 1704067200000,
      "level": "error",
      "operator_id": "source_1",
      "task_index": 0,
      "message": "Failed to parse JSON",
      "details": "Invalid JSON at line 1: unexpected character",
      "record": "{\"bad json"
    }
  ]
}
```

### Stop Job

Stop a running job.

```http
POST /api/v1/jobs/{job_id}/stop
```

#### Request Body

```json
{
  "mode": "checkpoint",
  "timeout_seconds": 30
}
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `mode` | string | Stop mode: `checkpoint`, `immediate`, `force` |
| `timeout_seconds` | integer | Maximum wait time |

#### Response

```json
{
  "message": "Job stop initiated",
  "expected_finish_time": 1704067230000
}
```

### Restart Job

Restart a failed or stopped job.

```http
POST /api/v1/jobs/{job_id}/restart
```

#### Request Body

```json
{
  "checkpoint_id": "chk_123456",
  "parallelism": 4
}
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `checkpoint_id` | string | Checkpoint to restore from |
| `parallelism` | integer | Override parallelism |

#### Response

```json
{
  "message": "Job restarted",
  "new_job_id": "job_9876543211"
}
```

## Job States

Jobs can be in the following states:

| State | Description |
|-------|-------------|
| `Created` | Job created but not started |
| `Scheduling` | Job is being scheduled |
| `Running` | Job is actively processing |
| `Stopping` | Job is stopping |
| `Stopped` | Job has been stopped |
| `Failed` | Job failed with error |
| `Finished` | Job completed successfully |
| `Rescaling` | Job is changing parallelism |
| `Recovering` | Job is recovering from failure |

## Metrics

### Operator Metrics

Get metrics for specific operators within a job.

```http
GET /api/v1/jobs/{job_id}/operators/{operator_id}/metrics
```

#### Response

```json
{
  "operator_id": "source_1",
  "operator_name": "KafkaSource",
  "subtasks": 4,
  "metrics": {
    "records_in": 1000000,
    "records_out": 1000000,
    "bytes_in": 104857600,
    "bytes_out": 104857600,
    "errors": 0,
    "warnings": 0,
    "backpressure": 0.0,
    "idle_time_percent": 5.2
  },
  "subtask_metrics": [
    {
      "subtask_index": 0,
      "records_in": 250000,
      "records_out": 250000
    }
  ]
}
```

### Metric Groups

Get aggregated metrics by group.

```http
GET /api/v1/jobs/{job_id}/metric-groups
```

#### Response

```json
{
  "sources": {
    "total_records": 1000000,
    "throughput": 10000,
    "operators": ["source_1", "source_2"]
  },
  "sinks": {
    "total_records": 950000,
    "throughput": 9500,
    "operators": ["sink_1"]
  },
  "transforms": {
    "total_records": 950000,
    "operators": ["filter_1", "map_1"]
  }
}
```

## Job Logs

### Get Job Logs

Retrieve log messages from a job.

```http
GET /api/v1/jobs/{job_id}/logs
```

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `level` | string | Filter by level: `info`, `warn`, `error` |
| `operator_id` | string | Filter by operator |
| `task_index` | integer | Filter by task |
| `since` | integer | Logs after timestamp |
| `until` | integer | Logs before timestamp |
| `limit` | integer | Maximum messages |

#### Response

```json
{
  "data": [
    {
      "id": "log_123",
      "timestamp": 1704067200000,
      "level": "info",
      "operator_id": "source_1",
      "task_index": 0,
      "message": "Connected to Kafka broker",
      "details": {
        "broker": "localhost:9092",
        "topic": "events"
      }
    }
  ]
}
```
