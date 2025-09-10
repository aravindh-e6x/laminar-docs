---
sidebar_position: 5
title: Pipelines
---

# Pipeline Management API

The Pipeline API allows you to create, manage, and monitor streaming pipelines in Laminar. Pipelines define the data processing logic using SQL queries and can be configured with various execution parameters.

## Pipeline Object

A pipeline object contains:

```json
{
  "id": "pipe_1234567890",
  "name": "order_processing",
  "query": "SELECT * FROM orders WHERE amount > 100",
  "udfs": [],
  "checkpoint_interval_micros": 10000000,
  "parallelism": 4,
  "stop": "none",
  "created_at": 1704067200000,
  "action": null,
  "action_text": "Running",
  "action_in_progress": false,
  "graph": {
    "nodes": [...],
    "edges": [...]
  },
  "preview": false
}
```

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique pipeline identifier |
| `name` | string | Human-readable pipeline name |
| `query` | string | SQL query defining the pipeline logic |
| `udfs` | array | User-defined functions used in the pipeline |
| `checkpoint_interval_micros` | integer | Checkpoint interval in microseconds |
| `parallelism` | integer | Number of parallel tasks |
| `stop` | string | Stop mode: `none`, `checkpoint`, `graceful`, `immediate`, `force` |
| `created_at` | integer | Creation timestamp (milliseconds since epoch) |
| `action` | string | Current action being performed |
| `action_text` | string | Human-readable action description |
| `action_in_progress` | boolean | Whether an action is currently running |
| `graph` | object | Pipeline execution graph |
| `preview` | boolean | Whether this is a preview pipeline |

## Endpoints

### Create Pipeline

Create a new pipeline with the specified configuration.

```http
POST /api/v1/pipelines
```

#### Request Body

```json
{
  "name": "order_processing",
  "query": "CREATE TABLE orders_output WITH (\n  connector = 'kafka',\n  topic = 'processed-orders'\n) AS\nSELECT \n  order_id,\n  customer_id,\n  amount * 1.1 as total_with_tax,\n  timestamp\nFROM orders\nWHERE amount > 100",
  "parallelism": 4,
  "checkpoint_interval_micros": 10000000,
  "udfs": []
}
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | string | Yes | - | Pipeline name (must be unique) |
| `query` | string | Yes | - | SQL query defining the pipeline |
| `parallelism` | integer | No | 1 | Number of parallel tasks |
| `checkpoint_interval_micros` | integer | No | 10000000 | Checkpoint interval in microseconds |
| `udfs` | array | No | [] | User-defined functions |

#### Response

```json
{
  "id": "pipe_1234567890",
  "name": "order_processing",
  "query": "...",
  "parallelism": 4,
  "checkpoint_interval_micros": 10000000,
  "created_at": 1704067200000,
  "graph": {...}
}
```

#### Example

```bash
curl -X POST http://localhost:5115/api/v1/pipelines \
  -H "Content-Type: application/json" \
  -d '{
    "name": "clickstream_analytics",
    "query": "SELECT user_id, COUNT(*) as clicks FROM events GROUP BY user_id",
    "parallelism": 2
  }'
```

### List Pipelines

Retrieve a list of all pipelines.

```http
GET /api/v1/pipelines
```

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Maximum number of results (default: 20, max: 100) |
| `offset` | integer | Number of results to skip |
| `starting_after` | string | Cursor for forward pagination |
| `name` | string | Filter by pipeline name (supports wildcards) |
| `state` | string | Filter by state: `running`, `stopped`, `failed` |

#### Response

```json
{
  "data": [
    {
      "id": "pipe_1234567890",
      "name": "order_processing",
      "created_at": 1704067200000,
      "action_text": "Running"
    }
  ],
  "has_more": false
}
```

#### Example

```bash
# Get all running pipelines
curl "http://localhost:5115/api/v1/pipelines?state=running"

# Paginate through pipelines
curl "http://localhost:5115/api/v1/pipelines?limit=10&offset=20"
```

### Get Pipeline

Retrieve details of a specific pipeline.

```http
GET /api/v1/pipelines/{pipeline_id}
```

#### Path Parameters

| Parameter | Description |
|-----------|-------------|
| `pipeline_id` | The pipeline ID |

#### Response

```json
{
  "id": "pipe_1234567890",
  "name": "order_processing",
  "query": "SELECT * FROM orders",
  "parallelism": 4,
  "checkpoint_interval_micros": 10000000,
  "created_at": 1704067200000,
  "graph": {
    "nodes": [...],
    "edges": [...]
  }
}
```

#### Example

```bash
curl http://localhost:5115/api/v1/pipelines/pipe_1234567890
```

### Update Pipeline

Update configuration of an existing pipeline.

```http
PATCH /api/v1/pipelines/{pipeline_id}
```

#### Path Parameters

| Parameter | Description |
|-----------|-------------|
| `pipeline_id` | The pipeline ID |

#### Request Body

```json
{
  "parallelism": 8,
  "checkpoint_interval_micros": 5000000,
  "stop": "checkpoint"
}
```

#### Updatable Fields

| Field | Type | Description |
|-------|------|-------------|
| `parallelism` | integer | Number of parallel tasks |
| `checkpoint_interval_micros` | integer | Checkpoint interval |
| `stop` | string | Stop mode: `none`, `checkpoint`, `graceful`, `immediate`, `force` |

#### Response

Returns the updated pipeline object.

#### Example

```bash
# Update parallelism
curl -X PATCH http://localhost:5115/api/v1/pipelines/pipe_1234567890 \
  -H "Content-Type: application/json" \
  -d '{"parallelism": 8}'

# Stop pipeline with checkpoint
curl -X PATCH http://localhost:5115/api/v1/pipelines/pipe_1234567890 \
  -H "Content-Type: application/json" \
  -d '{"stop": "checkpoint"}'
```

### Delete Pipeline

Delete a pipeline and all associated resources.

```http
DELETE /api/v1/pipelines/{pipeline_id}
```

#### Path Parameters

| Parameter | Description |
|-----------|-------------|
| `pipeline_id` | The pipeline ID |

#### Response

```json
{
  "message": "Pipeline deleted successfully"
}
```

#### Example

```bash
curl -X DELETE http://localhost:5115/api/v1/pipelines/pipe_1234567890
```

### Restart Pipeline

Restart a stopped pipeline.

```http
POST /api/v1/pipelines/{pipeline_id}/restart
```

#### Path Parameters

| Parameter | Description |
|-----------|-------------|
| `pipeline_id` | The pipeline ID |

#### Request Body

```json
{
  "force": false
}
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `force` | boolean | Force restart even if state doesn't match |

#### Response

```json
{
  "message": "Pipeline restart initiated",
  "job_id": "job_9876543210"
}
```

#### Example

```bash
curl -X POST http://localhost:5115/api/v1/pipelines/pipe_1234567890/restart \
  -H "Content-Type: application/json" \
  -d '{"force": false}'
```

### Validate Query

Validate a SQL query before creating a pipeline.

```http
POST /api/v1/pipelines/validate_query
```

#### Request Body

```json
{
  "query": "SELECT * FROM events WHERE user_id > 0",
  "udfs": []
}
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | SQL query to validate |
| `udfs` | array | No | User-defined functions |

#### Response

```json
{
  "graph": {
    "nodes": [...],
    "edges": [...]
  },
  "errors": []
}
```

If there are validation errors:

```json
{
  "graph": null,
  "errors": [
    "Table 'events' does not exist",
    "Invalid SQL syntax at line 3"
  ]
}
```

#### Example

```bash
curl -X POST http://localhost:5115/api/v1/pipelines/validate_query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT COUNT(*) FROM orders GROUP BY customer_id"
  }'
```

### Create Preview Pipeline

Create a preview pipeline for testing (limited lifetime, no checkpointing).

```http
POST /api/v1/pipelines/preview
```

#### Request Body

```json
{
  "query": "SELECT * FROM events LIMIT 100",
  "udfs": [],
  "enable_sinks": false
}
```

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | SQL query for preview |
| `udfs` | array | No | User-defined functions |
| `enable_sinks` | boolean | No | Whether to enable sink connectors |

#### Response

Returns a pipeline object with `preview: true`.

#### Example

```bash
curl -X POST http://localhost:5115/api/v1/pipelines/preview \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT * FROM test_source LIMIT 10",
    "enable_sinks": false
  }'
```

## Pipeline Jobs

### List Pipeline Jobs

Get all jobs for a specific pipeline.

```http
GET /api/v1/pipelines/{pipeline_id}/jobs
```

#### Path Parameters

| Parameter | Description |
|-----------|-------------|
| `pipeline_id` | The pipeline ID |

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Maximum number of results |
| `offset` | integer | Number of results to skip |

#### Response

```json
{
  "data": [
    {
      "id": "job_9876543210",
      "pipeline_id": "pipe_1234567890",
      "running_desired": true,
      "state": "Running",
      "run_id": 1,
      "start_time": 1704067200000,
      "finish_time": null,
      "tasks": 4,
      "failure_message": null,
      "created_at": 1704067200000
    }
  ]
}
```

#### Example

```bash
curl http://localhost:5115/api/v1/pipelines/pipe_1234567890/jobs
```

## Stop Modes

When stopping a pipeline, you can specify different modes:

| Mode | Description |
|------|-------------|
| `none` | Pipeline continues running |
| `checkpoint` | Stop after next checkpoint |
| `graceful` | Graceful shutdown with state preservation |
| `immediate` | Immediate stop without checkpoint |
| `force` | Force stop (may lose data) |

## Pipeline States

Pipelines can be in the following states:

| State | Description |
|-------|-------------|
| `Created` | Pipeline created but not started |
| `Running` | Pipeline is actively processing data |
| `Stopping` | Pipeline is in the process of stopping |
| `Stopped` | Pipeline has been stopped |
| `Failed` | Pipeline failed due to an error |
| `Finished` | Pipeline completed successfully |

## Error Codes

Pipeline-specific error codes:

| Code | Description |
|------|-------------|
| `INVALID_QUERY` | SQL query syntax error |
| `TABLE_NOT_FOUND` | Referenced table doesn't exist |
| `DUPLICATE_NAME` | Pipeline name already exists |
| `INVALID_PARALLELISM` | Parallelism value out of range |
| `PIPELINE_RUNNING` | Cannot modify running pipeline |
| `PIPELINE_NOT_FOUND` | Pipeline doesn't exist |

## Best Practices

1. **Validate queries** before creating pipelines
2. **Use appropriate parallelism** based on data volume
3. **Set checkpoint intervals** based on recovery requirements
4. **Monitor pipeline state** regularly
5. **Use preview pipelines** for testing
6. **Handle state transitions** properly
7. **Clean up stopped pipelines** to free resources
8. **Use meaningful names** for pipelines
9. **Document complex queries** with comments
10. **Test with small data** before production deployment