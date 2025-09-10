---
sidebar_position: 1
title: First Pipeline
---

# Building Your First Pipeline

This tutorial will guide you through creating your first streaming pipeline with Laminar. We'll build a simple pipeline that reads events from a source, performs transformations, and writes results to a sink.

## Prerequisites

Before starting, ensure you have:
- Access to a Laminar workspace (sign up at [laminar.cloud](https://laminar.cloud))
- Your API credentials (available in your workspace settings)
- Basic understanding of SQL

## Step 1: Set Up Your Environment

### Configure API Access

Set up your Laminar API credentials:

```bash
# Set your workspace URL and API key
export LAMINAR_API_URL="https://your-workspace.laminar.cloud/api/v1"
export LAMINAR_API_KEY="your-api-key-here"
```

You can find your workspace URL and API key in the Laminar Console under **Settings** → **API Access**.

## Step 2: Create a Data Source

For this tutorial, we'll use a webhook source that generates sample events.

### Create Connection Profile

First, create a connection profile for our webhook source:

```bash
curl -X POST $LAMINAR_API_URL/connection_profiles \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "tutorial-webhook",
    "connector": "webhook",
    "config": {
      "path": "/events",
      "port": 8080
    }
  }'
```

### Create Source Table

Now create a connection table that defines the schema:

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "events_source",
    "connector": "webhook",
    "connection_profile_id": "tutorial-webhook",
    "table_type": "source",
    "schema": {
      "fields": [
        {"name": "event_id", "data_type": "STRING", "nullable": false},
        {"name": "user_id", "data_type": "BIGINT", "nullable": false},
        {"name": "event_type", "data_type": "STRING", "nullable": false},
        {"name": "amount", "data_type": "DOUBLE", "nullable": true},
        {"name": "timestamp", "data_type": "TIMESTAMP", "nullable": false}
      ]
    },
    "event_time_field": "timestamp",
    "watermark_delay": "5 seconds"
  }'
```

## Step 3: Create a Sink

Let's create a sink to write our processed data:

```bash
curl -X POST $LAMINAR_API_URL/connection_tables \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "events_output",
    "connector": "webhook",
    "table_type": "sink",
    "schema": {
      "fields": [
        {"name": "user_id", "data_type": "BIGINT", "nullable": false},
        {"name": "event_count", "data_type": "BIGINT", "nullable": false},
        {"name": "total_amount", "data_type": "DOUBLE", "nullable": true},
        {"name": "window_start", "data_type": "TIMESTAMP", "nullable": false},
        {"name": "window_end", "data_type": "TIMESTAMP", "nullable": false}
      ]
    },
    "config": {
      "url": "https://webhook.site/your-unique-url",
      "method": "POST",
      "format": "json"
    }
  }'
```

## Step 4: Write Your Pipeline SQL

Now let's create a pipeline that aggregates events by user in 1-minute windows:

```sql
-- Calculate user metrics in 1-minute tumbling windows
INSERT INTO events_output
SELECT 
    user_id,
    COUNT(*) as event_count,
    SUM(amount) as total_amount,
    TUMBLE_START(timestamp, INTERVAL '1' MINUTE) as window_start,
    TUMBLE_END(timestamp, INTERVAL '1' MINUTE) as window_end
FROM events_source
GROUP BY 
    user_id,
    TUMBLE(timestamp, INTERVAL '1' MINUTE)
```

## Step 5: Create and Deploy the Pipeline

### Create the Pipeline

```bash
curl -X POST $LAMINAR_API_URL/pipelines \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "user_aggregation_pipeline",
    "query": "INSERT INTO events_output SELECT user_id, COUNT(*) as event_count, SUM(amount) as total_amount, TUMBLE_START(timestamp, INTERVAL '\''1'\'' MINUTE) as window_start, TUMBLE_END(timestamp, INTERVAL '\''1'\'' MINUTE) as window_end FROM events_source GROUP BY user_id, TUMBLE(timestamp, INTERVAL '\''1'\'' MINUTE)",
    "parallelism": 2
  }'
```

The pipeline will automatically start running.

## Step 6: Send Test Data

Laminar provides a test data generator for webhook sources. You can send test events through the Laminar Console or use the API:

```bash
# Generate test events through the API
curl -X POST $LAMINAR_API_URL/pipelines/user_aggregation_pipeline/test-data \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "events_source",
    "count": 10,
    "interval_ms": 1000
  }'
```

Alternatively, you can send custom events to your webhook endpoint:

```bash
# Get your webhook endpoint URL
WEBHOOK_URL=$(curl -s -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/connection_tables/events_source | jq -r '.webhook_url')

# Send test events
for i in {1..10}; do
  curl -X POST $WEBHOOK_URL \
    -H "Content-Type: application/json" \
    -d "{
      \"event_id\": \"evt_$i\",
      \"user_id\": $((RANDOM % 3 + 1)),
      \"event_type\": \"purchase\",
      \"amount\": $((RANDOM % 100 + 10)).99,
      \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)\"
    }"
  sleep 1
done
```

## Step 7: Monitor Your Pipeline

### Check Pipeline Status

```bash
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/user_aggregation_pipeline
```

### View Jobs

```bash
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/user_aggregation_pipeline/jobs
```

### Check Metrics

```bash
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/user_aggregation_pipeline/jobs/{job_id}/metrics
```

### View Logs

```bash
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/user_aggregation_pipeline/jobs/{job_id}/logs
```

## Step 8: Viewing Results

If you're using the webhook sink, you can see the aggregated results being posted to your endpoint. For debugging, you can also create a preview pipeline:

```bash
curl -X POST $LAMINAR_API_URL/pipelines/preview \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT * FROM events_output LIMIT 10",
    "enable_sinks": false
  }'
```

Then fetch the output:
```bash
curl -H "Authorization: Bearer $LAMINAR_API_KEY" \
  $LAMINAR_API_URL/pipelines/{preview_pipeline_id}/jobs/{job_id}/output
```

## Advanced Features

### Adding Filters

Modify your pipeline to only process certain events:

```sql
INSERT INTO events_output
SELECT 
    user_id,
    COUNT(*) as event_count,
    SUM(amount) as total_amount,
    TUMBLE_START(timestamp, INTERVAL '1' MINUTE) as window_start,
    TUMBLE_END(timestamp, INTERVAL '1' MINUTE) as window_end
FROM events_source
WHERE event_type = 'purchase' 
  AND amount > 50
GROUP BY 
    user_id,
    TUMBLE(timestamp, INTERVAL '1' MINUTE)
```

### Using Multiple Sources

Join data from multiple sources:

```sql
INSERT INTO enriched_events
SELECT 
    e.user_id,
    e.event_type,
    e.amount,
    u.user_name,
    u.user_category
FROM events_source e
JOIN users_source u ON e.user_id = u.user_id
```

### Adding UDFs

Create a custom function for complex logic:

```bash
curl -X POST $LAMINAR_API_URL/udfs \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "calculate_discount",
    "definition": "pub fn calculate_discount(amount: f64, user_category: String) -> f64 {\n    match user_category.as_str() {\n        \"premium\" => amount * 0.2,\n        \"gold\" => amount * 0.1,\n        _ => 0.0\n    }\n}",
    "language": "rust",
    "return_type": "DOUBLE",
    "parameters": [
      {"name": "amount", "type": "DOUBLE", "nullable": false},
      {"name": "user_category", "type": "STRING", "nullable": false}
    ]
  }'
```

Use the UDF in your pipeline:
```sql
SELECT 
    user_id,
    amount,
    calculate_discount(amount, user_category) as discount
FROM events_source
```

## Clean Up

To stop and delete your pipeline:

```bash
# Stop the pipeline
curl -X PATCH $LAMINAR_API_URL/pipelines/user_aggregation_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"stop": "immediate"}'

# Delete the pipeline
curl -X DELETE $LAMINAR_API_URL/pipelines/user_aggregation_pipeline \
  -H "Authorization: Bearer $LAMINAR_API_KEY"

# Delete connection tables
curl -X DELETE $LAMINAR_API_URL/connection_tables/events_source \
  -H "Authorization: Bearer $LAMINAR_API_KEY"
curl -X DELETE $LAMINAR_API_URL/connection_tables/events_output \
  -H "Authorization: Bearer $LAMINAR_API_KEY"

# Delete connection profile
curl -X DELETE $LAMINAR_API_URL/connection_profiles/tutorial-webhook \
  -H "Authorization: Bearer $LAMINAR_API_KEY"
```

## Next Steps

Congratulations! You've created your first Laminar pipeline. Next, explore:

- [Kafka Integration](./kafka) - Work with Apache Kafka
- [Change Data Capture](./cdc) - Implement CDC pipelines
- [Iceberg Tables](./iceberg) - Use Apache Iceberg
- [Advanced SQL](../../sql/intro) - Learn advanced SQL features
- [Monitoring & Observability](../../observability/intro) - Monitor your pipelines