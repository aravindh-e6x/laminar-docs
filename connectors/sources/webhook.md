---
sidebar_position: 5
title: Webhook
---

# Webhook Source Connector

The Webhook source connector enables Laminar to receive data through HTTP/HTTPS webhooks, providing a REST endpoint for external systems to push data into your streaming pipelines.

## Overview

The Webhook connector provides:
- **HTTP/HTTPS endpoints** for receiving webhook data
- **Multiple authentication methods** (Basic, Bearer, API Key, HMAC)
- **Request validation** and signature verification
- **Automatic retries** and acknowledgments
- **Rate limiting** and throttling
- **Request buffering** and batching
- **Custom headers** and response codes
- **Multiple content types** (JSON, XML, Form data, Raw)

## Configuration

### Basic HTTP Webhook

```sql
CREATE CONNECTION webhook_source TO webhook (
    port = 8080,
    path = '/webhook',
    format = 'json'
);
```

### HTTPS Webhook with TLS

```sql
CREATE CONNECTION webhook_https TO webhook (
    port = 8443,
    path = '/secure-webhook',
    format = 'json',
    tls_cert = '/certs/server.crt',
    tls_key = '/certs/server.key',
    tls_enabled = true
);
```

### Configuration Options

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `port` | integer | Yes | - | Port to listen on |
| `path` | string | No | `/webhook` | URL path for webhook endpoint |
| `format` | string | Yes | - | Data format (json, xml, form, raw) |
| `bind_address` | string | No | `0.0.0.0` | Network interface to bind |
| `max_body_size` | integer | No | 10485760 | Max request body size (bytes) |
| `timeout_ms` | integer | No | 30000 | Request timeout |
| `buffer_size` | integer | No | 1000 | Request buffer size |
| `rate_limit` | integer | No | - | Max requests per second |

## Authentication

### No Authentication (Open)

```sql
CREATE CONNECTION webhook_open TO webhook (
    port = 8080,
    path = '/public',
    format = 'json',
    auth_type = 'none'
);
```

### Basic Authentication

```sql
CREATE CONNECTION webhook_basic TO webhook (
    port = 8080,
    path = '/protected',
    format = 'json',
    auth_type = 'basic',
    auth_username = 'webhook_user',
    auth_password = '${secret:webhook_password}'
);
```

### Bearer Token

```sql
CREATE CONNECTION webhook_bearer TO webhook (
    port = 8080,
    path = '/api',
    format = 'json',
    auth_type = 'bearer',
    auth_token = '${secret:api_token}'
);
```

### API Key Authentication

```sql
CREATE CONNECTION webhook_api_key TO webhook (
    port = 8080,
    path = '/events',
    format = 'json',
    auth_type = 'api_key',
    api_key_header = 'X-API-Key',
    api_key_value = '${secret:api_key}'
);
```

### HMAC Signature Verification

```sql
CREATE CONNECTION webhook_hmac TO webhook (
    port = 8080,
    path = '/signed',
    format = 'json',
    auth_type = 'hmac',
    hmac_secret = '${secret:hmac_secret}',
    hmac_header = 'X-Hub-Signature-256',
    hmac_algorithm = 'sha256'
);
```

## Data Formats

### JSON Webhook

```sql
CREATE CONNECTION webhook_json TO webhook (
    port = 8080,
    path = '/json',
    format = 'json',
    json_path = '$.data'  -- Optional JSON path extraction
);

-- Usage
CREATE TABLE webhook_events (
    id TEXT,
    event_type TEXT,
    timestamp TIMESTAMP,
    payload JSON
) WITH (
    connector = 'webhook_json'
);
```

### XML Webhook

```sql
CREATE CONNECTION webhook_xml TO webhook (
    port = 8080,
    path = '/xml',
    format = 'xml',
    xml_root = 'event'  -- Root element name
);
```

### Form Data

```sql
CREATE CONNECTION webhook_form TO webhook (
    port = 8080,
    path = '/form',
    format = 'form'
);

-- Access form fields
CREATE TABLE form_submissions (
    name TEXT,
    email TEXT,
    message TEXT,
    submitted_at TIMESTAMP DEFAULT NOW()
) WITH (
    connector = 'webhook_form'
);
```

### Raw Binary

```sql
CREATE CONNECTION webhook_raw TO webhook (
    port = 8080,
    path = '/binary',
    format = 'raw'
);

-- Handle raw data
CREATE TABLE raw_payloads (
    data BYTEA,
    content_type TEXT,
    content_length INT,
    received_at TIMESTAMP DEFAULT NOW()
) WITH (
    connector = 'webhook_raw',
    include_headers = true
);
```

## Common Webhook Integrations

### GitHub Webhooks

```sql
CREATE CONNECTION github_webhook TO webhook (
    port = 8080,
    path = '/github',
    format = 'json',
    auth_type = 'hmac',
    hmac_secret = '${secret:github_webhook_secret}',
    hmac_header = 'X-Hub-Signature-256',
    hmac_algorithm = 'sha256'
);

-- Process GitHub events
CREATE PIPELINE github_processor AS
SELECT 
    headers->>'X-GitHub-Event' as event_type,
    headers->>'X-GitHub-Delivery' as delivery_id,
    JSON_EXTRACT(payload, '$.repository.full_name') as repo,
    JSON_EXTRACT(payload, '$.action') as action,
    JSON_EXTRACT(payload, '$.sender.login') as user,
    NOW() as processed_at
FROM github_webhook
WHERE headers->>'X-GitHub-Event' IN ('push', 'pull_request', 'issues');
```

### Stripe Webhooks

```sql
CREATE CONNECTION stripe_webhook TO webhook (
    port = 8080,
    path = '/stripe',
    format = 'json',
    auth_type = 'hmac',
    hmac_secret = '${secret:stripe_webhook_secret}',
    hmac_header = 'Stripe-Signature',
    hmac_algorithm = 'sha256',
    hmac_prefix = 't=,v1='  -- Stripe signature format
);

-- Process payment events
CREATE PIPELINE payment_processor AS
SELECT 
    JSON_EXTRACT(payload, '$.id') as event_id,
    JSON_EXTRACT(payload, '$.type') as event_type,
    JSON_EXTRACT(payload, '$.data.object.amount') / 100.0 as amount,
    JSON_EXTRACT(payload, '$.data.object.currency') as currency,
    JSON_EXTRACT(payload, '$.data.object.customer') as customer_id,
    TO_TIMESTAMP(CAST(JSON_EXTRACT(payload, '$.created') AS BIGINT)) as created_at
FROM stripe_webhook
WHERE JSON_EXTRACT(payload, '$.type') LIKE 'payment_intent.%';
```

### Slack Events

```sql
CREATE CONNECTION slack_webhook TO webhook (
    port = 8080,
    path = '/slack',
    format = 'json',
    auth_type = 'bearer',
    auth_token = '${secret:slack_verification_token}'
);

-- Handle Slack events
CREATE PIPELINE slack_processor AS
WITH events AS (
    SELECT 
        JSON_EXTRACT(payload, '$.type') as type,
        JSON_EXTRACT(payload, '$.challenge') as challenge,
        JSON_EXTRACT(payload, '$.event') as event
    FROM slack_webhook
)
SELECT 
    CASE 
        WHEN type = 'url_verification' THEN 
            JSON_OBJECT('challenge', challenge)
        ELSE 
            event
    END as response
FROM events;
```

## Request Handling

### Request Headers Access

```sql
CREATE CONNECTION webhook_headers TO webhook (
    port = 8080,
    path = '/with-headers',
    format = 'json'
);

-- Access request headers
CREATE TABLE webhook_requests (
    payload JSON,
    user_agent TEXT,
    content_type TEXT,
    custom_header TEXT,
    source_ip TEXT
) WITH (
    connector = 'webhook_headers',
    include_headers = true,
    header_mapping = MAP {
        'User-Agent': 'user_agent',
        'Content-Type': 'content_type',
        'X-Custom-Header': 'custom_header',
        'X-Forwarded-For': 'source_ip'
    }
);
```

### Query Parameters

```sql
CREATE CONNECTION webhook_params TO webhook (
    port = 8080,
    path = '/api',
    format = 'json',
    include_query_params = true
);

-- Access query parameters
CREATE TABLE api_requests (
    payload JSON,
    api_version TEXT,
    client_id TEXT,
    timestamp TIMESTAMP
) WITH (
    connector = 'webhook_params',
    param_mapping = MAP {
        'version': 'api_version',
        'client': 'client_id',
        'ts': 'timestamp'
    }
);
```

## Response Configuration

### Custom Response Codes

```sql
CREATE CONNECTION webhook_response TO webhook (
    port = 8080,
    path = '/custom',
    format = 'json',
    success_response_code = 201,
    error_response_code = 400,
    response_body = '{"status": "accepted", "id": "${request_id}"}'
);
```

### Conditional Responses

```sql
CREATE CONNECTION webhook_conditional TO webhook (
    port = 8080,
    path = '/conditional',
    format = 'json'
);

-- Pipeline with conditional response
CREATE PIPELINE conditional_processor AS
WITH validated AS (
    SELECT 
        *,
        CASE 
            WHEN JSON_EXTRACT(payload, '$.amount') > 0 THEN 202
            ELSE 400
        END as response_code,
        CASE 
            WHEN JSON_EXTRACT(payload, '$.amount') > 0 THEN 
                JSON_OBJECT('status', 'accepted', 'id', UUID())
            ELSE 
                JSON_OBJECT('status', 'rejected', 'error', 'Invalid amount')
        END as response_body
    FROM webhook_conditional
)
SELECT * FROM validated;
```

## Rate Limiting

### Basic Rate Limiting

```sql
CREATE CONNECTION webhook_rate_limited TO webhook (
    port = 8080,
    path = '/limited',
    format = 'json',
    rate_limit = 100,  -- 100 requests per second
    rate_limit_strategy = 'sliding_window',
    rate_limit_response = '{"error": "Rate limit exceeded"}'
);
```

### Per-Client Rate Limiting

```sql
CREATE CONNECTION webhook_per_client TO webhook (
    port = 8080,
    path = '/api',
    format = 'json',
    rate_limit_by = 'client_ip',  -- or 'api_key'
    rate_limit = 10,
    rate_limit_window = 60  -- 10 requests per minute per client
);
```

## Buffering and Batching

### Request Buffering

```sql
CREATE CONNECTION webhook_buffered TO webhook (
    port = 8080,
    path = '/buffered',
    format = 'json',
    buffer_size = 10000,
    buffer_timeout_ms = 5000,
    overflow_strategy = 'drop_oldest'  -- or 'reject'
);
```

### Batch Processing

```sql
CREATE CONNECTION webhook_batch TO webhook (
    port = 8080,
    path = '/batch',
    format = 'json',
    batch_size = 100,
    batch_timeout_ms = 1000
);

-- Process in batches
CREATE PIPELINE batch_processor AS
SELECT 
    COUNT(*) as batch_count,
    AVG(CAST(JSON_EXTRACT(payload, '$.value') AS DOUBLE)) as avg_value,
    MAX(received_at) as last_received,
    TUMBLE(received_at, INTERVAL '10 seconds') as window
FROM webhook_batch
GROUP BY TUMBLE(received_at, INTERVAL '10 seconds');
```

## Load Balancing

### Multiple Webhook Endpoints

```sql
-- Primary webhook
CREATE CONNECTION webhook_primary TO webhook (
    port = 8080,
    path = '/webhook',
    format = 'json'
);

-- Secondary webhook
CREATE CONNECTION webhook_secondary TO webhook (
    port = 8081,
    path = '/webhook',
    format = 'json'
);

-- Unified view
CREATE VIEW all_webhooks AS
SELECT *, 'primary' as source FROM webhook_primary
UNION ALL
SELECT *, 'secondary' as source FROM webhook_secondary;
```

### Health Check Endpoint

```sql
CREATE CONNECTION webhook_with_health TO webhook (
    port = 8080,
    path = '/webhook',
    format = 'json',
    health_check_path = '/health',
    health_check_response = '{"status": "healthy"}'
);
```

## Error Handling

### Validation and Error Responses

```sql
CREATE CONNECTION webhook_validated TO webhook (
    port = 8080,
    path = '/validated',
    format = 'json',
    validation_schema = '{
        "type": "object",
        "required": ["id", "amount"],
        "properties": {
            "id": {"type": "string"},
            "amount": {"type": "number", "minimum": 0}
        }
    }',
    validation_error_response = '{"error": "Invalid payload format"}'
);
```

### Dead Letter Queue

```sql
-- Main webhook
CREATE CONNECTION webhook_main TO webhook (
    port = 8080,
    path = '/main',
    format = 'json'
);

-- Pipeline with error handling
CREATE PIPELINE webhook_with_dlq AS
WITH processed AS (
    SELECT 
        *,
        CASE 
            WHEN IS_VALID_JSON(payload) THEN 'valid'
            ELSE 'invalid'
        END as status
    FROM webhook_main
)
INSERT INTO valid_events
SELECT * FROM processed WHERE status = 'valid'
UNION ALL
INSERT INTO error_events
SELECT 
    payload as raw_payload,
    'invalid_json' as error_type,
    NOW() as error_time
FROM processed WHERE status = 'invalid';
```

## Monitoring

### Webhook Metrics

```sql
-- Monitor webhook activity
SELECT 
    connection_name,
    endpoint,
    total_requests,
    successful_requests,
    failed_requests,
    avg_response_time_ms,
    requests_per_second,
    active_connections
FROM system.webhook_metrics
WHERE connection_name LIKE 'webhook_%';
```

### Request Analysis

```sql
-- Analyze request patterns
SELECT 
    DATE_TRUNC('minute', received_at) as minute,
    COUNT(*) as request_count,
    COUNT(DISTINCT source_ip) as unique_clients,
    AVG(response_time_ms) as avg_response_time,
    SUM(CASE WHEN response_code >= 400 THEN 1 ELSE 0 END) as errors
FROM webhook_requests
WHERE received_at > NOW() - INTERVAL '1 hour'
GROUP BY DATE_TRUNC('minute', received_at)
ORDER BY minute DESC;
```

## Security

### IP Whitelisting

```sql
CREATE CONNECTION webhook_whitelist TO webhook (
    port = 8080,
    path = '/restricted',
    format = 'json',
    ip_whitelist = '10.0.0.0/8,192.168.0.0/16',
    ip_whitelist_response = '{"error": "Unauthorized IP"}'
);
```

### Request Size Limits

```sql
CREATE CONNECTION webhook_limited TO webhook (
    port = 8080,
    path = '/upload',
    format = 'json',
    max_body_size = 1048576,  -- 1MB
    max_headers_size = 8192,   -- 8KB
    size_limit_response = '{"error": "Request too large"}'
);
```

## Best Practices

1. **Always use HTTPS** in production environments
2. **Implement authentication** for sensitive endpoints
3. **Validate signatures** for webhook providers that support it
4. **Set appropriate rate limits** to prevent abuse
5. **Configure request timeouts** to avoid hanging connections
6. **Use buffering** for high-volume webhooks
7. **Monitor metrics** for performance and errors
8. **Implement retry logic** for critical webhooks
9. **Use health checks** for load balancer integration
10. **Log failed requests** for debugging

## Troubleshooting

### Connection Test

```sql
-- Test webhook endpoint
SELECT test_connection('webhook_source');

-- Check endpoint status
SELECT * FROM system.webhook_endpoints
WHERE connection = 'webhook_source';
```

### Debug Logging

```sql
CREATE CONNECTION webhook_debug TO webhook (
    port = 8080,
    path = '/debug',
    format = 'json',
    debug_logging = true,
    log_headers = true,
    log_body = true
);

-- View debug logs
SELECT * FROM system.webhook_logs
WHERE connection = 'webhook_debug'
ORDER BY timestamp DESC
LIMIT 100;
```