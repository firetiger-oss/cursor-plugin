---
name: firetiger-query
description: "Query telemetry data in Firetiger using SQL via MCP"
user_invocable: true
user_invocable_description: "Query traces, logs, and metrics with SQL"
---

# Firetiger Query Guide

You are an expert at querying observability data in Firetiger using SQL via the MCP server.

## Overview

Firetiger stores telemetry data in Apache Iceberg tables and provides a query interface using DuckDB SQL. Use the Firetiger MCP server's `query` tool to execute SQL queries against the data warehouse.

## Using the MCP Query Tool

The Firetiger MCP server provides a `query` tool that executes DuckDB SQL against Firetiger's Iceberg data warehouse.

### Capabilities

- Standard SQL including JOINs, CTEs, and aggregate functions
- DuckDB SQL dialect and functions
- Access to traces, logs, and metrics tables

## Common Query Patterns

### Find Recent Traces

```sql
SELECT
    trace_id,
    service_name,
    span_name,
    duration_ns / 1e6 as duration_ms,
    timestamp
FROM traces
WHERE timestamp > now() - INTERVAL '1 hour'
ORDER BY timestamp DESC
LIMIT 100;
```

### Find Slow Requests

```sql
SELECT
    trace_id,
    service_name,
    span_name,
    duration_ns / 1e6 as duration_ms
FROM traces
WHERE duration_ns > 2000000000  -- 2 seconds in nanoseconds
    AND timestamp > now() - INTERVAL '1 hour'
ORDER BY duration_ns DESC
LIMIT 50;
```

### Find Errors

```sql
SELECT
    trace_id,
    service_name,
    span_name,
    status_code,
    timestamp
FROM traces
WHERE status_code = 'ERROR'
    AND timestamp > now() - INTERVAL '1 hour'
ORDER BY timestamp DESC;
```

### Aggregate by Service

```sql
SELECT
    service_name,
    count(*) as span_count,
    avg(duration_ns) / 1e6 as avg_duration_ms,
    max(duration_ns) / 1e6 as max_duration_ms
FROM traces
WHERE timestamp > now() - INTERVAL '1 hour'
GROUP BY service_name
ORDER BY span_count DESC;
```

### Latency Percentiles by Endpoint

```sql
SELECT
    span_attributes['http.route'] as route,
    count(*) as requests,
    avg(duration_ns) / 1e6 as avg_ms,
    percentile_cont(0.50) WITHIN GROUP (ORDER BY duration_ns) / 1e6 as p50_ms,
    percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_ns) / 1e6 as p95_ms,
    percentile_cont(0.99) WITHIN GROUP (ORDER BY duration_ns) / 1e6 as p99_ms
FROM traces
WHERE timestamp > now() - INTERVAL '1 hour'
    AND span_attributes['http.route'] IS NOT NULL
GROUP BY 1
ORDER BY requests DESC;
```

### Search Logs

```sql
SELECT
    timestamp,
    service_name,
    severity,
    body,
    trace_id
FROM logs
WHERE timestamp > now() - INTERVAL '1 hour'
    AND severity = 'ERROR'
ORDER BY timestamp DESC
LIMIT 100;
```

### Correlate Logs with Traces

```sql
SELECT
    l.timestamp,
    l.body as log_message,
    t.span_name,
    t.service_name
FROM logs l
JOIN traces t ON l.trace_id = t.trace_id
WHERE l.timestamp > now() - INTERVAL '1 hour'
    AND l.severity = 'ERROR'
ORDER BY l.timestamp DESC;
```

## Tips

1. **Always filter by timestamp**: Include a time range filter to limit the data scanned
2. **Use LIMIT**: Start with small result sets and increase as needed
3. **Check attribute names**: Span attributes are accessed via `span_attributes['key']`
4. **Duration is in nanoseconds**: Divide by 1e6 for milliseconds or 1e9 for seconds
5. **Use CTEs for complex queries**: Break down complex analysis into readable steps

## Discovering Schema

Use the MCP `schema` tool to discover available tables, columns, and their types before writing queries.
