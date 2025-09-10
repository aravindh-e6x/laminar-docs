---
sidebar_position: 1
title: Overview
---


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

Most Laminar API endpoints require Bearer token authentication. Include the authorization header in your requests:

```http
Authorization: Bearer YOUR_TOKEN
```

### Endpoints Requiring Authentication

All endpoints require authentication except:
- `GET /api/v1/ping` - Health check endpoint
- `GET /api/v1/connectors` - List available connectors

### Authentication Methods

In production environments, configure one of:
- **Bearer Tokens**: For API access
- **Service Accounts**: For service-to-service communication
- **OAuth 2.0**: For third-party integrations (future)

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

Successful responses return the data directly:

```json
{...}  // Single object
```

Or for collections with pagination:

```json
{
  "data": [...],  // Array of items
  "has_more": true  // Only for paginated endpoints
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


## API Categories

The Laminar API is organized into the following categories:

### Core Resources

- **[Connection Profiles](./connection-profiles)**: Store reusable connection configurations
- **[Connectors](./connectors)**: Discover available connector types
- **[Connection Tables](./connection-tables)**: Define data sources and sinks
- **[UDFs](./udfs)**: Manage user-defined functions
- **[Pipelines](./pipelines)**: Create and manage streaming pipelines
- **[Jobs](./jobs)**: Monitor and control pipeline executions

## Quick Start

### 1. Check API Health

```bash
curl http://localhost:5115/api/v1/ping
```

Note: This endpoint does not require authentication.

Response:
```json
"Pong"
```

### 2. Create a Pipeline

```bash
curl -X POST http://localhost:5115/api/v1/pipelines \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "name": "my_pipeline",
    "query": "SELECT * FROM events",
    "parallelism": 1
  }'
```

### 3. List Pipelines

```bash
curl http://localhost:5115/api/v1/pipelines \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### 4. Get Pipeline Status

```bash
curl http://localhost:5115/api/v1/pipelines/{pipeline_id}/jobs \
  -H "Authorization: Bearer YOUR_TOKEN"
```



## Support

For API support and questions:

- **Documentation**: This guide and endpoint references
- **Community**: Join our Slack channel
- **Issues**: Report bugs on GitHub
- **Examples**: See our example repository