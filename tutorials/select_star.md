---
title: Select Star - Basic Data Passthrough
---


## Overview
This example demonstrates the simplest streaming pipeline - reading all data from a source and writing it directly to a sink without any transformations.

## Key Concepts
- Basic table creation with source and sink configurations
- SELECT * syntax for passing all columns through
- Direct data flow without transformations

## Test Data
**Download**: [cars.json](/test-data/cars.json)

The cars dataset contains ride-sharing events with the following structure:
- `timestamp`: Event time
- `driver_id`: Unique identifier for drivers  
- `event_type`: Type of event (pickup or dropoff)
- `location`: Location name

## SQL Query

### 1. Source Configuration
```sql
CREATE TABLE cars (
  timestamp TIMESTAMP,
  driver_id BIGINT,
  event_type TEXT,
  location TEXT
) WITH (
  connector = 'single_file',
  path = '$input_dir/cars.json',
  format = 'json',
  type = 'source'
);
```
 This creates a source table that reads ride-sharing events from `cars.json`. The schema defines four fields matching the JSON structure. The `single_file` connector reads the entire file once and streams each JSON object as a separate event.

### 2. Sink Configuration
```sql
CREATE TABLE cars_output (
  timestamp TIMESTAMP,
  driver_id BIGINT,
  event_type TEXT,
  location TEXT
) WITH (
  connector = 'single_file',
  path = '$output_path',
  format = 'json',
  type = 'sink'
);
```
 This creates an output table with identical schema to receive the streamed data. Each processed record will be written as a JSON object to the output file, preserving the original structure.

### 3. Processing Pipeline
```sql
INSERT INTO cars_output 
SELECT * FROM cars
```
 This is the simplest possible streaming pipeline - a direct passthrough. The `SELECT *` reads all columns from the source and the `INSERT INTO` writes them unchanged to the sink. This pattern is useful for data migration, creating backups, or as a starting point for more complex transformations.

## Expected Output
The output will be identical to the input - each record from cars.json will appear in the output with the same structure and values. This pattern is useful for:
- Data migration between systems
- Creating data copies for backup
- Initial pipeline testing before adding transformations