---
sidebar_position: 2
title: Connection Profiles
---


Connection profiles store reusable configuration for connecting to external systems like databases, message queues, and cloud services. They provide a secure way to manage credentials and connection parameters that can be shared across multiple connection tables.

## Overview

Connection profiles enable you to:
- Store credentials and connection parameters securely
- Reuse connection configurations across multiple pipelines
- Test connections before using them in production
- Get autocomplete suggestions for database schemas and tables

## Endpoints

### List Connection Profiles

Get all configured connection profiles.

```http
GET /api/v1/connection_profiles
```

#### Response

```json
{
  "data": [
    {
      "id": "prof_abc123",
      "name": "production-kafka",
      "connector": "kafka",
      "description": "Production Kafka cluster",
      "config": {
        "bootstrap_servers": "kafka1.example.com:9092,kafka2.example.com:9092",
        "security_protocol": "SASL_SSL"
      },
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### Create Connection Profile

Create a new connection profile for external system access.

```http
POST /api/v1/connection_profiles
```

#### Request Body

```json
{
  "name": "production-kafka",
  "connector": "kafka",
  "description": "Production Kafka cluster",
  "config": {
    "bootstrap_servers": "kafka1.example.com:9092,kafka2.example.com:9092",
    "security_protocol": "SASL_SSL",
    "sasl_mechanism": "PLAIN",
    "sasl_username": "user",
    "sasl_password": "secret"
  }
}
```

#### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique name for the connection profile |
| `connector` | string | Yes | Connector type (kafka, mysql, postgres, etc.) |
| `description` | string | No | Human-readable description |
| `config` | object | Yes | Connector-specific configuration |

#### Response

```json
{
  "data": {
    "id": "prof_abc123",
    "name": "production-kafka",
    "connector": "kafka",
    "description": "Production Kafka cluster",
    "config": {
      "bootstrap_servers": "kafka1.example.com:9092,kafka2.example.com:9092",
      "security_protocol": "SASL_SSL"
    },
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

### Test Connection Profile

Test a connection profile configuration before saving it.

```http
POST /api/v1/connection_profiles/test
```

#### Request Body

Same as create connection profile request.

#### Response

```json
{
  "data": {
    "success": true,
    "message": "Connection successful",
    "details": {
      "latency_ms": 45,
      "version": "3.4.0"
    }
  }
}
```

Error response:

```json
{
  "error": {
    "code": "CONNECTION_FAILED",
    "message": "Failed to connect to Kafka cluster",
    "details": {
      "reason": "Authentication failed: Invalid credentials"
    }
  }
}
```

### Delete Connection Profile

Delete an existing connection profile.

```http
DELETE /api/v1/connection_profiles/{id}
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | Connection profile ID |

#### Response

```json
{
  "data": {
    "message": "Connection profile deleted successfully"
  }
}
```

#### Error Responses

- `404 Not Found` - Connection profile not found
- `409 Conflict` - Connection profile is in use by connection tables

### Get Autocomplete Suggestions

Get autocomplete suggestions for schemas, tables, and topics from a connection profile.

```http
GET /api/v1/connection_profiles/{id}/autocomplete
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | Connection profile ID |

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Type of suggestions (schemas, tables, topics) |
| `parent` | string | Parent context (e.g., schema name for tables) |
| `prefix` | string | Filter suggestions by prefix |

#### Response

```json
{
  "data": {
    "suggestions": [
      {
        "value": "public.users",
        "label": "users",
        "type": "table",
        "metadata": {
          "schema": "public",
          "rows": 1000000,
          "size": "120MB"
        }
      },
      {
        "value": "public.orders",
        "label": "orders",
        "type": "table",
        "metadata": {
          "schema": "public",
          "rows": 5000000,
          "size": "2.5GB"
        }
      }
    ]
  }
}
```
