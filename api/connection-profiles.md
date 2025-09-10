---
sidebar_position: 2
title: Connection Profiles
---

# Connection Profiles API

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

## Connector-Specific Configurations

### Kafka

```json
{
  "config": {
    "bootstrap_servers": "localhost:9092",
    "security_protocol": "SASL_SSL",
    "sasl_mechanism": "PLAIN",
    "sasl_username": "user",
    "sasl_password": "password",
    "ssl_ca_location": "/path/to/ca-cert",
    "ssl_certificate_location": "/path/to/client-cert",
    "ssl_key_location": "/path/to/client-key"
  }
}
```

### PostgreSQL

```json
{
  "config": {
    "host": "localhost",
    "port": 5432,
    "database": "mydb",
    "username": "user",
    "password": "password",
    "ssl_mode": "require",
    "ssl_cert": "-----BEGIN CERTIFICATE-----...",
    "connection_timeout": 30
  }
}
```

### MySQL

```json
{
  "config": {
    "host": "localhost",
    "port": 3306,
    "database": "mydb",
    "username": "user",
    "password": "password",
    "ssl_mode": "REQUIRED",
    "ssl_ca": "/path/to/ca.pem"
  }
}
```

### AWS Kinesis

```json
{
  "config": {
    "region": "us-east-1",
    "access_key_id": "AKIAIOSFODNN7EXAMPLE",
    "secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "session_token": "optional-session-token",
    "endpoint": "https://kinesis.us-east-1.amazonaws.com"
  }
}
```

### Redis

```json
{
  "config": {
    "host": "localhost",
    "port": 6379,
    "password": "password",
    "database": 0,
    "tls_enabled": true,
    "cluster_mode": false
  }
}
```

### Elasticsearch

```json
{
  "config": {
    "nodes": ["http://localhost:9200"],
    "username": "elastic",
    "password": "password",
    "api_key": "alternative-to-username-password",
    "cloud_id": "for-elastic-cloud",
    "ca_cert": "-----BEGIN CERTIFICATE-----..."
  }
}
```

## Examples

### Create a Kafka Connection Profile

```bash
curl -X POST http://localhost:5115/api/v1/connection_profiles \
  -H "Content-Type: application/json" \
  -d '{
    "name": "dev-kafka",
    "connector": "kafka",
    "description": "Development Kafka cluster",
    "config": {
      "bootstrap_servers": "localhost:9092"
    }
  }'
```

### Test MySQL Connection

```bash
curl -X POST http://localhost:5115/api/v1/connection_profiles/test \
  -H "Content-Type: application/json" \
  -d '{
    "connector": "mysql",
    "config": {
      "host": "localhost",
      "port": 3306,
      "database": "testdb",
      "username": "root",
      "password": "password"
    }
  }'
```

### Get Table Suggestions for PostgreSQL

```bash
curl "http://localhost:5115/api/v1/connection_profiles/prof_123/autocomplete?type=tables&parent=public"
```

## Best Practices

1. **Use descriptive names**: Choose clear, meaningful names for connection profiles
2. **Test before saving**: Always test connection profiles before creating them
3. **Secure credentials**: Store sensitive credentials using environment variables or secret management systems
4. **Reuse profiles**: Create shared profiles for commonly used connections
5. **Document configurations**: Add clear descriptions to help other users understand the purpose
6. **Regular validation**: Periodically test connection profiles to ensure they're still valid
7. **Minimize permissions**: Use credentials with minimal required permissions
8. **Use SSL/TLS**: Always enable encryption for production connections

## Error Handling

Common error scenarios:

### Invalid Credentials

```json
{
  "error": {
    "code": "AUTHENTICATION_FAILED",
    "message": "Failed to authenticate with the database",
    "details": {
      "connector": "postgresql",
      "reason": "password authentication failed for user 'admin'"
    }
  }
}
```

### Network Issues

```json
{
  "error": {
    "code": "CONNECTION_TIMEOUT",
    "message": "Connection timed out",
    "details": {
      "host": "db.example.com",
      "port": 5432,
      "timeout_seconds": 30
    }
  }
}
```

### Invalid Configuration

```json
{
  "error": {
    "code": "INVALID_CONFIG",
    "message": "Invalid connection configuration",
    "details": {
      "field": "port",
      "reason": "Port must be between 1 and 65535"
    }
  }
}
```

## Security Considerations

- Connection profiles may contain sensitive credentials
- Credentials are encrypted at rest in the database
- Use service accounts with minimal permissions
- Rotate credentials regularly
- Audit connection profile access
- Use network security (VPNs, private endpoints) where possible
- Enable SSL/TLS for all production connections