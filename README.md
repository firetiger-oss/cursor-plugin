<p align="center">
  <img src="assets/firetiger.svg" alt="Firetiger logo" width="220">
</p>

# Firetiger Cursor Plugin

The official [Cursor](https://cursor.com) plugin for [Firetiger](https://firetiger.com). Firetiger agents monitor, investigate, catalog issues, and apply runbooks autonomously while Cursor helps you build and debug code.

## Repository Layout

This repository publishes a single plugin:

- `.cursor-plugin/plugin.json`: plugin manifest
- `.cursor-plugin/marketplace.json`: marketplace manifest pointing at the root plugin
- `skills/`: Firetiger skills exposed to Cursor
- `mcp.json`: Firetiger MCP server configuration
- `assets/firetiger.svg`: plugin logo

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

## Validation

Run the repository check before publishing changes:

```bash
node scripts/validate-template.mjs
```

## Resources

- [Firetiger Documentation](https://docs.firetiger.com)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

## License

MIT
