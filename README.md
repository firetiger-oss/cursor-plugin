# Firetiger Cursor Plugin

The official [Cursor](https://cursor.com) plugin for [Firetiger](https://firetiger.com) - an observability platform that accepts OpenTelemetry data and stores telemetry in Apache Iceberg tables.

## What's Included

### MCP Server

Connect to Firetiger's API for executing TraceQL queries, searching logs, and fetching trace data directly from Cursor.

### Skills

| Skill | Description |
|-------|-------------|
| `firetiger` | Router skill - directs observability tasks to the appropriate specialized skill |
| `firetiger-setup` | OpenTelemetry instrumentation for Node.js, Python, Go, and Rust applications |
| `firetiger-query` | TraceQL queries, log search, metrics analysis, and DuckDB integration |
| `firetiger-aws` | AWS integrations - CloudWatch Logs forwarding and ECS task state events |

## Installation

Install this plugin in Cursor to get started:

1. Open Cursor Settings
2. Navigate to Plugins
3. Search for "Firetiger"
4. Click Install

## Usage

### Instrument Your Application

Ask Cursor to add observability to your application:

> "Add OpenTelemetry tracing to my Node.js app for Firetiger"

### Query Your Data

Search traces and logs using natural language:

> "Find all traces where the checkout service returned an error"
> "Show me the slowest API requests from the last hour"

### Set Up AWS Integrations

Configure CloudWatch and ECS integrations:

> "Forward my CloudWatch logs to Firetiger"
> "Set up ECS task state change events for Firetiger"

## Resources

- [Firetiger Documentation](https://docs.firetiger.com)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [TraceQL Reference](https://docs.firetiger.com/traceql)

## License

MIT
