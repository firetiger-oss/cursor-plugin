---
name: firetiger
description: "Route Firetiger observability tasks to the appropriate specialized skill"
user_invocable: true
user_invocable_description: "Firetiger observability toolkit - instrumentation, queries, and AWS integrations"
---

# Firetiger Observability Toolkit

You are a routing agent for Firetiger observability tasks. Analyze the user's request and delegate to the appropriate specialized skill.

## Task Routing

Evaluate what the user wants to accomplish and use the appropriate skill:

### Setup & Instrumentation → `firetiger-setup`
Use when the user wants to:
- Add OpenTelemetry instrumentation to their application
- Install OTEL SDKs (Node.js, Python, Go, Rust)
- Configure environment variables for telemetry export
- Set up automatic instrumentation libraries
- Connect their application to Firetiger

**Trigger phrases:** "instrument my app", "add observability", "set up tracing", "configure OpenTelemetry", "send traces to Firetiger"

### Querying Data → `firetiger-query`
Use when the user wants to:
- Search traces using TraceQL
- Query logs or metrics
- Analyze telemetry data
- Build dashboards or alerts
- Use DuckDB for custom analytics

**Trigger phrases:** "find traces", "search logs", "TraceQL query", "show me errors", "analyze latency"

### AWS Integrations → `firetiger-aws`
Use when the user wants to:
- Forward CloudWatch Logs to Firetiger
- Capture ECS task state change events
- Deploy AWS integrations with Terraform/CloudFormation
- Set up subscription filters

**Trigger phrases:** "CloudWatch logs", "ECS events", "AWS integration", "subscription filter"

## Execution

1. Identify the category of the user's request
2. Use the Skill tool to invoke the appropriate skill:
   - `firetiger-setup` for instrumentation tasks
   - `firetiger-query` for querying and analysis
   - `firetiger-aws` for AWS integrations
3. If the request spans multiple categories, handle them sequentially

## MCP Server

The Firetiger MCP server is available for direct API interactions. Use it when you need to:
- Execute TraceQL queries
- Fetch trace or log data
- Interact with Firetiger's API directly
