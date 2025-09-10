---
title: Schema Registry
sidebar_position: 3
---

# Schema Registry

Schema Registry provides centralized schema management for your streaming data, ensuring data compatibility and enabling schema evolution across your pipelines.

## Overview

Laminar integrates with Confluent Schema Registry to:
- Automatically resolve schemas by ID
- Validate data against schemas
- Support schema evolution
- Enable format conversion
- Ensure data quality

## Supported Formats

### Avro
Full support for Avro schemas with:
- Schema evolution (backward/forward compatibility)
- Efficient binary encoding
- Complex nested types
- Union types for nullable fields

### JSON Schema
JSON Schema validation with:
- Draft 7 specification support
- Custom validators
- Format validation
- Pattern matching

### Protobuf
Protocol Buffers support including:
- Proto3 syntax
- Nested messages
- Repeated fields
- Well-known types

## Configuration

### Connection Profile Setup

Configure Schema Registry in your Kafka connection profile:

```json
{
  "name": "kafka-with-registry",
  "connector": "kafka",
  "config": {
    "bootstrap_servers": "broker1:9092,broker2:9092",
    "schema_registry": {
      "endpoint": "https://schema-registry.laminar.cloud",
      "api_key": "${SCHEMA_REGISTRY_KEY}",
      "api_secret": "${SCHEMA_REGISTRY_SECRET}"
    }
  }
}
```

### Environment Variables

Store sensitive credentials as environment variables:
```bash
export SCHEMA_REGISTRY_KEY="your-api-key"
export SCHEMA_REGISTRY_SECRET="your-api-secret"
```

## Using with Kafka Sources

### Avro Format

```sql
CREATE TABLE events WITH (
  connector = 'kafka',
  connection = 'kafka-with-registry',
  topic = 'user-events',
  format = 'avro',
  'avro.confluent_schema_registry' = 'true'
);
```

### JSON with Schema Validation

```sql
CREATE TABLE orders WITH (
  connector = 'kafka',
  connection = 'kafka-with-registry',
  topic = 'orders',
  format = 'json',
  'json.confluent_schema_registry' = 'true',
  'json.schema_id' = '123'
);
```

### Protobuf Format

```sql
CREATE TABLE metrics WITH (
  connector = 'kafka',
  connection = 'kafka-with-registry',
  topic = 'metrics',
  format = 'protobuf',
  'protobuf.confluent_schema_registry' = 'true',
  'protobuf.message_type' = 'MetricEvent'
);
```

## Schema Evolution

### Backward Compatibility

New schema can read data written with old schema:

```json
// Version 1
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"}
  ]
}

// Version 2 (backward compatible)
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
```

### Forward Compatibility

Old schema can read data written with new schema:

```json
// Version 1
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": "string", "default": ""}
  ]
}

// Version 2 (forward compatible - removes email)
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"}
  ]
}
```

### Full Compatibility

Both backward and forward compatible - most restrictive but safest.

## Working with Multiple Schemas

### Subject Naming Strategies

Configure how subjects are named for your topics:

```sql
CREATE TABLE multi_event_stream WITH (
  connector = 'kafka',
  topic = 'events',
  format = 'avro',
  'avro.confluent_schema_registry' = 'true',
  'schema.registry.subject' = 'events-value',
  'schema.registry.strategy' = 'topic_record_name'
);
```

Strategies:
- **topic_name**: `<topic>-key` or `<topic>-value`
- **record_name**: `<record-name>`
- **topic_record_name**: `<topic>-<record-name>`

### Schema References

Handle schemas with references to other schemas:

```json
{
  "type": "record",
  "name": "Order",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "customer", "type": "Customer"},
    {"name": "items", "type": {"type": "array", "items": "OrderItem"}}
  ],
  "references": [
    {
      "name": "Customer",
      "subject": "customer-value",
      "version": 1
    },
    {
      "name": "OrderItem",
      "subject": "order-item-value",
      "version": 1
    }
  ]
}
```

## Format Conversion

### Avro to JSON

```sql
CREATE TABLE json_events AS
SELECT 
  id,
  to_json(data) as json_data
FROM avro_events;

INSERT INTO json_sink
SELECT * FROM json_events;
```

### JSON to Avro

```sql
CREATE TABLE avro_output WITH (
  connector = 'kafka',
  topic = 'output-avro',
  format = 'avro',
  'avro.confluent_schema_registry' = 'true',
  'schema.registry.subject' = 'output-avro-value'
)
AS SELECT 
  id,
  name,
  CAST(metadata AS STRUCT<version INT, timestamp BIGINT>) as metadata
FROM json_input;
```

## Schema Management

### Register New Schema

Use the Laminar API to register schemas:

```bash
curl -X POST https://api.laminar.cloud/v1/schemas \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "user-events-value",
    "schema": "{\"type\":\"record\",\"name\":\"User\",...}",
    "schemaType": "AVRO"
  }'
```

### List Schemas

```bash
curl https://api.laminar.cloud/v1/schemas \
  -H "Authorization: Bearer $API_KEY"
```

### Get Schema by ID

```bash
curl https://api.laminar.cloud/v1/schemas/123 \
  -H "Authorization: Bearer $API_KEY"
```

## Best Practices

### 1. Schema Design

- **Use defaults**: Add default values for new fields
- **Avoid renaming**: Never rename fields, add new ones instead
- **Use unions carefully**: Unions make schemas complex
- **Document fields**: Add doc strings to explain field purpose

### 2. Evolution Strategy

- **Start with BACKWARD**: Most common compatibility mode
- **Test changes**: Validate schema changes before deployment
- **Version control**: Track schemas in git
- **Gradual migration**: Roll out changes incrementally

### 3. Performance

- **Cache schemas**: Laminar caches schemas automatically
- **Reuse connections**: Connection pooling for registry clients
- **Batch operations**: Register multiple schemas together
- **Monitor latency**: Track schema resolution times

### 4. Security

- **Use HTTPS**: Always use encrypted connections
- **Rotate credentials**: Regular API key rotation
- **Limit access**: Use read-only credentials where possible
- **Audit changes**: Track schema modifications

## Troubleshooting

### Common Issues

#### Schema Not Found
```
Error: Schema with id 42 not found
```
**Solution**: Ensure schema is registered and accessible

#### Compatibility Violation
```
Error: Schema is not backward compatible
```
**Solution**: Check compatibility rules and adjust schema

#### Authentication Failed
```
Error: 401 Unauthorized
```
**Solution**: Verify API credentials and permissions

#### Network Issues
```
Error: Failed to connect to schema registry
```
**Solution**: Check network connectivity and firewall rules

### Debugging

Enable debug logging:
```sql
SET schema_registry_debug = true;
```

Check schema cache:
```sql
SELECT * FROM system.schema_cache;
```

## Advanced Topics

### Custom Serializers

Implement custom serialization logic:

```rust
#[udf]
fn custom_avro_serialize(data: String, schema_id: i32) -> Vec<u8> {
    // Custom serialization logic
}
```

### Schema Inference

Automatically infer schemas from data:

```sql
CREATE TABLE inferred_schema AS
SELECT 
  infer_schema(json_data) as schema,
  COUNT(*) as count
FROM raw_events
GROUP BY schema;
```

### Multi-Registry Support

Connect to multiple registries:

```json
{
  "primary_registry": {
    "endpoint": "https://primary.registry.com"
  },
  "secondary_registry": {
    "endpoint": "https://secondary.registry.com"
  }
}
```

## Next Steps

- [Kafka Connector](./sources/kafka) - Kafka integration details
- [Data Formats](./formats) - Supported data formats
- [Best Practices](./best-practices) - Production recommendations