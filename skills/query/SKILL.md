---
name: firetiger-query
description: "Query traces, logs, and metrics in Firetiger using TraceQL"
user_invocable: true
user_invocable_description: "Search and analyze telemetry data with TraceQL queries"
---

# Firetiger Query Guide

You are an expert at querying observability data in Firetiger using TraceQL and other query methods.

## TraceQL Overview

TraceQL is Firetiger's query language for searching traces. It uses a pipeline syntax similar to Grafana Tempo.

### Basic Syntax

```
{ <spanset selectors> } | <pipeline operators>
```

### Spanset Selectors

Select spans based on attributes:

```traceql
// By service name
{ service.name = "api-gateway" }

// By span name
{ name = "HTTP GET" }

// By status
{ status = error }

// By duration (in nanoseconds, use duration literals)
{ duration > 1s }

// By resource attributes
{ resource.service.name = "checkout" }

// By span attributes
{ span.http.method = "POST" }
{ span.http.status_code >= 400 }
```

### Comparison Operators

- `=` - Equals
- `!=` - Not equals
- `>`, `>=`, `<`, `<=` - Numeric comparisons
- `=~` - Regex match
- `!~` - Regex not match

### Combining Conditions

```traceql
// AND (within same selector)
{ service.name = "api" && status = error }

// OR (separate selectors)
{ service.name = "api" } || { service.name = "web" }

// Chained spansets (spans in same trace)
{ service.name = "frontend" } >> { service.name = "backend" }
```

### Pipeline Operators

Process selected spans:

```traceql
// Count spans
{ status = error } | count()

// Average duration
{ service.name = "api" } | avg(duration)

// Select specific attributes
{ name = "HTTP GET" } | select(span.http.url, duration)

// Filter by aggregate
{ service.name = "api" } | by(span.http.route) | count() > 100
```

## Common Query Patterns

### Find Slow Requests

```traceql
{ service.name = "api-gateway" && duration > 2s }
```

### Find Errors

```traceql
{ status = error }

// With specific error message
{ span.error.message =~ ".*timeout.*" }
```

### Find Requests by User

```traceql
{ span.user.id = "user-123" }
```

### Trace a Request Path

```traceql
{ span.http.url = "/api/checkout" } >> { service.name = "payment-service" }
```

### Find Database Queries

```traceql
{ span.db.system = "postgresql" && duration > 100ms }
```

### Count by Endpoint

```traceql
{ service.name = "api" } | by(span.http.route) | count()
```

### P99 Latency by Service

```traceql
{ } | by(service.name) | quantile(duration, 0.99)
```

## Log Queries

Search logs using attribute filters:

```
service.name = "api" AND level = "error"
message =~ "connection.*failed"
trace_id = "abc123..."
```

### Link Logs to Traces

Logs with `trace_id` and `span_id` attributes are automatically correlated with traces.

## Metrics Queries

Query metrics stored in Firetiger:

```promql
// Request rate
rate(http_requests_total[5m])

// Error rate
sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

// P99 latency histogram
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

## DuckDB Integration

For advanced analytics, query Firetiger's Iceberg tables directly with DuckDB:

```sql
-- Connect to Firetiger catalog
ATTACH 'firetiger' AS ft (TYPE ICEBERG);

-- Query traces
SELECT
    trace_id,
    service_name,
    span_name,
    duration_ns / 1e6 as duration_ms
FROM ft.traces
WHERE timestamp > now() - INTERVAL '1 hour'
    AND service_name = 'api-gateway'
ORDER BY duration_ns DESC
LIMIT 100;

-- Aggregate by endpoint
SELECT
    span_attributes['http.route'] as route,
    count(*) as requests,
    avg(duration_ns) / 1e6 as avg_ms,
    percentile_cont(0.99) WITHIN GROUP (ORDER BY duration_ns) / 1e6 as p99_ms
FROM ft.traces
WHERE timestamp > now() - INTERVAL '1 hour'
GROUP BY 1
ORDER BY requests DESC;
```

## Using the MCP Server

The Firetiger MCP server provides tools for executing queries programmatically:

- `firetiger_query_traces` - Execute TraceQL queries
- `firetiger_search_logs` - Search log entries
- `firetiger_get_trace` - Fetch a specific trace by ID

## Tips

1. **Start broad, then narrow**: Begin with service/time filters, then add specific conditions
2. **Use trace IDs**: When debugging, find one problematic trace and explore from there
3. **Check timestamps**: Ensure your query time range includes the data you're looking for
4. **Attribute names**: Use `resource.*` for resource attributes, `span.*` for span attributes
