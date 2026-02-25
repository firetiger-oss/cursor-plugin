# Firetiger Cursor Plugin

The official [Cursor](https://cursor.com) plugin for [Firetiger](https://firetiger.com) - an observability platform that accepts OpenTelemetry data and stores telemetry in Apache Iceberg tables.

## What's Included

### MCP Server

Connect to Firetiger's API for executing SQL queries against the data warehouse and interacting with Firetiger resources directly from Cursor.

### Skills

| Skill | Description |
|-------|-------------|
| `firetiger` | Router skill - directs observability tasks to the appropriate specialized skill |
| `firetiger-instrument` | OpenTelemetry instrumentation for Node.js, Python, Go, and Rust applications |
| `firetiger-query` | SQL queries against Firetiger's Iceberg data warehouse via MCP |
| `firetiger-investigate` | Start and manage investigations to diagnose issues with telemetry data |
| `firetiger-plan` | Plan and create new AI agents for automating observability workflows |
| `firetiger-run` | Run existing agents by creating sessions and interacting with them |

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

Search traces and logs using SQL:

> "Find all traces where the checkout service returned an error"
> "Show me the slowest API requests from the last hour"

## Resources

- [Firetiger Documentation](https://docs.firetiger.com)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

## License

MIT
