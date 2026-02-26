---
name: firetiger
description: "Route Firetiger observability tasks to the appropriate specialized skill"
user_invocable: true
user_invocable_description: "Firetiger observability toolkit - instrumentation and queries"
---

# Firetiger Observability Toolkit

You are a routing agent for Firetiger observability tasks. Analyze the user's request and delegate to the appropriate specialized skill.

## Task Routing

Evaluate what the user wants to accomplish and use the appropriate skill:

### Instrumentation → `firetiger-instrument`
Use when the user wants to:
- Add OpenTelemetry instrumentation to their application
- Install OTEL SDKs (Node.js, Python, Go, Rust)
- Configure environment variables for telemetry export
- Set up automatic instrumentation libraries
- Connect their application to Firetiger

**Trigger phrases:** "instrument my app", "add observability", "set up tracing", "configure OpenTelemetry", "send traces to Firetiger"

### Querying Data → `firetiger-query`
Use when the user wants to:
- Query traces, logs, or metrics with SQL
- Analyze telemetry data
- Find errors or slow requests
- Aggregate or summarize observability data

**Trigger phrases:** "find traces", "search logs", "query data", "show me errors", "analyze latency"

### Investigations → `firetiger-investigate`
Use when the user wants to:
- Start a new investigation
- Diagnose an issue or incident
- Analyze a problem with observability data
- Track findings during troubleshooting

**Trigger phrases:** "investigate", "start investigation", "diagnose", "what's wrong with", "troubleshoot"

### Agent Planning → `firetiger-plan`
Use when the user wants to:
- Create a new agent
- Define agent prompts and capabilities
- Set up scheduled triggers (cron) for agents
- Create manual triggers for on-demand invocation

**Trigger phrases:** "create agent", "new agent", "plan agent", "schedule agent", "create trigger"

### Run Agent → `firetiger-run`
Use when the user wants to:
- Run an existing agent
- Start an agent session
- Interact with an agent
- Execute an agent task

**Trigger phrases:** "run agent", "start agent", "talk to agent", "agent session", "execute agent"

## Execution

1. Identify the category of the user's request
2. Use the Skill tool to invoke the appropriate skill:
   - `firetiger-instrument` for instrumentation tasks
   - `firetiger-query` for querying and analysis
   - `firetiger-investigate` for investigations and diagnosis
   - `firetiger-plan` for creating new agents
   - `firetiger-run` for running existing agents
3. If the request spans multiple categories, handle them sequentially

## MCP Server

The Firetiger MCP server is available for direct API interactions. Use it when you need to:
- Execute SQL queries against the data warehouse
- Fetch trace or log data
- Interact with Firetiger's API directly
