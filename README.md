# Firetiger Cursor Plugin

The official [Cursor](https://cursor.com) plugin for [Firetiger](https://firetiger.com). Firetiger agents monitor, investigate, catalog issues, and apply runbooks autonomously -- the Cursor agents write all your code, Firetiger agents makes sure it works in prod.

## What's Included

### MCP Server

Connect to Firetiger's API for querying telemetry data, managing investigations, and interacting with AI agents directly from Cursor.

### Skills

| Skill | Description |
|-------|-------------|
| `firetiger` | Router skill - directs tasks to the appropriate specialized skill |
| `firetiger-instrument` | OpenTelemetry instrumentation for Node.js, Python, Go, and Rust applications |
| `firetiger-query` | SQL queries against Firetiger's data warehouse via MCP |
| `firetiger-investigate` | Start and manage investigations to diagnose issues with telemetry data |
| `firetiger-plan` | Plan and create new AI agents for automating workflows |
| `firetiger-run` | Run existing agents by creating sessions and interacting with them |

## Resources

- [Firetiger Documentation](https://docs.firetiger.com)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

## License

MIT
