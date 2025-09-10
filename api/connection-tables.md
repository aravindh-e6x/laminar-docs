---
sidebar_position: 3
title: Connection Tables
---


Connection tables define data sources and sinks for your streaming pipelines. They specify the schema, format, and connection details for reading from or writing to external systems.

## Overview

Connection tables allow you to:
- Define schemas for source and sink data
- Configure data formats (JSON, Avro, Parquet, etc.)
- Set up event time and watermark configurations
- Test schemas and connections before deployment
- Create both source and sink tables

## Endpoints

### List Connection Tables

Get all configured connection tables.

```http
GET /api/v1/connection_tables
```

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | integer | Maximum number of items to return (default: 20, max: 100) |
| `offset` | integer | Number of items to skip (default: 0) |
| `type` | string | Filter by table type (source, sink) |
| `connector` | string | Filter by connector type |

#### Response

```json
{
  "data": [
    {
      "id": "tbl_abc123",
      "name": "events",
      "connector": "kafka",
      "connection_profile_id": "prof_xyz789",
      "table_type": "source",
      "schema": {
        "fields": [
          {
            "name": "event_id",
            "data_type": "STRING",
            "nullable": false
          },
          {
            "name": "timestamp",
            "data_type": "TIMESTAMP",
            "nullable": false
          },
          {
            "name": "user_id",
            "data_type": "BIGINT",
            "nullable": true
          },
          {
            "name": "data",
            "data_type": "JSON",
            "nullable": true
          }
        ]
      },
      "config": {
        "topic": "events",
        "format": "json",
        "consumer_group": "laminar-consumer"
      },
      "event_time_field": "timestamp",
      "watermark_delay": "5 seconds",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "meta": {
    "total": 10,
    "limit": 20,
    "offset": 0
  }
}
```

### Create Connection Table

Create a new connection table for data sources or sinks.

```http
POST /api/v1/connection_tables
```

#### Request Body

```json
{
  "name": "user_events",
  "connector": "kafka",
  "connection_profile_id": "prof_xyz789",
  "table_type": "source",
  "schema": {
    "fields": [
      {
        "name": "event_id",
        "data_type": "STRING",
        "nullable": false
      },
      {
        "name": "timestamp",
        "data_type": "TIMESTAMP",
        "nullable": false
      },
      {
        "name": "user_id",
        "data_type": "BIGINT",
        "nullable": true
      },
      {
        "name": "event_type",
        "data_type": "STRING",
        "nullable": false
      },
      {
        "name": "properties",
        "data_type": "JSON",
        "nullable": true
      }
    ]
  },
  "config": {
    "topic": "user-events",
    "format": "json",
    "consumer_group": "analytics-pipeline"
  },
  "event_time_field": "timestamp",
  "watermark_delay": "10 seconds"
}
```

#### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique name for the connection table |
| `connector` | string | Yes | Connector type (kafka, kinesis, postgres, etc.) |
| `connection_profile_id` | string | No | ID of connection profile to use |
| `table_type` | string | Yes | Type of table (source or sink) |
| `schema` | object | Yes | Table schema definition |
| `config` | object | Yes | Connector-specific configuration |
| `event_time_field` | string | No | Field to use for event time (sources only) |
| `watermark_delay` | string | No | Watermark delay specification |

#### Response

```json
{
  "data": {
    "id": "tbl_new123",
    "name": "user_events",
    "connector": "kafka",
    "connection_profile_id": "prof_xyz789",
    "table_type": "source",
    "schema": {
      "fields": [...]
    },
    "config": {
      "topic": "user-events",
      "format": "json",
      "consumer_group": "analytics-pipeline"
    },
    "event_time_field": "timestamp",
    "watermark_delay": "10 seconds",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

### Test Connection Table

Test a connection table configuration before saving.

```http
POST /api/v1/connection_tables/test
```

#### Request Body

Same as create connection table request.

#### Response

```json
{
  "data": {
    "success": true,
    "message": "Connection table test successful",
    "sample_data": [
      {
        "event_id": "evt_123",
        "timestamp": "2024-01-01T10:00:00Z",
        "user_id": 12345,
        "event_type": "click",
        "properties": {"page": "home"}
      }
    ],
    "statistics": {
      "message_rate": 1000,
      "lag": 0,
      "partitions": 10
    }
  }
}
```

### Test Schema

Validate a schema definition against actual data.

```http
POST /api/v1/connection_tables/schemas/test
```

#### Request Body

```json
{
  "connector": "kafka",
  "connection_profile_id": "prof_xyz789",
  "schema": {
    "fields": [
      {
        "name": "id",
        "data_type": "BIGINT",
        "nullable": false
      },
      {
        "name": "data",
        "data_type": "STRING",
        "nullable": true
      }
    ]
  },
  "config": {
    "topic": "test-topic",
    "format": "json"
  }
}
```

#### Response

```json
{
  "data": {
    "valid": true,
    "inferred_schema": {
      "fields": [
        {
          "name": "id",
          "data_type": "BIGINT",
          "nullable": false,
          "sample_values": [1, 2, 3]
        },
        {
          "name": "data",
          "data_type": "STRING",
          "nullable": true,
          "sample_values": ["test", null, "data"]
        }
      ]
    },
    "compatibility": {
      "forward_compatible": true,
      "backward_compatible": true
    }
  }
}
```

### Delete Connection Table

Delete an existing connection table.

```http
DELETE /api/v1/connection_tables/{id}
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | Connection table ID |

#### Response

```json
{
  "data": {
    "message": "Connection table deleted successfully"
  }
}
```

#### Error Responses

- `404 Not Found` - Connection table not found
- `409 Conflict` - Connection table is in use by pipelines

## Schema Definition

### Data Types

Supported data types for schema fields:

| Type | Description | Example |
|------|-------------|---------|
| `BOOLEAN` | Boolean value | true, false |
| `TINYINT` | 8-bit integer | -128 to 127 |
| `SMALLINT` | 16-bit integer | -32,768 to 32,767 |
| `INT` | 32-bit integer | -2^31 to 2^31-1 |
| `BIGINT` | 64-bit integer | -2^63 to 2^63-1 |
| `FLOAT` | 32-bit floating point | 3.14 |
| `DOUBLE` | 64-bit floating point | 3.14159265359 |
| `DECIMAL(p,s)` | Fixed precision decimal | DECIMAL(10,2) |
| `STRING` | Variable-length string | "hello" |
| `BINARY` | Binary data | 0x48656C6C6F |
| `DATE` | Date without time | 2024-01-01 |
| `TIME` | Time without date | 14:30:00 |
| `TIMESTAMP` | Date and time | 2024-01-01T14:30:00Z |
| `INTERVAL` | Time interval | INTERVAL '1' DAY |
| `ARRAY<T>` | Array of elements | `ARRAY<INT>` |
| `MAP<K,V>` | Key-value pairs | `MAP<STRING,INT>` |
| `STRUCT<...>` | Nested structure | `STRUCT<name:STRING,age:INT>` |
| `JSON` | JSON data | `{"key": "value"}` |

### Schema Field Definition

```json
{
  "name": "field_name",
  "data_type": "STRING",
  "nullable": true,
  "description": "Field description",
  "metadata": {
    "format": "email",
    "min_length": 5,
    "max_length": 255
  }
}
```

## Format Configurations

### JSON Format

```json
{
  "format": "json",
  "format_options": {
    "ignore_parse_errors": true,
    "timestamp_format": "ISO8601"
  }
}
```

### Avro Format

```json
{
  "format": "avro",
  "format_options": {
    "schema_registry_url": "http://localhost:8081",
    "schema_id": 123,
    "schema_subject": "user-events-value"
  }
}
```

### Parquet Format

```json
{
  "format": "parquet",
  "format_options": {
    "compression": "snappy",
    "row_group_size": 100000
  }
}
```

### CSV Format

```json
{
  "format": "csv",
  "format_options": {
    "delimiter": ",",
    "quote": "\"",
    "escape": "\\",
    "header": true,
    "null_string": "NULL"
  }
}
```
