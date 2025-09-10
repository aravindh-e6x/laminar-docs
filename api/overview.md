---
sidebar_position: 1
title: Overview
---

# API Overview

Laminar provides a comprehensive REST API for managing pipelines, connections, jobs, and all aspects of the streaming platform. The API enables programmatic control over the entire system, making it easy to integrate Laminar into your existing infrastructure and workflows.

## Base URL

The API is available at:
```
http://<laminar-host>:5115/api
```

For local development:
```
http://localhost:5115/api
```

## API Version

The current API version is `v1`. All endpoints are prefixed with `/v1`:
```
http://localhost:5115/api/v1/<endpoint>
```

## Authentication

Currently, the Laminar API does not require authentication by default. In production environments, you should configure authentication using one of the following methods:

- **API Keys**: Static keys for service-to-service communication
- **JWT Tokens**: For user authentication
- **OAuth 2.0**: For third-party integrations

## Content Type

All API requests and responses use JSON:

```http
Content-Type: application/json
Accept: application/json
```

## HTTP Methods

The API follows RESTful conventions:

| Method | Usage |
|--------|-------|
| `GET` | Retrieve resources |
| `POST` | Create new resources |
| `PATCH` | Update existing resources |
| `DELETE` | Remove resources |

## Response Format

All successful responses follow this structure:

```json
{
  "data": {...},  // or [...] for collections
  "meta": {
    "timestamp": "2024-01-01T00:00:00Z",
    "version": "v1"
  }
}
```

For collections:

```json
{
  "data": [...],
  "meta": {
    "total": 100,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

## Error Handling

Error responses follow a consistent format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid pipeline configuration",
    "details": {
      "field": "query",
      "reason": "SQL syntax error at line 3"
    }
  }
}
```

Common error codes:

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Invalid request parameters |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource already exists |
| `INTERNAL_ERROR` | 500 | Server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |

## Rate Limiting

API requests are rate-limited to prevent abuse:

- **Default limit**: 1000 requests per minute
- **Burst limit**: 100 requests per second

Rate limit headers are included in responses:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1704067260
```

## Pagination

List endpoints support pagination using query parameters:

```http
GET /api/v1/pipelines?limit=20&offset=40
```

Parameters:
- `limit`: Maximum number of items to return (default: 20, max: 100)
- `offset`: Number of items to skip (default: 0)
- `starting_after`: Cursor for forward pagination
- `ending_before`: Cursor for backward pagination

## Filtering and Sorting

Many endpoints support filtering and sorting:

```http
GET /api/v1/pipelines?state=running&sort=created_at:desc
```

Common parameters:
- `state`: Filter by resource state
- `name`: Filter by name (supports wildcards)
- `created_after`: Filter by creation date
- `sort`: Sort results (format: `field:direction`)

## Async Operations

Long-running operations return immediately with a status:

```json
{
  "operation_id": "op_123456",
  "status": "pending",
  "message": "Pipeline creation initiated"
}
```

Check operation status:
```http
GET /api/v1/operations/op_123456
```

## API Categories

The Laminar API is organized into the following categories:

### Core Resources

- **[Pipelines](./pipelines)**: Create and manage streaming pipelines
- **[Jobs](./jobs)**: Monitor and control pipeline executions  
- **[Connection Profiles](./connection-profiles)**: Store reusable connection configurations
- **[Connection Tables](./connection-tables)**: Define data sources and sinks
- **[Connectors](./connectors)**: Discover available connector types
- **[UDFs](./udfs)**: Manage user-defined functions

## Quick Start

### 1. Check API Health

```bash
curl http://localhost:5115/api/v1/ping
```

Response:
```json
"Pong"
```

### 2. Create a Pipeline

```bash
curl -X POST http://localhost:5115/api/v1/pipelines \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my_pipeline",
    "query": "SELECT * FROM events",
    "parallelism": 1
  }'
```

### 3. List Pipelines

```bash
curl http://localhost:5115/api/v1/pipelines
```

### 4. Get Pipeline Status

```bash
curl http://localhost:5115/api/v1/pipelines/{pipeline_id}/jobs
```

## Client Libraries

Official client libraries are available for:

- **Python**: `pip install laminar-client`
- **JavaScript/TypeScript**: `npm install @laminar/client`
- **Go**: `go get github.com/laminar/go-client`
- **Java**: Maven/Gradle packages available

## OpenAPI Specification

The complete OpenAPI 3.0 specification is available at:

```
http://localhost:5115/api/v1/openapi.json
```

You can use this with tools like:
- Swagger UI
- Postman
- OpenAPI Generator (for client generation)

## WebSocket API

For real-time updates, Laminar provides a WebSocket API:

```javascript
const ws = new WebSocket('ws://localhost:5115/api/v1/ws');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Pipeline update:', data);
};
```

## Best Practices

1. **Use appropriate HTTP methods**: GET for reads, POST for creates, etc.
2. **Handle pagination**: Don't fetch all records at once
3. **Implement retry logic**: For transient failures
4. **Cache responses**: When appropriate
5. **Use webhooks**: For event-driven updates
6. **Monitor rate limits**: Respect rate limit headers
7. **Validate inputs**: Before sending requests
8. **Handle errors gracefully**: Parse error responses
9. **Use async operations**: For long-running tasks
10. **Keep connections alive**: Use connection pooling

## Examples

### Using cURL

```bash
# Create a pipeline
curl -X POST http://localhost:5115/api/v1/pipelines \
  -H "Content-Type: application/json" \
  -d @pipeline.json

# Get pipeline details
curl http://localhost:5115/api/v1/pipelines/pipe_123

# Update pipeline
curl -X PATCH http://localhost:5115/api/v1/pipelines/pipe_123 \
  -H "Content-Type: application/json" \
  -d '{"parallelism": 2}'

# Delete pipeline
curl -X DELETE http://localhost:5115/api/v1/pipelines/pipe_123
```

### Using Python

```python
import requests

# Create client
api_url = "http://localhost:5115/api/v1"

# Create pipeline
response = requests.post(
    f"{api_url}/pipelines",
    json={
        "name": "my_pipeline",
        "query": "SELECT * FROM events",
        "parallelism": 1
    }
)
pipeline = response.json()

# Get status
jobs = requests.get(
    f"{api_url}/pipelines/{pipeline['id']}/jobs"
).json()
```

### Using JavaScript

```javascript
// Create pipeline
const response = await fetch('http://localhost:5115/api/v1/pipelines', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'my_pipeline',
    query: 'SELECT * FROM events',
    parallelism: 1
  })
});

const pipeline = await response.json();
```

## Support

For API support and questions:

- **Documentation**: This guide and endpoint references
- **Community**: Join our Slack channel
- **Issues**: Report bugs on GitHub
- **Examples**: See our example repository