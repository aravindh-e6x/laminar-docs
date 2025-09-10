---
sidebar_position: 4
title: Connectors
---

# Connectors API

The Connectors API provides information about available data source and sink connectors in Laminar. This endpoint helps you discover supported connectors and their configurations.

## Overview

The Connectors API allows you to:
- List all available connectors
- Get connector metadata and capabilities
- Understand configuration requirements
- Discover supported formats and features

## Endpoints

### List Connectors

Get all available connectors with their metadata and configuration schemas.

```http
GET /api/v1/connectors
```

#### Response

```json
{
  "data": [
    {
      "id": "kafka",
      "name": "Apache Kafka",
      "description": "Apache Kafka connector for streaming data",
      "category": "messaging",
      "type": ["source", "sink"],
      "icon": "kafka-icon.svg",
      "version": "3.4.0",
      "features": {
        "exactly_once": true,
        "schema_registry": true,
        "transactions": true,
        "partitioning": true
      },
      "supported_formats": ["json", "avro", "protobuf", "raw"],
      "configuration_schema": {
        "source": {
          "required": ["topic", "format"],
          "properties": {
            "topic": {
              "type": "string",
              "description": "Kafka topic to consume from"
            },
            "consumer_group": {
              "type": "string",
              "description": "Consumer group ID"
            },
            "offset_reset": {
              "type": "string",
              "enum": ["earliest", "latest"],
              "default": "latest",
              "description": "Where to start consuming"
            },
            "format": {
              "type": "string",
              "enum": ["json", "avro", "protobuf", "raw"],
              "description": "Message format"
            }
          }
        },
        "sink": {
          "required": ["topic", "format"],
          "properties": {
            "topic": {
              "type": "string",
              "description": "Kafka topic to produce to"
            },
            "format": {
              "type": "string",
              "enum": ["json", "avro", "protobuf", "raw"],
              "description": "Message format"
            },
            "compression_type": {
              "type": "string",
              "enum": ["none", "gzip", "snappy", "lz4", "zstd"],
              "default": "none",
              "description": "Compression algorithm"
            }
          }
        }
      }
    },
    {
      "id": "postgresql",
      "name": "PostgreSQL",
      "description": "PostgreSQL database connector",
      "category": "database",
      "type": ["source", "sink"],
      "icon": "postgresql-icon.svg",
      "version": "14.0",
      "features": {
        "cdc": true,
        "bulk_insert": true,
        "upsert": true,
        "transactions": true
      },
      "supported_formats": ["json", "csv"],
      "configuration_schema": {
        "source": {
          "required": ["table"],
          "properties": {
            "table": {
              "type": "string",
              "description": "Table name to read from"
            },
            "schema": {
              "type": "string",
              "default": "public",
              "description": "Database schema"
            },
            "scan_mode": {
              "type": "string",
              "enum": ["full", "incremental"],
              "default": "full",
              "description": "Table scan mode"
            }
          }
        },
        "sink": {
          "required": ["table"],
          "properties": {
            "table": {
              "type": "string",
              "description": "Table name to write to"
            },
            "schema": {
              "type": "string",
              "default": "public",
              "description": "Database schema"
            },
            "write_mode": {
              "type": "string",
              "enum": ["append", "upsert", "overwrite"],
              "default": "append",
              "description": "Write mode"
            },
            "primary_keys": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Primary key columns for upsert"
            }
          }
        }
      }
    },
    {
      "id": "mysql-cdc",
      "name": "MySQL CDC",
      "description": "MySQL Change Data Capture connector",
      "category": "database",
      "type": ["source"],
      "icon": "mysql-icon.svg",
      "version": "8.0",
      "features": {
        "cdc": true,
        "snapshot": true,
        "schema_evolution": true,
        "parallel_snapshot": true
      },
      "supported_formats": ["debezium-json", "canal-json"],
      "configuration_schema": {
        "source": {
          "required": ["database", "table"],
          "properties": {
            "database": {
              "type": "string",
              "description": "Database name"
            },
            "table": {
              "type": "string",
              "description": "Table name or pattern"
            },
            "server_id": {
              "type": "integer",
              "description": "Unique server ID for binlog",
              "minimum": 1,
              "maximum": 2147483647
            },
            "scan_startup_mode": {
              "type": "string",
              "enum": ["initial", "latest", "timestamp", "binlog"],
              "default": "initial",
              "description": "Startup mode"
            }
          }
        }
      }
    },
    {
      "id": "kinesis",
      "name": "Amazon Kinesis",
      "description": "AWS Kinesis Data Streams connector",
      "category": "cloud",
      "type": ["source", "sink"],
      "icon": "kinesis-icon.svg",
      "version": "2.0",
      "features": {
        "auto_scaling": true,
        "encryption": true,
        "cross_region": true
      },
      "supported_formats": ["json", "avro", "raw"],
      "configuration_schema": {
        "source": {
          "required": ["stream_name", "region"],
          "properties": {
            "stream_name": {
              "type": "string",
              "description": "Kinesis stream name"
            },
            "region": {
              "type": "string",
              "description": "AWS region"
            },
            "starting_position": {
              "type": "string",
              "enum": ["TRIM_HORIZON", "LATEST", "AT_TIMESTAMP"],
              "default": "LATEST",
              "description": "Starting position"
            }
          }
        },
        "sink": {
          "required": ["stream_name", "region"],
          "properties": {
            "stream_name": {
              "type": "string",
              "description": "Kinesis stream name"
            },
            "region": {
              "type": "string",
              "description": "AWS region"
            },
            "partition_key": {
              "type": "string",
              "description": "Field to use as partition key"
            }
          }
        }
      }
    },
    {
      "id": "elasticsearch",
      "name": "Elasticsearch",
      "description": "Elasticsearch search and analytics engine",
      "category": "search",
      "type": ["sink"],
      "icon": "elasticsearch-icon.svg",
      "version": "8.0",
      "features": {
        "bulk_indexing": true,
        "upsert": true,
        "routing": true,
        "templates": true
      },
      "supported_formats": ["json"],
      "configuration_schema": {
        "sink": {
          "required": ["index"],
          "properties": {
            "index": {
              "type": "string",
              "description": "Index name or pattern"
            },
            "document_type": {
              "type": "string",
              "default": "_doc",
              "description": "Document type"
            },
            "id_field": {
              "type": "string",
              "description": "Field to use as document ID"
            },
            "bulk_size": {
              "type": "integer",
              "default": 1000,
              "description": "Bulk request size"
            }
          }
        }
      }
    },
    {
      "id": "redis",
      "name": "Redis",
      "description": "Redis in-memory data store",
      "category": "cache",
      "type": ["sink"],
      "icon": "redis-icon.svg",
      "version": "7.0",
      "features": {
        "clustering": true,
        "pub_sub": true,
        "transactions": true,
        "ttl": true
      },
      "supported_formats": ["json", "raw"],
      "configuration_schema": {
        "sink": {
          "required": ["command"],
          "properties": {
            "command": {
              "type": "string",
              "enum": ["SET", "HSET", "LPUSH", "RPUSH", "SADD", "ZADD", "PUBLISH"],
              "description": "Redis command to execute"
            },
            "key_field": {
              "type": "string",
              "description": "Field to use as Redis key"
            },
            "value_field": {
              "type": "string",
              "description": "Field to use as value"
            },
            "ttl_seconds": {
              "type": "integer",
              "description": "Time-to-live in seconds"
            }
          }
        }
      }
    },
    {
      "id": "webhook",
      "name": "Webhook",
      "description": "HTTP webhook connector",
      "category": "http",
      "type": ["source", "sink"],
      "icon": "webhook-icon.svg",
      "version": "1.0",
      "features": {
        "authentication": true,
        "retry": true,
        "batching": true
      },
      "supported_formats": ["json", "form", "raw"],
      "configuration_schema": {
        "source": {
          "required": ["path"],
          "properties": {
            "path": {
              "type": "string",
              "description": "HTTP endpoint path"
            },
            "method": {
              "type": "string",
              "enum": ["GET", "POST", "PUT"],
              "default": "POST",
              "description": "HTTP method"
            },
            "auth_type": {
              "type": "string",
              "enum": ["none", "basic", "bearer", "api_key"],
              "default": "none",
              "description": "Authentication type"
            }
          }
        },
        "sink": {
          "required": ["url"],
          "properties": {
            "url": {
              "type": "string",
              "description": "Target URL"
            },
            "method": {
              "type": "string",
              "enum": ["POST", "PUT", "PATCH"],
              "default": "POST",
              "description": "HTTP method"
            },
            "headers": {
              "type": "object",
              "description": "Custom headers"
            },
            "retry_count": {
              "type": "integer",
              "default": 3,
              "description": "Number of retries"
            }
          }
        }
      }
    },
    {
      "id": "filesystem",
      "name": "Filesystem",
      "description": "File system connector (local, S3, GCS, Azure)",
      "category": "storage",
      "type": ["source", "sink"],
      "icon": "filesystem-icon.svg",
      "version": "1.0",
      "features": {
        "cloud_storage": true,
        "partitioning": true,
        "compression": true,
        "format_detection": true
      },
      "supported_formats": ["json", "csv", "parquet", "avro", "orc"],
      "configuration_schema": {
        "source": {
          "required": ["path", "format"],
          "properties": {
            "path": {
              "type": "string",
              "description": "File or directory path"
            },
            "format": {
              "type": "string",
              "enum": ["json", "csv", "parquet", "avro", "orc"],
              "description": "File format"
            },
            "pattern": {
              "type": "string",
              "description": "File name pattern (glob)"
            },
            "scan_interval": {
              "type": "string",
              "default": "1m",
              "description": "Directory scan interval"
            }
          }
        },
        "sink": {
          "required": ["path", "format"],
          "properties": {
            "path": {
              "type": "string",
              "description": "Output directory path"
            },
            "format": {
              "type": "string",
              "enum": ["json", "csv", "parquet", "avro", "orc"],
              "description": "Output format"
            },
            "partition_by": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Partition columns"
            },
            "compression": {
              "type": "string",
              "enum": ["none", "gzip", "snappy", "lz4", "zstd"],
              "default": "none",
              "description": "Compression type"
            }
          }
        }
      }
    },
    {
      "id": "iceberg",
      "name": "Apache Iceberg",
      "description": "Apache Iceberg table format",
      "category": "storage",
      "type": ["sink"],
      "icon": "iceberg-icon.svg",
      "version": "1.0",
      "features": {
        "acid_transactions": true,
        "time_travel": true,
        "schema_evolution": true,
        "partition_evolution": true
      },
      "supported_formats": ["parquet", "avro", "orc"],
      "configuration_schema": {
        "sink": {
          "required": ["catalog", "database", "table"],
          "properties": {
            "catalog": {
              "type": "string",
              "description": "Iceberg catalog name"
            },
            "database": {
              "type": "string",
              "description": "Database name"
            },
            "table": {
              "type": "string",
              "description": "Table name"
            },
            "write_mode": {
              "type": "string",
              "enum": ["append", "overwrite", "upsert"],
              "default": "append",
              "description": "Write mode"
            }
          }
        }
      }
    },
    {
      "id": "delta",
      "name": "Delta Lake",
      "description": "Delta Lake table format",
      "category": "storage",
      "type": ["sink"],
      "icon": "delta-icon.svg",
      "version": "2.0",
      "features": {
        "acid_transactions": true,
        "time_travel": true,
        "schema_enforcement": true,
        "z_ordering": true
      },
      "supported_formats": ["parquet"],
      "configuration_schema": {
        "sink": {
          "required": ["path"],
          "properties": {
            "path": {
              "type": "string",
              "description": "Delta table path"
            },
            "write_mode": {
              "type": "string",
              "enum": ["append", "overwrite", "merge"],
              "default": "append",
              "description": "Write mode"
            },
            "merge_keys": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Keys for merge operation"
            }
          }
        }
      }
    }
  ]
}
```

## Connector Categories

Connectors are organized into categories based on their functionality:

| Category | Description | Examples |
|----------|-------------|----------|
| `messaging` | Message queue and streaming platforms | Kafka, RabbitMQ, NATS |
| `database` | Relational and NoSQL databases | PostgreSQL, MySQL, MongoDB |
| `cloud` | Cloud service providers | AWS Kinesis, Google Pub/Sub |
| `storage` | File and object storage systems | S3, GCS, HDFS, Local FS |
| `search` | Search and analytics engines | Elasticsearch, Solr |
| `cache` | In-memory data stores | Redis, Memcached |
| `http` | HTTP-based endpoints | Webhook, REST API |
| `iot` | IoT protocols | MQTT, CoAP |

## Connector Types

Each connector can support one or more types:

| Type | Description |
|------|-------------|
| `source` | Can read data from external systems |
| `sink` | Can write data to external systems |

## Connector Features

Common features across connectors:

| Feature | Description |
|---------|-------------|
| `exactly_once` | Supports exactly-once semantics |
| `cdc` | Change Data Capture support |
| `transactions` | Transactional guarantees |
| `schema_registry` | Schema registry integration |
| `bulk_insert` | Bulk/batch operations |
| `upsert` | Insert or update operations |
| `partitioning` | Data partitioning support |
| `encryption` | Data encryption in transit/at rest |
| `compression` | Data compression support |
| `auto_scaling` | Automatic scaling capabilities |

## Configuration Schema

Each connector provides a JSON Schema that describes its configuration requirements:

### Schema Structure

```json
{
  "source": {
    "required": ["field1", "field2"],
    "properties": {
      "field1": {
        "type": "string",
        "description": "Field description",
        "enum": ["option1", "option2"],
        "default": "option1"
      },
      "field2": {
        "type": "integer",
        "description": "Numeric field",
        "minimum": 1,
        "maximum": 100
      }
    }
  },
  "sink": {
    "required": ["field3"],
    "properties": {
      "field3": {
        "type": "string",
        "description": "Required field"
      }
    }
  }
}
```

### Field Types

- `string` - Text values
- `integer` - Whole numbers
- `number` - Floating-point numbers
- `boolean` - True/false values
- `array` - List of values
- `object` - Nested configuration

## Examples

### Get All Connectors

```bash
curl http://localhost:5115/api/v1/connectors
```

### Filter by Category

```bash
curl "http://localhost:5115/api/v1/connectors?category=database"
```

### Filter by Type

```bash
curl "http://localhost:5115/api/v1/connectors?type=source"
```

### Using Connector Information

```python
import requests

# Get all connectors
response = requests.get("http://localhost:5115/api/v1/connectors")
connectors = response.json()["data"]

# Find Kafka connector
kafka = next(c for c in connectors if c["id"] == "kafka")

# Get configuration schema
source_schema = kafka["configuration_schema"]["source"]
required_fields = source_schema["required"]
properties = source_schema["properties"]

print(f"Kafka source requires: {required_fields}")
print(f"Supported formats: {kafka['supported_formats']}")
```

## Best Practices

1. **Check connector availability**: Verify connector is available before creating connection tables
2. **Review configuration schema**: Understand required and optional fields
3. **Validate formats**: Ensure your data format is supported
4. **Check features**: Verify the connector supports required features
5. **Use appropriate connector**: Choose connectors optimized for your use case
6. **Review version compatibility**: Ensure connector version meets requirements

## Notes

- Connector availability may vary based on Laminar deployment
- Some connectors require additional dependencies or licenses
- Configuration schemas are enforced when creating connection tables
- Connector versions indicate the compatible external system version