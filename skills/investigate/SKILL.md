---
name: firetiger-investigate
description: "Start and interact with Firetiger investigations"
user_invocable: true
user_invocable_description: "Start an investigation to diagnose issues in Firetiger"
---

# Firetiger Investigation Guide

You are an expert at running investigations in Firetiger to diagnose issues and analyze telemetry data.

## Overview

Firetiger investigations are time-bounded analysis sessions that help you systematically diagnose issues using observability data. Investigations track your findings, queries, and conclusions in one place.

## Using MCP Tools

The Firetiger MCP server provides tools for managing investigations:

### Discover Investigation Schema

First, understand the available fields:

```
schema with collection: "investigations"
```

This returns the field definitions for investigations including required fields and their types.

### List Existing Investigations

Find ongoing or past investigations:

```
list with resource: "investigations"
```

You can filter by status, time range, or other criteria to find relevant investigations.

### Create a New Investigation

Start a new investigation:

```
create with resource: "investigations"
```

Include:
- **title**: Brief description of what you're investigating
- **description**: Detailed context about the issue
- **time_range**: The time window to focus on
- **services**: Relevant services to investigate

### Get Investigation Details

Retrieve a specific investigation:

```
get with name: "investigations/{id}"
```

### Update Investigation

Add findings or update status:

```
update with name: "investigations/{id}"
```

Update fields like:
- **findings**: Document what you discovered
- **status**: "in_progress", "resolved", "closed"
- **root_cause**: When identified

## Investigation Workflow

### 1. Start the Investigation

1. Use `schema` to understand available fields
2. Create a new investigation with clear title and context
3. Note the investigation ID for reference

### 2. Analyze Telemetry Data

Use the `query` tool to run SQL queries against Firetiger's data warehouse:

**Find errors in the time window:**
```sql
SELECT trace_id, service_name, span_name, status_code, timestamp
FROM traces
WHERE status_code = 'ERROR'
  AND timestamp BETWEEN '{start_time}' AND '{end_time}'
ORDER BY timestamp DESC;
```

**Analyze latency patterns:**
```sql
SELECT
    service_name,
    span_name,
    count(*) as count,
    avg(duration_ns) / 1e6 as avg_ms,
    max(duration_ns) / 1e6 as max_ms
FROM traces
WHERE timestamp BETWEEN '{start_time}' AND '{end_time}'
GROUP BY service_name, span_name
ORDER BY avg_ms DESC;
```

**Correlate logs with traces:**
```sql
SELECT l.timestamp, l.severity, l.body, t.span_name
FROM logs l
JOIN traces t ON l.trace_id = t.trace_id
WHERE l.timestamp BETWEEN '{start_time}' AND '{end_time}'
  AND l.severity IN ('ERROR', 'WARN')
ORDER BY l.timestamp;
```

### 3. Document Findings

Update the investigation with your findings:

```
update with name: "investigations/{id}"
```

Include:
- Patterns you identified
- Root cause analysis
- Affected services or endpoints
- Recommendations

### 4. Close the Investigation

When complete, update the status:

```
update with name: "investigations/{id}"
  status: "resolved"
  root_cause: "Description of the root cause"
  resolution: "How it was fixed or mitigated"
```

## Best Practices

1. **Define a clear time window**: Focus your analysis on a specific period
2. **Start broad, then narrow**: Begin with high-level queries and drill down
3. **Document as you go**: Update the investigation with findings incrementally
4. **Link related traces**: Note specific trace IDs that demonstrate the issue
5. **Consider dependencies**: Check upstream and downstream services

## Common Investigation Scenarios

### High Latency
1. Find slow traces in the time window
2. Identify which service/span is contributing most to latency
3. Check for database queries, external API calls, or resource contention

### Error Spike
1. Query for errors grouped by service and error type
2. Find the first occurrence of the error pattern
3. Correlate with deployments or configuration changes

### Missing Data
1. Check span counts by service over time
2. Look for gaps in the data
3. Verify instrumentation is working correctly
