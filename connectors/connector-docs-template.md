# Connector Documentation Template and Status

## Documentation Structure

Each connector should have documentation following this structure based on their JSON schemas:

### For Connectors with profile.json and table.json:
1. **Kafka** ✅ - `/docs/connectors/kafka.md`
2. **MQTT** ✅ - `/docs/connectors/mqtt.md`
3. **NATS** - Needs documentation
4. **RabbitMQ** - Needs documentation  
5. **Redis** - Needs documentation (sink only)
6. **Confluent** - Uses kafka table.json, needs separate profile doc

### For Connectors with table.json only:
1. **Webhook** ✅ - `/docs/connectors/webhook.md` (sink only)
2. **WebSocket** - Needs documentation (source only)
3. **SSE** - Needs documentation (source only)
4. **Polling HTTP** - Needs documentation (source only)
5. **Kinesis** - Needs documentation (source and sink)
6. **Fluvio** - Needs documentation (source and sink)
7. **Single File** - Needs documentation (source and sink, for testing)
8. **Nexmark** - Needs documentation (source only, benchmark)
9. **Impulse** - Needs documentation (source only, testing)
10. **Mock Data** - Needs documentation (source only, testing)

### Special Connectors (no JSON files):
1. **Filesystem** - Has custom configuration for different formats
2. **Delta Lake** - Sink only, part of filesystem
3. **Iceberg** - Sink only, part of filesystem
4. **Stdout** - Sink only, for debugging
5. **Blackhole** - Sink only, for testing
6. **Preview** - Internal use

## Template Structure

```markdown
---
sidebar_position: [number]
title: [Connector Name]
---

[Brief description from metadata()]

## Capabilities

- **Source**: Yes/No
- **Sink**: Yes/No
- **Formats**: [List supported formats]
- **[Other key features]**

## Configuration

### Connection Profile Configuration (if profile.json exists)
[Table of fields from profile.json]

### Table Configuration
[Table of fields from table.json]

## API Usage

### Create Connection Profile (if applicable)
[curl example with actual JSON structure]

### Create Connection Table
[curl example with actual JSON structure]

## SQL Usage

### Basic Example
[SQL CREATE TABLE with all parameters]

### Advanced Examples
[Multiple SQL examples showing different configurations]

## Examples

### [Use Case 1]
[Complete example with source/sink/processing]

### [Use Case 2]
[Another practical example]

## Best Practices
[Connector-specific recommendations]

## Limitations
[Known limitations or caveats]
```

## Fields to Extract from JSON Schemas

### From profile.json:
- Field name (convert to appropriate case)
- Type (string, object, array, etc.)
- Required (from "required" array)
- Default value (if specified)
- Description (from "description" field)
- Examples (from "examples" array)
- Format hints (var-str = env vars, uri = URL)
- Sensitive fields (from "sensitive" array)
- OneOf/AnyOf options (for variant configs)

### From table.json:
- Common fields (topic, type, format, etc.)
- Source-specific fields (offset, group_id, etc.)
- Sink-specific fields (commit_mode, key_field, etc.)
- Enums (list all possible values)
- Nested configurations

## SQL Parameter Mapping

JSON field → SQL parameter:
- camelCase → snake_case
- Nested objects → dot notation (e.g., `auth.type`)
- Arrays → comma-separated strings
- OneOf selections → type prefix (e.g., `auth.type = 'sasl'`)

## Priority Order for Documentation

1. **High Priority** (commonly used):
   - Kinesis (AWS users)
   - Redis (caching/state)
   - RabbitMQ (message queuing)
   - WebSocket (real-time)

2. **Medium Priority** (specific use cases):
   - NATS (lightweight messaging)
   - SSE (server-sent events)
   - Polling HTTP (REST APIs)
   - Fluvio (alternative to Kafka)

3. **Low Priority** (specialized):
   - Single File (testing)
   - Nexmark (benchmarking)
   - Impulse (demo/testing)
   - Mock Data (testing)
   - Filesystem/Delta/Iceberg (batch processing)
   - Stdout/Blackhole (debugging)